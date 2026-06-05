---
title: "Built-In Components: Widgets, Layout, and ANSI Utilities"
description: Tour the tui widget library—text, box, input, editor, loader, select-list, markdown, image—then master the ANSI utility belt for width-correct terminal layout.
category: terminal-ui
type: tutorial
tags: [components, editor, input, markdown, loader, select-list, box, text, image, ANSI, visibleWidth, truncateToWidth, wrapTextWithAnsi, sliceByColumn, layout, tui, widgets, east Asian width, spacer, truncated-text, settings-list, cancellable-loader, terminal rendering]
keywords: [built-in widgets, terminal UI components, ANSI escape codes, grapheme width, wide characters, word wrap, text truncation, column slicing, marked, get-east-asian-width]
sources: [S37, S40, S43]
---

**TL;DR** — The `tui` package ships a complete widget library that plugs directly into the `Component` interface: layout primitives (`Text`, `Box`, `Spacer`, `TruncatedText`), interactive inputs (`Input`, `Editor`), feedback indicators (`Loader`, `CancellableLoader`), selection UIs (`SelectList`, `SettingsList`), and rich-content renderers (`Markdown`, `Image`). Underneath them all sits a set of ANSI-aware utilities—`visibleWidth`, `truncateToWidth`, `wrapTextWithAnsi`, `sliceByColumn`—that make text layout correct even when strings contain invisible escape codes or double-width East Asian characters. By the end of this chapter you will know what each widget does, when to reach for it, and how the utility belt keeps your layout from going wrong.

# Built-In Components: Widgets, Layout, and ANSI Utilities

## Where we are

In [The TUI Class and Render Engine](./the-tui-class-and-render-engine.md), we saw that the render loop calls `render(width: number): string[]` on every registered `Component`, diffs the result against the last frame, and writes only the changed lines. In [Terminal Abstraction, Input Handling, and Keybindings](./terminal-abstraction-and-input.md), we saw how raw terminal bytes become named keys and how components receive them through `handleInput(data: string)`.

Now the question is: what components do we actually have to work with, and what math do we need to get width-correct output?

Let us work through both.

---

## Part 1 — The widget library

Every component in `src/components/` implements the `Component` interface from `tui`—meaning it has at minimum a `render(width: number): string[]` method and an `invalidate(): void` method. We saw the interface in [The TUI Class and Render Engine](./the-tui-class-and-render-engine.md); the one-sentence recap: `render` produces the lines for the current frame, `invalidate` tells the component to drop its cached output so the next `render` re-computes everything.

Let's tour the widgets grouped by purpose.

### Group 1 — Layout primitives: Text, Box, Spacer, TruncatedText

These are the building blocks you compose everything else out of.

#### Text — word-wrapping paragraph block

`Text` displays a multi-line string with automatic word wrapping. It accepts a message string, horizontal padding (`paddingX`), vertical padding (`paddingY`), and an optional background colour function.

```ts
// Simplified view of Text constructor and key methods
constructor(
  text: string = "",
  paddingX: number = 1,
  paddingY: number = 1,
  customBgFn?: (text: string) => string
)

setText(text: string): void   // Update content (clears cache)
render(width: number): string[]
```

When `render` runs, `Text`:

1. Replaces tabs with three spaces.
2. Wraps the string to `width - paddingX * 2` columns using `wrapTextWithAnsi` (we cover that function in Part 2).
3. Adds the left and right horizontal margins.
4. Optionally applies a background colour function across each line, padding each line to exactly `width` columns.
5. Adds empty lines above and below for vertical padding.

Notice what this means: `Text` already handles ANSI-styled content correctly because it delegates all the measurement work to `wrapTextWithAnsi`. You can pass chalk-coloured text and the wrapping will not corrupt the escape codes.

```ts
import { Text } from "tui";

const status = new Text("Agent is running.", 1, 0);
tui.addComponent(status);

// Later, update the message without constructing a new component:
status.setText("Agent finished in 3 turns.");
tui.requestRender();
```

#### Box — padded, optionally coloured container

