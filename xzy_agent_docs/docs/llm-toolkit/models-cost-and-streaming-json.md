---
title: "Model Metadata, Cost Calculation, and Streaming JSON Parsing"
description: "Learn how llm-toolkit's model registry and cost utilities work, then master the streaming JSON parser that handles partial tool-call argument buffers mid-stream."
category: llm-toolkit
type: tutorial
tags: [model metadata, getModel, getModels, getProviders, calculateCost, clampThinkingLevel, getSupportedThinkingLevels, streaming JSON, parseStreamingJson, parseJsonWithRepair, repairJson, partial-json, partial tool arguments, json repair, token counting, ModelThinkingLevel, thinking levels, llm-toolkit, Usage, Model, KnownProvider]
keywords: [model registry, model lookup, cost per token, input cost, output cost, cache read cost, cache write cost, thinking budget, extended thinking, incremental JSON, incomplete JSON, malformed JSON, control characters, escape repair, tool call streaming, delta arguments]
sources: [S17, S19]
---

**TL;DR** — Two concrete utilities power the agent layer: a model registry that lets you look up model capabilities and turn raw token counts into a dollar figure, and a streaming JSON parser that safely reconstructs tool-call arguments from partial fragments as they arrive over the wire. By the end of this chapter you will understand how to retrieve model metadata, calculate call cost, clamp a requested thinking level to what a model actually supports, and reliably parse incomplete JSON without crashing the stream.

# Model Metadata, Cost Calculation, and Streaming JSON Parsing

Before we dive in, two quick prerequisites from earlier chapters:

- **Message types and `StreamOptions`** — tool-call content blocks, `ThinkingLevel`, and the `Usage` record (tokens in/out, cache read/write, and their associated costs) are defined in [Message Types and the Streaming API](./message-types-and-streaming-api.md). We reference `Usage` and `ModelThinkingLevel` throughout this chapter.
- **The provider registry** — how a provider is registered and looked up is covered in [The API Registry: Registering and Looking Up Providers](./api-registry-and-extensibility.md). The model registry here is a sibling data structure scoped to individual models within a provider.

---

## Part 1 — Model Metadata

### The problem: the agent needs to know what each model can do

The agent isn't just sending text to a URL. Before it picks a model it needs to know:

- Does this model support extended thinking, and at what levels?
- How large is its context window?
- What does it cost per million tokens (input, output, cache)?

Without a structured answer to these questions the agent either hard-codes values or makes one-off API calls. Neither scales. What we need is a registry indexed by provider and model id — look up a model, get a typed object with everything we need.

### The model registry

`models.ts` builds that registry at module load time from a generated constant called `MODELS`. Think of `MODELS` as a nested object: the outer key is a provider name (a `KnownProvider` string such as `"anthropic"`), and the inner key is a model identifier. At startup, the module walks that structure and stores every model in a `Map<string, Map<string, Model<Api>>>`, keyed first by provider then by model id.

```ts
// Simplified view — the real module also imports types from ./types.ts
import { MODELS } from "./models.generated.ts";
import type { Api, KnownProvider, Model } from "./types.ts";

const modelRegistry: Map<string, Map<string, Model<Api>>> = new Map();

for (const [provider, models] of Object.entries(MODELS)) {
  const providerModels = new Map<string, Model<Api>>();
  for (const [id, model] of Object.entries(models)) {
    providerModels.set(id, model as Model<Api>);
  }
  modelRegistry.set(provider, providerModels);
}
```

`Model<TApi>` is a generic type — the `TApi` parameter narrows which provider API a model belongs to, so the type system can prevent you from passing an Anthropic model to an OpenAI provider adapter. Once the registry is populated you never mutate it; every exported function reads from it.

### `getModel` — look up one model

`getModel` takes a provider and a model id and returns the corresponding `Model` object, preserving the full generic narrowing.

