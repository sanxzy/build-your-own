---
title: "Message Types and the Core Streaming API"
description: "Define the complete type vocabulary every provider shares — Message shapes, content blocks, tools, and the Context — then build the streaming entry points that all provider adapters implement."
category: llm-toolkit
type: tutorial
tags: [types, Message, AssistantMessage, UserMessage, ToolResultMessage, content blocks, text, toolCall, thinking, image, StreamOptions, Provider, KnownApi, streamSimple, completeSimple, TypeScript, typebox, unified API]
keywords: [message types, streaming API, content blocks, tool definition, Context, StreamFunction, type vocabulary]
sources: [S5, S8]
---

**TL;DR** — Before we can talk to any LLM, we need a shared language. We'll define a unified type system — `Message`, `AssistantMessage`, `UserMessage`, `ToolResultMessage`, content blocks, `Tool`, and `Context` — that abstracts over every provider's native format. Then we'll build the two streaming entry points (`streamSimple` and `completeSimple`) that all provider adapters implement, and wire them through an API registry lookup. By the end, you'll have a working `streamSimple()` call that can stream from any registered provider through a single interface.

## The problem: every provider speaks a different language

Open a connection to the Anthropic API and you'll send JSON that looks like this:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [
    { "role": "user", "content": "Hello!" }
  ]
}
```

Open a connection to the OpenAI Responses API and the shape is different:

```json
{
  "model": "gpt-5",
  "input": [
    { "role": "user", "content": "Hello!" }
  ],
  "tools": [...]
}
```

Google Gemini uses yet another format. And each provider has its own way of representing tool calls, thinking blocks, streaming events, error conditions, and token usage.

If we wrote agent code directly against one provider's API, we'd be locked in. Switching providers would mean rewriting every integration point. What we need is a **unified type system** — one set of types that captures everything an LLM interaction can produce, with provider-specific adapters that translate to and from the native formats.

## The unified message types

We'll use three message types to represent every turn in a conversation. They're defined in `packages/llm-toolkit/src/types.ts`.

### `UserMessage` — what the user (or system) sends

```ts
export interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}
```

A user message is what kicks off a turn. The `content` can be a plain string (for simple text messages) or an array of content blocks (for messages with images). Every message carries a `timestamp` — a Unix timestamp in milliseconds — so we can track when each turn happened.

The content blocks that can appear in a user message:

```ts
export interface TextContent {
  type: "text";
  text: string;
  textSignature?: string;
}

export interface ImageContent {
  type: "image";
  data: string;      // base64 encoded image data
  mimeType: string;  // e.g., "image/jpeg", "image/png"
}
```

`textSignature` is an optional field that some providers use to attach metadata to specific text blocks (for example, OpenAI uses it for message-level identifiers). Provider adapters populate it when the upstream API provides one; the rest of the system can ignore it.

### `AssistantMessage` — what the LLM sends back

```ts
export interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;
  responseModel?: string;
  responseId?: string;
  diagnostics?: AssistantMessageDiagnostic[];
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number;
}
```

The assistant message is the richest type. Let's walk through each field:

**`content`** — an ordered array of content blocks. The LLM might respond with text, show its thinking, or request tool calls. The three content block types:

```ts
// TextContent — already defined above, reused here

export interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;  // true when safety filters redacted the thinking
}

export interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;  // Google-specific opaque signature
}
```

The `ThinkingContent` block represents the model's internal reasoning. Some providers (Anthropic with extended thinking, OpenAI with reasoning) expose what the model "thought" before producing its final answer. The `redacted` flag signals that safety filters stripped the thinking content — the encrypted payload is preserved in `thinkingSignature` so it can be passed back for multi-turn continuity, but the plain text is gone.

The `ToolCall` block represents a request to execute a tool. It has a unique `id` (the agent uses this to match results back to calls), the tool's `name`, and the `arguments` the model wants to pass. The arguments are a plain `Record<string, any>` — the schema validation happens at a different layer.

**`api` and `provider`** — these track which API and which provider produced this message:

```ts
export type KnownApi =
  | "openai-completions" | "openai-responses" | "azure-openai-responses"
  | "openai-codex-responses" | "anthropic-messages"
  | "bedrock-converse-stream" | "google-generative-ai" | "google-vertex"
  | "mistral-conversations";

