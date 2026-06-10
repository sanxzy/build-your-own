---
title: "The TUI Class and Differential Render Engine"
description: "Build the TUI class from scratch: three differential render strategies, synchronized terminal output, overlay compositing, and focus management."
category: terminal-ui
type: tutorial
tags: [TUI, differential rendering, synchronized output, overlay compositing, focus, render strategies, tui package, terminal, ANSI, screen update, CSI 2026, Component, Focusable, OverlayHandle, requestRender, doRender, compositeOverlays, applyLineResets, fullRender]
keywords: [terminal UI, flicker-free rendering, ANSI escape codes, viewport, kitty image protocol, incremental update, frame comparison, hardware cursor, IME]
sources: [S36, S38, S44]
---

**TL;DR** — Redrawing the whole terminal on every frame causes flicker and is wasteful. The `tui` package solves this with a three-strategy differential render engine that compares frames and paints only what changed, wraps every write in synchronized output to prevent tearing, composites overlays on top of base content, and routes keyboard focus to the right component. By the end of this chapter you will understand how each piece works and how to mount a component, trigger a render, and display a modal overlay.

# The TUI Class and Differential Render Engine

This chapter opens the third package in our workspace — `tui` — a minimal terminal UI framework that stands completely apart from `llm-toolkit` and `agent-core`. You do not need either of those packages to use `tui`; it depends only on the terminal itself.

> **Workspace recap.** The four-package workspace is described in [Workspace and Toolchain](../getting-started/workspace-and-toolchain.md). `tui` is the third package. It knows nothing about language models or agent loops; its sole concern is rendering components to the terminal and routing keyboard input.

Let's start with the problem that motivates everything in this chapter.

---

## The problem with naïve full redraws

Imagine the simplest possible update loop: every time the UI changes, clear the screen and reprint all content from scratch. In pseudocode:

```ts
// Naïve approach — do NOT ship this
function update(lines: string[]) {
  terminal.clearScreen();
  for (const line of lines) {
    terminal.write(line + "\n");
  }
}
```

This works for a static hello-world, but it has two serious problems in a live application:

1. **Flicker.** The clear and the subsequent writes are two separate operations. The terminal repaints between them, so the user sees a blank flash on every update — even a spinner animation triggers it.
2. **Wasted work.** A typing cursor blinks every 16 ms. If your UI has 40 lines and only one of them changes, you are rewriting 40 lines 60 times a second.

The `tui` package solves both problems. It tracks the previous frame and computes the smallest possible diff, and it wraps every write in a synchronized-output protocol that makes the terminal commit the full update atomically.

---

## What a renderable component looks like

Before we build the engine, we need to know what it renders. The `Component` interface is the contract every renderable object must satisfy:

```ts
// From tui.ts (simplified view — shows the full interface)
export interface Component {
  /**
   * Render the component to lines for the given viewport width.
   * Each returned string is one line. No line may exceed `width` columns.
   */
  render(width: number): string[];

  /** Called when the component has focus and a key is pressed. */
  handleInput?(data: string): void;

  /**
   * Set to true if the component wants raw key-release events
   * (Kitty keyboard protocol). Defaults to false.
   */
  wantsKeyRelease?: boolean;

  /** Invalidate cached render state. Called on theme changes or forced re-render. */
  invalidate(): void;
}
```

Three things to notice:

- `render(width)` returns **an array of strings, one per line**. The engine concatenates these arrays from all components to form the frame.
- Each line **must not exceed `width` columns** (measured in visible characters, ignoring ANSI escape codes). The engine will throw if a line is too wide, writing a crash log before it does.
- `handleInput` is optional. A component that only displays content and never receives keyboard input can omit it.

The smallest valid component looks like this:

```ts
import type { Component } from "tui";

class StatusLine implements Component {
  private message = "Ready";

  render(_width: number): string[] {
    return [this.message];
  }

  invalidate(): void {
    // No cache to clear in this simple component
  }

  setMessage(msg: string) {
    this.message = msg;
  }
}
```

Now let's build the class that manages and renders a tree of these components.

---

## The TUI class: structure and startup

`TUI` extends `Container`, which is itself a `Component` that holds child components and concatenates their output:

```ts
// From tui.ts (simplified view — essential structure only)
export class Container implements Component {
  children: Component[] = [];

  addChild(component: Component): void {
    this.children.push(component);
  }

  removeChild(component: Component): void {
    const index = this.children.indexOf(component);
    if (index !== -1) this.children.splice(index, 1);
  }

  render(width: number): string[] {
    const lines: string[] = [];
    for (const child of this.children) {
      lines.push(...child.render(width));
    }
    return lines;
  }

  invalidate(): void {
    for (const child of this.children) {
      child.invalidate?.();
    }
  }
}
```

`TUI` adds the render engine on top of this. Its constructor takes a `Terminal` — an interface (described in the next chapter) that abstracts raw stdin/stdout — and an optional flag to show the hardware cursor:

```ts
// From tui.ts
export class TUI extends Container {
  public terminal: Terminal;
  private previousLines: string[] = [];   // last painted frame
  private previousWidth = 0;
  private previousHeight = 0;
  private focusedComponent: Component | null = null;
  private renderRequested = false;
  private renderTimer: NodeJS.Timeout | undefined;
  private lastRenderAt = 0;
  private static readonly MIN_RENDER_INTERVAL_MS = 16;
  // ... (overlay state, cursor tracking — covered later)

  constructor(terminal: Terminal, showHardwareCursor?: boolean) {
    super();
    this.terminal = terminal;
    if (showHardwareCursor !== undefined) {
      this.showHardwareCursor = showHardwareCursor;
    }
  }
}
```

Calling `tui.start()` puts the terminal into raw mode, hides the cursor, queries terminal capabilities for image support, and schedules the first render:

```ts
// From tui.ts
start(): void {
  this.stopped = false;
  this.terminal.start(
    (data) => this.handleInput(data),
    () => this.requestRender(),       // called on terminal resize
  );
  this.terminal.hideCursor();
  this.queryCellSize();
  this.requestRender();
}
```

With that in place, here is the smallest complete program that mounts a component and starts the engine:

```ts
import { TUI, ProcessTerminal, matchesKey } from "tui";
import type { Component } from "tui";

// A minimal display-only component
class HelloComponent implements Component {
  render(_width: number): string[] {
    return ["Hello from the TUI!"];
  }
  invalidate(): void {}
}

const terminal = new ProcessTerminal(); // wraps process.stdin / process.stdout
const tui = new TUI(terminal);

tui.addChild(new HelloComponent());

// Ctrl+C exits (raw mode suppresses the default SIGINT)
tui.addInputListener((data) => {
  if (matchesKey(data, "ctrl+c")) {
    tui.stop();
    process.exit(0);
  }
});

tui.start();
```

We have a running TUI. Now let's understand what happens every time a frame needs to paint.

---

## Requesting a render: the rate-limited scheduler

Components never call `doRender()` directly. They call `requestRender()`, which coalesces multiple simultaneous requests into one deferred paint:

```ts
// From tui.ts (simplified view — core scheduling logic)
requestRender(force = false): void {
  if (force) {
    // Immediately clear the previous-frame state and schedule on nextTick
    this.previousLines = [];
    this.previousWidth = -1;   // -1 will cause widthChanged to be true
    this.previousHeight = -1;
    // ... reset cursor tracking
    this.renderRequested = true;
    process.nextTick(() => {
      if (!this.renderRequested) return;
      this.renderRequested = false;
      this.doRender();
    });
    return;
  }

  if (this.renderRequested) return;  // already pending — no-op
  this.renderRequested = true;
  process.nextTick(() => this.scheduleRender());
}

private scheduleRender(): void {
  if (this.stopped || this.renderTimer || !this.renderRequested) return;

  const elapsed = performance.now() - this.lastRenderAt;
  const delay = Math.max(0, TUI.MIN_RENDER_INTERVAL_MS - elapsed);

  this.renderTimer = setTimeout(() => {
    this.renderTimer = undefined;
    if (!this.renderRequested) return;
    this.renderRequested = false;
    this.lastRenderAt = performance.now();
    this.doRender();
    if (this.renderRequested) this.scheduleRender(); // re-schedule if more work arrived
  }, delay);
}
```

`MIN_RENDER_INTERVAL_MS` is 16 ms — roughly one 60 Hz frame. If `doRender()` finishes and `renderRequested` is already true again (a component updated while we were painting), the scheduler re-arms immediately. This gives you a natural frame budget without any additional coordination.

Now let's look at the heart of the system: the three rendering strategies inside `doRender()`.

---

## The three differential render strategies

Every call to `doRender()` begins with the same three steps:

1. Call `this.render(width)` to collect the new frame from all components.
2. Composite any overlays on top (covered below).
3. Compare the new frame to `this.previousLines` and decide which of three strategies to use.

Here is that decision tree, faithful to the source:

| Strategy | When it triggers | What happens |
|---|---|---|
| **First render** | `previousLines` is empty and no width/height change | Output all lines without clearing scrollback. Assumes a clean screen. |
| **Full re-render** | Width changed, height changed (non-Termux), content shrinks below working area (with `clearOnShrink`), or a changed line is above the previous viewport | `\x1b[2J\x1b[H\x1b[3J` clears screen + scrollback; all lines rewritten from scratch. |
| **Differential update** | Everything else (normal frame-to-frame updates) | Find `firstChanged` and `lastChanged` by line comparison. Move cursor to `firstChanged`; clear and rewrite only the lines in that range. |

Let's walk through each one.

### Strategy 1: First render

```ts
// From tui.ts (within doRender)
if (this.previousLines.length === 0 && !widthChanged && !heightChanged) {
  fullRender(false);  // false = do NOT clear before rendering
  return;
}
```

On the very first frame, `previousLines` is empty. The engine assumes the terminal is already clear (you just launched), so it writes all lines sequentially without sending a clear-screen sequence. This preserves any existing content above the cursor (e.g., shell history) rather than erasing it.

### Strategy 2: Full re-render

Several conditions trigger a full repaint:

```ts
// From tui.ts (within doRender)

// Width changes always need a full re-render because text wrapping changes
if (widthChanged) {
  fullRender(true);  // true = clear first
  return;
}

// Height changes need a full re-render to keep the viewport aligned
// Exception: Termux, where the keyboard show/hide constantly changes height
if (heightChanged && !isTermuxSession()) {
  fullRender(true);
  return;
}

// Content shrank below the working area (configurable, default off)
if (this.clearOnShrink && newLines.length < this.maxLinesRendered
    && this.overlayStack.length === 0) {
  fullRender(true);
  return;
}
```

The `fullRender(clear: boolean)` helper wraps every write in synchronized output:

```ts
// From tui.ts (simplified view)
const fullRender = (clear: boolean): void => {
  this.fullRedrawCount += 1;
  let buffer = "\x1b[?2026h";           // BEGIN synchronized output (CSI 2026)
  if (clear) {
    buffer += deleteKittyImages(...);    // remove any inline images first
    buffer += "\x1b[2J\x1b[H\x1b[3J";  // clear screen, home cursor, clear scrollback
  }
  for (let i = 0; i < newLines.length; i++) {
    if (i > 0) buffer += "\r\n";
    buffer += newLines[i];
  }
  buffer += "\x1b[?2026l";              // END synchronized output
  this.terminal.write(buffer);
  // ... update bookkeeping (previousLines, cursor positions, etc.)
};
```

The `\x1b[?2026h` / `\x1b[?2026l` pair is the CSI 2026 synchronized output mode. The terminal buffers everything between the two sequences and paints them in a single atomic update — the mechanism that prevents flicker. We will look at this more closely in the next section.

The test suite verifies all three full-redraw triggers:

```ts
// From tui-render.test.ts — width change triggers full re-render
it("triggers full re-render when terminal width changes", async () => {
  const terminal = new VirtualTerminal(40, 10);
  const tui = new TUI(terminal);
  // ...
  const initialRedraws = tui.fullRedraws;
  terminal.resize(60, 10);
  await terminal.waitForRender();
  assert.ok(tui.fullRedraws > initialRedraws, "Width change should trigger full redraw");
});
```

### Strategy 3: Differential update

When neither of the above conditions applies, the engine computes exactly which lines changed:

```ts
// From tui.ts (simplified view — the differential path)
let firstChanged = -1;
let lastChanged  = -1;
const maxLines = Math.max(newLines.length, this.previousLines.length);

for (let i = 0; i < maxLines; i++) {
  const oldLine = i < this.previousLines.length ? this.previousLines[i] : "";
  const newLine = i < newLines.length          ? newLines[i]           : "";
  if (oldLine !== newLine) {
    if (firstChanged === -1) firstChanged = i;
    lastChanged = i;
  }
}
```

Once we have the range `[firstChanged, lastChanged]`, the engine:

