---
title: "The Agent Loop: Turn-by-Turn LLM and Tool Execution"
description: "How agentLoop and agentLoopContinue drive the turn-by-turn cycle of streaming the model, executing tools, and feeding results back in."
category: agent-loop
type: tutorial
tags: [agentLoop, agentLoopContinue, turn, tool execution, tool call, LLM, streaming, agent-core, loop, abort, event, continuation, runLoop, streamAssistantResponse, executeToolCalls, AgentLoopConfig, AgentContext, StreamFn, EventStream, agent_start, turn_start, turn_end, agent_end, tool_execution_start, tool_execution_end, sequential, parallel, shouldStopAfterTurn, prepareNextTurn, getSteeringMessages, getFollowUpMessages, beforeToolCall, afterToolCall, terminate]
keywords: [agent turn cycle, LLM loop, tool call cycle, streaming agent, abort signal, continuation loop, steering messages, follow-up messages, tool batch termination, event sequence, tool execution order, tool result ordering]
sources: [S27, S34]
---

**TL;DR** — An agent is, at its heart, a loop: send the conversation to the model, stream back the reply, execute any tool calls the model requested, append the results, and go around again until the model stops asking for tools. This chapter builds `agentLoop` and `agentLoopContinue` step by step — the two public functions at the core of `agent-core`. By the end you will understand exactly which events fire, in what order, and how the loop handles tool batches, abort signals, steering messages, and graceful termination.

# The Agent Loop: Turn-by-Turn LLM and Tool Execution

> **Prerequisite:** This chapter assumes you know the agent's type vocabulary — `AgentContext`, `AgentTool`, `AgentEvent`, and `StreamFn`. If any of those are unfamiliar, read [Agent Context, Events, and Types](./agent-context-and-types.md) first, then return here. A brief recap appears below when each type is first used.

## The core idea: a loop, not a single call

When we give a language model access to tools, a single round-trip is rarely enough. The model may reply with a tool call ("please read that file"), we execute the tool, and then the model needs to see the result before it can finish. So instead of one call, we run a loop:

1. Send the full conversation history to the model and stream its reply.
2. If the reply contains tool calls, execute them, collect their results, and add everything to the conversation.
3. Go back to step 1 with the now-longer conversation.
4. Stop when the model's reply contains no tool calls (it produced text and stopped).

That is the entire concept. The implementation in `agent-core` (`src/agent-loop.ts`) realises this loop in roughly four layers, each of which we will build up one at a time:

| Layer | Function | Responsibility |
|---|---|---|
| Public entry point (new prompt) | `agentLoop` | Add the prompt messages, set up the event stream, kick off the loop |
| Public entry point (resume) | `agentLoopContinue` | Resume from an existing context without re-emitting the prompt |
| Shared async driver | `runLoop` | The while-loop that sequences turns |
| Per-turn streaming | `streamAssistantResponse` | Convert context → LLM messages, call `StreamFn`, collect events |
| Per-turn tool execution | `executeToolCalls` | Run tools in parallel or sequential, collect results |

Let's build each layer in turn.

---

## Layer 1 — The event stream wrapper

Before we get to the loop itself, we need a place to put the events the loop emits. The `EventStream<AgentEvent, AgentMessage[]>` type (from `llm-toolkit`) acts as both an async iterable (so callers can `for await` over events) and a promise-like that eventually resolves to the accumulated messages.

The private helper `createAgentStream` constructs one:

```ts
// Simplified view — from agent-loop.ts
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
  return new EventStream<AgentEvent, AgentMessage[]>(
    // The stream closes when it sees this event type
    (event: AgentEvent) => event.type === "agent_end",
    // The final value is extracted from that closing event
    (event: AgentEvent) => (event.type === "agent_end" ? event.messages : []),
  );
}
```

The `EventStream` constructor takes two functions: a predicate that identifies the terminal event (here `"agent_end"`), and a function that extracts the final value from it. Every event pushed to the stream is visible to anyone iterating it; when `"agent_end"` arrives, the stream closes and the caller can call `stream.result()` to get the `AgentMessage[]`.

An `AgentEventSink` is a function that accepts one `AgentEvent` and returns `Promise<void> | void`. The loop uses it to push events into the stream without needing a direct reference to the stream object:

```ts
export type AgentEventSink = (event: AgentEvent) => Promise<void> | void;
```

---

