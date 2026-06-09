---
title: "The Scheduler Loop"
description: Build the 30-second tick that atomically claims due triggers, advances next-run, recovers lost triggers on startup, and routes through the admission gate before enqueuing.
category: scheduling
type: tutorial
tags:
  - scheduler
  - scheduler loop
  - tick
  - claimDueTriggers
  - advanceNextRun
  - recoverLostTriggers
  - admission gate
  - AgentReadiness
  - schedule dispatch
  - 30s tick
  - template interpolation
  - skipped vs failed
  - run status sync
  - squad routing
  - schedule trigger
  - nextRunAt
  - cron
  - task queue
  - atomic claim
  - crash recovery
keywords:
  - periodic scheduler
  - heartbeat loop
  - trigger deduplication
  - missed trigger recovery
  - agent online check
  - dispatch gate
  - create_issue mode
  - run_only mode
  - trigger timezone
  - issue title template
  - date variable interpolation
sources: [S12, S7, S8]
---

**TL;DR** — The schedule data model tells us *when* each trigger is due, but we still need a running loop that actually fires them — exactly once, even when multiple orchestrator instances are alive or the server restarts mid-tick. This chapter builds that loop: a 30-second ticker that atomically claims due triggers, dispatches each one through an admission gate, and advances the next-run timestamp forward. We also add a startup recovery pass so a crash never leaves a trigger stuck with a `NULL` next-run forever.

# The Scheduler Loop

In [The Schedule Data Model](./schedule-data-model.md) we designed the `schedule_triggers` table and its `next_run_at` column. A trigger is "due" when `next_run_at <= now()`. But the table just stores data — it does not fire anything on its own. We need a process that wakes up periodically, looks for due triggers, and turns them into queued tasks.

That process is the **scheduler loop**. Let's build it from the ground up.

Before diving in, here is a quick map of the three concepts this chapter builds on:

- **Trigger and `nextRunAt`** — a row in `schedule_triggers` that tracks the cron expression and the wall-clock time the schedule should fire next. Introduced in [The Schedule Data Model](./schedule-data-model.md).
- **Task queue** — the table that holds pending work; runners claim rows from it and execute the agent. Covered in [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md).
- **Squad** — a group of agents with a designated leader who receives routed work. A schedule can target a whole squad, in which case the leader is the actual executing agent. See [Squads: A Leader That Delegates](../coordination/squads.md).

## Step 1 — The simplest possible tick

The scheduler's job is repetitive: every 30 seconds, query the database for triggers whose `next_run_at` has passed, and do something with each one.

Here is the skeleton before we add any of the complexity:

```ts
// src/orchestrator/scheduler.ts

const SCHEDULER_INTERVAL_MS = 30_000; // 30 seconds — sourced from S12

export async function runSchedulerLoop(
  db: DatabaseQueries,
  svc: ScheduleService,
  signal: AbortSignal,
): Promise<void> {
  while (!signal.aborted) {
    await tick(db, svc);
    await sleep(SCHEDULER_INTERVAL_MS, signal);
  }
}

async function tick(db: DatabaseQueries, svc: ScheduleService): Promise<void> {
  // We'll fill this in over the next few steps.
}

function sleep(ms: number, signal: AbortSignal): Promise<void> {
  return new Promise((resolve) => {
    const id = setTimeout(resolve, ms);
    signal.addEventListener("abort", () => { clearTimeout(id); resolve(); }, { once: true });
  });
}
```

The `AbortSignal` gives us a clean shutdown path: when the server receives SIGTERM we abort the signal, the sleep resolves immediately, and the loop exits at the `while (!signal.aborted)` check. The interval is exactly 30 seconds — not tunable at runtime — matching the source implementation (S12).

## Step 2 — Claiming due triggers atomically

Now we can implement the tick body. The first thing it needs to do is find which triggers are due. Here is the naive approach:

```ts
// Naive — DO NOT USE
async function tick(db: DatabaseQueries): Promise<void> {
  const triggers = await db.getDueTriggers(); // SELECT * WHERE next_run_at <= now()
  for (const t of triggers) {
    await dispatch(t);
    await db.advanceNextRun(t.id, computeNextRun(t.cronExpression));
  }
}
```

There is a problem here. Imagine two orchestrator instances running simultaneously (a common scenario during rolling restarts). Both call `getDueTriggers()` at the same tick, see the same rows, and both try to dispatch the same schedule. The agent receives two copies of the task.