1. Wraps the entire update in `\x1b[?2026h` … `\x1b[?2026l`.
2. Moves the hardware cursor to `firstChanged`.
3. For each line in the range, emits `\x1b[2K` (erase line) then the new content.
4. If the new frame is shorter than the old one, clears the extra trailing lines.

Notice that unchanged lines between `firstChanged` and `lastChanged` are still rewritten — the engine renders from `firstChanged` to `lastChanged`, not just the lines that changed. This is a deliberate tradeoff: the range is usually short, and skipping over unchanged lines mid-range would add expensive cursor movement for little benefit.

The test suite has clear behavioral coverage:

```ts
// From tui-render.test.ts — spinner animation: only middle line changes
it("renders correctly when only a middle line changes (spinner case)", async () => {
  // ...
  component.lines = ["Header", "Working...", "Footer"];
  tui.start();
  await terminal.waitForRender();

  for (const frame of ["|", "/", "-", "\\"]) {
    component.lines = ["Header", `Working ${frame}`, "Footer"];
    tui.requestRender();
    await terminal.waitForRender();

    assert.ok(viewport[0]?.includes("Header"),         "Header preserved");
    assert.ok(viewport[1]?.includes(`Working ${frame}`), "Spinner updated");
    assert.ok(viewport[2]?.includes("Footer"),         "Footer preserved");
  }
});
```

Only line 1 changes each frame, and the engine updates only that line without touching the header or footer.

---

## Synchronized output: eliminating flicker with CSI 2026

You have seen `\x1b[?2026h` and `\x1b[?2026l` on every write path. Let's be explicit about what they do.

CSI 2026 is a terminal mode called **synchronized output**. When the terminal sees `\x1b[?2026h`, it begins buffering all subsequent screen updates internally. Nothing repaints. When it sees `\x1b[?2026l`, it flushes the buffer and repaints the screen exactly once with the complete final state.

Without synchronized output:

1. Engine writes `\x1b[2K` (erase line) → terminal paints blank.
2. Engine writes new content → terminal paints content.
3. User sees the blank flash between steps 1 and 2.

With synchronized output, steps 1 and 2 are both buffered and committed together. The user never sees the intermediate blank state.

The engine also applies **per-line style resets** before writing, so ANSI color or style codes from one line cannot bleed into the next:

```ts
// From tui.ts
private static readonly SEGMENT_RESET = "\x1b[0m\x1b]8;;\x07";

private applyLineResets(lines: string[]): string[] {
  const reset = TUI.SEGMENT_RESET;
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    if (!isImageLine(line)) {
      lines[i] = normalizeTerminalOutput(line) + reset;
    }
  }
  return lines;
}
```

The reset is `\x1b[0m` (SGR reset — clears all colors and attributes) followed by `\x1b]8;;\x07` (OSC 8 reset — clears any active hyperlink). The test suite verifies style isolation:

```ts
// From tui-render.test.ts
it("resets styles after each rendered line", async () => {
  component.lines = ["\x1b[3mItalic", "Plain"];
  tui.start();
  await terminal.waitForRender();
  // getCellItalic checks the xterm buffer directly
  assert.strictEqual(getCellItalic(terminal, 1, 0), 0);
  // "Plain" must not inherit italic from the line above
});
```

---

## Overlay compositing: rendering components on top

So far we have rendered a flat list of components. But UIs need dialogs, menus, and pop-ups that float above the main content — overlays. The `tui` package handles this with an **overlay stack** that is composited into the frame before it is compared against `previousLines`.

### Showing an overlay

```ts
// From tui.ts API (mirroring the README)
const handle = tui.showOverlay(myDialogComponent, {
  anchor: "center",   // position relative to terminal
  width: 60,          // columns
  maxHeight: 20,      // rows
});
```

`showOverlay` pushes an entry onto `overlayStack`, records the current focused component as `preFocus` (so focus can be restored when the overlay closes), and calls `setFocus(component)` unless `nonCapturing: true` is set. It returns an `OverlayHandle`:

```ts
// From tui.ts — OverlayHandle interface
export interface OverlayHandle {
  hide(): void;                            // permanently remove the overlay
  setHidden(hidden: boolean): void;        // temporarily hide/show
  isHidden(): boolean;
  focus(): void;                           // bring to front and focus
  unfocus(options?: OverlayUnfocusOptions): void;  // release focus
  isFocused(): boolean;
}
```

### How compositing works

After `this.render(width)` collects the base frame lines and before the differential compare, the engine calls `compositeOverlays()`:

