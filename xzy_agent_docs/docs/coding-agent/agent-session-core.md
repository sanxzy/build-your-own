---
title: "AgentSession: The Core of the Coding Agent"
description: "How AgentSession wraps the Agent class and owns model selection, compaction, bash execution, session branching, and extension dispatch for all run modes."
category: coding-agent
type: tutorial
tags:
  [
    AgentSession,
    Agent class,
    session lifecycle,
    model selection,
    compaction,
    bash,
    branching,
    persistence,
    coding-agent,
    composition,
    shared core,
    modes,
    interactive mode,
    print mode,
    rpc mode,
    sdk mode,
    ExtensionRunner,
    AgentSessionConfig,
    AgentSessionEvent,
    compaction threshold,
    overflow recovery,
    cycleModel,
    setModel,
    setThinkingLevel,
    executeBash,
    navigateTree,
    subscribe,
    dispose,
    tool registry,
    system prompt,
  ]
keywords:
  [
    coding agent architecture,
    session hub,
    agent lifecycle,
    model cycling,
    thinking level,
    auto-compaction,
    session branching,
    tree navigation,
    extension events,
    bash execution,
    session persistence,
    PromptOptions,
    ModelCycleResult,
    SessionStats,
    ContextUsage,
    steer,
    followUp,
    sendUserMessage,
    abort,
    reload,
  ]
sources: [S51, S47]
---

**TL;DR** — `AgentSession` is the single shared core that every run mode of the coding agent (interactive, print, RPC, SDK) is built on top of. It wraps the `Agent` class directly — not through the higher-level `AgentHarness` — and takes ownership of model selection, compaction, bash execution, session branching, extension event dispatch, and persistence. By the end of this chapter you will understand what `AgentSession` holds, why it exists at this layer of the stack, and how its lifecycle flows from construction through each prompt turn to disposal.

# AgentSession: The Core of the Coding Agent

## The problem this layer solves

At this point in the library we have built three separate layers:

- **`llm-toolkit`** — streaming, providers, and the `Model` type (see [the LLM toolkit chapters](../llm-toolkit/)).
- **`agent-core`** — the stateful `Agent` class that runs the tool-call loop (see [the Agent class](../agent-loop/the-agent-class.md)).
- **`tui`** — the terminal render engine the interactive mode will use (see [the TUI class and render engine](../terminal-ui/the-tui-class-and-render-engine.md)).

Each of those is a standalone library. The `coding-agent` package depends on all three simultaneously, as its `package.json` makes clear:

```json
// coding-agent/package.json (illustrative)
{
  "name": "coding-agent",
  "version": "0.1.0",
  "description": "Coding agent CLI with read, bash, edit, write tools and session management",
  "dependencies": {
    "agent-core": "^0.1.0",
    "llm-toolkit": "^0.1.0",
    "tui": "^0.1.0"
  }
}
```

(Brand-neutral names and minimal starting versions shown — these three siblings are local monorepo packages, so the versions can be whatever your project uses.)

Now we have a real design problem: the agent will run in at least four distinct *modes* — an interactive TUI for humans, a print mode for piped output, an RPC mode for editor integrations, and an SDK mode for embedding — but those modes share most of the same concerns: which model to use, whether the context is getting too large, how to persist the session to disk, how to hand off events to extensions. We do not want to duplicate that logic four times.

That is exactly what `AgentSession` solves. It is a **shared core** that all four run modes compose. Each mode then adds only the I/O layer specific to that mode on top. The class lives in `src/core/agent-session.ts` (S51) and is the architectural hub of the `coding-agent` layer.

### One important distinction from the harness

You may recall from [harness, sessions, and compaction](../agent-loop/harness-session-and-compaction.md) that `agent-core` ships its own `AgentHarness` — a higher-level wrapper that manages multiple sessions. `AgentSession` is **not** a thin wrapper around `AgentHarness`. Instead, it wraps the lower-level `Agent` class directly:

```ts
// Simplified view of AgentSession's class fields (S51)
export class AgentSession {
  readonly agent: Agent;          // ← the raw Agent from agent-core
  readonly sessionManager: SessionManager;
  readonly settingsManager: SettingsManager;
  // ...
}
```

`AgentHarness` was designed for the harness's own multi-session coordination scheme. `AgentSession` builds its own composition on top of the primitive `Agent`, giving the coding agent full control over session trees, compaction policy, extension dispatch, and persistence without inheriting harness assumptions.

---

## What AgentSession holds

Let's walk through what the class owns, one concern at a time, building up to a picture of the full lifecycle.

### The configuration it needs at construction

`AgentSession` takes a single config object. We'll introduce the required pieces as we go, but the overall shape is:

```ts
// AgentSessionConfig — from S51 (simplified view)
export interface AgentSessionConfig {
  agent: Agent;                        // the pre-constructed Agent instance
  sessionManager: SessionManager;      // owns the on-disk session tree
  settingsManager: SettingsManager;    // reads/writes persistent user settings
  cwd: string;                         // working directory for the session
  resourceLoader: ResourceLoader;      // loads skills, prompts, themes, context files
  modelRegistry: ModelRegistry;        // resolves API keys; discovers available models
  scopedModels?: Array<{ model: Model<any>; thinkingLevel?: ThinkingLevel }>;
  customTools?: ToolDefinition[];      // extra tools registered outside extensions
  initialActiveToolNames?: string[];   // which tools start enabled (default: read, bash, edit, write)
  allowedToolNames?: string[];         // optional allowlist — only these are exposed
  excludedToolNames?: string[];        // optional denylist — these are never exposed
  baseToolsOverride?: Record<string, AgentTool>; // swap built-in tools (for custom runtimes)
  extensionRunnerRef?: { current?: ExtensionRunner };
  sessionStartEvent?: SessionStartEvent;
}
```

The constructor wires everything together and does three key things: it subscribes its internal handler to the `Agent`'s event stream, installs tool hooks (for before/after tool call interception), and calls `_buildRuntime()` to populate the tool registry and the `ExtensionRunner`.

```ts
// Constructor summary (S51)
constructor(config: AgentSessionConfig) {
  this.agent = config.agent;
  this.sessionManager = config.sessionManager;
  this.settingsManager = config.settingsManager;
  // ... copy all config fields ...

  // Always subscribe to agent events for internal handling
  // (session persistence, extensions, auto-compaction, retry logic)
  this._unsubscribeAgent = this.agent.subscribe(this._handleAgentEvent);
  this._installAgentToolHooks();

  this._buildRuntime({
    activeToolNames: this._initialActiveToolNames,
    includeAllExtensionTools: true,
  });
}
```

Notice that `AgentSession` subscribes to the `Agent` on construction and never needs external code to wire that up. The subscription is private; modes and callers get a *separate* subscription surface via `AgentSession.subscribe()`.

---

## Model selection

One of the first things a run mode needs to do is choose which model to use. `AgentSession` owns that logic completely.

### Setting a model directly

When the user selects a model (or a mode sets one programmatically), `setModel()` validates that authentication is configured, updates the agent's state, persists the change to the session file and to user settings, and then re-clamps the thinking level to what the new model supports:

```ts
// setModel — from S51
async setModel(model: Model<any>): Promise<void> {
  if (!this._modelRegistry.hasConfiguredAuth(model)) {
    throw new Error(`No API key for ${model.provider}/${model.id}`);
  }

  const previousModel = this.model;
  const thinkingLevel = this._getThinkingLevelForModelSwitch();
  this.agent.state.model = model;
  this.sessionManager.appendModelChange(model.provider, model.id);
  this.settingsManager.setDefaultModelAndProvider(model.provider, model.id);

  // Re-clamp thinking level for new model's capabilities
  this.setThinkingLevel(thinkingLevel);

  await this._emitModelSelect(model, previousModel, "set");
}
```

