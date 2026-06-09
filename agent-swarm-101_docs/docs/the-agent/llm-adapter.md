---
title: "The LLM Adapter"
description: "Build a real LLM adapter backed by provider config (baseURL + API key + model), covering OpenAI Chat Completions and Anthropic Messages shapes with streaming and usage/cost capture."
category: the-agent
type: tutorial
tags:
  [
    LLM adapter,
    OpenAI,
    Anthropic,
    provider config,
    baseURL,
    API key,
    model,
    streaming,
    SSE,
    usage tracking,
    cost tracking,
    OpenAI Chat Completions,
    Anthropic Messages,
    OpenRouter,
    Ollama,
    ANTHROPIC_API_KEY,
    OPENAI_API_KEY,
    model registry,
    swarm adapter,
    stream function,
    AssistantMessage,
    calculateCost,
    getEnvApiKey,
    openai-completions,
    anthropic-messages,
  ]
keywords:
  [
    llm provider adapter,
    provider config shape,
    real model call,
    token counting,
    cost per call,
    SSE parse,
    server-sent events,
    chat completions API,
    messages API,
    unified stream,
    getModel,
    complete,
    streamSimple,
    SwarmAdapter contract,
    mock adapter,
    local proxy,
    custom endpoint,
  ]
sources: [S50, S51, S46, S47, S43, S48]
---

**TL;DR** — The mock adapter proved that our wiring works; now we want _real_ model responses. This chapter builds an LLM adapter that turns an invocation context into an actual API call, configurable with any OpenAI Chat Completions-compatible endpoint or the Anthropic Messages API. By the end you will have an adapter that streams a response, captures token counts and cost, and returns the same unified `AdapterResult` that the rest of the swarm already expects.

# The LLM Adapter

In the previous chapter we built a [mock adapter](./mock-adapter.md) — a keyless stub whose only job was to prove that the `SwarmAdapter` interface wired up correctly. Now we have a different problem: real tasks need real answers, and real answers come from actual model APIs.

We also want one adapter to cover many providers. Your team might run `gpt-4o-mini` for cheap triage tasks, `claude-sonnet-4` for careful coding work, and a local Ollama model during development — all in the same swarm. Hard-coding one provider means rewriting the adapter each time someone swaps models. What we want instead is a **provider config** that any LLM adapter instance reads at construction time: here is the base URL, here is the key, here is the model id. The adapter does not care what sits behind the URL as long as the wire protocol is one it knows.

Before we get into the request shapes, a quick recap of the contract we are implementing. The full definition lives in [The Adapter Interface](./adapter-interface.md); here is the relevant portion:

```ts
// Simplified view of SwarmAdapter (canonical definition in adapter-interface.md)
interface SwarmAdapter {
  invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;
}
```

- `AdapterAgent` — a minimal description of the agent being invoked: its `id`, `workspaceId`, `name`, and `adapterConfig` (the backend-specific settings).
- `InvocationContext` — the execution context the orchestrator builds for this run: a `runId`, the agent's session state, task assignments, and a `config` map the adapter can read to find the messages or prompt it should send.
- `HeartbeatRun` — the lightweight handle (`id` + `externalRunId`) the orchestrator holds for a running execution; `status` and `cancel` both receive it.

The canonical `AdapterResult` shape (from P4) has two required fields and several optional ones an LLM adapter fills in:

```ts
// The unified result type (canonical definition in adapter-interface.md)
interface AdapterResult {
  exitCode: number | null;  // null for non-process adapters like this one
  timedOut: boolean;

  // Optional — reported by adapters that talk to an LLM:
  usage?: UsageSummary;     // { inputTokens, outputTokens, cachedInputTokens? }
  provider?: string | null; // e.g. "anthropic", "openai"
  model?: string | null;    // e.g. "gpt-4o-mini"
  costUsd?: number | null;  // total cost for this run in US dollars
  summary?: string | null;  // the model's text output

  // Other optional fields for error details, session continuity, structured output…
}
```

