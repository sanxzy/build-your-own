---
title: "Message Types and the Unified Streaming API"
description: "Learn the core data model — Provider, StreamOptions, message and content-block shapes — and the four entry points all providers share."
category: llm-toolkit
type: tutorial
tags:
  - types
  - message
  - content block
  - StreamOptions
  - KnownApi
  - Provider
  - stream
  - complete
  - streamSimple
  - completeSimple
  - unified API
  - llm-toolkit
  - streaming
  - TypeScript
  - tool call
  - thinking block
  - image block
  - KnownProvider
  - Context
  - AssistantMessage
  - UserMessage
  - ToolResultMessage
  - TextContent
  - ThinkingContent
  - ImageContent
  - ToolCall
  - StopReason
  - SimpleStreamOptions
  - ThinkingLevel
  - AssistantMessageEventStream
keywords:
  - LLM abstraction layer
  - unified provider API
  - multi-provider streaming
  - provider-agnostic LLM
  - streaming assistant messages
  - tool calling TypeScript
  - model content blocks
sources: [S5, S7, S8]
---

**TL;DR** — Every LLM provider speaks a different wire format. The `llm-toolkit` package gives you one data model and four entry points that work the same regardless of which provider sits underneath. By the end of this chapter you will understand the message and content-block types, know how `StreamOptions` controls a call, and be able to use all four entry points — from the one-liner `completeSimple` up to the full event-by-event `stream`.

# Message Types and the Unified Streaming API

## The problem: every provider is different

If you call Anthropic directly, you construct one kind of request object. If you call Google, you construct another. If you want to switch models mid-conversation, or fall back to a different provider, you have to translate the entire conversation history each time.

The `llm-toolkit` package solves this by defining a single data model that every provider adapter converts _to_ and _from_. Your application code always works with the same shapes. The adapters handle the provider-specific details invisibly.

Let's start by understanding that data model, because everything else in the library builds on it.

## The `Provider` and `KnownApi` concepts

Before we look at messages, we need two orientation concepts: **provider** and **API**.

A **provider** is a named service — `"anthropic"`, `"openai"`, `"google"`, and so on. The full list of first-party providers is captured in the `KnownProvider` union type:

```ts
// Simplified view — the actual union has ~25 members
export type KnownProvider =
  | "amazon-bedrock"
  | "anthropic"
  | "google"
  | "google-vertex"
  | "openai"
  | "azure-openai-responses"
  | "groq"
  | "mistral"
  | "openrouter"
  // ... and more
  ;

export type Provider = KnownProvider | string;
```

The `| string` at the end means you can also pass a custom identifier for any provider you register yourself — the type stays ergonomic without becoming a closed enum.

A **KnownApi** is the _protocol_ that sits underneath a provider. Several providers can share one API. For example, `"groq"`, `"cerebras"`, and `"xai"` all implement the OpenAI Chat Completions protocol, so they all use the `"openai-completions"` API identifier:

```ts
export type KnownApi =
  | "openai-completions"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-vertex"
  | "mistral-conversations";

export type Api = KnownApi | (string & {});
```

When you call `getModel('groq', 'some-model-id')`, the returned `Model` object's `api` field will be `"openai-completions"`. The entry points use that `api` value to look up the right adapter at runtime.

## The message model

Now let's look at what a conversation looks like. A conversation is a sequence of `Message` objects:

