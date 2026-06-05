---
title: "The Agent Harness: Compaction, Session Storage, and Skills"
description: "How AgentHarness layers compaction, JSONL session I/O, system-prompt construction, and skill loading on top of the Agent class."
category: agent-loop
type: tutorial
tags: [AgentHarness, compaction, prepareCompaction, compact, DEFAULT_COMPACTION_SETTINGS, session storage, JSONL, session tree, system prompt, skills, SKILL.md, agent-core, harness, context window, InMemorySessionStorage, JsonlSessionStorage, InMemorySessionRepo, JsonlSessionRepo, buildSessionContext, Session, loadSkills, formatSkillsForSystemPrompt, CompactionSettings, CompactionPreparation, CompactionResult, shouldCompact, estimateContextTokens, findCutPoint, generateSummary]
keywords: [context compaction, conversation summarization, session branching, session persistence, system prompt builder, skill discovery, YAML frontmatter, token budget, keepRecentTokens, reserveTokens, JSONL log, agent harness pattern, reusable agent building block]
sources: [S29, S30, S31, S32, S33, S35]
---

**TL;DR** — The `Agent` class (covered in the previous chapter) runs the core loop, but a real assistant also needs to stop the context window from filling up, remember the conversation across restarts, build a coherent system prompt, and load reusable instructions from files. `AgentHarness` is the `agent-core` class that layers all four of those concerns on top of `Agent`. By the end of this chapter you'll understand exactly how each layer works and how to use `AgentHarness` in your own projects. Note: the full coding agent we build later takes a different composition path — it has its own `AgentSession` — but the harness pattern here is the general-purpose reusable building block the library provides.

# The Agent Harness: Compaction, Session Storage, and Skills

## The problem the harness solves

In [the previous chapter](./the-agent-class.md) we built an `Agent` — a stateful wrapper around the agent loop that handles steering messages, abort, and subscribers. That was already a lot. But run the agent for a while and a new set of problems surface:

1. **The context window fills up.** Every user message, every tool result, every assistant reply stays in the conversation. After hundreds of exchanges the model starts seeing an enormous message list, which hurts quality and eventually exceeds the model's limit.
2. **Nothing survives a restart.** The `Agent` holds messages in memory. Close the process, lose the conversation.
3. **Every agent needs a system prompt.** But building one correctly — injecting the current date, the working directory, the available tools, the loaded skills — is repetitive boilerplate.
4. **Skills are reusable instruction files.** They should be discoverable from disk, validated, and surfaced to the model automatically.

`AgentHarness` solves all four concerns. We'll build understanding of each piece in isolation, then see how `AgentHarness` ties them together.

---

## Part 1 — Compaction: keeping the context window from filling up

### The algorithm at a glance

The heart of compaction is in `src/harness/compaction/compaction.ts`. The idea is straightforward: when the context grows large, summarise the old portion of the conversation, replace it with that summary, and keep only the most recent messages verbatim. The model sees a compact history summary followed by fresh recent context, instead of the entire raw history.

Two functions do the work:

- **`prepareCompaction`** — inspects the session's current branch entries, decides where to cut, and returns a `CompactionPreparation` object describing exactly what will be summarised.
- **`compact`** — takes a `CompactionPreparation`, calls the model to write the summary, and returns a `CompactionResult` ready to be persisted.

### The token budget: `DEFAULT_COMPACTION_SETTINGS`

Every decision in compaction is driven by a `CompactionSettings` object. The library ships a default:

```ts
// Simplified view of DEFAULT_COMPACTION_SETTINGS from compaction.ts
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
};
```

| Field | Type | Default | Meaning |
|---|---|---|---|
| `enabled` | `boolean` | `true` | Whether automatic compaction decisions are active. |
| `reserveTokens` | `number` | `16384` | Tokens set aside for the summary prompt and its output. |
| `keepRecentTokens` | `number` | `20000` | Approximate recent-context tokens to retain verbatim after compaction. |

When the harness checks whether to compact, it calls `shouldCompact`:

```ts
// Simplified from compaction.ts
export function shouldCompact(
  contextTokens: number,
  contextWindow: number,
  settings: CompactionSettings,
): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

So compaction is triggered when the context grows to within `reserveTokens` of the model's total context window size. At that threshold, `reserveTokens` are the budget for the summarisation call itself.

### Step 1 — Estimating how many tokens you have

Before anything can be cut, the code needs to know how large the current context is. The function `estimateContextTokens` does this:

```ts
// Simplified from compaction.ts
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
  // If the most recent assistant message reported usage, trust that number
  // and estimate only the "trailing" messages after it.
  // Otherwise, fall back to a character-count heuristic (chars / 4 ≈ tokens).
  ...
}
```

The estimate is a `ContextUsageEstimate`:

```ts
export interface ContextUsageEstimate {
  tokens: number;           // Estimated total
  usageTokens: number;      // Tokens from the last assistant usage block
  trailingTokens: number;   // Estimated tokens after that block
  lastUsageIndex: number | null;
}
```

The heuristic used per message is `Math.ceil(chars / 4)`, with a special constant `ESTIMATED_IMAGE_CHARS = 4800` for image content blocks. The test suite confirms this covers all message roles: `user`, `assistant`, `toolResult`, `custom`, `bashExecution`, `branchSummary`, and `compactionSummary`.

### Step 2 — Finding the cut point

`prepareCompaction` calls `findCutPoint` to pick where the summary boundary goes:

```ts
// Simplified from compaction.ts
export function findCutPoint(
  entries: SessionTreeEntry[],
  startIndex: number,
  endIndex: number,
  keepRecentTokens: number,
): CutPointResult
```

It walks the entries backwards from the end, accumulating token estimates. When accumulated tokens reach `keepRecentTokens`, it marks that position as the cut. The return value:

```ts
export interface CutPointResult {
  firstKeptEntryIndex: number;  // First entry kept verbatim
  turnStartIndex: number;       // Turn-start when the cut splits a turn; otherwise -1
  isSplitTurn: boolean;         // True when the cut falls mid-turn
}
```

Not every position is a valid cut — `findCutPoint` only considers "valid cut points": positions that start a turn (user messages, branch summaries, custom messages). Tool result messages, for example, are never valid cut points because they belong to the assistant turn that preceded them. You might expect the cut to always land at a clean turn boundary, but `isSplitTurn: true` tells us when the cut falls in the middle of a long turn. When that happens, `prepareCompaction` builds two separate summarisation jobs: one for the history before the split, and one for the prefix of the split turn.

### Step 3 — Preparing the compaction

`prepareCompaction` assembles everything into a `CompactionPreparation`:

```ts
export interface CompactionPreparation {
  firstKeptEntryId: string;         // Session entry id where retained history begins
  messagesToSummarize: AgentMessage[];  // Messages to include in the history summary
  turnPrefixMessages: AgentMessage[];   // Prefix of a split turn (if isSplitTurn)
  isSplitTurn: boolean;
  tokensBefore: number;             // Estimated context size before compaction
  previousSummary?: string;         // Existing summary to update (iterative compaction)
  fileOps: FileOperations;          // Read/written/edited files extracted from history
  settings: CompactionSettings;
}
```

Notice `previousSummary`. If a prior compaction entry exists in the session, `prepareCompaction` carries its summary forward so `generateSummary` can produce an *update* rather than a full fresh summary. The test confirms this:

```ts
// From compaction.test.ts — prepareCompaction picks up the previous summary
const preparation = getOrThrow(prepareCompaction(pathEntries, DEFAULT_COMPACTION_SETTINGS));
expect(preparation?.previousSummary).toBe("First summary");
```

`prepareCompaction` also returns `undefined` (wrapped in an `ok`) when there is nothing to compact — for example when the session is empty, or the last entry is already a compaction:

```ts
// From compaction.test.ts
expect(getOrThrow(prepareCompaction([compaction], DEFAULT_COMPACTION_SETTINGS))).toBeUndefined();
expect(getOrThrow(prepareCompaction([], DEFAULT_COMPACTION_SETTINGS))).toBeUndefined();
```

### Step 4 — Running the summary

`compact` calls `generateSummary` which talks to the model with a structured system prompt:

```
You are a context summarization assistant. Your task is to read a conversation
between a user and an AI coding assistant, then produce a structured summary
following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the
conversation. ONLY output the structured summary.
```

The summary format the model must follow:

```
## Goal
## Constraints & Preferences
## Progress
  ### Done
  ### In Progress
  ### Blocked
