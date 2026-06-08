---
title: "The Agent Loop: Turn-by-Turn Reasoning and Tool Execution"
description: "Build the core agent loop — the engine that assembles context, streams LLM responses, detects and executes tool calls, feeds results back, and decides when to stop."
category: agent-core
type: tutorial
tags: [agentLoop, agentLoopContinue, turn, tool execution, tool call, streaming, LLM, context window, abort, queue draining, early termination, agent-core, loop, event]
keywords: [agent loop, turn engine, tool execution cycle, streaming loop, context assembly]
sources: [S21, S20]
---

**TL;DR** — The agent loop is the heart of the system. It's a `while` loop that, each turn: assembles the Context from the agent's working memory, calls `streamSimple()` to get an event stream, processes events to accumulate the assistant's response, detects tool calls, executes them (sequentially or in parallel), feeds results back, and decides whether to take another turn or stop. We'll build it step by step.

## The turn cycle

Each turn of the agent loop has five phases:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 1. Assemble  │ ──→ │ 2. Stream    │ ──→ │ 3. Interpret │ ──→ │ 4. Execute   │ ──→ │ 5. Decide    │
│    Context   │     │    LLM       │     │    Response  │     │    Tools     │     │    Next      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       ↑                                                                                  │
       └──────────────────────────────────────────────────────────────────────────────────┘
                                          (if more turns needed)
```

Create `packages/agent-core/src/agent-loop.ts`.

## 1. Assembling the Context

The agent's `AgentContext` contains the full message history. Before each LLM call, we transform it into the LLM Toolkit's `Context`:

```ts
function buildLlmContext(
  agentContext: AgentContext,
  config: AgentLoopConfig,
): Context {
  return {
    systemPrompt: agentContext.systemPrompt,
    messages: config.convertToLlm(agentContext.messages),
    tools: agentContext.tools.map(t => ({
      name: t.name,
      description: t.description,
      parameters: t.parameters,
    })),
  };
}
```

The `convertToLlm` function (configurable per agent) transforms `AgentMessage[]` to `Message[]`. The default strips any agent-internal message types that the LLM doesn't understand.

## 2. Streaming the LLM

The stream call is straightforward given our LLM Toolkit:

```ts
const stream = streamFn(
  agentContext.model,
  buildLlmContext(agentContext, config),
  {
    apiKey: options?.apiKey,
    reasoning: agentContext.thinkingLevel !== "off"
      ? agentContext.thinkingLevel as ThinkingLevel
      : undefined,
    signal: abortSignal,
    sessionId: config.sessionId,
    cacheRetention: config.cacheRetention,
  },
);
```

The `streamFn` parameter is the indirection we designed in the previous chapter — in production it's `streamSimple`, in tests it's a scripted fake.

## 3. Interpreting the response

We iterate over the event stream and accumulate the assistant's response:

```ts
const contentBlocks: (TextContent | ThinkingContent | ToolCall)[] = [];
let stopReason: StopReason | undefined;
let usage: Usage | undefined;

for await (const event of stream) {
  emitAgentEvent(event);  // notify subscribers

  switch (event.type) {
    case "text_delta":
      accumulateTextBlock(contentBlocks, event.contentIndex, event.delta);
      break;
    case "thinking_delta":
      accumulateThinkingBlock(contentBlocks, event.contentIndex, event.delta);
      break;
    case "toolcall_end":
      contentBlocks[event.contentIndex] = event.toolCall;
      break;
    case "done":
      stopReason = event.reason;
      usage = event.message.usage;
      break;
    case "error":
      stopReason = event.reason;
      usage = event.error.usage;
      break;
  }
}
```

After the stream ends, we build the `AssistantMessage`:

```ts
const assistantMessage: AgentMessage = {
  role: "assistant",
  content: contentBlocks,
  model: agentContext.model.id,
  stopReason: stopReason ?? "stop",
  usage: usage ?? EMPTY_USAGE,
  timestamp: Date.now(),
};

