---
title: "The TUI Class and Differential Render Engine"
description: "Build the TUI class from scratch — the differential render engine with three strategies, synchronized terminal output that prevents tearing, overlay compositing for popups, and focus management for keyboard routing."
category: terminal-ui
type: tutorial
tags: [TUI, differential rendering, synchronized output, overlay compositing, focus, render loop, ANSI, screen update, terminal, tui package]
keywords: [terminal UI, differential rendering, render tree, ANSI output, screen diffing, overlay system]
sources: [S34, S43]
---

**TL;DR** — Rendering to a terminal is fundamentally different from rendering to a browser. There's no DOM, no CSS, and every update must be written as ANSI escape sequences. We'll build a TUI engine that computes diffs between frames, only writes changed characters, composites overlays, and manages focus — giving us a flicker-free terminal UI framework.

## The terminal rendering challenge

A terminal is a fixed grid of character cells. To update the display, you write ANSI escape sequences to stdout — sequences like `\x1b[2J` (clear screen) or `\x1b[10;5H` (move cursor to row 10, column 5).

The naive approach — clear the screen and redraw everything — causes visible flicker. Each frame, the screen goes blank for a moment while the new content is written.

The solution: **differential rendering**. Instead of redrawing the whole screen, compute which cells changed between the previous frame and the current one, then only write those cells.

## The render tree

Our TUI is built on a tree of components. Each component produces a `View` — a 2D grid of styled characters:

```ts
interface View {
  width: number;
  height: number;
  cells: Cell[][];  // cells[row][col]
}

interface Cell {
  char: string;     // the character (or "" for empty)
  style: Style;     // foreground, background, bold, etc.
}
```

Components implement a `render()` method that returns a `View`:

```ts
abstract class Component {
  abstract render(): View;
  abstract desiredSize(): { width: number; height: number };
  focusable(): boolean { return false; }
  handleKey(key: Key): boolean { return false; }
}
```

The TUI class owns the root component and orchestrates the render loop.

## The differential render engine

Create `packages/tui/src/tui.ts`. The engine has three strategies:

```ts
type RenderStrategy = "full" | "diff" | "overlay";

class Tui {
  private prevFrame: Cell[][] = [];
  private root: Component;

  render(): void {
    const view = this.root.render();
    const newFrame = view.cells;

    if (this.prevFrame.length === 0) {
      this.renderFull(newFrame);  // first frame: full render
    } else {
      this.renderDiff(this.prevFrame, newFrame);  // subsequent: diff
    }

    this.renderOverlays();  // popups, autocomplete, etc.
    this.prevFrame = newFrame;
  }
}
```

### Strategy 1: Full render

Used for the first frame or after a resize. Clear the screen and write every cell:

```ts
private renderFull(frame: Cell[][]): void {
  // Hide cursor, move to home
  this.write("\x1b[?25l\x1b[H");

  for (let row = 0; row < frame.length; row++) {
    for (let col = 0; col < frame[row].length; col++) {
      const cell = frame[row][col];
      this.write(`${cell.style.toAnsi()}${cell.char || " "}`);
    }
    this.write("\r\n");
  }

  // Reset style, show cursor
  this.write("\x1b[0m\x1b[?25h");
}
```

### Strategy 2: Diff render

For subsequent frames. Compare the previous frame with the new one cell by cell, only writing differences:

```ts
private renderDiff(prev: Cell[][], next: Cell[][]): void {
  for (let row = 0; row < next.length; row++) {
    for (let col = 0; col < next[row].length; col++) {
      const prevCell = prev[row]?.[col];
      const nextCell = next[row][col];

      if (!prevCell || cellChanged(prevCell, nextCell)) {
        // Move cursor to (row, col)
        this.write(`\x1b[${row + 1};${col + 1}H`);
        // Write the new cell
        this.write(`${nextCell.style.toAnsi()}${nextCell.char || " "}`);
      }
    }
  }
  // Reset style
  this.write("\x1b[0m");
}
```

The diff check: a cell changed if its character, foreground color, background color, or any style attribute (bold, italic, underline) differs from the previous frame.

### Strategy 3: Overlay compositing

Overlays are components rendered on top of the base view — autocomplete popups, dialogs, notifications. They're composited after the base diff:

```ts
private overlays: Component[] = [];

renderOverlays(): void {
  for (const overlay of this.overlays) {
    const view = overlay.render();
    // Composite: overlay cells replace base cells where non-empty
    for (let row = 0; row < view.height; row++) {
      for (let col = 0; col < view.width; col++) {
        const cell = view.cells[row][col];
        if (cell.char !== "") {
          this.write(`\x1b[${overlayY + row + 1};${overlayX + col + 1}H`);
          this.write(`${cell.style.toAnsi()}${cell.char}`);
        }
      }
    }
  }
}
```

This is how autocomplete suggestions appear below the input line — they're an overlay composited on top of the base chat view.

## Synchronized output

Writing ANSI sequences character-by-character can cause screen tearing if the terminal redraws between our writes. The solution: buffer all ANSI output and write it in a single `process.stdout.write()` call:

```ts
private buffer = "";

private write(s: string): void {
  this.buffer += s;
}

render(): void {
  this.buffer = "";
  // ... build the frame into buffer ...
  process.stdout.write(this.buffer);
}
```

A single write is atomic from the terminal's perspective — no tearing.

## Focus management

Keyboard input is routed to the focused component:

```ts
private focused: Component | null = null;

handleInput(data: Buffer): void {
  const key = parseKey(data);
  if (!key) return;

  // Try focused component first
  if (this.focused?.handleKey(key)) return;

  // Fall back to root
  this.root.handleKey(key);
}

focus(component: Component): void {
  this.focused?.blur();
  this.focused = component;
  component.focus();
}
```

Focus moves between components with Tab or by clicking (in terminals that support mouse). The focused component gets first crack at every keypress.

## The render loop

The TUI connects to a `Terminal` abstraction (which we'll build in the next chapter) for input events:

```ts
class Tui {
  constructor(private terminal: Terminal) {
    this.terminal.onResize(() => this.render());
    this.terminal.onData((data) => this.handleInput(data));
  }

  start(): void {
    this.terminal.enterRawMode();
    this.render();
  }

  stop(): void {
    this.terminal.leaveRawMode();
  }
}
```

On every resize event, the TUI re-renders with the full strategy (since the grid dimensions changed). On every data event (keypress), it routes to the focused component and re-renders the diff.

## What we've built

We have a flicker-free terminal UI engine:

- **Render tree** of components producing `View` grids
- **Three render strategies** (full, diff, overlay) for efficient screen updates
- **Synchronized output** via single-buffer writes to prevent tearing
- **Focus management** for keyboard routing
- **Render loop** driven by terminal input and resize events

In the next chapter, we'll build the Terminal abstraction — the layer that handles raw stdin/stdout, ANSI parsing, and key codes.

---

← Previous: [Cross-Provider Message Transforms](../agent-core/cross-provider-message-transforms.md) · Next: [Terminal Abstraction, Input, and Keybindings](./terminal-abstraction-and-input.md) →