## Key Decisions
## Next Steps
## Critical Context
```

When `isSplitTurn` is true, `compact` fires two parallel `generateSummary` calls — one for the full history and one for the split-turn prefix — and joins the results with a `---` separator. This way the model always has a coherent picture of both what came before the retained window and what the early part of the current turn accomplished.

The `maxTokens` budget passed to the model is `Math.min(Math.floor(0.8 * reserveTokens), model.maxTokens)` — at most 80% of `reserveTokens`, capped by the model's own output limit. The test for the split-turn path uses `Math.floor(0.5 * reserveTokens)` for the turn-prefix summary. Both caps exist to prevent the summary from consuming more tokens than the compaction was trying to save.

When the model is a reasoning model and `thinkingLevel !== "off"`, `generateSummary` passes `reasoning: thinkingLevel` in the completion options. The test verifies this precisely:

```ts
// From compaction.test.ts
expect(seenOptions[0]).toMatchObject({ reasoning: "medium", apiKey: "test-key" });
```

Custom instructions can be appended to the summarisation prompt:

```ts
// From compaction.test.ts — custom instructions appear in the prompt
expect(promptText).toContain("Additional focus: focus");
```

### Step 5 — Calling compact from AgentHarness

`AgentHarness` exposes a `compact()` method that orchestrates all of this. When called, the harness:

1. Fetches the current branch entries via `this.session.getBranch()`.
2. Calls `prepareCompaction(branchEntries, DEFAULT_COMPACTION_SETTINGS)`.
3. If `prepareCompaction` returns a preparation, fires the `session_before_compact` hook — listeners can cancel or provide a pre-built compaction.
4. Calls `compact(preparation, model, auth.apiKey, auth.headers, ...)`.
5. Persists the result with `this.session.appendCompaction(...)`.
6. Fires the `session_compact` event.

```ts
// Simplified from agent-harness.ts
async compact(customInstructions?: string): Promise<{
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  details?: unknown;
}>
```

The harness requires idle phase for `compact()` — calling it while the agent is mid-turn throws `AgentHarnessError("busy", ...)`.

---

## Part 2 — Session storage: persisting the conversation

### The problem

The compaction algorithm operates on `SessionTreeEntry[]` — a sequence of entries that describe the full history of a conversation, including messages, model changes, tool activations, compaction boundaries, and branch navigation. We need something that stores and retrieves this sequence, both in memory (for tests and short-lived usage) and on disk (for real use across process restarts).

### The shape of an entry

Each `SessionTreeEntry` has a shared shape: `type`, `id` (a short UUID), `parentId` (the previous entry's id or `null` for the root), and `timestamp`. The entry's `type` field determines what other fields it carries:

| Entry type | What it records |
|---|---|
| `message` | One `AgentMessage` (user, assistant, tool result, etc.) |
| `model_change` | Switch to a new model provider + model id |
| `thinking_level_change` | New thinking level (`"off"`, `"medium"`, `"high"`) |
| `active_tools_change` | Updated list of active tool names |
| `compaction` | Summary text, `firstKeptEntryId`, `tokensBefore`, optional `details` |
| `branch_summary` | Summary of a branch navigation (for tree-based session UIs) |
| `custom` | Arbitrary typed data |
| `custom_message` | A typed message that appears in the context |
| `label` | Human-readable label attached to another entry |
| `session_info` | Session name |
| `leaf` | Pointer to the current leaf (used for branching) |

### Two storage backends

The library provides two `SessionStorage` implementations behind a common interface:

**`InMemorySessionStorage`** — stores everything in `Map<string, SessionTreeEntry>`. No I/O. Fast for testing and ephemeral use. Created directly:

```ts
// Simplified from memory-storage.ts
const storage = new InMemorySessionStorage();
// or with pre-loaded entries:
const storage = new InMemorySessionStorage({ entries: [...], metadata });
```

**`JsonlSessionStorage`** — stores entries in a `.jsonl` file, one JSON line per entry. The first line is a session header:

```jsonl
{"type":"session","version":3,"id":"abc123","timestamp":"...","cwd":"/workspace"}
{"type":"message","id":"e1a2b3c4","parentId":null,"timestamp":"...","message":{...}}
{"type":"model_change","id":"f4d3e2c1","parentId":"e1a2b3c4","timestamp":"...","provider":"anthropic","modelId":"claude-sonnet-4-5"}
```

Opening an existing file reads it fully into memory and builds the `byId` map. New entries are appended one line at a time — no rewriting the file.

```ts
// Simplified from jsonl-storage.ts
const storage = await JsonlSessionStorage.create(fs, filePath, {
  cwd: "/workspace",
  sessionId: "abc123",
});
// Later, in a new process:
const storage = await JsonlSessionStorage.open(fs, existingFilePath);
```

### The Session class

Both storage backends are consumed through the `Session` class (`session.ts`), which wraps a `SessionStorage` and provides typed append methods:

```ts
// Simplified from session.ts
class Session<TMetadata extends SessionMetadata = SessionMetadata> {
  async appendMessage(message: AgentMessage): Promise<string>
  async appendModelChange(provider: string, modelId: string): Promise<string>
  async appendThinkingLevelChange(thinkingLevel: string): Promise<string>
  async appendActiveToolsChange(activeToolNames: string[]): Promise<string>
  async appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?): Promise<string>
  async appendLabel(targetId, label): Promise<string>
  async appendSessionName(name): Promise<string>
  async getBranch(fromId?: string): Promise<SessionTreeEntry[]>
  async buildContext(): Promise<SessionContext>
  async moveTo(entryId: string | null, summary?): Promise<string | undefined>
}
```

`getBranch` is worth understanding clearly: it calls `getPathToRoot(leafId)` on the storage, which walks `parentId` links backwards from the current leaf all the way to the root, then reverses the result. The branch is therefore a chronological flat array of entries from root to current leaf — and that is exactly what `prepareCompaction` and `buildSessionContext` consume.

### Building the context from entries

`buildSessionContext` (in `session.ts`) converts a branch of `SessionTreeEntry[]` into a `SessionContext` — the structure that the agent loop actually sees:

```ts
export interface SessionContext {
  messages: AgentMessage[];
  thinkingLevel: string;
  model: { provider: string; modelId: string } | null;
  activeToolNames: string[] | null;
}
```

The key logic: if the branch contains a `compaction` entry, `buildSessionContext` inserts a `compactionSummary` message at the front of `messages`, then adds only the "first kept" messages that follow the compaction boundary. The test confirms:

```ts
// From compaction.test.ts
const loaded = buildSessionContext([u1, a1, u2, a2, compaction, u3, a3]);
expect(loaded.messages).toHaveLength(5);
expect(loaded.messages[0]?.role).toBe("compactionSummary");
```

Seven raw entries collapse to five context messages — the compaction summary plus the four entries after the compaction boundary.

### Repository classes for session management

Above `Session` sit two repository classes that know how to create, list, open, delete, and fork sessions:

**`InMemorySessionRepo`** — manages a `Map<string, Session>` in memory. Its `create()` method needs no options; it generates a uuid and timestamp automatically.

**`JsonlSessionRepo`** — manages a directory of `.jsonl` files on disk. Each working directory gets its own subdirectory under the configured `sessionsRoot`:

```
~/.xzy/sessions/
  --workspace--          ← encoded cwd
    2025-01-15T10-30-00-000Z_abc123.jsonl
    2025-01-15T11-45-22-000Z_def456.jsonl
