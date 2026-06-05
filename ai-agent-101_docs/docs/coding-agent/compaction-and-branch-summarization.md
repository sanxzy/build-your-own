---
title: "Coding-Agent Compaction and Branch Summarization"
description: "How the coding agent tracks file operations during compaction and generates synthetic context messages when the user navigates the session tree."
category: coding-agent
type: tutorial
tags: [compaction, shouldCompact, file-op tracking, serialisation, branch summary, synthetic message, context window, coding-agent, compaction settings, token budget, CompactionSettings, CompactionDetails, BranchSummaryResult, collectEntriesForBranchSummary, generateBranchSummary, prepareCompaction, compact, findCutPoint, estimateContextTokens, keepRecentTokens, reserveTokens]
keywords: [context compaction, session summarization, file tracking, token overflow, context window management, agent memory, JSONL session, branch navigation, turn split, cut point]
sources: [S53, S54, S86]
---

**TL;DR** — Long coding sessions eventually push the context window to its limit. The coding agent solves this with two cooperating mechanisms: *compaction*, which summarises old turns while preserving a record of every file that was read or edited; and *branch summarization*, which synthesises a context message whenever the user navigates to a different branch of the session tree. By the end of this chapter you will understand how each mechanism decides when to run, what it captures, and how the two pieces fit together into one context-management strategy.

# Coding-Agent Compaction and Branch Summarization

## The problem with long sessions

Imagine you have been working with the agent for an hour. You have asked it to read configuration files, write new modules, debug a test failure, and refactor an interface. Every exchange — every user message, every assistant reply, every tool result — accumulates in the context window.

Language models have a fixed context window. Once the total token count of the conversation exceeds that window, the next request fails with a "prompt is too long" error. Even before the hard limit, a context window that is nearly full leaves the model little room to generate a useful reply — you are burning budget on history the model may not even need.

So we need to summarise. We run an LLM call that reads the old turns and writes a structured checkpoint, then we discard the old turns and inject that checkpoint in their place. The agent resumes from the checkpoint as if it had always had that summary.

But here is the complication unique to coding sessions: a naive summary might say "the user asked me to refactor the auth module" — but it will not necessarily say *which files were opened and which were changed*. If compaction silently drops that file list, the agent loses an important dimension of its working memory. The coding agent's compaction strategy solves this by explicitly tracking file operations alongside the prose summary.

> **Relationship to the general harness compaction.** The harness-level compaction (covered in [Harness Session and Compaction](../agent-loop/harness-session-and-compaction.md)) is a simpler, general-purpose mechanism that summarises conversation turns without any awareness of tool calls or file operations. The coding-agent compaction described here is built on top of that foundation and extends it with coding-specific file tracking and split-turn handling. If you have not read that chapter yet, a 1–2 sentence recap: the harness compaction monitors token usage after each assistant reply, and when usage crosses a threshold it runs a summarisation LLM call and replaces old entries in the session JSONL with a single compaction entry. The coding agent does the same, but adds the file-tracking layer described below.

---

## Step 1 — Deciding when to compact (`shouldCompact`)

Before we summarise anything, we need to know when to fire. Let's look at the threshold logic.

The compaction module exports a small group of settings that control all three dials:

```ts
// From src/core/compaction/compaction.ts (simplified view)
export interface CompactionSettings {
  enabled: boolean;
  reserveTokens: number;
  keepRecentTokens: number;
}

export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
};
```

| Field | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Master on/off switch. When `false`, auto-compaction never runs. |
| `reserveTokens` | `16384` | Tokens to keep free at the top of the context window. Compaction triggers before this headroom is consumed. |
| `keepRecentTokens` | `20000` | Approximate number of tokens from the *most recent* turns to keep verbatim after compaction. Older turns are replaced by the summary. |

The check itself is a single function:

```ts
export function shouldCompact(
  contextTokens: number,
  contextWindow: number,
  settings: CompactionSettings,
): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

In plain language: if `enabled` is false, do nothing. Otherwise, fire compaction when the current token count leaves less than `reserveTokens` tokens of free space in the window. The integration test confirms this logic:

```ts
// From test/suite/agent-session-compaction.test.ts
it("does not trigger threshold compaction below the threshold or when disabled", async () => {
  const belowThresholdHarness = await createHarness({
    settings: { compaction: { enabled: true, reserveTokens: 1000 } },
    models: [{ id: "faux-1", contextWindow: 200_000 }],
  });
  const disabledHarness = await createHarness({
    settings: { compaction: { enabled: false } },
  });

  // Neither harness triggers auto-compaction when well below threshold
  // or when compaction is explicitly disabled
  expect(belowThresholdSpy).not.toHaveBeenCalled();
  expect(disabledSpy).not.toHaveBeenCalled();
});
```

So far so good. Now let's look at how the agent estimates how full the context window actually is.

### Estimating token counts

The agent does not call the model's tokeniser — that would require an extra network round-trip. Instead, it uses two strategies depending on what it has available:

1. **Last assistant usage** — after every successful LLM reply, the API response includes an exact usage count. The agent picks the most recent non-aborted, non-errored assistant message and reads its `usage.totalTokens` (falling back to `usage.input + usage.output + usage.cacheRead + usage.cacheWrite` when the aggregate field is absent).

2. **Character-based estimate** — for messages that arrived after the last recorded usage (for example, a user message sent after the assistant's last reply), the agent estimates tokens as `Math.ceil(characterCount / 4)`. This overestimates slightly, making the heuristic conservative.

```ts
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
  const usageInfo = getLastAssistantUsageInfo(messages);

  if (!usageInfo) {
    // No assistant usage at all — estimate everything from character counts
    let estimated = 0;
    for (const message of messages) {
      estimated += estimateTokens(message);
    }
    return { tokens: estimated, usageTokens: 0, trailingTokens: estimated, lastUsageIndex: null };
  }

  const usageTokens = calculateContextTokens(usageInfo.usage);
  let trailingTokens = 0;
  for (let i = usageInfo.index + 1; i < messages.length; i++) {
    trailingTokens += estimateTokens(messages[i]);
  }

  return {
    tokens: usageTokens + trailingTokens,
    usageTokens,
    trailingTokens,
    lastUsageIndex: usageInfo.index,
  };
}
```

Notice the returned `ContextUsageEstimate` breaks down `tokens` into `usageTokens` (exact, from the API) and `trailingTokens` (estimated, from characters), so callers can distinguish precise from approximate parts. The integration tests exercise a specific edge case here: a pre-compaction assistant message that was already in the session when compaction fired should not double-count toward the next compaction threshold.

```ts
it("ignores stale pre-compaction assistant usage on pre-prompt checks", async () => {
  // ... sets up a session where the most recent recorded usage belongs to
  // a message that was already summarised in a previous compaction.
  // The check should not fire because that usage is from before the boundary.
  expect(runAutoCompactionSpy).not.toHaveBeenCalled();
});
```

---

## Step 2 — Finding where to cut (`findCutPoint`)

Once we know compaction should run, we need to decide which messages to summarise and which to keep verbatim. The goal is to preserve the most recent `keepRecentTokens` tokens of conversation as-is, and summarise everything older.

Finding the cut is more subtle than it sounds because we cannot cut arbitrarily mid-conversation. Specifically, we must never cut *between a tool call and its tool result* — the model requires tool results to immediately follow the tool call that requested them. So valid cut points are entries whose role is `user`, `assistant`, `bashExecution`, `custom`, or `branchSummary`; `toolResult` entries are never valid cut points.

```ts
// Simplified view of findCutPoint's core algorithm
export function findCutPoint(
  entries: SessionEntry[],
  startIndex: number,
  endIndex: number,
  keepRecentTokens: number,
): CutPointResult {
  const cutPoints = findValidCutPoints(entries, startIndex, endIndex);

  // Walk backwards from the newest entry, accumulating estimated token sizes
  let accumulatedTokens = 0;
  let cutIndex = cutPoints[0];

  for (let i = endIndex - 1; i >= startIndex; i--) {
    const entry = entries[i];
    if (entry.type !== "message") continue;

    accumulatedTokens += estimateTokens(entry.message);

    if (accumulatedTokens >= keepRecentTokens) {
      // Find the nearest valid cut point at or after this entry
      for (let c = 0; c < cutPoints.length; c++) {
        if (cutPoints[c] >= i) { cutIndex = cutPoints[c]; break; }
      }
      break;
    }
  }

  // Determine if we are cutting inside a turn (assistant message without its user pair)
  const cutEntry = entries[cutIndex];
  const isUserMessage = cutEntry.type === "message" && cutEntry.message.role === "user";
  const turnStartIndex = isUserMessage
    ? -1
    : findTurnStartIndex(entries, cutIndex, startIndex);

  return {
    firstKeptEntryIndex: cutIndex,
    turnStartIndex,
    isSplitTurn: !isUserMessage && turnStartIndex !== -1,
  };
}
```

The returned `CutPointResult` carries a flag `isSplitTurn`. A split turn happens when the nearest valid cut point falls inside an existing user-assistant exchange — for example, if the cut point is an assistant message that still has its user message behind the boundary. In that case, `turnStartIndex` points to the user message that opened the split turn, and the compaction pipeline generates a *second* summary specifically for the partial turn prefix before merging everything.

---

## Step 3 — File-operation tracking (`CompactionDetails`)

Here is the feature that distinguishes coding-agent compaction from the plain harness version. Every time we compact, we also walk the messages being discarded and extract two lists: files that were *read* and files that were *modified* (written or created).

These lists are stored in a typed details object:

```ts
export interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

