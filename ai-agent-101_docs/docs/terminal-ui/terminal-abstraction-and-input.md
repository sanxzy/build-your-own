---
title: "Terminal Abstraction, Input Handling, and Keybindings"
description: "How the Terminal interface and ProcessTerminal class decouple the TUI from stdin/stdout, and how the keybindings system turns raw bytes into named actions."
category: terminal-ui
type: tutorial
tags: [Terminal, ProcessTerminal, VirtualTerminal, stdin, stdout, resize, keybindings, matchesKey, Key, KeyId, Kitty keyboard protocol, input, testable terminal, tui, KeybindingsManager, TUI_KEYBINDINGS, StdinBuffer, raw mode, bracketed paste, modifyOtherKeys, drainInput, setProgress, OSC]
keywords: [terminal abstraction, keyboard input handling, key event, keybinding registry, declaration merging, key descriptor, kitty protocol, terminal resize, raw stdin, process.stdin, process.stdout]
sources: [S39, S41]
---

**TL;DR** — The TUI needs a place to write characters and a source of keystrokes and resize events. Hard-wiring `process.stdin`/`process.stdout` directly makes the TUI untestable and couples it to a real terminal. This chapter introduces the `Terminal` interface that decouples those concerns, the `ProcessTerminal` class that implements it for a live terminal, and then builds the keybindings layer on top — showing how raw bytes become named actions via `KeybindingsManager` and `matchesKey`, with Kitty keyboard protocol support for richer key information. By the end you will understand how input flows from the physical keyboard all the way to a named action like `"tui.input.submit"`.

# Terminal Abstraction, Input Handling, and Keybindings

## The problem: hard-wiring the terminal

In [the previous chapter](./the-tui-class-and-render-engine.md) we built the `TUI` class — the render engine that owns the screen. The TUI needs two things from the environment: somewhere to write ANSI output, and a stream of keyboard and resize events. The obvious move is to reach directly for `process.stdout` and `process.stdin`. But if we do that, every test that instantiates a `TUI` immediately tries to manipulate a real terminal, and most CI environments have no terminal at all.

We need the same design move we use for LLM providers: define a narrow interface, give it a real implementation for production, and leave room for a test double. That interface is `Terminal`.

## The `Terminal` interface

The `Terminal` interface (from `terminal.ts`) captures everything the TUI needs from the host environment. Let's walk through it in groups so nothing is mysterious.

### Input and lifecycle

```ts
export interface Terminal {
  // Called once to start listening; the TUI passes two callbacks:
  //   onInput  — called with each raw byte sequence from stdin
  //   onResize — called whenever the terminal window is resized
  start(onInput: (data: string) => void, onResize: () => void): void;

  // Stop the terminal and restore its original state (e.g. raw mode).
  stop(): void;

  // Drain any remaining stdin bytes before exit, to prevent key-release
  // escape sequences from leaking into the parent shell.
  //   maxMs  — maximum drain window (default: 1000 ms)
  //   idleMs — stop early if no bytes arrive within this window (default: 50 ms)
  drainInput(maxMs?: number, idleMs?: number): Promise<void>;
}
```

`start` is the entry point. The TUI calls it once, passing two callbacks: `onInput` fires each time a raw byte sequence arrives from the keyboard, and `onResize` fires whenever the window dimensions change. The `Terminal` implementation is responsible for wiring those callbacks to whatever the real (or simulated) source happens to be.

`drainInput` deserves a moment's attention. When the TUI exits, the physical keyboard may still be in the middle of sending a multi-byte escape sequence (for example, a key-release event from the Kitty keyboard protocol). If we close stdin immediately, those leftover bytes land in the parent shell's input buffer and look like garbage. `drainInput` waits up to `maxMs` milliseconds for the stream to go quiet for at least `idleMs` milliseconds, then returns — clean exit.

### Output

```ts
  // Write a string of ANSI escape sequences or plain text to the terminal.
  write(data: string): void;
```

