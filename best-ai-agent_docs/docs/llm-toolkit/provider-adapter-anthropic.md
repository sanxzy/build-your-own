---
title: "Provider Adapter: Anthropic Messages API"
description: "Implement the Anthropic provider adapter — SSE event parsing, prompt caching with cache_control, extended thinking block extraction, and tool call normalization into our unified event protocol."
category: llm-toolkit
type: tutorial
tags: [Anthropic, Messages API, SSE, server-sent events, cache-control, thinking blocks, extended thinking, tool normalization, prompt caching, provider adapter, llm-toolkit]
keywords: [Anthropic adapter, SSE parsing, cache_control, thinking extraction, tool use streaming, provider implementation]
sources: [S10, S16]
---

**TL;DR** — We'll build our first real provider adapter, connecting the Anthropic Messages API to our unified streaming interface. The adapter translates our `Context` into Anthropic's request format, parses SSE (Server-Sent Events) from the response stream, maps each Anthropic event to our typed `AssistantMessageEvent`, and handles prompt caching, extended thinking, and tool call streaming. By the end, `streamSimple({ provider: "anthropic", ... })` will produce live streaming responses.

## Install the Anthropic SDK

The adapter wraps the official `@anthropic-ai/sdk` package. Add it to the llm-toolkit package:

```bash
cd packages/llm-toolkit
npm install @anthropic-ai/sdk
```

## The adapter's job

A provider adapter has one function: implement the `StreamFunction` contract. It receives our unified types (`Model`, `Context`, `StreamOptions`) and returns an `AssistantMessageEventStream`. Everything in between is translation:

```
Our types                       Anthropic types
──────────                      ───────────────
Context                  ──→    MessageCreateParams
  .systemPrompt          ──→    system (string | array)
  .messages[]            ──→    messages[]
  .tools[]               ──→    tools[]

SSE response             ──→    AssistantMessageEventStream
  message_start          ──→    { type: "start" }
  content_block_start    ──→    { type: "text_start" | "thinking_start" | "toolcall_start" }
  content_block_delta    ──→    { type: "text_delta" | "thinking_delta" | "toolcall_delta" }
  content_block_stop     ──→    { type: "text_end" | "thinking_end" | "toolcall_end" }
  message_stop           ──→    { type: "done" }
  error                  ──→    { type: "error" }
```

## Building the request

Create `packages/llm-toolkit/src/providers/anthropic.ts`. Let's start with the request construction.

### System prompt

Anthropic accepts system prompts as either a top-level string or an array of content blocks (the array form enables cache control on system content):

```ts
function buildSystem(
  systemPrompt: string | undefined,
  cacheControl?: CacheControlEphemeral,
): MessageCreateParamsStreaming["system"] {
  if (!systemPrompt) return undefined;

  if (cacheControl) {
    return [{
      type: "text",
      text: systemPrompt,
      cache_control: cacheControl,
    }];
  }
  return systemPrompt;
}
```

### Messages

Converting our `Message[]` to Anthropic's `MessageParam[]` requires handling four message roles and transforming content blocks:

```ts
function toAnthropicMessage(msg: Message): MessageParam {
  switch (msg.role) {
    case "user": {
      const content = typeof msg.content === "string"
        ? msg.content
        : msg.content.map(block => {
            if (block.type === "text") return { type: "text", text: block.text };
            if (block.type === "image") return {
              type: "image",
              source: { type: "base64", media_type: block.mimeType as any, data: block.data },
            };
            throw new Error(`Unexpected user content block: ${block.type}`);
          });
      return { role: "user", content };
    }
    case "assistant": {
      const content = msg.content.map(block => {
        if (block.type === "text") return { type: "text", text: block.text };
        if (block.type === "thinking") return { type: "thinking", thinking: block.thinking, signature: block.thinkingSignature };
        if (block.type === "toolCall") return {
          type: "tool_use",
          id: block.id,
          name: block.name,
          input: block.arguments,
        };
        throw new Error(`Unexpected assistant content block: ${block.type}`);
      });
      return { role: "assistant", content };
    }
    case "toolResult": {
      return {
        role: "user",
        content: [{
          type: "tool_result",
          tool_use_id: msg.toolCallId,
          content: msg.content.map(b => {
            if (b.type === "text") return { type: "text", text: b.text };
            if (b.type === "image") return { type: "image", source: { type: "base64", media_type: b.mimeType, data: b.data } };
            throw new Error(`Unexpected tool result block: ${b.type}`);
          }),
          is_error: msg.isError,
        }],
      };
    }
  }
}
```