## Layer 2 — `agentLoop`: starting with a new prompt

Now we can build the first public entry point. `agentLoop` is what you call when a user types a new message and you want the agent to respond:

```ts
// Simplified view of the public entry point — agent-loop.ts
export function agentLoop(
  prompts: AgentMessage[],    // The new user message(s) to add
  context: AgentContext,      // System prompt + conversation history + registered tools
  config: AgentLoopConfig,    // Model, API key, hooks (beforeToolCall, etc.)
  signal?: AbortSignal,       // Optional: caller can cancel mid-loop
  streamFn?: StreamFn,        // Optional: override the streaming backend (useful in tests)
): EventStream<AgentEvent, AgentMessage[]> {
  const stream = createAgentStream();

  // Fire-and-forget: run the async loop, push events into the stream as they happen
  void runAgentLoop(prompts, context, config, async (event) => {
    stream.push(event);
  }, signal, streamFn).then((messages) => {
    stream.end(messages);
  });

  return stream; // Caller receives the stream immediately — no awaiting yet
}
```

`agentLoop` is intentionally synchronous in its return. It builds the stream, starts `runAgentLoop` (an async function) without awaiting it, and hands the stream back to the caller. The caller then iterates:

```ts
const stream = agentLoop([userMessage], context, config);

for await (const event of stream) {
  // handle streaming events: text deltas, tool calls, ...
}

const messages = await stream.result(); // ["user", "assistant", "toolResult", "assistant", ...]
```

Now let's look at what `runAgentLoop` actually does.

### `runAgentLoop` — stitching prompts into the context

`runAgentLoop` does two things before handing off to the shared loop:

1. It merges the new prompt messages into the context's existing messages.
2. It emits the `"agent_start"`, `"turn_start"`, and per-prompt `"message_start"` / `"message_end"` events so observers see the full picture from the beginning.

```ts
// Simplified view — runAgentLoop in agent-loop.ts
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): Promise<AgentMessage[]> {
  const newMessages: AgentMessage[] = [...prompts];

  // Build a new context snapshot that includes the incoming prompts
  const currentContext: AgentContext = {
    ...context,
    messages: [...context.messages, ...prompts],
  };

  await emit({ type: "agent_start" });
  await emit({ type: "turn_start" });

  // Emit message events for each prompt so observers see them
  for (const prompt of prompts) {
    await emit({ type: "message_start", message: prompt });
    await emit({ type: "message_end", message: prompt });
  }

  // Hand off to the shared loop driver
  await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
  return newMessages;
}
```

`AgentContext` — a quick recap: it holds the conversation history (`messages: AgentMessage[]`), the system prompt, and the list of registered `AgentTool` objects. See [Agent Context, Events, and Types](./agent-context-and-types.md) for the full type definition.

Notice that `runAgentLoop` builds a *new* context snapshot rather than mutating the caller's object. This is intentional: the loop needs to track growing message lists without surprising the caller's reference.

---

## Layer 3 — `agentLoopContinue`: resuming without a new prompt

Sometimes we do not have a new user message — we only need to re-run the model on an existing context. This happens on retries, or when the context already ends with a tool-result message that needs a model response. `agentLoopContinue` handles this:

```ts
// Simplified view — agentLoopContinue in agent-loop.ts
export function agentLoopContinue(
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
  // Guard: we must have at least one message to continue from
  if (context.messages.length === 0) {
    throw new Error("Cannot continue: no messages in context");
  }
  // Guard: we cannot continue from an assistant message — the model would see itself speaking last
  if (context.messages[context.messages.length - 1].role === "assistant") {
    throw new Error("Cannot continue from message role: assistant");
  }

  const stream = createAgentStream();

  void runAgentLoopContinue(context, config, async (event) => {
    stream.push(event);
  }, signal, streamFn).then((messages) => {
    stream.end(messages);
  });

  return stream;
}
```

The two guards are worth noting:

- **Empty context** — there is nothing to continue from; the caller must have made a programming error.
- **Last message is assistant** — the LLM API expects that the final message in the conversation is from the user (or a tool result). If the assistant spoke last, the next call would violate that contract.

`agentLoopContinue` differs from `agentLoop` in one key way that the tests confirm: it does **not** emit `message_start` / `message_end` events for the pre-existing messages, and it returns only the *new* messages generated during the continuation (not the ones already in context):