All rendering goes through `write`. There is nothing else. The TUI never reaches for `process.stdout` directly — only the `Terminal` implementation does.

### Dimensions

```ts
  get columns(): number;  // terminal width in character columns
  get rows(): number;     // terminal height in rows
```

These are read-only getters. The TUI queries them on every render pass because the window size can change at any time (hence `onResize`).

### Cursor and screen control

```ts
  // Move the cursor up (negative) or down (positive) by N lines.
  moveBy(lines: number): void;

  hideCursor(): void;   // suppress the blinking cursor during rendering
  showCursor(): void;   // restore it after rendering is complete

  clearLine(): void;          // erase the current line to the right
  clearFromCursor(): void;    // erase from the cursor to end of screen
  clearScreen(): void;        // erase the entire screen and reset cursor to (0,0)
```

These let the render engine reposition and erase efficiently between frames without the TUI knowing which escape sequences accomplish it.

### Extra capabilities

```ts
  // Set the terminal window title (OSC 0 sequence).
  setTitle(title: string): void;

  // Show or hide a progress indicator in the terminal chrome.
  //   active: true  — indeterminate spinner (OSC 9;4;3)
  //   active: false — clear the indicator (OSC 9;4;0)
  setProgress(active: boolean): void;

  // Whether the Kitty keyboard protocol is currently active.
  get kittyProtocolActive(): boolean;
```

The `kittyProtocolActive` getter tells the rest of the system whether Kitty's richer key encoding is in effect. We will revisit this once we look at `ProcessTerminal` in detail.

Here is the complete interface gathered in one place for reference:

```ts
// Simplified view — real signatures match what is shown above.
export interface Terminal {
  start(onInput: (data: string) => void, onResize: () => void): void;
  stop(): void;
  drainInput(maxMs?: number, idleMs?: number): Promise<void>;
  write(data: string): void;
  get columns(): number;
  get rows(): number;
  get kittyProtocolActive(): boolean;
  moveBy(lines: number): void;
  hideCursor(): void;
  showCursor(): void;
  clearLine(): void;
  clearFromCursor(): void;
  clearScreen(): void;
  setTitle(title: string): void;
  setProgress(active: boolean): void;
}
```

Because the TUI only ever holds a `Terminal` reference — not a `ProcessTerminal` — you can drop in a test implementation without touching the render engine. That is the point of the abstraction.

## `ProcessTerminal`: the real implementation

`ProcessTerminal` implements `Terminal` using `process.stdin` and `process.stdout`. Let's trace what happens when `start()` is called, step by step.

### Entering raw mode

Normally a terminal buffers input until the user presses Enter and echoes every character back. For a TUI we need neither: we want each keystroke immediately, and we do not want the terminal printing the keys the user types. That is what raw mode does.

```ts
start(onInput: (data: string) => void, onResize: () => void): void {
  this.inputHandler = onInput;
  this.resizeHandler = onResize;

  // Remember whether stdin was already in raw mode so we can restore it on stop().
  this.wasRaw = process.stdin.isRaw || false;
  if (process.stdin.setRawMode) {
    process.stdin.setRawMode(true);   // each keystroke delivered immediately
  }
  process.stdin.setEncoding("utf8");
  process.stdin.resume();

  // Bracketed paste mode: the terminal wraps paste content in
  // \x1b[200~...\x1b[201~ so we can distinguish it from typed input.
  process.stdout.write("\x1b[?2004h");

  // Listen for resize events on stdout (where the terminal reports dimensions).
  process.stdout.on("resize", this.resizeHandler);

  // On non-Windows, send SIGWINCH to ourselves so we refresh the dimensions
  // immediately — they may be stale after a process suspend/resume.
  if (process.platform !== "win32") {
    process.kill(process.pid, "SIGWINCH");
  }

  // Windows: enable VT input so modified keys (e.g. Shift+Tab → \x1b[Z)
  // arrive as VT sequences rather than losing their modifier state.
  this.enableWindowsVTInput();

  // Query the terminal for Kitty keyboard protocol support.
  this.queryAndEnableKittyProtocol();
}
```

