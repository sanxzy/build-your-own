---
title: "The Agent Harness: Compaction, Sessions, and Skills"
description: "Build the AgentHarness — a reusable building block that layers context compaction, JSONL session persistence, system prompt construction from templates, and skill loading on top of the Agent class."
category: agent-core
type: tutorial
tags: [AgentHarness, compaction, prepareCompaction, compact, shouldCompact, token estimation, summary generation, session storage, JSONL, session tree, system prompt, skills, SKILL.md, agent-core, harness, context window]
keywords: [AgentHarness, context compaction, token estimation, JSONL session, branch management, skill loading, system prompt templates]
sources: [S22, S27, S28, S29, S30, S25, S26, S31]
---

**TL;DR** — The Agent class handles the loop. The AgentHarness adds production concerns: **context compaction** (what happens when the conversation outgrows the model's context window), **JSONL session persistence** (saving and loading conversations), **system prompt construction** (assembling prompts from templates with tools, skills, and guidelines), and **skill loading** (discovering and injecting SKILL.md files). These are the features that separate a demo from a production agent.

## The problem: context windows are finite

A conversation with an AI coding agent can run for hundreds of turns — far exceeding any model's context window. When the window fills up, you have two choices: stop (lose all context) or **compact** (summarize the oldest turns to free up space).

Compaction is lossy but essential. We'll build a system that:
1. Estimates token usage to know when compaction is needed
2. Finds the right cut point in the conversation
3. Generates a summary of the truncated prefix
4. Injects the summary as a synthetic system message

## Token estimation

We can't count tokens exactly without access to the model's tokenizer, but we can estimate well enough for compaction decisions:

```ts
export function estimateTokens(text: string): number {
  // Conservative estimate: ~4 characters per token for English text
  // Code is denser at ~3 chars/token, but we use 4 to be safe
  return Math.ceil(text.length / 4);
}

export function estimateContextTokens(context: AgentContext): number {
  let tokens = estimateTokens(context.systemPrompt);
  for (const msg of context.messages) {
    tokens += estimateTokens(JSON.stringify(msg.content));
  }
  for (const tool of context.tools) {
    tokens += estimateTokens(tool.name + tool.description);
  }
  return tokens;
}

export function shouldCompact(
  context: AgentContext,
  model: Model,
  threshold?: number,
): boolean {
  const used = estimateContextTokens(context);
  const limit = threshold ?? Math.floor(model.contextWindow * 0.8);
  return used > limit;
}
```

We compact at 80% of the context window by default, leaving 20% headroom for the model's response.

## Finding the cut point

Compaction removes the oldest messages but must preserve conversation integrity. We can't cut in the middle of a tool call/result pair:

```ts
export function findCutPoint(
  messages: AgentMessage[],
  targetTokens: number,
): number {
  let accumulated = 0;
  for (let i = 0; i < messages.length; i++) {
    accumulated += estimateTokens(JSON.stringify(messages[i].content));
    if (accumulated >= targetTokens) {
      // Walk forward to a safe boundary:
      // after a complete assistant + toolResult cycle
      return findSafeBoundary(messages, i);
    }
  }
  return messages.length; // shouldn't compact
}
```

A safe boundary is after an assistant message that didn't request tools, or after tool results when the next message is from the user. We never cut between a tool call and its result — that would confuse the LLM.

## Generating a summary

Once we've identified the prefix to remove, we generate a summary using a cheaper/faster model:

```ts
export async function generateSummary(
  messages: AgentMessage[],
  streamFn: StreamFn,
): Promise<string> {
  const conversation = serializeConversation(messages);
  const response = await streamFn(smallModel, {
    systemPrompt: "Summarize this conversation. Include key decisions, file changes, and open questions.",
    messages: [{ role: "user", content: conversation, timestamp: Date.now() }],
  });
  const result = await response.result();
  return result.content.find(b => b.type === "text")?.text ?? "";
}
```

The summary is injected as a synthetic system message at the compaction boundary:

```ts
export function compact(
  context: AgentContext,
  cutPoint: number,
  summary: string,
): void {
  const removed = context.messages.splice(0, cutPoint);
  context.messages.unshift({
    role: "user",
    content: `[Previous conversation summary]\n${summary}`,
    timestamp: Date.now(),
  });
}
```

## JSONL session persistence

Sessions are stored as JSONL (JSON Lines) files — one JSON object per line, append-only. This format is human-readable, git-friendly, and trivially parseable:

```ts
// packs/agent-core/src/harness/session/jsonl-storage.ts
export class JsonlSessionStorage {
  constructor(private baseDir: string) {}

  async saveEntry(sessionId: string, entry: SessionEntry): Promise<void> {
    const path = this.sessionPath(sessionId);
    const line = JSON.stringify(entry) + "\n";
    await fs.promises.appendFile(path, line, "utf-8");
  }

  async loadEntries(sessionId: string): Promise<SessionEntry[]> {
    const path = this.sessionPath(sessionId);
    try {
      const content = await fs.promises.readFile(path, "utf-8");
      return content.trim().split("\n").map(line => JSON.parse(line));
    } catch {
      return [];
    }
  }

  async listSessions(): Promise<SessionInfo[]> {
    const files = await fs.promises.readdir(this.baseDir);
    return files
      .filter(f => f.endsWith(".jsonl"))
      .map(f => ({ id: f.replace(".jsonl", ""), path: path.join(this.baseDir, f) }));
  }

  private sessionPath(id: string): string {
    return path.join(this.baseDir, `${id}.jsonl`);
  }
}
```

Each entry in the JSONL file represents one conversation event: message added, compaction performed, branch created, etc. The append-only nature means we never lose data — sessions are an immutable log.

## Session branching

Sessions form a tree. When you fork a conversation, the new branch inherits the parent's history up to the fork point:

```ts
export interface SessionInfo {
  id: string;
  parentId?: string;
  createdAt: number;
  updatedAt: number;
  cwd: string;
  modelId: string;
  messageCount: number;
}

export interface SessionEntry {
  type: "message" | "compaction" | "branch" | "metadata";
  timestamp: number;
  data: unknown;
}
```

When the agent switches branches, the harness:
1. Saves the current branch state
2. Loads the target branch's entries
3. Replays entries to reconstruct the `AgentContext`
4. If the branch has compaction entries, injects summaries at the recorded cut points

## System prompt construction

The harness assembles the system prompt from templates:

```ts
export function buildSystemPrompt(params: {
  template: string;
  tools: AgentTool[];
  skills: SkillDefinition[];
  cwd: string;
  date: string;
  guidelines?: string;
}): string {
  return params.template
    .replace("{{tools}}", formatToolList(params.tools))
    .replace("{{skills}}", formatSkillList(params.skills))
    .replace("{{cwd}}", params.cwd)
    .replace("{{date}}", params.date)
    .replace("{{guidelines}}", params.guidelines ?? "");
}
```

The template defines the agent's personality, capabilities, and constraints. Tools and skills are injected as structured sections. The template uses `{{variable}}` substitution.

## Skill loading

Skills are YAML files (`SKILL.md`) that define reusable capabilities. The harness discovers them in multiple locations:

```ts
export function discoverSkills(paths: string[]): SkillDefinition[] {
  const skills: SkillDefinition[] = [];
  const seen = new Set<string>();

  for (const dir of paths) {
    for (const file of findSkillFiles(dir)) {
      const skill = parseSkillFile(file);
      if (!seen.has(skill.name)) {
        seen.add(skill.name);
        skills.push(skill);
      }
    }
  }

  return skills;
}
```

Skill files use YAML frontmatter for metadata and Markdown for instructions. The harness parses them and injects skill descriptions into the system prompt. When multiple skills with the same name exist, the first one found wins (local > project > global).

## The AgentHarness class

Putting it all together:

```ts
export class AgentHarness {
  readonly agent: Agent;
  private storage: JsonlSessionStorage;
  private skillPaths: string[];

  constructor(config: HarnessConfig) {
    this.storage = new JsonlSessionStorage(config.sessionDir);
    this.skillPaths = config.skillPaths ?? [];
    this.agent = new Agent({
      systemPrompt: buildSystemPrompt({
        template: config.systemPromptTemplate,
        tools: config.tools,
        skills: discoverSkills(this.skillPaths),
        cwd: config.cwd,
        date: new Date().toISOString().split("T")[0],
      }),
      tools: config.tools,
      model: config.model,
      queueMode: "all",
    });

    // Auto-compact before each turn
    this.agent.subscribe(async (event) => {
      if (event.type === "turn_start" && shouldCompact(this.agent.context, config.model)) {
        await this.compact();
      }
    });
  }

  async compact(): Promise<void> {
    const { context } = this.agent;
    const cutPoint = findCutPoint(context.messages, context.model.contextWindow * 0.5);
    const prefix = context.messages.slice(0, cutPoint);
    const summary = await generateSummary(prefix, streamSimple);
    compact(context, cutPoint, summary);
    await this.storage.saveEntry(context.sessionId, {
      type: "compaction",
      timestamp: Date.now(),
      data: { cutPoint, summary },
    });
  }

  async saveSession(): Promise<void> {
    for (const msg of this.agent.context.messages) {
      await this.storage.saveEntry(this.agent.context.sessionId!, {
        type: "message",
        timestamp: msg.timestamp,
        data: msg,
      });
    }
  }

  async loadSession(sessionId: string): Promise<void> {
    const entries = await this.storage.loadEntries(sessionId);
    for (const entry of entries) {
      if (entry.type === "message") {
        this.agent.context.messages.push(entry.data as AgentMessage);
      } else if (entry.type === "compaction") {
        compact(this.agent.context, entry.data.cutPoint, entry.data.summary);
      }
    }
  }
}
```

## What we've built

The AgentHarness wraps the Agent class with production-ready concerns:

- **Token estimation** and **shouldCompact** decision logic
- **Compaction** with safe cut-point detection and summary generation
- **JSONL session persistence** with append-only storage
- **Session branching** for conversation tree navigation
- **System prompt** assembly from templates with tool/skill injection
- **Skill discovery** with precedence-based loading

---

← Previous: [The Agent Class](./the-agent-class.md) · Next: [Cross-Provider Message Transforms](./cross-provider-message-transforms.md) →