The LLM adapter sets `exitCode: null` (it does not spawn a process), `timedOut: false`, and fills in `usage`, `provider`, `model`, `costUsd`, and `summary` from the model response. The orchestrator stores these to write cost events and budget records — it does not care which provider produced them.

The `InvocationContext` carries the messages or prompt the orchestrator has assembled for this run. The adapter reads them from `context` — there is no separate `Task` type needed here; everything the adapter requires to make the call arrives through `agent` and `context`.

## The provider config shape

The first building block is a **provider config** — the three things every real LLM call needs:

| Field | What it holds | Example |
|-------|---------------|---------|
| `api` | Which wire protocol to use | `"openai-completions"` or `"anthropic-messages"` |
| `provider` | A stable provider label | `"openai"`, `"anthropic"`, `"ollama"` |
| `baseUrl` | Where to send the HTTP request | `"https://api.openai.com/v1"` |
| `id` | The model id the provider expects | `"gpt-4o-mini"` |
| `cost` | Per-million-token prices | `{ input: 0.15, output: 0.6, cacheRead: 0, cacheWrite: 0 }` |
| `maxTokens` | Maximum output token budget | `16384` |

The `cost` field is in **US dollars per million tokens**, mirroring the convention used in the `@swarm/llm` model registry (S43, S48). That makes the arithmetic in `calculateCost` straightforward: multiply rate by actual tokens, divide by one million.

In TypeScript:

```ts
// packages/agent-core/src/adapters/llm-adapter.ts

export type ProviderApi = "openai-completions" | "anthropic-messages";

export interface ProviderConfig {
  api: ProviderApi;
  provider: string;
  baseUrl: string;
  id: string;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  maxTokens: number;
  reasoning?: boolean;
}
```

We keep the shape small; fields the wire protocol never reads (like internal display names) live elsewhere.

### Resolving the API key from the environment

We want the adapter to work without forcing every call site to pass a key — you should be able to set `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` in your shell and have things work automatically. The `@swarm/llm` package exposes `getEnvApiKey(provider)` for exactly this (S46):

```ts
// env-api-keys.ts (simplified view)
const envMap: Record<string, string> = {
  openai:    "OPENAI_API_KEY",
  anthropic: "ANTHROPIC_API_KEY",
  openrouter: "OPENROUTER_API_KEY",
  // ... other providers
};

export function getEnvApiKey(provider: string): string | undefined {
  // For "anthropic", checks ANTHROPIC_OAUTH_TOKEN first, then ANTHROPIC_API_KEY
  const envVars = getApiKeyEnvVars(provider);
  if (!envVars) return undefined;
  const found = envVars.filter(v => !!process.env[v]);
  return found.length > 0 ? process.env[found[0]] : undefined;
}
```

The key lookup order for Anthropic is: `ANTHROPIC_OAUTH_TOKEN` first, then `ANTHROPIC_API_KEY`. For OpenAI it is simply `OPENAI_API_KEY`. Any OpenAI-compatible provider you define with your own `provider` string will need its key passed explicitly unless you add it to the map, but `"openai"` and `"anthropic"` work out of the box.

We will use this in our adapter's `invoke` method: attempt to pull the key from the environment, and throw a clear error if it is absent.

## The OpenAI Chat Completions path

The **OpenAI Chat Completions** protocol is the most widely implemented: it is natively what `gpt-4o`, `gpt-4o-mini`, and the o-series reasoning models speak, but it is also the protocol that OpenRouter, Ollama, LM Studio, vLLM, and most local inference servers implement. That means one code path covers all of them — the only thing that changes is `baseUrl`.

### Building the request

The Completions endpoint expects a JSON body with `model`, `messages`, `stream: true`, and optionally `stream_options: { include_usage: true }` to get token counts in the stream (S50):

```ts
// Simplified view of the request shape
const params = {
  model: config.id,
  messages,       // converted from our internal Message[] format
  stream: true,
  stream_options: { include_usage: true },  // so usage arrives in the final chunk
  max_completion_tokens: config.maxTokens,  // or max_tokens for some providers
};
```

