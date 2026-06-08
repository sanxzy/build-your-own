---
title: "The EventStream: Observable Backbone for Streaming"
description: "Build the EventStream and AssistantMessageEventStream classes — the Observable pattern that every provider adapter plugs into, turning raw provider bytes into typed, awaitable events."
category: llm-toolkit
type: tutorial
tags: [EventStream, AssistantMessageEventStream, Observable, event, streaming, delta, token, toolCall event, abort, error, diagnostics, token usage, latency, async iterator]
keywords: [EventStream, async iterator, Observable pattern, streaming events, backpressure, result promise]
sources: [S9, S17]
---

**TL;DR** — Raw LLM responses arrive as unstructured byte streams (SSE chunks, WebSocket frames). We need a common event bus that every provider adapter can push typed events into, and every consumer (the agent loop, the UI) can read from. We'll build `EventStream<T, R>` — a generic async-iterable queue with a result promise — and specialize it into `AssistantMessageEventStream` for our event protocol. By the end, you'll have a streaming primitive that decouples providers from consumers.

## The problem: raw bytes to typed events

When the Anthropic API streams a response, it sends Server-Sent Events:

```
event: message_start
data: {"type": "message_start", "message": {...}}

event: content_block_start
data: {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello"}}
```

OpenAI's Responses API uses a different event format. Google Gemini uses yet another. And within each provider, the raw stream needs parsing, error detection, and retry logic.

The rest of our system — the agent loop, the terminal UI, the hook system — shouldn't know or care about any of this. They should receive a clean stream of typed events: `text_delta`, `thinking_start`, `toolcall_end`, `done`, `error`.

We'll solve this with an **Observable pattern**: each provider adapter owns the messy work of parsing its native format, and pushes typed `AssistantMessageEvent` objects into a shared `EventStream`. Everything downstream reads from that stream through a uniform async iterator interface.

## Building `EventStream<T, R>`

Create `packages/llm-toolkit/src/utils/event-stream.ts`. We'll build the generic class first, then specialize it.

The `EventStream` has two jobs:

1. **Be an async iterable** — consumers iterate over events with `for await (const event of stream)`.
2. **Hold a result promise** — consumers can also `await stream.result()` to get the final value without iterating.

```ts
export class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;
  private finalResultPromise: Promise<R>;
  private resolveFinalResult!: (result: R) => void;
  private isComplete: (event: T) => boolean;
  private extractResult: (event: T) => R;

  constructor(isComplete: (event: T) => boolean, extractResult: (event: T) => R) {
    this.isComplete = isComplete;
    this.extractResult = extractResult;
    this.finalResultPromise = new Promise((resolve) => {
      this.resolveFinalResult = resolve;
    });
  }
```

The constructor takes two functions that define what "done" means for this stream type:
- `isComplete` — given an event, does it signal the end of the stream?
- `extractResult` — given that terminal event, what's the final value?

### Pushing events

The producer (a provider adapter) calls `push()` for each event:

```ts
  push(event: T): void {
    if (this.done) return;  // silently ignore events after completion

    if (this.isComplete(event)) {
      this.done = true;
      this.resolveFinalResult(this.extractResult(event));
    }

    // If a consumer is waiting, deliver immediately.
    // Otherwise, queue for later.
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      this.queue.push(event);
    }
  }
```

The key design choice: events go to the fastest path. If a consumer is actively iterating and waiting, the event is delivered directly — no queueing, no memory allocation beyond the event object itself. If no consumer is ready yet, the event joins the queue.

The completion check happens *before* delivery. Once `isComplete(event)` returns true, the stream is closed — `result()` resolves, and future `push()` calls are no-ops.

There's also an `end()` method for producer-initiated termination:

```ts
  end(result?: R): void {
    this.done = true;
    if (result !== undefined) {
      this.resolveFinalResult(result);
    }
    // Wake up all waiting consumers
    while (this.waiting.length > 0) {
      const waiter = this.waiting.shift()!;
      waiter({ value: undefined as any, done: true });
    }
  }
```

This drains the waiter queue — every blocked consumer gets a `done: true` and the async iteration ends cleanly.

### Consuming events

The consumer side implements `AsyncIterable<T>`:

```ts
  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;          // drain the queue first
      } else if (this.done) {
        return;                              // stream ended, stop iterating
      } else {
        // Nothing queued and stream not done — wait for the producer
        const result = await new Promise<IteratorResult<T>>(
          (resolve) => this.waiting.push(resolve)
        );
        if (result.done) return;
        yield result.value;
      }
    }
  }
```

