---
title: "Provider Adapter: OpenAI Responses API"
description: "Implement the OpenAI Responses API adapter — event parsing, reasoning extraction, tool call handling with delta reassembly, prompt caching, and compatibility with OpenAI-compatible providers."
category: llm-toolkit
type: tutorial
tags: [OpenAI, Responses API, streaming, tool use, reasoning, prompt caching, delta reassembly, provider adapter, llm-toolkit, compatibility]
keywords: [OpenAI adapter, Responses API, reasoning_effort, tool streaming, OpenAI compatibility, delta parsing]
sources: [S11, S16]
---

**TL;DR** — We'll build the OpenAI Responses API adapter, which translates our unified types into OpenAI's request format, parses the streaming response, extracts reasoning blocks, reassembles tool call argument deltas into complete JSON, and supports prompt caching. The adapter also handles the many OpenAI-compatible providers (xAI, Groq, DeepSeek, OpenRouter) through a compatibility layer.

## The OpenAI Responses API

OpenAI's Responses API is a newer streaming interface designed for agent interactions. Unlike the older Chat Completions API, it natively supports tools, reasoning, and structured output in a single unified endpoint.

Install the OpenAI SDK:

```bash
cd packages/llm-toolkit
npm install openai
```

Create `packages/llm-toolkit/src/providers/openai-responses.ts`.

## Building the request

The OpenAI request format differs from Anthropic's in several ways. Messages are in an `input` array (not `messages`), tools have a different shape, and reasoning is controlled via `reasoning_effort`:

```ts
function buildOpenAIRequest(
  model: Model<"openai-responses">,
  context: Context,
  options?: SimpleStreamOptions,
) {
  return {
    model: model.id,
    input: context.messages.map(toOpenAIItem),
    instructions: context.systemPrompt,
    tools: context.tools?.length ? toOpenAITools(context.tools) : undefined,
    reasoning: options?.reasoning ? { effort: options.reasoning } : undefined,
    stream: true,
  };
}
```

### Message transformation

Our `Message[]` maps to OpenAI's input items. Key differences from Anthropic:

- OpenAI uses a flat `input` array where tool results reference their call by `call_id`
- Assistant messages with tool calls must list tools in the `output` array
- Thinking/reasoning content maps to `reasoning` output items

```ts
function toOpenAIItem(msg: Message): ResponseInputItem {
  switch (msg.role) {
    case "user":
      return { role: "user", content: msg.content };
    case "assistant": {
      const output: ResponseOutputItem[] = [];
      for (const block of msg.content) {
        if (block.type === "text") output.push({ type: "message", content: [{ type: "output_text", text: block.text }] });
        if (block.type === "thinking") output.push({ type: "reasoning", summary: [{ type: "summary_text", text: block.thinking }] });
        if (block.type === "toolCall") output.push({ type: "function_call", call_id: block.id, name: block.name, arguments: JSON.stringify(block.arguments) });
      }
      return { role: "assistant", output };
    }
    case "toolResult":
      return { role: "tool", call_id: msg.toolCallId, output: msg.content };
  }
}
```

## Streaming event parsing

OpenAI's stream delivers events with a different structure than Anthropic's SSE. The key event types and our mappings:

| OpenAI event | Our event |
|---|---|
| `response.created` | `{ type: "start" }` |
| `response.output_text.delta` | `{ type: "text_delta" }` |
| `response.reasoning_summary_text.delta` | `{ type: "thinking_delta" }` |
| `response.function_call_arguments.delta` | `{ type: "toolcall_delta" }` |
| `response.function_call_arguments.done` | `{ type: "toolcall_end" }` |
| `response.completed` | `{ type: "done" }` |
| `error` | `{ type: "error" }` |

The main loop in the adapter:

```ts
for await (const event of client.responses.stream(params)) {
  switch (event.type) {
    case "response.output_text.delta":
      stream.push({
        type: "text_delta",
        contentIndex: currentTextIndex,
        delta: event.delta,
        partial: updatePartialWithText(event),
      });
      break;

    case "response.function_call_arguments.delta":
      stream.push({
        type: "toolcall_delta",
        contentIndex: currentToolIndex,
        delta: event.delta,
        partial: updatePartialWithToolDelta(event),
      });
      break;

    case "response.function_call_arguments.done":
      stream.push({
        type: "toolcall_end",
        contentIndex: currentToolIndex,
        toolCall: {
          type: "toolCall",
          id: event.call_id,
          name: event.name,
          arguments: JSON.parse(event.arguments),
        },
        partial: updatePartialWithToolEnd(event),
      });
      break;

    case "response.completed":
      stream.push({
        type: "done",
        reason: mapStopReason(event.response),
        message: buildFinalMessage(event.response),
      });
      break;
  }
}
```

## Delta reassembly for tool calls

Unlike Anthropic (which parses tool input JSON for us), OpenAI streams tool call arguments as raw JSON string deltas. We need to accumulate deltas and parse the complete JSON only when the tool call ends:

```ts
// Track accumulating tool call arguments per content index
const toolArgBuffers = new Map<number, string>();

// During response.function_call_arguments.delta:
const buffer = (toolArgBuffers.get(eventIndex) ?? "") + event.delta;
toolArgBuffers.set(eventIndex, buffer);

// During response.function_call_arguments.done:
const fullArgs = toolArgBuffers.get(eventIndex) ?? "";
const parsed = JSON.parse(fullArgs);
```

This is important because a single `arguments` JSON string might arrive across dozens of deltas. We can't parse partial JSON — we have to wait for `done` to get the complete string.

## Reasoning (thinking) support

OpenAI models that support reasoning expose their internal chain-of-thought through `reasoning` items. The adapter maps these to our `ThinkingContent` blocks:

```ts
case "response.reasoning_summary_text.delta":
  stream.push({
    type: "thinking_delta",
    contentIndex: reasoningIndex,
    delta: event.delta,
    partial: updatePartialWithThinking(event),
  });
  break;
```

The unified `reasoning` option maps to OpenAI's `reasoning_effort`:

| Our level | OpenAI reasoning_effort |
|---|---|
| `"minimal"` | `"minimal"` |
| `"low"` | `"low"` |
| `"medium"` | `"medium"` |
| `"high"` | `"high"` |
| `"xhigh"` | `"high"` (OpenAI caps at high) |

## Prompt caching

OpenAI's prompt caching is automatic for repeated prefixes — you don't need to annotate cache breakpoints like you do with Anthropic. However, the adapter does support session affinity via the `session_id` header, which increases cache hit rates for multi-turn conversations:

```ts
if (options?.sessionId && options?.cacheRetention !== "none") {
  headers["session_id"] = options.sessionId;
}
```

For long cache retention (24 hours), the adapter adds `prompt_cache_retention: "24h"` to the request. This is a significant cost optimization — cached tokens are 50% cheaper.

## Compatibility with OpenAI-compatible providers

Many providers offer OpenAI-compatible APIs (xAI, Groq, DeepSeek, OpenRouter, Together, Fireworks, and more). Rather than write a separate adapter for each, our OpenAI adapter uses a compatibility configuration to handle differences:

```ts
export interface OpenAICompletionsCompat {
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;
  supportsReasoningEffort?: boolean;
  maxTokensField?: "max_completion_tokens" | "max_tokens";
  thinkingFormat?: "openai" | "openrouter" | "deepseek" | "together" | "zai" | "qwen";
  // ... more compat flags
}
```

For example, DeepSeek uses `thinking: { type: "enabled" }` instead of `reasoning_effort`, so the adapter sets `thinkingFormat: "deepseek"` and the request builder switches fields accordingly. Most compat flags are auto-detected from the base URL, so adding a new OpenAI-compatible provider often requires zero configuration.

## What we've built

We now have two working provider adapters (Anthropic and OpenAI), both implementing the same `StreamFunction` contract, both emitting the same event protocol. The rest of the system — the agent loop, the UI — doesn't know which one is running.

---

← Previous: [Provider Adapter: Anthropic Messages API](./provider-adapter-anthropic.md) · Next: [Provider Adapter: Google Gemini](./provider-adapter-google.md) →
