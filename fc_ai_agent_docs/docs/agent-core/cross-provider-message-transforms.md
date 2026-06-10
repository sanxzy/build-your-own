---
title: "Cross-Provider Message Transforms"
description: "How thinking blocks and provider-specific content are transformed when switching between providers mid-conversation — essential for multi-model workflows."
category: agent-core
type: explanation
tags: [message transform, thinking block, cross-provider, provider handoff, content conversion, provider compatibility, agent-core]
keywords: [cross-provider, message transform, thinking handoff, provider migration, content normalization]
sources: [S16]
---

**TL;DR** — When you switch from one LLM provider to another mid-conversation, the message history contains provider-specific content (thinking blocks, special formats) that the new provider may not understand. We'll build a transformation layer that normalizes messages for the target provider, ensuring seamless provider handoff.

## Why provider switching matters

A coding agent might use different models for different tasks:

- **Claude Sonnet** for complex reasoning and code generation
- **GPT-5** for quick completions and refactoring
- **Gemini** for tasks requiring Google Search grounding

When the agent switches providers mid-session, the conversation history contains messages formatted for the previous provider. If we send Anthropic-formatted thinking blocks to OpenAI, the request fails. If we send OpenAI reasoning items to Google, they're rejected.

## The transform layer

Create `packages/agent-core/src/harness/transform-messages.ts` (this is part of the LLM Toolkit but the agent harness orchestrates when transforms are needed):

The `transformMessages` function converts a `Message[]` from one provider's format to another's:

```ts
export function transformMessages(
  messages: Message[],
  sourceProvider: Provider,
  targetProvider: Provider,
): Message[] {
  return messages.map(msg => {
    if (msg.role === "user" || msg.role === "toolResult") {
      return msg; // user and tool messages are provider-agnostic
    }

    if (msg.role === "assistant") {
      return transformAssistantMessage(msg, targetProvider);
    }

    return msg;
  });
}
```

## Thinking block transforms

The primary transformation is thinking content. Different providers represent internal reasoning differently:

| Source | Target | Transform |
|---|---|---|
| Anthropic `thinking` block | OpenAI | Convert to `<thinking>` delimited text |
| OpenAI `reasoning` item | Anthropic | Wrap in `thinking` content block |
| Google `thought` | OpenAI/Anthropic | Extract thought text, discard boolean marker |
| Any thinking | Provider without thinking support | Convert to text with `<thinking>` markers, or drop |

```ts
function transformAssistantMessage(
  msg: AssistantMessage,
  targetProvider: Provider,
): AssistantMessage {
  const transformedContent = msg.content.map(block => {
    if (block.type !== "thinking") return block;

    // Providers that natively support thinking: keep as-is
    if (supportsThinkingBlocks(targetProvider)) {
      return block;
    }

    // Providers that don't: convert to text with markers
    return {
      type: "text",
      text: `<thinking>\n${block.thinking}\n</thinking>`,
    } as TextContent;
  });

  return { ...msg, content: transformedContent };
}
```

## Redacted thinking

When a provider redacts thinking content (safety filters), we preserve the opaque signature:

```ts
if (block.redacted && block.thinkingSignature) {
  // Can't show the thinking, but preserve the signature for
  // multi-turn continuity with the original provider.
  // For cross-provider, drop it (the signature means nothing to another provider).
  return targetProvider === msg.provider
    ? block  // same provider, keep the signature
    : null;  // different provider, can't use the signature
}
```

## Tool call normalization

Tool calls are mostly standardized, but there are edge cases:

- **Google** attaches `thoughtSignature` to tool calls (an opaque string for thought context reuse). Non-Google providers ignore this field. Our unified `ToolCall` type preserves it for Google round-trips.
- **OpenAI Codex** may use a different tool call ID format. The transform layer normalizes IDs to our standard `string` type.
- **Anthropic** includes `cache_control` on content blocks within messages. When transforming to a non-Anthropic provider, these are stripped (they're meaningless to other APIs).

## When transforms happen

The agent harness applies transforms at two points:

1. **On session load.** When loading a session that was created with a different default provider, the harness transforms all historical messages.
2. **On model switch.** When the user changes the model mid-session, the harness transforms the message history before the next turn.

The transform is transparent to the user — the conversation continues seamlessly regardless of which provider is active.

## Limitations

Not everything survives a provider switch:

- **Thinking signatures** from Anthropic redaction are provider-specific and lost on switch.
- **Google Search grounding metadata** is Gemini-only.
- **Prompt cache markers** are provider-specific and must be rebuilt for the new provider.

The agent harness logs these losses at debug level so they're visible but not alarming.

## What we've built

The cross-provider transform layer ensures that the agent can use different models for different tasks without breaking the conversation. This is what makes a multi-model agent practical — the ability to route to the best model for each specific need without losing context.

This completes the Agent Core section. In the next section, we'll build the Terminal UI — the interface layer that makes the agent usable.

---

← Previous: [The Agent Harness](./harness-compaction-sessions.md) · Next: [The TUI Class and Differential Render Engine](../terminal-ui/tui-class-and-render-engine.md) →
