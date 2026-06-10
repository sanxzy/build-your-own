---
title: "Provider Adapters: Google Gemini and the Faux Test Provider"
description: "Add the Google Gemini adapter (non-SSE response format, schema conversion, thinking) and the faux provider for deterministic, offline agent testing."
category: llm-toolkit
type: tutorial
tags: [Google, Gemini, google-generative-ai, faux provider, test provider, non-SSE, schema conversion, thinking, thinking budget, deterministic testing, cache simulation, scripted response, llm-toolkit, provider, registerFauxProvider, streamGoogle, streamSimpleGoogle, fauxText, fauxThinking, fauxToolCall, FauxProviderRegistration, GoogleOptions, ThinkingConfig, tool call ID, provider adapter, cross-provider]
keywords: [google gemini adapter, offline testing llm, mock provider, stub provider, test double, scripted llm, thinking tokens, budget tokens, thinkingLevel, generateContentStream, functionCall, thought signature, token simulation, prompt cache simulation]
sources: [S13, S14, S15, S23]
---

**TL;DR** — We have already built adapters for two providers. This chapter adds a third real one — Google Gemini — whose wire format differs from both, then introduces the faux (test) provider: a scripted stand-in you can drive from tests without an API key or network. After reading this page you will understand Gemini's non-SSE response shape, how the adapter handles thinking blocks and tool calls from its streaming chunks, and how to script the faux provider for deterministic, repeatable agent tests.

# Provider Adapters: Google Gemini and the Faux Test Provider

## Where we are

So far in the `llm-toolkit` library we have:

- Generic message types and the `Provider` concept — the shared vocabulary every adapter speaks (see [Message Types and the Streaming API](./message-types-and-streaming-api.md)).
- The `EventStream` / `AssistantMessageEventStream` — the push-based channel that carries `start`, `text_delta`, `toolcall_end`, `done`, and friends from an adapter to the caller (see [Event Stream and the Observable Backbone](./event-stream-and-observable-backbone.md)).
- Two concrete adapters — Anthropic (SSE-based) and OpenAI (completions-based) — that translate each provider's wire protocol into our canonical event sequence (see [Provider Adapters: Anthropic and OpenAI](./provider-adapters-anthropic-and-openai.md)).

Now we face two new problems:

1. Google Gemini speaks a different protocol than either prior adapter — it is not SSE in the raw sense, its streaming chunks carry `parts` arrays instead of delta strings, it handles tool calls inline rather than via separate event types, and its thinking tokens arrive as a distinct part type.
2. Testing agent code that calls real providers is slow, flaky, and requires API keys. We need a test-double provider we can drive from a unit test: scripted responses, zero network, deterministic output.

We will tackle both in order.

---

## Part 1 — The Google Gemini Adapter

### The response format problem

Recall the mental model from the [previous chapter](./provider-adapters-anthropic-and-openai.md): a provider adapter is a function with the signature `StreamFunction<TApi, TOptions>`. It takes a model, a context, and options; it returns an `AssistantMessageEventStream` and begins pushing events into it asynchronously.

The Anthropic adapter reads newline-delimited SSE lines and maps each to an event. The OpenAI completions adapter does likewise over a similar SSE format. Gemini's SDK (`@google/genai`) gives us an async iterable of `GenerateContentResponse` objects. Each object carries an array of *candidates*, each candidate an array of *parts*, and each part is one of: a text part (which may be marked as a thinking part), a function-call part, or other media. There is no outer `type` field — the part structure encodes the intent.

The adapter function is `streamGoogle`:

```ts
// From src/providers/google.ts (simplified view — real signature below)
export const streamGoogle: StreamFunction<"google-generative-ai", GoogleOptions> = (
  model,
  context,
  options,
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream();

  (async () => {
    // ... build params, call client.models.generateContentStream(params),
    // then iterate chunks and push events ...
    stream.end();
  })();

  return stream;
};
```

The function returns the stream immediately; the async IIFE (the self-calling `async () => { ... }()`) drives the actual API call and event pushes in the background. This is the same non-blocking pattern we saw in the Anthropic adapter.