```ts
// From runAgentLoopContinue — agent-loop.ts
const newMessages: AgentMessage[] = []; // starts empty — only new messages accumulate here

await emit({ type: "agent_start" });
await emit({ type: "turn_start" });

await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
return newMessages;
```

The test in S34 confirms this: when a context with one user message is continued, `stream.result()` returns exactly 1 message (the new assistant reply), not 2.

---

## Layer 4 — `runLoop`: the turn-by-turn driver

`runLoop` is the private function that both `runAgentLoop` and `runAgentLoopContinue` delegate to. It is the heart of the agent. Let's read it in chunks.

### The two loops

```ts
// Simplified view of runLoop's structure — agent-loop.ts
async function runLoop(
  initialContext: AgentContext,
  newMessages: AgentMessage[],
  initialConfig: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
): Promise<void> {
  let currentContext = initialContext;
  let config = initialConfig;
  let firstTurn = true;
  let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

  // Outer loop: re-enters when follow-up messages arrive after the agent would have stopped
  while (true) {
    let hasMoreToolCalls = true;

    // Inner loop: continues as long as there are tool calls or pending steering messages
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      if (!firstTurn) await emit({ type: "turn_start" });
      else firstTurn = false;

      // ... (inject steering, stream model, execute tools, check termination)
    }

    // Agent would stop here — check for follow-up messages
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) {
      pendingMessages = followUpMessages;
      continue; // re-enter outer loop
    }

    break; // nothing left to do
  }

  await emit({ type: "agent_end", messages: newMessages });
}
```

There are two loops here for two different reasons:

- **Inner loop**: handles the normal case — the model requests tools, we run them and loop back. Also handles *steering messages* (mid-turn user interruptions) that need to be injected before the next model call.
- **Outer loop**: handles *follow-up messages* — new messages that arrive *after* the agent would have naturally stopped. The outer loop re-enters the inner loop with those follow-ups as pending messages.

`getSteeringMessages` and `getFollowUpMessages` are optional hooks on `AgentLoopConfig`. They let the caller inject new messages asynchronously — for example, because the user typed something while the agent was working.

### One full inner-loop iteration

Here is what happens inside a single inner-loop pass:

```ts
// Inside the inner while loop — simplified view
// 1. Emit turn_start (skipped on the very first turn because the caller already emitted it)
if (!firstTurn) await emit({ type: "turn_start" });
else firstTurn = false;

// 2. Inject pending steering messages into context before calling the model
if (pendingMessages.length > 0) {
  for (const message of pendingMessages) {
    await emit({ type: "message_start", message });
    await emit({ type: "message_end", message });
    currentContext.messages.push(message);
    newMessages.push(message);
  }
  pendingMessages = [];
}

// 3. Call the model, streaming its reply
const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFn);
newMessages.push(message);

// 4. If streaming produced an error or an abort, stop immediately
if (message.stopReason === "error" || message.stopReason === "aborted") {
  await emit({ type: "turn_end", message, toolResults: [] });
  await emit({ type: "agent_end", messages: newMessages });
  return;
}

// 5. Execute any tool calls the model requested
const toolCalls = message.content.filter((c) => c.type === "toolCall");
const toolResults: ToolResultMessage[] = [];
hasMoreToolCalls = false;

if (toolCalls.length > 0) {
  const batch = await executeToolCalls(currentContext, message, config, signal, emit);
  toolResults.push(...batch.messages);
  hasMoreToolCalls = !batch.terminate; // if all tools said terminate, we stop

  for (const result of toolResults) {
    currentContext.messages.push(result);
    newMessages.push(result);
  }
}

// 6. Emit turn_end with the assistant message and its tool results
await emit({ type: "turn_end", message, toolResults });

// 7. Give the caller a chance to swap the context or model for the next turn
const snapshot = await config.prepareNextTurn?.({ message, toolResults, context: currentContext, newMessages });
if (snapshot) {
  currentContext = snapshot.context ?? currentContext;
  // model/reasoning may also be updated here
}

// 8. Give the caller a chance to stop the loop after this turn
if (await config.shouldStopAfterTurn?.({ message, toolResults, context: currentContext, newMessages })) {
  await emit({ type: "agent_end", messages: newMessages });
  return;
}

// 9. Collect any new steering messages before the next turn
pendingMessages = (await config.getSteeringMessages?.()) || [];
```

