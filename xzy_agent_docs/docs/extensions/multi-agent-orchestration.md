---
title: "Multi-Agent Orchestration"
description: "Build a multi-agent extension that fans work across planner, scout, worker, and reviewer subagents — each with its own isolated context window."
category: extensions
type: tutorial
tags: [multi-agent, orchestration, subagent, planner, scout, worker, reviewer, SDK, coding-agent, agent composition, parallel agents, fan-out, agentic, autonomous, chain, extension, registerTool, ExtensionAPI, discoverAgents, AgentConfig]
keywords: [subagent orchestration, spawn subagent, parallel coding agents, agent chain, agent fan-out, agent isolation, context window isolation, agent roles, workflow orchestration, xzy extension]
sources: [S82, S83]
---

**TL;DR** — A single agent with one context window can stall on large or multi-faceted tasks. This chapter builds a multi-agent orchestration extension that spawns specialised subagents — a planner that decomposes the work, scouts that explore the codebase in parallel, workers that implement, and a reviewer that checks the result. Each subagent runs in its own isolated context window via the `subagent` tool. By the end you will have a working orchestration extension and understand how to compose agents into fan-out, chain, and parallel workflows.

# Multi-Agent Orchestration

## The problem with one agent and one context window

Let's think about what happens when you ask the coding agent to tackle something large: "refactor the auth module, add OAuth support, then write tests." In a single session the agent needs to hold all the reconnaissance, the plan, the implementation, and the tests in one context window. Context fills up. The model starts forgetting early findings. Compaction kicks in. Even with compaction, a single agent doing serial work is slower than several agents working in parallel, and it is hard to specialise — the same context that is ideal for fast file lookup is not ideal for careful code review.

The solution is to break the work across specialised agents, each with its own isolated context window. A **planner** decomposes the task. **Scouts** explore the codebase and return compressed findings. **Workers** implement from a plan. A **reviewer** checks the result. The parent agent orchestrates the whole pipeline by calling a single `subagent` tool, collecting results, and handing them back to the model.

There is one important detail: the agent runtime ships no built-in subagents. You build them as an extension — which is exactly what we are about to do.

## What we are building

The extension we will walk through (S82) provides a `subagent` tool that exposes three modes to the model:

| Mode | Parameter shape | What it does |
|------|----------------|--------------|
| Single | `{ agent, task }` | One subagent, one task |
| Parallel | `{ tasks: [{agent, task}, ...] }` | Multiple subagents run concurrently (max 8, 4 at a time) |
| Chain | `{ chain: [{agent, task}, ...] }` | Sequential pipeline with `{previous}` placeholder for handoff |

Each mode spawns one or more child `xzy` processes, each isolated in its own context window, and collects their output back to the parent.

The extension directory looks like this:

```
subagent/
├── index.ts          # Extension entry point — registers the subagent tool
├── agents.ts         # Agent discovery logic
├── agents/           # Agent definition files (Markdown with YAML frontmatter)
│   ├── scout.md
│   ├── planner.md
│   ├── worker.md
│   └── reviewer.md
└── prompts/          # Workflow preset slash-commands
    ├── implement.md
    ├── scout-and-plan.md
    └── implement-and-review.md
```

We will build this up piece by piece: first the agent definition format, then the discovery mechanism, then the tool that runs them, and finally the workflow presets.

## Step 1 — Define the agents

Before the extension can spawn subagents, it needs to know what agents are available. The extension uses Markdown files with YAML frontmatter to declare each agent's identity, allowed tools, model, and system prompt. Here is the structure:

```markdown
---
name: my-agent
description: What this agent does
tools: read, grep, find, ls
model: claude-haiku-4-5
---

System prompt for the agent goes here.
```

The `tools` field is a comma-separated list of the tools this agent is permitted to use. The `model` field is optional — if absent the agent inherits the default. The body below the frontmatter becomes the agent's system prompt.

Let's look at the four roles the sample extension defines:

### scout — fast codebase recon

```markdown
---
name: scout
description: Fast codebase recon that returns compressed context for handoff to other agents
tools: read, grep, find, ls, bash
model: claude-haiku-4-5
---

You are a scout. Quickly investigate a codebase and return structured findings
that another agent can use without re-reading everything.

Your output will be passed to an agent who has NOT seen the files you explored.
```

The scout runs on `claude-haiku-4-5` — a fast, inexpensive model suited for broad lookups. Its system prompt instructs it to produce structured output (files with line ranges, key code snippets, an architecture summary) that another agent can consume without any additional reads. Notice the explicit instruction: "Your output will be passed to an agent who has NOT seen the files you explored." That sentence is the contract that makes chaining work.

