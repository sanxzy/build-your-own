---
title: "Multi-Agent Orchestration: Fan-Out with Subagents"
description: "Build a multi-agent orchestration extension — a planner spawns scout, worker, and reviewer subagents, fans work across parallel coding sessions, and aggregates results."
category: extensions
type: tutorial
tags: [multi-agent, orchestration, subagent, planner, scout, worker, reviewer, fan-out, parallel, agent composition, extensions]
keywords: [multi-agent, subagent orchestration, fan-out, parallel agents, planner-worker, agent composition]
sources: [S20, S50]
---

**TL;DR** — Complex tasks benefit from multiple agents working in parallel. We'll build a multi-agent orchestration extension: a **planner** breaks down the task, spawns **scout** subagents to explore the codebase, **worker** subagents to make changes in parallel, and a **reviewer** to verify the results. This is the final chapter — it composes everything we've built into the most advanced pattern in the library.

## The orchestration pattern

The multi-agent pattern mirrors how a senior developer leads a team:

```
User: "Add dark mode to the settings page"
          │
          ▼
    ┌──────────┐
    │ PLANNER  │  ← Breaks task into sub-tasks
    └────┬─────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│ SCOUT │ │ SCOUT │  ← Explore codebase in parallel
└───┬───┘ └───┬───┘
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│WORKER │ │WORKER │  ← Make changes in parallel
└───┬───┘ └───┬───┘
    │         │
    └────┬────┘
         ▼
    ┌──────────┐
    │ REVIEWER │  ← Verify correctness
    └──────────┘
```

## The SDK: creating subagents programmatically

First, we need the ability to create agent sessions programmatically. The SDK entry point wraps `AgentSession` creation:

```ts
// From the coding-agent SDK
async function createAgentSession(options: {
  model?: string;
  tools?: string[];       // subset of tools to enable
  systemPrompt?: string;  // specialized prompt
  cwd?: string;
  parentSession?: string;
}): Promise<AgentSession> {
  const session = new AgentSession();
  await session.start({
    modelId: options.model ?? "claude-sonnet-4-6",
    cwd: options.cwd ?? process.cwd(),
    sessionId: undefined, // new session
  });

  // If tools are restricted, deregister the others
  if (options.tools) {
    session.restrictTools(options.tools);
  }

  // If a specialized system prompt is provided, override
  if (options.systemPrompt) {
    session.setSystemPrompt(options.systemPrompt);
  }

  return session;
}
```

## The Planner

The planner is the orchestrator. It breaks down the user's task and coordinates subagents:

```ts
// planner-extension.ts
export function activate(api: ExtensionAPI): void {
  api.registerCommand({
    name: "orchestrate",
    description: "Run a multi-agent workflow on a task",
    handler: async (task: string) => {
      // Phase 1: Plan
      api.log("info", `Planning: ${task}`);
      const plan = await createPlan(task, api);

      // Phase 2: Scout (parallel)
      api.log("info", `Scouting with ${plan.scoutTasks.length} scouts...`);
      const scoutResults = await Promise.all(
        plan.scoutTasks.map(async (scoutTask) => {
          const scout = await createAgentSession({
            tools: ["read", "grep", "glob", "list"],
            systemPrompt: `You are a code scout. ${scoutTask.prompt}`,
            cwd: api.getSession().cwd,
          });
          scout.prompt(scoutTask.query);
          await scout.waitForIdle();
          return extractFindings(scout);
        }),
      );

      // Phase 3: Work (parallel)
      api.log("info", `Working with ${plan.workTasks.length} workers...`);
      const workResults = await Promise.all(
        plan.workTasks.map(async (workTask) => {
          const worker = await createAgentSession({
            tools: ["read", "write", "edit", "bash", "grep", "glob"],
            systemPrompt: `You are a code worker. Context from scouts:\n${formatScoutResults(scoutResults)}\n\n${workTask.prompt}`,
            cwd: api.getSession().cwd,
          });
          worker.prompt(workTask.query);
          await worker.waitForIdle();
          return extractChanges(worker);
        }),
      );

      // Phase 4: Review
      api.log("info", "Reviewing changes...");
      const reviewer = await createAgentSession({
        tools: ["read", "grep", "glob", "bash"],
        systemPrompt: `You are a code reviewer. Verify that the following changes are correct and complete:\n${formatChanges(workResults)}`,
        cwd: api.getSession().cwd,
      });
      reviewer.prompt("Review all changes. Report any issues.");
      await reviewer.waitForIdle();
      const review = extractReview(reviewer);

      return formatFinalReport(plan, scoutResults, workResults, review);
    },
  });
}
```

## Parallel execution

The key insight: scouts run in parallel, workers run in parallel (after scouts complete), and the reviewer runs after workers. This is a pipeline with parallelism at each stage:

```ts
// Scout fan-out: all scouts run concurrently
const scoutResults = await Promise.all(scoutTasks.map(runScout));

// Worker fan-out: all workers run concurrently, with scout context
const workResults = await Promise.all(workTasks.map(t => runWorker(t, scoutResults)));

// Review: single reviewer (could also be a panel)
const review = await runReviewer(workResults);
```

Each subagent is an independent `AgentSession` — they don't share state, don't block each other, and can use different models (e.g., scouts use a cheap fast model, workers use a capable model).

## Error handling in orchestration

Subagents can fail. The orchestrator handles failures gracefully:

```ts
const results = await Promise.allSettled(
  workTasks.map(async (task) => {
    try {
      return await runWorker(task);
    } catch (err) {
      return { error: err.message, task };
    }
  }),
);

const succeeded = results.filter(r => r.status === "fulfilled");
const failed = results.filter(r => r.status === "rejected");

if (failed.length > 0) {
  api.log("warn", `${failed.length} workers failed, continuing with ${succeeded.length}`);
}
```

Failed subagents don't block the workflow — the reviewer sees which tasks succeeded and which failed, and the final report surfaces both.

## Use cases

Multi-agent orchestration shines for:

- **Codebase migrations** — one worker per module, parallelizing the work
- **Bug investigations** — scouts explore different hypotheses simultaneously
- **PR reviews** — reviewer subagents check different aspects (security, performance, style)
- **Documentation generation** — workers each draft a section, reviewer checks consistency

## What we've built — the complete library

This is the final chapter. Let's review everything we've built:

| Section | Chapters | What you can do |
|---|---|---|
| Getting Started | 2 | Understand the architecture, set up the workspace |
| LLM Toolkit | 7 | Stream from any LLM provider through a unified API |
| Agent Core | 5 | Run multi-turn conversations with tool execution and compaction |
| Terminal UI | 4 | Render a flicker-free chat interface in the terminal |
| Coding Agent | 10 | Use the full CLI with tools, hooks, sessions, and three run modes |
| Extensions | 4 | Add custom tools, providers, hooks, and multi-agent workflows |

You've built, from scratch, a production-grade AI coding agent. Each layer is tested independently. The code is modular — you can swap out the TUI for a web interface, add new LLM providers, or extend the agent with custom tools. The architecture you've learned applies to any AI agent, not just coding assistants.

The best way to solidify what you've learned: pick one layer and extend it. Add a new provider. Build a custom tool. Write an extension. The foundation is yours.

---

← Previous: [Building Real Extensions: Tools, State, Commands, and Hooks](./building-real-extensions.md)
