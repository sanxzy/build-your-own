---
title: "Agent Context, Events, and Types"
description: "A guided tour of the agent-core type vocabulary: AgentContext, AgentLoopConfig, AgentEvent, AgentTool, AgentMessage, QueueMode, and StreamFn."
category: agent-loop
type: tutorial
tags: [AgentContext, AgentLoopConfig, AgentEvent, AgentTool, AgentMessage, QueueMode, StreamFn, agent-core, types, state, tool definition, agent framework, ToolExecutionMode, ThinkingLevel, AgentToolResult, AgentState, BeforeToolCallContext, AfterToolCallContext, ShouldStopAfterTurnContext, CustomAgentMessages, AgentToolCall, AgentToolUpdateCallback]
keywords: [agent type system, tool schema TypeBox, agent loop data model, event streaming agent, parallel sequential tool execution, agent state management, low-level agent API, convertToLlm, transformContext, declaration merging custom messages]
sources: [S24, S26]
---

**TL;DR** — Before we can write the agent loop itself, we need to understand the data it operates on. This chapter introduces every core type in `agent-core`: the conversation snapshot (`AgentContext`), the loop's configuration (`AgentLoopConfig`), the events the loop emits (`AgentEvent`), how tools are defined (`AgentTool`), and the supporting types that hold it all together. By the end you will be able to read any agent-core type signature and know exactly what each field does.

# Agent Context, Events, and Types

## Why we need a type vocabulary first

In the previous chapters we built the LLM toolkit — a way to send a message to a model and stream back a structured response. That is the "one-shot call" layer. An *agent* is what runs that call inside a loop, inspects the result, executes any tool calls the model requested, and then calls the model again with the results — repeating until there is nothing left to do.

Before we write that loop, we need its data model. Just as you would define your database schema before writing queries, we define the types before we write the logic that manipulates them. This chapter is that foundation. The next chapter, [The Agent Loop: Turn-by-Turn LLM and Tool Execution](./the-agent-loop.md), will use every type defined here.

There are a few prerequisite concepts worth recapping quickly:

- **Message and content-block types** — the `Message`, `UserMessage`, `AssistantMessage`, and `ToolResultMessage` types that the LLM toolkit defines. These are what a model actually understands. If you need a refresher, see [Message Types and the Streaming API](../llm-toolkit/message-types-and-streaming-api.md).
- **EventStream observable** — an async iterable that emits typed events as a model streams its response. The agent loop extends this idea upward. See [EventStream and the Observable Backbone](../llm-toolkit/event-stream-and-observable-backbone.md).
- **Model and streaming JSON** — the `Model<TCaps>` metadata type and how tool-call arguments arrive as streaming JSON fragments. See [Models, Cost, and Streaming JSON](../llm-toolkit/models-cost-and-streaming-json.md).

With those in mind, let us work through each type in dependency order: the smallest building blocks first, then the composites.

---

## The message layer: AgentMessage and CustomAgentMessages

The LLM toolkit's `Message` type is a union of `UserMessage`, `AssistantMessage`, and `ToolResultMessage` — the three roles a real language model understands. The agent layer adds one more concept on top: **custom app-specific messages**.

An application built on `agent-core` might want to store UI notifications, status banners, or other data in the conversation transcript without ever sending them to the model. To support this without losing type safety, the library uses a TypeScript pattern called *declaration merging*:

```ts
// agent-core/src/types.ts (simplified view)

/**
 * Extensible interface for custom app messages.
 * Apps extend this via declaration merging.
 */
export interface CustomAgentMessages {
  // Empty by default — apps extend this interface
}

/**
 * AgentMessage: union of standard LLM messages + any custom messages
 * the app has declared via CustomAgentMessages.
 */
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

`Message` here is imported directly from `llm-toolkit`. By default, `AgentMessage` is identical to `Message`. An app adds custom message types by extending `CustomAgentMessages` in its own module:

```ts
// In your app code — extends the empty interface
declare module "agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}

// Now this is valid everywhere AgentMessage is accepted:
const msg: AgentMessage = { role: "notification", text: "Ready", timestamp: Date.now() };
```

This is what "declaration merging" means: TypeScript merges your `interface` declaration with the one in `agent-core`, so the union type grows automatically.

The key rule is that **only `user`, `assistant`, and `toolResult` messages ever reach the model**. Everything else is filtered in `convertToLlm` — a function we will meet in `AgentLoopConfig` below.

---

## AgentContext — the conversation snapshot

Now we can define the simplest container: `AgentContext`. This is the snapshot of conversation state that the agent loop reads at the start of each turn.

```ts
// agent-core/src/types.ts

