---
title: "Autocomplete and Building a Complete Chat Interface"
description: "Add slash-command and file-path autocomplete to the editor, then wire all TUI building blocks into a complete, runnable chat interface."
category: terminal-ui
type: tutorial
tags: [autocomplete, CombinedAutocompleteProvider, slash-command autocomplete, file-path autocomplete, chat interface, spinner, markdown render, editor, TUI hello world, tui, AutocompleteProvider, AutocompleteItem, SlashCommand, AutocompleteSuggestions, Loader, Markdown, Text, Editor, ProcessTerminal]
keywords: [terminal autocomplete, slash command completion, file completion, tab completion, fuzzy file search, fd integration, chat UI, terminal chat app, TUI capstone, at-prefix completion]
sources: [S42, S45]
---

**TL;DR** — This chapter adds `CombinedAutocompleteProvider` to the editor so that typing `/` suggests slash commands and typing `@` or a path prefix suggests files from the filesystem. We then wire every TUI building block together — message area, loading spinner, editor with autocomplete, and the render loop — into a complete, runnable terminal chat application. By the end you will have built a real chat interface using only the `tui` package.

# Autocomplete and Building a Complete Chat Interface

We have now covered the three core TUI abstractions:

- **`TUI` and its render loop** — the parent container that owns a list of child components, calls each one's `render()` method whenever state changes, and writes the diff to the terminal. See [The TUI Class and Render Engine](./the-tui-class-and-render-engine.md) for the full walkthrough.
- **`ProcessTerminal` and key input** — the raw-mode wrapper that captures keystrokes and feeds them into the focus system. See [Terminal Abstraction and Input](./terminal-abstraction-and-input.md).
- **Built-in components** — `Editor`, `Markdown`, `Text`, `Loader`, and the layout helpers. See [Built-In Components: Widgets, Layout, and ANSI Utilities](./built-in-components-and-layout.md).

What we are still missing is something that helps the user *discover* what they can type. When someone opens a chat prompt they should be able to press `/` and see available commands, or type `@src/` and get file completions. That is what autocomplete provides.

Once we have autocomplete wired up, we have every piece we need to build a full chat interface. That is exactly what we will do in this chapter.

## What the editor needs from autocomplete

The `Editor` component manages cursor position and multi-line text, but it knows nothing about the *domain* — it does not know what slash commands exist, or where the project files live. We need to give it a pluggable strategy that the editor can call at the right moments: "here is the current text and cursor position — what should I suggest?"

That is the `AutocompleteProvider` interface:

```ts
export interface AutocompleteProvider {
  // Returns suggestions for the current cursor position, or null if none apply
  getSuggestions(
    lines: string[],
    cursorLine: number,
    cursorCol: number,
    options: { signal: AbortSignal; force?: boolean },
  ): Promise<AutocompleteSuggestions | null>;

  // Applies the chosen suggestion, returning the updated text and cursor
  applyCompletion(
    lines: string[],
    cursorLine: number,
    cursorCol: number,
    item: AutocompleteItem,
    prefix: string,
  ): { lines: string[]; cursorLine: number; cursorCol: number };

  // Optional: called on Tab to decide whether file completion should trigger
  shouldTriggerFileCompletion?(
    lines: string[],
    cursorLine: number,
    cursorCol: number,
  ): boolean;
}
```

Notice the `signal: AbortSignal` in `getSuggestions`. File-system lookups are asynchronous; if the user keeps typing before suggestions arrive, the editor cancels the in-flight request via the signal. The provider is expected to respect it and stop early.

The return type `AutocompleteSuggestions` carries both the matching items and the *prefix* — the portion of text that the selection will replace:

```ts
export interface AutocompleteSuggestions {
  items: AutocompleteItem[];
  prefix: string; // text being completed, e.g. "/del" or "src/"
}

export interface AutocompleteItem {
  value: string;       // the text to insert
  label: string;       // shown in the suggestion list
  description?: string; // optional hint shown alongside the label
}
```

`applyCompletion` receives the full `prefix` so it can locate and replace exactly the right span in the current line — no ambiguity about where the substitution happens.

## Slash commands and the `SlashCommand` type

Slash commands are slightly richer than plain `AutocompleteItem` entries because they can accept *arguments* — after the user picks `/run`, they might continue typing a filename, and the provider should offer file completions for that argument.

