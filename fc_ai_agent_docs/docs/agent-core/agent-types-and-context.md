---
title: "Agent Types, Context, and the Stream Function"
description: "Define the type vocabulary of the agent framework — AgentContext, AgentLoopConfig, AgentTool, hooks, and the StreamFn contract that bridges the LLM Toolkit to the agent loop."
category: agent-core
type: tutorial
tags: [AgentContext, AgentLoopConfig, AgentEvent, AgentTool, StreamFn, ToolExecutionMode, BeforeToolCall, AfterToolCall, hooks, agent-core, types, state]
keywords: [agent types, AgentContext, tool execution mode, hooks, StreamFn contract, agent state machine]
sources: [S19, S5]
---

**TL;DR** — The LLM Toolkit handles talking to models. The agent core handles *deciding what to do*. We need a new set of types — `AgentContext` (the agent's working memory), `AgentTool` (tools with execution functions), hooks for intercepting tool calls, and a `StreamFn` contract — that bridge raw LLM streaming to intelligent, multi-turn behavior.

## The AgentContext: the agent's working memory

Where the LLM Toolkit has `Context` (a snapshot sent to one API call), the agent has `AgentContext` — persistent state that lives across turns:

```ts
export interface AgentContext {
  systemPrompt: string;
  messages: AgentMessage[];
  tools: AgentTool[];
  model: Model;
  thinkingLevel: ModelThinkingLevel;
  usage: Usage;  // cumulative across all turns
  mode?: string;
  cwd?: string;
  skills?: SkillDefinition[];
  extensions?: ExtensionContext[];
}
```

The key differences from the LLM Toolkit's `Context`:

- **`messages`** uses `AgentMessage[]` — our internal richer message type that includes event metadata, not just the LLM-facing `Message[]`.
- **`usage`** is cumulative — every turn adds to it, so we always know the total session cost.
- **`tools`** are `AgentTool[]` — these include execution logic, not just schemas.
- **`mode`, `cwd`, `skills`, `extensions`** are behavioral state that influences prompt construction and available capabilities.

## AgentTool: a tool that can execute

The LLM Toolkit's `Tool` type is just a schema — name, description, parameters. The agent needs to actually *run* tools:

```ts
export interface AgentTool<TDetails = any> {
  name: string;
  description: string;
  parameters: TSchema;
  execute: (
    args: any,
    context: AgentContext,
    signal?: AbortSignal,
  ) => Promise<AgentToolResult>;
}
```

The `execute` function receives the validated arguments, the full agent context (so tools can read the message history, check the model, etc.), and an abort signal for cancellation.

## Hooks: intercepting the tool lifecycle

Two hooks let the agent harness (and extensions) intercept every tool execution:

```ts
export interface BeforeToolCallContext {
  assistantMessage: AssistantMessage;
  toolCall: AgentToolCall;
  args: unknown;          // validated arguments
  context: AgentContext;
}

export interface BeforeToolCallResult {
  block?: boolean;        // prevent execution
  reason?: string;        // shown when blocked
  modifiedArgs?: unknown; // override the arguments
}

export type BeforeToolCallHook = (
  ctx: BeforeToolCallContext,
) => BeforeToolCallResult | Promise<BeforeToolCallResult>;
```

The `BeforeToolCall` hook can block dangerous tool calls before they execute. An extension might check: "Is this bash command trying to `rm -rf /`? Block it."

```ts
export interface AfterToolCallContext {
  assistantMessage: AssistantMessage;
  toolCall: AgentToolCall;
  result: AgentToolResult;
  context: AgentContext;
}

export interface AfterToolCallResult {
  content?: (TextContent | ImageContent)[];  // replace the result content
  details?: unknown;                          // replace structured details
  isError?: boolean;                          // override error flag
  terminate?: boolean;                        // signal early stop
}

export type AfterToolCallHook = (
  ctx: AfterToolCallContext,
) => AfterToolCallResult | Promise<AfterToolCallResult>;
```

The `AfterToolCall` hook can modify results before they reach the LLM, or signal that the agent should stop after the current batch. The `terminate` flag is how tools signal "I'm done, no need for another turn" — if every tool in a batch sets `terminate: true`, the loop stops.

## Tool execution modes

Tools can be executed sequentially or in parallel:

```ts
export type ToolExecutionMode = "sequential" | "parallel";
```

- **`"sequential"`** — each tool call is prepared, executed, and finalized before the next one starts. Slower but safer when tool calls depend on each other.
- **`"parallel"`** — tool calls are prepared sequentially (so hooks can see them all), then allowed tools execute concurrently. Results are finalized in completion order. Faster for independent operations.

## Queue mode

When the user sends a message while the agent is mid-turn, it goes into a queue. The `QueueMode` controls when queued messages are injected:

```ts
export type QueueMode = "all" | "one-at-a-time";
```

- **`"all"`** — drain every queued message at the next drain point.
- **`"one-at-a-time"`** — inject only the oldest queued message, leaving the rest for later. Lets the user interject without flooding the context.

## StreamFn contract

The agent loop doesn't call `streamSimple()` directly. It accepts a `StreamFn` — any function matching the signature:

```ts
export type StreamFn = (
  ...args: Parameters<typeof streamSimple>
) => ReturnType<typeof streamSimple>;
```

This indirection is critical for testing. The agent loop can be exercised with a fake streaming function that returns scripted responses — no network calls, no API keys. The `StreamFn` contract specifies:

1. Must not throw for request/model/runtime failures.
2. Must return an `AssistantMessageEventStream`.
3. Failures must be encoded in the stream via `error` events and a final `AssistantMessage` with `stopReason: "error"` or `"aborted"`.

## The state machine

The Agent class tracks a state machine:

```ts
export type AgentState = "idle" | "running" | "stopping";
```

- **`idle`** — no active turn. Ready to accept a new prompt.
- **`running`** — mid-turn, processing an LLM response or executing tools.
- **`stopping`** — abort has been requested. Finishing the current operation, then returning to idle.

Transitions: `idle → running` (new prompt) → `idle` (turn complete) or `stopping` (abort) → `idle`.

## What we've built

We've defined the complete type vocabulary that sits between the LLM Toolkit and the agent loop. In the next chapter, we'll implement the loop itself — the turn-by-turn engine that assembles context, streams responses, executes tools, and decides when to stop.

---

← Previous: [Model Registry, Costs, and Streaming JSON](../llm-toolkit/models-and-streaming-json.md) · Next: [The Agent Loop: Turn-by-Turn Reasoning and Tool Execution](./the-agent-loop.md) →