The `stream_options: { include_usage: true }` field tells the API to include a `usage` object on the final SSE chunk (S50). Without it, most OpenAI-compatible endpoints do not report token counts, which means we cannot track cost.

Notice the token limit field name: OpenAI uses `max_completion_tokens` for modern models, but some compatible servers (Ollama, older proxies, Moonshot, Together AI) still expect the older `max_tokens`. The `@swarm/llm` compat layer auto-detects this from the `baseUrl`, but when you build a model config by hand, pick the one your server expects (S50):

```ts
// max_completion_tokens is the current standard for OpenAI itself
// older OpenAI-compatible servers may require max_tokens instead
const tokenField =
  config.provider === "ollama" ? "max_tokens" : "max_completion_tokens";
```

### Parsing the streamed (SSE) response

The response arrives as **Server-Sent Events** — a text stream where each line starts with `data:`, and the payload is JSON. The adapter loops over chunks, accumulates text deltas, and watches for the `usage` object (S50):

```ts
// Simplified view of the SSE parsing loop for openai-completions
let text = "";
let usage: RawUsage | undefined;

for await (const chunk of openaiStream) {
  if (!chunk || typeof chunk !== "object") continue;

  // usage arrives on the final chunk (when stream_options.include_usage is set)
  if (chunk.usage) {
    usage = chunk.usage;
  }

  const choice = Array.isArray(chunk.choices) ? chunk.choices[0] : undefined;
  if (!choice) continue;

  if (choice.delta?.content) {
    text += choice.delta.content;
  }

  // finish_reason tells us how generation ended
  if (choice.finish_reason) {
    stopReason = mapStopReason(choice.finish_reason);
  }
}
```

The `finish_reason` values map to our internal `StopReason` type (S50):

| OpenAI `finish_reason` | Our `stopReason` |
|------------------------|-----------------|
| `"stop"` | `"stop"` |
| `"length"` | `"length"` |
| `"tool_calls"` | `"toolUse"` |
| `"content_filter"` | `"error"` |
| `"network_error"` | `"error"` |

### Turning raw token counts into cost

Once we have the raw usage object from the final chunk, we compute cost using `calculateCost` from `@swarm/llm` (S48):

```ts
// models.ts — calculateCost (simplified view)
export function calculateCost(model: { cost: CostRates }, usage: Usage): Usage["cost"] {
  usage.cost.input     = (model.cost.input     / 1_000_000) * usage.input;
  usage.cost.output    = (model.cost.output    / 1_000_000) * usage.output;
  usage.cost.cacheRead = (model.cost.cacheRead / 1_000_000) * usage.cacheRead;
  usage.cost.cacheWrite= (model.cost.cacheWrite/ 1_000_000) * usage.cacheWrite;
  usage.cost.total     =
    usage.cost.input + usage.cost.output + usage.cost.cacheRead + usage.cost.cacheWrite;
  return usage.cost;
}
```

The raw OpenAI chunk usage shape uses `prompt_tokens` and `completion_tokens`. Cache-read tokens appear in `prompt_tokens_details.cached_tokens` (S50):

```ts
// Simplified view of parseChunkUsage for openai-completions
function parseRawUsage(raw: RawChunkUsage, cost: CostRates): Usage {
  const promptTokens   = raw.prompt_tokens ?? 0;
  const cacheReadTokens= raw.prompt_tokens_details?.cached_tokens ?? 0;
  const cacheWriteTokens = raw.prompt_tokens_details?.cache_write_tokens ?? 0;
  const input = Math.max(0, promptTokens - cacheReadTokens - cacheWriteTokens);
  const output= raw.completion_tokens ?? 0;

  const usage: Usage = {
    input,
    output,
    cacheRead: cacheReadTokens,
    cacheWrite: cacheWriteTokens,
    totalTokens: input + output + cacheReadTokens + cacheWriteTokens,
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
  };
  calculateCost({ cost }, usage);
  return usage;
}
```

The subtraction (`promptTokens - cacheReadTokens - cacheWriteTokens`) prevents double-counting: `prompt_tokens` includes cached tokens, so subtracting them gives us the uncached input token count (S50).

### Why OpenRouter and Ollama work with the same code