```ts
export interface SlashCommand {
  name: string;
  description?: string;
  argumentHint?: string; // shown as "hint — description" in the list
  // Called when the user is typing the argument after "/name "
  // Return null to signal no argument completion available
  getArgumentCompletions?(argumentPrefix: string): Awaitable<AutocompleteItem[] | null>;
}
```

`Awaitable<T>` is just `T | Promise<T>` — argument completions may be synchronous or asynchronous.

You can mix `SlashCommand` objects (with argument support) and plain `AutocompleteItem` objects (name + label only) in the same list. `CombinedAutocompleteProvider` handles both.

## Meet `CombinedAutocompleteProvider`

Rather than writing two separate providers — one for slash commands, one for file paths — the `tui` package ships a single class that handles both triggers and routes each to the right logic:

```ts
export class CombinedAutocompleteProvider implements AutocompleteProvider {
  constructor(
    commands: (SlashCommand | AutocompleteItem)[] = [],
    basePath: string,
    fdPath: string | null = null,
  )
}
```

| Parameter  | Type                                   | Purpose                                                                                 |
|------------|----------------------------------------|-----------------------------------------------------------------------------------------|
| `commands` | `(SlashCommand \| AutocompleteItem)[]` | The slash commands to suggest when the user types `/`.                                   |
| `basePath` | `string`                               | The root directory used to resolve relative file paths (typically `process.cwd()`).     |
| `fdPath`   | `string \| null`                       | Path to the `fd` binary for fuzzy `@`-prefix file search. `null` disables fuzzy search. |

The provider chooses its strategy based on what is in the text before the cursor:

| Trigger in text before cursor       | Strategy                                                               |
|-------------------------------------|------------------------------------------------------------------------|
| Starts with `@` (or `@"`)           | Fuzzy file search via `fd`, scoped to `basePath`                       |
| Starts with `/`, no space yet       | Fuzzy-filter the command list by what follows the `/`                  |
| Starts with `/name ` (space after)  | Call `getArgumentCompletions` on the matched command                   |
| Contains `/`, `.`, `~/`, or ends ` `| `readdirSync`-based path completion relative to `basePath`             |
| Tab pressed explicitly (`force`)    | Force file completion regardless of the current prefix                 |

We will see exactly how this routing works in `getSuggestions` below.

## How `getSuggestions` routes between providers

Let's walk through the routing logic step by step, because understanding it tells you what to type to test each path.

### Step 1 — Check for the `@` prefix

```ts
// Inside CombinedAutocompleteProvider.getSuggestions (simplified)
const atPrefix = this.extractAtPrefix(textBeforeCursor);
if (atPrefix) {
  const { rawPrefix, isQuotedPrefix } = parsePathPrefix(atPrefix);
  const suggestions = await this.getFuzzyFileSuggestions(rawPrefix, {
    isQuotedPrefix,
    signal: options.signal,
  });
  if (suggestions.length === 0) return null;
  return { items: suggestions, prefix: atPrefix };
}
```

`extractAtPrefix` looks at the token immediately before the cursor. If it starts with `@` (bare or quoted as `@"`), the provider launches a fuzzy `fd` search anchored at `basePath`. Results come back scored: exact filename match scores 100, prefix match 80, substring in filename 50, substring in full path 30. Directories get a +10 bonus so they float to the top. Only the top 20 results are returned to keep the list manageable.

If `fdPath` is `null`, fuzzy search returns an empty array — `@` suggestions will not appear, and the caller should pass `null` in that argument when `fd` is not available.

### Step 2 — Check for the `/` prefix (no space yet)

```ts
if (!options.force && textBeforeCursor.startsWith("/")) {
  const spaceIndex = textBeforeCursor.indexOf(" ");

  if (spaceIndex === -1) {
    // Still typing the command name — fuzzy-filter the list
    const prefix = textBeforeCursor.slice(1);
    const filtered = fuzzyFilter(commandItems, prefix, (item) => item.name);
    if (filtered.length === 0) return null;
    return { items: filtered, prefix: textBeforeCursor };
  }

  // Space found — we are in argument position
  const commandName = textBeforeCursor.slice(1, spaceIndex);
  const argumentText = textBeforeCursor.slice(spaceIndex + 1);
  const command = this.commands.find((cmd) => /* name matches */ ...);
  if (!command?.getArgumentCompletions) return null;

  const argumentSuggestions = await command.getArgumentCompletions(argumentText);
  if (!Array.isArray(argumentSuggestions) || argumentSuggestions.length === 0) return null;
  return { items: argumentSuggestions, prefix: argumentText };
}
```

