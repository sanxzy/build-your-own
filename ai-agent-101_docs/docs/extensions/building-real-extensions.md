---
title: "Building Real Extensions: Tools, State, Commands, and Hooks"
description: "A step-by-step walkthrough building five progressively richer extensions: safety hooks, stateful tools, custom compaction, pub/sub coordination, and dynamic resource discovery."
category: extensions
type: tutorial
tags: [extension, todo extension, permission gate, tool hook, state, slash command, renderer, session_start, session_before_compact, custom compaction, event bus, pub-sub, dynamic resources, coding-agent, examples, tool_call, resources_discover, ExtensionAPI, registerTool, registerCommand, session_tree, beforeToolCall]
keywords: [extension tutorial, safety hook, stateful extension, custom summariser, inter-extension communication, dynamic skills, extension state persistence, bash guard, confirmation prompt]
sources: [S79, S80, S81, S72, S84]
---

**TL;DR** — This chapter builds five complete extensions of increasing complexity, each teaching one new capability of the extension system. We start with a one-file safety hook that intercepts dangerous bash commands, move to a full stateful todo manager (tool + command + renderer + session replay), then replace the default summariser, wire two extensions together with a pub/sub bus, and finally contribute skills and themes dynamically at startup. By the end you will have touched every major surface of the extension API.

# Building Real Extensions: Tools, State, Commands, and Hooks

Before we dive in, two quick recaps:

- The **ExtensionAPI** (the object your extension's default export receives — called `ext` throughout this chapter) is the surface through which an extension registers tools, hooks events, registers commands, and reads session state. Covered in depth in [Extension API and Types](./extension-api-and-types.md).
- **Loading and lifecycle** — extensions are discovered on startup, each module's default export is called with a fresh `ExtensionAPI`, and their registrations are applied before the first turn. The full loading sequence is in [Loading and Running Extensions](./extension-loader-and-runner.md).

Now let's build something real.

---

## Step 1 — A Permission Gate: Intercept Dangerous Bash Commands

The agent can run arbitrary shell commands via the built-in `bash` tool. That's useful, but it creates a risk: a confused or overly-confident model might try `rm -rf ./src` or `sudo` something unexpected. We want a human checkpoint before any command that matches a set of danger patterns.

The problem is straightforward: we need to see every `bash` tool call *before* the tool actually runs, check the command string against a list of patterns, and either let it through or ask the user to confirm.

### How beforeToolCall works

When the agent decides to call a tool, it emits a `tool_call` event. Your handler receives the event and a context (`ctx`). If you return `{ block: true, reason: "..." }`, the call is cancelled and the reason is shown to the model. If you return `undefined` (or nothing), the call proceeds normally.

Here is the complete permission-gate extension:

```ts
// .xzy/extensions/permission-gate.ts
import type { ExtensionAPI } from "coding-agent";

export default function (ext: ExtensionAPI) {
  const dangerousPatterns = [
    /\brm\s+(-rf?|--recursive)/i,
    /\bsudo\b/i,
    /\b(chmod|chown)\b.*777/i,
  ];

  ext.on("tool_call", async (event, ctx) => {
    // Only intercept the bash tool; let everything else through
    if (event.toolName !== "bash") return undefined;

    const command = event.input.command as string;
    const isDangerous = dangerousPatterns.some((p) => p.test(command));

    if (isDangerous) {
      if (!ctx.hasUI) {
        // Non-interactive mode (e.g. headless/RPC): block by default, no prompt possible
        return { block: true, reason: "Dangerous command blocked (no UI for confirmation)" };
      }

      const choice = await ctx.ui.select(
        `Dangerous command:\n\n  ${command}\n\nAllow?`,
        ["Yes", "No"],
      );

      if (choice !== "Yes") {
        return { block: true, reason: "Blocked by user" };
      }
    }

    return undefined;
  });
}
```

Let's trace through the logic:

1. We register a handler for the `"tool_call"` event.
2. We immediately bail out (`return undefined`) if the tool being called is not `bash`. This is important — a `tool_call` fires for every tool, and we only care about one.
3. We test `event.input.command` against three patterns: recursive deletion, `sudo`, and wide-open `chmod`/`chown`.
4. If the command is dangerous, we check `ctx.hasUI`. In a headless or RPC session there is no terminal to prompt the user, so we block outright.
5. In interactive mode we call `ctx.ui.select()`, which presents the user with a choice. Only `"Yes"` allows the command through.

Notice that returning `undefined` is the signal for "proceed normally" — you do not have to explicitly allow a call, only explicitly block it.

### What this teaches

| Concept | How it's used here |
|---|---|
| `ext.on("tool_call", ...)` | Hook any tool invocation before it executes |
| `event.toolName` | Filter to a specific tool |
| `event.input` | Read the raw input arguments the model passed |
| `ctx.hasUI` | Detect interactive vs. headless mode |
| `ctx.ui.select()` | Present a choice prompt to the user |
| Return `{ block: true, reason }` | Cancel the tool call with a visible reason |
| Return `undefined` | Pass the tool call through unchanged |

---

## Step 2 — A Todo Extension: Tool + State + Command + Renderer

The permission gate was a one-concern extension. Now we want to build something with multiple moving parts:

- A **custom tool** the LLM can call to manage a todo list.
- **Persisted state** that survives branch switches without writing any external file.
- A **/todos slash command** so the user can view the list at any time.
- A **custom renderer** so the TUI shows todo operations with rich formatting instead of raw JSON.

The challenge is state. If we keep todos in a plain JavaScript array, the agent's history branching will break things: switching to a different branch in the conversation tree leaves the in-memory list pointing at the wrong history. We need a smarter approach.

### Storing state in session entries, not in memory

The key insight: instead of treating the JavaScript variable as the source of truth, we make the **session history** the source of truth. Every tool result from the `todo` tool carries a `details` field containing the current todo list at that point in time. When we need the current state, we replay the session entries from the beginning of the current branch, picking up the last `todo` tool result we find.

This means branching works for free — when the session switches to a different branch, we replay *that* branch's history.

Here are the types we'll use:

```ts
// The data stored in each tool result's details field
interface TodoDetails {
  action: "list" | "add" | "toggle" | "clear";
  todos: Todo[];
  nextId: number;
  error?: string;
}

interface Todo {
  id: number;
  text: string;
  done: boolean;
}
```

### Rebuilding state from the session

The `reconstructState` function walks `ctx.sessionManager.getBranch()` — which returns all session entries on the current branch in order — and finds every `toolResult` message for our `todo` tool. It takes the last one it finds, because later entries represent more recent state:

```ts
// Simplified view of reconstructState
const reconstructState = (ctx: ExtensionContext) => {
  todos = [];
  nextId = 1;

  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type !== "message") continue;
    const msg = entry.message;
    if (msg.role !== "toolResult" || msg.toolName !== "todo") continue;

    const details = msg.details as TodoDetails | undefined;
    if (details) {
      todos = details.todos;
      nextId = details.nextId;
    }
  }
};
```

We call this function on two events: `session_start` (when the agent first opens) and `session_tree` (when the user switches branches). Both pass the current `ctx`, so the replay always runs against the right branch.

```ts
ext.on("session_start", async (_event, ctx) => reconstructState(ctx));
ext.on("session_tree",  async (_event, ctx) => reconstructState(ctx));
```

### Registering the todo tool

`ext.registerTool()` takes a tool descriptor. The LLM sees the `name`, `description`, and `parameters`; the `execute` function runs when the model calls the tool; `renderCall` and `renderResult` are optional renderers for the TUI.

The `parameters` field uses a TypeBox schema to describe what the LLM should pass:

```ts
import { StringEnum } from "llm-toolkit";
import { Type } from "typebox";

const TodoParams = Type.Object({
  action: StringEnum(["list", "add", "toggle", "clear"] as const),
  text: Type.Optional(Type.String({ description: "Todo text (for add)" })),
  id: Type.Optional(Type.Number({ description: "Todo ID (for toggle)" })),
});
```

The `execute` function handles each action and always returns both `content` (what the model reads back) and `details` (our state snapshot for the history):

```ts
// Simplified view of the execute handler
async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
  switch (params.action) {
    case "add": {
      if (!params.text) {
        return {
          content: [{ type: "text", text: "Error: text required for add" }],
          details: { action: "add", todos: [...todos], nextId, error: "text required" },
        };
      }
      const newTodo: Todo = { id: nextId++, text: params.text, done: false };
      todos.push(newTodo);
      return {
        content: [{ type: "text", text: `Added todo #${newTodo.id}: ${newTodo.text}` }],
        details: { action: "add", todos: [...todos], nextId },
      };
    }
    // ... list, toggle, clear handled similarly
  }
}
```

The `details` field is the critical piece. It is stored verbatim alongside the tool result in the session history, which is exactly what `reconstructState` reads back.

### Custom renderers: renderCall and renderResult

Without a custom renderer, the TUI would display tool calls and their results as raw text. With `renderCall` and `renderResult`, you control how each operation appears. Both functions return a `Text` object — a simple renderable from the `tui` package:

```ts
renderCall(args, theme, _context) {
  let text = theme.fg("toolTitle", theme.bold("todo ")) + theme.fg("muted", args.action);
  if (args.text) text += ` ${theme.fg("dim", `"${args.text}"`)}`;
  if (args.id !== undefined) text += ` ${theme.fg("accent", `#${args.id}`)}`;
  return new Text(text, 0, 0);
},