The OpenAI Chat Completions path works for any endpoint that speaks the same protocol. You aim it at a different `baseUrl` and it works:

| Provider | `baseUrl` | Key env var |
|----------|-----------|-------------|
| OpenAI | `https://api.openai.com/v1` | `OPENAI_API_KEY` |
| OpenRouter | `https://openrouter.ai/api/v1` | `OPENROUTER_API_KEY` |
| Ollama (local) | `http://localhost:11434/v1` | _(none needed)_ |
| LM Studio | `http://localhost:1234/v1` | _(none needed)_ |
| vLLM | `http://localhost:8000/v1` | _(none needed)_ |

For Ollama, the `apiKey` field is irrelevant — pass `"dummy"` or an empty string (S43). For OpenRouter, set `provider` to `"openrouter"` so `getEnvApiKey` picks up `OPENROUTER_API_KEY` automatically (S46).

## The Anthropic Messages path

The **Anthropic Messages API** speaks a different wire protocol: the request body uses `messages` (same concept) but the streaming events are named differently, and usage arrives in `message_start` and `message_delta` events rather than on the final SSE chunk.

### Building the request

Anthropic's request has `model`, `messages`, `max_tokens`, and `stream: true` at the top level (S51):

```ts
// Simplified view of the Anthropic request shape
const params: MessageCreateParamsStreaming = {
  model: config.id,
  messages,
  max_tokens: config.maxTokens,
  stream: true,
  system: context.systemPrompt
    ? [{ type: "text", text: context.systemPrompt }]
    : undefined,
};
```

Notice that `system` is a **content block array**, not a plain string — the Anthropic API accepts cache-control annotations per block, which is why it is structured this way (S51).

### Parsing the Anthropic SSE event stream

The Anthropic stream sends named events. The ones we care about are (S51):

| Event type | What it carries |
|------------|-----------------|
| `message_start` | Initial input token count (`usage.input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`) |
| `content_block_start` | New content block begins (type: `"text"`, `"thinking"`, or `"tool_use"`) |
| `content_block_delta` | Incremental text/thinking/JSON delta |
| `content_block_stop` | Block complete |
| `message_delta` | Stop reason + updated output token count |
| `message_stop` | Stream complete |

The custom SSE decoder in `@swarm/llm` (S51) reads the raw `ReadableStream<Uint8Array>` line by line, parsing the `event:` and `data:` fields:

```ts
// Simplified view of the SSE parsing loop for anthropic-messages
let text = "";
let usage: Usage = { input: 0, output: 0, cacheRead: 0, cacheWrite: 0,
                     totalTokens: 0, cost: { ... } };

for await (const event of iterateAnthropicEvents(response)) {
  if (event.type === "message_start") {
    // Capture initial token counts (input arrives here)
    usage.input     = event.message.usage.input_tokens ?? 0;
    usage.cacheRead = event.message.usage.cache_read_input_tokens ?? 0;
    usage.cacheWrite= event.message.usage.cache_creation_input_tokens ?? 0;

  } else if (event.type === "content_block_start" && event.content_block.type === "text") {
    // a new text block is starting

  } else if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    text += event.delta.text;

  } else if (event.type === "message_delta") {
    // output tokens and stop reason arrive here
    if (event.usage.output_tokens != null) {
      usage.output = event.usage.output_tokens;
    }
    if (event.delta.stop_reason) {
      stopReason = mapStopReason(event.delta.stop_reason);
    }
  }
}
// compute total and cost
usage.totalTokens = usage.input + usage.output + usage.cacheRead + usage.cacheWrite;
calculateCost({ cost: config.cost }, usage);
```

The Anthropic stop reason strings map to our internal type (S51):

| Anthropic `stop_reason` | Our `stopReason` |
|-------------------------|-----------------|
| `"end_turn"` | `"stop"` |
| `"max_tokens"` | `"length"` |
| `"tool_use"` | `"toolUse"` |
| `"pause_turn"` | `"stop"` |
| `"refusal"` | `"error"` |