Two sub-cases here:

1. **No space yet** — the user is still typing `/del`. We fuzzy-filter the command names and return all that match. The `prefix` is the full `/del` text so that `applyCompletion` knows exactly what to replace.
2. **Space found** — the user has committed to a command name (`/run `) and is now typing an argument. We look up the command by name and call its `getArgumentCompletions`. If the command does not define that method, we return `null` and let the next step try file-path completion instead.

### Step 3 — Fall through to file-path completion

```ts
const pathMatch = this.extractPathPrefix(textBeforeCursor, options.force ?? false);
if (pathMatch === null) return null;

const suggestions = this.getFileSuggestions(pathMatch);
if (suggestions.length === 0) return null;
return { items: suggestions, prefix: pathMatch };
```

`extractPathPrefix` checks whether the token before the cursor looks like a path:

- It contains `/`
- It starts with `.` or `~/`
- There is an unclosed quote (`"`) which begins a path token
- `force` is `true` (Tab was pressed explicitly)

`getFileSuggestions` uses `readdirSync` to list the directory implied by the prefix, then keeps only entries whose names start with the search stem. Directories are sorted first, then alphabetically within each group.

### Applying a completion

`applyCompletion` checks which *kind* of completion was accepted and handles each case:

| Accepted item kind              | Behaviour                                                                                                     |
|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| Slash command name              | Inserts `/<value> ` (adds a trailing space so the cursor is ready for arguments).                             |
| `@` file attachment             | Inserts the path; no trailing space for directories (so the user can keep drilling down with more keystrokes).|
| Command argument                | Inserts the value; no trailing space for directories.                                                         |
| Plain file path                 | Inserts the path; no trailing space for directories.                                                          |

For quoted paths (`"src/util"`) the method correctly removes the closing quote from `afterCursor` if the inserted value already ends with one, to avoid doubling up.

## Wiring autocomplete into the editor

Now that we understand the provider, let's build a minimal editor with autocomplete attached. The `Editor` component has a `setAutocompleteProvider` method that accepts any `AutocompleteProvider`:

```ts
import { CombinedAutocompleteProvider } from "tui";
import { Editor } from "tui";
import { ProcessTerminal } from "tui";
import { TUI } from "tui";

// Step 1: Set up the terminal and TUI container
const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

// Step 2: Create the editor
const editor = new Editor(tui, /* editorTheme */);

// Step 3: Create the autocomplete provider
const autocompleteProvider = new CombinedAutocompleteProvider(
  [
    { name: "delete", description: "Delete the last message" },
    { name: "clear",  description: "Clear all messages" },
  ],
  process.cwd(), // basePath for file resolution
  // No fdPath — fuzzy @ search disabled for now
);

// Step 4: Attach the provider to the editor
editor.setAutocompleteProvider(autocompleteProvider);

// Step 5: Add editor to TUI and start
tui.addChild(editor);
tui.setFocus(editor);
tui.start();
```

At this point the editor is alive: typing `/` will show the `/delete` and `/clear` commands; typing a path like `./src/` will list files from `process.cwd()`. You won't see any messages yet — that comes in the next step.

## Building the complete chat interface

We now have all the pieces. Let's assemble them into a full chat interface, one concern at a time.

### The starting point — a welcome message

The TUI's `addChild` method adds any component to the scrollable area above the editor. We start with a static `Text` greeting so the screen is not blank:

```ts
import { Text } from "tui";

tui.addChild(
  new Text(
    "Welcome to Simple Chat!\n\nType your messages below. Type '/' for commands. Press Ctrl+C to exit.",
  ),
);
```

`Text` is a plain-text component that wraps at the terminal width. It is the lightest way to put a fixed message on screen — no markdown parsing, no styling logic. We added it to the TUI *before* the editor, so it appears above the input area.

### Accepting and displaying user input