export type Api = KnownApi | (string & {});

export type KnownProvider =
  | "anthropic" | "openai" | "google" | "google-vertex"
  | "amazon-bedrock" | "azure-openai-responses" | "openai-codex"
  | "github-copilot" | "deepseek" | "xai" | "groq" | "openrouter"
  | "mistral" | "cerebras" | "vercel-ai-gateway" /* ... and more */;

export type Provider = KnownProvider | string;
```

Both are string union types with a `Known*` subset for the providers we ship first-party adapters for. The `(string & {})` suffix means any string is valid — custom providers aren't second-class citizens, they just don't get autocomplete for the known variants.

**`stopReason`** — why the LLM stopped generating:

```ts
export type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";
```

- `"stop"` — the model finished naturally (it said what it wanted to say)
- `"length"` — the model hit the max token limit mid-response
- `"toolUse"` — the model stopped because it wants to execute tools
- `"error"` — something went wrong (check `errorMessage`)
- `"aborted"` — the caller cancelled the request

**`usage`** — token counts and cost:

```ts
export interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

Every assistant message carries its own usage data. This lets us track cumulative costs across a session, make compaction decisions based on token counts, and display usage to the user.

### `ToolResultMessage` — the result of executing a tool

```ts
export interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number;
}
```

When the agent executes a tool, it creates a `ToolResultMessage`. The `toolCallId` matches the `id` from the `ToolCall` that requested it — this is how the LLM knows which result corresponds to which call. The `content` array holds the tool's output (text, images, or both). The `isError` flag tells the LLM whether the tool succeeded or failed. The optional `details` field carries structured data that the agent harness can use without parsing the content text.

### The union type

```ts
export type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

Every turn in a conversation is one of these three. A conversation is an ordered array of `Message[]`.

## The Tool type

Tools give the LLM the ability to act. Each tool is defined by a name, a description, and a JSON Schema for its parameters:

```ts
export interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

The `parameters` field uses TypeBox schemas (`TSchema`) — a TypeScript-first JSON Schema library. This gives us compile-time type checking of tool parameter schemas, not just runtime validation. When a tool call comes in with `arguments: { filePath: "/src/app.ts" }`, the agent harness validates those arguments against the schema before executing the tool.

## The Context — what we send to the LLM

A single `Context` object bundles everything the LLM needs for one turn:

```ts
export interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

- `systemPrompt` — the optional system-level instructions ("You are a coding agent. Be thorough."). This is sent once at the start and doesn't count as a conversation turn.
- `messages` — the ordered conversation history. Every `UserMessage`, `AssistantMessage`, and `ToolResultMessage` from previous turns.
- `tools` — the tools available to the LLM for this turn. The agent can change the tool set between turns (for example, removing tools that don't apply to the current mode).

## Streaming options

The `StreamOptions` type captures everything that controls how a request behaves:

```ts
export interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: Transport;
  cacheRetention?: CacheRetention;
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | undefined;
  onResponse?: (response: ProviderResponse, model: Model<Api>) => void;
  headers?: Record<string, string>;
  timeoutMs?: number;
  maxRetries?: number;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}
```

And `SimpleStreamOptions` extends it with a convenience field:

```ts
export interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;  // "minimal" | "low" | "medium" | "high" | "xhigh"
  thinkingBudgets?: ThinkingBudgets;
}
```

The `reasoning` field is a unified thinking-level that the provider adapter maps to its native equivalent. Anthropic calls it "thinking budget", OpenAI calls it "reasoning effort", but our code just says `reasoning: "high"`.

## The stream function

With the types defined, the streaming API is straightforward. We'll write it in `packages/llm-toolkit/src/stream.ts`:

```ts
import { getApiProvider } from "./api-registry.ts";
import { getEnvApiKey } from "./env-api-keys.ts";
import type {
  Api, AssistantMessage, AssistantMessageEventStream,
  Context, Model, SimpleStreamOptions, StreamOptions,
} from "./types.ts";