/** Context snapshot passed into the low-level agent loop. */
export interface AgentContext {
  /** System prompt included with the request. */
  systemPrompt: string;
  /** Transcript visible to the model. */
  messages: AgentMessage[];
  /** Tools available for this run. */
  tools?: AgentTool<any>[];
}
```

Three fields — nothing more. Notice that `tools` is optional: a context without tools is a pure conversational loop that never calls any external function. The loop appends new messages to `messages` as the conversation progresses, so this object is effectively the ever-growing transcript.

You will use `AgentContext` directly when driving the low-level `agentLoop()` function. When you use the higher-level `Agent` class instead, the class manages this object internally and exposes it through `agent.state`.

---

## AgentTool — how a tool is defined

An agent without tools is just a stateful chat wrapper. Tools are what make it an agent. Let us look at how a tool is expressed in the type system, because the shape has some important subtleties.

### What the LLM toolkit already gives us

`AgentTool` extends `Tool`, which comes from `llm-toolkit`. `Tool` carries the fields the model needs to know about: the tool's `name`, `description`, and `parameters` (a TypeBox JSON Schema object). `AgentTool` adds the runtime fields the agent needs:

```ts
// agent-core/src/types.ts

/** Tool definition used by the agent runtime. */
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any>
  extends Tool<TParameters> {

  /** Human-readable label for UI display. */
  label: string;

  /**
   * Optional compatibility shim for raw tool-call arguments before schema validation.
   * Must return an object that matches TParameters.
   */
  prepareArguments?: (args: unknown) => Static<TParameters>;

  /** Execute the tool call. Throw on failure instead of encoding errors in content. */
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;

  /**
   * Per-tool execution mode override.
   * "sequential": this tool must run one at a time with others.
   * "parallel": this tool can run concurrently with others.
   * If omitted, the global toolExecution mode applies.
   */
  executionMode?: ToolExecutionMode;
}
```

The two generic type parameters are worth noting:

- `TParameters extends TSchema` — the TypeBox schema for this tool's arguments. TypeBox lets you write a JSON Schema as a TypeScript value, and the type `Static<TParameters>` gives you the corresponding TypeScript type. That means `params` in `execute` is fully typed — no casting needed.
- `TDetails` — the type of the `details` field in the result. This is a structured payload for logs and UI rendering, separate from the `content` the model sees.

### The AgentToolResult shape

Every tool returns an `AgentToolResult<T>`:

```ts
/** Final or partial result produced by a tool. */
export interface AgentToolResult<T> {
  /** Text or image content returned to the model. */
  content: (TextContent | ImageContent)[];
  /** Arbitrary structured details for logs or UI rendering. */
  details: T;
  /**
   * Hint that the agent should stop after the current tool batch.
   * Early termination only happens when every finalized tool result
   * in the batch sets this to true.
   */
  terminate?: boolean;
}
```

`content` is what the model reads — a list of text or image blocks. `details` is what your UI or logging system reads — whatever structured data makes sense for your tool. They travel together so you can render rich results without teaching the model about your internal data shapes.

### A complete tool definition

Here is a concrete example that puts the types together. We define a `read_file` tool that uses TypeBox to declare its parameter schema:

```ts
import { Type } from "typebox";
import type { AgentTool } from "agent-core";
import * as fs from "fs/promises";

const readFileTool: AgentTool<
  // TParameters: the TypeBox schema
  ReturnType<typeof Type.Object>,
  // TDetails: what details looks like
  { path: string; size: number }
