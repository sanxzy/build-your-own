---
title: "Provider Adapters: Anthropic and OpenAI"
description: "How the Anthropic SSE adapter and the OpenAI Chat Completions adapter translate generic requests and streamed bytes into EventStream events."
category: llm-toolkit
type: tutorial
tags: [Anthropic, OpenAI, provider adapter, SSE, server-sent events, cache-control, delta reassembly, xAI, Groq, tool normalisation, thinking, compat flags, llm-toolkit, provider, streaming, EventStream, AssistantMessageEventStream, streamAnthropic, streamOpenAICompletions, CacheRetention, AnthropicOptions, OpenAICompletionsOptions, transformMessages]
keywords: [SSE parsing, content_block_delta, input_json_delta, thinking blocks, budget tokens, adaptive thinking, fine-grained tool streaming, interleaved thinking beta, cache_control ephemeral, tool call normalisation, OpenAI-compatible provider, cerebras, deepseek, groq, provider compat detection]
sources: [S11, S12, S16, S22]
---

**TL;DR** — Each AI provider speaks its own wire protocol. A _provider adapter_ is the bridge: it converts our generic `Context` and `Message` types into the provider's request format, streams the response back, and pushes typed events into an `AssistantMessageEventStream`. This chapter walks through building both the Anthropic SSE adapter (cache control, thinking blocks, tool normalisation) and the OpenAI Chat Completions adapter (delta reassembly, the compat-flag system that lets one adapter serve xAI, Groq, Cerebras, DeepSeek, and others). By the end you will understand every translation layer between a user turn and a parsed assistant message.

# Provider Adapters: Anthropic and OpenAI

## The problem — every vendor speaks a different language

By this point we have a unified type model (see [Message Types and the Streaming API](./message-types-and-streaming-api.md) for a recap of `Message`, `AssistantMessage`, and `Context`) and an observable event stream (see [The EventStream Observable Backbone](./event-stream-and-observable-backbone.md) for how `AssistantMessageEventStream` works). What we do not yet have is anything that actually calls a provider.

The gap is significant. Consider what each provider requires:

- **Anthropic** speaks its own streaming format built on **Server-Sent Events (SSE)**: a sequence of named events (`message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`) each carrying a JSON payload. Thinking blocks, tool calls, and cache control all have provider-specific shapes.
- **OpenAI Chat Completions** speaks a different SSE dialect: a stream of JSON chunks where text and tool arguments arrive as _deltas_ that must be accumulated. The same endpoint is used by xAI, Groq, Cerebras, and DeepSeek — each with small but important behavioural differences.

We need a consistent internal interface that hides these differences. The pattern is:

1. Convert our generic `Context` (messages + tools + system prompt) into the provider's request object.
2. Open a streaming HTTP connection.
3. Parse the provider's bytes into typed events.
4. Push those events into the `AssistantMessageEventStream` so the rest of the system stays provider-agnostic.

Each adapter is a `StreamFunction` — a function with this shape:

```ts
// Simplified view of the StreamFunction type
type StreamFunction<TApi, TOptions> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions,
) => AssistantMessageEventStream;
```

Let's build the Anthropic adapter first, then the OpenAI one.

---

## The Anthropic SSE adapter

### Step 1 — Start the stream and initialise the output skeleton

The Anthropic adapter is exported as `streamAnthropic`. It follows the same structural pattern as every other adapter: create an `AssistantMessageEventStream`, fire off an async IIFE that does all the work, and return the stream immediately. The IIFE fills the stream while the caller can already subscribe to events.

```ts
// Simplified view of streamAnthropic
export const streamAnthropic: StreamFunction<"anthropic-messages", AnthropicOptions> = (
  model,
  context,
  options,
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream();

  (async () => {
    // Build a skeleton AssistantMessage to mutate as chunks arrive.
    const output: AssistantMessage = {
      role: "assistant",
      content: [],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, totalTokens: 0,
               cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 } },
      stopReason: "stop",
      timestamp: Date.now(),
    };

    try {
      // ... build client, build params, stream, parse, push events ...
      stream.push({ type: "done", reason: output.stopReason, message: output });
      stream.end();
    } catch (error) {
      output.stopReason = options?.signal?.aborted ? "aborted" : "error";
      output.errorMessage = error instanceof Error ? error.message : JSON.stringify(error);
      stream.push({ type: "error", reason: output.stopReason, error: output });
      stream.end();
    }
  })();

  return stream; // returned immediately; the async work runs behind it
};
```