function resolveApiProvider(api: Api) {
  const provider = getApiProvider(api);
  if (!provider) {
    throw new Error(`No API provider registered for api: ${api}`);
  }
  return provider;
}

function withEnvApiKey<TOptions extends StreamOptions>(
  model: Model<Api>,
  options: TOptions | undefined,
): TOptions | undefined {
  if (options?.apiKey) return options;
  const apiKey = getEnvApiKey(model.provider);
  if (!apiKey) return options;
  return { ...options, apiKey } as TOptions;
}

export function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);
  return provider.streamSimple(model, context, withEnvApiKey(model, options));
}

export async function completeSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): Promise<AssistantMessage> {
  const stream = streamSimple(model, context, options);
  return stream.result();
}
```

Let's trace what happens when you call `streamSimple()`:

1. **Provider resolution.** `resolveApiProvider(model.api)` looks up the provider adapter registered for this API type (e.g., `"anthropic-messages"`). If no adapter is registered, it throws — you'll get a clear error before any network call happens.

2. **API key resolution.** `withEnvApiKey()` checks if the caller provided an explicit `apiKey`. If not, it tries to resolve one from environment variables (e.g., `ANTHROPIC_API_KEY` for Anthropic, `OPENAI_API_KEY` for OpenAI). The environment variable conventions are defined in `env-api-keys.ts`.

3. **Delegation.** The resolved provider adapter's `streamSimple()` method is called. It receives the model, the context, and the (possibly enriched) options. What it returns is an `AssistantMessageEventStream`.

The `completeSimple()` variant is a convenience wrapper — it calls `streamSimple()`, waits for the stream to finish, and returns the final `AssistantMessage`. Use `streamSimple()` when you want to process events as they arrive (the agent loop does this). Use `completeSimple()` when you just want the final result.

## The event protocol

Every `AssistantMessageEventStream` emits a sequence of typed events. Here's the complete protocol:

```ts
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

Every event carries a `partial` — a snapshot of the `AssistantMessage` as it exists so far. The UI can render partial content (streaming text appears character by character) while the agent loop waits for the final `done` or `error` event before acting on tool calls.

The event sequence for a typical response that asks to run a tool:

```
start           → { type: "start", partial: { content: [], ... } }
text_start      → { type: "text_start", contentIndex: 0, partial: ... }
text_delta      → { type: "text_delta", contentIndex: 0, delta: "I'll", partial: ... }
text_delta      → { type: "text_delta", contentIndex: 0, delta: " read", partial: ... }
text_delta      → { type: "text_delta", contentIndex: 0, delta: " that", partial: ... }
text_end        → { type: "text_end", contentIndex: 0, content: "I'll read that file", partial: ... }
toolcall_start  → { type: "toolcall_start", contentIndex: 1, partial: ... }
toolcall_delta  → { type: "toolcall_delta", contentIndex: 1, delta: "...", partial: ... }
toolcall_end    → { type: "toolcall_end", contentIndex: 1, toolCall: {...}, partial: ... }
done            → { type: "done", reason: "toolUse", message: AssistantMessage }
```

Notice that `contentIndex` tracks which content block each event belongs to. The first text block is index 0, the tool call is index 1, and so on. This lets the UI render multiple parallel content streams correctly.

## What we've built

We now have:

- **Three message types** (`UserMessage`, `AssistantMessage`, `ToolResultMessage`) that capture every turn in a conversation
- **Content blocks** for text, thinking, tool calls, and images — the atoms that compose each message
- **`Tool`** — a typed tool definition with TypeBox schema parameters
- **`Context`** — the bundle of system prompt, messages, and tools sent to each LLM call
- **`streamSimple()` and `completeSimple()`** — the two entry points that resolve a provider adapter, enrich with env-based API keys, and delegate
- **The event protocol** — the typed event sequence every provider adapter emits

Right now, calling `streamSimple()` throws — we haven't registered any provider adapters yet. That's what we'll fix in the next three chapters, starting with Anthropic.

---

← Previous: [Setting Up the Workspace and Toolchain](../getting-started/workspace-and-toolchain.md) · Next: [The EventStream: Observable Backbone for Streaming](./eventstream-observable-backbone.md) →