```ts
// From tui.ts (simplified view)
private compositeOverlays(lines: string[], termWidth: number, termHeight: number): string[] {
  if (this.overlayStack.length === 0) return lines;
  const result = [...lines];

  // Sort visible overlays by focusOrder so higher-focus overlays paint on top
  const visibleEntries = this.overlayStack
    .filter((e) => this.isOverlayVisible(e))
    .sort((a, b) => a.focusOrder - b.focusOrder);

  // Render each overlay, resolve its position, then splice it into the frame
  for (const entry of visibleEntries) {
    const { width, maxHeight } = this.resolveOverlayLayout(entry.options, 0, termWidth, termHeight);
    let overlayLines = entry.component.render(width);
    if (maxHeight !== undefined && overlayLines.length > maxHeight) {
      overlayLines = overlayLines.slice(0, maxHeight);
    }
    const { row, col } = this.resolveOverlayLayout(entry.options, overlayLines.length, termWidth, termHeight);
    for (let i = 0; i < overlayLines.length; i++) {
      result[row + i] = this.compositeLineAt(result[row + i], overlayLines[i], col, width, termWidth);
    }
  }
  return result;
}
```

The key operation is `compositeLineAt()`, which splices one overlay line into one base line at a given column. It preserves the base content to the left of the overlay, inserts the overlay content (padded to `overlayWidth`), and restores the base content to the right — all in a single pass with ANSI-aware width tracking.

### Overlay positioning options

`OverlayOptions` gives you three positioning systems, resolved in priority order:

| Priority | Mechanism | Example |
|---|---|---|
| 1 (highest) | Absolute `row` / `col` | `{ row: 5, col: 10 }` |
| 2 | Percentage `row` / `col` | `{ row: "25%", col: "50%" }` |
| 3 (default) | `anchor` string | `{ anchor: "bottom-right" }` |

After position is resolved, `margin` clamps the result to stay within terminal bounds. `minWidth` applies as a floor after the width calculation.

Valid anchor values: `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`, `top-center`, `bottom-center`, `left-center`, `right-center`.

The `visible` callback is evaluated every frame — useful for hiding an overlay when the terminal is too narrow:

```ts
const handle = tui.showOverlay(sidebar, {
  anchor: "right-center",
  width: 30,
  visible: (termWidth, _termHeight) => termWidth >= 100,
});
```

### Hiding and removing overlays

```ts
handle.hide();          // permanently removes from stack; focus returns to preFocus
handle.setHidden(true); // temporarily invisible; can be shown again with setHidden(false)
tui.hideOverlay();      // hides the topmost overlay
tui.hasOverlay();       // true if any visible overlay exists
```

---

## Focus management

Focus determines which component receives keyboard input. The engine maintains `focusedComponent: Component | null` and routes every keypress there:

```ts
// From tui.ts (within handleInput — simplified)
if (this.focusedComponent?.handleInput) {
  // Filter key-release events unless the component opts in
  if (isKeyRelease(data) && !this.focusedComponent.wantsKeyRelease) return;
  this.focusedComponent.handleInput(data);
  this.requestRender();
}
```

`tui.setFocus(component)` sets the focused component. When the component implements the `Focusable` interface, the engine also sets `component.focused = true` (and `false` on the previous holder), enabling it to show a cursor marker:

```ts
// From tui.ts
export interface Focusable {
  /** Set by TUI when focus changes. */
  focused: boolean;
}
```

Components that display a text cursor should implement `Focusable` and emit `CURSOR_MARKER` — a zero-width APC escape sequence — at the cursor position in their rendered output. The engine finds the marker, strips it, and moves the hardware cursor there (useful for IME candidate window positioning).

Focus and overlays interact through a **focus restore** mechanism: when `showOverlay()` captures focus, it saves the previous `focusedComponent` as `preFocus`. When the overlay hides or calls `handle.unfocus()`, focus returns to `preFocus` (or to the topmost remaining capturing overlay).

A practical example:

```ts
import { TUI, ProcessTerminal, matchesKey } from "tui";
import type { Component } from "tui";

class ConfirmDialog implements Component {
  onYes?: () => void;
  onNo?: () => void;

  render(_width: number): string[] {
    return [
      "┌─────────────────────────┐",
      "│  Delete file? [y/n]     │",
      "└─────────────────────────┘",
    ];
  }

  handleInput(data: string): void {
    if (matchesKey(data, "y")) this.onYes?.();
    if (matchesKey(data, "n") || matchesKey(data, "escape")) this.onNo?.();
  }

  invalidate(): void {}
}

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);
// ... add base components, set initial focus

const dialog = new ConfirmDialog();
const handle = tui.showOverlay(dialog, { anchor: "center", width: 28 });

dialog.onYes = () => {
  // perform deletion
  handle.hide();                 // removes overlay, focus returns to base component
};
dialog.onNo = () => handle.hide();

tui.start();
```

---

## Kitty image tracking

The engine includes a side-channel for inline images rendered via the Kitty graphics protocol. Lines containing a Kitty sequence are detected by `isImageLine()`. The engine tracks which image IDs were present in the previous frame via `previousKittyImageIds`, and deletes stale placements before redrawing:

```ts
// From tui-render.test.ts — image cleanup correctness
it("deletes changed image ids before drawing moved placements", async () => {
  // ...
  const deleteIndex = writes.indexOf(deleteKittyImage(42));
  const drawIndex   = writes.indexOf(newImage);
  assert.ok(deleteIndex >= 0, "changed old image should be deleted");
  assert.ok(deleteIndex < drawIndex, "old image must be deleted before the new placement is drawn");
});
```

The ordering guarantee — delete before draw — is essential because Kitty image placement IDs are reused: if you draw before deleting, the old placement may flicker in briefly before the new one takes its position.

---

## Putting it all together: a complete example

Here is a self-contained program that shows base content plus a dismissible overlay:

```ts
import { TUI, ProcessTerminal, matchesKey } from "tui";
import type { Component } from "tui";
import type { OverlayHandle } from "tui";

// Simple status display
class StatusBar implements Component {
  private text: string;
  constructor(text: string) { this.text = text; }

  render(width: number): string[] {
    const line = `[ ${this.text} ]`.padEnd(width);
    return [line];
  }
  invalidate(): void {}
}

// Overlay: shows a banner, press Escape to dismiss
class Banner implements Component {
  render(_width: number): string[] {
    return [
      "┌──────────────────────┐",
      "│  Press Esc to close  │",
      "└──────────────────────┘",
    ];
  }
  handleInput(data: string): void { /* handled externally */ }
  invalidate(): void {}
}

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

// Mount base component
const status = new StatusBar("Running");
tui.addChild(status);

// Show overlay immediately
const banner = new Banner();
let handle: OverlayHandle = tui.showOverlay(banner, {
  anchor: "center",
  width: 26,
});

tui.addInputListener((data) => {
  if (matchesKey(data, "escape")) {
    handle.hide();
  }
  if (matchesKey(data, "ctrl+c")) {
    tui.stop();
    process.exit(0);
  }
});

tui.start();
```

When you press Escape, `handle.hide()` removes the overlay from the stack and triggers a re-render. Focus returns to wherever it was before `showOverlay()` captured it.

---

## Debug and configuration knobs

The engine reads several environment variables at startup:

| Variable | Effect |
|---|---|
| `XZY_HARDWARE_CURSOR=1` | Show the hardware terminal cursor (hidden by default). Also settable via `tui.setShowHardwareCursor(true)`. |
| `XZY_CLEAR_ON_SHRINK=1` | Full re-render when content shrinks below the working area (default: off). Also settable via `tui.setClearOnShrink(true)`. |
| `XZY_TUI_WRITE_LOG=<path>` | Capture the raw ANSI byte stream written to stdout for debugging. |

<!-- GAP: The source uses `PI_HARDWARE_CURSOR`, `PI_CLEAR_ON_SHRINK`, and `PI_TUI_WRITE_LOG` as the environment variable names (S38 lines 283-284, S36 line 788). The brand-agnostic replacements shown above (`XZY_*`) follow the WRITER-BRIEF substitution table. -->

You can also call `tui.requestRender(true)` (force mode) to trigger an unconditional full redraw from scratch — useful after a theme change that invalidates all component caches.

---

## What comes next

We have built the core render engine. The next chapter covers the `Terminal` abstraction — the `ProcessTerminal` and `VirtualTerminal` implementations, how raw mode and resize events work, and how the engine's keybinding system routes input before it reaches components.

---

← Previous: [Cross-Provider Message Transforms and Handoff](../agent-loop/cross-provider-message-transforms.md) · Next: [Terminal Abstraction, Input Handling, and Keybindings](./terminal-abstraction-and-input.md) →