### planner — implementation plans only

```markdown
---
name: planner
description: Creates implementation plans from context and requirements
tools: read, grep, find, ls
model: claude-sonnet-4-5
---

You are a planning specialist. You receive context (from a scout) and requirements,
then produce a clear implementation plan.

You must NOT make any changes. Only read, analyze, and plan.
```

The planner has no `bash` tool and its prompt explicitly forbids it from making changes. This is a deliberate constraint — a planner that accidentally edits files contaminates the scout's findings. The planner's output is a numbered step list with exact file paths and what to change.

### worker — full capabilities

```markdown
---
name: worker
description: General-purpose subagent with full capabilities, isolated context
model: claude-sonnet-4-5
---

You are a worker agent with full capabilities. You operate in an isolated context window
to handle delegated tasks without polluting the main conversation.
```

The worker has no `tools` restriction, so it receives the default tool set. Its system prompt instructs it to work autonomously and report back in a structured format — completed files, key functions touched, and notes for the next agent (useful when it hands off to a reviewer).

### reviewer — read-only analysis

```markdown
---
name: reviewer
description: Code review specialist for quality and security analysis
tools: read, grep, find, ls, bash
model: claude-sonnet-4-5
---

Bash is for read-only commands only: `git diff`, `git log`, `git show`.
Do NOT modify files or run builds.
```

The reviewer has `bash` in its allowed tools, but its system prompt restricts bash to read-only operations: `git diff`, `git log`, `git show`. This is a soft constraint — the extension notes that tool permissions are not perfectly enforceable, so the system prompt reinforces the intent.

### Where agents live

Agent files can live in two places:

| Scope | Location | Loaded when |
|-------|----------|-------------|
| User-level | `~/.xzy/agents/*.md` | Always (default) |
| Project-level | `.xzy/agents/*.md` | Only with `agentScope: "project"` or `"both"` |

When `agentScope` is `"both"` and a project agent shares a name with a user agent, the project agent takes precedence. This lets a repo ship specialised overrides of generic user-level agents.

## Step 2 — Discover agents at runtime

The extension discovers available agents fresh on each invocation. This is intentional: you can edit an agent file mid-session and the next invocation picks up the change without restarting.

The discovery logic lives in `agents.ts` and exports two types plus one function:

```ts
// Simplified view of AgentConfig and discoverAgents()
export type AgentScope = "user" | "project" | "both";

export interface AgentConfig {
  name: string;
  description: string;
  tools?: string[];    // undefined means "all default tools"
  model?: string;      // undefined means "inherit default"
  systemPrompt: string;
  source: "user" | "project";
  filePath: string;
}

export function discoverAgents(cwd: string, scope: AgentScope): AgentDiscoveryResult {
  const userDir = path.join(getAgentDir(), "agents");
  const projectAgentsDir = findNearestProjectAgentsDir(cwd);

  const userAgents   = scope === "project" ? [] : loadAgentsFromDir(userDir, "user");
  const projectAgents = scope === "user" || !projectAgentsDir
    ? []
    : loadAgentsFromDir(projectAgentsDir, "project");

  const agentMap = new Map<string, AgentConfig>();

  if (scope === "both") {
    // User agents first, project agents override by name
    for (const agent of userAgents)   agentMap.set(agent.name, agent);
    for (const agent of projectAgents) agentMap.set(agent.name, agent);
  } else if (scope === "user") {
    for (const agent of userAgents)   agentMap.set(agent.name, agent);
  } else {
    for (const agent of projectAgents) agentMap.set(agent.name, agent);
  }

  return { agents: Array.from(agentMap.values()), projectAgentsDir };
}
```

`getAgentDir()` returns the user-level agent directory (the `~/.xzy/` area managed by the SDK — see [SDK and Programmatic Use](../coding-agent/sdk-and-programmatic-use.md) for the full SDK reference). `findNearestProjectAgentsDir()` walks up the directory tree from `cwd` looking for a `.xzy/agents/` directory, stopping at the filesystem root.

Each `.md` file is parsed with `parseFrontmatter()` — a utility exported by the `coding-agent` package — which splits the YAML header from the body. Files missing `name` or `description` in the frontmatter are silently skipped.

## Step 3 — Register the subagent tool

The extension's entry point, `index.ts`, follows the same pattern as every extension we have seen: export a default function that receives the `ExtensionAPI` object and registers capabilities. Here the capability is a single tool named `subagent`.