renderResult(result, { expanded }, theme, _context) {
  const details = result.details as TodoDetails | undefined;
  if (!details) {
    const text = result.content[0];
    return new Text(text?.type === "text" ? text.text : "", 0, 0);
  }
  if (details.error) {
    return new Text(theme.fg("error", `Error: ${details.error}`), 0, 0);
  }
  // ... render based on details.action
}
```

The `theme` object carries colour helpers (`theme.fg("accent", ...)`, `theme.bold(...)`) so your renderer respects whatever theme the user has configured.

### Registering the /todos slash command

Slash commands are user-facing — the user types `/todos` in the input, and the handler runs. Here we open a full-screen TUI overlay (`ctx.ui.custom`) showing the current list:

```ts
ext.registerCommand("todos", {
  description: "Show all todos on the current branch",
  handler: async (_args, ctx) => {
    if (ctx.mode !== "tui") {
      ctx.ui.notify("/todos requires interactive mode", "error");
      return;
    }

    await ctx.ui.custom<void>((_tui, theme, _kb, done) => {
      return new TodoListComponent(todos, theme, () => done());
    });
  },
});
```

`ctx.ui.custom` accepts a factory that returns a component with a `render(width)` method and a `handleInput(data)` method. The `done` callback closes the overlay when called. The `TodoListComponent` in the source handles `Escape`/`Ctrl+C` to dismiss and renders the list with a header, completion count, and each item's check state.

### What this teaches

| Concept | How it's used here |
|---|---|
| `ext.registerTool({ name, execute, renderCall, renderResult })` | Register a new tool the LLM can call |
| Tool result `details` field | Store structured state alongside each result |
| `ctx.sessionManager.getBranch()` | Read all entries on the current branch |
| `ext.on("session_start")` / `"session_tree"` | Re-run state reconstruction after load or branch switch |
| `ext.registerCommand(name, { handler })` | Register a user-facing slash command |
| `ctx.ui.custom(factory)` | Open a full-screen custom TUI overlay |
| `ctx.mode` | Check whether the agent is running in TUI vs. other modes |

---

## Step 3 — Custom Compaction: Override the Default Summariser

Compaction is what happens when the conversation history grows too long for the context window — the agent summarises older turns and trims them. By default the agent handles this automatically.

<!-- GAP: exact default token threshold and default summariser model — source silent -->

But you might want a different summarisation model (cheaper, faster, or better for your domain), or a different prompt that captures more detail. The `session_before_compact` event lets you intercept the compaction flow and return your own summary.

If you return nothing from your handler, the default compaction runs. If you return a `{ compaction: { summary, firstKeptEntryId, tokensBefore } }` object, that becomes the compaction result.

### What the event gives you

The `session_before_compact` event's `preparation` field contains everything you need:

| Field | What it is |
|---|---|
| `messagesToSummarize` | The older messages that will be discarded |
| `turnPrefixMessages` | The most recent turn prefix kept for context |
| `tokensBefore` | Total token count before compaction |
| `firstKeptEntryId` | The entry id at which recent history starts (kept verbatim after the summary) |
| `previousSummary` | Any summary from a prior compaction round, for continuity |

The event also carries a `signal` — an `AbortSignal` you should pass to any LLM call you make, so the compaction can be cancelled if the user interrupts.

### A full custom compaction handler

```ts
// .xzy/extensions/custom-compaction.ts
import { complete } from "llm-toolkit";
import type { ExtensionAPI } from "coding-agent";
import { convertToLlm, serializeConversation } from "coding-agent";

