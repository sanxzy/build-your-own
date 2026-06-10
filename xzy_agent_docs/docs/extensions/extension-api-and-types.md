---
title: "The Extension API: Handlers, Context, and Events"
description: "Learn the full ExtensionAPI surface — handler registration, context object, ToolDefinition, and all lifecycle event types — by building a minimal working extension."
category: extensions
type: tutorial
tags: [ExtensionAPI, extension, handler, context, ToolDefinition, event types, provider config, registerTool, registerProvider, session_start, tool_call, extensions, coding-agent, plugin, ExtensionHandler, ExtensionContext, ExtensionCommandContext, ExtensionFactory, registerCommand, registerShortcut, on, lifecycle, ResourcesDiscoverEvent, AgentStartEvent, AgentEndEvent, TurnStartEvent, TurnEndEvent, MessageStartEvent, MessageUpdateEvent, MessageEndEvent, ToolExecutionStartEvent, ToolExecutionEndEvent, ModelSelectEvent, InputEvent, ToolCallEvent, ToolResultEvent]
keywords: [extension system, custom tools, event subscription, provider registration, defineTool, TypeBox, TSchema, ExtensionRuntime, ExtensionMode, WidgetPlacement, sendUserMessage, appendEntry, setSessionName, registerFlag, getFlag]
sources: [S48, S77, S78]
---

**TL;DR** — The extension system is the agent's primary extensibility surface. An extension is a single TypeScript file with a default-export function that receives one argument — the `ExtensionAPI` object — and uses it to register tools, commands, keyboard shortcuts, and lifecycle event handlers. This chapter walks through the full API: the extension shape, the `ExtensionContext` available inside every handler, the `ToolDefinition` contract, and a complete map of every lifecycle event the system emits. By the end you will have a running hello-world extension and a solid mental map for writing more complex ones.

# The Extension API: Handlers, Context, and Events

Every time we want users to add a new tool, intercept a bash command, or hook into the agent's turn lifecycle, we face a choice: ask them to fork the codebase and edit core files, or give them a stable contract they can implement once and drop in. The extension system takes the second path. It lets anyone add capabilities in TypeScript, without touching core, and without requiring a rebuild.

This chapter opens the extensions layer. We'll start from first principles — the shape of an extension file — then build up to the full API surface and the complete lifecycle event map.

## What an extension is

Before we touch any types, let's establish the invariant: **an extension is a TypeScript module whose default export is a function**. That function receives one argument — an object implementing `ExtensionAPI` — and uses it to register everything the extension needs. The function may be synchronous or async.

```ts
// The complete shape of an extension module
export default function (ext: ExtensionAPI): void | Promise<void> {
  // register tools, commands, event handlers here
}
```

The type alias for this shape is `ExtensionFactory`:

```ts
// From types.ts (S48)
export type ExtensionFactory = (ext: ExtensionAPI) => void | Promise<void>;
```

The single parameter is the extension API object; we'll name it `ext` in our examples throughout this chapter. What matters is the type — `ExtensionAPI` — which is the contract we'll explore in full.

> **Prerequisites.** This chapter builds on two earlier concepts:
>
> - **AgentSession** — a live session holds the conversation history, session tree, and compaction state that extensions can observe and react to. See [AgentSession](../coding-agent/agent-session-core.md) for depth.
> - **Built-in tools / AgentTool shape** — the agent ships with tools for reading files, running bash commands, and editing files. Extensions register their own tools using the same shape. See [Built-in Tools](../coding-agent/built-in-tools.md) for how those are structured.

## The hello-world extension

Let's start with the smallest possible extension: one that registers a single tool the LLM can call. This is the `hello.ts` example (S78) from the extensions directory.