`Box` is a container that wraps around zero or more child `Component` instances and applies uniform padding plus an optional background colour to all of them together.

```ts
// Simplified view of Box
constructor(paddingX = 1, paddingY = 1, bgFn?: (text: string) => string)

addChild(component: Component): void
removeChild(component: Component): void
clear(): void
setBgFn(bgFn?: (text: string) => string): void
render(width: number): string[]
```

When `Box` renders, it:

1. Computes `contentWidth = width - paddingX * 2`.
2. Renders each child at `contentWidth` and concatenates their lines.
3. Prepends `paddingX` spaces on the left of every content line.
4. Applies the background function (if any) across each line and pads to full `width`.
5. Wraps the block in `paddingY` empty background-coloured lines above and below.

`Box` caches its output. It re-renders only when a child changes (calling `addChild` or `removeChild` invalidates the cache), the `width` changes, or the background function's output changes (detected by a sample comparison on each render).

A typical usage: wrap a group of components in a `Box` to give them a shared coloured background or breathing room.

```ts
import { Box, Text } from "tui";

const panel = new Box(2, 1, (s) => chalk.bgGray(s));
panel.addChild(new Text("Step 1: Reading files…", 0, 0));
panel.addChild(new Text("Step 2: Analysing…", 0, 0));
tui.addComponent(panel);
```

#### Spacer — vertical gap

`Spacer` renders a configurable number of blank lines. Nothing more.

```ts
constructor(lines: number = 1)
setLines(lines: number): void
render(_width: number): string[]   // returns `lines` empty strings
```

Use it to separate visually distinct sections of your layout without modifying surrounding components.

#### TruncatedText — single-line label that never wraps

`TruncatedText` renders exactly one visible line. If the string is longer than the available column width, it truncates with an ellipsis (delegating to `truncateToWidth`). If the string contains a newline, only the first line is shown.

```ts
constructor(text: string, paddingX: number = 0, paddingY: number = 0)
render(width: number): string[]
```

This is the right tool for status bars, menu items, file-path labels, or any context where you need guaranteed single-line output.

---

### Group 2 — Text input: Input and Editor

Both `Input` and `Editor` implement `Component` and `Focusable`. The `Focusable` interface adds a `focused: boolean` field that the TUI sets when moving keyboard focus—the components use it to decide whether to render the cursor.

#### Input — single-line text field with horizontal scrolling

`Input` is a single-line text field. It maintains `value: string` and `cursor: number` (a byte offset into the value). When the text is wider than the display area, `Input` scrolls horizontally to keep the cursor visible, using `sliceByColumn` to extract the correct window.

Key public surface:

```ts
getValue(): string
setValue(value: string): void
handleInput(data: string): void   // routes keybinding actions
render(width: number): string[]   // renders ">" prompt + scrolled content

onSubmit?: (value: string) => void
onEscape?: () => void
```

Internally, `Input` uses `getKeybindings()` to match key sequences to named actions (`tui.input.submit`, `tui.editor.cursorLeft`, `tui.editor.deleteWordBackward`, etc.) — the same system we saw in the previous chapter. It also maintains a kill ring for Emacs-style cut-and-paste and an `UndoStack` for undo/redo.

The rendered line always begins with `"> "` as a prompt. The cursor is displayed using terminal reverse-video (`\x1b[7m … \x1b[27m`) and, when focused, is preceded by a zero-width `CURSOR_MARKER` for IME positioning.

```ts
import { Input } from "tui";

const field = new Input();
field.onSubmit = (value) => {
  console.log("User submitted:", value);
  field.setValue("");
};
tui.addComponent(field);
tui.setFocus(field);
```

#### Editor — multi-line editor with word-wrap, history, and autocomplete

`Editor` is the full-featured multi-line text editor. It manages a logical `lines: string[]` array and a cursor position `{ cursorLine, cursorCol }`. When it renders, it word-wraps each logical line into visual layout lines using `wordWrapLine`, then scrolls vertically to keep the cursor within a viewport sized at 30% of the terminal height (minimum 5 lines).

