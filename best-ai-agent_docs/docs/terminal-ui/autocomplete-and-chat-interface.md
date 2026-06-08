---
title: "Autocomplete and Assembling a Chat Interface"
description: "Build the autocomplete engine with fuzzy matching and ranked suggestions, then assemble all TUI components into a complete chat interface — input, streaming markdown output, and autocomplete."
category: terminal-ui
type: tutorial
tags: [autocomplete, fuzzy matching, path completion, slash commands, chat interface, spinner, markdown rendering, editor, TUI composition, tui]
keywords: [fuzzy autocomplete, chat interface, TUI assembly, slash commands, path completion]
sources: [S39, S34, S42]
---

**TL;DR** — We have all the individual TUI components. Now we'll build the autocomplete engine that suggests slash commands and file paths as the user types, then assemble everything into a complete chat interface — the terminal equivalent of ChatGPT's web UI, with streaming markdown rendering, input history, and an autocomplete popup.

## Autocomplete engine

Create `packages/tui/src/autocomplete.ts`. The autocomplete engine provides suggestions as the user types. It's provider-based — different completion sources can be combined:

```ts
interface AutocompleteProvider {
  getSuggestions(input: string, cursorPos: number): Suggestion[];
}

interface Suggestion {
  text: string;
  display: string;
  description?: string;
}
```

### Fuzzy matching

Suggestions are ranked by fuzzy match score:

```ts
function fuzzyScore(query: string, candidate: string): number {
  let score = 0;
  let qi = 0;

  for (let ci = 0; ci < candidate.length && qi < query.length; ci++) {
    if (candidate[ci].toLowerCase() === query[qi].toLowerCase()) {
      score += 10; // character match
      if (ci === 0 || candidate[ci - 1] === "/" || candidate[ci - 1] === "-") {
        score += 5; // word boundary bonus
      }
      if (ci === qi) {
        score += 3; // consecutive match bonus
      }
      qi++;
    } else {
      score -= 1; // gap penalty
    }
  }

  return qi === query.length ? score : 0; // zero if not all chars matched
}
```

The scoring favors:
- Matches at word boundaries (`/`, `-`, start of string)
- Consecutive matches (the query appears as a substring)
- Higher score = better match

### Slash-command autocomplete

Commands like `/help`, `/model`, `/session` are registered with descriptions:

```ts
class SlashCommandProvider implements AutocompleteProvider {
  private commands: Array<{ command: string; description: string }> = [];

  register(command: string, description: string): void {
    this.commands.push({ command, description });
  }

  getSuggestions(input: string): Suggestion[] {
    if (!input.startsWith("/")) return [];
    const query = input.slice(1);

    return this.commands
      .map(c => ({
        text: "/" + c.command,
        display: c.command,
        description: c.description,
        score: fuzzyScore(query, c.command),
      }))
      .filter(s => s.score > 0)
      .sort((a, b) => b.score - a.score)
      .slice(0, 10);
  }
}
```

### File-path autocomplete

For file path arguments, the autocomplete reads the filesystem:

```ts
class FilePathProvider implements AutocompleteProvider {
  getSuggestions(input: string, cursorPos: number): Suggestion[] {
    // Extract the path fragment at cursor position
    const fragment = extractPathFragment(input, cursorPos);
    if (!fragment) return [];

    const dir = path.dirname(fragment);
    const prefix = path.basename(fragment);

    try {
      const entries = fs.readdirSync(dir || ".");
      return entries
        .filter(e => e.startsWith(prefix))
        .map(e => ({
          text: path.join(dir, e) + (fs.statSync(path.join(dir, e)).isDirectory() ? "/" : ""),
          display: e,
          description: fs.statSync(path.join(dir, e)).isDirectory() ? "directory" : "file",
          score: e.startsWith(prefix) ? (e === prefix ? 100 : 50) : 10,
        }))
        .sort((a, b) => b.score - a.score)
        .slice(0, 10);
    } catch {
      return [];
    }
  }
}
```

### Combined provider

Multiple providers are combined into one:

```ts
class CombinedAutocompleteProvider implements AutocompleteProvider {
  constructor(private providers: AutocompleteProvider[]) {}

  getSuggestions(input: string, cursorPos: number): Suggestion[] {
    const all = this.providers.flatMap(p => p.getSuggestions(input, cursorPos));
    return all
      .sort((a, b) => (b.score ?? 0) - (a.score ?? 0))
      .slice(0, 10);
  }
}
```