The extraction is cumulative. When a session has been compacted more than once, the second compaction must not lose track of files that were mentioned in the *first* compaction's summary. So `extractFileOperations` begins by reading the previous compaction entry's `details` field and seeding the file sets from it:

```ts
// Simplified view of extractFileOperations
function extractFileOperations(
  messages: AgentMessage[],
  entries: SessionEntry[],
  prevCompactionIndex: number,
): FileOperations {
  const fileOps = createFileOps(); // { read: Set<string>, edited: Set<string>, written: Set<string> }

  // Seed from previous compaction's stored details (if native, not from an extension hook)
  if (prevCompactionIndex >= 0) {
    const prevCompaction = entries[prevCompactionIndex] as CompactionEntry;
    if (!prevCompaction.fromHook && prevCompaction.details) {
      const details = prevCompaction.details as CompactionDetails;
      for (const f of details.readFiles)    fileOps.read.add(f);
      for (const f of details.modifiedFiles) fileOps.edited.add(f);
    }
  }

  // Then scan tool calls in the messages being compacted now
  for (const msg of messages) {
    extractFileOpsFromMessage(msg, fileOps);
  }

  return fileOps;
}
```

Notice the `fromHook` check. Extensions can supply their own compaction summaries (we will see how in a moment). When they do, the native file-tracking logic skips reading their `details` because the format may differ from `CompactionDetails`.

After the file lists are assembled, they are formatted and appended to the prose summary text. The final `CompactionResult` carries them in `details`:

```ts
export interface CompactionResult<T = unknown> {
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  details?: T;
}

// Inside compact():
const { readFiles, modifiedFiles } = computeFileLists(fileOps);
summary += formatFileOperations(readFiles, modifiedFiles);

return {
  summary,
  firstKeptEntryId,
  tokensBefore,
  details: { readFiles, modifiedFiles } as CompactionDetails,
};
```

This means the next compaction can pick up where the previous one left off, building a running union of all files touched across the entire session, even if the raw messages that mentioned them have long since been discarded.

---

## Step 4 — Preparing and running the compaction

Now we have the building blocks: a cut point, a list of messages to summarise, and an accumulated file-ops set. The `prepareCompaction` function assembles them into a `CompactionPreparation` object, which is a pure value — no network calls, no side effects:

```ts
export interface CompactionPreparation {
  firstKeptEntryId: string;
  messagesToSummarize: AgentMessage[];
  turnPrefixMessages: AgentMessage[];
  isSplitTurn: boolean;
  tokensBefore: number;
  previousSummary?: string;     // summary from the last compaction, for iterative updates
  fileOps: FileOperations;
  settings: CompactionSettings;
}
```

The `previousSummary` field deserves attention. Rather than generating a summary from scratch every time, the summarisation prompt has two modes:

- **First compaction.** Uses `SUMMARIZATION_PROMPT` — a template that asks the LLM to write a structured checkpoint covering goal, progress, key decisions, and next steps.
- **Subsequent compactions.** Uses `UPDATE_SUMMARIZATION_PROMPT` — the same structured template, but prefaced with instructions to *preserve* the previous summary and *merge* new information into it. The previous summary is passed inside `<previous-summary>` tags.

This iterative update approach keeps the summary from growing unboundedly: each compaction produces a single summary of fixed maximum size, not a concatenation of all previous summaries.

The `compact()` function then calls the LLM and handles the split-turn case:

```ts
// Simplified view of compact()
export async function compact(
  preparation: CompactionPreparation,
  model: Model<any>,
  apiKey: string | undefined,
  // ...
): Promise<CompactionResult> {
  const { messagesToSummarize, turnPrefixMessages, isSplitTurn, fileOps } = preparation;

  let summary: string;

  if (isSplitTurn && turnPrefixMessages.length > 0) {
    // Generate both summaries in parallel
    const [historyResult, turnPrefixResult] = await Promise.all([
      generateSummary(messagesToSummarize, /* ... */),
      generateTurnPrefixSummary(turnPrefixMessages, /* ... */),
    ]);
    // Merge: main history + a "Turn Context (split turn)" section
    summary = `${historyResult}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult}`;
  } else {
    summary = await generateSummary(messagesToSummarize, /* ... */);
  }

  // Append file operation lists to the prose summary
  const { readFiles, modifiedFiles } = computeFileLists(fileOps);
  summary += formatFileOperations(readFiles, modifiedFiles);

  return { summary, firstKeptEntryId: preparation.firstKeptEntryId,
           tokensBefore: preparation.tokensBefore,
           details: { readFiles, modifiedFiles } };
}
```

The turn-prefix summary uses a smaller token budget (`Math.floor(0.5 * reserveTokens)` versus `Math.floor(0.8 * reserveTokens)` for the main summary) because it only needs to cover the partial turn prefix, not the full history.

### Extensions can override the summary

The test suite demonstrates another path: an extension can listen for the `session_before_compact` event and return its own `compaction` object, bypassing the default summarisation entirely:

```ts
// From test/suite/agent-session-compaction.test.ts
it("manually compacts using an extension-provided summary", async () => {
  const harness = await createHarness({
    extensionFactories: [
      (ext) => {
        ext.on("session_before_compact", async (event) => ({
          compaction: {
            summary: "summary from extension",
            firstKeptEntryId: event.preparation.firstKeptEntryId,
            tokensBefore: event.preparation.tokensBefore,
            details: { source: "extension" },
          },
        }));
      },
    ],
  });

  const result = await harness.session.compact();
  expect(result.summary).toBe("summary from extension");
});
```

When an extension takes over, its `details` is stored as-is and the `fromHook` flag is set on the resulting session entry (the flag name comes from an older design and is retained for session-file compatibility). Native compaction skips reading `fromHook` entries during file-list accumulation, as shown in the `extractFileOperations` code above.

### What auto-compaction looks like end to end

The integration harness shows the full auto-compaction lifecycle. A session is seeded with messages, then `_runAutoCompaction("threshold", false)` fires, and the test verifies that one compaction entry appears in the session and the first message in the resulting context has role `compactionSummary`:

```ts
it("auto-compacts with a custom streamFn when registry auth is absent", async () => {
  const harness = await createHarness({ withConfiguredAuth: false });
  seedCompactableSession(harness);
  const getStreamCallCount = useSummaryStreamFn(harness, "auto summary from custom stream");

  await sessionInternals._runAutoCompaction("threshold", false);

  const compactionEntries = harness.sessionManager
    .getEntries()
    .filter((e) => e.type === "compaction");
  expect(compactionEntries).toHaveLength(1);
  expect(getStreamCallCount()).toBe(1);
});
```

Notice the custom `streamFn` — even when no registry auth is configured, a caller-supplied stream function lets compaction proceed. This is how embedders that provide their own model access (rather than relying on the built-in API-key registry) can still get compaction.

---

## Step 5 — What happens when the user navigates the session tree (branch summarization)

So far we have looked at compaction within a single linear session. Now let's introduce the second mechanism.

Recall from [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) that the session tree is a persistent JSONL structure where each entry has a `parentId`. The user can navigate to any node in that tree, effectively "checking out" a different conversational branch. When that happens, the active context switches to the messages on the new branch — but the messages on the *old* branch are no longer in view.

We now have a problem: the agent's context loses all memory of what happened on the old branch. If the user was mid-task on branch A and jumps to branch B, then back to branch A, the agent on branch A will have no memory of the work done before the navigation.

Branch summarization solves this. Before the navigation completes, the system generates a summary of the branch being *left*, and injects that summary as a synthetic message into the session entry for the target branch. When the user later looks at branch A again, they find a `branch_summary` entry at the top of the context explaining what happened on branch B during the detour.

### Collecting the entries to summarise

The first function is `collectEntriesForBranchSummary`. It takes the session manager, the current leaf node id (`oldLeafId`), and the target node id (`targetId`), then walks the tree to find which entries belong exclusively to the old branch (i.e., entries that are on the path from `oldLeafId` to the common ancestor with `targetId`, but not on the target path):

```ts
// From src/core/compaction/branch-summarization.ts (simplified)
export function collectEntriesForBranchSummary(
  session: ReadonlySessionManager,
  oldLeafId: string | null,
  targetId: string,
): CollectEntriesResult {
  if (!oldLeafId) {
    return { entries: [], commonAncestorId: null };
  }

  // Build set of all node ids on the current (old) path
  const oldPath = new Set(session.getBranch(oldLeafId).map((e) => e.id));

  // Walk the target path root-first, find the deepest node that also appears on the old path
  const targetPath = session.getBranch(targetId);
  let commonAncestorId: string | null = null;
  for (let i = targetPath.length - 1; i >= 0; i--) {
    if (oldPath.has(targetPath[i].id)) {
      commonAncestorId = targetPath[i].id;
      break;
    }
  }

  // Collect entries from old leaf back to (but not including) the common ancestor
  const entries: SessionEntry[] = [];
  let current: string | null = oldLeafId;
  while (current && current !== commonAncestorId) {
    const entry = session.getEntry(current);
    if (!entry) break;
    entries.push(entry);
    current = entry.parentId;
  }

  // Reverse to restore chronological order
  entries.reverse();

  return { entries, commonAncestorId };
}
```

Two design choices worth noting:

1. **Compaction entries are included, not skipped.** Unlike the regular compaction pipeline, `collectEntriesForBranchSummary` does not stop at compaction boundaries — it walks right through them. Their summaries become part of the context handed to the branch summariser, providing cumulative history even for long branches.

2. **Tool results are skipped later.** When the collected entries are converted to `AgentMessage` objects in `prepareBranchEntries`, `toolResult` messages are omitted. The relevant context is already carried in the assistant message that made the tool call.

### Preparing entries with a token budget

The branch entries then go through `prepareBranchEntries`, which walks them from newest to oldest and adds messages until it hits a token budget:

```ts
export function prepareBranchEntries(
  entries: SessionEntry[],
  tokenBudget: number = 0,
): BranchPreparation {
  const messages: AgentMessage[] = [];
  const fileOps = createFileOps();
  let totalTokens = 0;

  // First pass: collect cumulative file ops from any nested branch_summary entries
  // (only from native summaries, not extension-generated ones)
  for (const entry of entries) {
    if (entry.type === "branch_summary" && !entry.fromHook && entry.details) {
      const details = entry.details as BranchSummaryDetails;
      for (const f of details.readFiles)    fileOps.read.add(f);
      for (const f of details.modifiedFiles) fileOps.edited.add(f);
    }
  }

  // Second pass: walk newest-to-oldest, add messages until budget is hit
  for (let i = entries.length - 1; i >= 0; i--) {
    const entry = entries[i];
    const message = getMessageFromEntry(entry);
    if (!message) continue;

    extractFileOpsFromMessage(message, fileOps);

    const tokens = estimateTokens(message);

    if (tokenBudget > 0 && totalTokens + tokens > tokenBudget) {
      // Exception: compaction or branch_summary entries may be added even slightly over budget
      // if they fit within 90% of remaining space — they carry essential context
      if (entry.type === "compaction" || entry.type === "branch_summary") {
        if (totalTokens < tokenBudget * 0.9) {
          messages.unshift(message);
          totalTokens += tokens;
        }
      }
      break;
    }

    messages.unshift(message);
    totalTokens += tokens;
  }

  return { messages, fileOps, totalTokens };
}
```

The token budget is `contextWindow - reserveTokens`, where `reserveTokens` defaults to `16384` if not specified in the options.

### Generating the branch summary message

With the messages collected, `generateBranchSummary` serialises the conversation to text, builds the prompt, and calls the model:

```ts
export async function generateBranchSummary(
  entries: SessionEntry[],
  options: GenerateBranchSummaryOptions,
): Promise<BranchSummaryResult> {
  const {
    model, apiKey, headers, signal,
    customInstructions, replaceInstructions,
    reserveTokens = 16384, streamFn,
  } = options;

  const tokenBudget = (model.contextWindow || 128000) - reserveTokens;
  const { messages, fileOps } = prepareBranchEntries(entries, tokenBudget);

  if (messages.length === 0) {
    return { summary: "No content to summarize" };
  }

  // Convert to LLM-compatible messages, then serialise to plain text
  // (serialisation prevents the model from treating the history as a conversation to continue)
  const llmMessages = convertToLlm(messages);
  const conversationText = serializeConversation(llmMessages);

  // Build the prompt
  let instructions = replaceInstructions && customInstructions
    ? customInstructions
    : customInstructions
      ? `${BRANCH_SUMMARY_PROMPT}\n\nAdditional focus: ${customInstructions}`
      : BRANCH_SUMMARY_PROMPT;

  const promptText = `<conversation>\n${conversationText}\n</conversation>\n\n${instructions}`;

  // Call LLM with maxTokens: 2048
  const response = streamFn
    ? await (await streamFn(model, context, requestOptions)).result()
    : await completeSimple(model, context, requestOptions);

  if (response.stopReason === "aborted") return { aborted: true };
  if (response.stopReason === "error")   return { error: response.errorMessage || "Summarization failed" };

  let summary = response.content
    .filter((c) => c.type === "text")
    .map((c) => c.text)
    .join("\n");

  // Prepend a preamble so the agent knows this is a branch summary, not primary history
  summary = BRANCH_SUMMARY_PREAMBLE + summary;

  // Append cumulative file operation lists
  const { readFiles, modifiedFiles } = computeFileLists(fileOps);
  summary += formatFileOperations(readFiles, modifiedFiles);

  return { summary, readFiles, modifiedFiles };
}
```

The `BRANCH_SUMMARY_PREAMBLE` is a short header prepended to every generated summary:

```
The user explored a different conversation branch before returning here.
Summary of that exploration:

```

This preamble tells the model reading the summary that the context it is about to see came from a navigation event, not from the current linear history. The `BRANCH_SUMMARY_PROMPT` itself follows the same structured format as the regular compaction prompt (Goal, Constraints, Progress, Key Decisions, Next Steps), so the model receives consistent context regardless of whether it came from a linear compaction or a branch navigation.

The `GenerateBranchSummaryOptions` interface exposes two customisation hooks:

| Field | Type | Effect |
|---|---|---|
| `customInstructions` | `string \| undefined` | Appended to the default prompt as "Additional focus: ..." |
| `replaceInstructions` | `boolean \| undefined` | When `true`, `customInstructions` replaces the default prompt entirely |
| `reserveTokens` | `number` (default 16384) | Tokens kept free for prompt + LLM response |
| `streamFn` | `StreamFn \| undefined` | Optional session stream function; bypasses the registry auth path |

### The result type

`generateBranchSummary` returns a `BranchSummaryResult`:

```ts
export interface BranchSummaryResult {
  summary?: string;      // The generated summary text (with preamble + file lists)
  readFiles?: string[];  // Files read on the summarised branch
  modifiedFiles?: string[];  // Files written/modified on the summarised branch
  aborted?: boolean;     // True when the AbortSignal fired before completion
  error?: string;        // Error message when LLM returned an error stopReason
}
```

The caller (the session manager) stores this result in the session tree as a `branch_summary` entry, and also records `readFiles` and `modifiedFiles` in the entry's `details` field. This means future branch summarisations will pick up those file lists via the `fromHook`-guarded first-pass in `prepareBranchEntries`.

---

## How the two mechanisms work together

Let's trace a complete scenario to see how compaction and branch summarization cooperate.

You start a session and ask the agent to read three files and edit two of them. The agent makes those tool calls, and the session accumulates several hundred tokens.

1. **You navigate to a sibling branch** (maybe to explore an alternative approach). Before the navigation completes, `collectEntriesForBranchSummary` walks from your current leaf back to the common ancestor, collecting the edit session's entries. `generateBranchSummary` generates a structured summary — including the file-operation lists — and a `branch_summary` entry is injected into the session at the new branch's leaf. The file lists (`readFiles`, `modifiedFiles`) are stored in the entry's `details`.

2. **You continue working on the new branch.** Tokens accumulate. Eventually `shouldCompact` fires. `prepareCompaction` builds the compaction preparation, and as part of `extractFileOperations` it reads the `branch_summary` entry's `details` to seed the file sets. The compaction summary captures both the new work *and* the file history from the navigation. The final `CompactionDetails` in the compaction entry carries the union of all file operations since the last compaction boundary.

3. **You navigate back to the original branch.** `collectEntriesForBranchSummary` now collects the entries from the new branch (including its compaction entry). `prepareBranchEntries` includes the compaction entry as a summary message (it does not stop at compaction boundaries), so the branch summary for this return navigation encompasses everything that happened on the detour.

At every step, file-operation state propagates forward through the chain of `details` fields, never silently dropping information even as the raw message history is pruned.

---

## Reference: compaction triggers and states

| Trigger | Description |
|---|---|
| `"threshold"` | Token count crossed `contextWindow - reserveTokens`. Normal proactive compaction. |
| `"overflow"` | LLM returned "prompt is too long" error. Emergency compaction attempted once. |
| `disabled` (`enabled: false`) | Auto-compaction is off. Manual `session.compact()` still works. |

| Compaction path | Who generates the summary |
|---|---|
| Default | `compact()` using the configured model |
| Extension override | `session_before_compact` hook returns a custom `compaction` object |
| Custom `streamFn` | Caller-supplied stream function; model still generates the summary |

| Condition | `_runAutoCompaction` retry behaviour |
|---|---|
| Overflow recovery fails | Does not retry — emits `compaction_end` with error message after one attempt |
| Threshold compaction fails | Emits `compaction_end` with error; session continues without compaction |

---

See also:
- [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) — the JSONL session tree and branch navigation this chapter builds on.
- [Harness Session and Compaction](../agent-loop/harness-session-and-compaction.md) — the general harness compaction pattern that the coding-agent variant extends.
- [Model Registry, Settings, and Resource Loading](./model-registry-and-settings.md) — where `CompactionSettings` is loaded from the settings file.

---

← Previous: [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) · Next: [Model Registry, Settings, and Resource Loading](./model-registry-and-settings.md) →