The `output` skeleton is mutated in-place as each SSE event arrives. This means every `partial` snapshot we push into the stream is a reference to the same object — callers always see the accumulated state.

### Step 2 — Forming the request: cache control and thinking

Before we open the HTTP connection we need to build the Anthropic request object. Two parameters deserve special attention: **cache control** and **thinking configuration**.

#### Cache control

The Anthropic API supports prompt caching: attaching a `cache_control` marker to content tells the API to cache it and reuse it across turns, reducing input token costs. `llm-toolkit` supports two retention levels:

| `CacheRetention` value | Effect |
|---|---|
| `"none"` | No `cache_control` is attached anywhere |
| `"short"` | `{ type: "ephemeral" }` — default |
| `"long"` | `{ type: "ephemeral", ttl: "1h" }` — when the model supports it |

The `resolveCacheRetention` function determines which level to use. It checks the caller-supplied `cacheRetention` option first, then falls back to the `XZY_CACHE_RETENTION` environment variable, then defaults to `"short"`:

```ts
// Simplified view of resolveCacheRetention
function resolveCacheRetention(cacheRetention?: CacheRetention): CacheRetention {
  if (cacheRetention) return cacheRetention;
  if (process.env.XZY_CACHE_RETENTION === "long") return "long";
  return "short";
}
```

Once the retention level is known, `getCacheControl` builds the actual `CacheControlEphemeral` object (or omits it for `"none"`). The `ttl: "1h"` field is only added when `getAnthropicCompat(model).supportsLongCacheRetention` is true, which is false for providers like Fireworks that do not support the extended TTL.

The `cache_control` object then gets stamped onto:

