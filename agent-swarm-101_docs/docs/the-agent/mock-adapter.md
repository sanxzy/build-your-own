---
title: "The Mock Adapter"
description: Build a keyless mock adapter that implements the SwarmAdapter interface and returns scripted, deterministic responses — no API key required.
category: the-agent
type: tutorial
tags: [mock adapter, mock provider, echo adapter, keyless, scripted response, canned response, mock LLM, testing, development, no API key, configurable usage, deterministic, SwarmAdapter, adapter interface, token usage, cost, registration]
keywords: [fake LLM, stub adapter, in-process adapter, test double, zero-cost adapter, swarm mock, scripted reply, token simulation, adapter registration, mock stream]
sources: [S49, S27]
---

**TL;DR** — Every chapter in this book needs a runnable agent. Getting one shouldn't require a paid API key or network access. In this chapter we build the **mock adapter**: a complete implementation of the `SwarmAdapter` contract that returns scripted, deterministic text (or tool calls) with configurable token counts and zero real spend. By the end you will have a drop-in adapter you can use in tests, in CI, and in every subsequent chapter until you are ready to swap in a real LLM.

# The Mock Adapter

We left the previous chapter with a clean interface: `SwarmAdapter` defines `invoke`, `status`, and `cancel`, and the result type `AdapterResult` carries an exit code, a timeout flag, optional usage and cost fields, an optional summary, and optional session-continuity data. If you haven't read that chapter yet, the two-minute recap is in the [prerequisites section](#prerequisites-recap) below.

The problem now is practical. Every reader of this book is in a different situation — some have Anthropic keys, some have OpenAI keys, some have neither. And even readers who do have keys shouldn't burn spend running chapter examples that exist purely to demonstrate system wiring. We need an adapter that:

- **Requires no API key** and makes no network calls.
- **Returns exactly what we tell it to** — so examples are reproducible.
- **Reports plausible token usage and zero cost** by default (configurable so tests can assert on billing logic).
- **Behaves like a real streaming provider** — emitting delta events in the same shape a live LLM would — so the orchestrator code we write against it is exactly the code we'd use in production.

That adapter is the mock adapter. Let's build it from the ground up.

## Prerequisites recap

Before we start, here is the 30-second version of what we are building on.

**The adapter interface** (`SwarmAdapter`) is the contract every adapter in Swarm must satisfy. It has exactly three methods — defined in full in [The Adapter Interface](./adapter-interface.md):

| Method | Signature | Purpose |
|---|---|---|
| `invoke` | `(agent: AdapterAgent, context: InvocationContext) => Promise<AdapterResult>` | Run the agent for one task |
| `status` | `(run: HeartbeatRun) => Promise<RunStatus>` | Check whether a run is still active |
| `cancel` | `(run: HeartbeatRun) => Promise<void>` | Abort an in-progress run |

**`AdapterResult`** is the unified return value from `invoke`. Required fields are `exitCode` and `timedOut`; everything else is optional and reported only when the adapter has the data:

```ts
// Simplified view of AdapterResult (from S27/S18)
interface AdapterResult {
  exitCode:     number | null;   // process exit code; null for non-process adapters
  timedOut:     boolean;         // true if the run exceeded its time limit
  errorMessage?: string | null;
  usage?:        UsageSummary;   // token counts (inputTokens, outputTokens, cachedInputTokens)
  sessionId?:    string | null;  // session-continuity id for the next run
  provider?:     string | null;  // e.g. "anthropic", "openai"
  model?:        string | null;  // e.g. "mock-1"
  costUsd?:      number | null;  // total cost in US dollars
  summary?:      string | null;  // plain-text description of what the agent did
}
```

A successful run has `exitCode: 0` and `timedOut: false`. A timed-out run sets `timedOut: true`; a non-zero-exit run fills `exitCode` and `errorMessage`. We covered all the fields and their rationale in [The Adapter Interface](./adapter-interface.md).

**Your First Agent** (the [getting-started chapter](../getting-started/your-first-agent.md)) already showed an agent completing a task backed by something that always worked — producing a predictable result with no external dependencies. That something was the mock adapter running under the hood. Here we build and understand it explicitly.

## The registration model

Before we write a single line of mock logic, we need to understand how the orchestrator selects an adapter at all.

Swarm uses a **provider registry**: a map from a provider identifier (a string like `"mock"` or `"claude-cli"`) to an executable stream function. When the orchestrator needs to run an agent, it looks up the agent's configured provider in the registry and calls the registered stream function.

