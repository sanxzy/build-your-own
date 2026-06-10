---
title: "Terminal Abstraction, Input, and Keybindings"
description: "Build the Terminal abstraction over raw stdin/stdout, platform-aware key parsing with Kitty keyboard protocol support, and a configurable keybinding system with chord sequences."
category: terminal-ui
type: tutorial
tags: [Terminal, ProcessTerminal, VirtualTerminal, stdin, stdout, resize, key parsing, Kitty keyboard protocol, matchesKey, keybindings, chord sequences, action dispatch, tui]
keywords: [terminal abstraction, raw mode, key parsing, VT sequences, Kitty protocol, keybinding config, chord matching]
sources: [S35, S40, S41]
---

**TL;DR** — Terminals are messy. Raw input arrives as byte sequences that differ by platform and terminal emulator, and output requires careful raw mode management. We'll build a `Terminal` abstraction with a real implementation (`ProcessTerminal`) and a virtual one for testing, a platform-aware key parser that handles VT/xterm sequences and the Kitty keyboard protocol, and a configurable keybinding system with chord sequences.

## The Terminal abstraction

Create `packages/tui/src/terminal.ts`. The `Terminal` interface defines what the TUI needs:

```ts
interface Terminal {
  onData(cb: (data: Buffer) => void): void;
  onResize(cb: (cols: number, rows: number) => void): void;
  write(data: string): void;
  getSize(): { cols: number; rows: number };
  enterRawMode(): void;
  leaveRawMode(): void;
}
```

### ProcessTerminal: the real implementation

`ProcessTerminal` wraps stdin/stdout with raw mode management:

```ts
class ProcessTerminal implements Terminal {
  private rawMode = false;

  constructor() {
    this.#stdin = process.stdin;
    this.#stdout = process.stdout;
  }

  enterRawMode(): void {
    if (this.rawMode) return;
    this.#stdin.setRawMode(true);
    this.#stdin.resume();
    this.#stdin.on("data", this.#onData);
    this.#stdout.on("resize", this.#onResize);
    this.rawMode = true;
  }

  leaveRawMode(): void {
    if (!this.rawMode) return;
    this.#stdin.setRawMode(false);
    this.#stdin.pause();
    this.#stdin.off("data", this.#onData);
    this.#stdout.off("resize", this.#onResize);
    this.rawMode = false;
  }

  write(data: string): void {
    this.#stdout.write(data);
  }
}
```

Raw mode disables line buffering and echo — every keypress is delivered immediately as a byte sequence, not after Enter.

### VirtualTerminal: for testing

```ts
class VirtualTerminal implements Terminal {
  cols: number;
  rows: number;
  output = "";  // captured output for assertions

  write(data: string): void { this.output += data; }
  simulateKey(data: Buffer): void { /* notify listeners */ }
  resize(cols: number, rows: number): void { /* notify listeners */ }
}
```

VirtualTerminal lets us write deterministic tests for the TUI: simulate keypresses, capture output, and assert on the rendered screen without a real terminal.

## Key parsing

Raw terminal input arrives as byte sequences. The `parseKey` function normalizes them into `Key` objects:

```ts
interface Key {
  name: string;          // "a", "Enter", "Backspace", "Up", etc.
  ctrl: boolean;
  alt: boolean;
  shift: boolean;
  meta: boolean;
  sequence: string;      // raw byte sequence
}

function parseKey(data: Buffer): Key | null {
  const seq = data.toString();
  // Single bytes: ctrl+key combinations
  if (data.length === 1) {
    const byte = data[0];
    if (byte < 32) {
      // Ctrl+letter: bytes 1-26 map to ctrl+a through ctrl+z
      return { name: String.fromCharCode(byte + 96), ctrl: true, /* ... */ };
    }
    return { name: seq, ctrl: false, /* ... */ };
  }

  // Escape sequences: \x1b[...
  if (seq.startsWith("\x1b[")) {
    return parseCsiSequence(seq);  // arrow keys, home, end, etc.
  }

  // Kitty keyboard protocol: \x1b[...u
  if (seq.startsWith("\x1b[") && seq.endsWith("u")) {
    return parseKittySequence(seq);
  }

  // Alt+key: \x1b followed by the key
  if (seq.startsWith("\x1b") && data.length === 2) {
    return { name: seq[1], alt: true, /* ... */ };
  }

  return { name: seq, /* ... */ };
}
```