## The autocomplete popup

The suggestions render as an overlay below the input line:

```ts
class AutocompletePopup extends Component {
  private suggestions: Suggestion[] = [];
  private selectedIndex = 0;

  setSuggestions(suggestions: Suggestion[]): void {
    this.suggestions = suggestions;
    this.selectedIndex = 0;
  }

  render(): View {
    if (this.suggestions.length === 0) {
      return { width: 0, height: 0, cells: [] };
    }

    const maxWidth = Math.max(...this.suggestions.map(s => s.display.length + 20));
    const cells: Cell[][] = [];

    for (let i = 0; i < this.suggestions.length; i++) {
      const s = this.suggestions[i];
      const isSelected = i === this.selectedIndex;
      const bg = isSelected ? Style.bgBlue : Style.bgDefault;
      const line = `${s.display.padEnd(20)} ${s.description ?? ""}`.padEnd(maxWidth);
      cells.push(line.split("").map(c => ({ char: c, style: bg })));
    }

    return { width: maxWidth, height: cells.length, cells };
  }
}
```

The popup appears as an overlay below the cursor position. Tab inserts the selected suggestion. The popup updates on every keystroke.

## Assembling the chat interface

Now we wire everything together. The chat interface has four regions:

```
┌──────────────────────────────────┐
│  Chat history                    │  ← Markdown (scrollable)
│  (streaming markdown)            │
│                                  │
├──────────────────────────────────┤
│  > User input here...       █    │  ← Input (with cursor)
├──────────────────────────────────┤
│  /help    Show available commands│  ← AutocompletePopup (overlay)
│  /model   Change the model       │
└──────────────────────────────────┘
```

```ts
class ChatInterface extends Component {
  private messages: Message[] = [];
  private input: Input;
  private markdown: Markdown;
  private autocomplete: AutocompletePopup;
  private provider: CombinedAutocompleteProvider;

  constructor() {
    super();
    this.input = new Input();
    this.markdown = new Markdown();
    this.autocomplete = new AutocompletePopup();
    this.provider = new CombinedAutocompleteProvider([
      new SlashCommandProvider(),
      new FilePathProvider(),
    ]);

    this.input.onChange((value, cursor) => {
      const suggestions = this.provider.getSuggestions(value, cursor);
      this.autocomplete.setSuggestions(suggestions);
    });

    this.input.onSubmit((value) => {
      this.autocomplete.clear();
      this.history.push(value);
    });
  }

  render(): View {
    return new Box({
      direction: "column",
      children: [this.markdown, this.input],
      gap: 0,
    }).render();
    // Autocomplete is rendered as an overlay, not in the Box
  }
}
```

### Streaming markdown from the agent

When the agent loop emits text deltas, they're fed to the markdown component:

```ts
// In the chat interface's agent subscriber:
agent.subscribe((event) => {
  switch (event.type) {
    case "text_start":
      this.markdown.startBlock(event.contentIndex);
      break;
    case "text_delta":
      this.markdown.appendToBlock(event.contentIndex, event.delta);
      break;
    case "text_end":
      this.markdown.finishBlock(event.contentIndex);
      break;
    case "done":
      this.messages.push(event.message);
      this.markdown.finalize();
      this.loader.stop();
      break;
  }
});
```

The markdown component accumulates chunks, re-parses on each delta, and re-renders — creating smooth streaming text in the terminal.

## What we've built

The Terminal UI section is complete. We have:

- **TUI engine** with differential rendering, overlay compositing, and focus management
- **Terminal abstraction** with real and virtual implementations
- **Key parser** supporting VT/xterm sequences and the Kitty keyboard protocol
- **Keybinding system** with chord sequences
- **Widget library** — Box, Text, Input, Markdown, Loader, SelectList
- **Autocomplete engine** with fuzzy matching, slash commands, and file paths
- **Complete chat interface** — streaming markdown output, input with history, autocomplete popup

In the next section, we'll build the Coding Agent — composing the agent core and terminal UI into a working CLI application.

---

← Previous: [Built-In Components: Widgets and Layout](./components-and-layout.md) · Next: [AgentSession: The Core of the Coding Agent](../coding-agent/agent-session-core.md) →
