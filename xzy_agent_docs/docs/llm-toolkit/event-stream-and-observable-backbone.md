---
title: "The EventStream Observable Backbone"
description: "Build the EventStream and AssistantMessageEventStream classes that let every provider push streaming events and callers consume them as an async iterable."
category: llm-toolkit
type: tutorial
tags: [EventStream, AssistantMessageEventStream, observable, event, streaming, delta, token, tool call event, abort, error handling, llm-toolkit, async iterable, AsyncIterator, push, backpressure, queue, done event, error event, text_start, text_delta, text_end, toolcall_start, toolcall_delta, toolcall_end, thinking_start, thinking_delta, thinking_end, result, streaming response]
keywords: [observable stream, event queue, async iteration, streaming LLM, push consumer, pull producer, for-await-of, Symbol.asyncIterator, streaming backbone, event sequence]
sources: [S10, S21]
---

**TL;DR** — When an LLM streams its reply, tokens and tool calls arrive one at a time over an open connection. We need a clean boundary between the *producer* (the provider adapter that receives raw bytes) and the *consumer* (the application code that reacts to events). This chapter builds `EventStream` — a generic push-in / pull-out observable — and then `AssistantMessageEventStream`, the concrete specialisation every provider adapter returns. By the end you will understand exactly what events arrive, in what order, and how to handle abort and error cases.

# The EventStream Observable Backbone

## The problem: a streaming response has no single "done" moment

Recall from [Message Types and the Unified Streaming API](./message-types-and-streaming-api.md) that the `stream()` entry point returns a value you can iterate over with `for await … of`. Each iteration step hands you one `AssistantMessageEvent` — a small typed object representing a token delta, the start of a tool call, a finish signal, and so on.

But there is a tension here. The *provider adapter* (the code that speaks HTTP/SSE to the model API) learns about events as they arrive from the network — it is a **producer** that pushes data. The *caller* in your application walks the events at its own pace — it is a **consumer** that pulls data.

You need something that lets the producer push whenever a chunk arrives, lets the consumer pull whenever it is ready, and buffers any mismatch between the two. That something is `EventStream`.

## Building EventStream from first principles

Let's start with a minimal mental model: a first-in-first-out queue between producer and consumer, plus a way to signal "no more events will come".

### Step 1 — The queue and the "waiting" slot

The core idea is that either events are piling up in a queue (producer is faster than the consumer), or the consumer is waiting for the next event (consumer is faster than the producer). We never need to hold more than one pending consumer at a time because `for await … of` issues requests serially.

```ts
// Simplified view — just the queue mechanics
class EventStream<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  push(event: T): void {
    // Deliver to a waiting consumer if one exists; otherwise buffer it
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      this.queue.push(event);
    }
  }

  end(): void {
    this.done = true;
    // Wake up any consumer that is already waiting
    while (this.waiting.length > 0) {
      const waiter = this.waiting.shift()!;
      waiter({ value: undefined as any, done: true });
    }
  }
}
```

Notice the two paths in `push()`:
- If a consumer called `next()` and is already waiting on a Promise, we resolve that Promise immediately.
- If no consumer is waiting yet, we append to `queue` so the event is ready when the consumer does ask.

`end()` wakes every pending consumer with `done: true`, which is the signal that tells `for await … of` to stop.

### Step 2 — Making it async-iterable

JavaScript's `for await … of` loop works with any object that implements the `AsyncIterable<T>` protocol — that means exposing a `[Symbol.asyncIterator]()` method that returns an `AsyncIterator<T>`. Let's add that.

```ts
async *[Symbol.asyncIterator](): AsyncIterator<T> {
  while (true) {
    if (this.queue.length > 0) {
      // Events already buffered — yield immediately
      yield this.queue.shift()!;
    } else if (this.done) {
      // No events and producer is finished — stop iteration
      return;
    } else {
      // Nothing yet — park here until the producer calls push() or end()
      const result = await new Promise<IteratorResult<T>>(
        (resolve) => this.waiting.push(resolve)
      );
      if (result.done) return;
      yield result.value;
    }
  }
}
```

The loop has three branches that match the three states the stream can be in at any moment:
1. **Buffered events available** — serve them without suspending.
2. **No events and stream is closed** — return, ending the `for await` loop.
3. **No events yet and stream is open** — park by adding a resolver to `waiting`; the producer will call it when the next event arrives.