The `Editor` fires `onSubmit` when the user presses Enter. We need to handle that event and render the user's message back as a `Markdown` component — so code blocks, bold, and other formatting are preserved:

```ts
import { Markdown } from "tui";

let isResponding = false;

editor.onSubmit = (value: string) => {
  if (isResponding) return; // ignore while a response is in flight

  const trimmed = value.trim();
  if (!trimmed) return;

  isResponding = true;
  editor.disableSubmit = true; // prevent double-submission

  // Insert the user's message just before the editor
  const userMessage = new Markdown(value, 1, 1, /* markdownTheme */);
  const children = tui.children;
  children.splice(children.length - 1, 0, userMessage);

  tui.requestRender();
};
```

A few things to notice here:

- `editor.disableSubmit = true` locks the editor while we are waiting for a response. The user can still type and edit, but pressing Enter does nothing until we re-enable it.
- We insert the `Markdown` component at `children.length - 1` — that is, *just before* the editor (which is always the last child). This keeps the editor pinned at the bottom.
- `tui.requestRender()` triggers one render pass. Without it the new message would not appear until the next keystroke.

`Markdown` is constructed with `(content, paddingLeft, paddingRight, theme)`. The `1, 1` arguments give the message a single-column margin on each side — enough breathing room without wasting space.

### Adding a loading spinner

After inserting the user message we need to signal that the agent is thinking. We do that with a `Loader` component, which renders an animated spinner:

```ts
import { Loader } from "tui";
import chalk from "chalk";

// Inside onSubmit, after inserting userMessage:
const loader = new Loader(
  tui,
  (s) => chalk.cyan(s),   // spinner animation colour
  (s) => chalk.dim(s),    // label text colour
  "Thinking...",           // label shown beside the spinner
);
children.splice(children.length - 1, 0, loader);
tui.requestRender();
```

`Loader` takes the TUI reference as its first argument so it can schedule render ticks internally — the spinning animation is driven by periodic calls to `tui.requestRender()` under the hood.

We again insert the loader *before* the editor (`children.length - 1`), so the layout is: welcome text → user message → spinning loader → editor.

### Receiving the response and cleaning up

When the (simulated or real) response arrives, we remove the loader and insert the reply:

```ts
// Simulate an async response
setTimeout(() => {
  tui.removeChild(loader);

  const responseText = "I see what you mean."; // replace with real LLM output
  const botMessage = new Markdown(responseText, 1, 1, /* markdownTheme */);
  children.splice(children.length - 1, 0, botMessage);

  isResponding = false;
  editor.disableSubmit = false;

  tui.requestRender();
}, 1000);
```

`tui.removeChild(loader)` removes the spinner from `tui.children` and triggers a render, so the spinner disappears cleanly. Then we splice in the response and re-enable the editor.

In a real integration you would replace `setTimeout` with your actual async call — streaming tokens from an LLM, processing tool calls, etc. The shape of the code stays the same: insert a loader, await the response, remove the loader, insert the result.

### Handling slash commands in `onSubmit`

We registered `/delete` and `/clear` in the autocomplete provider so the editor can suggest them, but we also need to actually *execute* them when submitted. That logic goes in `onSubmit` before the main message path:

```ts
editor.onSubmit = (value: string) => {
  if (isResponding) return;

  const trimmed = value.trim();

  if (trimmed === "/delete") {
    const children = tui.children;
    // children[0]   = welcome Text
    // children[1..n-2] = message components
    // children[n-1] = editor
    if (children.length > 3) {
      // Remove the component just before the editor
      children.splice(children.length - 2, 1);
    }
    tui.requestRender();
    return; // handled — do not fall through to message submission
  }

  if (trimmed === "/clear") {
    const children = tui.children;
    // Keep welcome text (index 0) and editor (last); remove everything in between
    children.splice(2, children.length - 3);
    tui.requestRender();
    return;
  }

  // ... message submission logic
};
```

The index arithmetic reflects the fixed layout we chose: the welcome `Text` is always at index 0, and the `Editor` is always at the last position. Message components live in between.

## The complete example

Putting it all together, here is the full runnable chat interface. This is a generic version of the TUI "hello world" — it uses only the `tui` package and a colour library; you can drop it into any project and extend it with a real LLM call.