```ts
// hello.ts — minimal custom tool extension
// Genericised from the source example (S78)

import { Type } from "typebox";            // JSON-schema builder for tool parameters
import { defineTool, type ExtensionAPI } from "coding-agent";

// Step 1: Define the tool in isolation, outside the factory function.
// defineTool() is a pass-through helper that preserves TypeScript's
// generic parameter inference — without it, params would widen to `unknown`.
const helloTool = defineTool({
  name: "hello",                           // The name the LLM uses in tool calls
  label: "Hello",                          // Human-readable label shown in the UI
  description: "A simple greeting tool",  // What the LLM reads to decide when to call it
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),

  async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },   // Persisted in the session entry
    };
  },
});

// Step 2: Export the factory. The agent loader calls this and passes the API object.
export default function (ext: ExtensionAPI) {
  ext.registerTool(helloTool);
}
```

Three things to notice:

1. `defineTool()` is called **outside** the factory function. The tool definition is static — we build it once and reuse it. The factory is called once per session; the tool object does not need to be re-created.
2. `parameters` uses `Type.Object(...)` from the `typebox` library. TypeBox generates JSON Schema at runtime, which the agent sends to the LLM as the tool's parameter schema. The LLM uses that schema to produce valid arguments.
3. `execute` receives `(toolCallId, params, signal, onUpdate, ctx)`. We ignore four of them in this minimal case. We'll cover each when we need it.

### Running the extension

Extensions can be loaded two ways (from S77):

```bash
# Inline: pass on the command line
xzy --extension hello.ts

# Persistent: drop into the extensions directory for auto-discovery
cp hello.ts ~/.xzy/agent/extensions/
```

Once loaded, the LLM sees `hello` in its tool list and can call it with `{ "name": "Alice" }`. The result `Hello, Alice!` is returned as a text content block.

## The ExtensionAPI surface

Now that we have a working extension, let's look at the full `ExtensionAPI` interface. It is the only object the factory receives, and it is how the extension communicates with the rest of the system.

### Registering things

The API exposes four registration methods:

| Method | What it registers |
|---|---|
| `registerTool(tool)` | A tool the LLM can call |
| `registerCommand(name, options)` | A slash command available to the user |
| `registerShortcut(shortcut, options)` | A keyboard shortcut in interactive mode |
| `registerFlag(name, options)` | A CLI flag the user can pass on startup |

And a paired getter for flags:

```ts
getFlag(name: string): boolean | string | undefined
```

You call `registerFlag` during the factory function to declare the flag and its default, then call `getFlag` at any later point — in a `session_start` handler, for example — to read the value the user passed on the command line.

### Subscribing to events

The `on()` method is how an extension hooks into the agent's lifecycle. Every call wires a handler function to one named event. The extension system dispatches events at well-defined moments; the handler receives a typed event object and the current `ExtensionContext`.

```ts
ext.on("session_start", async (event, ctx) => {
  // event: SessionStartEvent  — typed to the specific event
  // ctx:   ExtensionContext    — what the extension can see and do
});
```

We'll map every available event in [The lifecycle event map](#the-lifecycle-event-map) below.

### Actions

Beyond registration and event subscription, `ExtensionAPI` also exposes a set of action methods that can be called at any time after the factory returns:

| Method | What it does |
|---|---|
| `sendMessage(message, options?)` | Inject a custom message into the session |
| `sendUserMessage(content, options?)` | Send a user message that triggers a turn |
| `appendEntry(customType, data?)` | Append a custom entry to the session for state persistence (not sent to the LLM) |
| `setSessionName(name)` | Set the display name shown in the session selector |
| `getSessionName()` | Read back the current session name |
| `setLabel(entryId, label)` | Attach a navigation label to a session entry |
| `exec(command, args, options?)` | Execute a shell command |
| `getActiveTools()` | List currently active tool names |
| `getAllTools()` | List all tools with schemas and metadata |
| `setActiveTools(toolNames)` | Replace the active tool set |
| `getCommands()` | List available slash commands |
| `setModel(model)` | Switch the current LLM model |
| `getThinkingLevel()` | Read the current thinking level |
| `setThinkingLevel(level)` | Change the thinking level |
| `registerProvider(name, config)` | Register or override a model provider |
| `unregisterProvider(name)` | Remove a previously registered provider |
| `events` | Shared `EventBus` for inter-extension communication |