The mock adapter self-registers on creation. That is the key design choice: you call one setup function, it creates an internal adapter instance, registers it under a fresh unique api identifier, and returns a handle you use to configure responses and tear down later.

```ts
// src/adapters/mock/index.ts
import { registerApiProvider } from "@swarm/llm/api-registry";

export interface MockProviderRegistration {
  api: string;                             // the unique id to pass in agent config
  setResponses(responses: MockResponseStep[]): void;   // replace the response queue
  appendResponses(responses: MockResponseStep[]): void; // add to the queue
  getPendingResponseCount(): number;       // inspect the queue
  unregister(): void;                      // remove from the global registry
}
```

The `api` string on the returned handle is what you put in the agent's config when you want that agent to use this mock. Every call to `registerMockProvider()` produces a **fresh, independent** api identifier (generated with a random suffix), so you can register multiple mocks in the same process without collision — important when running parallel tests.

## Building the mock provider step by step

### Step 1 — The canned response helpers

The smallest thing we need is a way to describe what the mock should say. We represent each response as an `AssistantMessage` — the same type a real LLM would return — and we provide a few factory functions to build common shapes without ceremony.

```ts
// src/adapters/mock/content.ts
import type { AssistantMessage, TextContent, ThinkingContent, ToolCall } from "@swarm/llm/types";

export type MockContentBlock = TextContent | ThinkingContent | ToolCall;

/** Wrap a plain string in a TextContent block */
export function mockText(text: string): TextContent {
  return { type: "text", text };
}

/** Wrap a reasoning string in a ThinkingContent block */
export function mockThinking(thinking: string): ThinkingContent {
  return { type: "thinking", thinking };
}

/** Build a ToolCall block with an auto-generated id */
export function mockToolCall(
  name: string,
  arguments_: ToolCall["arguments"],
  options: { id?: string } = {},
): ToolCall {
  return {
    type:      "toolCall",
    id:        options.id ?? randomId("tool"),
    name,
    arguments: arguments_,
  };
}
```

These three helpers cover the three content block types that can appear in an assistant message: plain text, reasoning/thinking content, and a tool invocation. We'll use them throughout the chapter's examples.

There is also a convenience function to build a full `AssistantMessage` from any combination of blocks:

```ts
// Simplified view of mockAssistantMessage()
const DEFAULT_USAGE = {
  input: 0, output: 0, cacheRead: 0, cacheWrite: 0, totalTokens: 0,
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
};

export function mockAssistantMessage(
  content: string | MockContentBlock | MockContentBlock[],
  options: {
    stopReason?: AssistantMessage["stopReason"];
    errorMessage?: string;
  } = {},
): AssistantMessage {
  const blocks = typeof content === "string"
    ? [mockText(content)]
    : Array.isArray(content) ? content : [content];

  return {
    role:         "assistant",
    content:      blocks,
    api:          "mock",
    provider:     "mock",
    model:        "mock-1",
    usage:        DEFAULT_USAGE,
    stopReason:   options.stopReason ?? "stop",
    errorMessage: options.errorMessage,
    timestamp:    Date.now(),
  };
}
```

Notice `DEFAULT_USAGE`: every counter is zero, every cost is zero. By default the mock pretends the call was free. We will make usage configurable later in this chapter — but zero is a sensible default because the point of the mock is to avoid real spend, and most tests don't care about billing.

### Step 2 — Scripted and factory responses

A static pre-baked `AssistantMessage` works for simple tests. But sometimes you need a response that depends on what was in the request — for example, to echo the task text back, or to behave differently on the second call. For those cases we allow a **factory function** as well:

```ts
// src/adapters/mock/types.ts
import type { AssistantMessage, Context, StreamOptions, Model } from "@swarm/llm/types";

export type MockResponseFactory = (
  context: Context,
  options:  StreamOptions | undefined,
  state:    { callCount: number },
  model:    Model<string>,
) => AssistantMessage | Promise<AssistantMessage>;

export type MockResponseStep = AssistantMessage | MockResponseFactory;
```

A `MockResponseStep` is therefore either:
- A pre-built `AssistantMessage` (the common case for scripted replies), or
- A `MockResponseFactory` — a function that receives the full conversation context, any stream options, a state object tracking how many times this provider has been called, and the model being requested. The factory can return a message synchronously or asynchronously.

This is the key to the "echo" pattern: a factory reads `context.messages` and builds its reply from the last user message. We'll show that in the [Try it yourself](#try-it-yourself) section.

### Step 3 — The response queue and the stream function