Notice that `ToolResultMessage` (role `"toolResult"` in our system) maps to Anthropic's `user` role with `tool_result` content blocks. This is a provider-specific detail — Anthropic requires tool results to be sent as user messages.

### Tools

Our `Tool[]` maps directly to Anthropic's tool format with one addition — we enable `eager_input_streaming` so tool call arguments stream incrementally:

```ts
function toAnthropicTools(tools: Tool[]): MessageCreateParamsStreaming["tools"] {
  return tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    input_schema: tool.parameters,
    eager_input_streaming: { type: "json" as const },
  }));
}
```

### Putting it together

The `streamAnthropic` function assembles the request and starts streaming:

```ts
export const streamAnthropic: StreamFunction<"anthropic-messages"> = (
  model, context, options,
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream();
  const client = new Anthropic({ apiKey: options?.apiKey });

  const params: MessageCreateParamsStreaming = {
    model: model.id,
    max_tokens: options?.maxTokens ?? 4096,
    system: buildSystem(context.systemPrompt, getCacheControl(model, options?.cacheRetention).cacheControl),
    messages: context.messages.map(toAnthropicMessage),
    tools: context.tools?.length ? toAnthropicTools(context.tools) : undefined,
    stream: true,
  };

  // ... SSE parsing and event mapping (next section)
  return stream;
};
```

## Parsing SSE and mapping events

Anthropic's streaming response is a sequence of SSE events. Each line starts with `event:` (the event type) followed by `data:` (the JSON payload). The Anthropic SDK gives us a typed async iterable of `RawMessageStreamEvent`, which we map to our event protocol:

```ts
(async () => {
  try {
    const anthropicStream = client.messages.stream(params);

    for await (const event of anthropicStream) {
      switch (event.type) {
        case "message_start":
          stream.push({
            type: "start",
            partial: buildPartialMessage(event.message, model),
          });
          break;

        case "content_block_start": {
          const block = event.content_block;
          if (block.type === "text") {
            stream.push({
              type: "text_start",
              contentIndex: event.index,
              partial: buildPartial(event, model),
            });
          } else if (block.type === "thinking") {
            stream.push({
              type: "thinking_start",
              contentIndex: event.index,
              partial: buildPartial(event, model),
            });
          } else if (block.type === "tool_use") {
            stream.push({
              type: "toolcall_start",
              contentIndex: event.index,
              partial: buildPartial(event, model),
            });
          }
          break;
        }

        case "content_block_delta": {
          const delta = event.delta;
          if (delta.type === "text_delta") {
            stream.push({
              type: "text_delta",
              contentIndex: event.index,
              delta: delta.text,
              partial: buildPartial(event, model),
            });
          } else if (delta.type === "thinking_delta") {
            stream.push({
              type: "thinking_delta",
              contentIndex: event.index,
              delta: delta.thinking,
              partial: buildPartial(event, model),
            });
          } else if (delta.type === "input_json_delta") {
            stream.push({
              type: "toolcall_delta",
              contentIndex: event.index,
              delta: delta.partial_json,
              partial: buildPartial(event, model),
            });
          }
          break;
        }

        case "content_block_stop":
          // Emit text_end / thinking_end / toolcall_end based on accumulated content
          emitContentEnd(event, stream, model);
          break;

        case "message_delta":
          // Update usage and stop reason
          updateMessageWithDelta(event);
          break;

        case "message_stop":
          stream.push({
            type: "done",
            reason: mapStopReason(finalMessage.stopReason),
            message: finalMessage,
          });
          break;
      }
    }
  } catch (err) {
    stream.push({
      type: "error",
      reason: "error",
      error: buildErrorMessage(err, model),
    });
  }
})();
```