The `events` property is an `EventBus` — a lightweight pub/sub channel that lets multiple loaded extensions communicate with each other without tight coupling.

## The ExtensionContext

Every event handler receives two arguments: the event itself and an `ExtensionContext` object. The context is what the extension can *see and do* during a handler invocation. Let's look at what it exposes.

```ts
// Simplified view of ExtensionContext (from S48)
interface ExtensionContext {
  ui: ExtensionUIContext;          // Show dialogs, set status, render widgets
  mode: ExtensionMode;             // "tui" | "rpc" | "json" | "print"
  hasUI: boolean;                  // true in TUI and RPC modes
  cwd: string;                     // Current working directory
  sessionManager: ReadonlySessionManager;  // Read session history
  modelRegistry: ModelRegistry;    // Resolve API keys
  model: Model<any> | undefined;   // The currently selected model
  isIdle(): boolean;               // True when not streaming
  signal: AbortSignal | undefined; // Active when streaming
  abort(): void;                   // Cancel the current operation
  hasPendingMessages(): boolean;   // Messages queued but not yet processed
  shutdown(): void;                // Gracefully exit the agent
  getContextUsage(): ContextUsage | undefined;  // Token usage info
  compact(options?: CompactOptions): void;       // Trigger compaction
  getSystemPrompt(): string;       // The current effective system prompt
}
```

A few things to notice:

- `mode` tells you which runtime is active. Guard terminal-only UI calls (like `setWidget`) with `if (ctx.mode === "tui")` so the extension behaves correctly when the agent runs in non-interactive modes.
- `hasUI` is a convenient boolean that is `true` in `"tui"` and `"rpc"` modes — both support dialogs. Use it when all you care about is "can I show a dialog?".
- `sessionManager` is read-only from handlers. The full `SessionManager` (writable) is only available in command handlers.

### ExtensionCommandContext

Command handlers (registered via `registerCommand`) receive a richer context: `ExtensionCommandContext`. It extends `ExtensionContext` with session-control methods that are only safe to call from user-initiated commands — not from event handlers that fire mid-agent-loop.

```ts
// Additional methods available in command handlers (from S48)
interface ExtensionCommandContext extends ExtensionContext {
  getSystemPromptOptions(): BuildSystemPromptOptions;
  waitForIdle(): Promise<void>;
  newSession(options?): Promise<{ cancelled: boolean }>;
  fork(entryId, options?): Promise<{ cancelled: boolean }>;
  navigateTree(targetId, options?): Promise<{ cancelled: boolean }>;
  switchSession(sessionPath, options?): Promise<{ cancelled: boolean }>;
  reload(): Promise<void>;
}
```

The additional methods let a command wait for the agent to finish (`waitForIdle`), open a new session, fork from an entry point, navigate the session tree, switch to a different session file, or trigger a reload of extensions and skills. These are the building blocks for commands like `/handoff` or `/plan`.

### The UI context

`ctx.ui` gives an extension access to the agent's user interface. The full `ExtensionUIContext` interface (from S48) includes:

| Method | What it does |
|---|---|
| `select(title, options, opts?)` | Show a selection dialog; returns chosen string or undefined |
| `confirm(title, message, opts?)` | Show a yes/no confirmation dialog |
| `input(title, placeholder?, opts?)` | Show a text input dialog |
| `notify(message, type?)` | Show a notification (`"info"`, `"warning"`, `"error"`) |
| `setStatus(key, text)` | Set a status key in the footer; pass `undefined` to clear |
| `setWorkingMessage(message?)` | Change the streaming "working" label |
| `setWorkingVisible(visible)` | Show or hide the streaming loader row |
| `setWorkingIndicator(options?)` | Customise the spinner frames and interval |
| `setHiddenThinkingLabel(label?)` | Change the label on collapsed thinking blocks |
| `setWidget(key, content, options?)` | Set a widget above or below the editor |
| `setFooter(factory)` | Replace the built-in footer component |
| `setHeader(factory)` | Replace the built-in header component |
| `setTitle(title)` | Set the terminal window/tab title |
| `pasteToEditor(text)` | Paste text into the input editor |
| `setEditorText(text)` | Set the editor content programmatically |
| `getEditorText()` | Read the current editor content |
| `editor(title, prefill?)` | Open a multi-line editor; returns edited text |
| `addAutocompleteProvider(factory)` | Stack additional autocomplete behavior |
| `setEditorComponent(factory)` | Replace the editor component entirely |
| `custom(factory, options?)` | Show a fully custom component with keyboard focus |
| `theme` | The current `Theme` object for styling |
| `getAllThemes()` | List available themes with names and paths |
| `getTheme(name)` | Load a theme by name without switching |
| `setTheme(theme)` | Switch to a theme by name or `Theme` object |
| `getToolsExpanded()` | Read the tool output expansion state |
| `setToolsExpanded(expanded)` | Set tool output expansion state |

Dialog methods (`select`, `confirm`, `input`) accept an optional `ExtensionUIDialogOptions` argument:

```ts
interface ExtensionUIDialogOptions {
  signal?: AbortSignal;  // Programmatically dismiss the dialog
  timeout?: number;      // Auto-dismiss after N milliseconds (with live countdown)
}
```

Widgets can be placed in two positions:

```ts
type WidgetPlacement = "aboveEditor" | "belowEditor";  // defaults to "aboveEditor"
```

`onTerminalInput` registers a raw terminal input listener (interactive mode only) and returns an unsubscribe function. The handler can return `{ consume: true }` to prevent the keystroke from reaching the normal input pipeline, or `{ data: "<replacement>" }` to substitute different data.

## The ToolDefinition shape

We saw `defineTool()` in the hello-world example. Let's look at the full `ToolDefinition` interface now, because every optional field controls something meaningful.

```ts
// From S48 — full ToolDefinition interface (simplified for readability)
interface ToolDefinition<TParams extends TSchema = TSchema, TDetails = unknown, TState = any> {
  name: string;               // ID used in LLM tool calls
  label: string;              // Human-readable label for the UI
  description: string;        // Description the LLM reads to decide when to call the tool
  promptSnippet?: string;     // One-line snippet for the "Available tools" section in the system prompt
  promptGuidelines?: string[];// Bullet points appended to the system prompt Guidelines section
  parameters: TParams;        // TypeBox schema — becomes the JSON Schema sent to the LLM

  renderShell?: "default" | "self";       // "self" = tool renders its own UI framing
  executionMode?: "sequential" | "parallel"; // Override parallel/sequential execution
  prepareArguments?: (args: unknown) => Static<TParams>; // Pre-validate argument shim

  execute(
    toolCallId: string,
    params: Static<TParams>,
    signal: AbortSignal | undefined,
    onUpdate: AgentToolUpdateCallback<TDetails> | undefined,
    ctx: ExtensionContext,
  ): Promise<AgentToolResult<TDetails>>;

  renderCall?: (args, theme, context) => Component;   // Custom call UI
  renderResult?: (result, options, theme, context) => Component; // Custom result UI
}
```

Walk through the fields you'll care about most:

- **`name`** — must be unique across all active tools. The LLM uses exactly this string in its tool calls.
- **`description`** — this is prompt text. Write it the way you'd describe the tool to the LLM in a system prompt bullet point. Be specific.
- **`promptSnippet`** — if provided, appears in the "Available tools" section of the default system prompt. Omitting it hides the tool from that section (the tool is still callable, just not mentioned there).
- **`promptGuidelines`** — bullet points added to the Guidelines section of the system prompt when this tool is active. Use this to give the LLM rules about when and how to call your tool.
- **`executionMode`** — `"sequential"` forces the tool to run one at a time with other tool calls; `"parallel"` allows concurrent execution. When omitted, the agent's default applies.
- **`prepareArguments`** — a compatibility shim: if the LLM sends arguments that don't quite match the schema (e.g., string where a number is expected), this function can coerce them before schema validation. Useful when targeting older models.
- **`onUpdate`** — the `execute` function receives `onUpdate` as its fourth argument. Call it with partial results during long-running operations to stream progress to the UI. Pass `undefined` updates when you have nothing to show yet.