export default function (ext: ExtensionAPI) {
  ext.on("session_before_compact", async (event, ctx) => {
    ctx.ui.notify("Custom compaction triggered", "info");

    const { preparation, signal } = event;
    const {
      messagesToSummarize,
      turnPrefixMessages,
      tokensBefore,
      firstKeptEntryId,
      previousSummary,
    } = preparation;

    // Pick a model for summarisation — here we try Gemini Flash
    const model = ctx.modelRegistry.find("google", "gemini-2.5-flash");
    if (!model) {
      ctx.ui.notify("Model not found, using default compaction", "warning");
      return; // fall back to default
    }

    const auth = await ctx.modelRegistry.getApiKeyAndHeaders(model);
    if (!auth.ok || !auth.apiKey) {
      ctx.ui.notify("Auth failed, using default compaction", "warning");
      return;
    }

    // Combine all messages for a full summary
    const allMessages = [...messagesToSummarize, ...turnPrefixMessages];

    ctx.ui.notify(
      `Summarising ${allMessages.length} messages (${tokensBefore.toLocaleString()} tokens) with ${model.id}…`,
      "info",
    );

    const conversationText = serializeConversation(convertToLlm(allMessages));
    const previousContext = previousSummary
      ? `\n\nPrevious session summary:\n${previousSummary}`
      : "";

    const summaryMessages = [
      {
        role: "user" as const,
        content: [
          {
            type: "text" as const,
            text: `Summarise this conversation comprehensively.${previousContext}

Cover: main goals, key decisions and rationale, code changes, current state,
open questions, and next steps. The summary will replace the entire conversation
history, so include everything needed to continue effectively.

Format as structured markdown.

<conversation>
${conversationText}
</conversation>`,
          },
        ],
        timestamp: Date.now(),
      },
    ];

    try {
      const response = await complete(
        model,
        { messages: summaryMessages },
        { apiKey: auth.apiKey, headers: auth.headers, maxTokens: 8192, signal },
      );

      const summary = response.content
        .filter((c): c is { type: "text"; text: string } => c.type === "text")
        .map((c) => c.text)
        .join("\n");

      if (!summary.trim()) {
        if (!signal.aborted) ctx.ui.notify("Empty summary, using default", "warning");
        return;
      }

      return {
        compaction: { summary, firstKeptEntryId, tokensBefore },
      };
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      ctx.ui.notify(`Compaction failed: ${message}`, "error");
      return; // fall back to default on error
    }
  });
}
```

There are several patterns here worth calling out:

- **Three fallback paths, all safe.** If the model is not found, auth fails, or the LLM throws, we return `undefined` in each case. The default compaction runs. The user is notified, but the session is never left in a broken state.
- **`signal` threading.** We pass the abort signal to `complete()`. If the user cancels while compaction is in progress, the LLM call is abandoned cleanly.
- **`convertToLlm` + `serializeConversation`** are exported helpers from `coding-agent` that convert session entries into a plain-text conversation transcript the summariser can read.
- **`firstKeptEntryId`** tells the session manager where recent history begins. We pass it through unchanged; recent messages are kept verbatim after the summary.

### What this teaches

| Concept | How it's used here |
|---|---|
| `ext.on("session_before_compact", ...)` | Intercept the compaction flow |
| `event.preparation` | Access the messages, tokens, and previous summary |
| Return `undefined` | Fall back to the default summariser |
| Return `{ compaction: { summary, firstKeptEntryId, tokensBefore } }` | Provide a custom summary |
| `ctx.modelRegistry.find(provider, modelId)` | Resolve a model by provider and ID |
| `ctx.modelRegistry.getApiKeyAndHeaders(model)` | Retrieve auth credentials for a model |
| `event.signal` | AbortSignal to cancel in-flight LLM calls |

---

## Step 4 — EventBus: Coordinating Two Extensions

So far each extension has lived in its own world. But sometimes two extensions need to talk. An analytics extension might want to know when a todo is added; a test-runner extension might want to notify a status-bar extension when tests pass.

The mechanism for this is the **EventBus** — a lightweight pub/sub channel that any extension can emit to and subscribe from. It is separate from the agent's lifecycle event system (which is for extension-to-agent communication); the EventBus is for extension-to-extension communication.

### The EventBus API

```ts
// Exported from coding-agent's core
export interface EventBus {
  emit(channel: string, data: unknown): void;
  on(channel: string, handler: (data: unknown) => void): () => void;
}
```

`emit(channel, data)` broadcasts a value to all current subscribers on that channel. `on(channel, handler)` registers a subscriber and returns an **unsubscribe function** — call the returned function to remove that handler. Channel names are plain strings; pick something unique to avoid collisions with other extensions.

The full `EventBusController` (what `createEventBus()` returns internally) adds a `clear()` method that removes all listeners; this is called by the session manager during cleanup.

Error handling is built in: if a subscriber's handler throws or rejects, the error is logged but does not propagate to the emitter or crash other subscribers.

### Using the EventBus between two extensions

Extensions receive a reference to the shared `EventBus` on their context (`ctx.eventBus`).

<!-- GAP: ctx.eventBus access from ExtensionContext — sources show EventBus types and createEventBus factory, but do not confirm the exact field name on ctx; source silent -->

Here is a conceptual example of two extensions coordinating. Extension A emits when a significant event occurs; Extension B subscribes and reacts:

```ts
// extension-a.ts — emits a custom event
import type { ExtensionAPI } from "coding-agent";

