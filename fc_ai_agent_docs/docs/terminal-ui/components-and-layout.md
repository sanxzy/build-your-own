---
title: "Built-In Components: Widgets and Layout"
description: "Build the widget library — Box (flexbox-inspired layout), Input (multi-line with cursor and history), Markdown (streaming to terminal with syntax highlighting), Loader (spinner), and the ANSI utility belt for width-correct text layout."
category: terminal-ui
type: tutorial
tags: [components, editor, input, markdown, loader, select-list, box, text, image, ANSI utilities, visibleWidth, layout, flexbox, tui, widgets]
keywords: [TUI components, flexbox layout, text input, streaming markdown, syntax highlighting, ANSI width, cursor management]
sources: [S34, S36, S37, S38, S42, S43]
---

**TL;DR** — With the render engine working, we need widgets. We'll build a component library: `Box` for flexbox-inspired layout, `Text` for ANSI-aware rendering, `Input` for multi-line text with cursor movement and history, `Markdown` for streaming markdown with syntax highlighting, and `Loader` for animated spinners. Plus the ANSI utility belt for getting text widths right when escape sequences are involved.

## Box: flexbox-inspired layout

Terminals don't have CSS. The `Box` component gives us a simple layout model:

```ts
class Box extends Component {
  constructor(private config: {
    direction?: "row" | "column";
    children: Component[];
    gap?: number;
    padding?: number;
    border?: boolean;
  }) { super(); }

  render(): View {
    const childViews = this.config.children.map(c => c.render());
    if (this.config.direction === "row") {
      return this.layoutRow(childViews);
    }
    return this.layoutColumn(childViews);
  }

  private layoutRow(views: View[]): View {
    const height = Math.max(...views.map(v => v.height));
    const width = views.reduce((sum, v) => sum + v.width, 0)
      + (this.config.gap ?? 0) * (views.length - 1);

    const cells = createEmptyGrid(height, width);
    let col = 0;
    for (const view of views) {
      copyView(cells, view, 0, col);
      col += view.width + (this.config.gap ?? 0);
    }

    return { width, height, cells };
  }

  private layoutColumn(views: View[]): View {
    const width = Math.max(...views.map(v => v.width));
    const height = views.reduce((sum, v) => sum + v.height, 0)
      + (this.config.gap ?? 0) * (views.length - 1);

    const cells = createEmptyGrid(height, width);
    let row = 0;
    for (const view of views) {
      copyView(cells, view, row, 0);
      row += view.height + (this.config.gap ?? 0);
    }

    return { width, height, cells };
  }
}
```

Box enables layout patterns like:

```ts
// A chat interface layout: input at the bottom, messages above
new Box({
  direction: "column",
  children: [
    messageList,     // scrollable message area (flex: 1)
    new Box({        // input area
      direction: "row",
      children: [prompt, input],
      gap: 1,
    }),
  ],
});
```

## Text: ANSI-aware rendering

The `Text` component renders a string into styled cells, handling ANSI escape sequences within the text:

```ts
class Text extends Component {
  render(): View {
    const tokens = this.parseAnsi(this.content);
    const cells: Cell[][] = [[]];
    let row = 0;

    for (const token of tokens) {
      for (const char of token.text) {
        if (char === "\n") {
          cells.push([]);
          row++;
        } else {
          cells[row].push({ char, style: token.style });
        }
      }
    }

    return { width: maxRowWidth(cells), height: cells.length, cells };
  }
}
```

## Input: multi-line text editor

The `Input` component is the most complex widget. It provides:

- **Cursor movement** — left/right by character, up/down by line, home/end, word navigation
- **Selection** — shift + movement creates a selection range
- **Insertion** — typing at cursor, with auto-wrapping
- **Deletion** — backspace, delete, delete word, delete line
- **History** — up/down cycles through input history
- **Completion UI** — triggers autocomplete on Tab

The component tracks cursor position (row, col) and selection range (start, end):