The three type parameters — `TParams`, `TDetails`, `TState` — control TypeScript's inference:

| Parameter | Role |
|---|---|
| `TParams` | The TypeBox schema type; inferred from `parameters` |
| `TDetails` | The type of `result.details` returned from `execute` |
| `TState` | Shared renderer state for the tool's UI components |

## Registering a provider

Beyond tools and commands, extensions can register or override model providers via `registerProvider`. This is how you connect the agent to a proxy, an on-premises endpoint, or a model the built-in provider list doesn't include.

```ts
// Register a new provider pointing at a proxy (from S48 example, genericised)
ext.registerProvider("my-proxy", {
  baseUrl: "https://proxy.example.com",
  apiKey: "$PROXY_API_KEY",    // Env-var interpolation: $VAR or ${VAR} or !command
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384,
    },
  ],
});
```

The `ProviderConfig` interface covers every registration scenario:

```ts
// From S48 — ProviderConfig
interface ProviderConfig {
  name?: string;         // Display name in the UI
  baseUrl?: string;      // API endpoint
  apiKey?: string;       // Literal, $ENV_VAR, ${ENV_VAR}, or !command
  api?: Api;             // API type (e.g., "anthropic-messages", "openai-responses")
  headers?: Record<string, string>;   // Custom request headers
  authHeader?: boolean;  // Add Authorization: Bearer <apiKey> header automatically
  models?: ProviderModelConfig[];     // Full model list (replaces existing for this provider)
  streamSimple?: ...;    // Custom streaming handler for non-standard APIs
  oauth?: { ... };       // OAuth login/refresh/getApiKey for /login support
}
```

Two usage modes:

- **`models` provided** — replaces all models for this provider. Use when registering a new provider from scratch.
- **Only `baseUrl` provided** — overrides the URL for an existing provider's models. Use to point a built-in provider at a local proxy.

To remove a provider:

```ts
ext.unregisterProvider("my-proxy");
```

Both `registerProvider` and `unregisterProvider` take effect immediately when called after the initial load phase, so you can call them from command handlers or event callbacks without a `/reload`.

## The lifecycle event map

Now we can look at the complete set of events an extension can subscribe to. This is the map you'll return to every time you write a new handler. Events are dispatched in the order shown; handlers are called in registration order when multiple extensions subscribe to the same event.

### Resource and session events

These fire around the session lifecycle — startup, switches, compaction, and shutdown.

| Event type | When it fires | Result type | Can block? |
|---|---|---|---|
| `resources_discover` | After `session_start` — extensions return extra resource paths | `ResourcesDiscoverResult` | No |
| `session_start` | Session started, loaded, or reloaded | — | No |
| `session_before_switch` | Before switching to another session file | `SessionBeforeSwitchResult` | Yes (`cancel: true`) |
| `session_before_fork` | Before forking from an entry | `SessionBeforeForkResult` | Yes |
| `session_before_compact` | Before context compaction | `SessionBeforeCompactResult` | Yes |
| `session_compact` | After compaction completes | — | No |
| `session_shutdown` | Before the extension runtime is torn down | — | No |
| `session_before_tree` | Before navigating the session tree | `SessionBeforeTreeResult` | Yes |
| `session_tree` | After navigating the session tree | — | No |

`session_start` carries a `reason` field — `"startup" | "reload" | "new" | "resume" | "fork"` — so the handler can behave differently depending on whether this is the first load or a resume. It also carries `previousSessionFile` for `"new"`, `"resume"`, and `"fork"` reasons.

`resources_discover` fires after `session_start`. Its handler should return a `ResourcesDiscoverResult`:

```ts
interface ResourcesDiscoverResult {
  skillPaths?: string[];   // Additional skill directories
  promptPaths?: string[];  // Additional prompt file directories
  themePaths?: string[];   // Additional theme directories
}
```

`session_shutdown` carries a `reason` — `"quit" | "reload" | "new" | "resume" | "fork"` — and `targetSessionFile` when the shutdown is due to a session switch.

### Agent and turn events

These fire around the LLM call cycle — before the agent starts, each turn, and when it finishes.

| Event type | When it fires | Result type | Can modify? |
|---|---|---|---|
| `before_agent_start` | After user submits prompt, before agent loop | `BeforeAgentStartEventResult` | Yes (system prompt, inject message) |
| `context` | Before each LLM call — sees the full message list | `ContextEventResult` | Yes (replace messages) |
| `before_provider_request` | Before the HTTP request is sent | `BeforeProviderRequestEventResult` | Yes (replace payload) |
| `after_provider_response` | After HTTP response received, before stream consumed | — | No |
| `agent_start` | When an agent loop starts | — | No |
| `agent_end` | When an agent loop ends (with final messages) | — | No |
| `turn_start` | At the start of each turn | — | No |
| `turn_end` | At the end of each turn (with turn messages + tool results) | — | No |

`before_agent_start` is the event to intercept if you want to prepend to the system prompt or inject a custom message before the agent begins. Its result type:

```ts
interface BeforeAgentStartEventResult {
  message?: Pick<CustomMessage, "customType" | "content" | "display" | "details">;
  systemPrompt?: string;  // Replace the system prompt for this turn (chained across extensions)
}
```

`context` is fired before every LLM call and gives the handler mutable access to the message list via `ContextEventResult`:

```ts
interface ContextEventResult {
  messages?: AgentMessage[];  // Return to replace the message list
}
```

`turn_start` carries `turnIndex` (the zero-based turn counter) and `timestamp`. `turn_end` carries `turnIndex`, the final `message`, and `toolResults`.

### Message events

These fire as each individual message streams in and out.

| Event type | When it fires | Result type |
|---|---|---|
| `message_start` | When a message begins (user, assistant, or tool result) | — |
| `message_update` | Token-by-token during assistant streaming | — |
| `message_end` | When a message finishes | `MessageEndEventResult` |

`message_update` carries `assistantMessageEvent` — the raw streaming delta from the provider. `message_end` lets a handler replace the finalized message (the replacement must keep the original role).

### Tool execution events

These fire around the actual execution of a tool call — distinct from the LLM's decision to make the call.

| Event type | When it fires | What it carries |
|---|---|---|
| `tool_execution_start` | Tool begins executing | `toolCallId`, `toolName`, `args` |
| `tool_execution_update` | Partial/streaming output available | `toolCallId`, `toolName`, `args`, `partialResult` |
| `tool_execution_end` | Tool finishes | `toolCallId`, `toolName`, `result`, `isError` |

These are observation-only — they do not have result types that can intercept or modify execution. Use `tool_call` (below) if you need to block or modify before execution begins.

### Tool call and result events

These are the two broadest interception points in the extension API: they fire on every tool call and let an extension inspect, modify, or block it.

| Event type | When it fires | Result type | Can block? |
|---|---|---|---|
| `tool_call` | Before a tool executes — `event.input` is mutable | `ToolCallEventResult` | Yes |
| `tool_result` | After a tool executes | `ToolResultEventResult` | No (but can modify) |

`tool_call` is dispatched for every tool call — built-in and extension-registered. The `event.input` object is mutable: **mutate it in place** to patch arguments before execution. Later `tool_call` handlers (other extensions) will see the mutated values. To block execution entirely, return `{ block: true, reason: "..." }`.

```ts
// Block dangerous bash commands (from S77 example, genericised)
ext.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
    const ok = await ctx.ui.confirm("Dangerous command", "Allow rm -rf?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

`tool_call` events are narrowed by `toolName`. The source provides type guard helpers — use them instead of direct equality checks on `toolName`, because `CustomToolCallEvent.toolName` is typed as `string` (which overlaps all literals):

```ts
import { isToolCallEventType } from "coding-agent";