One subtlety: Anthropic does not emit a `total_tokens` field. We compute it ourselves by summing the four components (S51). This is why both provider paths go through the same `calculateCost` function — the formula is provider-agnostic once you have the four counts.

## Selecting the right path: the unified `stream()` function

We now have two provider-specific paths. We need something that looks at a model's config and dispatches to the right one. The `@swarm/llm` package solves this with a **unified `stream()` function** and a **model registry** (S47, S48):

```ts
// stream.ts (simplified view from @swarm/llm)
export function stream<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): AssistantMessageEventStream {
  // 1. Look up the API provider function registered for model.api
  const provider = resolveApiProvider(model.api);
  // 2. If no explicit apiKey in options, pull from environment
  const opts = withEnvApiKey(model, options);
  // 3. Dispatch to the provider's stream() implementation
  return provider.stream(model, context, opts);
}
```

`withEnvApiKey` calls `getEnvApiKey(model.provider)` and merges the result into `options` only if no key was already supplied (S47). The env-key lookup is the same one we saw in S46.

The model registry (S48) maps `(provider, modelId)` pairs to `Model<Api>` objects, each carrying the `api`, `baseUrl`, `cost`, `maxTokens`, and other fields we discussed:

```ts
// models.ts (simplified view)
export function getModel<TProvider, TModelId>(
  provider: TProvider,
  modelId: TModelId,
): Model<...> {
  return modelRegistry.get(provider)?.get(modelId as string) as Model<...>;
}

// Usage:
const model = getModel("openai", "gpt-4o-mini");
// model.api      === "openai-completions"
// model.baseUrl  === "https://api.openai.com/v1"
// model.cost     === { input: <X>, output: <Y>, ... }  ($ per million tokens — from provider pricing)
// model.maxTokens=== 16384
```

For models not in the built-in registry — a local Ollama model, a custom proxy — you construct the `Model<Api>` object directly (S43):

```ts
import type { Model } from "@swarm/llm";

const ollamaModel: Model<"openai-completions"> = {
  id: "llama-3.1-8b",
  name: "Llama 3.1 8B (Ollama)",
  api: "openai-completions",
  provider: "ollama",
  baseUrl: "http://localhost:11434/v1",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 32000,
};
```

The `stream()` function treats this custom model identically to a registered one — it reads `model.api` to pick the provider path and `model.provider` to look up the API key.

## Putting it all together: the LLM adapter class

We now have all the pieces. Let's assemble them into the adapter class that the orchestrator will use:

```ts
// packages/agent-core/src/adapters/llm-adapter.ts

import {
  stream,
  getEnvApiKey,
  type Model,
  type Api,
  type Context,
  type AssistantMessage,
} from "@swarm/llm";
import type {
  SwarmAdapter,
  AdapterAgent,
  AdapterResult,
  HeartbeatRun,
  InvocationContext,
  RunStatus,
} from "../types.ts";

export class LlmAdapter implements SwarmAdapter {
  private model: Model<Api>;

  constructor(model: Model<Api>) {
    this.model = model;
  }

  async invoke(
    _agent: AdapterAgent,
    context: InvocationContext,
  ): Promise<AdapterResult> {
    // Resolve the API key from the environment; fail fast with a clear message
    // if it is absent (e.g. OPENAI_API_KEY or ANTHROPIC_API_KEY not set).
    const apiKey = getEnvApiKey(this.model.provider);
    if (!apiKey) {
      throw new Error(
        `No API key for provider: ${this.model.provider}. ` +
        `Set the appropriate env var (e.g. OPENAI_API_KEY or ANTHROPIC_API_KEY).`
      );
    }

    // Build the LLM context from what the orchestrator passed in.
    // context.config and context.context carry the messages/prompt assembled
    // for this run — no separate Task type needed.
    const llmContext: Context = {
      systemPrompt: (context.config["systemPrompt"] as string) ?? undefined,
      messages: (context.config["messages"] as Context["messages"]) ?? [],
    };

    // stream() dispatches to the correct API path (openai-completions or
    // anthropic-messages) based on this.model.api, and injects the env key.
    const s = stream(this.model, llmContext, { apiKey });

    // .result() awaits the full stream and returns the final AssistantMessage.
    // For a streaming UI, iterate the event stream directly instead (see below).
    const message: AssistantMessage = await s.result();

    if (message.stopReason === "error" || message.stopReason === "aborted") {
      throw new Error(
        `LLM call failed (${message.stopReason}): ${message.errorMessage}`
      );
    }

    // Collect the plain-text output from all text content blocks.
    const outputText = message.content
      .filter((b) => b.type === "text")
      .map((b) => (b as { text: string }).text)
      .join("");

    // Map into P4's AdapterResult shape.
    // exitCode is null — this adapter does not spawn a process.
    // usage, model, provider, costUsd, and summary are the optional LLM fields.
    return {
      exitCode: null,
      timedOut: false,
      usage: message.usage
        ? {
            inputTokens: message.usage.input,
            outputTokens: message.usage.output,
            cachedInputTokens: message.usage.cacheRead,
          }
        : undefined,
      model: this.model.id,
      provider: this.model.provider,
      costUsd: message.usage?.cost?.total ?? null,
      summary: outputText,
    };
  }

  async status(_run: HeartbeatRun): Promise<RunStatus> {
    // Local invocations are synchronous — by the time invoke() resolves the
    // run is already complete; there is no external state to poll.
    return "succeeded";
  }

  async cancel(_run: HeartbeatRun): Promise<void> {
    // For a streaming invocation you would store an AbortController at invoke
    // time and call abort() here. Synchronous usage needs no action.
  }
}
```