```ts
export type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

There are three roles, and each has its own shape. Let's walk through them one at a time.

### `UserMessage` — what the human sends

```ts
export interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number; // Unix timestamp in milliseconds
}
```

The `content` field accepts either a plain string (for text-only messages) or an array of content blocks. The array form lets you mix text and images in the same message. We will look at the block types in a moment.

### `AssistantMessage` — what the model responds with

```ts
export interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;
  responseModel?: string;
  responseId?: string;
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number; // Unix timestamp in milliseconds
}
```

Notice that `AssistantMessage` carries provenance — `api`, `provider`, and `model` — so you always know which service produced a given turn. The `usage` field tracks token counts and cost, and `stopReason` tells you _why_ generation ended.

The `stopReason` type covers every outcome you need to handle:

```ts
export type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";
```

| `stopReason` | When it appears |
|---|---|
| `"stop"` | Model finished its response normally |
| `"length"` | Generation hit the max-token limit |
| `"toolUse"` | Model invoked a tool and expects results back |
| `"error"` | A runtime or provider error interrupted the response |
| `"aborted"` | The caller cancelled the request via `AbortSignal` |

### `ToolResultMessage` — the result you send back after a tool call

```ts
export interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number; // Unix timestamp in milliseconds
}
```

When the model returns `stopReason: "toolUse"`, you execute the tool and push a `ToolResultMessage` into the conversation. The `toolCallId` links the result back to the specific call that triggered it. `isError: true` signals to the model that execution failed, which it can use to retry or adapt its response. The `content` array supports both text and images, so you can return screenshots or chart images from a tool.

## Content blocks

Both `UserMessage` and `AssistantMessage` carry arrays of **content blocks** — typed objects that represent different kinds of content. There are four block types in total:

| Block type | TypeScript interface | `type` literal | Who uses it |
|---|---|---|---|
| Plain text | `TextContent` | `"text"` | User messages, assistant responses |
| Model thinking | `ThinkingContent` | `"thinking"` | Assistant responses (reasoning-capable models) |
| Base64 image | `ImageContent` | `"image"` | User messages, tool results |
| Tool invocation | `ToolCall` | `"toolCall"` | Assistant responses |

Let's look at each shape:

```ts
export interface TextContent {
  type: "text";
  text: string;
  textSignature?: string;
}

export interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;
}

export interface ImageContent {
  type: "image";
  data: string;    // base64-encoded image bytes
  mimeType: string; // e.g. "image/png", "image/jpeg"
}

export interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;
}
```

A few things worth noticing here:

- `ThinkingContent` carries a `redacted` flag. When `true`, the provider's safety filters have hidden the thinking text; the opaque `thinkingSignature` is preserved so you can pass it back to the API for multi-turn continuity.
- `ToolCall.arguments` is always a `Record<string, any>` — a plain object — even when arguments are still streaming in partially. The library never leaves it `undefined`.
- `ImageContent.data` is base64-encoded bytes, and `mimeType` tells you the format.

## The `Context` object

With message types in hand, we can look at `Context` — the complete input you pass to any entry point:

```ts
export interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

That is the entire input surface. A system prompt, the conversation history, and an optional list of tools. There are no provider-specific fields here — the adapters handle translation. `Context` is also fully JSON-serialisable, which makes it straightforward to persist conversations to disk or a database and resume them later.

## `StreamOptions` — controlling a call

Every entry point accepts an optional `StreamOptions` object that lets you tune the request:

```ts
export interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: Transport;          // "sse" | "websocket" | "websocket-cached" | "auto"
  cacheRetention?: CacheRetention; // "none" | "short" | "long" — default: "short"
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | undefined | Promise<unknown | undefined>;
  onResponse?: (response: ProviderResponse, model: Model<Api>) => void | Promise<void>;
  headers?: Record<string, string>;
  timeoutMs?: number;
  websocketConnectTimeoutMs?: number;
  maxRetries?: number;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}
```

Key fields to know for everyday use:

| Field | Purpose |
|---|---|
| `apiKey` | Explicit API key. If omitted, the library reads the matching `XZY_*` environment variable automatically. |
| `signal` | An `AbortSignal` to cancel the request mid-stream. |
| `maxTokens` | Cap on generated tokens. |
| `temperature` | Sampling temperature. |
| `cacheRetention` | Prompt cache preference: `"none"`, `"short"` (default), or `"long"`. |
| `sessionId` | Enables session-based caching on providers that support it. |
| `onPayload` | Callback to inspect or mutate the raw request payload before it is sent. Useful for debugging. |
| `maxRetries` | How many times to retry on transient failures. |
| `maxRetryDelayMs` | Cap on retry wait time (default: 60 000 ms). |

Providers that do not support a given option silently ignore it, so you can set `cacheRetention` or `sessionId` without worrying about which providers support them.

## The simplified option type: `SimpleStreamOptions`