```ts
constructor(tui: TUI, theme: EditorTheme, options?: EditorOptions)

// Core API
getText(): string
setText(text: string): void
getLines(): string[]
getCursor(): { line: number; col: number }
insertTextAtCursor(text: string): void

// Callbacks
onSubmit?: (text: string) => void
onChange?: (text: string) => void
disableSubmit: boolean
```

`EditorOptions` lets you set `paddingX` and `autocompleteMaxVisible` (clamped to the range 3–20, default 5). `EditorTheme` includes `borderColor` and `selectList` styling.

A few important behaviours to know:

- **Bracketed paste handling.** When the user pastes more than 10 lines or 1 000 characters, `Editor` stores the content and inserts a placeholder marker like `[paste #1 +42 lines]` into the visible text. On submit, all markers are expanded back to their original content before firing `onSubmit`.
- **Prompt history.** Call `addToHistory(text)` after a successful submission. Arrow-up/down then browses previous prompts.
- **Autocomplete hook.** Attach an `AutocompleteProvider` via `setAutocompleteProvider(provider)`. The editor auto-triggers on `/` (slash commands), `@`, and `#` symbol tokens, then renders a `SelectList` dropdown inline.
- **Scroll indicators.** When content above or below the viewport is hidden, the editor renders a border line like `─── ↑ 3 more ───` or `─── ↓ 2 more ───`.

The editor renders horizontal borders (`─` repeated to `width`) above and below the text area, styled by `theme.borderColor`.

```ts
import { Editor } from "tui";

const editor = new Editor(tui, {
  borderColor: (s) => chalk.dim(s),
  selectList: mySelectListTheme,
});
editor.onSubmit = (text) => {
  agent.sendMessage(text);
  editor.addToHistory(text);
};
tui.addComponent(editor);
tui.setFocus(editor);
```

---

### Group 3 — Feedback indicators: Loader and CancellableLoader

When the agent is working and the UI needs to show activity, reach for `Loader`.

#### Loader — animated spinner with message

`Loader` extends `Text`. It runs a `setInterval` to advance through animation frames and calls `tui.requestRender()` on each tick.

```ts
constructor(
  ui: TUI,
  spinnerColorFn: (str: string) => string,
  messageColorFn: (str: string) => string,
  message: string = "Loading...",
  indicator?: LoaderIndicatorOptions
)

start(): void    // begin animation
stop(): void     // clear the interval
setMessage(message: string): void
setIndicator(indicator?: LoaderIndicatorOptions): void
```

`LoaderIndicatorOptions` has two fields:

| Field | Purpose | Default |
|---|---|---|
| `frames` | Array of animation frame strings | `["⠋","⠙","⠹","⠸","⠼","⠴","⠦","⠧","⠇","⠏"]` |
| `intervalMs` | Milliseconds per frame | `80` |

Pass an empty `frames` array to hide the spinner and show only the message. Each frame is passed through `spinnerColorFn` before display; the message is passed through `messageColorFn`.

`Loader.render(width)` prepends an empty line before its parent `Text` output, which gives the spinner visual breathing room.

```ts
import { Loader } from "tui";

const loader = new Loader(
  tui,
  (s) => chalk.cyan(s),   // spinner colour
  (s) => chalk.dim(s),    // message colour
  "Thinking…"
);
loader.start();
tui.addComponent(loader);

// When done:
loader.stop();
tui.removeComponent(loader);
```

#### CancellableLoader — loader the user can abort

`CancellableLoader` extends `Loader` and adds an `AbortController`. When the user presses Escape (`tui.select.cancel`), it calls `abortController.abort()` and fires the optional `onAbort` callback.

```ts
get signal(): AbortSignal   // pass to fetch(), agent calls, etc.
get aborted(): boolean
onAbort?: () => void

handleInput(data: string): void   // listens for Escape
dispose(): void                    // stops the animation
```

```ts
import { CancellableLoader } from "tui";

const loader = new CancellableLoader(tui, chalk.cyan, chalk.dim, "Working…");
loader.onAbort = () => {
  tui.removeComponent(loader);
  loader.dispose();
};
doWork(loader.signal).then(() => {
  loader.stop();
  tui.removeComponent(loader);
});
loader.start();
tui.addComponent(loader);
```