## Prompt caching

Anthropic's prompt cache can dramatically reduce costs for repeated system prompts and conversation prefixes. Cache writes cost more but cache reads cost 90% less. Our adapter supports three cache retention levels:

```ts
function getCacheControl(
  model: Model<"anthropic-messages">,
  cacheRetention?: CacheRetention,
): { retention: CacheRetention; cacheControl?: CacheControlEphemeral } {
  const retention = cacheRetention ?? "short";
  if (retention === "none") return { retention };

  return {
    retention,
    cacheControl: retention === "long"
      ? { type: "ephemeral", ttl: 3600 }  // 1 hour
      : { type: "ephemeral" },             // 5 minutes (default)
  };
}
```

The adapter applies `cache_control` markers to:
- The system prompt (always cached)
- The last tool definition (tools are often reused across turns)
- The last user and assistant text content blocks

This strategy maximizes cache hits for multi-turn conversations where the beginning of the context (system prompt, early messages) stays the same while only the tail changes.

## Extended thinking

Anthropic's extended thinking feature exposes the model's internal reasoning before it produces a final answer. Our adapter captures thinking blocks as `ThinkingContent`:

```ts
// Thinking content carries the model's internal reasoning
export interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;
}
```

The `thinkingSignature` is preserved across turns — even if thinking content was redacted by safety filters, the opaque signature lets Anthropic reference the original reasoning context in subsequent requests. This is critical for multi-turn conversations with thinking enabled.

The adapter also maps our unified `reasoning` option to Anthropic's thinking budget:

```ts
function mapThinkingLevel(level?: ThinkingLevel): { budgetTokens: number } | undefined {
  if (!level) return undefined;
  const budgets: Record<ThinkingLevel, number> = {
    minimal: 1024, low: 2048, medium: 4096, high: 8192, xhigh: 16384,
  };
  return { budgetTokens: budgets[level] };
}
```

## Tool call normalization

Anthropic represents tool calls as `tool_use` content blocks with `input: {...}` (already-parsed JSON). Our unified format uses `arguments: Record<string, any>`. The adapter normalizes during event mapping:

```ts
// In content_block_stop for tool_use blocks:
const toolCall: ToolCall = {
  type: "toolCall",
  id: block.id,
  name: block.name,
  arguments: block.input,  // Anthropic parses JSON for us
};
stream.push({
  type: "toolcall_end",
  contentIndex: index,
  toolCall,
  partial: buildPartial(event, model),
});
```

Some compatible providers (like Amazon Bedrock with Anthropic models) send tool input as a JSON string instead of a parsed object. The adapter handles both cases — parsed objects pass through, JSON strings get parsed.

## Registering the adapter

Finally, we register the adapter with the API registry so `streamSimple()` can find it:

```ts
import { registerApiProvider } from "../api-registry.ts";

registerApiProvider("anthropic-messages", {
  streamSimple: streamAnthropic as any,
  completeSimple: /* async wrapper */,
});
```

With the adapter registered, calling `streamSimple({ provider: "anthropic", model: "claude-sonnet-4-6", ... })` will resolve to our `streamAnthropic` function and return live streaming events.

## What we've built

We now have our first working provider adapter. Given a valid `ANTHROPIC_API_KEY`, you can:

```ts
const stream = streamSimple({
  provider: "anthropic",
  model: "claude-sonnet-4-6",
}, {
  systemPrompt: "You are a helpful assistant.",
  messages: [{ role: "user", content: "Hello!", timestamp: Date.now() }],
});

for await (const event of stream) {
  if (event.type === "text_delta") console.write(event.delta);
}
```

In the next chapter, we'll add the OpenAI Responses API adapter, reusing the same EventStream backbone and message transform patterns.

---

← Previous: [The EventStream: Observable Backbone for Streaming](./eventstream-observable-backbone.md) · Next: [Provider Adapter: OpenAI Responses API](./provider-adapter-openai.md) →