A few things to notice:

- `stream()` returns an `AssistantMessageEventStream`. Calling `.result()` on it awaits the entire stream and returns the final `AssistantMessage` — convenient for synchronous use. For a streaming UI you would iterate the event stream directly with `for await (const event of s) { ... }` instead (S43).
- `message.usage` already has `cost` fully computed inside the provider path — `calculateCost` is called before the stream resolves (S50, S51). You do not need to call it again.
- The `stopReason` field on `AssistantMessage` uses our internal vocabulary (`"stop"`, `"length"`, `"toolUse"`, `"error"`, `"aborted"`). Both the OpenAI and Anthropic provider paths map their provider-specific finish-reason strings to this shared enum before returning (S50, S51).

### Wiring in the adapter

In your orchestrator's adapter factory, you can now construct an `LlmAdapter` from any registered model or custom model config:

```ts
import { getModel } from "@swarm/llm";
import { LlmAdapter } from "./adapters/llm-adapter.ts";

// Using a model from the registry (OPENAI_API_KEY must be set)
const gptAdapter = new LlmAdapter(getModel("openai", "gpt-4o-mini"));

// Using a local Ollama model (no key needed)
const ollamaAdapter = new LlmAdapter({
  id: "llama-3.1-8b",
  name: "Llama 3.1 8B (Ollama)",
  api: "openai-completions",
  provider: "ollama",
  baseUrl: "http://localhost:11434/v1",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 32000,
});

// Using Anthropic (ANTHROPIC_API_KEY must be set)
const claudeAdapter = new LlmAdapter(getModel("anthropic", "claude-sonnet-4-20250514"));
```

The orchestrator calls `adapter.invoke(agent, context)` on each — passing the agent descriptor and the invocation context it already holds — gets back an `AdapterResult` with `usage` and cost filled, and the per-run budget accounting works identically regardless of which backend answered.

## The event stream: what you get when you iterate

When you need streaming output — for example, to pipe tokens to a terminal UI — use `for await` on the stream rather than `.result()`. The events emitted by both provider paths share the same type names (S43):

| Event type | When it fires | Useful fields |
|------------|---------------|---------------|
| `start` | Stream begins | `partial` — initial shell of the `AssistantMessage` |
| `text_start` | A text block opens | `contentIndex` — position in the content array |
| `text_delta` | Incremental text chunk | `delta` — the new characters |
| `text_end` | Text block complete | `content` — full accumulated text |
| `toolcall_start` | A tool call begins | `contentIndex` |
| `toolcall_delta` | Tool argument JSON chunk | `delta`, partial `arguments` |
| `toolcall_end` | Tool call complete | `toolCall.name`, `toolCall.arguments` |
| `done` | Stream finished | `reason`, `message` — the full `AssistantMessage` |
| `error` | Error or abort | `error` — partial `AssistantMessage` with `errorMessage` |