That is the complete single-turn sequence. Notice that `hasMoreToolCalls` starts as `true` so the first pass always runs, and it is only set to `false` when there were no tool calls *or* when the tool batch signalled termination.

---

## Layer 5 — `streamAssistantResponse`: calling the model

`streamAssistantResponse` is where the `AgentMessage[]` conversation history gets translated into what the LLM API actually expects, and where the streaming events flow.

`AgentLoopConfig.convertToLlm` — a brief recap: this is a mandatory function on the config that translates the agent's internal `AgentMessage[]` format into the `Message[]` format the LLM provider understands. See [Agent Context, Events, and Types](./agent-context-and-types.md) for the full shape.

```ts
// Simplified view — streamAssistantResponse in agent-loop.ts
async function streamAssistantResponse(
  context: AgentContext,
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
): Promise<AssistantMessage> {
  // Optional: transform the raw AgentMessage[] before conversion (e.g., prune old messages)
  let messages = context.messages;
  if (config.transformContext) {
    messages = await config.transformContext(messages, signal);
  }

  // Convert to LLM-compatible messages (e.g., filter custom message types)
  const llmMessages = await config.convertToLlm(messages);

  // Build the LLM context object
  const llmContext: Context = {
    systemPrompt: context.systemPrompt,
    messages: llmMessages,
    tools: context.tools,
  };

  // Use caller-supplied streamFn, or the default streamSimple from llm-toolkit
  const streamFunction = streamFn || streamSimple;

  // Resolve the API key (may be dynamic / expiring)
  const resolvedApiKey =
    (config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

  const response = await streamFunction(config.model, llmContext, {
    ...config,
    apiKey: resolvedApiKey,
    signal,
  });

  // Consume the streaming response, emitting events for each chunk
  for await (const event of response) {
    // ...emit message_start, message_update per chunk, message_end at the end
  }

  return finalMessage;
}
```

`StreamFn` — a quick recap: a function that takes a model descriptor and an LLM context, and returns an async iterable of streaming events. Passing your own `streamFn` is how tests inject a mock without touching the network.

The key transformation chain is:

```
AgentMessage[]
  → (transformContext, optional)
  → AgentMessage[]
  → convertToLlm (mandatory)
  → Message[]
  → streamFn → streaming events
```

The test in S34 verifies this order: `transformContext` is called first and its output is what `convertToLlm` receives, not the original messages.

### Streaming events emitted per turn

As the model streams its reply, `streamAssistantResponse` emits a sequence of `AgentEvent` objects:

| Streaming event | When emitted |
|---|---|
| `message_start` | Once, when the first chunk arrives (partial message created) |
| `message_update` | For every chunk: `text_start`, `text_delta`, `text_end`, `thinking_start/delta/end`, `toolcall_start/delta/end` |
| `message_end` | Once, when the LLM signals `done` or `error` with the final `AssistantMessage` |

A partial `AssistantMessage` is pushed into `context.messages` immediately at `message_start` (so the context is always up to date even mid-stream). It is replaced with the final message at `message_end`.

---

## Layer 6 — `executeToolCalls`: parallel vs. sequential

Once the model replies with one or more tool calls, we need to execute them. The question is: in what order?

By default the loop runs tools in parallel. But some tools are not safe to run concurrently — for example, a file-writing tool that must not race with itself. Two mechanisms control this:

| Mechanism | Takes priority | How |
|---|---|---|
| `AgentLoopConfig.toolExecution = "sequential"` | Config-level | Forces all tools sequential for this config |
| `AgentTool.executionMode = "sequential"` | Tool-level | If *any* tool in the batch sets this, the whole batch becomes sequential |

```ts
// executeToolCalls dispatch — agent-loop.ts
function executeToolCalls(...): Promise<ExecutedToolCallBatch> {
  const hasSequentialToolCall = toolCalls.some(
    (tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
  );
  if (config.toolExecution === "sequential" || hasSequentialToolCall) {
    return executeToolCallsSequential(...);
  }
  return executeToolCallsParallel(...);
}
```

The tests in S34 verify this precisely: when a tool declares `executionMode: "sequential"`, the second tool in the batch does not start until the first has finished, even if the config does not set `toolExecution: "sequential"`.

### The parallel execution order subtlety