### The Kitty keyboard protocol

Modern terminals (Kitty, WezTerm, Ghostty) support the Kitty keyboard protocol, which encodes modifiers, key codes, and key events (press/repeat/release) in a structured escape sequence. This gives us:

- Distinction between `Ctrl+I` and `Tab` (they're the same byte in legacy mode)
- `Shift+Enter` detection
- Key release events (useful for hold-to-repeat)

The parser extracts the Unicode codepoint, modifiers, and event type from the Kitty sequence format: `\x1b[codepoint;modifiers u`.

## Keybindings

Keybindings map key patterns to actions. Create `packages/tui/src/keybindings.ts`:

```ts
interface Keybinding {
  keys: string;     // e.g., "ctrl+k", "alt+enter"
  action: string;   // e.g., "submit", "abort", "clear"
  description: string;
}
```

The `matchesKey` function checks if a parsed `Key` matches a binding pattern:

```ts
function matchesKey(key: Key, pattern: string): boolean {
  const parts = pattern.toLowerCase().split("+");
  const modifiers = parts.slice(0, -1);
  const keyName = parts[parts.length - 1];

  const modMatch = {
    ctrl: modifiers.includes("ctrl") === key.ctrl,
    alt: modifiers.includes("alt") === key.alt,
    shift: modifiers.includes("shift") === key.shift,
    meta: modifiers.includes("meta") === key.meta,
  };

  return Object.values(modMatch).every(Boolean) && key.name.toLowerCase() === keyName;
}
```

### Chord sequences

Some actions require a sequence of keypresses (like Vim's `gg` or `dd`). Chords are keybindings where one binding sets up a prefix and the next keypress completes the sequence:

```ts
class KeybindingManager {
  private bindings: Keybinding[] = [];
  private chordPrefix: string | null = null;
  private chordTimeout: NodeJS.Timeout | null = null;

  handleKey(key: Key): string | null {
    const prefix = this.chordPrefix;

    // Check for chord completion
    if (prefix) {
      clearTimeout(this.chordTimeout!);
      this.chordPrefix = null;
      const chordBinding = this.bindings.find(
        b => b.keys === `${prefix} ${key.name}`
      );
      if (chordBinding) return chordBinding.action;
    }

    // Check for chord start
    const binding = this.bindings.find(b => {
      if (b.keys.includes(" ")) {
        const [first] = b.keys.split(" ");
        if (matchesKey(key, first)) {
          this.chordPrefix = first;
          this.chordTimeout = setTimeout(() => { this.chordPrefix = null; }, 1000);
          return false; // don't return action yet
        }
      }
      return matchesKey(key, b.keys);
    });

    return binding?.action ?? null;
  }
}
```

The chord timeout (1 second) prevents getting stuck in chord mode if the user pauses. After the timeout, the prefix is cleared and the next keypress starts fresh.

## What we've built

- **`Terminal` interface** — abstract over stdin/stdout with raw mode management
- **`ProcessTerminal`** — real terminal implementation
- **`VirtualTerminal`** — testable terminal for deterministic assertions
- **Key parser** — normalizes VT/xterm and Kitty protocol byte sequences into `Key` objects
- **Keybinding system** — matches key patterns with modifier awareness and chord sequences

---

← Previous: [The TUI Class and Differential Render Engine](./tui-class-and-render-engine.md) · Next: [Built-In Components: Widgets and Layout](./components-and-layout.md) →
