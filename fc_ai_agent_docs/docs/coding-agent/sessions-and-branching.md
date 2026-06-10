---
title: "Sessions, Branching, and the Session Tree"
description: "Build the session storage model — JSONL append-only format, session trees with parentId branching, fork and clone operations, and compaction labels for navigating long conversation histories."
category: coding-agent
type: tutorial
tags: [session manager, JSONL, parentId, branching, fork, clone, session tree, compaction label, cwd discovery, coding-agent, session, history, navigation]
keywords: [session tree, JSONL storage, fork, branch, conversation history, append-only log]
sources: [S29, S30]
---

**TL;DR** — Conversations aren't linear. You might ask the agent to try approach A, then fork and try approach B, then merge back to A. We'll build a session tree where every conversation is a node with a `parentId`, stored as an append-only JSONL file. Forking copies the history up to a point. Branching creates a new timeline. Compaction labels mark where summaries live in the tree.

## The session tree model

Each session is a node in a tree:

```ts
interface Session {
  id: string;
  parentId?: string;      // which session this branched from
  forkPoint?: number;     // message index where the fork happened
  createdAt: number;
  updatedAt: number;
  cwd: string;
  modelId: string;
  branchName?: string;    // human-readable label
}

interface SessionEntry {
  type: "message" | "compaction" | "fork" | "branch" | "metadata";
  timestamp: number;
  messageIndex: number;   // monotonically increasing within a session
  data: unknown;
}
```

## JSONL storage format

Each session is a single `.jsonl` file. Lines are appended, never modified:

```jsonl
{"type":"metadata","timestamp":1717800000000,"messageIndex":0,"data":{"cwd":"/project","modelId":"claude-sonnet-4-6","parentId":"abc123","forkPoint":42}}
{"type":"message","timestamp":1717800001000,"messageIndex":1,"data":{"role":"user","content":"Add dark mode toggle"}}
{"type":"message","timestamp":1717800002000,"messageIndex":2,"data":{"role":"assistant","content":[...]}}
{"type":"compaction","timestamp":1717800100000,"messageIndex":50,"data":{"cutPoint":10,"summary":"The user asked to add dark mode..."}}
```

The append-only format has several advantages:
- **Crash-safe** — the last write might be partial, but previous entries are intact
- **Git-friendly** — diffs are meaningful (one change = one line)
- **Streamable** — can be read line-by-line without loading the whole file

## Session Manager

Create `packages/coding-agent/src/core/session-manager.ts`:

```ts
class SessionManager {
  constructor(private sessionsDir: string) {}

  async create(params: {
    cwd: string;
    modelId: string;
    parentId?: string;
    forkPoint?: number;
    branchName?: string;
  }): Promise<Session> {
    const id = uuidv7();
    const session: Session = {
      id,
      parentId: params.parentId,
      forkPoint: params.forkPoint,
      createdAt: Date.now(),
      updatedAt: Date.now(),
      cwd: params.cwd,
      modelId: params.modelId,
      branchName: params.branchName,
    };

    // Write metadata entry
    await this.appendEntry(id, {
      type: "metadata",
      timestamp: Date.now(),
      messageIndex: 0,
      data: session,
    });

    return session;
  }

  async fork(sessionId: string, forkPoint: number, branchName?: string): Promise<Session> {
    const parent = await this.load(sessionId);
    const entries = await this.loadEntries(sessionId);

    // Create new session with parentId pointing to the original
    const child = await this.create({
      cwd: parent.cwd,
      modelId: parent.modelId,
      parentId: sessionId,
      forkPoint,
      branchName,
    });

    // Copy entries up to the fork point
    for (const entry of entries) {
      if (entry.messageIndex <= forkPoint) {
        await this.appendEntry(child.id, { ...entry, messageIndex: entry.messageIndex });
      }
    }

    // Record the fork
    await this.appendEntry(sessionId, {
      type: "fork",
      timestamp: Date.now(),
      messageIndex: -1,
      data: { childId: child.id, forkPoint },
    });

    return child;
  }

  async getTree(sessionId: string): Promise<SessionTree> {
    const root = await this.load(this.findRoot(sessionId));
    const children = await this.listChildren(root.id);
    return {
      session: root,
      children: await Promise.all(children.map(c => this.getTree(c.id))),
    };
  }

  private async loadEntries(sessionId: string): Promise<SessionEntry[]> {
    const filePath = path.join(this.sessionsDir, `${sessionId}.jsonl`);
    try {
      const content = await fs.promises.readFile(filePath, "utf-8");
      return content.trim().split("\n").map(line => JSON.parse(line));
    } catch {
      return [];
    }
  }

  private async appendEntry(sessionId: string, entry: SessionEntry): Promise<void> {
    const filePath = path.join(this.sessionsDir, `${sessionId}.jsonl`);
    await fs.promises.appendFile(filePath, JSON.stringify(entry) + "\n", "utf-8");
  }
}
```

## Branch navigation

When the agent switches branches, it replays entries from the target session:

```ts
async switchBranch(sessionId: string): Promise<void> {
  // Save current session
  await this.saveCurrentSession();

  // Load target session entries
  const entries = await this.sessionManager.loadEntries(sessionId);

  // Reset agent context
  this.harness.agent.reset();

  // Replay entries to reconstruct the conversation
  for (const entry of entries) {
    if (entry.type === "message") {
      this.harness.agent.context.messages.push(entry.data);
    } else if (entry.type === "compaction") {
      // Inject compaction summary at the recorded cut point
      compactAtPoint(this.harness.agent.context, entry.data.cutPoint, entry.data.summary);
    }
  }

  this.currentSessionId = sessionId;
}
```

## Compaction labels

When compaction happens, a label entry is written that records where the summary was injected. This preserves the tree structure even through compaction:

```ts
await this.appendEntry(sessionId, {
  type: "compaction",
  timestamp: Date.now(),
  messageIndex: cutPoint,
  data: { cutPoint, summary, removedCount: messagesRemoved },
});
```

## What we've built

- **Session tree** with parent-child relationships via `parentId`
- **JSONL append-only storage** — crash-safe, git-friendly, streamable
- **Fork and branch** operations that copy history to a fork point
- **Compaction labels** that preserve tree structure through summarization

---

← Previous: [System Prompt Construction and Skill Loading](./system-prompt-and-skills.md) · Next: [Context Compaction and Branch Summarization](./compaction-and-summarization.md) →