```ts
// From src/models.ts
export function getModel<
  TProvider extends KnownProvider,
  TModelId extends keyof (typeof MODELS)[TProvider]
>(
  provider: TProvider,
  modelId: TModelId,
): Model<ModelApi<TProvider, TModelId>> {
  const providerModels = modelRegistry.get(provider);
  return providerModels?.get(modelId as string) as Model<ModelApi<TProvider, TModelId>>;
}
```

The generic `ModelApi` helper extracts the `api` type from the static `MODELS` definition, so the return type is as narrow as TypeScript can make it — handy when a downstream function is typed to a specific API.

In practice you'd call it like this:

```ts
import { getModel } from "llm-toolkit";

const model = getModel("anthropic", "claude-3-5-sonnet-20241022");
// model is typed as Model<"anthropic"> — fully narrowed
```

If the provider or model id doesn't exist in the registry, `getModel` returns `undefined` (the `?.get()` propagates). You should guard that at the call site.

### `getProviders` and `getModels` — enumerate the registry

Sometimes you need to list everything — for example when building a model-selection UI or validating a user's config.

`getProviders` returns all registered provider names as a `KnownProvider[]`:

```ts
export function getProviders(): KnownProvider[] {
  return Array.from(modelRegistry.keys()) as KnownProvider[];
}
```

`getModels` takes a provider name and returns an array of all `Model` objects for that provider:

```ts
export function getModels<TProvider extends KnownProvider>(
  provider: TProvider,
): Model<ModelApi<TProvider, keyof (typeof MODELS)[TProvider]>>[] {
  const models = modelRegistry.get(provider);
  return models
    ? (Array.from(models.values()) as Model<ModelApi<TProvider, keyof (typeof MODELS)[TProvider]>>[])
    : [];
}
```

If the provider isn't found it returns an empty array rather than throwing.

A quick usage sketch:

```ts
import { getProviders, getModels } from "llm-toolkit";

for (const provider of getProviders()) {
  const models = getModels(provider);
  console.log(`${provider}: ${models.map(m => m.id).join(", ")}`);
}
```

---

### The problem: how much did that call cost?

After a model returns, `stream()` gives you a `Usage` record — input token count, output token count, cacheRead token count, cacheWrite token count, and a `cost` sub-object. The token counts are raw numbers from the API. The cost fields start at zero. To turn those counts into a dollar figure we need the per-million-token pricing that lives on the `Model` object.

### `calculateCost` — pricing in, dollar figures out

`calculateCost` takes the `Model` you called and the `Usage` record returned from the stream, mutates the `cost` sub-object in place, and returns it.

```ts
// From src/models.ts
export function calculateCost<TApi extends Api>(
  model: Model<TApi>,
  usage: Usage,
): Usage["cost"] {
  usage.cost.input      = (model.cost.input      / 1_000_000) * usage.input;
  usage.cost.output     = (model.cost.output     / 1_000_000) * usage.output;
  usage.cost.cacheRead  = (model.cost.cacheRead  / 1_000_000) * usage.cacheRead;
  usage.cost.cacheWrite = (model.cost.cacheWrite / 1_000_000) * usage.cacheWrite;
  usage.cost.total =
    usage.cost.input +
    usage.cost.output +
    usage.cost.cacheRead +
    usage.cost.cacheWrite;
  return usage.cost;
}
```

The pattern is the same for each token type: the model stores the price as a dollar amount per million tokens, so we divide by one million then multiply by the raw count. Notice that the function **mutates** `usage.cost` and also returns it — if you need an immutable record, copy `usage.cost` after the call.

Here is how that looks end-to-end:

```ts
import { getModel, calculateCost } from "llm-toolkit";

// After a streaming call completes and you have the usage record:
const model = getModel("anthropic", "claude-3-5-sonnet-20241022");
const cost = calculateCost(model, usage);

console.log(`input:  $${cost.input.toFixed(6)}`);
console.log(`output: $${cost.output.toFixed(6)}`);
console.log(`total:  $${cost.total.toFixed(6)}`);
```

The four cost fields and how they map to model pricing:

| `usage` field   | `model.cost` field   | What it counts                           |
|-----------------|----------------------|------------------------------------------|
| `usage.input`   | `model.cost.input`   | Tokens in the prompt sent to the model   |
| `usage.output`  | `model.cost.output`  | Tokens the model generated in reply      |
| `usage.cacheRead` | `model.cost.cacheRead` | Prompt tokens served from cache        |
| `usage.cacheWrite` | `model.cost.cacheWrite` | Prompt tokens written to cache       |

---

### The problem: not every model supports every thinking level

Extended thinking (where the model reasons before answering) is a capability only some models have, and even among those models, not every level in the full scale is valid. If the user asks for `"high"` thinking on a model that tops out at `"low"`, the request will either fail or silently downgrade with unpredictable results. We need a way to ask "what levels does this model accept?" and to safely clamp a requested level to the highest it supports.

### Thinking levels — the full scale

The six levels in ascending order are:

| Level     | Index | Meaning                          |
|-----------|-------|----------------------------------|
| `"off"`   | 0     | No extended thinking             |
| `"minimal"` | 1   | Minimal reasoning budget         |
| `"low"`   | 2     | Low reasoning budget             |
| `"medium"` | 3    | Medium reasoning budget          |
| `"high"`  | 4     | High reasoning budget            |
| `"xhigh"` | 5     | Extra-high reasoning budget      |

This ordered array is the internal constant `EXTENDED_THINKING_LEVELS` in `models.ts`.

### `getSupportedThinkingLevels` — ask the model what it accepts

```ts
// From src/models.ts
const EXTENDED_THINKING_LEVELS: ModelThinkingLevel[] =
  ["off", "minimal", "low", "medium", "high", "xhigh"];

export function getSupportedThinkingLevels<TApi extends Api>(
  model: Model<TApi>,
): ModelThinkingLevel[] {
  if (!model.reasoning) return ["off"];

  return EXTENDED_THINKING_LEVELS.filter((level) => {
    const mapped = model.thinkingLevelMap?.[level];
    if (mapped === null) return false;
    if (level === "xhigh") return mapped !== undefined;
    return true;
  });
}
```

Two things to notice:

1. If `model.reasoning` is falsy the function short-circuits to `["off"]` — the model does not support reasoning at all.
2. For models that do support reasoning it filters `EXTENDED_THINKING_LEVELS` using `model.thinkingLevelMap`. A level is excluded if the map entry is explicitly `null`; `"xhigh"` is additionally excluded if the map has no entry for it at all (i.e. `undefined`). All other levels pass through.

### `clampThinkingLevel` — the safe downgrade

Now we can build the clamping function. Given a model and a requested level, it returns the highest level the model supports that is at or above what was requested. If no higher level is available it falls back to the closest lower level. If nothing is available at all it returns `"off"`.

```ts
// From src/models.ts
export function clampThinkingLevel<TApi extends Api>(
  model: Model<TApi>,
  level: ModelThinkingLevel,
): ModelThinkingLevel {
  const availableLevels = getSupportedThinkingLevels(model);
  if (availableLevels.includes(level)) return level;          // exact match

  const requestedIndex = EXTENDED_THINKING_LEVELS.indexOf(level);
  if (requestedIndex === -1) return availableLevels[0] ?? "off"; // unknown level

  // Try to find a level at or above the requested index
  for (let i = requestedIndex; i < EXTENDED_THINKING_LEVELS.length; i++) {
    const candidate = EXTENDED_THINKING_LEVELS[i];
    if (availableLevels.includes(candidate)) return candidate;
  }

  // Fall back to the nearest lower level
  for (let i = requestedIndex - 1; i >= 0; i--) {
    const candidate = EXTENDED_THINKING_LEVELS[i];
    if (availableLevels.includes(candidate)) return candidate;
  }

  return availableLevels[0] ?? "off";
}
```

Walk through an example: say the model supports `["off", "low", "medium"]` and the user requests `"high"`.