If you have not built an extension before, here is a brief recap: the extension system is covered in depth in [Building Real Extensions](./building-real-extensions.md). An extension is a file (or directory with `index.ts`) that exports a default function receiving `ExtensionAPI`. You register tools with `ext.registerTool(...)`, passing a name, description, TypeBox parameter schema, and an `execute` function. The extension loader calls your function at startup.

### The tool's parameter schema

The `subagent` tool accepts three mutually exclusive parameter shapes — only one may be active per call:

```ts
// Simplified view of SubagentParams
const SubagentParams = Type.Object({
  // Single mode
  agent: Type.Optional(Type.String({ description: "Name of the agent to invoke" })),
  task:  Type.Optional(Type.String({ description: "Task to delegate" })),

  // Parallel mode
  tasks: Type.Optional(Type.Array(Type.Object({
    agent: Type.String(),
    task:  Type.String(),
    cwd:   Type.Optional(Type.String()),
  }))),

  // Chain mode
  chain: Type.Optional(Type.Array(Type.Object({
    agent: Type.String(),
    task:  Type.String({ description: "Task; use {previous} for prior step's output" }),
    cwd:   Type.Optional(Type.String()),
  }))),

  agentScope:          Type.Optional(AgentScopeSchema),  // "user" | "project" | "both"
  confirmProjectAgents: Type.Optional(Type.Boolean()),   // default: true
  cwd:                 Type.Optional(Type.String()),
});
```

The `execute` function checks exactly one of `chain`, `tasks`, or `agent + task` is present and dispatches accordingly. If more or fewer than one mode is active it returns an error listing the available agents.

### Security confirmation for project-local agents

When `agentScope` is `"project"` or `"both"` and the session has a UI, the extension calls `ctx.ui.confirm()` before running any project-local agents. This is the pattern from the extension API's `ctx` object — the same one we used for permission gates in [Building Real Extensions](./building-real-extensions.md).

```ts
// (Inside execute — simplified)
if ((agentScope === "project" || agentScope === "both") && confirmProjectAgents && ctx.hasUI) {
  const ok = await ctx.ui.confirm(
    "Run project-local agents?",
    `Agents: ${names}\nSource: ${dir}\n\nProject agents are repo-controlled. Only continue for trusted repositories.`,
  );
  if (!ok) return { content: [{ type: "text", text: "Canceled: project-local agents not approved." }], ... };
}
```

You can bypass this prompt by passing `confirmProjectAgents: false` — useful in non-interactive (RPC/print) mode.

## Step 4 — Spawn a subagent process

Here is the core mechanism: each subagent is a separate `xzy` process launched with `spawn()` from `node:child_process`. The parent reads its JSON-mode output line by line and accumulates the messages. Let's walk through `runSingleAgent()`:

```ts
// Simplified view of runSingleAgent()
async function runSingleAgent(
  defaultCwd: string,
  agents:     AgentConfig[],
  agentName:  string,
  task:       string,
  cwd:        string | undefined,
  step:       number | undefined,
  signal:     AbortSignal | undefined,
  onUpdate:   OnUpdateCallback | undefined,
  makeDetails: (results: SingleResult[]) => SubagentDetails,
): Promise<SingleResult> {

  const agent = agents.find((a) => a.name === agentName);

  // Build the CLI args
  const args: string[] = ["--mode", "json", "-p", "--no-session"];
  if (agent.model) args.push("--model", agent.model);
  if (agent.tools && agent.tools.length > 0) args.push("--tools", agent.tools.join(","));

  // Write the agent's system prompt to a temp file
  // and pass it as --append-system-prompt <path>
  if (agent.systemPrompt.trim()) {
    const tmp = await writePromptToTempFile(agent.name, agent.systemPrompt);
    args.push("--append-system-prompt", tmp.filePath);
  }

  // The task becomes the user message
  args.push(`Task: ${task}`);

  // Spawn the process
  const invocation = getPiInvocation(args);
  const proc = spawn(invocation.command, invocation.args, {
    cwd:   cwd ?? defaultCwd,
    shell: false,
    stdio: ["ignore", "pipe", "pipe"],
  });

  // Parse JSON events from stdout line by line
  proc.stdout.on("data", (data) => {
    // ... parse newline-delimited JSON events
    // message_end events contain the full AssistantMessage
    // tool_result_end events contain tool results
    // accumulate into currentResult.messages
  });

  // Propagate abort
  if (signal?.aborted) proc.kill("SIGTERM");
}
```