The logic is a three-way branch on each iteration:
1. **Queue has events** → yield the next one immediately.
2. **Queue empty, stream done** → return (stop the iterator).
3. **Queue empty, stream still active** → push a resolver into `waiting` and block. The producer will call it when the next event arrives.

This is backpressure-free by design — the queue grows unbounded if the consumer is slower than the producer. For LLM streaming, this is fine (event rates are bounded by network speed and token generation rate). For higher-throughput scenarios, you'd add a buffer limit.

### Getting the final result

Sometimes you don't want to iterate — you just want the final `AssistantMessage`:

```ts
  result(): Promise<R> {
    return this.finalResultPromise;
  }
```

This is what `completeSimple()` uses internally. The promise resolves when the producer pushes the terminal event (or calls `end()`).

## Specializing to `AssistantMessageEventStream`

Now we specialize for our event protocol:

```ts
import type { AssistantMessage, AssistantMessageEvent } from "../types.ts";

export class AssistantMessageEventStream
  extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",
      (event) => {
        if (event.type === "done") return event.message;
        if (event.type === "error") return event.error;
        throw new Error("Unexpected event type for final result");
      },
    );
  }
}
```

The stream completes on `done` or `error` events. The final result is the completed `AssistantMessage` (with full `content`, `usage`, and `stopReason`), regardless of whether it ended successfully or with an error. This means you can always inspect the final state — check `stopReason` and `errorMessage` to determine what happened.

## How a provider adapter uses this

Here's a simplified view of how the Anthropic adapter (which we'll build in the next chapter) wires into the EventStream:

```ts
// Simplified — the real adapter has error handling, retries, and more
function streamAnthropic(
  model: Model<"anthropic-messages">,
  context: Context,
  options: StreamOptions,
): AssistantMessageEventStream {
  const stream = new AssistantMessageEventStream();

  // Start the async work — the EventStream decouples producer from consumer
  (async () => {
    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "x-api-key": options.apiKey!, "anthropic-version": "2023-06-01" },
        body: JSON.stringify(buildAnthropicRequest(model, context, options)),
      });

      const reader = response.body!.getReader();
      const decoder = new TextDecoder();

      // Parse SSE events and push typed events into the stream
      for await (const sseEvent of parseSSE(reader, decoder)) {
        const typedEvent = mapAnthropicEvent(sseEvent);
        if (typedEvent) stream.push(typedEvent);
      }
    } catch (err) {
      // Errors become error events, not thrown exceptions
      stream.push({
        type: "error",
        reason: "error",
        error: createErrorMessage(err),
      });
    }
  })();

  return stream;
}
```

The pattern: the adapter returns the `EventStream` *immediately* (synchronously), then kicks off the async HTTP work in the background. Events are pushed as they arrive. The consumer starts iterating right away — it doesn't matter whether the first event arrives before or after the first `await`, because the queue handles both cases.

## Error handling: errors are events, not exceptions

A critical contract in our streaming architecture: **provider adapters must never throw**. Every failure — network error, authentication failure, rate limit, invalid response — must be encoded as an `error` event in the stream:

```ts
{ type: "error", reason: "error", error: AssistantMessage }
```

The `AssistantMessage` in the error event has `stopReason: "error"` and `errorMessage` set. The agent loop checks `stopReason` after each turn — it doesn't need try/catch around `streamSimple()`.

Similarly, when the caller aborts the request (via `signal`), the adapter emits:

```ts
{ type: "error", reason: "aborted", error: AssistantMessage }
```

This consistency means the agent loop has exactly one code path for handling completion: check the final `AssistantMessage.stopReason`.

## What we've built

We now have a complete streaming primitive:

- **`EventStream<T, R>`** — a generic async-iterable queue with a result promise. Producers push events; consumers iterate or await the result.
- **`AssistantMessageEventStream`** — the specialized stream for our event protocol, completing on `done` or `error`.
- A clear contract: providers push typed events, never throw; errors are events with a full `AssistantMessage`.

In the next chapter, we'll build our first real provider adapter — the Anthropic Messages API integration — and see the EventStream in action with live network calls.

---

← Previous: [Message Types and the Core Streaming API](./message-types-and-core-api.md) · Next: [Provider Adapter: Anthropic Messages API](./provider-adapter-anthropic.md) →