> = {
  name: "read_file",
  label: "Read File",                         // shown in the UI
  description: "Read the contents of a file",
  parameters: Type.Object({
    path: Type.String({ description: "Absolute path to the file" }),
  }),
  // executionMode omitted → global toolExecution setting applies
  execute: async (toolCallId, params, signal, onUpdate) => {
    // params.path is typed as string — no casting
    const content = await fs.readFile(params.path, "utf-8");

    // Optional: stream partial progress to the UI
    onUpdate?.({
      content: [{ type: "text", text: "Reading..." }],
      details: { path: params.path, size: 0 },
    });

    return {
      content: [{ type: "text", text: content }],
      details: { path: params.path, size: content.length },
    };
  },
};
```

Notice the error convention: **throw on failure, do not encode errors in `content`**. The agent catches thrown errors and reports them to the model as tool errors with `isError: true`. Returning a string like `"Error: file not found"` in `content` would look like a successful result to the model.

### AgentTool fields at a glance

| Field | Type | Required | Purpose |
|---|---|---|---|
| `name` | `string` | yes | Identifier the model uses when calling the tool |
| `description` | `string` | yes | Natural-language description sent to the model |
| `parameters` | `TSchema` (TypeBox) | yes | JSON Schema for the tool's arguments |
| `label` | `string` | yes | Human-readable name for UI display |
| `execute` | function | yes | The implementation; throws on failure |
| `prepareArguments` | function | no | Pre-validation shim for raw args |
| `executionMode` | `"sequential" \| "parallel"` | no | Per-tool override; omit to use global setting |

---

## ToolExecutionMode and QueueMode

Two small union types control how the loop sequences work. Let us meet them here, before they appear in `AgentLoopConfig`.

```ts
// agent-core/src/types.ts

/**
 * Configuration for how tool calls from a single assistant message are executed.
 *
 * "sequential": each tool call is prepared, executed, and finalized before the next starts.
 * "parallel":   tool calls are preflighted sequentially, then allowed tools execute concurrently.
 *               tool_execution_end is emitted in tool completion order;
 *               tool-result message artifacts are emitted later in assistant source order.
 */
export type ToolExecutionMode = "sequential" | "parallel";

/**
 * Controls how many queued user messages are injected when the loop
 * reaches a queue drain point.
 *
 * "all":            drain and inject every queued message at that point.
 * "one-at-a-time":  drain and inject only the oldest queued message,
 *                   leaving the rest queued for later drain points.
 */
export type QueueMode = "all" | "one-at-a-time";
```

`ToolExecutionMode` governs whether the assistant's tool calls in one turn run one-at-a-time or concurrently. The default is `"parallel"`. If any single tool in a batch has `executionMode: "sequential"`, the entire batch executes sequentially regardless of the global setting.

`QueueMode` governs the steering and follow-up queues — the mechanism that lets you inject messages into a running agent. We will see these queues in context in the next chapter.

---

## StreamFn — the model-calling function

The agent loop needs to call the model. Rather than hard-coding a specific API client, it accepts a **function** that matches the `StreamFn` type:

```ts
// agent-core/src/types.ts

/**
 * Stream function used by the agent loop.
 *
 * Contract:
 * - Must not throw or return a rejected promise for request/model/runtime failures.
 * - Must return an AssistantMessageEventStream.
 * - Failures must be encoded in the returned stream via protocol events and a
 *   final AssistantMessage with stopReason "error" or "aborted" and errorMessage.
 */
