---
title: "Provider Adapter: Google Gemini"
description: "Implement the Google Gemini provider adapter — non-SSE response handling, schema conversion, thinking support, and the transform layer that maps Gemini's format to our unified event protocol."
category: llm-toolkit
type: tutorial
tags: [Google, Gemini, Generative AI, thinking, search grounding, schema conversion, non-SSE format, provider adapter, llm-toolkit]
keywords: [Google adapter, Gemini API, GenerateContent, thinking config, schema mapping, tool declaration]
sources: [S12, S16]
---

**TL;DR** — Google's Gemini API uses a different streaming format than both Anthropic (SSE) and OpenAI. We'll build the Gemini adapter, handling non-SSE streaming, schema conversion between our TypeBox tool schemas and Gemini's OpenAPI-like function declarations, thinking configuration, and Google-specific features like search grounding. By the end, all three major providers work through the same `streamSimple()` interface.

## Install and setup

```bash
cd packages/llm-toolkit
npm install @google/genai
```

Create `packages/llm-toolkit/src/providers/google.ts`.

## How Gemini streaming differs

Gemini's `generateContentStream` returns an async iterable of `GenerateContentResponse` chunks. Each chunk contains a `candidates` array, and each candidate has `content` with `parts[]`. This is fundamentally different from SSE — there are no `event:` lines, no `data:` payloads. The streaming is at the SDK level, not the HTTP level.

Our mapping challenge:

```
Gemini chunk                        Our event
────────────                        ─────────
candidates[0].content.parts[]       →  content blocks
  part.text                         →  text_delta (accumulated)
  part.thought                      →  thinking_delta
  part.functionCall                 →  toolcall_start → toolcall_end
usageMetadata                       →  usage (on final chunk)
finishReason                        →  stopReason
```

## Tool schema conversion

Gemini expects tools declared as `FunctionDeclaration` objects with OpenAPI-like parameter schemas. Our system uses TypeBox schemas. The adapter converts between them:

```ts
function toGeminiTool(tool: Tool): FunctionDeclaration {
  return {
    name: tool.name,
    description: tool.description,
    parameters: typeBoxToGeminiSchema(tool.parameters),
  };
}
```

The schema conversion handles TypeBox's type system (string, number, boolean, object, array, union, enum) and maps them to Gemini's expected JSON Schema subset. Some TypeBox features (like `$ref` or `anyOf` with mixed types) require special handling because Gemini's schema support is more limited.

## Thinking configuration

Gemini exposes thinking through a `thinkingConfig` parameter. Our unified `reasoning` option maps to Gemini's thinking budget:

```ts
function mapThinkingConfig(reasoning?: ThinkingLevel) {
  if (!reasoning) return undefined;
  const budgets: Record<ThinkingLevel, number> = {
    minimal: 512, low: 1024, medium: 2048, high: 4096, xhigh: 8192,
  };
  return {
    thinkingBudget: budgets[reasoning],
    includeThoughts: true,
  };
}
```

Gemini's thinking tokens are included in the response by default when `includeThoughts` is true. Each thought appears as a `part.thought` (a boolean) with the actual thinking text in a separate `part.text` when the model is "thinking." The adapter extracts and normalizes these into our `ThinkingContent` blocks.

## Search grounding

Gemini supports Google Search grounding — the model can search the web mid-response and cite sources. Our adapter exposes this through provider-specific options:

```ts
if (options?.googleSearchGrounding) {
  tools.push({ googleSearch: {} });
}
```

When grounding is active, Gemini may return `groundingMetadata` in the response, which the adapter surfaces through the diagnostics channel.

## Streaming loop

The main adapter function follows the same pattern as Anthropic and OpenAI:

```ts
export const streamGoogle: StreamFunction<"google-generative-ai"> = (
  model, context, options,
): AssistantMessageEventStream => {
  const stream = new AssistantMessageEventStream();

  (async () => {
    try {
      const client = new GoogleGenAI({ apiKey: options?.apiKey });
      const response = await client.models.generateContentStream({
        model: model.id,
        contents: context.messages.map(toGeminiContent),
        config: {
          systemInstruction: context.systemPrompt,
          tools: context.tools?.length ? [{ functionDeclarations: context.tools.map(toGeminiTool) }] : undefined,
          thinkingConfig: mapThinkingConfig(options?.reasoning),
        },
      });

      for await (const chunk of response) {
        if (!chunk.candidates?.length) continue;
        const candidate = chunk.candidates[0];
        if (!candidate.content?.parts) continue;

        for (let i = 0; i < candidate.content.parts.length; i++) {
          const part = candidate.content.parts[i];
          if (part.text) emitTextDelta(stream, i, part.text);
          if (part.thought) emitThinkingDelta(stream, i, part);
          if (part.functionCall) emitToolCall(stream, i, part.functionCall);
        }

        if (candidate.finishReason) {
          stream.push({
            type: "done",
            reason: mapGeminiFinishReason(candidate.finishReason),
            message: buildFinalMessage(chunk),
          });
        }
      }
    } catch (err) {
      stream.push({ type: "error", reason: "error", error: buildErrorMessage(err, model) });
    }
  })();

  return stream;
};
```

## Message transformation quirks

Gemini has several message format requirements that differ from our unified types:

1. **No empty content arrays.** Gemini rejects messages with empty `parts[]`. If a message's content is empty after transformation, the adapter inserts a placeholder text part.

2. **Alternating roles.** Gemini requires strict alternation between `user` and `model` roles. The adapter merges consecutive messages of the same role when necessary.

3. **Tool results as function responses.** Gemini represents tool results as `functionResponse` parts within a `user`-role content, similar to Anthropic's `tool_result` blocks within user messages.

4. **Thought signatures.** Gemini can attach an opaque `thoughtSignature` to function calls, enabling thought context reuse across turns. The adapter preserves this in our `ToolCall.thoughtSignature` field.

## What we've built

With three provider adapters (Anthropic, OpenAI, Google), our LLM Toolkit can now talk to the three major AI providers through a single unified interface. The code that calls `streamSimple()` doesn't know which provider is on the other end — it just receives typed events.

In the next chapter, we'll build the API registry that ties all three adapters together and the authentication layer that handles API keys and OAuth.

---

← Previous: [Provider Adapter: OpenAI Responses API](./provider-adapter-openai.md) · Next: [The API Registry and Authentication](./api-registry-and-auth.md) →