`_modelRegistry.hasConfiguredAuth()` checks that an API key (or OAuth credential) is available, preventing a runtime failure mid-session.

### Cycling through models

The interactive mode exposes a keyboard shortcut to cycle through available models. `cycleModel()` delegates to either `_cycleScopedModel()` (when the user passed `--models` on the command line) or `_cycleAvailableModel()` (all models the registry knows about):

```ts
// cycleModel signature — from S51
async cycleModel(
  direction: "forward" | "backward" = "forward"
): Promise<ModelCycleResult | undefined>

// ModelCycleResult — from S51
export interface ModelCycleResult {
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  isScoped: boolean; // true when cycling --models list, false for all available
}
```

If only one model is available, `cycleModel()` returns `undefined`.

### Thinking levels

Models have different capabilities around extended thinking (step-by-step reasoning before answering). `AgentSession` manages the *thinking level* alongside the model:

```ts
// Thinking level management — from S51
const THINKING_LEVELS: ThinkingLevel[] = ["off", "minimal", "low", "medium", "high"];

setThinkingLevel(level: ThinkingLevel): void
cycleThinkingLevel(): ThinkingLevel | undefined  // undefined if model has no thinking support
getAvailableThinkingLevels(): ThinkingLevel[]
supportsThinking(): boolean
```

`setThinkingLevel()` clamps the requested level to what the current model actually supports (via `clampThinkingLevel()` from `llm-toolkit`), persists the change to the session and settings, and emits a `thinking_level_changed` event to listeners.

---

## Sending a prompt

Now that we have a model, we need to actually send prompts. The main entry point is `prompt()`:

```ts
// prompt signature — from S51
async prompt(text: string, options?: PromptOptions): Promise<void>

export interface PromptOptions {
  expandPromptTemplates?: boolean;   // default: true — expands /skill:name and /template
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp"; // required if already streaming
  source?: InputSource;              // defaults to "interactive"
  preflightResult?: (success: boolean) => void;
}
```

When the agent is not currently streaming, `prompt()` runs through several steps in order:

1. **Extension command check.** If the text starts with `/` and matches a registered extension command, that command executes immediately and `prompt()` returns without touching the LLM.
2. **Extension input interception.** The `ExtensionRunner` fires an `input` event; extensions may transform or fully handle the text.
3. **Skill and template expansion.** `/skill:name args` is expanded to the full skill block content; `/template args` expands file-based prompt templates.
4. **Model and auth validation.** If no model is selected or no auth is configured, an error is thrown.
5. **Pre-prompt compaction check.** If the last assistant message was aborted and the context is over threshold, compaction runs before the new prompt goes out.
6. **Message construction.** The user text (and any pending "next turn" custom messages from extensions) is assembled into the message array.
7. **Extension `before_agent_start` hook.** Extensions can inject extra custom messages or modify the system prompt for this turn only.
8. **Agent run.** The assembled messages go to `agent.prompt()`, and `AgentSession` loops over `_handlePostAgentRun()` to handle retries and compaction.

When the agent *is* streaming, `prompt()` instead queues the message via `steer()` (interrupt after current tool calls) or `followUp()` (wait until the agent is fully done), as specified by `streamingBehavior`.

### The event flow on each turn

When a prompt reaches the agent, `AgentSession`'s internal `_handleAgentEvent` listener fires for every event the agent emits. It does four things in order for each event:

1. **Queue state management.** If a `message_start` user event arrives and the message text matches something in the steering or follow-up queues, that entry is removed and a `queue_update` event is emitted to surface the updated queue to the UI.
2. **Extension dispatch.** `_emitExtensionEvent()` translates core `AgentEvent` types into the richer extension event types (`TurnStartEvent`, `MessageEndEvent`, `ToolExecutionStartEvent`, etc.) and fires them through the `ExtensionRunner`. On `message_end`, extensions can return a replacement message object.
3. **Listener notification.** All callers that called `AgentSession.subscribe()` receive the event.
4. **Session persistence.** On `message_end`, regular LLM messages (user, assistant, toolResult) are appended to the `SessionManager`. Custom messages from extensions are appended as `CustomMessageEntry` records.