```

The cwd is encoded as `--<path-with-slashes-replaced-by-dashes>--`. Sessions are sorted newest-first when listed. The `fork` method copies a session's entries up to an optional entry id and creates a new file, which is how branching conversation trees are implemented.

---

## Part 3 — System-prompt building

### The problem

Every call to `AgentHarness.prompt()` needs a fresh system prompt. That prompt has to include the current date, the working directory, the list of available tools, and the skills the model should know about. Building this correctly every turn — and keeping it consistent — is best centralised.

<!-- GAP: The content of system-prompt.ts (S32) was already read. Let me confirm the actual exported functions. -->

The harness's `createTurnState` method calls the `systemPrompt` option. When that option is a function, the harness calls it with:

```ts
// Simplified from agent-harness.ts
systemPrompt = await this.systemPrompt({
  env: this.env,
  session: this.session,
  model: this.model,
  thinkingLevel: this.thinkingLevel,
  activeTools,
  resources,
});
```

So you can provide your own system-prompt builder as a plain async function. If the `systemPrompt` option is a plain string, the harness uses that directly. If it is omitted entirely, the harness falls back to `"You are a helpful assistant."`.

The `agent-core` library also exports `formatSkillsForSystemPrompt` from `skills.ts` — a helper you call from inside your system-prompt builder function to append the skills block. We'll see that in the next section.

---

## Part 4 — Skills: discoverable instruction files

### What is a skill?

A skill is a Markdown file that the model can read when a task matches the skill's description. The file has a YAML frontmatter block at the top and instruction prose in the body. The harness loads skills from directories, validates them, and includes their names, descriptions, and file paths in the system prompt so the model knows to consult them.

### SKILL.md discovery

`loadSkills(env, dirs)` in `skills.ts` traverses one or more directories looking for skill files:

1. In each directory, if a file named `SKILL.md` is present, load it as the skill for that directory and stop (do not recurse further into subdirectories from that point).
2. If no `SKILL.md` is found, recurse into subdirectories.
3. At the root level only, any `.md` file is also treated as a potential skill.
4. Ignore files and directories that match patterns in `.gitignore`, `.ignore`, or `.fdignore`.
5. Directories named `node_modules` or starting with `.` are always skipped.

This means a typical skill directory looks like:

```
~/.xzy/skills/
  my-skill/
    SKILL.md          ← loaded as skill named "my-skill"
    helper.sh         ← not touched
  another-skill/
    SKILL.md          ← loaded as skill named "another-skill"