---

### Group 4 — Selection UIs: SelectList and SettingsList

#### SelectList — keyboard-navigable pick list

`SelectList` renders a list of items with up/down navigation, optional prefix-based filtering, and an optional two-column layout (label + description).

```ts
interface SelectItem {
  value: string;
  label: string;
  description?: string;
}

constructor(
  items: SelectItem[],
  maxVisible: number,
  theme: SelectListTheme,
  layout?: SelectListLayoutOptions
)

setFilter(filter: string): void          // prefix match on item.value
setSelectedIndex(index: number): void
getSelectedItem(): SelectItem | null
handleInput(keyData: string): void
render(width: number): string[]

onSelect?: (item: SelectItem) => void
onCancel?: () => void
onSelectionChange?: (item: SelectItem) => void
```

Navigation wraps: up from the first item goes to the last; down from the last goes to the first. When the list is longer than `maxVisible`, the visible window scrolls to centre the selected item, and a scroll indicator line `(3/12)` appears.

When the terminal is wider than 40 columns and an item has a `description`, `SelectList` renders a two-column layout: the label on the left (truncated to `primaryColumnWidth`) and the description on the right. `SelectListLayoutOptions` lets you tune `minPrimaryColumnWidth` and `maxPrimaryColumnWidth`; the actual width is clamped to the widest label in the filtered set.

`SelectListTheme` lets you colour the selected prefix, selected text, description text, scroll indicator, and the "no match" message.

```ts
import { SelectList } from "tui";

const list = new SelectList(
  [
    { value: "new",    label: "New session",    description: "Start a fresh conversation" },
    { value: "resume", label: "Resume session", description: "Pick up where you left off" },
  ],
  5,          // max visible items
  myTheme
);
list.onSelect = (item) => console.log("Selected:", item.value);
list.onCancel = () => tui.removeComponent(list);
tui.addComponent(list);
```

#### SettingsList — labelled settings with cycling values and submenus

`SettingsList` is a specialised list for key/value settings panels. Each `SettingItem` has a `label`, a `currentValue`, and optionally a `values` array (Enter cycles through them) or a `submenu` factory (Enter opens a child `Component`).

```ts
interface SettingItem {
  id: string;
  label: string;
  description?: string;
  currentValue: string;
  values?: string[];
  submenu?: (currentValue: string, done: (selectedValue?: string) => void) => Component;
}

constructor(
  items: SettingItem[],
  maxVisible: number,
  theme: SettingsListTheme,
  onChange: (id: string, newValue: string) => void,
  onCancel: () => void,
  options?: SettingsListOptions
)

updateValue(id: string, newValue: string): void
handleInput(data: string): void
render(width: number): string[]
```

`SettingsListOptions` has one field: `enableSearch?: boolean`. When enabled, a search `Input` appears at the top, and `SettingsList` uses fuzzy matching (via an internal `fuzzyFilter`) to narrow the visible items as the user types.

When a submenu is active, `render` and `handleInput` are fully delegated to the submenu component. Pressing Escape inside the submenu calls `done()`, which collapses back to the main list.

The selected item's `description` is word-wrapped and shown below the list. A hint line (`Enter/Space to change · Esc to cancel`) appears at the bottom.

---

### Group 5 — Rich content: Markdown and Image

#### Markdown — full GFM renderer for the terminal

`Markdown` parses a Markdown string using the `marked` library (a dependency of the `tui` package) and renders it to styled terminal lines. It handles paragraphs, headings (H1–H6), bold, italic, strikethrough, inline code, fenced code blocks, blockquotes, ordered and unordered lists (including nested), tables, horizontal rules, and hyperlinks.

```ts
constructor(
  text: string,
  paddingX: number,
  paddingY: number,
  theme: MarkdownTheme,
  defaultTextStyle?: DefaultTextStyle,
  options?: MarkdownOptions
)

setText(text: string): void
invalidate(): void
render(width: number): string[]
```

