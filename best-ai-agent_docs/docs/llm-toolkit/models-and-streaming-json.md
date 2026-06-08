---
title: "Model Registry, Costs, and Streaming JSON"
description: "Build the model registry with pricing and capability metadata, implement cost calculation, and create the streaming JSON parser that handles partial tool-call arguments mid-stream."
category: llm-toolkit
type: tutorial
tags: [model registry, getModel, getModels, calculateCost, clampThinkingLevel, streaming JSON, partial parse, tool arguments, json repair, token counting, llm-toolkit]
keywords: [model registry, cost calculation, streaming JSON parser, partial JSON repair, token counting]
sources: [S6, S7]
---

**TL;DR** — To make intelligent decisions about model selection, cost tracking, and tool argument parsing, we need three utilities: a model registry that stores every model's capabilities and pricing, a cost calculator that determines how much each API call actually cost, and a streaming JSON parser that can handle partial, malformed tool-call argument buffers arriving character-by-character from the LLM.

## The model registry

The model registry is a lookup table from model IDs to their metadata. Each entry captures what the model can do and what it costs. Create `packages/llm-toolkit/src/models.ts`:

```ts
export interface Model<TApi extends Api = Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  contextWindow: number;
  maxTokens: number;
  inputCostPer1M: number;
  outputCostPer1M: number;
  cacheReadCostPer1M?: number;
  cacheWriteCostPer1M?: number;
  supportsImages: boolean;
  supportsThinking: boolean;
  supportsPromptCaching: boolean;
  thinkingLevels: ThinkingLevel[];
  description: string;
}
```

The registry is a `Map<string, Model>` keyed by model ID:

```ts
const models = new Map<string, Model>();

export function registerModel(model: Model): void {
  models.set(model.id, model);
}

export function getModel(id: string): Model | undefined {
  return models.get(id);
}

export function getModels(provider?: Provider): Model[] {
  const all = Array.from(models.values());
  return provider ? all.filter(m => m.provider === provider) : all;
}
```

### Model definitions

Each model is defined with full metadata:

```ts
registerModel({
  id: "claude-sonnet-4-6",
  name: "Claude Sonnet 4.6",
  api: "anthropic-messages",
  provider: "anthropic",
  contextWindow: 200000,
  maxTokens: 8192,
  inputCostPer1M: 3.00,
  outputCostPer1M: 15.00,
  cacheReadCostPer1M: 0.30,
  cacheWriteCostPer1M: 3.75,
  supportsImages: true,
  supportsThinking: true,
  supportsPromptCaching: true,
  thinkingLevels: ["minimal", "low", "medium", "high"],
  description: "Fast, capable model for coding and general tasks",
});

registerModel({
  id: "gpt-5",
  name: "GPT-5",
  api: "openai-responses",
  provider: "openai",
  contextWindow: 128000,
  maxTokens: 16384,
  inputCostPer1M: 2.50,
  outputCostPer1M: 10.00,
  cacheReadCostPer1M: 1.25,
  supportsImages: true,
  supportsThinking: true,
  supportsPromptCaching: true,
  thinkingLevels: ["minimal", "low", "medium", "high"],
  description: "OpenAI's flagship model",
});
```

The registry grows as new models are released. The auto-generated `models.generated.ts` file contains the complete catalog, refreshed from upstream sources.

## Cost calculation

With per-model pricing stored in the registry, cost calculation is straightforward:

```ts
export function calculateCost(
  model: Model,
  usage: { input: number; output: number; cacheRead: number; cacheWrite: number },
): { input: number; output: number; cacheRead: number; cacheWrite: number; total: number } {
  const inputCost = (usage.input / 1_000_000) * model.inputCostPer1M;
  const outputCost = (usage.output / 1_000_000) * model.outputCostPer1M;
  const cacheReadCost = (usage.cacheRead / 1_000_000) * (model.cacheReadCostPer1M ?? 0);
  const cacheWriteCost = (usage.cacheWrite / 1_000_000) * (model.cacheWriteCostPer1M ?? 0);

  return {
    input: inputCost,
    output: outputCost,
    cacheRead: cacheReadCost,
    cacheWrite: cacheWriteCost,
    total: inputCost + outputCost + cacheReadCost + cacheWriteCost,
  };
}
```

This is called by every provider adapter when building the final `AssistantMessage`, so every response carries its exact dollar cost. The agent harness accumulates these across turns to track session spending.

## The streaming JSON parser