export default function (ext: ExtensionAPI) {
  ext.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash") {
      // Notify other extensions that a bash call is happening
      ctx.eventBus.emit("bash:invoked", { command: event.input.command });
    }
    return undefined;
  });
}
```

```ts
// extension-b.ts — subscribes and reacts
import type { ExtensionAPI } from "coding-agent";

export default function (ext: ExtensionAPI) {
  ext.on("session_start", async (_event, ctx) => {
    const unsubscribe = ctx.eventBus.on("bash:invoked", (data) => {
      const { command } = data as { command: string };
      ctx.ui.notify(`Shell command running: ${command}`, "info");
    });

    // Store unsubscribe if you need to clean up later
    void unsubscribe;
  });
}
```

The `on()` return value is the cleanup function. You do not have to call it — the bus is cleared between sessions automatically via `clear()`. But if you want to stop receiving events mid-session (e.g. after a one-time event), calling the returned function unregisters your handler.

### What this teaches

| Concept | How it's used here |
|---|---|
| `EventBus.emit(channel, data)` | Publish an event to a named channel |
| `EventBus.on(channel, handler)` | Subscribe to a named channel; returns unsubscribe fn |
| Unsubscribe function | Remove a handler from a channel |
| `createEventBus()` | Factory that creates a bus backed by `EventEmitter` |
| Error isolation | Each handler's errors are caught independently |

---

## Step 5 — Dynamic Resources: Contributing Skills, Prompts, and Themes

The final extension surface we'll cover is **resource discovery**. When the agent starts, it collects skills (markdown files that teach the agent new behaviours), prompt files, and theme files from a set of known directories. An extension can add to those directories dynamically by handling the `resources_discover` event.

This is useful when an extension ships its own bundled skills or themes alongside its code — instead of requiring the user to configure extra paths, the extension contributes them automatically.

### The resources_discover event

Your handler returns an object with up to three path arrays:

```ts
// Return shape from a resources_discover handler
{
  skillPaths:  string[];   // paths to skill .md files
  promptPaths: string[];   // paths to prompt .md files
  themePaths:  string[];   // paths to theme .json files
}
```

Any key you omit is ignored. You can return just `skillPaths`, just `themePaths`, or any combination.

### A complete dynamic-resources extension

```ts
// .xzy/extensions/dynamic-resources/index.ts
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import type { ExtensionAPI } from "coding-agent";