Two things to notice: (1) `wasRaw` records the prior state so `stop()` can restore it exactly — the TUI should not leave the terminal in raw mode after it exits. (2) `process.stdout` is where resize events come from, not `process.stdin` — `stdout` is the stream connected to the terminal window.

### Dimensions and their fallbacks

```ts
get columns(): number {
  return process.stdout.columns || Number(process.env.COLUMNS) || 80;
}

get rows(): number {
  return process.stdout.rows || Number(process.env.LINES) || 24;
}
```

`process.stdout.columns` and `.rows` are set by Node.js from the terminal, but they can be zero in some environments (pipes, CI). The fallbacks check the `COLUMNS` and `LINES` environment variables, then default to the classic 80×24.

### Stopping cleanly

`stop()` reverses everything `start()` did: disables bracketed paste, tears down the Kitty protocol, destroys the `StdinBuffer`, removes all event listeners, pauses stdin, and restores the original raw mode state. Pausing stdin is important — without it, buffered input (such as a queued Ctrl+D) could be re-read after raw mode is disabled and accidentally close the parent shell over an SSH connection.

### The `write` method and the debug log

```ts
write(data: string): void {
  process.stdout.write(data);
  if (this.writeLogPath) {
    fs.appendFileSync(this.writeLogPath, data, { encoding: "utf8" });
  }
}
```

If the environment variable `XZY_TUI_WRITE_LOG` is set, every byte written to the terminal is also appended to a log file — useful for replaying or debugging render output offline. If the variable points to a directory, the implementation auto-generates a timestamped filename under that directory.

### Output helpers

Each cursor and screen control method writes the appropriate ANSI escape sequence directly:

| Method | Escape sequence sent |
|---|---|
| `moveBy(+N)` | `\x1b[{N}B` — cursor down N lines |
| `moveBy(-N)` | `\x1b[{N}A` — cursor up N lines |
| `hideCursor()` | `\x1b[?25l` |
| `showCursor()` | `\x1b[?25h` |
| `clearLine()` | `\x1b[K` — erase to end of line |
| `clearFromCursor()` | `\x1b[J` — erase to end of screen |
| `clearScreen()` | `\x1b[2J\x1b[H` — erase screen, move to (1,1) |
| `setTitle(t)` | `\x1b]0;{t}\x07` — OSC 0 window title |
| `setProgress(true)` | `\x1b]9;4;3\x07` — OSC 9;4;3 indeterminate |
| `setProgress(false)` | `\x1b]9;4;0;\x07` — OSC 9;4;0 clear |

`setProgress(true)` also starts a keepalive interval that re-sends the progress sequence every 1000 ms, because some terminal emulators time out progress indicators.

## The `StdinBuffer`: from raw bytes to discrete sequences

Here is a problem with reading from `process.stdin` in raw mode: the operating system can batch multiple keystrokes into a single `data` event. If a user types quickly, you might receive `"ab"` or `"\x1b[A\x1b[B"` (two arrow keys) in one chunk. The `matchesKey` function and key-release detection both assume each event is a single, complete escape sequence.

`ProcessTerminal` solves this with a `StdinBuffer` (constructed with a 10 ms inter-sequence timeout) that parses the incoming byte stream and emits one `"data"` event per discrete sequence. Paste content gets its own `"paste"` event; `ProcessTerminal` re-wraps it in the bracketed paste markers (`\x1b[200~...\x1b[201~`) before forwarding to the input handler, so downstream code sees a consistent format regardless of how the bytes arrived.

You do not need to instantiate `StdinBuffer` directly — `ProcessTerminal.start()` creates it and wires it up automatically.

## The Kitty keyboard protocol: richer key events