The solution is to fold the *claim* and the *read* into a single atomic database operation. We call this `claimDueTriggers`. Under the hood it is an `UPDATE ... RETURNING` that atomically sets a "claimed" marker (typically by setting `next_run_at = NULL`) and returns the rows it just claimed. Only one instance can claim any given trigger row; the second instance gets back an empty result set.

```ts
// src/orchestrator/scheduler.ts (revised tick body)

async function tick(db: DatabaseQueries, svc: ScheduleService): Promise<void> {
  let triggers: ScheduleTriggerRow[];
  try {
    triggers = await db.claimDueTriggers(); // atomic UPDATE…RETURNING
  } catch (err) {
    console.warn("scheduler: failed to claim due triggers", { error: err });
    return; // skip this tick; try again in 30s
  }

  if (triggers.length === 0) return;

  console.info("scheduler: claimed triggers", { count: triggers.length });

  for (const trigger of triggers) {
    await dispatchTrigger(db, svc, trigger);
    await advanceNextRun(db, trigger);
  }
}
```

Notice the order: we dispatch the task *and then* advance `next_run_at`. The claim already set `next_run_at = NULL`, so the trigger is invisible to further ticks until we write the new value. We advance after dispatch so that a dispatch crash leaves the trigger in the recoverable NULL state rather than silently skipping the run. We will handle that recovery case in Step 4.

### Advancing next-run

Once a trigger fires we need to compute the *next* wall-clock time it should fire, based on the cron expression, and write it back:

```ts
// src/orchestrator/scheduler.ts

async function advanceNextRun(
  db: DatabaseQueries,
  trigger: ScheduleTriggerRow,
): Promise<void> {
  if (!trigger.cronExpression) return; // non-cron triggers have no next-run

  const tz = trigger.timezone || DEFAULT_TRIGGER_TIMEZONE; // "UTC" default — S12, S7
  let next: Date;
  try {
    next = computeNextRun(trigger.cronExpression, tz);
  } catch (err) {
    console.warn("scheduler: failed to compute next run", {
      triggerId: trigger.id,
      cron: trigger.cronExpression,
      error: err,
    });
    return;
  }

  try {
    await db.advanceNextRun(trigger.id, next);
  } catch (err) {
    console.warn("scheduler: failed to advance next_run_at", {
      triggerId: trigger.id,
      error: err,
    });
  }
}

const DEFAULT_TRIGGER_TIMEZONE = "UTC"; // S7: DefaultAutopilotTriggerTimezone
```

`computeNextRun` uses the cron library to calculate the next occurrence after `now()` in the trigger's timezone. The timezone defaults to `"UTC"` when the trigger has none set — this matches the source constant `DefaultAutopilotTriggerTimezone` (S7).

## Step 3 — The admission gate: should we dispatch at all?

We claimed a trigger. Before we create any task, we need to ask: *is the target agent actually able to do work right now?*

This matters because agents can be in several states that make dispatching pointless. If we skip the check and enqueue the task anyway, it will sit in the queue forever — or worse, a crash-loop runner will pick it up repeatedly. The source (S7) explicitly names the problem: "without it a paused laptop / offline daemon causes scheduled triggers to pile thousands of doomed tasks onto the task queue."

### AgentReadiness — a shared readiness check

The readiness check lives in its own function so that every code path that might dispatch to an agent — the scheduler, the issue-assignment handler, the squad routing logic — all use the same gate and stay in sync. Its logic is (S8):

1. If the agent is archived (`archivedAt IS NOT NULL`) → **not ready**; reason: `"agent is archived"`.
2. If the agent has no runner bound (`runtimeId IS NULL`) → **not ready**; reason: `"agent has no runtime bound"`.
3. Load the runner record and check its `status`. If `status != "online"` → **not ready**; reason: `"agent runtime is <status>"`.
4. Otherwise → **ready**.

```ts
// src/orchestrator/agent-readiness.ts

export interface ReadinessResult {
  ready: boolean;
  reason: string; // non-empty when !ready
}

export async function agentReadiness(
  db: DatabaseQueries,
  agent: AgentRow,
): Promise<ReadinessResult> {
  // Gate 1: archived check — S8
  if (agent.archivedAt !== null) {
    return { ready: false, reason: "agent is archived" };
  }
  // Gate 2: runner bound check — S8
  if (agent.runtimeId === null) {
    return { ready: false, reason: "agent has no runtime bound" };
  }
  // Gate 3: runner online check — S8
  const runtime = await db.getAgentRuntime(agent.runtimeId);
  if (runtime.status !== "online") {
    return { ready: false, reason: `agent runtime is ${runtime.status}` };
  }
  return { ready: true, reason: "" };
}
```