For the two simplified entry points (`streamSimple` and `completeSimple`), there is an extended options type that adds a provider-agnostic reasoning control:

```ts
export interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;
  thinkingBudgets?: ThinkingBudgets;
}

export type ThinkingLevel = "minimal" | "low" | "medium" | "high" | "xhigh";
```

Instead of passing provider-specific flags like `thinkingEnabled: true` or `reasoningEffort: "medium"`, you pass a single `ThinkingLevel` string. The library translates it to whatever each provider expects.

## The four entry points

Now we have the data model. Let's walk the four entry points, from the simplest convenience wrapper to the fully-controllable streaming call.

### Entry point 1: `completeSimple` — one-liner completions

`completeSimple` is the best starting point. You call it with a model, a context, and optionally a `SimpleStreamOptions` object. It returns a `Promise<AssistantMessage>`.

```ts
export async function completeSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): Promise<AssistantMessage>
```

Here is a complete example — from import to reading the response:

```ts
import { getModel, completeSimple } from "llm-toolkit";

const model = getModel("openai", "gpt-4o-mini");

const response = await completeSimple(model, {
  systemPrompt: "You are a concise assistant.",
  messages: [
    { role: "user", content: "What is 7 × 8?", timestamp: Date.now() }
  ]
});

// response is an AssistantMessage
for (const block of response.content) {
  if (block.type === "text") {
    console.log(block.text); // "56"
  }
}

console.log(`Stop reason: ${response.stopReason}`);
console.log(`Tokens: ${response.usage.input} in, ${response.usage.output} out`);
console.log(`Cost: $${response.usage.cost.total.toFixed(6)}`);
```

The API key is read from the environment automatically (the library looks for the matching variable for `"openai"`). If you need an explicit key, pass `{ apiKey: "..." }` as the third argument.

To enable reasoning on a model that supports it, pass the `reasoning` option:

```ts
const model = getModel("anthropic", "claude-sonnet-4-20250514");

const response = await completeSimple(model, {
  messages: [{ role: "user", content: "Solve: 2x + 5 = 13", timestamp: Date.now() }]
}, {
  reasoning: "medium"  // "minimal" | "low" | "medium" | "high" | "xhigh"
});

for (const block of response.content) {
  if (block.type === "thinking") {
    console.log("Thinking:", block.thinking);
  } else if (block.type === "text") {
    console.log("Answer:", block.text);
  }
}
```

### Entry point 2: `complete` — non-streaming with provider-specific options

`complete` works like `completeSimple` but accepts `ProviderStreamOptions` — a superset of `StreamOptions` that lets you pass provider-specific flags directly:

```ts
export async function complete<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): Promise<AssistantMessage>
```

Internally, `complete` calls `stream` and waits for the result:

```ts
// Simplified view of complete()
export async function complete(model, context, options) {
  const s = stream(model, context, options);
  return s.result();
}
```

You would reach for `complete` when you need a provider-specific option that `SimpleStreamOptions` does not expose, for example the Anthropic `thinkingBudgetTokens` or OpenAI `reasoningEffort`:

```ts
import { getModel, complete } from "llm-toolkit";

const model = getModel("anthropic", "claude-sonnet-4-20250514");

const response = await complete(model, {
  messages: [{ role: "user", content: "Explain quantum entanglement.", timestamp: Date.now() }]
}, {
  thinkingEnabled: true,
  thinkingBudgetTokens: 4096
});
```

### Entry point 3: `streamSimple` — streaming with unified reasoning option

`streamSimple` gives you a live event stream while keeping the unified `SimpleStreamOptions` interface:

```ts
export function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream
```

It returns an `AssistantMessageEventStream` — an async iterable you loop over with `for await`. Let's walk through streaming a response and printing text deltas as they arrive:

```ts
import { getModel, streamSimple } from "llm-toolkit";

const model = getModel("google", "gemini-2.5-flash");

const s = streamSimple(model, {
  messages: [{ role: "user", content: "Count from 1 to 5.", timestamp: Date.now() }]
}, {
  reasoning: "low"
});

for await (const event of s) {
  switch (event.type) {
    case "thinking_start":
      process.stdout.write("[thinking] ");
      break;
    case "thinking_delta":
      process.stdout.write(event.delta);
      break;
    case "thinking_end":
      process.stdout.write("\n");
      break;
    case "text_delta":
      process.stdout.write(event.delta);
      break;
    case "done":
      console.log(`\nDone. Stop reason: ${event.reason}`);
      break;
    case "error":
      console.error(`Error (${event.reason}):`, event.error.errorMessage);
      break;
  }
}

// Retrieve the complete AssistantMessage after the loop
const finalMessage = await s.result();
console.log(`Total cost: $${finalMessage.usage.cost.total.toFixed(6)}`);
```

Notice `s.result()` — you call this _after_ the loop to get the fully assembled `AssistantMessage`. The stream collects all events and builds the final message for you; you do not have to reconstruct it yourself.

### Entry point 4: `stream` — full streaming with provider-specific options

`stream` is the most capable entry point. It has the same signature as `streamSimple` but accepts `ProviderStreamOptions` instead of `SimpleStreamOptions`:

```ts
export function stream<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions,
): AssistantMessageEventStream
```

This is the function to use in an agent loop, where you need to handle tool calls, observe every event type, and control the conversation turn-by-turn. Here is a complete example that handles both text output and a single tool call:

```ts
import { getModel, stream, type Context } from "llm-toolkit";

const model = getModel("openai", "gpt-4o-mini");

const context: Context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    { role: "user", content: "What time is it in UTC?", timestamp: Date.now() }
  ],
  tools: [{
    name: "get_time",
    description: "Returns the current UTC time as an ISO 8601 string.",
    parameters: {
      type: "object",
      properties: {},
      required: []
    }
  }]
};

const s = stream(model, context);

for await (const event of s) {
  switch (event.type) {
    case "start":
      // event.partial is an AssistantMessage in its initial (empty) state
      break;
    case "text_delta":
      process.stdout.write(event.delta);
      break;
    case "toolcall_start":
      console.log(`\n[Tool call starting at content index ${event.contentIndex}]`);
      break;
    case "toolcall_delta":
      // event.partial.content[event.contentIndex] is a ToolCall with partial arguments
      // Arguments may be incomplete — check before accessing nested fields
      break;
    case "toolcall_end":
      console.log(`\nTool called: ${event.toolCall.name}`);
      console.log(`Arguments: ${JSON.stringify(event.toolCall.arguments)}`);
      break;
    case "done":
      console.log(`\nFinished. Stop reason: ${event.reason}`);
      break;
    case "error":
      console.error(`Error (${event.reason}):`, event.error.errorMessage);
      break;
  }
}

// Collect the final message and push it to the conversation
const finalMessage = await s.result();
context.messages.push(finalMessage);

// If the model called a tool, respond with the result
const toolCalls = finalMessage.content.filter(b => b.type === "toolCall");
for (const call of toolCalls) {
  const result = new Date().toISOString(); // Execute the actual tool here

  context.messages.push({
    role: "toolResult",
    toolCallId: call.id,
    toolName: call.name,
    content: [{ type: "text", text: result }],
    isError: false,
    timestamp: Date.now()
  });
}

// If there were tool calls, continue the conversation
if (toolCalls.length > 0) {
  const continuation = await stream(model, context);
  for await (const event of continuation) {
    if (event.type === "text_delta") process.stdout.write(event.delta);
  }
}
```

## The complete event reference

Every `stream` and `streamSimple` call emits events from the `AssistantMessageEvent` union. Here is the full set:

| Event `type` | Key fields | When it fires |
|---|---|---|
| `start` | `partial`: initial `AssistantMessage` | Before any content arrives |
| `text_start` | `contentIndex` | A new text block begins |
| `text_delta` | `delta`, `contentIndex` | A chunk of text arrives |
| `text_end` | `content` (full text), `contentIndex` | The text block is complete |
| `thinking_start` | `contentIndex` | A new thinking block begins |
| `thinking_delta` | `delta`, `contentIndex` | A chunk of thinking arrives |
| `thinking_end` | `content` (full thinking), `contentIndex` | The thinking block is complete |
| `toolcall_start` | `contentIndex` | A tool call begins |
| `toolcall_delta` | `delta`, `contentIndex`, `partial` | Tool arguments streaming in |
| `toolcall_end` | `toolCall` (complete `ToolCall`), `contentIndex` | The tool call is complete |
| `done` | `reason` (`"stop"` \| `"length"` \| `"toolUse"`), `message` | Stream ended successfully |
| `error` | `reason` (`"error"` \| `"aborted"`), `error` (partial `AssistantMessage`) | Stream ended with an error |