Now we have raw bytes flowing into `onInput`, one sequence at a time. There is still a problem: the traditional VT100 encoding is ambiguous. `\x1b[A` means "arrow up", but there is no way to tell whether the Shift key was held, and you cannot distinguish a key press from a key release. That makes it impossible to implement shortcuts like Shift+Enter reliably on plain terminals.

The [Kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) is a modern extension that addresses this. When the terminal supports it, each key event carries modifier state, key type (press / repeat / release), and an alternate encoding of the key that survives modifier combinations unambiguously.

### Negotiation

`ProcessTerminal` negotiates Kitty support at startup:

1. It writes the query sequence `\x1b[>7u\x1b[?u\x1b[c` immediately after entering raw mode. The `7` is the bitfield of requested flags (1 + 2 + 4 — see the table below). The trailing `\x1b[c` is a Device Attributes query that all terminals answer; it acts as a sentinel — if the terminal does not know Kitty, the DA response tells us.
2. It watches incoming bytes during a 150 ms window (`KITTY_KEYBOARD_PROTOCOL_FALLBACK_TIMEOUT_MS`). A response matching `\x1b[?{N}u` means Kitty is active and the terminal is using `N` as the flag set. Any other response (or timeout) means Kitty is not available.
3. If Kitty is not available, `ProcessTerminal` falls back to `modifyOtherKeys` mode 2 (`\x1b[>4;2m`), which at least reports some modifier combinations.

The three requested Kitty flags are:

| Flag value | Meaning |
|---|---|
| 1 | Disambiguate escape codes |
| 2 | Report event types (press / repeat / release) |
| 4 | Report alternate keys (shifted key, base layout key) |

The combined request is `1 + 2 + 4 = 7`, which is what you see in `DESIRED_KITTY_KEYBOARD_PROTOCOL_FLAGS = 7`.

On exit, `ProcessTerminal` disables the protocol by writing `\x1b[<u` — the Kitty "pop" sequence — before draining or stopping.

### Apple Terminal special case

Apple's built-in Terminal.app does not support Kitty, and its Shift+Enter sends a plain `\r`. `ProcessTerminal` detects this combination (`process.platform === "darwin"` and `TERM_PROGRAM === "Apple_Terminal"`) and normalises the Shift+Enter sequence to `\x1b[13;2u` — the same encoding Kitty would produce — so the rest of the system sees consistent input.

## A virtual terminal for tests

Because everything talks through the `Terminal` interface, writing a test double is straightforward. You implement the twelve or so methods, recording calls to `write()` and feeding synthetic bytes through `onInput`. The TUI never knows it is not connected to a real terminal:

```ts
// Simplified example of a VirtualTerminal for use in tests.
// Not from the assigned sources — illustrates the abstraction contract.
class VirtualTerminal implements Terminal {
  readonly output: string[] = [];
  private _columns = 80;
  private _rows = 24;
  private inputHandler?: (data: string) => void;
  private resizeHandler?: () => void;

  start(onInput: (data: string) => void, onResize: () => void): void {
    this.inputHandler = onInput;
    this.resizeHandler = onResize;
  }
  stop(): void {}
  async drainInput(): Promise<void> {}
  write(data: string): void { this.output.push(data); }
  get columns(): number { return this._columns; }
  get rows(): number { return this._rows; }
  get kittyProtocolActive(): boolean { return false; }
  moveBy(_lines: number): void {}
  hideCursor(): void {}
  showCursor(): void {}
  clearLine(): void {}
  clearFromCursor(): void {}
  clearScreen(): void {}
  setTitle(_title: string): void {}
  setProgress(_active: boolean): void {}

  // Test helpers
  pressKey(sequence: string): void { this.inputHandler?.(sequence); }
  resize(cols: number, rows: number): void {
    this._columns = cols;
    this._rows = rows;
    this.resizeHandler?.();
  }
}
```

With this in hand, a test can drive the TUI entirely from strings and inspect the output without touching a real terminal.