A few things worth noticing:

**JSON mode and no-session.** The child process runs with `--mode json` (structured output, no TUI) and `--no-session` (stateless — no session persistence). Each subagent is ephemeral.

**Tool restriction.** The `--tools` flag restricts which tools the child can use. This is how the planner's read-only constraint and the scout's bash access are enforced at the process level.

**System prompt injection.** The agent's system prompt body (everything below the frontmatter) is written to a temp file and passed as `--append-system-prompt`. The temp file is deleted in a `finally` block after the process exits.

**Abort propagation.** The parent's `AbortSignal` is wired to send `SIGTERM` to the child, followed by `SIGKILL` after 5 seconds if the child has not exited.

**Context isolation.** Each child process has its own memory, its own model context, and its own conversation history. The only communication between parent and child is the task string in and the final output text out. There is no shared state.

**Usage tracking.** As `message_end` events arrive, the extension accumulates `input`, `output`, `cacheRead`, `cacheWrite`, `cost`, `contextTokens`, and `turns` for each result, so the parent can display per-agent usage stats.

### What happens when a subagent fails

The `isFailedResult()` helper checks three conditions:

```ts
function isFailedResult(result: SingleResult): boolean {
  return result.exitCode !== 0
    || result.stopReason === "error"
    || result.stopReason === "aborted";
}
```

| Condition | Meaning |
|-----------|---------|
| `exitCode !== 0` | Child process exited with an error |
| `stopReason === "error"` | LLM returned an error response |
| `stopReason === "aborted"` | User sent Ctrl+C; parent killed the subprocess |

In chain mode the tool stops at the first failing step and returns an error that includes which step failed and its error message. In parallel mode each task is reported independently — some can fail while others succeed.

## Step 5 — Wire up the three modes

### Chain mode: sequential with `{previous}`

Chain mode is the key to multi-step pipelines. Each step in the array runs sequentially. The task string for each step may contain the literal placeholder `{previous}`, which the extension replaces with the previous step's final output text before spawning the child:

```ts
// Inside the chain execution loop (simplified)
let previousOutput = "";

for (let i = 0; i < params.chain.length; i++) {
  const step = params.chain[i];
  const taskWithContext = step.task.replace(/\{previous\}/g, previousOutput);

  const result = await runSingleAgent(ctx.cwd, agents, step.agent, taskWithContext, ...);
  results.push(result);

  if (isFailedResult(result)) {
    return { content: [{ type: "text", text: `Chain stopped at step ${i + 1} (${step.agent}): ...` }], isError: true };
  }

  previousOutput = getFinalOutput(result.messages);
}

// Return only the last step's output to the parent model
return { content: [{ type: "text", text: getFinalOutput(results[results.length - 1].messages) }], ... };
```

Notice that the parent model sees only the last step's output. The intermediate scout and planner outputs are visible in the tool details panel (the `details` object carries all `SingleResult` entries) but are not cluttering the main conversation context.

### Parallel mode: concurrent fan-out with concurrency cap

Parallel mode runs multiple subagents concurrently using `mapWithConcurrencyLimit()`:

```ts
const MAX_PARALLEL_TASKS = 8;
const MAX_CONCURRENCY   = 4;

// Up to 8 tasks total, 4 running simultaneously
const results = await mapWithConcurrencyLimit(
  params.tasks,
  MAX_CONCURRENCY,
  async (t, index) => runSingleAgent(ctx.cwd, agents, t.agent, t.task, ...),
);
```

The concurrency limit prevents resource exhaustion — you will not accidentally saturate API rate limits or spawn eight processes simultaneously. Each completed task's output is capped at 50 KB (`PER_TASK_OUTPUT_CAP = 50 * 1024`) before being assembled into the tool's return value. The full output is preserved in tool details; only the truncated version is surfaced to the parent model's context.

The parallel return value assembles a summary for each task:

```ts
// Each task gets a section header with its status
const summaries = results.map((r) => {
  const output = truncateParallelOutput(getResultOutput(r));
  const status = isFailedResult(r) ? `failed (${r.stopReason})` : "completed";
  return `### [${r.agent}] ${status}\n\n${output}`;
});