Parallel mode starts all tools at once, but it **preserves the source-order** of tool results. This means `tool_execution_end` events may arrive out of order (whichever tool finishes first fires first), but the `ToolResultMessage` objects pushed into the context, and the `toolResults` array on the `turn_end` event, always match the order the model originally listed the tool calls.

The S34 test for this:

```ts
// Parallel tool test — from agent-loop.test.ts
// tool-1 ("first") is slow; tool-2 ("second") is fast
expect(toolExecutionEndIds).toEqual(["tool-2", "tool-1"]);  // completion order
expect(toolResultIds).toEqual(["tool-1", "tool-2"]);        // source order preserved
expect(turnToolResultIds).toEqual(["tool-1", "tool-2"]);    // source order preserved
```

### Per-tool lifecycle and hooks

Each tool call goes through a fixed lifecycle. Two hooks on `AgentLoopConfig` let you intercept it:

| Hook | Called | Can do |
|---|---|---|
| `beforeToolCall` | Before the tool executes | Block execution (`return { block: true, reason: "..." }`), or mutate `args` in place |
| `afterToolCall` | After the tool returns | Rewrite the result content, flip `isError`, or set `terminate: true` |

If the tool name is not found in `context.tools`, execution returns an immediate error result (`"Tool X not found"`) without calling any hook.

If `signal?.aborted` is true at any check point inside the per-tool lifecycle, execution short-circuits with an `"Operation aborted"` error result.

Events emitted per tool call:

| Event | Payload |
|---|---|
| `tool_execution_start` | `toolCallId`, `toolName`, `args` |
| `tool_execution_update` | `toolCallId`, `toolName`, `args`, `partialResult` (for streaming tools) |
| `tool_execution_end` | `toolCallId`, `toolName`, `result`, `isError` |
| `message_start` + `message_end` | The resulting `ToolResultMessage` |

### Termination via tool results

A tool can set `terminate: true` on its returned `AgentToolResult`. The loop checks: if **every** tool in a batch sets `terminate: true`, the batch's `terminate` flag is `true`, and `hasMoreToolCalls` is set to `false`. The agent stops looping after that turn without calling the model again.

If only *some* tools in a parallel batch set `terminate: true`, the batch is not terminated — the loop continues as normal.

---

## Putting it together: the complete event sequence

Here is the exact event sequence for a two-turn exchange where the model makes one tool call in the first turn, then gives a plain text reply in the second. This is confirmed by the S34 test `"should stop after the current turn when shouldStopAfterTurn returns true"` (which asserts the full `events.map(e => e.type)` array):

```
agent_start
turn_start
  message_start    ← user prompt
  message_end
  message_start    ← assistant reply (tool call)
  message_end
  tool_execution_start
  tool_execution_end
  message_start    ← tool result
  message_end
turn_end
agent_end
```

For a two-turn exchange (tool call in turn 1, plain reply in turn 2):

```
agent_start
turn_start
  message_start    ← user prompt
  message_end
  message_start    ← assistant reply (tool call)
  message_end
  tool_execution_start
  tool_execution_end
  message_start    ← tool result
  message_end
turn_end
turn_start         ← second turn begins
  message_start    ← final assistant reply
  message_end
turn_end
agent_end
```

One `"turn_start"` per model call; one `"turn_end"` per model reply (including its tool results). The `"agent_end"` event carries the full list of new messages accumulated since the loop started.

---

## Termination paths

The loop ends in one of four ways:

| Path | How it happens |
|---|---|
| No tool calls | Model returns a plain text reply; `hasMoreToolCalls` is `false`; inner loop exits |
| Tool batch terminates | Every tool result has `terminate: true` |
| `shouldStopAfterTurn` returns `true` | Config hook fires after `turn_end`; loop exits immediately |
| Error or abort | `message.stopReason` is `"error"` or `"aborted"`; loop exits immediately |

In every path, `"agent_end"` is always the last event.

### Abort mid-turn

Passing an `AbortSignal` lets the caller cancel the loop at any point. The signal is threaded through to both the streaming call and each tool execution. When aborted:

- If streaming is in progress, the stream produces a final message with `stopReason: "aborted"`, the loop emits `turn_end` and `agent_end`, and returns.
- If a tool is executing, `signal.aborted` is checked before and after `beforeToolCall` and before calling `tool.execute`. An aborted tool produces an immediate `"Operation aborted"` error result. In sequential mode, the loop also checks `signal.aborted` after each tool and breaks the loop if it is set.