```ts
/**
 * Complete chat interface using the tui package.
 * Replace the setTimeout block with your actual LLM integration.
 */

import chalk from "chalk";
import { CombinedAutocompleteProvider } from "tui";
import { Editor } from "tui";
import { Loader } from "tui";
import { Markdown } from "tui";
import { Text } from "tui";
import { ProcessTerminal } from "tui";
import { TUI } from "tui";

// ─── 1. Bootstrap ──────────────────────────────────────────────────────────

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

// ─── 2. Welcome message ────────────────────────────────────────────────────

tui.addChild(
  new Text(
    "Welcome to Simple Chat!\n\n" +
    "Type your messages below. Type '/' for commands. Press Ctrl+C to exit.",
  ),
);

// ─── 3. Editor with autocomplete ───────────────────────────────────────────

const editor = new Editor(tui, /* your editorTheme */);

const autocompleteProvider = new CombinedAutocompleteProvider(
  [
    { name: "delete", description: "Delete the last message" },
    { name: "clear",  description: "Clear all messages" },
  ],
  process.cwd(),
  // pass a path to `fd` as third argument to enable fuzzy @ file search
);
editor.setAutocompleteProvider(autocompleteProvider);

tui.addChild(editor);
tui.setFocus(editor);

// ─── 4. Submission handler ─────────────────────────────────────────────────

let isResponding = false;

editor.onSubmit = (value: string) => {
  if (isResponding) return;

  const trimmed = value.trim();

  // Slash commands
  if (trimmed === "/delete") {
    const children = tui.children;
    if (children.length > 3) {
      children.splice(children.length - 2, 1);
    }
    tui.requestRender();
    return;
  }

  if (trimmed === "/clear") {
    const children = tui.children;
    children.splice(2, children.length - 3);
    tui.requestRender();
    return;
  }

  // Regular message
  if (!trimmed) return;

  isResponding = true;
  editor.disableSubmit = true;

  // Show user message
  const userMessage = new Markdown(value, 1, 1, /* your markdownTheme */);
  const children = tui.children;
  children.splice(children.length - 1, 0, userMessage);

  // Show loading spinner
  const loader = new Loader(
    tui,
    (s) => chalk.cyan(s),
    (s) => chalk.dim(s),
    "Thinking...",
  );
  children.splice(children.length - 1, 0, loader);
  tui.requestRender();

  // ── Replace this block with your LLM call ──
  setTimeout(() => {
    tui.removeChild(loader);

    const responseText = "I see what you mean.";
    const botMessage = new Markdown(responseText, 1, 1, /* your markdownTheme */);
    children.splice(children.length - 1, 0, botMessage);

    isResponding = false;
    editor.disableSubmit = false;
    tui.requestRender();
  }, 1000);
  // ───────────────────────────────────────────
};

// ─── 5. Start ──────────────────────────────────────────────────────────────

tui.start();
```

Run this file with `ts-node` or `tsx` and you get a working terminal chat interface — messages scroll up, the spinner appears while waiting, slash commands manipulate the history, and the editor offers autocomplete as you type.

## What the layout looks like at runtime

The `tui.children` array always reflects this order during a response:

```
index 0        → Text("Welcome to Simple Chat!")
index 1        → Markdown(user message)
index 2        → Loader("Thinking...")        ← removed when response arrives
index 3        → Editor                       ← always last
```

After the response arrives:

```
index 0        → Text("Welcome to Simple Chat!")
index 1        → Markdown(user message)
index 2        → Markdown(bot response)
index 3        → Editor
```

Every subsequent exchange appends two more components (user + bot) before the editor. The TUI's render loop redraws the entire list on each `requestRender()` call, calculating the height each component needs and scrolling the terminal output accordingly.

## Where to go from here

You have now assembled a complete terminal chat interface from raw building blocks. The `tui` package's job is done: it hands you a render loop, a key-input abstraction, a set of ready-made components, and an autocomplete system — you supply the layout and the response logic.

The next layer up is the coding agent that drives that response logic: it manages conversation state, routes LLM calls, compacts history, and coordinates tool use. That is where we are headed.

---

← Previous: [Built-In Components: Widgets, Layout, and ANSI Utilities](./built-in-components-and-layout.md) · Next: [AgentSession: The Core of the Coding Agent](../coding-agent/agent-session-core.md) →