A `db.getAgentRuntime` error (a transient database hiccup) propagates as an exception. The caller — the admission check — must decide what to do with it. The source (S7, `shouldSkipDispatch`) **fails open** on transient DB errors: it logs a warning but does *not* skip the dispatch. The reasoning is that a temporary connection drop should not silently eat a scheduled run. If the DB is truly down, the whole tick will fail at `claimDueTriggers` anyway.

### Skipped vs failed — an important distinction

When the admission gate rejects a dispatch, we do not want to record the run as `failed`. "Failed" means the agent tried to do the work and something went wrong. "Skipped" means we looked at the situation and decided *not* to try. The distinction matters for alerting and for any auto-pause logic that watches failure rates (S7).

| Outcome | Status | Meaning |
|---|---|---|
| Agent not ready, assignee gone, squad archived | `skipped` | Intentional no-op; not the agent's fault |
| Unknown `execution_mode` in config | `failed` | Misconfiguration; needs human attention |
| Task enqueue or task creation threw an unexpected error | `failed` | Unexpected error during actual dispatch |
| Readiness check threw a transient DB error | *(fail-open; try dispatch anyway)* | Not a skip, not a fail — just a log warning |

Hard-skip conditions — things where retrying will never help (e.g. the assignee agent row was deleted, the squad was archived) — should always produce `skipped`, not `failed`. If the failure monitor sees a constant stream of `failed` runs for a schedule whose agent was deleted, it might auto-pause the schedule; calling it `skipped` avoids that false alarm (S7).

## Step 4 — Dispatching: two modes and squad routing

With the gate passed, we create the actual work. Schedules support two execution modes (S7):

| Mode | What happens |
|---|---|
| `create_task` | Create a task row linked to a new issue, then enqueue it. The issue gets a title generated from the template. |
| `run_only` | Create a task row directly, without an issue. The runner gets the task immediately. |

```ts
// src/orchestrator/dispatch.ts (Simplified view of dispatchSchedule)

export async function dispatchSchedule(
  db: DatabaseQueries,
  svc: ScheduleService,
  schedule: ScheduleRow,
  triggerId: string,
): Promise<ScheduleRunRow> {
  // Pre-flight admission gate — S7: shouldSkipDispatch
  const skipReason = await shouldSkipDispatch(db, schedule);
  if (skipReason) {
    return recordSkippedRun(db, schedule, triggerId, skipReason);
  }

  // Determine the initial run status based on execution mode — S7
  const initialStatus =
    schedule.executionMode === "run_only" ? "running" : "task_created";

  const run = await db.createScheduleRun({
    scheduleId: schedule.id,
    triggerId,
    source: "schedule",
    status: initialStatus,
  });

  switch (schedule.executionMode) {
    case "create_task":
      await dispatchCreateTask(db, svc, schedule, run);
      break;
    case "run_only":
      await dispatchRunOnly(db, svc, schedule, run);
      break;
    default:
      await failRun(db, run.id, `unknown execution_mode: ${schedule.executionMode}`);
      throw new Error(`unknown execution_mode: ${schedule.executionMode}`);
  }

  await db.updateScheduleLastRunAt(schedule.id);
  return run;
}
```

### Template interpolation in `create_task` mode

When a schedule fires in `create_task` mode it creates an issue with a title. That title can contain template variables — the source (S7) defines exactly one supported variable: `{{date}}`. The template engine is a simple regex replace (S7):

```ts
// src/orchestrator/template.ts

// Matches {{ date }} or {{date}} — whitespace inside braces is tolerated — S7
const TEMPLATE_TOKEN_RE = /\{\{\s*([^{}]*?)\s*\}\}/g;

const SUPPORTED_VARIABLES = ["date"] as const; // S7: SupportedIssueTitleTemplateVariables

export function interpolateTaskTitle(
  schedule: ScheduleRow,
  run: ScheduleRunRow,
  timezone: string,
): string {
  // Fall back to the schedule title if no template is set — S7
  const tmpl =
    schedule.taskTitleTemplate?.trim()
      ? schedule.taskTitleTemplate
      : schedule.title;

  const triggerDate = formatRunDate(run, timezone);

  return tmpl.replace(TEMPLATE_TOKEN_RE, (_match, name: string) => {
    switch (name.trim()) {
      case "date":
        return triggerDate; // e.g. "2026-06-09" — S7
      default:
        return _match; // unknown token: leave it verbatim — S7
    }
  });
}

export function validateTaskTitleTemplate(tmpl: string): void {
  if (!tmpl) return;
  for (const [, name] of tmpl.matchAll(TEMPLATE_TOKEN_RE)) {
    if (!SUPPORTED_VARIABLES.includes(name.trim() as typeof SUPPORTED_VARIABLES[number])) {
      throw new Error(
        `unknown template variable "${name.trim()}"; supported: {{${SUPPORTED_VARIABLES.join("}}, {{")}}}`,
      );
    }
  }
}
```

