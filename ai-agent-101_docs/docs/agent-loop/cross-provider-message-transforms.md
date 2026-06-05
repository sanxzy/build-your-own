---
title: "Cross-Provider Message Transforms and Handoff"
description: "How thinking blocks and provider-specific content are transformed when handing a conversation between providers, and how correctness is verified with replay tests."
category: agent-loop
type: explanation
tags: [message transform, thinking block, cross-provider, handoff, provider replay, agent-core, text conversion, transform-messages, provider compatibility, tool call ID, redacted thinking, thinking signature, aborted message, synthetic tool result, image downgrade, isSameModel]
keywords: [switch model mid-session, provider switch, conversation portability, tool ID normalization, cross-provider test, provider handoff, message history replay, thinking block conversion, orphaned tool call]
sources: [S16, S23]
---

**TL;DR** — When a conversation is continued with a different model or provider, the recorded history may contain provider-specific structures — such as thinking blocks, encrypted reasoning signatures, or unusual tool-call ID formats — that the new provider cannot accept. The `transformMessages` function converts that history into a portable form before it is replayed. This chapter explains the problem, walks through each transformation rule, and shows the cross-provider replay test strategy that verifies the approach across a wide range of real providers.

# Cross-Provider Message Transforms and Handoff

> **Prerequisites.** This chapter builds on two earlier ideas, recapped briefly where they first appear below:
>
> - **The `Message` type** — a conversation is recorded as a list of typed `Message` values (`role: "user"`, `role: "assistant"` with content blocks, and `role: "toolResult"`), defined in the `agent-core` types module. See [Agent Context and Types](./agent-context-and-types.md) for the full definitions.
> - **Sessions and the harness** — the harness records messages into a session and replays them on the next turn; `transformMessages` slots in right before that replay payload is assembled. See [The Agent Harness](./harness-session-and-compaction.md).

## The problem: one provider's output is another's error

Let's start by naming the situation precisely. Our agent records every turn of a conversation as a typed `Message` — user turns, assistant turns (with their content blocks), and tool results. Those messages accumulate in a session (see [the harness chapter](./harness-session-and-compaction.md) for how sessions persist and replay conversations).

As long as you keep talking to the same model, replaying this history is straightforward: hand it back verbatim and the model continues. But what happens when you switch the active model mid-session? Maybe the user typed `/model switch` in the interactive CLI, or a skill programmatically chose a different model for the next sub-task.

Now we have a mismatch problem. Consider an assistant turn that was recorded while talking to a provider that supports extended thinking — the kind where the model emits a reasoning chain alongside the final answer. That turn's content might look like:

```ts
// Simplified view of an assistant message recorded with a thinking-capable provider
{
  role: "assistant",
  provider: "anthropic",
  api: "anthropic",
  model: "claude-sonnet-4-5",
  content: [
    {
      type: "thinking",
      thinking: "The user wants me to double 21. That's 42.",
      thinkingSignature: "sig_abc123...",
    },
    { type: "text", text: "The result is 42." },
    {
      type: "toolCall",
      id: "toolu_xyz",
      name: "double_number",
      args: { value: 21 },
      thoughtSignature: "sig_abc123...",
    },
  ],
  stopReason: "tool_use",
}
```

If we hand this message verbatim to a provider that does not support thinking blocks, it will reject or mishandle the `thinking`-typed content block. Similarly, some providers generate tool-call IDs in formats other providers reject — for instance, the OpenAI Responses API generates IDs that are over 450 characters long and can contain pipe characters (`|`), which are illegal in Anthropic's `^[a-zA-Z0-9_-]+$` format (max 64 chars).

The solution is a single transformation step that normalises the history before replaying it with the new provider.

## Introducing `transformMessages`