`MarkdownTheme` is a set of styling functions—one per markdown element:

| Function | Applied to |
|---|---|
| `heading` | Heading text |
| `bold` | `**bold**` spans |
| `italic` | `*italic*` spans |
| `strikethrough` | `~~struck~~` spans |
| `underline` | Underlined text |
| `code` | Inline `` `code` `` |
| `codeBlock` | Fenced code block body lines |
| `codeBlockBorder` | The ` ``` ` fence lines |
| `quote` | Blockquote body |
| `quoteBorder` | The `│ ` left border |
| `hr` | Horizontal rule |
| `link` | Link text |
| `linkUrl` | Fallback URL in parentheses |
| `listBullet` | The `-` / `1.` marker |
| `highlightCode` | Optional syntax highlighter `(code, lang) => string[]` |

The optional `codeBlockIndent` field (defaults to `"  "`) sets the prefix before each code line. `MarkdownOptions.preserveOrderedListMarkers` makes ordered lists use the original numbering from the source instead of normalising from 1.

Tables get a box-drawing border (`┌─…─┐`, `├─…─┤`, `└─…─┘`) and word-wrap cells that are too wide, shrinking columns proportionally when the terminal is narrow.

`Markdown` caches the rendered lines keyed on `(text, width)`. Call `invalidate()` or `setText(newText)` to clear the cache.

```ts
import { Markdown } from "tui";

const response = new Markdown(
  "## Result\n\nDone! See `output.ts` for details.",
  1, 0,
  myMarkdownTheme
);
tui.addComponent(response);
```

#### Image — Kitty/Sixel image with text fallback

`Image` renders an image from base-64 encoded data. It detects terminal image capabilities (Kitty graphics protocol or Sixel) and, when available, encodes and emits the appropriate escape sequence. When the terminal has no image support, it renders a plain-text fallback description.

```ts
interface ImageOptions {
  maxWidthCells?: number;   // default: min(width-2, 60)
  maxHeightCells?: number;  // default: computed from aspect ratio
  filename?: string;        // used in fallback text
  imageId?: number;         // reuse a Kitty image ID (e.g. for animations)
}

constructor(
  base64Data: string,
  mimeType: string,
  theme: ImageTheme,
  options?: ImageOptions,
  dimensions?: ImageDimensions
)

getImageId(): number | undefined
invalidate(): void
render(width: number): string[]
```

`ImageTheme` has a single field: `fallbackColor`, a function used to style the fallback text.

`Image.render` returns a number of lines equal to the image's height in terminal rows, so the TUI's differential renderer correctly accounts for the space the image occupies. The height is computed from the pixel dimensions and the terminal's cell size (obtained via `getCellDimensions()`).

---

### Widget summary table

| Class | Group | Interactive | Key callbacks |
|---|---|---|---|
| `Text` | Layout | No | — |
| `Box` | Layout | No | — |
| `Spacer` | Layout | No | — |
| `TruncatedText` | Layout | No | — |
| `Input` | Input | Yes | `onSubmit`, `onEscape` |
| `Editor` | Input | Yes | `onSubmit`, `onChange` |
| `Loader` | Feedback | No | — |
| `CancellableLoader` | Feedback | Yes (Esc) | `onAbort` |
| `SelectList` | Selection | Yes | `onSelect`, `onCancel`, `onSelectionChange` |
| `SettingsList` | Selection | Yes | `onChange`, `onCancel` |
| `Markdown` | Rich content | No | — |
| `Image` | Rich content | No | — |

---

## Part 2 — The ANSI utility belt

Here is the problem we need to solve before we can talk about any of these utilities.

### Why `string.length` is wrong for terminal layout

Imagine you want to truncate a label to 20 columns. You might write:

```ts
// WRONG — do not do this
const label = text.slice(0, 20);
```

This breaks in (at least) two ways.

**Way 1 — ANSI escape codes.** Chalk and similar libraries wrap text in invisible bytes. The string `"\x1b[31mRed\x1b[0m"` renders as the three visible characters `Red` but has length 12. If you slice it at column 5, you cut through the middle of an escape sequence, producing garbage in the terminal.

**Way 2 — Wide (East Asian) characters.** Many CJK characters occupy two terminal columns but have a JavaScript string length of 1 (one UTF-16 code unit). The character `中` has `"中".length === 1` but takes up 2 columns. Slicing at position 10 to produce a 10-column string can actually yield 9 or 11 visible columns depending on where the wide characters fall.

The `tui` package installs `get-east-asian-width` specifically to solve the second problem. The utilities in `src/utils.ts` use it together with `Intl.Segmenter` (for grapheme-aware iteration) to provide ANSI-safe, Unicode-correct layout primitives.

Let's walk through each utility in turn.

### `visibleWidth(str: string): number`

`visibleWidth` returns the number of terminal columns a string occupies—ignoring all ANSI escape codes and correctly counting wide characters as 2.

```ts
import { visibleWidth } from "tui";