1. `"high"` is not in `availableLevels`, so we continue.
2. `requestedIndex` is 4 (`"high"` in `EXTENDED_THINKING_LEVELS`).
3. We scan upward: `"high"` (4) — not available; `"xhigh"` (5) — not available.
4. We scan downward: `"medium"` (3) — available. Return `"medium"`.

The agent calls `clampThinkingLevel` before constructing the request, so it never sends an invalid budget to the API.

A companion utility, `modelsAreEqual`, compares two `Model` objects by both `id` and `provider` — it returns `false` if either argument is `null` or `undefined`, making it safe to use in guarded contexts.

```ts
export function modelsAreEqual<TApi extends Api>(
  a: Model<TApi> | null | undefined,
  b: Model<TApi> | null | undefined,
): boolean {
  if (!a || !b) return false;
  return a.id === b.id && a.provider === b.provider;
}
```

---

## Part 2 — Streaming JSON Parsing

### The problem: tool arguments arrive as broken fragments

When a model invokes a tool during streaming, it doesn't send the complete JSON object for the tool's arguments in one go. It sends a sequence of text deltas — partial chunks that arrive one event at a time. Each delta might look like:

```
{"path": "/src/foo
```

then:

```
.ts", "content": "cons
```

then:

```
t x = 1;\n"}
```

If we try to `JSON.parse` any one of those intermediate states, we get a `SyntaxError`. But for the agent's UI — showing a live preview of what the model is about to do — we want to parse as much as possible at each tick. We also need to handle the case where the JSON is not just incomplete but structurally broken: raw newlines or tab characters inside a string literal, badly escaped backslashes, and so on.

The solution is a two-layer approach: repair first, then parse with partial-JSON tolerance.

### Layer 1 — `repairJson`: fix structural damage

`repairJson` is a single-pass character scanner that normalises malformed JSON string literals. It handles two classes of damage:

- **Raw control characters inside strings** (e.g. a literal newline `\n` — code point ≤ 0x1f) — it escapes them to their JSON-safe equivalents (`\\n`, `\\t`, etc., or `\\uXXXX` for others).
- **Invalid backslash escapes** (e.g. `\p`) — it doubles the backslash to `\\p` so the literal backslash is preserved.

```ts
// From src/utils/json-parse.ts

const VALID_JSON_ESCAPES = new Set(['"', "\\", "/", "b", "f", "n", "r", "t", "u"]);

function isControlCharacter(char: string): boolean {
  const codePoint = char.codePointAt(0);
  return codePoint !== undefined && codePoint >= 0x00 && codePoint <= 0x1f;
}

function escapeControlCharacter(char: string): string {
  switch (char) {
    case "\b": return "\\b";
    case "\f": return "\\f";
    case "\n": return "\\n";
    case "\r": return "\\r";
    case "\t": return "\\t";
    default:
      return `\\u${char.codePointAt(0)?.toString(16).padStart(4, "0") ?? "0000"}`;
  }
}

export function repairJson(json: string): string {
  let repaired = "";
  let inString = false;

  for (let index = 0; index < json.length; index++) {
    const char = json[index];

    if (!inString) {
      repaired += char;
      if (char === '"') inString = true;
      continue;
    }

    if (char === '"') {
      repaired += char;
      inString = false;
      continue;
    }

    if (char === "\\") {
      const nextChar = json[index + 1];

      if (nextChar === undefined) {
        repaired += "\\\\";   // trailing backslash — double it
        continue;
      }

      if (nextChar === "u") {
        const unicodeDigits = json.slice(index + 2, index + 6);
        if (/^[0-9a-fA-F]{4}$/.test(unicodeDigits)) {
          repaired += `\\u${unicodeDigits}`;
          index += 5;         // consume \uXXXX entirely
          continue;
        }
      }

      if (VALID_JSON_ESCAPES.has(nextChar)) {
        repaired += `\\${nextChar}`;
        index += 1;           // consume the escape pair
        continue;
      }

      repaired += "\\\\";     // invalid escape — double the backslash
      continue;
    }

    repaired += isControlCharacter(char) ? escapeControlCharacter(char) : char;
  }

  return repaired;
}
```