The mock provider keeps an internal **queue** of pending responses. Each time the orchestrator calls `invoke`, the provider shifts one step off the front of the queue, resolves it (calling the factory if needed), and streams the resulting message back as delta events.

```ts
// Simplified view of registerMockProvider() — src/adapters/mock/register.ts
export function registerMockProvider(options: RegisterMockProviderOptions = {}): MockProviderRegistration {
  const api              = options.api ?? randomId("mock");
  const provider         = options.provider ?? "mock";
  const tokensPerSecond  = options.tokensPerSecond;        // optional simulated streaming speed
  const minTokenSize     = options.tokenSize?.min ?? 3;    // chars per streamed chunk (min)
  const maxTokenSize     = options.tokenSize?.max ?? 5;    // chars per streamed chunk (max)
  let   pendingResponses: MockResponseStep[] = [];
  const state            = { callCount: 0 };

  const stream: StreamFunction = (requestModel, context, streamOptions) => {
    const outer = createAssistantMessageEventStream();
    const step  = pendingResponses.shift();   // consume the next scripted step
    state.callCount++;

    queueMicrotask(async () => {
      try {
        if (!step) {
          // No response queued — surface an error so tests fail clearly
          const msg = createErrorMessage(new Error("No more mock responses queued"), ...);
          outer.push({ type: "error", reason: "error", error: msg });
          outer.end(msg);
          return;
        }

        const resolved = typeof step === "function"
          ? await step(context, streamOptions, state, requestModel)
          : step;

        const message = withUsageEstimate(cloneMessage(resolved, api, provider, requestModel.id),
                                          context, streamOptions, promptCache);
        await streamWithDeltas(outer, message, minTokenSize, maxTokenSize, tokensPerSecond, streamOptions?.signal);
      } catch (error) {
        const msg = createErrorMessage(error, ...);
        outer.push({ type: "error", reason: "error", error: msg });
        outer.end(msg);
      }
    });

    return outer;
  };

  registerApiProvider({ api, stream }, sourceId);

  return {
    api,
    setResponses(responses)     { pendingResponses = [...responses]; },
    appendResponses(responses)  { pendingResponses.push(...responses); },
    getPendingResponseCount()   { return pendingResponses.length; },
    unregister()                { unregisterApiProviders(sourceId); },
  };
}
```

A few details worth understanding:

- **`queueMicrotask`** — the stream function returns synchronously (the caller gets the stream object immediately). The actual work is deferred to a microtask so callers can set up event handlers before any events fire. This mirrors how a real async HTTP provider would behave.
- **`pendingResponses.shift()`** — each call consumes exactly one response from the front of the queue. If the queue is empty, the provider emits an error event rather than hanging. Your tests will fail fast with a clear message rather than timing out.
- **`withUsageEstimate`** — even though the mock defaults to zero cost, this function computes a token estimate based on the text lengths so that multi-call sessions can accumulate plausible numbers. We look at this in detail next.

### Step 4 — Configurable usage and cost

The `DEFAULT_USAGE` we set earlier is all zeros. For tests that exercise billing logic — rate limiting, budget caps, spend reports — you need the mock to report non-zero usage.

There are two ways to configure usage:

**Option A — Attach usage to the scripted message directly.** When you call `mockAssistantMessage()`, the returned message always carries `DEFAULT_USAGE` (all zeros). But you can override this on any individual message:

```ts
const expensiveResponse = mockAssistantMessage("Here is the analysis.");
expensiveResponse.usage = {
  input:       1200,
  output:      350,
  cacheRead:   0,
  cacheWrite:  1200,
  totalTokens: 2750,
  cost: { input: 0.0036, output: 0.00175, cacheRead: 0, cacheWrite: 0.0012, total: 0.00655 },
};
mock.setResponses([expensiveResponse]);
```

**Option B — Let the provider estimate usage automatically.** The `withUsageEstimate` function, called inside the stream function, computes a rough token count from actual text lengths (using a `Math.ceil(chars / 4)` heuristic). It also simulates prompt-cache behaviour: when a `sessionId` is present in stream options and `cacheRetention` is not `"none"`, it tracks the common prefix between successive prompts and splits the input tokens into `input` (new), `cacheRead` (tokens reread from cache), and `cacheWrite` (new tokens written to cache). Cost stays zero in both paths — the mock model's per-token cost is defined as zero.

The upshot: if your test cares about whether usage numbers were logged or budget caps were triggered, use Option A and set explicit values. If you want the mock to produce *plausible-looking* (but free) numbers for display tests, rely on Option B by passing a `sessionId`.