The `transformMessages` function (exported from `agent-core`'s `providers/transform-messages.ts`) takes a message array, a target `Model`, and an optional ID-normalisation callback, and returns a new array safe for that model.

```ts
// Simplified signature
export function transformMessages<TApi extends Api>(
  messages: Message[],
  model: Model<TApi>,
  normalizeToolCallId?: (id: string, model: Model<TApi>, source: AssistantMessage) => string,
): Message[]
```

- `messages` — the raw conversation history.
- `model` — the **target** model we are about to send to. The function inspects `model.provider`, `model.api`, and `model.id` to decide what can be preserved versus what must be converted.
- `normalizeToolCallId` — an optional callback the caller supplies when they know the target provider's ID constraints. The function itself detects the need; the caller decides the strategy.

The function works in two passes. We'll walk through each one.

## Pass 1 — per-message content transformation

The first pass maps over every message and applies three distinct transformations.

### Distinguishing "same model" from "different model"

Before touching any content block, the function computes a flag for each assistant message:

```ts
// Inside the first-pass map — Simplified view
const isSameModel =
  assistantMsg.provider === model.provider &&
  assistantMsg.api    === model.api      &&
  assistantMsg.model  === model.id;
```

`isSameModel` is `true` only when the recorded message came from the exact same provider+API+model triple we are targeting now. This single boolean controls almost every downstream decision — keeping provider-native structures when replaying to the same model, converting them when handing off to a different one.

### Thinking blocks

A `thinking` content block is the assistant's internal reasoning chain. It comes in two flavours: plain thinking text, and *redacted* (encrypted) thinking, which is an opaque blob the provider uses for extended reasoning that should remain private.

Here is how each case is handled:

| Thinking block case | Same model | Different model |
|---|---|---|
| Redacted (`block.redacted === true`) | Keep as-is | **Drop** (opaque blob is meaningless to another provider) |
| Non-empty text with `thinkingSignature` | Keep as-is (needed for replay) | Keep as-is (same model branch handles this) |
| Non-empty text, no signature, same model | Keep as-is | Convert to plain `{ type: "text", text: block.thinking }` |
| Empty or whitespace-only thinking text | Drop | Drop |

The key insight for the cross-provider case: we do not discard the reasoning — we *demote* it to a plain text block. The new provider sees the thinking as an ordinary text turn, which is always valid. Information is preserved; format incompatibility is avoided.

```ts
// From transform-messages.ts — thinking block handling (annotated)
if (block.type === "thinking") {
  // Redacted thinking is encrypted — meaningless to another provider; drop it.
  if (block.redacted) {
    return isSameModel ? block : [];
  }
  // Non-empty thinking with signature and same model: keep (needed for replay).
  if (isSameModel && block.thinkingSignature) return block;
  // Empty thinking: drop unconditionally.
  if (!block.thinking || block.thinking.trim() === "") return [];
  // Same model with non-empty text (no signature): keep.
  if (isSameModel) return block;
  // Different model with non-empty thinking: demote to plain text.
  return { type: "text" as const, text: block.thinking };
}
```

Notice the empty-check (`block.thinking.trim() === ""`). Some providers — particularly those using encrypted reasoning — emit a thinking block whose text is empty but whose `thinkingSignature` is present. There is nothing useful to demote in that case, so we drop it rather than emit an empty text block.

### Tool-call `thoughtSignature` removal

Related to thinking: when a `toolCall` block was recorded with a `thoughtSignature` (a link back to the reasoning that produced this tool invocation), that signature is only meaningful to the originating model. For a different model, it is noise at best and an API error at worst. The function strips it:

```ts
// Cross-model: drop thoughtSignature from tool calls
if (!isSameModel && toolCall.thoughtSignature) {
  normalizedToolCall = { ...toolCall };
  delete (normalizedToolCall as { thoughtSignature?: string }).thoughtSignature;
}
```

### Tool-call ID normalisation

If the caller supplied `normalizeToolCallId`, the function applies it to each `toolCall` block from a different-model assistant turn:

```ts
if (!isSameModel && normalizeToolCallId) {
  const normalizedId = normalizeToolCallId(toolCall.id, model, assistantMsg);
  if (normalizedId !== toolCall.id) {
    toolCallIdMap.set(toolCall.id, normalizedId);           // remember old → new
    normalizedToolCall = { ...normalizedToolCall, id: normalizedId };
  }
}
```

The `toolCallIdMap` is then used when we encounter the corresponding `toolResult` message — the tool-result must reference the same ID as the call, so both must be rewritten consistently:

```ts
if (msg.role === "toolResult") {
  const normalizedId = toolCallIdMap.get(msg.toolCallId);
  if (normalizedId && normalizedId !== msg.toolCallId) {
    return { ...msg, toolCallId: normalizedId };
  }
  return msg;
}
```

This paired rewrite keeps the tool-call/tool-result pairing intact even after ID normalisation.

### User messages and image downgrade

User messages are passed through unchanged in the first-pass map. However, before the map runs, the function calls `downgradeUnsupportedImages`:

```ts
const imageAwareMessages = downgradeUnsupportedImages(messages, model);
```

If the target model does not list `"image"` in its `model.input` capabilities, every image block in user messages and tool results is replaced by a text placeholder:

- User-message image → `"(image omitted: model does not support images)"`
- Tool-result image → `"(tool image omitted: model does not support images)"`

Consecutive images are collapsed to a single placeholder (the internal `previousWasPlaceholder` flag prevents duplicate placeholders from appearing side-by-side).

## Pass 2 — structural cleanup: aborted turns and orphaned tool calls

The second pass enforces two API-level structural rules that apply regardless of provider.

### Dropping aborted and errored assistant turns

An assistant message with `stopReason === "error"` or `stopReason === "aborted"` is an incomplete turn: the model may have produced partial reasoning without a following message, or started tool calls it never finished. Replaying those turns to any provider — even the same one — tends to cause API errors (for example, OpenAI requires that reasoning content always be followed by another item).

The second pass skips these messages entirely:

```ts
if (assistantMsg.stopReason === "error" || assistantMsg.stopReason === "aborted") {
  continue;  // Drop from output — let the model retry from the last valid state
}
```

### Synthetic tool results for orphaned tool calls

Sometimes a turn ends mid-flight: the assistant emitted tool calls, but the conversation was interrupted before the results arrived — perhaps the session was aborted right after the tool-call turn. The next provider would receive an assistant turn with tool calls followed by nothing, which most providers reject (they expect every tool call to have a corresponding result).

The second pass detects this by tracking `pendingToolCalls` as it walks the output array:

1. When it sees an assistant turn with tool calls, it records them in `pendingToolCalls`.
2. As it sees `toolResult` messages, it marks those IDs as satisfied.
3. If it encounters another assistant turn — or a user message — before all pending calls are satisfied, it synthesises a placeholder result for each unsatisfied call:

```ts
// Synthetic result for an orphaned tool call
{
  role: "toolResult",
  toolCallId: tc.id,
  toolName: tc.name,
  content: [{ type: "text", text: "No result provided" }],
  isError: true,
  timestamp: Date.now(),
}
```

The synthetic result is marked `isError: true` and carries the text `"No result provided"`. This tells the model something went wrong — it should not hallucinate a result — while satisfying the API's structural requirement that every tool call has an answer.

This same logic also runs at the end of the pass: if the conversation ends with unresolved tool calls (e.g. the very last assistant turn contained a tool call that was never answered), synthetic results are emitted for those too.

## What the cross-provider replay tests verify

Knowing *how* the transform works is one thing; knowing it holds in practice across a diverse set of real providers is another. The `cross-provider-handoff.test.ts` test file (S23) establishes this empirically.

### The test strategy

The test suite maintains a list of provider/model pairs — including Anthropic, Google, OpenAI (both Completions and Responses APIs), Azure, OpenAI Codex, GitHub Copilot variants, Amazon Bedrock, xAI, Cerebras, Cloudflare, Groq, Hugging Face, Together AI, Mistral, MiniMax, and others.

For each pair that has credentials available, the suite:

1. **Generates a fixture context.** It sends the model a user message asking it to use a tool (`double_number`), collects the assistant's response (which may include thinking blocks, tool calls with provider-native IDs, etc.), feeds back the tool result, and collects the final response. This produces four messages: user → assistant (with tool call) → tool result → assistant (final).

2. **Tests cross-provider handoff.** For each target provider/model, it takes all *other* providers' fixture messages, concatenates them into one history, appends a fresh user message (`"just say 'Hello, handoff successful!'"`) and sends the whole thing through `completeSimple` — which internally calls `transformMessages` before building the API payload.

3. **Asserts no failures.** Every target must complete without an API error. A single failure fails the whole test.

The fixture generation timeout is 300 seconds; the handoff test itself runs up to 600 seconds, reflecting that it makes many real API calls in sequence.

### What survives a provider switch

The test is designed so that fixtures include the hard cases: thinking blocks (produced by thinking-capable models), tool call IDs in various formats (including the 450+ char IDs from the OpenAI Responses API), and — if the session was aborted — incomplete turns. After `transformMessages`:

- Thinking blocks from provider A become plain text blocks when replayed to provider B.
- Redacted/encrypted thinking from provider A is dropped entirely for provider B.
- Oversized or format-violating tool-call IDs are normalised by the caller-supplied `normalizeToolCallId` function.
- Aborted assistant turns vanish from the history, so provider B never sees them.
- Orphaned tool calls receive synthetic error results, satisfying every provider's structural requirement.

### Why the fixture is generated fresh each run

The test does not cache fixtures to disk between runs. This is deliberate: tool-call ID formats, thinking-block shapes, and model capabilities can change as providers release updates. Generating fresh fixtures on each run ensures the test catches regressions against current provider behaviour, not a snapshot from six months ago.

## Connecting this to model switching in the agent

This transform is what makes `/model` switching mid-session safe. When the user issues such a command (see the coding-agent's slash-command handling for the implementation detail), the harness replays the accumulated session history through `transformMessages` targeted at the new model before sending the next turn. From the model's perspective it receives a clean, structurally valid history in its own preferred format — even if that history was built up across three different providers.

Two things are worth noting about the design choice here:

**Information is preserved where possible.** Thinking text is demoted, not discarded. A future turn can still reason about what was thought earlier, because the text is still there — only no longer in a `thinking`-typed block that could trigger API errors.

**Structural validity takes priority.** Aborted turns and orphaned tool calls are cleaned up unconditionally, even for same-model replays where the content would technically be accepted. A broken turn in the middle of a history is a latent bug regardless of which model sees it.

---

← Previous: [The Agent Harness: Compaction, Session Storage, and Skills](./harness-session-and-compaction.md) · Next: [The TUI Class and Differential Render Engine](../terminal-ui/the-tui-class-and-render-engine.md) →