## Wiring a `ProcessTerminal` to the TUI

In production code, the wire-up is two lines:

```ts
import { TUI } from "tui";
import { ProcessTerminal } from "tui";

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

// The TUI calls terminal.start() internally when you call tui.start().
tui.start();

// ... render loop runs ...

tui.stop();
// terminal.stop() is called internally; raw mode is restored.
```

The `TUI` constructor accepts any `Terminal`, so swapping in a `VirtualTerminal` for tests requires no other changes.

## From raw bytes to named actions: the keybindings layer

We now have typed input arriving as raw byte strings through `onInput`. The next problem is turning those bytes into something a component can reason about. The character `"\r"` means Enter; `"\x1b[A"` means arrow-up; `"\x1b[27;5;13u"` means Ctrl+Enter in Kitty encoding. No component wants to do that translation itself.

The keybindings layer (`keybindings.ts`) introduces two abstractions: a `KeyId` string type that names a key combination (like `"ctrl+enter"` or `"shift+tab"`), and a `KeybindingsManager` that maps named actions to one or more `KeyId` values.

### `KeyId` and `matchesKey`

`KeyId` is a string type defined in `keys.ts` (a companion file). Each `KeyId` names a key combination in a human-readable form: `"enter"`, `"ctrl+c"`, `"shift+tab"`, `"alt+left"`, and so on.

The function `matchesKey(data: string, key: KeyId): boolean` takes a raw byte sequence and a `KeyId` and returns `true` if they match — accounting for both plain VT100 encoding and Kitty encoding. <!-- GAP: exact signature and matching logic of matchesKey and the KeyId string union/type are defined in keys.ts (not in assigned sources S39/S41); the above description is inferred from usage in keybindings.ts and terminal.ts. -->

### The `Keybindings` interface and declaration merging

The global action namespace lives in the `Keybindings` interface:

```ts
export interface Keybindings {
  "tui.editor.cursorUp": true;
  "tui.editor.cursorDown": true;
  "tui.editor.cursorLeft": true;
  "tui.editor.cursorRight": true;
  // ... (all built-in actions follow the same pattern)
  "tui.input.submit": true;
  "tui.input.newLine": true;
  "tui.input.tab": true;
  "tui.select.confirm": true;
  "tui.select.cancel": true;
}

// The union of all registered action names:
export type Keybinding = keyof Keybindings;
```

The clever part is that `Keybindings` is an interface, not a type alias. TypeScript allows interface declaration merging: a downstream package can add its own action names by reopening the interface in its own source file:

```ts
// In a downstream package, e.g. your custom component:
declare module "tui" {
  interface Keybindings {
    "myapp.sidebar.toggle": true;
    "myapp.search.open": true;
  }
}
```

After this merge, `"myapp.sidebar.toggle"` is a valid `Keybinding` everywhere — the type checker enforces it.

### `KeybindingDefinition` and `TUI_KEYBINDINGS`

Each action is described by a `KeybindingDefinition`:

```ts
export interface KeybindingDefinition {
  defaultKeys: KeyId | KeyId[];  // one or more key combinations that trigger this action
  description?: string;           // human-readable label
}
```

The built-in bindings live in the `TUI_KEYBINDINGS` constant. Here is a representative sample:

| Action | Default key(s) | Description |
|---|---|---|
| `tui.editor.cursorUp` | `"up"` | Move cursor up |
| `tui.editor.cursorDown` | `"down"` | Move cursor down |
| `tui.editor.cursorLeft` | `["left", "ctrl+b"]` | Move cursor left |
| `tui.editor.cursorRight` | `["right", "ctrl+f"]` | Move cursor right |
| `tui.editor.cursorWordLeft` | `["alt+left", "ctrl+left", "alt+b"]` | Move cursor word left |
| `tui.editor.cursorWordRight` | `["alt+right", "ctrl+right", "alt+f"]` | Move cursor word right |
| `tui.editor.cursorLineStart` | `["home", "ctrl+a"]` | Move to line start |
| `tui.editor.cursorLineEnd` | `["end", "ctrl+e"]` | Move to line end |
| `tui.editor.deleteCharBackward` | `"backspace"` | Delete character backward |
| `tui.editor.deleteCharForward` | `["delete", "ctrl+d"]` | Delete character forward |
| `tui.editor.deleteWordBackward` | `["ctrl+w", "alt+backspace"]` | Delete word backward |
| `tui.editor.deleteWordForward` | `["alt+d", "alt+delete"]` | Delete word forward |
| `tui.editor.deleteToLineStart` | `"ctrl+u"` | Delete to line start |
| `tui.editor.deleteToLineEnd` | `"ctrl+k"` | Delete to line end |
| `tui.editor.yank` | `"ctrl+y"` | Yank (paste kill ring) |
| `tui.editor.yankPop` | `"alt+y"` | Yank pop |
| `tui.editor.undo` | `"ctrl+-"` | Undo |
| `tui.editor.jumpForward` | `"ctrl+]"` | Jump forward to character |
| `tui.editor.jumpBackward` | `"ctrl+alt+]"` | Jump backward to character |
| `tui.editor.pageUp` | `"pageUp"` | Page up |
| `tui.editor.pageDown` | `"pageDown"` | Page down |
| `tui.input.submit` | `"enter"` | Submit input |
| `tui.input.newLine` | `"shift+enter"` | Insert newline |
| `tui.input.tab` | `"tab"` | Tab / autocomplete |
| `tui.input.copy` | `"ctrl+c"` | Copy selection |
| `tui.select.up` | `"up"` | Move selection up |
| `tui.select.down` | `"down"` | Move selection down |
| `tui.select.pageUp` | `"pageUp"` | Selection page up |
| `tui.select.pageDown` | `"pageDown"` | Selection page down |
| `tui.select.confirm` | `"enter"` | Confirm selection |
| `tui.select.cancel` | `["escape", "ctrl+c"]` | Cancel selection |

You might notice that some actions share the same default key. `"tui.select.up"` and `"tui.editor.cursorUp"` both default to `"up"`. That is intentional — different components activate different subsets of actions; the same keystroke routes differently depending on which component has focus.

### `KeybindingsManager`: the runtime registry

`KeybindingsManager` holds the live binding table and exposes the matching API:

```ts
export class KeybindingsManager {
  constructor(definitions: KeybindingDefinitions, userBindings: KeybindingsConfig = {}) { ... }

  // Does the raw byte sequence `data` trigger the named `keybinding`?
  matches(data: string, keybinding: Keybinding): boolean { ... }

  // Which KeyIds are currently mapped to this action (after user overrides)?
  getKeys(keybinding: Keybinding): KeyId[] { ... }

  // Return the original KeybindingDefinition for an action.
  getDefinition(keybinding: Keybinding): KeybindingDefinition { ... }

  // List any conflicts (two user-configured actions mapped to the same key).
  getConflicts(): KeybindingConflict[] { ... }

  // Replace the user overrides at runtime and rebuild the binding table.
  setUserBindings(userBindings: KeybindingsConfig): void { ... }

  // The current user overrides.
  getUserBindings(): KeybindingsConfig { ... }

  // Fully resolved binding table (default + user overrides merged).
  getResolvedBindings(): KeybindingsConfig { ... }
}
```

The `matches` method is the hot path — every incoming byte sequence is checked against one or more actions by calling `matchesKey(data, keyId)` for each resolved `KeyId`. It returns `true` as soon as one matches.

#### User overrides and conflict detection

When `userBindings` is provided, `KeybindingsManager` uses those keys instead of the defaults for each action. If a user maps two different actions to the same key, `getConflicts()` returns a `KeybindingConflict` for each such collision:

```ts
export interface KeybindingConflict {
  key: KeyId;          // the conflicting key
  keybindings: string[];  // which actions both claim it
}
```

Conflict detection only applies to user-configured bindings, not to the built-in defaults (where the same key intentionally appears on multiple actions for different contexts).

### The global singleton

```ts
export function setKeybindings(keybindings: KeybindingsManager): void { ... }

export function getKeybindings(): KeybindingsManager {
  if (!globalKeybindings) {
    globalKeybindings = new KeybindingsManager(TUI_KEYBINDINGS);
  }
  return globalKeybindings;
}
```

Most code calls `getKeybindings()` to get the global instance. If nobody has called `setKeybindings()` first, it lazy-initialises with `TUI_KEYBINDINGS` and no user overrides. To apply user-configured shortcuts at startup, create a `KeybindingsManager` with both arguments and call `setKeybindings()` before the TUI starts:

```ts
import { KeybindingsManager, TUI_KEYBINDINGS, setKeybindings } from "tui";

const userConfig: KeybindingsConfig = {
  "tui.input.submit": "ctrl+enter",  // override Enter → Ctrl+Enter
};

setKeybindings(new KeybindingsManager(TUI_KEYBINDINGS, userConfig));
```

### Matching a keybinding in a component

Here is what a component's input handler looks like end to end:

```ts
import { getKeybindings } from "tui";

function handleInput(data: string): void {
  const kb = getKeybindings();

  if (kb.matches(data, "tui.input.submit")) {
    submitForm();
    return;
  }
  if (kb.matches(data, "tui.input.newLine")) {
    insertNewline();
    return;
  }
  if (kb.matches(data, "tui.editor.deleteCharBackward")) {
    deleteCharBefore();
    return;
  }
  // Fall through: treat as printable character.
  appendCharacter(data);
}
```

The component never inspects the raw byte string directly — it only asks the `KeybindingsManager` whether the bytes match a named action. When the user remaps a key, the component picks up the change automatically through `getKeybindings()`.

### Conflict-free merging: `normalizeKeys`

Internally, `KeybindingsManager` calls a private `normalizeKeys` helper before storing each binding. It converts a single `KeyId` or an array of `KeyId` values into a deduplicated array — so if a user somehow provides `["enter", "enter"]`, only one `"enter"` entry ends up in the table.

## Putting it all together

Let's trace a single key press through the full stack from physical keyboard to named action:

1. The user presses Ctrl+Enter.
2. If Kitty is active, the terminal sends `\x1b[13;5u`. On a plain VT100 terminal it might send `\r` or a `modifyOtherKeys` variant.
3. `StdinBuffer` collects the bytes and emits the complete sequence as a single `"data"` event.
4. `ProcessTerminal.forwardInputSequence()` (handling Apple Terminal normalization if needed) calls `onInput` with the sequence.
5. The TUI delivers the string to the focused component's input handler.
6. The component calls `getKeybindings().matches(data, "tui.input.submit")`.
7. `KeybindingsManager.matches()` iterates the resolved `KeyId` list for `"tui.input.submit"` (e.g. `["enter"]`) and calls `matchesKey(data, "enter")` for each.
8. `matchesKey` recognises `\x1b[13;5u` as a Ctrl+Enter Kitty sequence — which does not match plain `"enter"` — so control falls through to the next handler.
9. The component checks `"tui.input.newLine"` (mapped to `"shift+enter"` by default) — also no match.
10. The character is treated as printable and appended to the buffer.

This trace shows why Kitty protocol matters: it lets us distinguish Enter from Ctrl+Enter from Shift+Enter, which plain VT100 cannot do. The `kittyProtocolActive` getter lets any part of the system know which encoding regime is in effect.

---

← Previous: [The TUI Class and Differential Render Engine](./the-tui-class-and-render-engine.md) · Next: [Built-In Components: Widgets, Layout, and ANSI Utilities](./built-in-components-and-layout.md) →