// Resolve paths relative to this extension file
const baseDir = dirname(fileURLToPath(import.meta.url));

export default function (ext: ExtensionAPI) {
  ext.on("resources_discover", () => {
    return {
      skillPaths:  [join(baseDir, "SKILL.md")],
      promptPaths: [join(baseDir, "dynamic.md")],
      themePaths:  [join(baseDir, "dynamic.json")],
    };
  });
}
```

A few things to notice:

- We use `import.meta.url` + `fileURLToPath` + `dirname` to resolve paths relative to this extension file. This works regardless of the current working directory when the agent is launched.
- The handler is synchronous — you return the path object directly (no `async` needed).
- The resources must exist on disk at the paths you return; the agent will attempt to load them.

If you ship an extension as an npm package (or a folder that might be installed anywhere), this pattern makes the extension fully self-contained: its bundled skill file travels with it and is discovered automatically.

### What this teaches

| Concept | How it's used here |
|---|---|
| `ext.on("resources_discover", ...)` | Contribute paths for skill, prompt, and theme files |
| `skillPaths` / `promptPaths` / `themePaths` | The three resource types an extension can contribute |
| `import.meta.url` + path resolution | Resolve extension-relative paths robustly |
| Self-contained extension packaging | Bundle resources with the extension, no user config needed |

---

## Putting It All Together

Each extension we built demonstrates a different layer of the system:

| Extension | Event(s) | New capability |
|---|---|---|
| Permission gate | `tool_call` | Intercept and optionally block any tool invocation |
| Todo | `session_start`, `session_tree`, `registerTool`, `registerCommand` | Custom tool + state replay + slash command + custom renderer |
| Custom compaction | `session_before_compact` | Replace the default summariser |
| EventBus coordination | *(bus-level, not lifecycle events)* | Extension-to-extension pub/sub |
| Dynamic resources | `resources_discover` | Contribute skills, prompts, and themes at startup |

These five patterns cover most of what production extensions need. You can combine them freely — the todo extension could emit `"todo:added"` events on the bus; a reporting extension could subscribe and log them; a custom compaction extension could include those logs in its summary.

The extension system is designed around composition: each extension does one thing, and they cooperate through the shared event system and bus.

---

← Previous: [Loading and Running Extensions: Discovery, Isolation, and Lifecycle](./extension-loader-and-runner.md) · Next: [Multi-Agent Orchestration](./multi-agent-orchestration.md) →