You might wonder: what if the producer calls `push()` *while* we are in the `await new Promise(...)` ? That is exactly the case handled by the `waiter` branch in `push()` — the resolver is already in `this.waiting`, so `push()` calls it immediately and the `await` resolves.

### Step 3 — The final result promise

Streaming is all well and good, but once the loop finishes you often want the *completed* value — the assembled `AssistantMessage` with the full accumulated text, usage counts, and stop reason. We cannot extract that from the last event mid-loop without complicating caller code.

The solution is a separate `Promise<R>` that resolves exactly once, when the stream identifies its "completion event". This gives callers two clean interfaces: `for await` for incremental events, and `await stream.result()` for the final assembled object.

```ts
// The real EventStream constructor (from S10)
constructor(
  isComplete: (event: T) => boolean,
  extractResult: (event: T) => R
) {
  this.isComplete = isComplete;
  this.extractResult = extractResult;
  this.finalResultPromise = new Promise((resolve) => {
    this.resolveFinalResult = resolve;
  });
}
```

The two callbacks are the only things that distinguish one kind of stream from another:
- `isComplete(event)` — returns `true` when this event signals the end of the stream (e.g. when `event.type === "done"`).
- `extractResult(event)` — given that terminal event, pulls out the final assembled value.

When `push()` receives an event for which `isComplete` returns `true`, it resolves `finalResultPromise` *before* delivering the event to the consumer's queue, so `result()` is always ready by the time the loop ends.

Here is the complete, real implementation from S10:

```ts
// From src/utils/event-stream.ts
import type { AssistantMessage, AssistantMessageEvent } from "../types.ts";

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

  push(event: T): void {
    if (this.done) return;

    if (this.isComplete(event)) {
      this.done = true;
      this.resolveFinalResult(this.extractResult(event));
    }

    // Deliver to waiting consumer or queue it
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      this.queue.push(event);
    }
  }

  end(result?: R): void {
    this.done = true;
    if (result !== undefined) {
      this.resolveFinalResult(result);
    }
    // Notify all waiting consumers that we're done
    while (this.waiting.length > 0) {
      const waiter = this.waiting.shift()!;
      waiter({ value: undefined as any, done: true });
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        return;
      } else {
        const result = await new Promise<IteratorResult<T>>(
          (resolve) => this.waiting.push(resolve)
        );
        if (result.done) return;
        yield result.value;
      }
    }
  }

  result(): Promise<R> {
    return this.finalResultPromise;
  }
}
```

Notice that `push()` guards with `if (this.done) return` — once the stream is closed, any subsequent push from a misbehaving producer is silently dropped. This prevents double-resolution of the result promise.

Notice also that `end()` accepts an optional `result` argument. This covers the *abort* case: the provider adapter may terminate the stream early before a natural `"done"` event arrives. Passing `result` to `end()` lets the caller still get a meaningful (partial) assembled message rather than a forever-pending promise.

## Layering AssistantMessageEventStream on top

Now we have a generic `EventStream<T, R>`. For LLM streaming, `T` is `AssistantMessageEvent` (a union of all the event kinds you can receive), and `R` is `AssistantMessage` (the assembled reply). Rather than making every caller pass the two callbacks, we wrap that logic in a concrete subclass:

```ts
// From src/utils/event-stream.ts
export class AssistantMessageEventStream
  extends EventStream<AssistantMessageEvent, AssistantMessage>
{
  constructor() {
    super(
      // isComplete: the stream is done when the event type is "done" or "error"
      (event) => event.type === "done" || event.type === "error",

      // extractResult: pull the assembled message out of the terminal event
      (event) => {
        if (event.type === "done") {
          return event.message;
        } else if (event.type === "error") {
          return event.error;
        }
        throw new Error("Unexpected event type for final result");
      },
    );
  }
}
```

Two things to notice here:

1. **Both `"done"` and `"error"` close the stream.** An error event is still a terminal event — it closes the iteration and resolves the result promise with whatever `event.error` contains. This means `result()` always resolves; it never hangs even when the model returns an error.

2. **The `"done"` event carries `event.message`** — the fully assembled `AssistantMessage` including all content blocks, usage counts, and stop reason. The `"error"` event carries `event.error` instead.

There is also a plain factory function for contexts where you cannot use `new` directly (for example, in extension code that may not import the class):