When an LLM generates a tool call, it streams the arguments as JSON — but the JSON arrives in fragments. A single `{"filePath": "/src/app.ts", "line": 42}` might arrive as:

```
Chunk 1: {"filePath
Chunk 2: ": "/src
Chunk 3: /app.ts",
Chunk 4:  "line":
Chunk 5:  42}
```

We can't parse each chunk independently — they're not valid JSON. We need a parser that accumulates the buffer and extracts whatever valid JSON structures it can at each step.

Create `packages/llm-toolkit/src/utils/json-parse.ts`:

### The incremental parser

```ts
export function parseStreamingJson(
  buffer: string,
): { parsed: Record<string, any> | null; remainder: string } {
  // Try to parse the full buffer as complete JSON
  try {
    return { parsed: JSON.parse(buffer), remainder: "" };
  } catch {
    // Not complete yet — try to salvage partial results
  }

  // Try to close open structures and parse
  const closed = tryCloseJson(buffer);
  if (closed) {
    try {
      return { parsed: JSON.parse(closed.closed), remainder: closed.remainder };
    } catch {
      // Still not parseable
    }
  }

  return { parsed: null, remainder: buffer };
}
```

### JSON repair

When JSON is malformed (missing closing brackets, trailing commas, unquoted keys), we attempt repair before parsing:

```ts
export function parseJsonWithRepair(text: string): any {
  // First try straight parse
  try { return JSON.parse(text); } catch {}

  // Attempt repairs:
  let repaired = text;

  // 1. Close unclosed strings (add missing quote at end)
  repaired = closeUnclosedStrings(repaired);

  // 2. Close unclosed objects and arrays
  repaired = closeOpenStructures(repaired);

  // 3. Remove trailing commas before } or ]
  repaired = repaired.replace(/,(\s*[}\]])/g, "$1");

  // 4. Try to handle truncated numbers
  repaired = handleTruncatedNumbers(repaired);

  try { return JSON.parse(repaired); } catch {}

  // 5. Last resort: extract whatever key-value pairs we can
  return extractPartialKeyValues(text);
}
```

The repair strategies are applied in order of safety — closing structures before removing content. The goal is to extract *as much valid data as possible* from a partially-received JSON string, not to guess at missing values.

### How adapters use this

Provider adapters that receive JSON string deltas (OpenAI, not Anthropic) accumulate in a buffer and parse on each delta:

```ts
// In the OpenAI adapter's toolcall_delta handler:
toolArgBuffers.set(contentIndex, (toolArgBuffers.get(contentIndex) ?? "") + delta);
const buffer = toolArgBuffers.get(contentIndex)!;
const { parsed } = parseStreamingJson(buffer);

// We get a partial view of the arguments as they arrive
// (used by the UI to show progress, not for execution)
```

The final parse happens at `toolcall_end` when the complete JSON is guaranteed:

```ts
// At toolcall_end:
const fullJson = toolArgBuffers.get(contentIndex) ?? "";
const args = parseJsonWithRepair(fullJson);
```

The repair step is essential because LLM-generated JSON is often imperfect — trailing commas, inconsistent quoting, or minor syntax issues that a strict parser would reject.

## What we've built

The LLM Toolkit layer is now complete. We have:

| Component | File | Purpose |
|---|---|---|
| Types | `types.ts` | Message, content block, tool, context, event protocol |
| EventStream | `utils/event-stream.ts` | Async-iterable observable for streaming |
| Stream API | `stream.ts` | `streamSimple()`, `completeSimple()` entry points |
| API Registry | `api-registry.ts` | Pluggable provider registration and lookup |
| Auth | `oauth.ts`, `env-api-keys.ts` | OAuth PKCE + environment-based API keys |
| Models | `models.ts` | Model registry with pricing and capabilities |
| JSON Parse | `utils/json-parse.ts` | Streaming + repair JSON parser |
| Provider: Anthropic | `providers/anthropic.ts` | SSE parsing, cache-control, thinking |
| Provider: OpenAI | `providers/openai-responses.ts` | Responses API, reasoning, compat |
| Provider: Google | `providers/google.ts` | GenerateContent, thinking, grounding |

In the next section, we'll build the Agent Core — the loop that turns raw LLM streaming into intelligent, multi-turn tool-using behavior.

---

← Previous: [The API Registry and Authentication](./api-registry-and-auth.md) · Next: [Agent Types, Context, and the Stream Function](../agent-core/agent-types-and-context.md) →