The scanner tracks whether it is inside a string (`inString`) and applies escape logic only inside strings — outside them JSON structure characters are passed through untouched.

A quick worked example. Suppose the model emits a file-write tool argument that contains a raw tab character:

```
Input:  {"content": "hello	world"}
                            ^ raw tab (0x09) inside the string
```

`repairJson` reaches the tab character while `inString = true`, identifies it as a control character, and replaces it with `"\\t"`:

```
Output: {"content": "hello\tworld"}
```

That output is valid JSON which `JSON.parse` can now handle.

### Layer 2 — `parseJsonWithRepair`: repair then parse

`parseJsonWithRepair` is a thin wrapper that first tries `JSON.parse` on the raw input, and only if that throws does it run `repairJson` and try again.

```ts
// From src/utils/json-parse.ts
export function parseJsonWithRepair<T>(json: string): T {
  try {
    return JSON.parse(json) as T;
  } catch (error) {
    const repairedJson = repairJson(json);
    if (repairedJson !== json) {
      return JSON.parse(repairedJson) as T;
    }
    throw error;   // repair made no change, re-throw the original error
  }
}
```

Notice the guard `if (repairedJson !== json)`: if repair produced the same string as the input then repair didn't help, so the original error is re-thrown rather than silently failing in an infinite repair loop.

This function handles **structurally complete but character-damaged** JSON. It still throws if the JSON is syntactically incomplete — an unclosed `{`, a missing `"`, a truncated array. That is the problem the next layer solves.

### Layer 3 — `parseStreamingJson`: tolerant incremental parsing

`parseStreamingJson` is the function the stream consumer actually calls. It receives a potentially `undefined`, empty, or partial JSON string and returns a typed object — never throws.

```ts
// From src/utils/json-parse.ts
import { parse as partialParse } from "partial-json";

export function parseStreamingJson<T = Record<string, unknown>>(
  partialJson: string | undefined,
): T {
  if (!partialJson || partialJson.trim() === "") {
    return {} as T;        // nothing yet — return empty object
  }

  try {
    return parseJsonWithRepair<T>(partialJson);   // fast path: complete JSON
  } catch {
    try {
      const result = partialParse(partialJson);   // try partial-json as-is
      return (result ?? {}) as T;
    } catch {
      try {
        const result = partialParse(repairJson(partialJson));  // repair then partial-parse
        return (result ?? {}) as T;
      } catch {
        return {} as T;    // give up — return empty object rather than throwing
      }
    }
  }
}
```

The fallback chain has four levels:

| Step | Input fed to parser | What handles it |
|------|---------------------|-----------------|
| 1 | Raw `partialJson` | `parseJsonWithRepair` — succeeds for complete (possibly damaged) JSON |
| 2 | Raw `partialJson` | `partialParse` from the `partial-json` library — succeeds for incomplete but well-formed JSON |
| 3 | `repairJson(partialJson)` | `partialParse` again — succeeds for incomplete *and* damaged JSON |
| 4 | — | Return `{}` — all parsers failed; safest no-op |

`partial-json` (imported as `partialParse`) is a library that extends JSON parsing to accept valid prefixes — an unclosed object, a truncated string — by inferring the minimal completion. It is the key to live-preview mid-stream.

The function returns `{}` (an empty object cast to `T`) in every failure case instead of throwing. This is intentional: during streaming, the caller should handle `{}` as "not enough data yet" and re-try on the next delta rather than crashing the stream handler.

### A complete worked example: tool arguments arriving mid-stream

Let's trace through a realistic sequence. Imagine the model is calling a `write_file` tool and the arguments are streaming in three deltas.