---

## Wiring `agentLoopContinue` correctly

`agentLoopContinue` has one important constraint the source makes explicit:

> **The last message in context must convert to a `user` or `toolResult` message via `convertToLlm`.** If it does not, the LLM provider will reject the request.

This cannot be validated inside `agentLoopContinue` because `convertToLlm` is only called inside `streamAssistantResponse`. The two guards that *are* checked synchronously at call time:

1. `context.messages.length === 0` → throws `"Cannot continue: no messages in context"`
2. `context.messages[context.messages.length - 1].role === "assistant"` → throws `"Cannot continue from message role: assistant"`

The second guard uses the agent's own `role` field on the last message, not the LLM message format. Custom message types (e.g., `role: "custom"`) pass this guard — the caller is responsible for ensuring their `convertToLlm` will map that custom role to something the LLM accepts.

---

## Summary of `AgentLoopConfig` hooks

`AgentLoopConfig` is the configuration object passed to `agentLoop` and `agentLoopContinue`. Here are the hooks it exposes, in the order the loop calls them:

| Hook | Required | Called when |
|---|---|---|
| `convertToLlm` | **Yes** | Every turn, to translate `AgentMessage[]` to `Message[]` |
| `transformContext` | No | Every turn, before `convertToLlm`, to prune/transform the message list |
| `getSteeringMessages` | No | Before the first turn, and at the end of each turn, to inject mid-session user messages |
| `beforeToolCall` | No | Before each tool executes; can block or mutate args |
| `afterToolCall` | No | After each tool returns; can rewrite result or set `terminate` |
| `prepareNextTurn` | No | After `turn_end`; can swap `context`, `model`, or `thinkingLevel` |
| `shouldStopAfterTurn` | No | After `prepareNextTurn`; return `true` to stop the loop |
| `getFollowUpMessages` | No | When the inner loop would exit; return messages to re-enter the outer loop |
| `getApiKey` | No | Every turn; for dynamic/expiring API keys |

<!-- GAP: The AgentLoopConfig type definition is in src/types.ts (S26), which is not an assigned source for this page. The field-by-field type signatures and defaults cannot be confirmed from S27/S34 alone; the table above is derived from observable usage in S27 and confirmed behaviors in S34, but exact TypeScript types for each hook's parameter shapes are not verified here. -->

---

## A minimal working example

Here is the smallest complete usage of `agentLoop` you can run. It uses a mock stream function so there is no real LLM call:

```ts
import { agentLoop } from "agent-core";
import type { AgentContext, AgentLoopConfig, AgentMessage } from "agent-core";

// A minimal context: empty history, no tools
const context: AgentContext = {
  systemPrompt: "You are a helpful assistant.",
  messages: [],
  tools: [],
};

// The new user message we are sending
const prompt: AgentMessage = {
  role: "user",
  content: "Hello!",
  timestamp: Date.now(),
};

// Config: only convertToLlm is mandatory; it filters to the message types the LLM accepts
const config: AgentLoopConfig = {
  model: yourModelDescriptor,
  convertToLlm: (messages) =>
    messages.filter(
      (m) => m.role === "user" || m.role === "assistant" || m.role === "toolResult",
    ),
};

const stream = agentLoop([prompt], context, config);

for await (const event of stream) {
  if (event.type === "message_update") {
    // Stream text deltas to the UI
    process.stdout.write(event.message.content?.[0]?.text ?? "");
  }
}

const newMessages = await stream.result();
console.log(`Loop completed. ${newMessages.length} new message(s).`);
```

And a minimal `agentLoopContinue` for a retry scenario where the context already has a user message:

```ts
import { agentLoopContinue } from "agent-core";

// Context already contains the user's message from a previous failed attempt
const contextWithExistingMessage: AgentContext = {
  systemPrompt: "You are helpful.",
  messages: [{ role: "user", content: "Retry this.", timestamp: Date.now() }],
  tools: [],
};

// Resume without re-adding the user message
const stream = agentLoopContinue(contextWithExistingMessage, config);

for await (const event of stream) {
  // handle events
}

// result() contains only the NEW messages (just the assistant reply)
const newMessages = await stream.result();
```

---

← Previous: [Agent Context, Events, and Types](./agent-context-and-types.md) · Next: [The Agent Class: State Machine, Steering, and Lifecycle](./the-agent-class.md) →