return {
  content: [{ type: "text",
    text: `Parallel: ${successCount}/${results.length} succeeded\n\n${summaries.join("\n\n---\n\n")}`
  }],
  details: makeDetails("parallel")(results),
};
```

The parent model receives one block of text containing all agents' outputs, clearly delimited, so it can reason across them in a single follow-up turn.

## Step 6 — Workflow presets as slash-commands

So far we have a `subagent` tool the model can call directly. But there is a higher-level convenience: pre-authored workflow prompt files that the slash-command system turns into `/implement`, `/scout-and-plan`, and `/implement-and-review` commands.

These are Markdown files in the `prompts/` directory. Here is the full implement workflow:

```markdown
---
description: Full implementation workflow - scout gathers context, planner creates plan, worker implements
---
Use the subagent tool with the chain parameter to execute this workflow:

1. First, use the "scout" agent to find all code relevant to: $@
2. Then, use the "planner" agent to create an implementation plan for "$@"
   using the context from the previous step (use {previous} placeholder)
3. Finally, use the "worker" agent to implement the plan from the previous step
   (use {previous} placeholder)

Execute this as a chain, passing output between steps via {previous}.
```

`$@` expands to the argument the user typed after the slash command. So `/implement add Redis caching to the session store` produces the scout task "find all code relevant to: add Redis caching to the session store", then feeds the scout's output to the planner, then feeds the plan to the worker.

The three presets cover common patterns:

| Slash command | Chain |
|--------------|-------|
| `/implement <query>` | scout → planner → worker |
| `/scout-and-plan <query>` | scout → planner (no implementation) |
| `/implement-and-review <query>` | worker → reviewer → worker (apply feedback) |

The `implement-and-review` pattern is particularly useful: the first worker implements, the reviewer reads the diff and produces a structured critique, and the second worker applies the review feedback. The entire cycle happens without the parent agent writing a single line of code — it orchestrates, subagents act.

## Step 7 — Install the extension

To install the extension from the repository root:

```bash
# Create the extension directory (must be a subdirectory with index.ts)
mkdir -p ~/.xzy/extensions/subagent

# Symlink the extension files
ln -sf "$(pwd)/packages/coding-agent/examples/extensions/subagent/index.ts" \
  ~/.xzy/extensions/subagent/index.ts
ln -sf "$(pwd)/packages/coding-agent/examples/extensions/subagent/agents.ts" \
  ~/.xzy/extensions/subagent/agents.ts