Events for different content blocks can be interleaved — you may see `text_delta` followed by `toolcall_start` followed by another `text_delta`. Always track by `contentIndex`, not by event order (S43).

```ts
const s = stream(model, context, { apiKey });

for await (const event of s) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta);
  } else if (event.type === "done") {
    const msg = event.message;
    console.log(
      `\ntokens: ${msg.usage.input} in / ${msg.usage.output} out` +
      `  cost: $${msg.usage.cost.total.toFixed(6)}`
    );
  }
}
```

## The result shape the orchestrator sees

When `invoke()` resolves, the `AdapterResult` carries everything the orchestrator needs to write a run record. The shape conforms to the canonical P4 contract — `exitCode` and `timedOut` are always present; the LLM-specific optional fields carry the usage, identity, and output:

```ts
// What the orchestrator receives back (P4 AdapterResult shape)
// NOTE: cost values below are illustrative only.
// Actual rates come from the model's cost config (your provider's pricing page).
// Suppose input is priced at $X per million tokens and output at $Y per million tokens:
{
  exitCode: null,         // not a process; always null for this adapter
  timedOut: false,

  usage: {
    inputTokens: 84,
    outputTokens: 12,
    cachedInputTokens: 0,
  },
  model: "gpt-4o-mini",
  provider: "openai",
  costUsd: (X / 1_000_000) * 84 + (Y / 1_000_000) * 12,  // illustrative
  summary: "The answer is 42.",
}
```

The formula is what matters: `calculateCost` divides each rate (dollars per million tokens, from `model.cost`) by one million, then multiplies by the actual token count (S48). The specific dollar values depend on which model and provider you use — check your provider's current pricing page or the model registry entry to find the real rates.

This is the same shape whether the call went to OpenAI, Anthropic, or a local Ollama server. The orchestrator has no provider-specific code — it reads `costUsd` and writes a cost event, and the budget tracker accumulates totals regardless of which backend produced them.

<!-- GAP: per-model price rates are provider-specific and not fixed by this guide; values shown in the AdapterResult example above are illustrative — check your provider's current pricing page or the model registry for real rates -->
<!-- GAP: The exact Model<Api> interface definition (all fields) is not reproduced in full in S43/S48/S47 — the tutorial reconstructs the essential fields from usage in those sources; a reader wanting the exhaustive type definition would need to read the @swarm/llm types.ts directly -->

## Try it yourself

Here are three exercises that extend what we built:

**1. Aim at a local OpenAI-compatible server.** Start Ollama (`ollama serve`) and pull a model (`ollama pull llama3.2`). Construct a custom `Model<"openai-completions">` with `baseUrl: "http://localhost:11434/v1"` and `provider: "ollama"`. Call `invoke` with a short task — you should see real output and zero cost (because `cost` is `{ input: 0, output: 0, ... }`).

**2. Add a third provider profile.** Create a model config targeting OpenRouter: `provider: "openrouter"`, `baseUrl: "https://openrouter.ai/api/v1"`, and set `OPENROUTER_API_KEY` in your shell. Pass a model id that OpenRouter supports (for example `"anthropic/claude-3-5-haiku"`) — the `openai-completions` path handles it transparently because OpenRouter speaks the Chat Completions protocol.

**3. Log per-call cost.** Wrap the `LlmAdapter.invoke` method to print `result.costUsd` after each call. Run the same prompt through the `gpt-4o-mini` config and through a zero-cost Ollama config. Verify that the totals differ appropriately (the Ollama result should show `0` because its `cost` config is all zeros), and that the orchestrator's budget tracker would accumulate the right totals over multiple calls.

---

← Previous: [The Mock Adapter](./mock-adapter.md) · Next: [Process and HTTP Adapters](./process-and-http-adapters.md) →