```ts
class Input extends Component {
  private value = "";
  private cursor = 0;
  private selection: { start: number; end: number } | null = null;
  private history: string[] = [];
  private historyIndex = -1;

  handleKey(key: Key): boolean {
    if (key.name === "left")  { this.moveCursor(-1, key.shift); return true; }
    if (key.name === "right") { this.moveCursor(1, key.shift);  return true; }
    if (key.name === "home")  { this.cursorTo(0, key.shift);    return true; }
    if (key.name === "end")   { this.cursorTo(this.value.length, key.shift); return true; }
    if (key.name === "backspace") { this.deleteBefore(); return true; }
    if (key.name === "delete")    { this.deleteAfter();  return true; }
    if (key.name === "up")        { this.historyPrev();  return true; }
    if (key.name === "down")      { this.historyNext();  return true; }
    if (key.name === "enter")     { this.submit();       return true; }

    // Printable characters
    if (key.sequence.length === 1 && !key.ctrl && !key.alt) {
      this.insert(key.sequence);
      return true;
    }

    return false;
  }
}
```

## Markdown: streaming to the terminal

The `Markdown` component renders streaming markdown text to styled terminal output. It handles:

- **Headings** — `# H1`, `## H2`, etc. in bold
- **Code blocks** — ```-delimited blocks with syntax highlighting
- **Inline code** — backtick-delimited spans in a distinct color
- **Lists** — unordered (`-`) and ordered (`1.`)
- **Bold and italic** — `**bold**` and `*italic*`
- **Links** — `[text](url)` rendered as text with the URL in a dim footer

The component uses the `marked` library for parsing and our own renderer for terminal output:

```ts
class Markdown extends Component {
  private tokens: Token[] = [];

  appendChunk(chunk: string): void {
    // Parse the cumulative markdown and extract new tokens
    const newTokens = marked.lexer(this.buffer + chunk);
    this.tokens = newTokens;
    this.buffer += chunk;
    this.markDirty();
  }

  render(): View {
    const cells: Cell[][] = [];
    for (const token of this.tokens) {
      renderToken(token, cells);
    }
    return { width: maxWidth(cells), height: cells.length, cells };
  }
}
```

The streaming design: `appendChunk()` is called as markdown text arrives from the LLM. The component re-parses and re-renders on each chunk, creating a smooth streaming effect in the terminal.

## Loader: animated spinner

A simple animated component that cycles through spinner frames on a timer:

```ts
class Loader extends Component {
  private frame = 0;
  private frames = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"];
  private timer: NodeJS.Timeout | null = null;

  start(): void {
    this.timer = setInterval(() => {
      this.frame = (this.frame + 1) % this.frames.length;
      this.markDirty();
    }, 80);
  }

  render(): View {
    const char = this.frames[this.frame];
    const text = `${char} ${this.message}`;
    return renderText(text, this.color);
  }

  stop(): void {
    if (this.timer) { clearInterval(this.timer); this.timer = null; }
  }
}
```

## The ANSI utility belt

ANSI escape sequences are invisible characters that affect rendering but don't take up cells. Getting string widths right requires stripping them:

```ts
function ansiRegex(): RegExp {
  return /\x1b\[[0-9;]*[a-zA-Z]/g;
}

function visibleLength(str: string): number {
  return str.replace(ansiRegex(), "").length;
}

function truncateToWidth(str: string, width: number): string {
  const visible = str.replace(ansiRegex(), "");
  if (visible.length <= width) return str + " ".repeat(width - visible.length);

  // Truncate while preserving ANSI codes
  let visibleCount = 0;
  let result = "";
  let i = 0;
  while (i < str.length && visibleCount < width) {
    if (str[i] === "\x1b") {
      const match = str.slice(i).match(ansiRegex());
      if (match) {
        result += match[0];
        i += match[0].length;
        continue;
      }
    }
    result += str[i];
    visibleCount++;
    i++;
  }
  return result;
}
```

East Asian characters (CJK) are double-width in most terminals. The `get-east-asian-width` package classifies each character so we can calculate the correct display width.

## What we've built

A complete widget library for terminal UIs:

| Component | Purpose |
|---|---|
| `Box` | Flexbox-inspired layout (row/column, gap, padding) |
| `Text` | ANSI-aware text rendering |
| `Input` | Multi-line text editor with cursor, selection, history |
| `Markdown` | Streaming markdown with syntax highlighting |
| `Loader` | Animated spinner with configurable frames |
| ANSI utils | Visible width calculation, width-correct truncation |

---

← Previous: [Terminal Abstraction, Input, and Keybindings](./terminal-abstraction-and-input.md) · Next: [Autocomplete and Assembling a Chat Interface](./autocomplete-and-chat-interface.md) →