# Symlink the agent definitions
mkdir -p ~/.xzy/agents
for f in packages/coding-agent/examples/extensions/subagent/agents/*.md; do
  ln -sf "$(pwd)/$f" ~/.xzy/agents/$(basename "$f")
done

# Symlink the workflow presets
mkdir -p ~/.xzy/prompts
for f in packages/coding-agent/examples/extensions/subagent/prompts/*.md; do
  ln -sf "$(pwd)/$f" ~/.xzy/prompts/$(basename "$f")
done
```

After installing, start `xzy` and the `subagent` tool and the three slash commands are available immediately.

### Try it out

Single agent (manual):

```
Use scout to find all authentication code
```

Parallel fan-out (the model calls the tool directly):

```
Run 2 scouts in parallel: one to find models, one to find providers
```

Full chain via slash-command:

```
/implement add Redis caching to the session store
```

## Step 8 — Display: collapsed and expanded views

The extension implements `renderCall` and `renderResult` so the TUI shows something useful during and after execution.

**Collapsed view** (default) for a single agent result:
- Status icon (`✓`/`✗`/`⏳`) and agent name
- Last 10 items (tool calls and text), with a "... N earlier items" prefix if truncated
- Usage stats: `3 turns ↑12k ↓4k $0.0082 ctx:16k claude-sonnet-4-5`

**Expanded view** (press `Ctrl+O`) for a single result:
- Full task text
- All tool calls with formatted arguments
- Final output rendered as Markdown
- Per-task usage

For chain and parallel modes, each step/task gets its own section in both views.

**Streaming during parallel execution:**
- The TUI shows all tasks with live status: `⏳ running`, `✓ done`, `✗ failed`
- A summary line updates as tasks finish: "2/3 done, 1 running"

## Extending the pattern: custom providers per role

The extension system goes further than tools and agent definitions. You can also register a custom LLM provider — giving orchestration pipelines the ability to use different providers for different agent roles.

The custom-provider example (S83) demonstrates `ext.registerProvider()`:

```ts
// Simplified view of a custom provider registration (from S83)
export default function (ext: ExtensionAPI) {
  ext.registerProvider("custom-anthropic", {
    baseUrl: "https://api.anthropic.com",
    apiKey:  "$CUSTOM_ANTHROPIC_API_KEY",   // env var name
    api:     "custom-anthropic-api",

    models: [
      {
        id:            "claude-opus-4-5",
        name:          "Claude Opus 4.5 (Custom)",
        reasoning:     true,
        input:         ["text", "image"],
        cost:          { input: 5, output: 25, cacheRead: 0.5, cacheWrite: 6.25 },
        contextWindow: 200000,
        maxTokens:     64000,
      },
      {
        id:            "claude-sonnet-4-5",
        name:          "Claude Sonnet 4.5 (Custom)",
        reasoning:     true,
        input:         ["text", "image"],
        cost:          { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens:     64000,
      },
    ],

    oauth: {
      name:         "Custom Anthropic (Claude Pro/Max)",
      login:        loginAnthropic,
      refreshToken: refreshAnthropicToken,
      getApiKey:    (cred) => cred.access,
    },

    streamSimple: streamCustomAnthropic,
  });
}
```

`registerProvider()` takes a provider id, a `baseUrl`, an API key (either a literal value or an env var name prefixed with `$`), a list of model definitions, an optional `oauth` block for PKCE-based OAuth login, and a `streamSimple` function that receives the model, context, and options and returns an `AssistantMessageEventStream`.

Once registered, the provider's models appear in `/model` and can be selected like any built-in model. An orchestration extension could, in principle, use a faster or cheaper model for scouting by registering a secondary provider and routing scout agent invocations to it.

The OAuth block in the example performs a full PKCE flow: `login` generates a verifier/challenge pair, builds the authorization URL, prompts the user to paste the code, and exchanges it for tokens. `refreshToken` handles token renewal. `getApiKey` extracts the bearer token from stored credentials.

<!-- GAP: The exact call signature of `registerProvider` from the ExtensionAPI type definition is not present in S83 (which shows a usage example but not the type). The exact type of the options object — beyond what is inferred from the example — is not confirmed by the assigned sources. -->

## Composing a generic orchestration pattern

Let's put the whole picture together. A generic orchestration pipeline that decomposes, explores, implements, and reviews looks like this:

```ts
// Example: what the model calls when you type /implement <query>
subagent({
  chain: [
    {
      agent: "scout",
      task:  "Find all code relevant to: add Redis caching to the session store",
    },
    {
      agent: "planner",
      task:  "Create an implementation plan for 'add Redis caching to the session store'. Context: {previous}",
    },
    {
      agent: "worker",
      task:  "Implement this plan: {previous}",
    },
  ],
  agentScope: "user",
})
```

What makes this robust:

1. **Isolation.** The scout's context is independent of the planner's. The planner does not carry file contents it never needed. The worker starts fresh with just the plan.
2. **Specialisation.** The scout runs a fast, inexpensive model with the right tool set. The planner is restricted from writing. The reviewer is restricted to read-only bash.
3. **Parallelism when possible.** If you want to explore two subsystems at once, use `tasks` (parallel mode) for the scout step, then pass both results into the planner.
4. **Composability.** You can chain a parallel fan-out and a sequential pipeline in the same session by calling the tool twice in successive model turns — first in parallel, then in chain.

## Where to put your own agents

To add a new agent role, create a Markdown file following the frontmatter format and drop it into `~/.xzy/agents/`. It is available on the next invocation of the `subagent` tool — no restart required. You can also add it to `.xzy/agents/` in a specific project directory for repo-scoped roles, and enable it with `agentScope: "both"`.

The agent's `tools` field and system prompt together define its security posture. Keep both minimal: give the agent only the access it needs for its role, and state that constraint clearly in the system prompt so the model internalises it.

---

## You have now built the full stack

We started at the very bottom: a streaming call to an LLM (`llm-toolkit`), then an agent loop that ran tools in a cycle, then a terminal UI that made the loop interactive, then the coding agent that brought file editing and project awareness, and finally the extension system that lets you attach new tools, commands, hooks, and — as this chapter showed — whole fleets of specialised subagents.

Every layer in the stack is something you assembled. You know how the stream arrives, how the loop decides when to call tools and when to stop, how the TUI renders events, how the extension loader discovers and initialises your code, and now how a parent agent can fan work out across isolated child agents and collect the results.

The capabilities we have covered — multi-provider routing, parallel fan-out, chain pipelines with context handoff, custom agent definitions — are the building blocks of production agentic systems. From here you can extend in any direction: new agent roles, new workflow presets, a custom provider that routes to a different inference backend per task, or an orchestration layer that spawns agents across multiple projects. The foundation is in place.

---

← Previous: [Building Real Extensions: Tools, State, Commands, and Hooks](./building-real-extensions.md)