```

### Frontmatter validation

Each skill file must have valid YAML frontmatter with at minimum a `description` field. The loader enforces:

| Field | Required | Constraint |
|---|---|---|
| `description` | Yes | Non-empty; max 1024 characters |
| `name` | No | Defaults to the parent directory name; max 64 chars; must match `^[a-z0-9-]+$`; no leading/trailing/consecutive hyphens |
| `disable-model-invocation` | No | Boolean; when `true` the skill is hidden from the model (not surfaced in the prompt) |

A skill file without a `description` is silently skipped (a diagnostic warning is emitted, but it does not block other skills from loading). The `name` from frontmatter must match the parent directory name — if it does not, a warning is emitted. A file like this is a valid skill:

```markdown
---
description: Summarise the project's current test coverage and suggest improvements.
---

Read the test files in `src/` and run `npm test -- --coverage`.
Then list untested functions and propose concrete test cases.
```

Save it as `~/.xzy/skills/test-coverage/SKILL.md` and the skill named `test-coverage` will be loaded.

### Sourced skills

`loadSourcedSkills` is a variant of `loadSkills` that attaches a provenance value to every loaded skill and diagnostic. You provide an array of `{ path, source }` pairs, and the loader preserves each `source` on the returned skill objects. The harness does not interpret `source`; you define the shape that makes sense for your application.

### Formatting skills into the system prompt

Once loaded, skills are formatted for the model by `formatSkillsForSystemPrompt`:

```ts
// Simplified from skills.ts
export function formatSkillsForSystemPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter((skill) => !skill.disableModelInvocation);
  if (visibleSkills.length === 0) return "";

  return [
    "The following skills provide specialized instructions for specific tasks.",
    "Read the full skill file when the task matches its description.",
    "When a skill file references a relative path, resolve it against the skill directory ...",
    "",
    "<available_skills>",
    ...visibleSkills.map((skill) => [
      "  <skill>",
      `    <name>${skill.name}</name>`,
      `    <description>${skill.description}</description>`,
      `    <location>${skill.filePath}</location>`,
      "  </skill>",
    ].join("\n")),
    "</available_skills>",
  ].join("\n");
}
```

The XML-like block tells the model: when a user's request resembles a skill's description, read the file at `<location>` to get full instructions.

### Invoking a skill directly

`AgentHarness` has a `.skill(name, additionalInstructions?)` method. It finds the skill by name in `resources.skills`, wraps it in a structured prompt block using `formatSkillInvocation`, and runs it as a turn:

```ts
// Simplified from skills.ts
export function formatSkillInvocation(skill: Skill, additionalInstructions?: string): string {
  const skillBlock =
    `<skill name="${skill.name}" location="${skill.filePath}">` +
    `\nReferences are relative to ${dirname(skill.filePath)}.\n\n` +
    `${skill.content}\n</skill>`;
  return additionalInstructions
    ? `${skillBlock}\n\n${additionalInstructions}`
    : skillBlock;
}
```

This is distinct from the model *discovering* a skill on its own. `harness.skill("test-coverage")` immediately starts a turn whose first user message is the full skill content, wrapped so the model knows exactly where to find related files.

---

## Part 5 — AgentHarness: putting it together

### Construction

`AgentHarness` is constructed from an `AgentHarnessOptions` object. The required fields are:

```ts
// Simplified from agent-harness.ts (constructor)
new AgentHarness({
  env,          // ExecutionEnv — file system and runtime access
  session,      // A Session<...> object (from Part 2)
  model,        // Model<any> — which model to call
  tools,        // AgentTool[] — available tools
  systemPrompt, // string | async function returning a string
  resources: {
    skills,         // Skill[] loaded by loadSkills
    promptTemplates, // PromptTemplate[]
  },
  streamOptions,       // HTTP/transport options
  getApiKeyAndHeaders, // async (model) => { apiKey, headers }
  thinkingLevel,       // "off" | "medium" | "high" (defaults to "off")
  activeToolNames,     // subset of tools to expose (defaults to all)
  steeringMode,        // "one-at-a-time" | "all" (defaults to "one-at-a-time")
  followUpMode,        // "one-at-a-time" | "all" (defaults to "one-at-a-time")
})
```

Tool names must be unique. Duplicate names throw an `AgentHarnessError("invalid_argument", ...)` at construction time.

### The turn lifecycle

When you call `harness.prompt("describe the bug")`, the harness:

1. Checks that `phase === "idle"`. If not, throws `AgentHarnessError("busy", ...)`.
2. Sets `phase = "turn"`.
3. Calls `createTurnState()` — which reads the session branch, builds context, resolves the system prompt function.
4. Runs `executeTurn()` — which calls `runAgentLoop` with the messages, context, and a `StreamFn` that wraps `streamSimple`.
5. Every time the agent loop emits a message, `handleAgentEvent` is called — which appends messages to the session, emits subscriber events, and triggers `save_point` after each turn.
6. On completion, `flushPendingSessionWrites` drains any deferred session writes (model changes, tool changes, etc. queued during the turn).
7. `phase` returns to `"idle"`.

The `prepareNextTurn` callback (injected into the loop config) is called between turns in a multi-turn exchange. It flushes pending writes and rebuilds turn state so the next turn picks up any model or tool changes that happened during the current turn.

### Phase state machine

`AgentHarness` has a simple phase field that prevents concurrent operations:

| Phase | Meaning |
|---|---|
| `"idle"` | Ready for a new operation |
| `"turn"` | Running `prompt()`, `skill()`, or `promptFromTemplate()` |
| `"compaction"` | Running `compact()` |
| `"branch_summary"` | Running `navigateTree()` |

Operations that need idle phase (`compact()`, `navigateTree()`, new `prompt()`) throw `AgentHarnessError("busy", ...)` if called while another operation is active. `steer()` and `followUp()` can be called during a running turn — they push messages into queues that the loop drains between turns.

### Hook system

`AgentHarness` exposes two registration methods:

- **`subscribe(listener)`** — receives every event (both agent-loop events and harness-own events). Use this for logging or UI rendering.
- **`on(type, handler)`** — registers a typed hook for a specific event type. The hook can return a result that modifies behaviour:

| Hook type | What the return value can do |
|---|---|
| `"before_agent_start"` | Prepend extra messages; override system prompt |
| `"context"` | Replace the messages array before the loop sees it |
| `"tool_call"` | Block a tool call with `{ block: true, reason: "..." }` |
| `"tool_result"` | Replace the tool result content or terminate the loop |
| `"session_before_compact"` | Cancel compaction or provide a pre-built compaction |
| `"before_provider_request"` | Patch HTTP/transport options per request |
| `"before_provider_payload"` | Replace the raw API payload |

Both `subscribe` and `on` return an unsubscribe function.

### Pending session writes

Session mutations (model change, tool change, etc.) that happen *during* a running turn cannot be written to the session immediately, because the session's leaf pointer is mid-update. `AgentHarness` keeps a `pendingSessionWrites` queue. Each pending write type (`"message"`, `"model_change"`, `"thinking_level_change"`, `"active_tools_change"`, `"custom"`, `"custom_message"`, `"label"`, `"session_info"`, `"leaf"`) is flushed in order when `flushPendingSessionWrites` runs — at turn end, before `prepareNextTurn`, and in the `compact()` call.

### A minimal working example

Let's put everything together in a small example that uses in-memory storage (so no disk is needed) and a plain string system prompt:

```ts
// Minimal AgentHarness usage — in-memory session, no skills
import { AgentHarness } from "agent-core/harness/agent-harness";
import { InMemorySessionStorage } from "agent-core/harness/session/memory-storage";
import { Session } from "agent-core/harness/session/session";