### Step 5 — Streaming delta events

You might wonder: does a mock adapter actually need to simulate streaming? The answer is yes — because the orchestrator and the UI code it feeds are written against a streaming interface. They listen for `text_start`, `text_delta`, `text_end`, `toolcall_start`, and `toolcall_delta` events. If the mock returned the final message in one synchronous shot, the orchestrator's streaming path would never execute, and you'd be testing a code path that never runs in production.

The mock's `streamWithDeltas` function walks the message's `content` array block by block. For each block it:

1. Emits a `*_start` event with an empty partial.
2. Splits the text (or serialised tool-call arguments) into chunks of 3–5 characters (configurable via `tokenSize`).
3. Emits one `*_delta` event per chunk, appending each chunk to the partial.
4. Emits a `*_end` event with the completed block.

The chunk sizes are random within the configured range, which means the streaming output is non-deterministic at the individual-chunk level while being deterministic at the message level. That is intentional: it exercises the orchestrator's delta-accumulation logic without making test assertions fragile against specific chunk boundaries.

If you want to test abort behaviour, pass an `AbortSignal` in the stream options. The mock checks the signal between every chunk and emits an `error` / `aborted` event if the signal fires, then closes the stream — exactly what a real HTTP provider would do on a cancelled request.

```ts
// Simulated streaming with configurable token size (conceptual)
const tokenSize = { min: 3, max: 5 };   // default — each chunk is 3–5 chars
// Pass tokensPerSecond to simulate real network latency
const slowMock = registerMockProvider({ tokensPerSecond: 50 });
```

When `tokensPerSecond` is not set (the default), each chunk yields via `queueMicrotask` so the event loop stays responsive without any real delay. Set it to a positive number to introduce actual `setTimeout`-based delays proportional to chunk size — useful for UI tests that measure time-to-first-token or show a streaming animation.

### Step 6 — Implementing status and cancel

The `SwarmAdapter` contract requires `status` and `cancel` in addition to `invoke`. For a mock adapter these are trivial:

```ts
// src/adapters/mock/register.ts (continued)
status(_run: HeartbeatRun): Promise<RunStatus> {
  // In-process mock: a "run" completes the instant invoke() resolves.
  // There is nothing in-flight to query.
  return Promise.resolve("succeeded");
}

cancel(_run: HeartbeatRun): Promise<void> {
  // Nothing to cancel; if you want abort-in-flight, pass an AbortSignal to invoke().
  return Promise.resolve();
}
```

This is the right trade-off: the mock is in-process. By the time `status` could be called, the stream has already completed. A real process or HTTP adapter would track live run IDs; the mock doesn't need to because it has no out-of-process state. If you need to test mid-run cancellation, the abort-signal path (described in step 5) is the right mechanism.

### Step 7 — The result shape

The mock's `invoke` method resolves to an `AdapterResult` — the same shape defined in [The Adapter Interface](./adapter-interface.md) and used by every other adapter. The mock is in-process, so fields that only make sense for out-of-process backends (`provider`, `model`, `sessionId`) are populated from the scripted message, while cost defaults to zero.

```ts
// Successful mock completion
const result: AdapterResult = {
  exitCode: 0,
  timedOut: false,
  usage:    { inputTokens: 0, outputTokens: 0 },  // zero by default; override via mockAssistantMessage
  costUsd:  0,
  summary:  "<serialised agent output>",
};

// Mock error (e.g. no response queued, or a factory threw)
const errorResult: AdapterResult = {
  exitCode:     1,
  timedOut:     false,
  errorMessage: "No more mock responses queued",
  costUsd:      0,
};
```

When the stream closes with `stopReason: "stop"`, the adapter returns `exitCode: 0` and populates `summary` with the completed message text. When the stream closes with `stopReason: "error"` or `stopReason: "aborted"`, it fills `errorMessage` and returns `exitCode: 1`. The orchestrator's result-handling code sees the same `AdapterResult` shape regardless of which adapter ran.

## Putting it together — a complete working example

Here is a full setup you can drop into a test or a chapter scaffold:

```ts
import { registerMockProvider, mockAssistantMessage, mockText, mockToolCall } from "@swarm/adapters/mock";

// 1. Register a mock provider — no API key needed
const mock = registerMockProvider();

// 2. Script the responses the orchestrator will receive
mock.setResponses([
  mockAssistantMessage("I've read the task. Here is my plan."),
  mockAssistantMessage([
    mockThinking("The user wants a summary. I should call the summarise tool."),
    mockToolCall("summarise", { text: "This quarter's highlights..." }),
  ]),
  mockAssistantMessage("Done. I've summarised the document.", { stopReason: "stop" }),
]);

// 3. Wire the agent to use this mock (the api string is the key)
const agent = createAgent({
  name:     "analyst",
  provider: mock.api,     // <- this is how the orchestrator finds the mock
  model:    "mock-1",
});

// 4. Run a task — the mock will consume one response per LLM call
const result = await agent.runTask({ description: "Summarise Q3 results" });
console.log(result.summary);   // plain-text output from the third response

// 5. Clean up (important in tests to avoid registry leaks)
mock.unregister();
```

Notice the shape: you register the mock, script some responses, create an agent pointing at `mock.api`, run a task, and then unregister. The responses are consumed in order — one per LLM call the agent makes. If the agent makes more calls than there are scripted responses, it receives an error message rather than hanging.

## The mock as the default for this book

For every subsequent chapter until we reach [The LLM Adapter](./llm-adapter.md), the mock adapter is the default execution backend. When a chapter says "run the agent," it means: register a mock with one or two scripted responses, create the agent pointing at `mock.api`, and run the task. This pattern will be shown once per chapter that introduces it.

The reasons are the same ones we stated at the top:

- No API key required — every reader can run every example.
- Deterministic output — examples are reproducible and CI-safe.
- Zero spend — exploring the orchestrator's internals costs nothing.
- Same streaming shape — the orchestrator code you write against the mock is exactly the code you'd use in production.

When you are ready to swap in a real provider, you change exactly one line: replace `mock.api` with the api identifier of a real adapter. The orchestrator, the agents, and all the wiring stay the same.

## Mock adapter options reference

| Option | Type | Default | Effect |
|---|---|---|---|
| `api` | `string` | random `mock:<timestamp>:<rand>` | Override the provider's api identifier |
| `provider` | `string` | `"mock"` | Label in the model registry |
| `models` | `MockModelDefinition[]` | one model: `{ id: "mock-1", name: "Mock Model", cost: all zero }` | Define the mock model(s) |
| `tokensPerSecond` | `number` | none | Introduce real streaming delay (ms per chunk ∝ token count) |
| `tokenSize.min` | `number` | `3` | Minimum characters per streamed chunk |
| `tokenSize.max` | `number` | `5` | Maximum characters per streamed chunk |

The default model has `contextWindow: 128000`, `maxTokens: 16384`, `reasoning: false`, `input: ["text", "image"]`, and all-zero costs. Override individual fields via the `models` array when your test needs specific cost entries or context window limits.

## Try it yourself

The following three exercises extend the mock beyond the defaults. Try each one before moving to the next chapter.

**Exercise 1 — Echo adapter.** Make the mock reply with the exact text of the task description. You'll need a `MockResponseFactory` that reads `context.messages` and finds the last user message:

```ts
// Hint: a factory receives the full context
mock.setResponses([
  (context) => {
    const lastUser = [...context.messages].reverse().find(m => m.role === "user");
    const text = typeof lastUser?.content === "string"
      ? lastUser.content
      : lastUser?.content.find(b => b.type === "text")?.text ?? "(empty)";
    return mockAssistantMessage(`Echo: ${text}`);
  },
]);
```

**Exercise 2 — Configurable fake latency.** Register the mock with `tokensPerSecond: 20` and measure how long a response with 100 characters of text takes to complete. Then increase to `tokensPerSecond: 200` and compare. Notice that the latency scales with output length, not input.

```ts
const slowMock = registerMockProvider({ tokensPerSecond: 20 });
slowMock.setResponses([mockAssistantMessage("A".repeat(100))]);
const start = Date.now();
await agent.runTask({ description: "test" });
console.log(`Elapsed: ${Date.now() - start}ms`);
slowMock.unregister();
```

**Exercise 3 — Fake tool call.** Return a `mockToolCall` block and observe how the orchestrator handles a tool-use response. The tool should be one registered on the agent. Check whether the orchestrator re-invokes the mock for the next turn after it receives the tool result:

```ts
mock.setResponses([
  mockAssistantMessage([mockToolCall("search", { query: "Q3 revenue" })]),
  mockAssistantMessage("Based on the search results, revenue was up 12%."),
]);
```

If the queue reaches zero before the agent's loop finishes, you'll see the "No more mock responses queued" error — a useful signal that the agent made more LLM calls than you expected.

---

← Previous: [The Adapter Interface](./adapter-interface.md) · Next: [The LLM Adapter](./llm-adapter.md) →