If a token is unrecognised, `interpolateTaskTitle` leaves it verbatim in the title (S7). Validation at save time (`validateTaskTitleTemplate`) rejects unknown tokens before they can be stored, so by the time we reach the scheduler, any token in the template is one we know how to handle.

### Squad routing

When a schedule targets a *squad* rather than a specific agent, the scheduler needs to figure out which agent actually receives the task — the squad's current leader (S7, `resolveScheduleLeader`).

```ts
// src/orchestrator/dispatch.ts (Simplified)

async function resolveScheduleLeader(
  db: DatabaseQueries,
  schedule: ScheduleRow,
): Promise<AgentRow> {
  if (schedule.assigneeType === "squad") {
    // S7: resolveAutopilotLeader — squad path
    const squad = await db.getSquad(schedule.assigneeId);
    if (squad.archivedAt !== null) {
      throw new SquadArchivedError();
    }
    return db.getAgent(squad.leaderId); // the leader is the executing agent
  }
  // Default: the schedule's direct assignee — S7
  return db.getAgent(schedule.assigneeId);
}
```

In `create_task` mode, the issue is created with `assigneeType = "squad"` and `assigneeId = squad.id`. The existing event listener chain then routes it to the squad leader when the task is enqueued (S7: `shouldEnqueueSquadLeaderOnAssign → enqueueSquadLeaderTask`). In `run_only` mode, the leader is resolved at dispatch time and the task is created directly with `agentId = leader.id` (S7: `dispatchRunOnly`).

## Step 5 — Crash recovery on startup

Now we can go back to the problem we left open in Step 2. After `claimDueTriggers` sets `next_run_at = NULL`, the trigger is invisible until `advanceNextRun` writes the next timestamp. But what if the server crashes between the claim and the advance? The trigger stays stuck with `next_run_at = NULL` forever — it will never fire again.

We fix this by running a recovery pass **once, at startup**, before the tick loop begins (S12):

```ts
// src/orchestrator/scheduler.ts

export async function runSchedulerLoop(
  db: DatabaseQueries,
  svc: ScheduleService,
  signal: AbortSignal,
): Promise<void> {
  // Recovery pass runs once before the loop starts — S12: recoverLostTriggers
  await recoverLostTriggers(db);

  while (!signal.aborted) {
    await tick(db, svc);
    await sleep(SCHEDULER_INTERVAL_MS, signal);
  }
}

async function recoverLostTriggers(db: DatabaseQueries): Promise<void> {
  let stuckTriggers: ScheduleTriggerRow[];
  try {
    stuckTriggers = await db.getTriggersWithNullNextRun(); // S12: RecoverLostTriggers
  } catch (err) {
    console.warn("scheduler: failed to load stuck triggers for recovery", { error: err });
    return;
  }

  if (stuckTriggers.length === 0) return;

  console.info("scheduler: recovering stuck triggers", { count: stuckTriggers.length });

  for (const trigger of stuckTriggers) {
    if (!trigger.cronExpression) continue; // non-cron trigger; skip — S12

    const tz = trigger.timezone || DEFAULT_TRIGGER_TIMEZONE;
    let next: Date;
    try {
      next = computeNextRun(trigger.cronExpression, tz);
    } catch (err) {
      console.warn("scheduler: failed to compute next run for recovery", {
        triggerId: trigger.id,
        error: err,
      });
      continue;
    }

    try {
      await db.advanceNextRun(trigger.id, next);
    } catch (err) {
      console.warn("scheduler: failed to recover trigger", {
        triggerId: trigger.id,
        error: err,
      });
    }
  }
}
```

`getTriggersWithNullNextRun` queries for all cron triggers where `next_run_at IS NULL`. For each one, it recomputes the next occurrence from the current time and writes it back via `advanceNextRun` (S12). The trigger is now visible to future ticks again.

Notice we call `computeNextRun` from the current moment, not from the moment the trigger last fired. This means a trigger that was stuck for three ticks will fire at its *next* scheduled time, not catch up on the missed ticks. This is the correct "skip, don't catch up" behaviour for schedules that represent recurring work (sending a report, running a daily job). The [Schedule Data Model](./schedule-data-model.md) chapter discusses how catch-up policy is configured; the recovery pass here always uses the forward-looking next-run.