export type StreamFn = (
  ...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple> | Promise<ReturnType<typeof streamSimple>>;
```

`streamSimple` is the function exported by `llm-toolkit` that sends a request and returns a stream of `AssistantMessageEvent` values. `StreamFn` is defined as having the same parameter list and the same return type — which means the toolkit's own `streamSimple` (or `stream`) satisfies `StreamFn` directly. You only need a custom `StreamFn` when you want to proxy requests through a backend, add authentication middleware, or swap in a test double.

The critical contract: `StreamFn` must never reject. Any failure — network error, rate limit, model timeout — must be encoded as events in the returned stream, ending with an `AssistantMessage` whose `stopReason` is `"error"` or `"aborted"`. This keeps the loop's error handling simple: it only needs to inspect the final message, not catch arbitrary promise rejections.

---

## ThinkingLevel

Some models support extended reasoning ("thinking") budgets. The type system captures this as:

```ts
// agent-core/src/types.ts

export type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
```

`"xhigh"` is only supported by selected model families. You can detect whether a specific model supports a given thinking level using the model metadata from `llm-toolkit` (see [Models, Cost, and Streaming JSON](../llm-toolkit/models-cost-and-streaming-json.md)).

---

## AgentLoopConfig — all the knobs for one run

`AgentLoopConfig` is the configuration object you pass to the low-level `agentLoop()` function. It gathers every decision the loop delegates to the caller: which model, how to translate messages, how to handle tool calls, when to stop. Let us walk through each field.

```ts
// agent-core/src/types.ts (key fields shown; some hook signatures simplified)

export interface AgentLoopConfig extends SimpleStreamOptions {
  // --- Required ---
  model: Model<any>;
  convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;

  // --- Optional context transforms ---
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;

  // --- Optional auth ---
  getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;

  // --- Loop control ---
  shouldStopAfterTurn?: (context: ShouldStopAfterTurnContext) => boolean | Promise<boolean>;
  prepareNextTurn?: (context: PrepareNextTurnContext) => AgentLoopTurnUpdate | undefined | Promise<AgentLoopTurnUpdate | undefined>;

  // --- Steering and follow-up queues ---
  getSteeringMessages?: () => Promise<AgentMessage[]>;
  getFollowUpMessages?: () => Promise<AgentMessage[]>;

  // --- Tool execution ---
  toolExecution?: ToolExecutionMode;           // default: "parallel"
  beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>;
  afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>;
}
```

`AgentLoopConfig` extends `SimpleStreamOptions` — the base options type from `llm-toolkit` (things like the system prompt used by the streaming call). The fields above are the agent-specific additions.

### The two required fields

**`model`** is a `Model<any>` from `llm-toolkit` — the model descriptor that carries provider id, model id, and capability metadata.

**`convertToLlm`** is the translation layer between `AgentMessage[]` and `Message[]`. Recall that `AgentMessage` is a superset of `Message`: it may include custom app messages that the model has never heard of. `convertToLlm` is called before each LLM request to filter those out and convert anything that needs converting:

```ts
// Example: keep standard LLM messages, drop custom notification messages
convertToLlm: (messages) => messages.flatMap(m => {
  if (m.role === "notification") return [];   // filter out UI-only messages
  return [m];                                 // pass through user/assistant/toolResult
})
```

The contract is strict: **must not throw or reject**. If it does, it aborts the low-level loop without producing a normal event sequence.

### transformContext — optional pre-filter

`transformContext` runs *before* `convertToLlm`. It receives the full `AgentMessage[]` and returns a (possibly shorter) `AgentMessage[]`. This is where you put context-window management — pruning old messages, summarising history, injecting external context:

```ts
transformContext: async (messages, signal) => {
  if (estimateTokens(messages) > MAX_TOKENS) {
    return pruneOldMessages(messages);
  }
  return messages;
}
```

The message flow through the loop is therefore:

```
AgentMessage[]  →  transformContext()  →  AgentMessage[]  →  convertToLlm()  →  Message[]  →  LLM
                     (optional)                               (required)
```

### beforeToolCall and afterToolCall — tool hooks

These two hooks bracket each tool execution. They receive context objects that tell them which tool is about to run and what the agent state looks like at that moment.

**`beforeToolCall`** runs after argument validation. Return `{ block: true, reason: "..." }` to prevent the tool from running; the loop emits an error tool result in its place. Return `undefined` to allow the tool to proceed:

```ts
beforeToolCall: async ({ toolCall, args, context }) => {
  if (toolCall.name === "bash") {
    return { block: true, reason: "bash is disabled in this session" };
  }
  // return undefined to allow
}
```

**`afterToolCall`** runs after execution and before the final `tool_execution_end` event. Return an `AfterToolCallResult` to override fields of the result. The merge is field-by-field, not deep:

```ts
afterToolCall: async ({ toolCall, result, isError, context }) => {
  if (toolCall.name === "notify_done" && !isError) {
    return { terminate: true };   // stop the loop after this batch
  }
}
```

The context objects these hooks receive are:

```ts
interface BeforeToolCallContext {
  assistantMessage: AssistantMessage; // the message that requested the call
  toolCall: AgentToolCall;            // the raw tool-call content block
  args: unknown;                      // validated arguments
  context: AgentContext;              // current agent context
}

interface AfterToolCallContext extends BeforeToolCallContext {
  result: AgentToolResult<any>;       // the tool's own result
  isError: boolean;                   // whether the tool threw
}
```

`AgentToolCall` is a helper type that extracts the `{ type: "toolCall" }` variant from `AssistantMessage["content"][number]` — it is the raw content block the model emits to call a tool.

### shouldStopAfterTurn — graceful loop exit

The loop runs until there are no more tool calls and no queued messages. `shouldStopAfterTurn` lets you request a graceful stop after a specific turn:

```ts
shouldStopAfterTurn: async ({ message, toolResults, context, newMessages }) => {
  return shouldCompactBeforeNextTurn(context.messages);
}
```

If it returns `true`, the loop emits `agent_end` and exits cleanly. It does not abort any running tools, does not cancel the provider stream, and does not alter the assistant message's `stopReason`. This is the right hook for "stop before the context window fills up".

### AgentLoopConfig fields at a glance

| Field | Required | Default | Purpose |
|---|---|---|---|
| `model` | yes | — | The model to use |
| `convertToLlm` | yes | — | Translate `AgentMessage[]` to `Message[]` before each LLM call |
| `transformContext` | no | — | Pre-filter messages before `convertToLlm` |
| `getApiKey` | no | — | Dynamic API key resolution (e.g. short-lived OAuth tokens) |
| `shouldStopAfterTurn` | no | — | Return `true` to exit gracefully after the current turn |
| `prepareNextTurn` | no | — | Return replacement context/model/thinking for the next turn |
| `getSteeringMessages` | no | — | Inject mid-run messages after tool calls finish |
| `getFollowUpMessages` | no | — | Inject messages when the agent would otherwise stop |
| `toolExecution` | no | `"parallel"` | Global tool execution mode |
| `beforeToolCall` | no | — | Pre-execution hook; can block a tool |
| `afterToolCall` | no | — | Post-execution hook; can modify result or set `terminate` |

---

## AgentEvent — what the loop emits

We have the data going *in*. Now let us look at what the loop emits *out*. `AgentEvent` is the union type covering every event in a run's lifecycle:

```ts
// agent-core/src/types.ts

export type AgentEvent =
  // Agent lifecycle
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // Turn lifecycle — one turn = one LLM call + its tool calls
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // Message lifecycle — for user, assistant, and toolResult messages
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  // Tool execution lifecycle
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

There are three nested lifecycle levels: the **agent** level (the whole run), the **turn** level (one LLM call plus its tool calls), and the **message/tool** level (individual messages and individual tool executions).

### Event reference table

| Event | Payload fields | Notes |
|---|---|---|
| `agent_start` | — | First event in every run |
| `agent_end` | `messages: AgentMessage[]` | Last event; awaited subscribers still count toward settlement |
| `turn_start` | — | Marks the beginning of one LLM call + its tool calls |
| `turn_end` | `message`, `toolResults` | `toolResults` is the array of `ToolResultMessage` produced in this turn |
| `message_start` | `message` | Emitted for user, assistant, and toolResult messages |
| `message_update` | `message`, `assistantMessageEvent` | **Assistant only** during streaming; `assistantMessageEvent` carries the delta |
| `message_end` | `message` | Message is complete |
| `tool_execution_start` | `toolCallId`, `toolName`, `args` | Tool is about to execute |
| `tool_execution_update` | `toolCallId`, `toolName`, `args`, `partialResult` | Tool is streaming progress (optional) |
| `tool_execution_end` | `toolCallId`, `toolName`, `result`, `isError` | Tool finished; `isError` is `true` if the tool threw |

`message_update` is the event you consume to build a streaming UI — each emission carries the `assistantMessageEvent` from the underlying `EventStream` (a `text_delta`, `thinking_delta`, `tool_call_start`, etc.). The pattern from the README:

```ts
agent.subscribe((event) => {
  if (event.type === "message_update" &&
      event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});
```

### A complete single-turn event sequence

When you call `agent.prompt("Hello!")` and the model responds with plain text (no tools):

```
agent_start
turn_start
message_start     { message: userMessage }
message_end       { message: userMessage }
message_start     { message: assistantMessage (partial) }
message_update    { message: ..., assistantMessageEvent: { type: "text_delta", delta: "Hi" } }
message_update    { message: ..., assistantMessageEvent: { type: "text_delta", delta: " there" } }
message_end       { message: assistantMessage (complete) }
turn_end          { message: assistantMessage, toolResults: [] }
agent_end         { messages: [userMessage, assistantMessage] }
```

When the model calls a tool, the sequence extends:

```
... (same start as above through message_end for the assistant message)
tool_execution_start   { toolCallId: "...", toolName: "read_file", args: { path: "..." } }
tool_execution_update  { ..., partialResult: { content: [...], details: {...} } }  // if onUpdate() called
tool_execution_end     { toolCallId: "...", toolName: "read_file", result: ..., isError: false }
message_start     { message: toolResultMessage }
message_end       { message: toolResultMessage }
turn_end          { message: assistantMessage, toolResults: [toolResultMessage] }
turn_start                                      // loop continues — next LLM call
... (model responds to tool result)
turn_end          { message: ..., toolResults: [] }
agent_end         { messages: [...] }
```

In parallel mode, `tool_execution_end` events arrive as each tool finishes. The `turn_end.toolResults` array always preserves assistant source order regardless.

---

## AgentState — the running agent's public view

When you use the `Agent` class (the high-level wrapper over the loop), you access conversation state through `agent.state`. The `AgentState` interface is the shape of that object:

```ts
// agent-core/src/types.ts

export interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;

  // Setter copies the top-level array before storing
  set tools(tools: AgentTool<any>[]);
  get tools(): AgentTool<any>[];

  // Setter copies the top-level array before storing
  set messages(messages: AgentMessage[]);
  get messages(): AgentMessage[];

  // Read-only derived state
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

Two things stand out: `tools` and `messages` are accessor properties. Assigning a new array to `agent.state.tools = [...]` internally copies the top-level array before storing it — but mutating the returned array mutates the current state directly. This is a performance choice: the copy only happens on assignment, not on every read.

`isStreaming` remains `true` until all awaited `agent_end` subscribers settle — not just until the last model token arrives. `streamingMessage` holds the partial `AssistantMessage` while the model is streaming, which is useful for building optimistic UIs.

---

## How the types compose

We now have the full vocabulary. Let us see how it all fits together in the low-level API — the entry point that the next chapter will explore in depth:

```ts
import { agentLoop } from "agent-core";
import { getModel, stream } from "llm-toolkit";

// Step 1: build the context — the initial conversation state
const context: AgentContext = {
  systemPrompt: "You are a helpful assistant.",
  messages: [],
  tools: [readFileTool],  // our AgentTool from earlier
};

// Step 2: build the config — all the loop's decisions
const config: AgentLoopConfig = {
  model: getModel("anthropic", "claude-sonnet-4-20250514"),

  // stream from llm-toolkit satisfies StreamFn directly
  // (not required explicitly — the loop uses it by default)

  // Required: translate our messages to LLM format
  convertToLlm: (messages) =>
    messages.filter(m => ["user", "assistant", "toolResult"].includes(m.role)),

  // Optional: stop after context grows too large
  shouldStopAfterTurn: async ({ context }) =>
    context.messages.length > 50,
};

// Step 3: run — agentLoop is an async iterable of AgentEvent
const userMessage: AgentMessage = {
  role: "user",
  content: "Read package.json",
  timestamp: Date.now(),
};

for await (const event of agentLoop([userMessage], context, config)) {
  if (event.type === "message_update" &&
      event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
  if (event.type === "agent_end") {
    console.log("\nDone. Total messages:", event.messages.length);
  }
}
```

Every type we met in this chapter has a role here: `AgentContext` holds the conversation, `AgentLoopConfig` configures the loop's decisions, `AgentEvent` is what `for await` yields, and `AgentTool` (via `readFileTool`) tells the model what functions it can call.

---

## What we have built

Let us take stock of the type vocabulary we now understand:

| Type | Role |
|---|---|
| `AgentMessage` | Union of LLM messages + custom app messages |
| `CustomAgentMessages` | Extensible interface; add custom types via declaration merging |
| `AgentContext` | Conversation snapshot: system prompt, messages, tools |
| `AgentTool<TParams, TDetails>` | Tool definition: schema, execute function, execution mode |
| `AgentToolResult<T>` | What a tool returns: content for the model, details for the app |
| `AgentLoopConfig` | All configuration for one agent run |
| `StreamFn` | The model-calling function signature; satisfied by `llm-toolkit`'s `stream` |
| `ToolExecutionMode` | `"sequential"` or `"parallel"` — how tools in one turn are scheduled |
| `QueueMode` | `"one-at-a-time"` or `"all"` — how queued messages are drained |
| `ThinkingLevel` | Reasoning budget: `"off"` through `"xhigh"` |
| `AgentEvent` | The union of all events the loop emits |
| `AgentState` | The `Agent` class's public state interface |

In the next chapter we will take these types and trace exactly what happens inside `agentLoop()` — how it calls the model, handles tool execution in both sequential and parallel modes, drains the steering and follow-up queues, and eventually emits `agent_end`.

---

← Previous: [OAuth and API-Key Authentication](../llm-toolkit/oauth-and-api-key-auth.md) · Next: [The Agent Loop: Turn-by-Turn LLM and Tool Execution](./the-agent-loop.md) →