ext.on("tool_call", async (event, ctx) => {
  if (isToolCallEventType("bash", event)) {
    // event is now BashToolCallEvent — event.input.command is typed
    console.log(event.input.command);
  }
});
```

The same pattern applies to `tool_result` events with corresponding `isBashToolResult`, `isReadToolResult`, `isEditToolResult`, `isWriteToolResult`, `isGrepToolResult`, `isFindToolResult`, and `isLsToolResult` helpers.

`tool_result` lets a handler replace the result content, details, or error state:

```ts
interface ToolResultEventResult {
  content?: (TextContent | ImageContent)[];
  details?: unknown;
  isError?: boolean;
}
```

### Model, input, and user bash events

The remaining events cover model selection, user input, and direct bash commands.

| Event type | When it fires | Result type |
|---|---|---|
| `model_select` | When a new model is selected | — |
| `thinking_level_select` | When a new thinking level is selected | — |
| `input` | When user input is received, before agent processing | `InputEventResult` |
| `user_bash` | When the user runs a `!` or `!!` prefixed bash command | `UserBashEventResult` |

`model_select` carries `model`, `previousModel`, and `source` — the source is `"set" | "cycle" | "restore"` (how the model was changed).

`input` fires before the agent processes user input. Its result controls what happens next:

```ts
type InputEventResult =
  | { action: "continue" }                                   // Process normally
  | { action: "transform"; text: string; images?: ImageContent[] } // Replace input
  | { action: "handled" };                                   // Extension handled it — skip agent
```

The event carries a `streamingBehavior` field (`"steer" | "followUp" | undefined`) that tells you how the input will be delivered if the agent is currently streaming — useful for skipping expensive preprocessing for mid-stream steers.

`user_bash` fires when the user types `!command` (execute and include in context) or `!!command` (execute and exclude from context). The `excludeFromContext` boolean distinguishes the two. The result can supply custom `BashOperations` (to redirect execution) or a full `BashResult` replacement.

## Putting it together: a guided quick reference

Here is the complete extension API surface in one view:

**Registration (call during the factory function):**

```
registerTool(tool)             → LLM-callable tool
registerCommand(name, opts)    → /command
registerShortcut(key, opts)    → keyboard shortcut
registerFlag(name, opts)       → CLI flag (read back with getFlag)
registerMessageRenderer(type, renderer) → custom message renderer
registerProvider(name, config) → model provider
unregisterProvider(name)       → remove provider
```

**Event subscription (`on(eventName, handler)`) — all events:**

```
Session lifecycle:  resources_discover, session_start, session_before_switch,
                    session_before_fork, session_before_compact, session_compact,
                    session_shutdown, session_before_tree, session_tree

Agent/turn:         before_agent_start, context, before_provider_request,
                    after_provider_response, agent_start, agent_end,
                    turn_start, turn_end

Messages:           message_start, message_update, message_end

Tool execution:     tool_execution_start, tool_execution_update, tool_execution_end

Interception:       tool_call (blockable), tool_result (modifiable)

Model/input:        model_select, thinking_level_select, input, user_bash
```

**Actions (call any time after factory returns):**

```
sendMessage, sendUserMessage, appendEntry
setSessionName, getSessionName, setLabel
exec, getActiveTools, getAllTools, setActiveTools, getCommands
setModel, getThinkingLevel, setThinkingLevel
events (EventBus)
```

## What comes next

We've covered the full `ExtensionAPI` surface — the shape of an extension, handler registration, the `ExtensionContext`, `ToolDefinition`, and every lifecycle event. The next chapter goes deeper into how the extension loader discovers extension files, isolates them, manages the load lifecycle, and surfaces errors when a handler throws.

---

← Previous: [Package Management and Config Migrations](../coding-agent/package-manager-and-migrations.md) · Next: [Loading and Running Extensions: Discovery, Isolation, and Lifecycle](./extension-loader-and-runner.md) →