**Delta 1** (very partial — the JSON isn't even an object yet):

```
{"path": "/src/util
```

```ts
const args1 = parseStreamingJson('{"path": "/src/util');
// -> Step 1 fails (JSON.parse throws — unclosed object)
// -> Step 2: partialParse('{"path": "/src/util')
//    partial-json infers the minimal completion → { path: "/src/util" }
// args1 = { path: "/src/util" }
```

**Delta 2** (more complete, but content includes a raw newline):

```
{"path": "/src/utils.ts", "content": "const x = 1;
const y = 2;
```

```ts
const args2 = parseStreamingJson(
  '{"path": "/src/utils.ts", "content": "const x = 1;\nconst y = 2;'
  //                                                    ^ raw newline (0x0a) in string
);
// -> Step 1: JSON.parse throws; repairJson escapes the newline to \\n;
//    JSON.parse again — still fails (unclosed string)
//    so parseJsonWithRepair throws
// -> Step 2: partialParse of the raw string — may also fail due to the raw newline
// -> Step 3: partialParse(repairJson(input))
//    repairJson escapes the newline, then partialParse handles the unclosed string
// args2 = { path: "/src/utils.ts", content: "const x = 1;\nconst y = 2;" }
```

**Delta 3** (complete JSON, no damage):

```
{"path": "/src/utils.ts", "content": "const x = 1;\nconst y = 2;\n"}
```

```ts
const args3 = parseStreamingJson(
  '{"path": "/src/utils.ts", "content": "const x = 1;\\nconst y = 2;\\n"}'
);
// -> Step 1: JSON.parse succeeds immediately
// args3 = { path: "/src/utils.ts", content: "const x = 1;\nconst y = 2;\n" }
```

At each tick the agent has the best available read of the tool arguments. When delta 3 completes, the object is fully formed and the stream can proceed to execute the tool.

### Where `parseStreamingJson` fits in the stream

In the stream-processing layer (covered in [Message Types and the Streaming API](./message-types-and-streaming-api.md)), each tool-call delta carries a string of accumulated JSON in its `partial_json` field. You accumulate those deltas into a buffer and call `parseStreamingJson` on the buffer after each delta — building a live view of the tool arguments before the call completes:

```ts
import { parseStreamingJson } from "llm-toolkit";

let argBuffer = "";

// Inside your delta handler:
for await (const delta of stream) {
  if (delta.type === "tool_call_delta") {
    argBuffer += delta.partial_json ?? "";
    const args = parseStreamingJson(argBuffer);
    // args is always a plain object — render a live preview here
  }
}
```

The buffer grows with each delta. `parseStreamingJson` always returns something, so you can render a live preview without any try/catch at the call site.

---

## Summary

We built two families of utilities in this chapter:

**Model registry and metadata:**

| Function | What it does |
|---|---|
| `getModel(provider, modelId)` | Returns the `Model` object for a specific provider and model id |
| `getProviders()` | Lists all registered provider names as `KnownProvider[]` |
| `getModels(provider)` | Returns all models for a given provider as `Model[]`; empty array if not found |
| `calculateCost(model, usage)` | Mutates `usage.cost` with per-token pricing and returns it |
| `getSupportedThinkingLevels(model)` | Returns the `ModelThinkingLevel[]` the model actually supports |
| `clampThinkingLevel(model, level)` | Returns the highest supported level ≥ requested; falls back downward; never throws |
| `modelsAreEqual(a, b)` | Compares two `Model` objects by `id` and `provider`; safe on null/undefined |

**Streaming JSON:**

| Function | What it does |
|---|---|
| `repairJson(json)` | Escapes raw control characters and doubles invalid backslashes inside JSON string literals |
| `parseJsonWithRepair<T>(json)` | Parses JSON; if it fails, repairs then re-parses; throws only if repair didn't help |
| `parseStreamingJson<T>(partialJson)` | Four-level fallback: repair+parse → partial-parse → repair+partial-parse → `{}`; never throws |

These two families keep the agent's runtime correct in both dimensions: it always knows what a model can do and what it will cost, and it can safely read tool arguments from a live stream without ever crashing on a truncated buffer.

---

← Previous: [The API Registry: Registering and Looking Up Providers](./api-registry-and-extensibility.md) · Next: [OAuth and API-Key Authentication](./oauth-and-api-key-auth.md) →