### Building the request

Before we can stream, we need to convert our internal `Context` and `GoogleOptions` into a `GenerateContentParameters` object the SDK understands. The adapter does this in `buildParams`:

```ts
// Simplified view of buildParams (google.ts)
function buildParams(
  model: Model<"google-generative-ai">,
  context: Context,
  options: GoogleOptions = {},
): GenerateContentParameters {
  // 1. Convert our messages to Google's `contents` format
  const contents = convertMessages(model, context);

  // 2. Map scalar options
  const generationConfig: GenerateContentConfig = {};
  if (options.temperature !== undefined) generationConfig.temperature = options.temperature;
  if (options.maxTokens !== undefined)    generationConfig.maxOutputTokens = options.maxTokens;

  const config: GenerateContentConfig = {
    ...generationConfig,
    ...(context.systemPrompt && { systemInstruction: sanitizeSurrogates(context.systemPrompt) }),
    ...(context.tools?.length  && { tools: convertTools(context.tools) }),
  };

  // 3. Map tool-choice mode
  if (context.tools?.length && options.toolChoice) {
    config.toolConfig = {
      functionCallingConfig: { mode: mapToolChoice(options.toolChoice) },
    };
  }

  // 4. Map thinking config (covered below)
  // ...

  return { model: model.id, contents, config };
}
```

Two points worth noticing here:

- **`convertMessages`** — our internal `Message[]` (with `role: "user" | "assistant" | "toolResult"` and typed content blocks) is not the same shape as Google's `contents` array. A helper from `google-shared.ts` handles that mapping. We keep the helper call opaque here because the schema details live in that file, which is outside this chapter's scope.
- **`convertTools`** — our internal `Tool` type uses JSON Schema via the `typebox` library. Google's function declarations use a slightly different schema dialect. Again, `google-shared.ts` performs that conversion.
- **`sanitizeSurrogates`** — Gemini's API rejects strings containing lone UTF-16 surrogate code points. The adapter sanitises the system prompt before sending it.

The `systemInstruction` field is where the system prompt goes in Gemini's API — not in the message history.

### Tool choice options

The `GoogleOptions` type extends our generic `StreamOptions` with two Gemini-specific fields:

```ts
export interface GoogleOptions extends StreamOptions {
  toolChoice?: "auto" | "none" | "any";
  thinking?: {
    enabled: boolean;
    budgetTokens?: number;  // -1 for dynamic, 0 to disable
    level?: GoogleThinkingLevel;
  };
}
```

`toolChoice` maps to Google's `functionCallingConfig.mode`:

| Our value | Google mode |
|-----------|-------------|
| `"auto"`  | `AUTO`      |
| `"none"`  | `NONE`      |
| `"any"`   | `ANY`       |

The mapping is done by `mapToolChoice` from `google-shared.ts`.

### Iterating chunks: text and thinking parts

Once we have params, `streamGoogle` calls:

```ts
const googleStream = await client.models.generateContentStream(params);
```

Then it iterates:

```ts
for await (const chunk of googleStream) {
  output.responseId ||= chunk.responseId;   // keep the first non-empty response ID
  const candidate = chunk.candidates?.[0];
  if (candidate?.content?.parts) {
    for (const part of candidate.content.parts) {
      // handle text/thinking parts
      // handle functionCall parts
    }
  }
  // handle finishReason and usageMetadata
}
```

For text-like parts (`part.text !== undefined`), the adapter must distinguish between a regular text part and a thinking part. It calls `isThinkingPart(part)` (from `google-shared.ts`) to make that decision.

The adapter tracks the "currently open" content block in a local variable `currentBlock`. When a new part arrives that changes type (e.g., thinking → text, or text → thinking), it closes the previous block with a `text_end` or `thinking_end` event, then opens a new one:

```ts
let currentBlock: TextContent | ThinkingContent | null = null;

// For each part with part.text !== undefined:
const isThinking = isThinkingPart(part);

if (!currentBlock ||
    (isThinking && currentBlock.type !== "thinking") ||
    (!isThinking && currentBlock.type !== "text")) {

  // Close the previous block if there is one
  if (currentBlock) {
    stream.push(currentBlock.type === "text"
      ? { type: "text_end",     contentIndex: ..., content: currentBlock.text,    partial: output }
      : { type: "thinking_end", contentIndex: ..., content: currentBlock.thinking, partial: output });
  }

  // Open the new block
  if (isThinking) {
    currentBlock = { type: "thinking", thinking: "", thinkingSignature: undefined };
    output.content.push(currentBlock);
    stream.push({ type: "thinking_start", contentIndex: ..., partial: output });
  } else {
    currentBlock = { type: "text", text: "" };
    output.content.push(currentBlock);
    stream.push({ type: "text_start", contentIndex: ..., partial: output });
  }
}

// Accumulate the delta and push it
if (currentBlock.type === "thinking") {
  currentBlock.thinking += part.text;
  currentBlock.thinkingSignature = retainThoughtSignature(
    currentBlock.thinkingSignature, part.thoughtSignature,
  );
  stream.push({ type: "thinking_delta", contentIndex: ..., delta: part.text, partial: output });
} else {
  currentBlock.text += part.text;
  currentBlock.textSignature = retainThoughtSignature(
    currentBlock.textSignature, part.thoughtSignature,
  );
  stream.push({ type: "text_delta", contentIndex: ..., delta: part.text, partial: output });
}
```

Notice **`thinkingSignature`** / **`textSignature`** — these are retained from `part.thoughtSignature` using `retainThoughtSignature`. The thought signature is a cryptographic marker the Gemini API attaches to thinking content; preserving it is required for the model to reuse its own cached thinking in extended-thinking scenarios.

### Handling function-call parts

When a part has `part.functionCall`, the adapter closes any open text/thinking block, then emits the three-event toolcall sequence (`toolcall_start`, `toolcall_delta`, `toolcall_end`) in a single synchronous burst:

```ts
if (part.functionCall) {
  // Close current block if open ...

  // Gemini may not always supply a unique ID; generate one if needed
  const providedId = part.functionCall.id;
  const needsNewId =
    !providedId ||
    output.content.some((b) => b.type === "toolCall" && b.id === providedId);
  const toolCallId = needsNewId
    ? `${part.functionCall.name}_${Date.now()}_${++toolCallCounter}`
    : providedId;

  const toolCall: ToolCall = {
    type: "toolCall",
    id: toolCallId,
    name: part.functionCall.name || "",
    arguments: (part.functionCall.args as Record<string, any>) ?? {},
    ...(part.thoughtSignature && { thoughtSignature: part.thoughtSignature }),
  };

  output.content.push(toolCall);
  stream.push({ type: "toolcall_start", contentIndex: ..., partial: output });
  stream.push({ type: "toolcall_delta", contentIndex: ...,
                delta: JSON.stringify(toolCall.arguments), partial: output });
  stream.push({ type: "toolcall_end", contentIndex: ..., toolCall, partial: output });
}
```

The ID-generation guard is worth calling out: a module-level `toolCallCounter` is incremented each time a new ID is needed. This guarantees uniqueness within a process even if Gemini returns duplicate or absent IDs.

### Stop reason and usage

After the loop, the adapter reads `candidate.finishReason` and maps it to our canonical `stopReason` (via `mapStopReason` from `google-shared.ts`). If the output contains any tool call, the stop reason is overridden to `"toolUse"` regardless of what Gemini reported.

Usage comes from `chunk.usageMetadata`:

```ts
output.usage = {
  input:      (chunk.usageMetadata.promptTokenCount   || 0)
            - (chunk.usageMetadata.cachedContentTokenCount || 0),
  output:     (chunk.usageMetadata.candidatesTokenCount || 0)
            + (chunk.usageMetadata.thoughtsTokenCount   || 0),
  cacheRead:   chunk.usageMetadata.cachedContentTokenCount || 0,
  cacheWrite:  0,
  totalTokens: chunk.usageMetadata.totalTokenCount || 0,
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
};
calculateCost(model, output.usage);
```