// 1. Create a session backed by in-memory storage
const storage = new InMemorySessionStorage();
const session = new Session(storage);

// 2. Build the harness
const harness = new AgentHarness({
  env: myEnv,           // your ExecutionEnv implementation
  session,
  model: myModel,       // a Model<any> from the registry
  tools: [myReadTool],  // AgentTool[]
  systemPrompt: "You are a helpful assistant.",
  getApiKeyAndHeaders: async () => ({ apiKey: process.env.API_KEY! }),
  streamOptions: {},
});

// 3. Subscribe to events for logging
harness.subscribe((event) => {
  if (event.type === "message_end") {
    console.log("[assistant]", event.message);
  }
});

// 4. Run a prompt
const result = await harness.prompt("What files are in the current directory?");
console.log(result.content);

// 5. Compact when the context grows
const { summary, tokensBefore } = await harness.compact();
console.log(`Compacted ${tokensBefore} tokens into a summary.`);
```

For on-disk persistence, swap `InMemorySessionStorage` for a `JsonlSessionStorage`-backed session via `JsonlSessionRepo`:

```ts
// On-disk session via JsonlSessionRepo
import { JsonlSessionRepo } from "agent-core/harness/session/jsonl-repo";

const repo = new JsonlSessionRepo({
  fs: myFileSystem,
  sessionsRoot: "~/.xzy/sessions",
});