This single internal handler is the reason all run modes can share the same persistence and extension behaviour without duplicating it.

---

## Compaction

The session context grows as conversation history accumulates. Beyond a certain point the context window fills up and LLM calls start failing. `AgentSession` monitors this and compacts automatically.

### What compaction does

Compaction summarises older conversation history into a compact text block, then replaces the old messages in the agent's in-memory state with that summary plus any messages that were sent after the compaction point. The on-disk session file keeps the full history; only the live context is trimmed.

### The two triggers

```
_checkCompaction() is called after every agent_end and before each new prompt.
```

Inside it, two cases are handled:

| Trigger | Condition | Behaviour |
|---|---|---|
| **Overflow** | LLM returned a context-overflow error (`isContextOverflow()` true) | Remove the error message from agent state; compact; auto-retry the turn. Attempted only once per agent run — if it fails again, an error is emitted. |
| **Threshold** | Context tokens from `calculateContextTokens()` cross the configured threshold | Compact; do NOT auto-retry (the user continues manually). |

### Manual compaction

Callers (and the interactive mode's `/compact` command) invoke `compact()` directly:

```ts
// compact — from S51
async compact(customInstructions?: string): Promise<CompactionResult>
```

`compact()` first aborts any running agent operation, then:

1. Emits `compaction_start` with `reason: "manual"`.
2. Fires a `session_before_compact` extension event — extensions may cancel or supply their own compaction result.
3. If no extension supplied a result, calls the `compact()` function from the compaction module.
4. Appends the compaction entry to `SessionManager` and rebuilds `agent.state.messages` from the new session context.
5. Emits `session_compact` to extensions, then `compaction_end` to listeners.

Both manual and auto-compaction emit `compaction_start` / `compaction_end` events so the UI can show progress and results consistently.

```ts
// AgentSessionEvent types for compaction — from S51
| { type: "compaction_start"; reason: "manual" | "threshold" | "overflow" }
| {
    type: "compaction_end";
    reason: "manual" | "threshold" | "overflow";
    result: CompactionResult | undefined;
    aborted: boolean;
    willRetry: boolean;
    errorMessage?: string;
  }
```

---

## Bash execution

The coding agent is not just a chat interface — users and extensions can run shell commands that inject their output into the agent's context. `AgentSession` owns this too:

```ts
// executeBash — from S51
async executeBash(
  command: string,
  onChunk?: (chunk: string) => void,
  options?: {
    excludeFromContext?: boolean;  // "!!" prefix: don't send output to LLM
    operations?: BashOperations;   // override for remote execution
  }
): Promise<BashResult>
```

Internally, `executeBash()` applies any configured shell command prefix (e.g. `shopt -s expand_aliases`) from settings, then delegates to `executeBashWithOperations()`. The result is recorded into the session as a `BashExecutionMessage`:

```ts
// BashExecutionMessage shape stored in session (S51)
{
  role: "bashExecution",
  command: string,
  output: string,
  exitCode: number,
  cancelled: boolean,
  truncated: boolean,
  fullOutputPath: string | undefined,
  timestamp: number,
  excludeFromContext: boolean | undefined,
}
```

One subtlety: if the agent is currently streaming when a bash command finishes, adding the message immediately would break the `tool_use` / `tool_result` ordering that the LLM expects. So `AgentSession` queues the message in `_pendingBashMessages` and flushes it after the current agent turn ends.

The running command can be cancelled via `abortBash()`.

---

## Session branching and tree navigation

A session in the coding agent is not a flat list of messages — it is a **tree**. Each node in the tree has a parent, and the current "branch" is the path from the root to the current leaf. This lets users navigate back to any earlier state and continue from there, creating alternative conversation branches without losing history.

`AgentSession.navigateTree()` handles switching branches:

```ts
// navigateTree — from S51
async navigateTree(
  targetId: string,
  options: {
    summarize?: boolean;           // generate an LLM summary of the abandoned branch
    customInstructions?: string;   // instructions for the summarizer
    replaceInstructions?: boolean; // if true, customInstructions replaces the default prompt
    label?: string;                // label to attach to the summary entry
  } = {}
): Promise<{
  editorText?: string;   // restored editor text if target was a user message
  cancelled: boolean;
  aborted?: boolean;
  summaryEntry?: BranchSummaryEntry;
}>
```

When navigating, `AgentSession`:

1. Collects the entries that exist between the current leaf and the target (the "abandoned" sub-branch).
2. Optionally fires a `session_before_tree` extension event, which can cancel navigation or supply its own branch summary.
3. If summarization was requested, calls `generateBranchSummary()` to produce a condensed LLM summary of the abandoned branch.
4. Calls `SessionManager.branch()` (or `branchWithSummary()` if there is a summary) to move the leaf pointer.
5. Rebuilds `agent.state.messages` from the new branch context.
6. Emits a `session_tree` event to extensions.

If the navigation target was a user message, the user's original text is returned as `editorText` so the interactive mode can restore it to the input field — letting the user revise and re-send without re-typing.

---

## Extension event dispatch

The `ExtensionRunner` is the bridge between `AgentSession` and any loaded extension plugins. `AgentSession` builds and owns the runner, wiring it up with the session's capabilities:

```ts
// _buildRuntime creates the ExtensionRunner (S51)
this._extensionRunner = new ExtensionRunner(
  extensionsResult.extensions,
  extensionsResult.runtime,
  this._cwd,
  this.sessionManager,
  this._modelRegistry,
);
```

Extensions receive a rich set of events at every stage of the agent turn. The internal `_emitExtensionEvent()` method translates core `AgentEvent` values into the typed extension events:

| Core event | Extension event emitted |
|---|---|
| `agent_start` | `agent_start` |
| `agent_end` | `agent_end` |
| `turn_start` | `TurnStartEvent` (with `turnIndex`, `timestamp`) |
| `turn_end` | `TurnEndEvent` (with `turnIndex`, `message`, `toolResults`) |
| `message_start` | `MessageStartEvent` |
| `message_update` | `MessageUpdateEvent` |
| `message_end` | `MessageEndEvent` — extensions may return a replacement message |
| `tool_execution_start` | `ToolExecutionStartEvent` |
| `tool_execution_update` | `ToolExecutionUpdateEvent` |
| `tool_execution_end` | `ToolExecutionEndEvent` |

Modes bind extension UI context and mode information via `bindExtensions()`:

```ts
// bindExtensions — from S51
async bindExtensions(bindings: ExtensionBindings): Promise<void>

export interface ExtensionBindings {
  uiContext?: ExtensionUIContext;
  mode?: ExtensionMode;
  commandContextActions?: ExtensionCommandContextActions;
  abortHandler?: () => void;
  shutdownHandler?: ShutdownHandler;
  onError?: ExtensionErrorListener;
}
```

After binding, `AgentSession` emits the `session_start` event so extensions can initialise themselves.

---

## Persistence

`AgentSession` does not call a "save" method itself — persistence is woven into the event handler. Every time `_handleAgentEvent` fires a `message_end`, it calls `sessionManager.appendMessage(event.message)` for LLM messages or `sessionManager.appendCustomMessageEntry(...)` for extension custom messages. Model changes, thinking-level changes, compaction results, and branch summaries are all appended to the `SessionManager` at the moment they occur. The session file on disk is therefore always up to date after each event, not just at the end of a turn.

---

## The full lifecycle

Putting it all together, here is the lifecycle of an `AgentSession` from construction to disposal:

```
1. Construct AgentSession(config)
   └─ subscribes internal handler to agent events
   └─ installs tool hooks (beforeToolCall / afterToolCall)
   └─ builds ExtensionRunner, populates tool registry, sets active tools
   └─ builds initial system prompt

2. Mode calls bindExtensions(bindings)
   └─ wires UI context, mode, command context, error handler
   └─ emits session_start to extensions
   └─ extensions may register additional resources (skills, prompts)
   └─ system prompt is rebuilt with extension resources

3. For each user turn:
   a. session.prompt(text, options)
      └─ extension command / input / before_agent_start hooks
      └─ builds message array (user + any pending custom messages)
      └─ agent.prompt(messages)  ← the Agent class runs the tool-call loop

   b. On every agent event → _handleAgentEvent fires:
      └─ queue state updates
      └─ extension event dispatch
      └─ listener notification (subscribe() callers)
      └─ session persistence (appendMessage on message_end)

   c. After agent_end:
      └─ retry logic (if overloaded/rate-limited, exponential backoff)
      └─ compaction check (threshold → compact; overflow → compact + retry)

4. Between turns (anytime):
   └─ executeBash(command)      — run a shell command, inject result
   └─ setModel(model)           — switch model, re-clamp thinking level
   └─ cycleModel(direction)     — rotate through available models
   └─ compact(instructions)     — manual compaction
   └─ navigateTree(targetId)    — branch to a different history node

5. session.dispose()
   └─ aborts retry, compaction, branch summary, bash
   └─ aborts agent
   └─ invalidates ExtensionRunner (stale ctx)
   └─ disconnects internal agent subscription
   └─ clears all listeners
   └─ cleans up session resources
```

---

## Reading session state

At any point, callers can inspect the current state through read-only properties and methods:

```ts
// Read-only surface of AgentSession (S51)
session.state            // full AgentState from agent-core
session.model            // current Model<any> | undefined
session.thinkingLevel    // current ThinkingLevel
session.isStreaming       // boolean — is a response in flight?
session.systemPrompt     // effective system prompt (may include per-turn extension edits)
session.messages         // all messages including BashExecutionMessage
session.sessionId        // unique session ID (stable across restarts)
session.sessionFile      // path to on-disk .jsonl file (or undefined if disabled)
session.sessionName      // display name, if set
session.isCompacting     // any compaction in progress?
session.isBashRunning    // bash command in flight?
session.retryAttempt     // current retry counter (0 if not retrying)
```

For context-window pressure reporting, `getContextUsage()` returns a `ContextUsage`:

```ts
// ContextUsage — from S51
interface ContextUsage {
  tokens: number | null;    // null immediately after compaction (no post-compaction usage yet)
  contextWindow: number;
  percent: number | null;
}
```

`getSessionStats()` returns a detailed breakdown of message counts, token usage (input, output, cache reads/writes), and cost.

---

## How modes plug into AgentSession

`AgentSession` deliberately does not own any I/O. It emits events; modes react to them. The pattern each mode follows is the same:

1. Construct an `Agent` instance (from `agent-core`).
2. Construct a `SessionManager`, `SettingsManager`, `ResourceLoader`, and `ModelRegistry`.
3. Construct `AgentSession(config)` — this is the hub.
4. Subscribe to `AgentSession` events for mode-specific rendering (`session.subscribe(listener)`).
5. Call `session.bindExtensions(bindings)` to connect UI context.
6. Start accepting user input, forwarding it to `session.prompt(text, options)`.

The interactive mode adds the TUI render layer on top. Print mode sends output to stdout. RPC mode serialises events as JSON. The SDK mode exposes a programmatic API. None of them re-implement model selection, compaction, persistence, or extension dispatch — all of that lives in `AgentSession`.

Forward references for the chapters that build on this hub:

- **Built-in tools** (read, write, edit, bash, and others) — [Built-In Coding Tools](./built-in-tools.md)
- **Session management and the tree** — covered in the session management chapter
- **Run modes** (interactive, print, RPC, SDK) — covered in the modes chapters

---

← Previous: [Autocomplete and Building a Complete Chat Interface](../terminal-ui/autocomplete-and-a-complete-chat-interface.md) · Next: [Built-In Coding Tools: Bash, Read, Write, Edit, and More](./built-in-tools.md) →