Notice that **thinking tokens** (`thoughtsTokenCount`) are added to `output` tokens — they contribute to the output cost even though they are not in the final text.

### Thinking support: budgets and levels

The `streamGoogle` function is the lower-level entry point where you pass `GoogleOptions` directly. The higher-level `streamSimpleGoogle` bridges from the generic `SimpleStreamOptions` (which uses a provider-agnostic `reasoning` field) down to the Gemini-specific configuration.

```ts
export const streamSimpleGoogle: StreamFunction<"google-generative-ai", SimpleStreamOptions> = (
  model, context, options,
) => {
  if (!options?.reasoning) {
    // No reasoning requested — explicitly disable thinking
    return streamGoogle(model, context, { ...base, thinking: { enabled: false } });
  }

  const clampedReasoning = clampThinkingLevel(model, options.reasoning);
  // ...

  if (isGemini3ProModel(model) || isGemini3FlashModel(model) || isGemma4Model(model)) {
    // Gemini 3-family uses a thinking *level* (LOW / MEDIUM / HIGH / MINIMAL)
    return streamGoogle(model, context, {
      ...base,
      thinking: { enabled: true, level: getThinkingLevel(effort, model) },
    });
  }

  // Gemini 2.x uses a *budget* in tokens
  return streamGoogle(model, context, {
    ...base,
    thinking: { enabled: true, budgetTokens: getGoogleBudget(model, effort, options.thinkingBudgets) },
  });
};
```

Two thinking mechanisms exist because Google changed the API between model generations:

| Model family | Thinking mechanism | How disabled |
|--------------|--------------------|--------------|
| Gemini 2.x   | `thinkingBudget` (token count, `0` = off, `-1` = dynamic) | Set `thinkingBudget: 0` |
| Gemini 3 Flash / Flash-Lite | `thinkingLevel` enum | Set `thinkingLevel: "MINIMAL"` |
| Gemini 3 Pro | `thinkingLevel` enum | Set `thinkingLevel: "LOW"` (cannot fully disable) |
| Gemma 4 | `thinkingLevel` enum | Set `thinkingLevel: "MINIMAL"` |

When thinking is disabled for a reasoning-capable model, `getDisabledThinkingConfig` returns the appropriate config — notably, Gemini 3 Pro **cannot fully disable thinking**; the adapter uses the lowest available level instead.

Default token budgets for Gemini 2.x models by effort level:

| Effort   | gemini-2.5-pro | gemini-2.5-flash | gemini-2.5-flash-lite |
|----------|----------------|------------------|-----------------------|
| minimal  | 128            | 128              | 512                   |
| low      | 2 048          | 2 048            | 2 048                 |
| medium   | 8 192          | 8 192            | 8 192                 |
| high     | 32 768         | 24 576           | 24 576                |

If none of the known model ID patterns match, `getGoogleBudget` returns `-1` (dynamic budget).

### How the Gemini adapter gets registered

All built-in providers are registered lazily by `registerBuiltInApiProviders` in `register-builtins.ts`. The Google adapter is wrapped in a lazy loader so the SDK module is not imported until first use:

```ts
// register-builtins.ts (simplified)
registerApiProvider({
  api: "google-generative-ai",
  stream: streamGoogle,        // lazy-wrapped version
  streamSimple: streamSimpleGoogle,
});
```

You will learn the full registry API in the next chapter. For now the key point is: after `registerBuiltInApiProviders()` runs (which it does automatically when `register-builtins.ts` is imported), any call to `stream(model, context)` where `model.api === "google-generative-ai"` will route to `streamGoogle`.

---

## Part 2 — The Faux (Test) Provider

### The testing problem

Every adapter we have built so far needs a live API key and a network call to exercise. That is fine for production — but for testing the agent loop, tool dispatch logic, context compaction, or any other higher-level code, live API calls are the wrong tool:

- They are slow (hundreds of milliseconds to seconds each).
- They are non-deterministic (the model's output changes on every run).
- They require secrets (API keys in CI, developer machines, etc.).
- A test that exercises an error path (e.g., "what happens when the model returns a tool call with a broken argument?") cannot easily force the real provider into that state.

What we need is a **test double** — a provider that behaves exactly like a real provider from the outside (it registers, it accepts calls, it emits the same event sequence) but whose responses are **fully scripted by the test**.

### `registerFauxProvider`

The entry point is `registerFauxProvider`, exported from `src/providers/faux.ts`:

```ts
import { registerFauxProvider, fauxText, fauxToolCall, fauxAssistantMessage }
  from "llm-toolkit";

const faux = registerFauxProvider();
```

`registerFauxProvider` accepts an optional `RegisterFauxProviderOptions` and returns a `FauxProviderRegistration` handle:

```ts
export interface RegisterFauxProviderOptions {
  api?: string;          // defaults to a random unique id like "faux:1234:abc"
  provider?: string;     // defaults to "faux"
  models?: FauxModelDefinition[];
  tokensPerSecond?: number;
  tokenSize?: {
    min?: number;        // default 3
    max?: number;        // default 5
  };
}

export interface FauxProviderRegistration {
  api: string;
  models: [Model<string>, ...Model<string>[]];
  getModel(): Model<string>;
  getModel(modelId: string): Model<string> | undefined;
  state: { callCount: number };
  setResponses:          (responses: FauxResponseStep[]) => void;
  appendResponses:       (responses: FauxResponseStep[]) => void;
  getPendingResponseCount: () => number;
  unregister:            () => void;
}
```

When no `api` is specified, `registerFauxProvider` generates a random unique API identifier (`faux:${Date.now()}:${randomHex}`). This means **multiple faux providers can coexist** in the same registry — useful when you want to test interaction between two different provider identities.

### The default model

If you do not pass `models`, the provider registers one model:

| Field           | Default value                           |
|-----------------|-----------------------------------------|
| `id`            | `"faux-1"`                              |
| `name`          | `"Faux Model"`                          |
| `reasoning`     | `false`                                 |
| `input`         | `["text", "image"]`                     |
| `contextWindow` | `128000`                                |
| `maxTokens`     | `16384`                                 |
| `baseUrl`       | `"http://localhost:0"` (never contacted)|
| `cost`          | all zeros                               |

You can override any of these by passing a `FauxModelDefinition` array.

### Scripting responses

The faux provider maintains an internal queue of `FauxResponseStep` values. A step is either a pre-built `AssistantMessage` or a factory function that receives the live context and produces one:

```ts
export type FauxResponseFactory = (
  context: Context,
  options: StreamOptions | undefined,
  state: { callCount: number },
  model: Model<string>,
) => AssistantMessage | Promise<AssistantMessage>;

export type FauxResponseStep = AssistantMessage | FauxResponseFactory;
```

The queue is consumed in order — each call to the adapter shifts one step off the front. If the queue is empty when a call arrives, the adapter emits an error event (`"No more faux responses queued"`).

You populate the queue with `setResponses` (replaces the queue entirely) or `appendResponses` (extends it):

```ts
faux.setResponses([
  fauxAssistantMessage("Hello!"),
  fauxAssistantMessage(fauxToolCall("read_file", { path: "/src/index.ts" })),
]);
```

### Helper constructors

Three factory functions build common response building blocks:

```ts
// A plain text content block
export function fauxText(text: string): TextContent {
  return { type: "text", text };
}

// A thinking content block
export function fauxThinking(thinking: string): ThinkingContent {
  return { type: "thinking", thinking };
}

// A tool call content block (id auto-generated if not supplied)
export function fauxToolCall(
  name: string,
  arguments_: ToolCall["arguments"],
  options: { id?: string } = {},
): ToolCall {
  return {
    type: "toolCall",
    id: options.id ?? randomId("tool"),
    name,
    arguments: arguments_,
  };
}
```

These are the building blocks for `fauxAssistantMessage`:

```ts
export function fauxAssistantMessage(
  content: string | FauxContentBlock | FauxContentBlock[],
  options: {
    stopReason?: AssistantMessage["stopReason"];
    errorMessage?: string;
    responseId?: string;
    timestamp?: number;
  } = {},
): AssistantMessage
```

Passing a plain `string` is equivalent to passing `fauxText(string)` — the helper normalises both forms. You can also pass an array to mix content blocks:

```ts
fauxAssistantMessage([
  fauxThinking("Let me think about this..."),
  fauxText("The answer is 42."),
])
```

### A complete test example

Here is a self-contained example that scripts two responses — a tool call then a final answer — and asserts that the right events arrived:

```ts
// Example test (vitest / any test runner)
import {
  registerFauxProvider,
  fauxAssistantMessage,
  fauxText,
  fauxToolCall,
} from "llm-toolkit";
import { stream } from "llm-toolkit";  // the top-level stream() entry point

it("agent can handle a tool-call round trip", async () => {
  const faux = registerFauxProvider();

  // Script two responses in order
  faux.setResponses([
    // Turn 1: model calls a tool
    fauxAssistantMessage(
      fauxToolCall("list_files", { directory: "/src" }),
      { stopReason: "toolUse" },
    ),
    // Turn 2: model gives a final text answer
    fauxAssistantMessage(
      fauxText("I can see 3 files in /src."),
      { stopReason: "stop" },
    ),
  ]);

  const model = faux.getModel();

  // First call — should get the tool call
  const events1: string[] = [];
  const eventStream1 = stream(model, {
    systemPrompt: "You are a helpful assistant.",
    messages: [{ role: "user", content: "List my source files.", timestamp: Date.now() }],
  }, { apiKey: "not-used" });

  for await (const event of eventStream1) {
    events1.push(event.type);
  }

  expect(events1).toContain("toolcall_start");
  expect(events1).toContain("toolcall_end");
  expect(events1).toContain("done");
  expect(faux.state.callCount).toBe(1);

  // Clean up — remove from registry so it doesn't affect other tests
  faux.unregister();
});
```

You can see the pattern: `setResponses` → `getModel()` → make calls → assert on events → `unregister()`. The test never touches a network socket.

### Factory responses — inspecting the context

Sometimes you need the scripted response to depend on what the caller actually sent. Use a `FauxResponseFactory`:

```ts
faux.setResponses([
  async (context, options, state) => {
    // Inspect the real context the agent sent
    const lastMessage = context.messages.at(-1);
    const text = lastMessage?.role === "user"
      ? `You asked: "${lastMessage.content}"`
      : "Got it.";
    return fauxAssistantMessage(fauxText(text));
  },
]);
```

The factory receives `context` (the full `Context` object with system prompt, messages, and tools), `options` (the `StreamOptions` the caller passed), `state` (a shared `{ callCount: number }` counter across all calls to this provider), and the resolved `Model<string>`. The return value is an `AssistantMessage` or a `Promise<AssistantMessage>`.

### Cache simulation

One aspect of the faux provider that deserves attention is its **cache simulation**. When a caller passes a `sessionId` in options and `cacheRetention !== "none"`, the faux provider tracks the serialised prompt for that session in an internal `Map<string, string>`. On subsequent calls with the same session ID, it computes how many characters the new prompt shares with the previous one (the common prefix), converts that to token estimates, and populates the `cacheRead` / `cacheWrite` fields of `usage` accordingly:

```ts
// Simplified view of withUsageEstimate (faux.ts)
function withUsageEstimate(message, context, options, promptCache): AssistantMessage {
  const promptText = serializeContext(context);
  const promptTokens = estimateTokens(promptText);   // ceil(chars / 4)
  let input = promptTokens;
  let cacheRead = 0;
  let cacheWrite = 0;

  if (options?.sessionId && options?.cacheRetention !== "none") {
    const previous = promptCache.get(options.sessionId);
    if (previous) {
      const cachedChars = commonPrefixLength(previous, promptText);
      cacheRead = estimateTokens(previous.slice(0, cachedChars));
      cacheWrite = estimateTokens(promptText.slice(cachedChars));
      input = Math.max(0, promptTokens - cacheRead);
    } else {
      cacheWrite = promptTokens;
    }
    promptCache.set(options.sessionId, promptText);
  }

  return { ...message, usage: { input, output: ..., cacheRead, cacheWrite, ... } };
}
```

This means tests that check token-cost accounting or compaction logic (which care about `cacheRead` / `cacheWrite`) will see realistic-looking usage numbers without needing a real provider.

Token estimation uses `ceil(charCount / 4)` — a rough approximation intentionally kept simple because the faux provider is for structural testing, not precision billing.

### Delta streaming in the faux provider

Even though responses are pre-scripted, the faux provider still emits the full event sequence that a real provider would: `start`, then per-block `thinking_start` / `thinking_delta` / `thinking_end` or `text_start` / `text_delta` / `text_end`, then `done` (or `error` if `stopReason` is `"error"`). The text is split into chunk-sized pieces of `minTokenSize`–`maxTokenSize` characters (defaults: 3–5 chars each), with optional pacing via `tokensPerSecond`.

This means code that consumes the `EventStream` generically — such as a UI renderer accumulating streaming text — will behave identically whether the provider is faux or real. The faux provider is a structural substitute, not a short-circuit.

Abort signals are also honoured: if `options.signal.aborted` is true at any point during streaming, the provider emits an error event with `reason: "aborted"` and ends the stream.

### `unregister` and test isolation

Calling `faux.unregister()` removes the faux provider from the global registry. Always call it in your test teardown:

```ts
afterEach(() => {
  faux.unregister();
});
```

Without this, a faux provider registered in one test can leak into the next, causing confusing "No more faux responses queued" errors.

### Why this matters for everything we build next

The faux provider is the foundation for testing every layer above `llm-toolkit`. The agent loop, context compaction, tool dispatch, and session management chapters all rely on being able to say "given that the model returns *this*, the agent should do *that*" — and the faux provider is what makes that possible offline and deterministically.

In the agent loop and coding agent test suites, you will see exactly this pattern: register a faux provider, script a sequence of responses (tool call, tool result, final answer), run the agent, and assert on the resulting conversation history or file system changes. No API key required.

---

## Cross-provider consistency

The cross-provider handoff test (`test/cross-provider-handoff.test.ts`) illustrates a related guarantee: contexts produced by one real provider can be consumed by another. It generates a small conversation with one provider — user message, tool call, tool result, final answer — then feeds that conversation (with authentic tool-call IDs and thinking blocks) to every other provider and checks that none of them reject it.

This works because all adapters — Anthropic, OpenAI, and Gemini — normalise their wire-format differences into the same internal `Message[]` shape. The faux provider obeys the same contract: a faux-generated `AssistantMessage` with a `ToolCall` block has the same structure as one produced by a real provider, so it can be handed off to a different adapter in the next turn.

---

## Summary of what we built

| Component | Key function | What it does |
|-----------|-------------|--------------|
| Google adapter | `streamGoogle` | Streams Gemini API chunks; handles text, thinking, and function-call parts |
| Param builder | `buildParams` | Converts `Context` + `GoogleOptions` → `GenerateContentParameters` |
| Simple wrapper | `streamSimpleGoogle` | Maps generic `reasoning` field to Gemini thinking-budget or thinking-level config |
| Thinking config | `getDisabledThinkingConfig`, `getThinkingLevel`, `getGoogleBudget` | Model-family-aware thinking configuration |
| Faux provider | `registerFauxProvider` | Registers a scripted test-double provider in the global registry |
| Response helpers | `fauxText`, `fauxThinking`, `fauxToolCall`, `fauxAssistantMessage` | Build scripted response content blocks |
| Cache simulation | `withUsageEstimate` | Simulates prompt-cache token accounting for session-based calls |

---

← Previous: [Provider Adapters: Anthropic and OpenAI](./provider-adapters-anthropic-and-openai.md) · Next: [The API Registry: Registering and Looking Up Providers](./api-registry-and-extensibility.md) →