```ts
// From src/utils/event-stream.ts
export function createAssistantMessageEventStream(): AssistantMessageEventStream {
  return new AssistantMessageEventStream();
}
```

## The event sequence a caller observes

Now let's zoom out to what the caller actually sees when it does `for await (const event of s)`. The test suite in S21 documents three primary scenarios.

### Scenario A — Text generation

When the model produces a text reply, events arrive in this order:

| Position | Event type | What it carries | Notes |
|---|---|---|---|
| 1 | `text_start` | — | Signals the beginning of a new text content block |
| 2..N | `text_delta` | `event.delta: string` | Each delta is a token or token fragment to append |
| N+1 | `text_end` | — | The text block is complete |
| last | *(loop ends)* | — | `result()` resolves with the full `AssistantMessage` |

In code, that looks like:

```ts
import { stream } from "llm-toolkit";

const s = stream(model, context);

let accumulated = "";
for await (const event of s) {
  if (event.type === "text_start") {
    // A new text block is opening; set up any rendering state here
  } else if (event.type === "text_delta") {
    accumulated += event.delta;
    process.stdout.write(event.delta);   // stream to terminal as it arrives
  } else if (event.type === "text_end") {
    // The text block is complete
  }
}

const response = await s.result();
// response.content contains the fully assembled content blocks
console.log("Stop reason:", response.stopReason);
```

The test in S21 (`handleStreaming`) asserts that `textStarted`, `textChunks.length > 0`, and `textCompleted` are all true after the loop — confirming this three-event envelope.

### Scenario B — Tool call generation

When the model decides to call a tool, the events are:

| Position | Event type | What it carries | Notes |
|---|---|---|---|
| 1 | `toolcall_start` | `event.partial` (partial message), `event.contentIndex` (index into content array) | The tool call block is opened; `event.partial.content[contentIndex]` is a `toolCall` block with `name` and `id` already populated |
| 2..N | `toolcall_delta` | `event.delta: string` (raw JSON fragment), `event.contentIndex`, `event.partial` | Each delta appends to the tool's argument JSON; `event.partial.content[contentIndex].arguments` is progressively parsed |
| N+1 | `toolcall_end` | `event.contentIndex`, `event.partial` | The tool call is complete; `arguments` is fully parsed |
| last | *(loop ends)* | — | `result()` resolves with `stopReason: "toolUse"` |

The test (`handleToolCall`) confirms:
- At `toolcall_start`: `partial.content[contentIndex].type === "toolCall"`, `name` is populated, `id` is truthy.
- At `toolcall_delta`: accumulated `event.delta` strings, when concatenated and parsed, form valid JSON; `arguments` is always a defined object (never `undefined`) even during streaming.
- At `toolcall_end`: `arguments.a`, `arguments.b`, and `arguments.operation` are fully resolved.
- After the loop: `s.result()` has `stopReason === "toolUse"` and `content` contains a `toolCall` block.

```ts
const s = await stream(model, context);
let toolArgJson = "";

for await (const event of s) {
  if (event.type === "toolcall_start") {
    const block = event.partial.content[event.contentIndex];
    // block.type === "toolCall", block.name and block.id are available
    console.log("Tool call starting:", block.name);
  } else if (event.type === "toolcall_delta") {
    toolArgJson += event.delta;   // accumulate raw JSON
  } else if (event.type === "toolcall_end") {
    const block = event.partial.content[event.contentIndex];
    // block.arguments is the fully parsed object
    console.log("Tool call ready:", block.arguments);
  }
}

const response = await s.result();
// response.stopReason === "toolUse"
```

### Scenario C — Thinking / extended reasoning

Some models support an internal "thinking" phase before their reply. The event envelope mirrors the text pattern:

| Position | Event type | Notes |
|---|---|---|
| 1 | `thinking_start` | Thinking block begins |
| 2..N | `thinking_delta` | `event.delta` carries fragments of the model's internal reasoning |
| N+1 | `thinking_end` | Thinking block complete |
| — | *(then text events)* | The visible reply follows |

The test (`handleThinking`) asserts that `thinkingStarted`, `thinkingChunks.length > 0`, and `thinkingCompleted` are all true, and that `response.content` contains a block with `type === "thinking"`.

## Abort and error behavior

### Calling `end()` early (abort)

A provider adapter can terminate the stream before the model finishes — for example, when the user presses Ctrl-C or when a timeout fires. The adapter calls `stream.end(partialResult)`. This does three things:

1. Sets `done = true`, so any subsequent `push()` calls are ignored.
2. Resolves `finalResultPromise` with `partialResult` if one was provided.
3. Wakes any consumer suspended in `waiting` with `{ done: true }`, ending the `for await` loop.

The caller's `for await` loop exits cleanly, and `await s.result()` resolves with whatever partial result the adapter passed — it never hangs.

### Error events

When the model API returns an error mid-stream, the provider adapter pushes an `AssistantMessageEvent` with `type === "error"`. Because `isComplete` in `AssistantMessageEventStream` returns `true` for `"error"` events, the stream closes and the result promise resolves with `event.error`. The `"error"` event is still delivered to the `for await` loop as a normal event, so a caller can inspect it inline:

```ts
for await (const event of s) {
  if (event.type === "error") {
    console.error("Model returned an error:", event.error);
    // The loop will end naturally after this event
  }
}

// result() has already resolved — no hang
const response = await s.result();
```

### What happens if `result()` is never awaited

`result()` returns a plain `Promise<R>`. If you never `await` it, the Promise is garbage-collected. The stream itself does not rely on the caller calling `result()` — it resolves internally at close time regardless.

## Putting it all together: a complete streaming loop

Here is a self-contained example that handles all three event families and reads the final result:

```ts
import { stream } from "llm-toolkit";
import type { Context, Model } from "llm-toolkit";

async function streamWithEvents(model: Model<any>, context: Context) {
  const s = stream(model, context);

  let textBuffer = "";

  for await (const event of s) {
    switch (event.type) {
      case "text_start":
        textBuffer = "";
        break;
      case "text_delta":
        textBuffer += event.delta;
        process.stdout.write(event.delta);
        break;
      case "text_end":
        // Text block is done; textBuffer has the full text
        break;
      case "toolcall_start": {
        const block = event.partial.content[event.contentIndex];
        console.log("\nTool call:", block.name);
        break;
      }
      case "toolcall_delta":
        // You can accumulate event.delta if you need raw JSON
        break;
      case "toolcall_end": {
        const block = event.partial.content[event.contentIndex];
        console.log("Arguments:", block.arguments);
        break;
      }
      case "thinking_start":
        console.log("[thinking...]");
        break;
      case "thinking_delta":
        // event.delta is a fragment of the model's reasoning
        break;
      case "thinking_end":
        break;
      case "error":
        console.error("Stream error:", event.error);
        break;
    }
  }

  // By the time we reach here, result() is always resolved
  const response = await s.result();
  console.log("\nStop reason:", response.stopReason);
  console.log("Usage:", response.usage);
  return response;
}
```

Notice that we call `stream(model, context)` — not `await stream(...)`. The `stream()` entry point returns the `AssistantMessageEventStream` synchronously; the network call and event push happen asynchronously as the provider adapter drives it. (The provider wires itself to the stream's `push` and `end` methods internally.)

<!-- GAP: The exact signature/return type of the `stream()` function (and whether it is synchronous or returns a promise) is not shown in S10 (event-stream.ts) — S10 defines only the EventStream and AssistantMessageEventStream classes. The stream() entry point is defined in a separate module not assigned to this page. The description above is inferred from S21 test usage: `const s = stream(model, context)` used without `await` in some test helpers and with `await` in others; source silent on the canonical return type. -->

## Summary: what each method does

| Method / accessor | Who calls it | What it does |
|---|---|---|
| `push(event)` | Provider adapter | Enqueues an event or delivers it to a waiting consumer; detects the terminal event and resolves `finalResultPromise` |
| `end(result?)` | Provider adapter | Closes the stream, optionally resolving `finalResultPromise` with a partial result |
| `[Symbol.asyncIterator]()` | Runtime (via `for await`) | Returns the async iterator; loops over buffered and future events |
| `result()` | Caller | Returns `Promise<R>`; resolves when the stream closes (via a terminal event or `end()`) |

## What comes next

We now have a producer/consumer bridge that any provider adapter can drive by calling `push()` and `end()`. In the next chapter we will look at how the Anthropic and OpenAI provider adapters actually do that — translating raw SSE bytes into the `AssistantMessageEvent` types we've been working with here.

---

← Previous: [Message Types and the Unified Streaming API](./message-types-and-streaming-api.md) · Next: [Provider Adapters: Anthropic and OpenAI](./provider-adapters-anthropic-and-openai.md) →