// Create a new session for the current working directory
const session = await repo.create({ cwd: process.cwd() });

// Or open an existing one by metadata
const [mostRecent] = await repo.list({ cwd: process.cwd() });
const session = mostRecent ? await repo.open(mostRecent) : await repo.create({ cwd: process.cwd() });
```

### Loading skills and wiring them in

```ts
// Loading skills and adding them to the harness resources
import { loadSkills, formatSkillsForSystemPrompt } from "agent-core/harness/skills";

const { skills, diagnostics } = await loadSkills(env, [
  "~/.xzy/skills",    // global skills
  ".xzy/skills",      // project-local skills
]);

for (const d of diagnostics) {
  if (d.type === "warning") console.warn(d.message, d.path);
}

const harness = new AgentHarness({
  // ...
  resources: { skills },
  systemPrompt: ({ activeTools, resources }) => {
    const toolList = activeTools.map((t) => t.name).join(", ");
    const skillsBlock = formatSkillsForSystemPrompt(resources.skills ?? []);
    return [
      "You are a helpful coding assistant.",
      `Active tools: ${toolList}`,
      skillsBlock,
    ].filter(Boolean).join("\n\n");
  },
});

// Later, invoke a skill directly
await harness.skill("test-coverage", "Focus on the src/auth module.");
```

---

## What the coding agent does differently

`AgentHarness` is the general-purpose reusable harness that `agent-core` provides. The full coding agent we build later in this series takes a different path: it introduces its own `AgentSession` class (in the `coding-agent` package) which has tighter integration with extension points, a more specialised compaction strategy, and project-specific system-prompt logic. The ideas are the same — session storage, compaction, prompt building, skill loading — but the coding agent re-implements them in its own composition rather than delegating to `AgentHarness` directly.

That means: use `AgentHarness` for your own agent projects, or as a reference for the patterns. When we reach the coding-agent chapters you will recognise every concept, even though the class names differ.

---

## Summary

We've walked through four layers that `AgentHarness` assembles:

1. **Compaction** — `prepareCompaction` finds the cut point and builds a `CompactionPreparation`; `compact` calls the model to write the summary; `DEFAULT_COMPACTION_SETTINGS` controls the token budget.
2. **Session storage** — entries form a linked tree (`parentId` chain); `Session` provides typed append methods; `InMemorySessionStorage` for tests, `JsonlSessionStorage` for disk; `buildSessionContext` collapses the branch into messages the loop can use.
3. **System-prompt building** — the `systemPrompt` option is a string or an async function; `formatSkillsForSystemPrompt` produces the skills block you include in it.
4. **Skill loading** — `loadSkills` traverses directories for `SKILL.md` files, validates frontmatter, and returns typed `Skill` objects; `loadSourcedSkills` adds provenance; `harness.skill()` runs a skill as an immediate turn.

---

← Previous: [The Agent Class: State Machine, Steering, and Lifecycle](./the-agent-class.md) · Next: [Cross-Provider Message Transforms and Handoff](./cross-provider-message-transforms.md) →