## Step 6 — Run and issue status synchronisation

A schedule run does not end when the task is enqueued. The run row needs to reach a terminal state (`completed`, `failed`) when the underlying work finishes. The source (S7) handles this with two sync functions — one for each execution mode.

### `create_task` mode: sync from the issue

When a run is in `create_task` mode, its lifetime is tied to the issue created at dispatch time. When the issue moves to a terminal status, the run should follow (S7: `SyncRunFromIssue`):

| Issue status | Run outcome |
|---|---|
| `done` or `in_review` | `completed` |
| `cancelled` or `blocked` | `failed` (reason: `"issue <status>"`) |

```ts
// src/orchestrator/dispatch.ts (Simplified)

export async function syncRunFromTask(
  db: DatabaseQueries,
  task: TaskRow,
): Promise<void> {
  if (!task.scheduleRunId) return; // not a scheduled task — S7

  const run = await db.getScheduleRun(task.scheduleRunId);

  switch (task.status) {
    case "completed":
      await db.completeScheduleRun(run.id, task.result ?? null);
      break;
    case "failed":
    case "cancelled": {
      const reason = task.error ?? `task ${task.status}`;
      await db.failScheduleRun(run.id, reason);
      break;
    }
  }
}
```

### `run_only` mode: sync from the task

In `run_only` mode there is no issue, so we sync from the task row directly (S7: `SyncRunFromTask`):

```ts
// src/orchestrator/dispatch.ts (Simplified)

export async function syncRunFromIssue(
  db: DatabaseQueries,
  issue: IssueRow,
): Promise<void> {
  if (issue.originType !== "schedule") return; // not a scheduled task — S7

  const run = await db.getScheduleRunByIssue(issue.id);
  if (!run) return;

  switch (issue.status) {
    case "done":
    case "in_review":
      await db.completeScheduleRun(run.id, null);
      break;
    case "cancelled":
    case "blocked":
      await db.failScheduleRun(run.id, `issue ${issue.status}`);
      break;
  }
}
```

These two sync functions are called by the event subscribers that already listen for task and issue status changes. The scheduler itself does not poll for completions — it dispatches and walks away, trusting the event system to close the run.

## The complete loop at a glance

Putting it all together, the scheduler's execution path for each tick is:

```
startup
  └─ recoverLostTriggers()        ← rewrite NULL next_run_at rows
       (once, before the loop starts)

every 30 seconds
  └─ claimDueTriggers()           ← atomic UPDATE…RETURNING; sets next_run_at = NULL
       for each claimed trigger:
         ├─ loadSchedule()
         ├─ shouldSkipDispatch()  ← admission gate (agentReadiness)
         │    ├─ skip  → recordSkippedRun()
         │    └─ pass  → createScheduleRun()
         │                  ├─ create_task: interpolateTitle → createIssue → enqueueTask
         │                  └─ run_only:   resolveLeader → createTask → notifyRunner
         └─ advanceNextRun()      ← rewrite next_run_at to next cron occurrence

on issue/task terminal status
  └─ syncRunFromIssue / syncRunFromTask  ← close the run row
```

## Try it yourself

Here are three experiments that let you feel the scheduler's behaviour rather than just read about it.

**Experiment 1 — A schedule that fires every minute.**
Create a schedule with `cronExpression = "* * * * *"` (every minute) in `create_task` mode targeting a simple agent. Start the orchestrator. You should see a new task appear in the queue roughly every 60 seconds (the scheduler tick is every 30 seconds, so the worst-case delay is about 30 seconds after the trigger's `nextRunAt`).

**Experiment 2 — Crash recovery / skip behaviour.**
Let the experiment-1 schedule run for two ticks. Then stop the orchestrator for three minutes. When you restart it, `recoverLostTriggers` will run first. Because the cron expression fires every minute, the recovered trigger's new `nextRunAt` is set to the *next* future occurrence — the three missed ticks are skipped, not caught up. Verify this by looking at the `schedule_runs` table: you should see a gap of three minutes with no runs, then the schedule resuming from the next occurrence.

**Experiment 3 — Squad routing.**
Create a schedule with `assigneeType = "squad"`. In `create_task` mode, check that the created issue has `assigneeId = squad.id`. In `run_only` mode, check that the task row has `agentId = squad.leaderId`. Stop the squad leader's runner and watch the next tick produce a `skipped` run with reason `"squad leader agent runtime is offline"` (or similar) instead of a `failed` run.

---

← Previous: [The Schedule Data Model](./schedule-data-model.md) · Next: [Budgets and Cost Tracking](../governance/budgets-and-cost-tracking.md) →