agentContext.messages.push(assistantMessage);
agentContext.usage = accumulateUsage(agentContext.usage, usage);
```

## 4. Executing tools

If the assistant's `stopReason` is `"toolUse"`, we need to execute the requested tools:

```ts
if (stopReason === "toolUse") {
  const toolCalls = contentBlocks.filter(
    (b): b is ToolCall => b.type === "toolCall",
  );

  if (config.toolExecutionMode === "parallel") {
    // Prepare all tool calls first (so hooks can see them)
    const prepared = [];
    for (const tc of toolCalls) {
      const beforeResult = await callBeforeHook(tc, agentContext);
      if (beforeResult.block) {
        prepared.push({ blocked: true, reason: beforeResult.reason, toolCall: tc });
      } else {
        prepared.push({ blocked: false, args: beforeResult.modifiedArgs ?? tc.arguments, toolCall: tc });
      }
    }

    // Execute allowed tools concurrently
    const results = await Promise.all(
      prepared.map(async (p) => {
        if (p.blocked) {
          return createBlockedResult(p.toolCall, p.reason);
        }
        const tool = agentContext.tools.find(t => t.name === p.toolCall.name);
        if (!tool) return createMissingToolResult(p.toolCall);
        try {
          const result = await tool.execute(p.args, agentContext, signal);
          return await callAfterHook(p.toolCall, result, agentContext);
        } catch (err) {
          return createErrorResult(p.toolCall, err);
        }
      }),
    );

    // Add tool results to context
    for (const result of results) {
      agentContext.messages.push(createToolResultMessage(result));
    }

    // Check early termination
    if (results.every(r => r.terminate)) {
      stopReason = "stop";
    }
  } else {
    // Sequential: execute one at a time, feeding results back immediately
    for (const tc of toolCalls) {
      // ... same logic, but one at a time
    }
  }

  // If we executed tools, continue the loop
  if (!earlyTermination) {
    continue; // back to phase 1 with tool results in context
  }
}
```

The early termination check is powerful: if every tool in a batch signals `terminate: true`, the loop stops. This lets a tool say "I've completed the task — no need to ask the LLM what to do next."

## 5. Deciding whether to continue

After tool execution, the loop returns to phase 1. The LLM sees the tool results as new messages and decides whether it needs more tools or is done. The loop continues until:

- The LLM returns `stopReason: "stop"` or `"length"` (natural completion)
- An error occurs (`stopReason: "error"`)
- The caller aborts (`stopReason: "aborted"`)
- Early termination is triggered (all tools in a batch signal `terminate`)

## Queue draining

When the user sends a message while the agent is mid-turn, the message goes into a queue. At drain points (the start of each turn), queued messages are injected:

```ts
function drainQueue(context: AgentContext, queue: AgentMessage[], mode: QueueMode): void {
  if (mode === "all") {
    context.messages.push(...queue.splice(0));
  } else {
    const next = queue.shift();
    if (next) context.messages.push(next);
  }
}
```

`"all"` mode injects everything at once. `"one-at-a-time"` injects one message per turn, which gives the agent a chance to respond to each user message before seeing the next one.

## The complete loop

Putting it all together:

```ts
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): Promise<AgentMessage[]> {
  const fn = streamFn ?? streamSimple;
  const queue: AgentMessage[] = [...prompts];

  while (true) {
    if (signal?.aborted) break;

    // Inject queued messages
    drainQueue(context, queue, config.queueMode ?? "all");
    if (context.messages.length === 0) break;

    // Stream
    const stream = fn(context.model, buildLlmContext(context, config), {
      apiKey: config.apiKey,
      reasoning: context.thinkingLevel !== "off" ? context.thinkingLevel as any : undefined,
      signal,
    });

    emit({ type: "turn_start", context });

    // Interpret
    const { contentBlocks, stopReason, usage } = await interpretStream(stream, emit);
    const assistantMsg = createAssistantMessage(contentBlocks, stopReason, usage);
    context.messages.push(assistantMsg);
    emit({ type: "assistant_message", message: assistantMsg });

    // Execute tools or stop
    if (stopReason === "toolUse") {
      const keepGoing = await executeTools(context, contentBlocks, config, emit, signal);
      if (!keepGoing) break;
      continue;
    }

    break; // stop, length, error, aborted — all terminal
  }

  emit({ type: "turn_end" });
  return context.messages;
}
```

## What we've built

We have a working agent loop. Given an `AgentContext` with tools, a model, and a streaming function, it:

1. Streams the LLM's response
2. Detects tool calls in the response
3. Executes tools (sequentially or in parallel)
4. Feeds results back for another turn
5. Handles errors, aborts, and early termination
6. Manages a message queue for concurrent user input

In the next chapter, we'll wrap this loop in the `Agent` class with a state machine, event subscribers, and lifecycle management.

---

← Previous: [Agent Types, Context, and the Stream Function](./agent-types-and-context.md) · Next: [The Agent Class: State Machine and Lifecycle](./the-agent-class.md) →