- The **system prompt** block (so the system context is cached across all turns).
- The **last user message block** (so the growing conversation history is cached up to the current turn).
- The **last tool definition** in the tools array (when `supportsCacheControlOnTools` is true for the model's provider).

```ts
// From buildParams / convertMessages — how cache_control reaches the last user message
if (cacheControl && params.length > 0) {
  const lastMessage = params[params.length - 1];
  if (lastMessage.role === "user") {
    if (Array.isArray(lastMessage.content)) {
      const lastBlock = lastMessage.content[lastMessage.content.length - 1];
      if (lastBlock && (lastBlock.type === "text" || lastBlock.type === "image" || lastBlock.type === "tool_result")) {
        (lastBlock as any).cache_control = cacheControl;
      }
    } else if (typeof lastMessage.content === "string") {
      // Anthropic requires a content-block array to attach cache_control to a plain string
      lastMessage.content = [{ type: "text", text: lastMessage.content, cache_control: cacheControl }] as any;
    }
  }
}
```

#### Thinking configuration

Anthropic models can be configured to produce explicit reasoning traces — "thinking blocks" — alongside their text output. The `AnthropicOptions` interface exposes several knobs:

| Option | Type | Effect |
|---|---|---|
| `thinkingEnabled` | `boolean` | Turns thinking on or off |
| `thinkingBudgetTokens` | `number` | Token cap for budget-based (older) models. Default: 1024 when `thinkingEnabled` is true and no budget is provided |
| `effort` | `AnthropicEffort` | Effort level for adaptive thinking models: `"low"`, `"medium"`, `"high"`, `"xhigh"`, `"max"` |
| `thinkingDisplay` | `AnthropicThinkingDisplay` | `"summarized"` (default when thinking is enabled) or `"omitted"` |
| `interleavedThinking` | `boolean` | Whether to request the interleaved-thinking beta header. Default: `true` |

`buildParams` translates these options into Anthropic's `thinking` request field:

```ts
// From buildParams — thinking mode selection (simplified)
if (model.reasoning) {
  if (options?.thinkingEnabled) {
    const display: AnthropicThinkingDisplay = options.thinkingDisplay ?? "summarized";
    if (model.compat?.forceAdaptiveThinking === true) {
      // Adaptive models: Claude decides when/how much to think
      params.thinking = { type: "adaptive", display };
      if (options.effort) {
        params.output_config = { effort: options.effort };
      }
    } else {
      // Budget-based for older models
      params.thinking = {
        type: "enabled",
        budget_tokens: options.thinkingBudgetTokens || 1024,
        display,
      };
    }
  } else if (options?.thinkingEnabled === false) {
    params.thinking = { type: "disabled" };
  }
}
```

Note that temperature is suppressed when thinking is enabled (`options.thinkingEnabled` is truthy) or when the model does not support it (`!compat.supportsTemperature`).

#### Beta headers

Two beta features are toggled via `anthropic-beta` request headers:

- `"fine-grained-tool-streaming-2025-05-14"` — enables streaming tool input deltas when the model does **not** support eager tool input streaming (i.e., the `supportsEagerToolInputStreaming` compat flag is false).
- `"interleaved-thinking-2025-05-14"` — enables thinking blocks interleaved with text. Skipped automatically for adaptive thinking models (they have it built in).

### Step 3 — Parsing the SSE stream

This is where the protocol translation work lives. Anthropic streams SSE over an HTTP response body. We have a hand-rolled decoder because the SDK's higher-level iterator can miss edge cases at proxy boundaries.

The decoder chain works in layers:

1. **`iterateSseMessages`** — reads raw bytes from a `ReadableStream<Uint8Array>`, accumulates them into a string buffer, and yields individual `ServerSentEvent` objects. It handles both `\n` and `\r\n` line endings via `consumeLine` / `nextLineBreakIndex`.

2. **`decodeSseLine`** — processes one line of SSE text. An empty line flushes the accumulated event. A line beginning with `:` is a keep-alive comment and is discarded. Otherwise it splits on the first `:` to get `fieldName` and `value`, accumulating `event` and `data` lines into the `SseDecoderState`.

3. **`iterateAnthropicEvents`** — filters the `ServerSentEvent` stream to the six Anthropic message events (in the `ANTHROPIC_MESSAGE_EVENTS` set: `message_start`, `message_delta`, `message_stop`, `content_block_start`, `content_block_delta`, `content_block_stop`). Unknown events such as `done` or `proxy.stats` are silently skipped. Each accepted event's `data` field is parsed with `parseJsonWithRepair` — a fault-tolerant JSON parser that can recover from truncated or corrupted JSON.

```ts
const ANTHROPIC_MESSAGE_EVENTS: ReadonlySet<string> = new Set([
  "message_start",
  "message_delta",
  "message_stop",
  "content_block_start",
  "content_block_delta",
  "content_block_stop",
]);
```

A test from S22 captures both edge cases directly:

```ts
// From test/anthropic-sse-parsing.test.ts — unknown events after message_stop are ignored
it("ignores unknown SSE events after message_stop", async () => {
  const response = createSseResponse([
    ...minimalAnthropicEvents,
    { event: "done", data: "[DONE]" },      // non-Anthropic event
    { event: "proxy.stats", data: "not json" }, // proxy-injected event
  ]);
  const result = await stream.result();
  expect(result.stopReason).toBe("stop");
  expect(result.content).toEqual([{ type: "text", text: "Hello" }]);
});

// Same file — malformed tool JSON in a delta is repaired
it("repairs malformed SSE JSON and malformed streamed tool JSON", async () => {
  const malformedToolJsonDelta =
    String.raw`{"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"{\"path\":\"A\H\",\"text\":\"col1\tcol2\"}"}}`;
  // ...
  const result = await stream.result();
  expect(result.stopReason).toBe("toolUse");
  expect(toolCall?.arguments).toEqual({ path: "A\\H", text: "col1\tcol2" });
});
```

The first test confirms that proxy-injected events (`done`, `proxy.stats`) do not corrupt the parsed output. The second confirms that a tool argument payload containing invalid escape sequences (`\H`, a literal tab character) is repaired rather than thrown.

### Step 4 — Mapping SSE events to EventStream pushes

Once we have a typed `RawMessageStreamEvent`, we translate it into `AssistantMessageEventStream` pushes. Each content block type (text, thinking, tool use) follows the same three-event lifecycle: `_start` → one or more `_delta` → `_stop`.

#### Text blocks

```ts
// content_block_start with type "text" → creates a TextContent block
case "content_block_start" (type === "text"):
  const block = { type: "text", text: "", index: event.index };
  output.content.push(block);
  stream.push({ type: "text_start", contentIndex: output.content.length - 1, partial: output });

// content_block_delta with type "text_delta" → appends to the block
case "content_block_delta" (delta.type === "text_delta"):
  block.text += event.delta.text;
  stream.push({ type: "text_delta", contentIndex: index, delta: event.delta.text, partial: output });

// content_block_stop → finalises
case "content_block_stop":
  stream.push({ type: "text_end", contentIndex: index, content: block.text, partial: output });
```

#### Thinking blocks

Thinking blocks come in two varieties. A `"thinking"` content block carries a `thinking_delta` with the reasoning text, plus a trailing `signature_delta` (an opaque encrypted signature the API requires when replaying thinking across turns). A `"redacted_thinking"` block is one the API has chosen not to surface — it carries only the opaque `data` payload needed for replay; the adapter stores it as `{ thinking: "[Reasoning redacted]", thinkingSignature: event.content_block.data, redacted: true }`.

```ts
// Simplified view of thinking block handling at content_block_start
} else if (event.content_block.type === "thinking") {
  const block = { type: "thinking", thinking: "", thinkingSignature: "", index: event.index };
  output.content.push(block);
  stream.push({ type: "thinking_start", ... });
} else if (event.content_block.type === "redacted_thinking") {
  const block = {
    type: "thinking",
    thinking: "[Reasoning redacted]",
    thinkingSignature: event.content_block.data,
    redacted: true,
    index: event.index,
  };
  output.content.push(block);
  stream.push({ type: "thinking_start", ... });
}

// content_block_delta with type "signature_delta" → accumulates the thinking signature
} else if (event.delta.type === "signature_delta") {
  block.thinkingSignature = (block.thinkingSignature || "") + event.delta.signature;
}
```

The thinking signature must survive into the next turn — it is what lets Anthropic verify that the thinking block has not been tampered with.

#### Tool call blocks

Tool argument JSON arrives as a stream of `input_json_delta` events — partial JSON fragments that must be accumulated and parsed incrementally. The adapter tracks a `partialJson` scratch string per tool block:

```ts
} else if (event.delta.type === "input_json_delta") {
  block.partialJson += event.delta.partial_json;
  block.arguments = parseStreamingJson(block.partialJson); // partial parse — best effort
  stream.push({ type: "toolcall_delta", contentIndex: index, delta: event.delta.partial_json, partial: output });
}

// At content_block_stop, do a final parse and strip the scratch buffer
block.arguments = parseStreamingJson(block.partialJson);
delete (block as { partialJson?: string }).partialJson;
stream.push({ type: "toolcall_end", contentIndex: index, toolCall: block, partial: output });
```

After the stop, `partialJson` is deleted from the block so it never appears in the persisted `AssistantMessage`.

#### Tool name normalisation

You might wonder: why would tool names need normalising? Some OAuth-authenticated paths route through a Claude Code proxy that expects tool names in Claude Code's canonical capitalisation (`Read`, `Write`, `Edit`, `Bash`, …). The adapter keeps a lookup map and converts names bi-directionally:

```ts
const claudeCodeTools = ["Read", "Write", "Edit", "Bash", "Grep", "Glob", ...];
const ccToolLookup = new Map(claudeCodeTools.map((t) => [t.toLowerCase(), t]));

// On the way out (our name → Claude Code name, for OAuth requests)
const toClaudeCodeName = (name: string) => ccToolLookup.get(name.toLowerCase()) ?? name;

// On the way in (Claude Code name → our name, when parsing tool_use blocks)
const fromClaudeCodeName = (name: string, tools?: Tool[]) => {
  if (tools && tools.length > 0) {
    const lowerName = name.toLowerCase();
    const matched = tools.find((tool) => tool.name.toLowerCase() === lowerName);
    if (matched) return matched.name;
  }
  return name;
};
```

`isOAuthToken` (which checks whether the API key contains `"sk-ant-oat"`) determines which direction the normalisation goes.

#### Tool call ID normalisation

Anthropic requires tool call IDs to match `^[a-zA-Z0-9_-]+$` and be at most 64 characters. Tool call IDs that come from cross-provider turns (e.g., an OpenAI Responses API `|`-separated ID) are normalised before they reach the Anthropic request:

```ts
function normalizeToolCallId(id: string): string {
  return id.replace(/[^a-zA-Z0-9_-]/g, "_").slice(0, 64);
}
```

#### Stop reason mapping

Anthropic's `stop_reason` strings do not match the internal `StopReason` type. `mapStopReason` translates them:

| Anthropic `stop_reason` | Internal `StopReason` |
|---|---|
| `"end_turn"` | `"stop"` |
| `"max_tokens"` | `"length"` |
| `"tool_use"` | `"toolUse"` |
| `"refusal"` | `"error"` |
| `"pause_turn"` | `"stop"` |
| `"stop_sequence"` | `"stop"` |
| `"sensitive"` | `"error"` |

Any unrecognised value throws — this surfaces newly added Anthropic stop reasons rather than silently swallowing them.

### Step 5 — Token usage

`message_start` delivers initial token counts (`input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`). `message_delta` may deliver updated counts. The adapter captures both, favouring the `message_start` counts when `message_delta` omits them (some proxies set those fields to `null`):

```ts
// message_delta — only update if non-null (preserves message_start values when proxy omits them)
if (event.usage.input_tokens != null) output.usage.input = event.usage.input_tokens;
if (event.usage.output_tokens != null) output.usage.output = event.usage.output_tokens;
if (event.usage.cache_read_input_tokens != null) output.usage.cacheRead = event.usage.cache_read_input_tokens;
if (event.usage.cache_creation_input_tokens != null) output.usage.cacheWrite = event.usage.cache_creation_input_tokens;
// Anthropic does not provide total_tokens; compute from components
output.usage.totalTokens = output.usage.input + output.usage.output + output.usage.cacheRead + output.usage.cacheWrite;
```

---

## The OpenAI Chat Completions adapter

### The problem with one adapter for many providers

The OpenAI Chat Completions endpoint is a standard that many providers implement — xAI's Grok, Groq, Cerebras, DeepSeek, and others all speak it. But "compatible" does not mean "identical". Differences include:

- Which HTTP field carries the token limit (`max_tokens` vs `max_completion_tokens`)
- Whether the `store` field is accepted
- Whether reasoning/thinking must be toggled via a special field, and what that field is called
- Whether tool results require a `name` field
- Whether an assistant turn must follow a tool result
- Which role (`system` vs `developer`) receives the system prompt
- How to enable caching

Maintaining a separate adapter per provider would mean duplicating a large amount of streaming logic. Instead, `streamOpenAICompletions` is a single adapter parameterised by a **compatibility object** that captures provider differences.

### Step 1 — The compat system

Every provider quirk is expressed as a field on `OpenAICompletionsCompat`. The `getCompat` function resolves the effective compat for a model: it first runs `detectCompat` (which auto-detects from the provider name and `baseUrl`), then overlays any fields the caller explicitly set in `model.compat`.

Here is the full set of compat flags the adapter reads:

| Flag | Type | What it controls |
|---|---|---|
| `supportsStore` | `boolean` | Whether to send `store: false` in the request |
| `supportsDeveloperRole` | `boolean` | Whether to use `"developer"` role for the system prompt (reasoning models) |
| `supportsReasoningEffort` | `boolean` | Whether `reasoning_effort` is a valid request field |
| `supportsUsageInStreaming` | `boolean` | Whether `stream_options: { include_usage: true }` is safe to send |
| `maxTokensField` | `"max_tokens" \| "max_completion_tokens"` | Which field carries the token limit |
| `requiresToolResultName` | `boolean` | Whether tool result messages need a `name` field |
| `requiresAssistantAfterToolResult` | `boolean` | Whether to inject a synthetic assistant turn after tool results |
| `requiresThinkingAsText` | `boolean` | Whether thinking blocks must be converted to plain text |
| `requiresReasoningContentOnAssistantMessages` | `boolean` | Whether to inject an empty `reasoning_content` field (DeepSeek) |
| `thinkingFormat` | `string` | How to express thinking/reasoning (`"openai"`, `"deepseek"`, `"zai"`, `"qwen"`, `"openrouter"`, etc.) |
| `supportsStrictMode` | `boolean` | Whether to include `strict: false` in tool definitions |
| `cacheControlFormat` | `"anthropic" \| undefined` | Whether to apply Anthropic-style `cache_control` fields via this OpenAI endpoint |
| `sendSessionAffinityHeaders` | `boolean` | Whether to send `x-session-affinity` / `session_id` headers |
| `supportsLongCacheRetention` | `boolean` | Whether the provider supports the 24h prompt cache retention |
| `zaiToolStream` | `boolean` | Whether to send `tool_stream: true` (zai-specific) |

`detectCompat` auto-detects many of these. For example, xAI / Grok is identified by `provider === "xai"` or `baseUrl.includes("api.x.ai")`, and so `supportsReasoningEffort` is set to `false` for it. DeepSeek sets `thinkingFormat: "deepseek"` and `requiresReasoningContentOnAssistantMessages: true`.

### Step 2 — Building the OpenAI request

`buildParams` constructs the `ChatCompletionCreateParamsStreaming` object. Let's walk through the key decisions.

**Token limit field.** The OpenAI spec uses `max_completion_tokens`; some providers (Moonshot, chutes.ai, Cloudflare AI Gateway, Together, NVIDIA NIM, ant-ling) use the older `max_tokens`:

```ts
if (options?.maxTokens) {
  if (compat.maxTokensField === "max_tokens") {
    (params as any).max_tokens = options.maxTokens;
  } else {
    params.max_completion_tokens = options.maxTokens;
  }
}
```

**Usage in streaming.** Most providers accept `stream_options: { include_usage: true }` to get token counts in the final chunk. This is skipped when `supportsUsageInStreaming` is `false`.

**Prompt cache key.** For OpenAI's own API, `prompt_cache_key` (the session ID) is sent whenever cache retention is not `"none"`. For providers that support long retention, `prompt_cache_retention: "24h"` is added.

**Thinking/reasoning.** The `thinkingFormat` compat flag drives a multi-branch switch that maps `reasoningEffort` to each provider's native field:

```ts
// Simplified view of thinking format dispatch in buildParams
if (compat.thinkingFormat === "deepseek" && model.reasoning) {
  params.thinking = { type: options?.reasoningEffort ? "enabled" : "disabled" };
  if (options?.reasoningEffort && compat.supportsReasoningEffort) {
    params.reasoning_effort = model.thinkingLevelMap?.[options.reasoningEffort] ?? options.reasoningEffort;
  }
} else if (compat.thinkingFormat === "openrouter" && model.reasoning) {
  if (options?.reasoningEffort) {
    params.reasoning = { effort: model.thinkingLevelMap?.[options.reasoningEffort] ?? options.reasoningEffort };
  } else if (model.thinkingLevelMap?.off !== null) {
    params.reasoning = { effort: model.thinkingLevelMap?.off ?? "none" };
  }
} else if (options?.reasoningEffort && model.reasoning && compat.supportsReasoningEffort) {
  // Standard OpenAI reasoning_effort
  params.reasoning_effort = model.thinkingLevelMap?.[options.reasoningEffort] ?? options.reasoningEffort;
}
```

**Anthropic cache control via OpenAI endpoint.** Some providers (e.g., OpenRouter when routing to Anthropic models) accept Anthropic-style `cache_control` on OpenAI-format messages. When `cacheControlFormat === "anthropic"`, `applyAnthropicCacheControl` stamps `cache_control` onto:
- The system/developer message.
- The last tool definition.
- The last user or assistant message in the conversation.

### Step 3 — Converting messages for OpenAI

`convertMessages` in the OpenAI adapter (S12) transforms our generic `Message[]` into `ChatCompletionMessageParam[]`. It calls `transformMessages` (S16) first to handle cross-provider concerns (image downgrade for non-vision models, thinking block conversion, orphaned tool call cleanup, tool call ID normalisation).

After `transformMessages`, the adapter handles role-specific translation:

**System prompt.** If `model.reasoning && compat.supportsDeveloperRole`, the system prompt gets the `"developer"` role instead of `"system"`. This matters for OpenAI's reasoning models and some OpenRouter-routed Anthropic models.

**Assistant messages.** There is a subtle issue: OpenAI's Chat Completions API expects `content` to be a plain string, not an array of `{type:"text",text:"..."}` objects. Some models (e.g., DeepSeek V3.2 via NVIDIA NIM) literally mirror the content-block structure in their output if you send it as an array, producing recursive nesting. So the adapter always converts assistant text to a plain string:

```ts
// Always send assistant content as a plain string
if (assistantText.length > 0) {
  assistantMsg.content = assistantText; // joined plain string
}
```

**Thinking blocks in assistant messages.** When `requiresThinkingAsText` is true, thinking blocks are converted to plain text without tags (to avoid the model mimicking the tags). When it is false, the thinking text is placed in a provider-specific field on the assistant message (e.g., `reasoning_content`, `reasoning`) using the `thinkingSignature` stored on the `ThinkingContent` block.

**Tool results.** The adapter groups consecutive `toolResult` messages into individual `"tool"` role messages. When `requiresAssistantAfterToolResult` is true, it inserts a synthetic `"assistant"` turn between tool results and the next user message. Image content in tool results is forwarded as a separate user message with `image_url` parts (only when the model supports image input).

**Tool call ID normalisation.** OpenAI Responses API generates IDs that can be 450+ characters containing `|` separators. `normalizeToolCallId` in the OpenAI adapter extracts the `call_id` portion (before the `|`) and sanitises it to `^[a-zA-Z0-9_-]+$`, max 40 characters.

### Step 4 — Delta reassembly: parsing the OpenAI stream

Where Anthropic sends self-contained events (each `content_block_start` tells you the block's full type), OpenAI sends incremental `delta` objects that you accumulate into full blocks. The adapter maintains explicit "block trackers":

```ts
// Lazy initialisation helpers inside the streaming loop (simplified view)
const ensureTextBlock = () => {
  if (!textBlock) {
    textBlock = { type: "text", text: "" };
    blocks.push(textBlock);
    stream.push({ type: "text_start", contentIndex: getContentIndex(textBlock), partial: output });
  }
  return textBlock;
};

const ensureThinkingBlock = (thinkingSignature: string) => {
  if (!thinkingBlock) {
    thinkingBlock = { type: "thinking", thinking: "", thinkingSignature };
    blocks.push(thinkingBlock);
    stream.push({ type: "thinking_start", contentIndex: getContentIndex(thinkingBlock), partial: output });
  }
  return thinkingBlock;
};

const ensureToolCallBlock = (toolCall: StreamingToolCallDelta) => {
  // Look up by stream index first, then by ID
  let block = toolCallBlocksByIndex.get(toolCall.index);
  if (!block && toolCall.id) block = toolCallBlocksById.get(toolCall.id);
  if (!block) {
    block = { type: "toolCall", id: toolCall.id || "", name: toolCall.function?.name || "",
              arguments: {}, partialArgs: "", streamIndex: toolCall.index };
    // register by both index and id
    stream.push({ type: "toolcall_start", ... });
  }
  return block;
};
```

Text deltas arrive in `choice.delta.content`. Thinking/reasoning content can arrive in `reasoning_content`, `reasoning`, or `reasoning_text` — the adapter scans these fields in order and uses the first non-empty one to avoid duplication (some providers like chutes.ai return the same content in multiple fields). Tool call argument fragments arrive in `choice.delta.tool_calls[*].function.arguments` and are accumulated in `partialArgs`.

At the end of the stream, `finishBlock` is called for every tracked block, finalising `partialArgs` into `arguments` via `parseStreamingJson` and emitting the `_end` events.

**Finish reason validation.** Unlike Anthropic (which sends an explicit `message_stop` event), the OpenAI adapter relies on `finish_reason` in a chunk's `choice`. If the stream ends without any `finish_reason`, the adapter throws:

```ts
if (!hasFinishReason) {
  throw new Error("Stream ended without finish_reason");
}
```

Stop reason mapping for OpenAI:

| OpenAI `finish_reason` | Internal `StopReason` |
|---|---|
| `"stop"`, `"end"` | `"stop"` |
| `"length"` | `"length"` |
| `"function_call"`, `"tool_calls"` | `"toolUse"` |
| `"content_filter"` | `"error"` |
| `"network_error"` | `"error"` |
| anything else | `"error"` |

### Step 5 — Token usage for OpenAI

Usage arrives in the final streaming chunk's `chunk.usage` field (or `choice.usage` for non-standard providers like Moonshot). The `parseChunkUsage` function normalises the OpenAI usage schema:

```ts
function parseChunkUsage(rawUsage, model): AssistantMessage["usage"] {
  const promptTokens = rawUsage.prompt_tokens || 0;
  // cached_tokens is cache-read hits; cache_write_tokens is the separate write count
  const cacheReadTokens = rawUsage.prompt_tokens_details?.cached_tokens
                       ?? rawUsage.prompt_cache_hit_tokens ?? 0;
  const cacheWriteTokens = rawUsage.prompt_tokens_details?.cache_write_tokens || 0;
  // input = prompt minus any cached portions
  const input = Math.max(0, promptTokens - cacheReadTokens - cacheWriteTokens);
  // completion_tokens already includes reasoning_tokens
  const outputTokens = rawUsage.completion_tokens || 0;
  return { input, output: outputTokens, cacheRead: cacheReadTokens, cacheWrite: cacheWriteTokens,
           totalTokens: input + outputTokens + cacheReadTokens + cacheWriteTokens, ... };
}
```

The key subtlety: `cached_tokens` in `prompt_tokens_details` is a cache-**read** count (hits). OpenAI does not emit `cache_write_tokens` itself, but OpenRouter-compatible providers can include it. We must not subtract writes from cached reads, or spec-compliant providers are under-reported.

---

## How `transformMessages` bridges the two adapters (S16)

Both adapters call `transformMessages` before building their provider-specific request. This shared helper handles three cross-provider concerns:

**1. Image downgrade.** If the model's `input` array does not include `"image"`, all image content blocks in user messages and tool results are replaced with placeholder text:
- User images: `"(image omitted: model does not support images)"`
- Tool result images: `"(tool image omitted: model does not support images)"`

**2. Thinking block handling on cross-model turns.** When an assistant message was generated by a *different* model than the one we are calling now (`isSameModel` is false):
- Redacted thinking blocks are dropped (they are model-specific encrypted payloads).
- Regular thinking blocks are converted to plain `TextContent`.
- Tool call thought signatures (`thoughtSignature`) are removed.
- Tool call IDs are normalised via the `normalizeToolCallId` callback (each adapter passes its own normaliser).

When `isSameModel` is true, thinking blocks and signatures are preserved as-is for multi-turn continuity.

**3. Orphaned tool call recovery.** If a conversation contains an assistant turn with tool calls but no corresponding tool results (e.g., from an aborted turn), `transformMessages` inserts synthetic `toolResult` messages with `content: [{ type: "text", text: "No result provided" }]` and `isError: true`. This prevents API errors from providers that require every tool call to have a result.

**4. Errored assistant messages are skipped.** Assistant messages with `stopReason === "error"` or `stopReason === "aborted"` are filtered out entirely. Replaying a partial response can cause API errors (e.g., OpenAI rejects a reasoning block not followed by a valid continuation).

---

## Side-by-side comparison

| Concern | Anthropic SSE | OpenAI Chat Completions |
|---|---|---|
| Wire format | SSE with named Anthropic events | SSE with JSON delta chunks |
| Text content | `content_block_start` → `text_delta` → `content_block_stop` | Accumulate `choice.delta.content` |
| Tool arguments | `input_json_delta` accumulated in `partialJson` | `function.arguments` delta accumulated in `partialArgs` |
| Tool call identity | Present in `content_block_start` | May arrive in separate chunks; tracked by `index` and `id` |
| Thinking support | Native `thinking` / `redacted_thinking` blocks with `signature_delta` | Provider-specific: `reasoning_content`, `reasoning`, `reasoning_text`, or a format flag |
| Cache control | `{ type: "ephemeral", ttl?: "1h" }` on system, last user message, last tool | Same shape via `cacheControlFormat: "anthropic"`, or native OpenAI prompt cache key |
| Stop signal | Explicit `message_stop` event; missing it is an error | `finish_reason` in a chunk's `choice`; missing it is an error |
| Usage timing | Split: counts in `message_start`, updated in `message_delta` | Single final chunk `chunk.usage` (or `choice.usage` for some) |
| Provider quirks | Handled via `AnthropicMessagesCompat` flags per model | Handled via `OpenAICompletionsCompat` auto-detected from provider/URL |
| Token limit field | `max_tokens` | `max_completion_tokens` or `max_tokens` (compat flag) |

---

## What we have built

We now have two adapters that do the full translation:

- **`streamAnthropic`** — takes a `Context`, builds an Anthropic request (with cache control, thinking configuration, and tool normalisation), parses the SSE stream through three layers of decoders (byte → SSE event → typed Anthropic event → `EventStream` push), handles thinking blocks including redacted payloads, and maps stop reasons.
- **`streamOpenAICompletions`** — does the same for the OpenAI Chat Completions format, and scales to many providers via a compat-flag system that auto-detects quirks from provider names and `baseUrl` patterns.

Both adapters share the `transformMessages` pre-processing step that handles cross-model image downgrade, thinking block conversion, and orphaned tool call recovery.

The next chapter extends this pattern to two more adapters: Google Gemini and the faux test provider.

---

← Previous: [The EventStream Observable Backbone](./event-stream-and-observable-backbone.md) · Next: [Provider Adapters: Google Gemini and the Faux Test Provider](./provider-adapters-google-and-faux.md) →