visibleWidth("hello")                    // → 5  (pure ASCII fast path)
visibleWidth("\x1b[31mRed\x1b[0m")      // → 3  (escape codes stripped)
visibleWidth("中文")                     // → 4  (2 columns × 2 chars)
visibleWidth("hello 中")                 // → 8  (5 + 1 space + 2)
```

Internally, `visibleWidth` takes several paths:

- **Fast path — pure printable ASCII.** If every byte in the string is in the range `0x20`–`0x7E`, `visibleWidth` returns `str.length`. This is O(n) string scan but avoids all segmentation overhead.
- **Cached result.** Non-ASCII strings are looked up in an LRU cache of 512 entries. If hit, the cached width is returned immediately.
- **Full path.** Tabs are normalised to three spaces. ANSI escape sequences are stripped in a single pass (covering CSI `m`/`G`/`K`/`H`/`J` codes, OSC hyperlinks, and APC sequences). Then the cleaned string is iterated grapheme-by-grapheme using a shared `Intl.Segmenter` instance, and each grapheme's width is computed via `graphemeWidth` (which calls `eastAsianWidth` from `get-east-asian-width`, checks for emoji, and handles regional indicators and Thai/Lao AM vowels).

Tabs count as 3 columns in this system.

### `truncateToWidth(text, maxWidth, ellipsis?, pad?): string`

`truncateToWidth` shortens a string to fit within `maxWidth` terminal columns, appending an ellipsis if content was removed. ANSI codes are preserved in the kept portion and a reset (`\x1b[0m`) is injected after the kept text before the ellipsis, so colours do not bleed into the ellipsis.

```ts
truncateToWidth(text, maxWidth, ellipsis = "...", pad = false): string
```

| Parameter | Effect |
|---|---|
| `text` | Input string, may contain ANSI codes |
| `maxWidth` | Maximum visible column count |
| `ellipsis` | Appended when text is truncated (default `"..."`) |
| `pad` | If `true`, pad with spaces to exactly `maxWidth` (default `false`) |

```ts
import { truncateToWidth } from "tui";

truncateToWidth("Hello, world!", 8)
// → "Hello..."

truncateToWidth("\x1b[32mGreen text\x1b[0m", 9)
// → "\x1b[32mGreen t\x1b[0m..."   (colour kept, reset before ellipsis)

truncateToWidth("Hi", 10, "…", true)
// → "Hi        "   (padded to 10 columns)
```

When `maxWidth <= 0`, the function returns `""`. When the text already fits within `maxWidth`, no ellipsis is added, and `pad` is applied if requested.

The ellipsis width itself is taken into account: the function truncates to `maxWidth - visibleWidth(ellipsis)` columns of kept content, then appends the ellipsis to fill exactly `maxWidth`.

### `wrapTextWithAnsi(text, width): string[]`

`wrapTextWithAnsi` wraps styled text at word boundaries, returning an array of lines each no wider than `width` columns. Active ANSI styles are preserved across line breaks—each new line restarts with the active style codes from the end of the previous line, so a word highlighted in red does not lose its colour after a wrap.

```ts
wrapTextWithAnsi(text: string, width: number): string[]
```

Important behaviour notes:

- **Word wrap, not character wrap.** The function breaks at spaces. When a single token is wider than `width`, it falls back to character-level breaking.
- **Newlines in input.** The function splits the input on `\n` and wraps each paragraph independently. Style state carries across `\n` boundaries, so styles that span multiple logical lines (e.g. a coloured block) continue correctly.
- **No padding.** The returned lines are not padded to `width`. Each line is trimmed of trailing whitespace. If you need padding or background colour, apply it after wrapping (as `Text` and `Markdown` do).
- **Underline and OSC 8 hyperlinks get special end-of-line handling.** Underline is explicitly closed at line end to avoid bleeding into padding spaces; hyperlinks are closed and re-opened on the next line, which keeps them clickable in terminals that require OSC 8 to be on a single physical line.

```ts
import { wrapTextWithAnsi } from "tui";

wrapTextWithAnsi("The quick brown fox jumped over the lazy dog", 20)
// → [
//     "The quick brown fox",
//     "jumped over the lazy",
//     "dog"
//   ]

wrapTextWithAnsi("\x1b[33mWarning:\x1b[0m this is a long message that needs wrapping", 20)
// → lines where the first line preserves \x1b[33m on "Warning:"
//   and subsequent lines are unstyled (reset was in the source)
```

### `sliceByColumn(line, startCol, length, strict?): string`

`sliceByColumn` extracts a range of terminal columns from a string, correctly skipping ANSI codes and handling wide characters at range boundaries.

```ts
sliceByColumn(line: string, startCol: number, length: number, strict = false): string
```

| Parameter | Meaning |
|---|---|
| `startCol` | First column to include (0-indexed) |
| `length` | Number of columns to include |
| `strict` | If `true`, exclude wide chars that would extend past `startCol + length` |

```ts
import { sliceByColumn } from "tui";

// Pure ASCII — straightforward:
sliceByColumn("Hello, world!", 7, 5)
// → "world"

// ANSI codes are skipped in column counting but included in output when in range:
sliceByColumn("\x1b[31mRed text\x1b[0m", 4, 4)
// → "text" (with the red colour code, since it was applied before column 4)

// Wide character at boundary:
//   "AB中DE" — columns: A=0, B=1, 中=2+3, D=4, E=5
sliceByColumn("AB中DE", 2, 2)           // → "中"  (the wide char occupies cols 2-3)
sliceByColumn("AB中DE", 2, 2, true)     // → ""   (strict: 中 would extend to col 3,
                                         //         past 2+2-1=3... actually fits, varies by char)
```

`sliceByColumn` is what `Input` uses for horizontal scrolling: when the value is too wide to display, it extracts the visible window starting at the scroll offset. You will use it whenever you need to implement viewport-relative text display.

### Composing the utilities

In practice the utilities work together in a short, consistent pipeline that every widget in the library follows:

```ts
// 1. Wrap content to available columns (ANSI-safe)
const lines = wrapTextWithAnsi(content, contentWidth);

// 2. For each line, add margins then pad or colour
for (const line of lines) {
  const withMargins = leftMargin + line + rightMargin;
  const visibleLen = visibleWidth(withMargins);   // ← never use .length
  const padding = " ".repeat(Math.max(0, width - visibleLen));
  result.push(withMargins + padding);
}

// 3. For labels that must not wrap, truncate instead:
const label = truncateToWidth(rawLabel, availableWidth);
```

Notice that `string.length` never appears in step 2 — it is always `visibleWidth()`. This is the habit to build: any time you need to know how wide a terminal string is, use `visibleWidth`.

---

## What is next

We now have a full picture of the widget toolbox and the width-correct primitives that underpin it. The next natural question is: how does autocomplete work, and how do we wire all of these widgets together into a usable chat interface? That is what we build in the next chapter.

---

← Previous: [Terminal Abstraction, Input Handling, and Keybindings](./terminal-abstraction-and-input.md) · Next: [Autocomplete and Building a Complete Chat Interface](./autocomplete-and-a-complete-chat-interface.md) →