One important behaviour to know: **streaming events for different content blocks are not guaranteed to arrive in contiguous sequences.** A provider may send a text delta, then a tool-call delta, then another text delta, all before either block has finished. Always use `contentIndex` to map deltas and end-events to their block — never assume that `text_start`→`text_delta`→`text_end` will be uninterrupted by events for other blocks.

## Comparing the four entry points

| Entry point | Returns | Options type | Best for |
|---|---|---|---|
| `completeSimple` | `Promise<AssistantMessage>` | `SimpleStreamOptions` | Quick completions, unified reasoning control |
| `complete` | `Promise<AssistantMessage>` | `ProviderStreamOptions` | Non-streaming with provider-specific flags |
| `streamSimple` | `AssistantMessageEventStream` | `SimpleStreamOptions` | Streaming with unified reasoning control |
| `stream` | `AssistantMessageEventStream` | `ProviderStreamOptions` | Full streaming, agent loops, tool handling |

## How API key injection works

You may have noticed that none of the examples pass an API key explicitly. That is because `stream.ts` injects it automatically. When a call comes in, the library checks whether an explicit `apiKey` was passed in options. If not, it calls `getEnvApiKey(model.provider)` to look up the matching environment variable for that provider:

```ts
// Simplified view of the injection logic
function withEnvApiKey(model, options) {
  if (options?.apiKey) return options;          // already set — use it
  const apiKey = getEnvApiKey(model.provider);  // look up e.g. OPENAI_API_KEY
  if (!apiKey) return options;                  // not found — pass through
  return { ...options, apiKey };
}
```

The environment variable names follow each provider's own convention (e.g. `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`). In browser environments where there are no environment variables, you must pass `apiKey` explicitly in the options object.

## Putting it all together

Let's look at a self-contained snippet that uses the full data model: a two-turn conversation that starts with a user message, gets an assistant response, and pushes the context forward:

```ts
import { getModel, stream, completeSimple, type Context } from "llm-toolkit";

async function runConversation() {
  const model = getModel("anthropic", "claude-sonnet-4-20250514");

  const context: Context = {
    systemPrompt: "Answer briefly.",
    messages: []
  };

  // Turn 1 — use completeSimple for a quick first response
  context.messages.push({
    role: "user",
    content: "Name one programming language.",
    timestamp: Date.now()
  });

  const turn1 = await completeSimple(model, context);
  context.messages.push(turn1);

  // turn1.content is (TextContent | ThinkingContent | ToolCall)[]
  for (const block of turn1.content) {
    if (block.type === "text") console.log("Assistant:", block.text);
  }

  // Turn 2 — use stream to observe events live
  context.messages.push({
    role: "user",
    content: "Why do you like it?",
    timestamp: Date.now()
  });

  const s = stream(model, context);
  process.stdout.write("Assistant: ");
  for await (const event of s) {
    if (event.type === "text_delta") process.stdout.write(event.delta);
  }
  console.log();

  const turn2 = await s.result();
  context.messages.push(turn2);

  console.log(`Conversation cost so far: $${(
    turn1.usage.cost.total + turn2.usage.cost.total
  ).toFixed(6)}`);
}

runConversation();
```

## What comes next

We have covered the data model and the front door of `llm-toolkit`. The next chapter dives into the `AssistantMessageEventStream` itself — the observable backbone that makes `stream` and `streamSimple` work — and shows you how to compose and transform event streams.

---
← Previous: [Setting Up the Workspace and Toolchain](../getting-started/workspace-and-toolchain.md) · Next: [The EventStream Observable Backbone](./event-stream-and-observable-backbone.md) →
