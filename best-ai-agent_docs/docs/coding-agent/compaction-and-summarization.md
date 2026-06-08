---
title: "Context Compaction and Branch Summarization"
description: "Implement the coding agent's compaction strategy — file-operation tracking for smarter cut points, conversation serialization, shouldCompact thresholds, and branch summarization for navigating the session tree."
category: coding-agent
type: tutorial
tags: [compaction, shouldCompact, file-op tracking, serialization, branch summary, synthetic message, context window, coding-agent, token budget]
keywords: [compaction strategy, file-operation tracking, branch summary, token budget management]
sources: [S27, S28]
---

**TL;DR** — The Agent Core has generic compaction. The coding agent extends it with domain-specific intelligence: tracking which files were modified so compaction preserves relevant context, smarter cut-point detection that avoids splitting edit sequences, and branch summarization that creates a synthetic context message when navigating to a different branch in the session tree.

## File-operation tracking

When a coding agent edits files, the compaction system should know. If we're about to truncate messages about `auth.ts`, but the agent just edited `auth.ts`, those recent messages are more valuable than older ones about `config.ts`.

```ts
interface FileOperation {
  path: string;
  operation: "read" | "write" | "edit" | "bash";
  timestamp: number;
  messageIndex: number;
}

class FileOpTracker {
  private operations: FileOperation[] = [];

  record(op: FileOperation): void {
    this.operations.push(op);
  }

  getRecentlyTouchedFiles(window: number): Set<string> {
    const cutoff = Date.now() - window;
    return new Set(
      this.operations
        .filter(op => op.timestamp > cutoff)
        .map(op => op.path),
    );
  }

  getFileOpMessages(filePath: string): number[] {
    return this.operations
      .filter(op => op.path === filePath)
      .map(op => op.messageIndex);
  }
}
```

The compaction system uses this to weight messages: messages referencing recently-touched files get a higher preservation score.

## Smart cut-point detection

The generic compaction's `findCutPoint` is extended with file-awareness:

```ts
function findSmartCutPoint(
  messages: AgentMessage[],
  targetTokens: number,
  fileTracker: FileOpTracker,
): number {
  const recentlyTouched = fileTracker.getRecentlyTouchedFiles(5 * 60 * 1000); // 5 min
  const scores = messages.map((msg, i) => {
    let score = estimateTokens(JSON.stringify(msg.content));
    // Messages referencing touched files cost more to cut
    if (referencesFiles(msg, recentlyTouched)) {
      score *= 2; // double the cost — harder to cut
    }
    // Assistant messages with tool calls cost more to cut
    if (msg.role === "assistant" && hasToolCalls(msg)) {
      score *= 3; // never split a tool call/result pair
    }
    return { index: i, score };
  });

  let accumulated = 0;
  for (const { index, score } of scores) {
    accumulated += score;
    if (accumulated >= targetTokens) {
      // Walk forward to a safe boundary
      return findSafeBoundary(messages, index);
    }
  }
  return messages.length;
}
```

## Branch summarization

When navigating the session tree, the agent needs context about what happened in branches it's leaving or entering. The `generateBranchSummary` function creates a synthetic message:

```ts
async function generateBranchSummary(
  branchSession: Session,
  entries: SessionEntry[],
): Promise<string> {
  const messages = entries.filter(e => e.type === "message");
  const userMessages = messages.filter(m => m.data.role === "user");
  const toolCalls = messages.filter(m =>
    m.data.role === "assistant" && hasToolCalls(m.data),
  );
  const compactions = entries.filter(e => e.type === "compaction");

  return [
    `## Branch: ${branchSession.branchName ?? branchSession.id.slice(0, 8)}`,
    `Created: ${new Date(branchSession.createdAt).toISOString()}`,
    `Messages: ${messages.length}`,
    ``,
    `### User requests`,
    ...userMessages.map(m => `- ${summarizeContent(m.data.content)}`),
    ``,
    `### Tools executed`,
    `${toolCalls.length} tool calls across ${countUniqueTools(toolCalls)} tools`,
    ``,
    compactions.length > 0
      ? `### Compactions\n${compactions.length} compaction(s) performed.`
      : "",
  ].filter(Boolean).join("\n");
}
```

The branch summary is injected as a context message when the agent switches branches:

```ts
// On branch switch:
const summary = await generateBranchSummary(previousBranch, entries);
this.harness.agent.context.messages.unshift({
  role: "user",
  content: `[Previous branch context]\n${summary}`,
  timestamp: Date.now(),
});
```

## Compaction thresholds

The coding agent uses tighter thresholds than the generic agent core:

```ts
const COMPACTION_THRESHOLDS = {
  warning: 0.7,   // 70% — start tracking for compaction
  compact: 0.8,   // 80% — compact before next turn
  critical: 0.9,  // 90% — compact immediately, even mid-turn
};
```

At `warning`, the agent starts preferring shorter responses. At `compact`, it compacts before the next turn. At `critical`, it pauses mid-turn to compact — rare but necessary for very long tool outputs.

## What we've built

- **File-operation tracking** that weights compaction decisions by which files were recently edited
- **Smart cut-point detection** that preserves edit sequences and tool call/result pairs
- **Branch summarization** that creates context messages for tree navigation
- **Tiered compaction thresholds** (warning, compact, critical)

---

← Previous: [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) · Next: [Model Registry, Settings, and Configuration](./model-registry-and-config.md) →
