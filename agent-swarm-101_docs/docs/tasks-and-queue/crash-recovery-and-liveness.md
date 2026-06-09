---
title: "Crash Recovery and Liveness"
description: "Add a liveness sweeper that detects stale runners, fails their in-flight tasks, and recovers orphaned work to make the queue resilient to crashes and restarts."
category: tasks-and-queue
type: tutorial
tags:
  - crash recovery
  - orphan recovery
  - liveness
  - runtime sweeper
  - stale runtime
  - heartbeat
  - markRunnersOffline
  - failTasksForOfflineRunners
  - session pinning
  - work-dir pinning
  - task timeout
  - queued TTL
  - offline GC
  - rerun semantics
  - force_fresh_session
  - in_progress
  - dispatched
  - queued
  - task lifecycle
  - queue resilience
  - runner crash
  - orchestrator restart
keywords:
  - stuck task recovery
  - daemon restart
  - stale heartbeat detection
  - orphaned task
  - task timeout sweep
  - queued expiry
  - runner garbage collection
  - session resume
  - fresh session rerun
  - liveness store
  - redis heartbeat
  - periodic sweeper
  - task failure recovery
sources: [S5, S13, S36]
---

**TL;DR** — When a runner crashes mid-task the task stays `in_progress` forever, and no other runner will ever claim it. This chapter adds the liveness machinery that prevents that: a heartbeat field on the run, a periodic sweeper that marks stale runners offline and fails their tasks, and an orphan-recovery endpoint the orchestrator calls when it restarts. By the end you will have a queue that heals itself after crashes, timeouts, and restarts.

# Crash Recovery and Liveness

In the [previous chapter](./task-queue-and-claim-loop.md) we built the atomic claim loop: a runner queries for the oldest `queued` task, flips it to `in_progress`, and stamps it with its own run id. That stamp means "I own this" — but what happens if the runner dies?

The task stays `in_progress`. No other runner will touch it, because the claim loop skips anything that isn't `queued`. The user sees their task frozen. There is no automatic recovery.

We need to answer three questions:

1. How does the orchestrator notice that a runner has died?
2. What does it do with that runner's tasks?
3. What happens when the orchestrator itself restarts?

We will answer them in order, building a complete liveness subsystem.

---

## Step 1 — Liveness fields on the run

Before we can detect a dead runner we need a place to record what we know about a run's health. Let's start with the run's schema.

A **run** (introduced in [A Run: Sessions, Usage, and Cost](../the-agent/a-run.md)) is the record for one execution window of an agent — it carries the session id, usage, cost, and now the liveness state. The run is how the orchestrator tracks not just *what* ran but *whether it is still running*.

The run table carries two liveness columns (S36):

```ts
// Simplified view of the heartbeat_runs schema (S36)
// livenessState and livenessReason are the fields we care about here.
export const runs = pgTable("heartbeat_runs", {
  id: uuid("id").primaryKey().defaultRandom(),
  agentId: uuid("agent_id").notNull(),
  status: text("status").notNull().default("queued"),

  // When did the process actually start, and what is its OS pid?
  processPid: integer("process_pid"),
  processGroupId: integer("process_group_id"),
  processStartedAt: timestamp("process_started_at", { withTimezone: true }),

  // Rolling activity window — updated whenever the agent produces output.
  lastOutputAt: timestamp("last_output_at", { withTimezone: true }),
  lastOutputSeq: integer("last_output_seq").notNull().default(0),

  // The sweeper (or the runner itself) writes these when it decides the run
  // is alive, stale, or dead.
  livenessState: text("liveness_state"),
  livenessReason: text("liveness_reason"),

  // Retry chain — a failed run can spawn a successor.
  retryOfRunId: uuid("retry_of_run_id").references((): AnyPgColumn => runs.id, {
    onDelete: "set null",
  }),
  processLossRetryCount: integer("process_loss_retry_count").notNull().default(0),
  scheduledRetryAt: timestamp("scheduled_retry_at", { withTimezone: true }),
  scheduledRetryAttempt: integer("scheduled_retry_attempt").notNull().default(0),
  scheduledRetryReason: text("scheduled_retry_reason"),

  // Session resume pointer — pinned after the agent's first message.
  sessionIdBefore: text("session_id_before"),
  sessionIdAfter: text("session_id_after"),

  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Notice `livenessState` and `livenessReason` — they give the orchestrator a human-readable record of *why* a run was declared dead, which is valuable for debugging and for the UI ("agent process lost" vs. "task timed out").

Also notice `lastOutputAt`. Runners update this timestamp every time they produce output. An old `lastOutputAt` is a signal that the agent process may be stuck or dead, distinct from the runner heartbeat itself.

Now we have a place to write liveness state. But we still need something to *check* it on a schedule.

---

## Step 2 — The sweeper loop

Here is the core problem we are solving in this step: the orchestrator cannot wait for a runner to say "I died" — a crashed process cannot send a message. The orchestrator must proactively check.

We build a periodic sweep loop that runs every 30 seconds (the `sweepInterval` in S13). Each tick performs four distinct jobs:

| Job | What it does |
|---|---|
| `sweepStaleRunners` | Find runners whose last heartbeat is older than the stale threshold; mark them offline; fail their tasks |
| `sweepStaleTasks` | Fail tasks stuck in `dispatched` or `running` too long, even when the runner is still online |
| `sweepExpiredQueuedTasks` | Expire tasks that have been `queued` past a TTL without ever being claimed |
| `gcRunners` | Delete offline runners with no active agents after a long retention window |

Let's look at each in turn.

### The main tick

```ts
// src/orchestrator/sweeper.ts

const SWEEP_INTERVAL_MS = 30_000          // 30 s between ticks (S13)
const STALE_THRESHOLD_SECS = 150          // runner is stale if no heartbeat for 150 s (S13)
const OFFLINE_RUNNER_TTL_SECS = 7 * 24 * 3600 // 7 days before GC (S13)
const DISPATCH_TIMEOUT_SECS = 300         // 5 min: dispatched→running transition (S13)
const RUNNING_TIMEOUT_SECS = 9000         // 2.5 h: server-side backstop for long runs (S13)
const QUEUED_TTL_SECS = 2 * 3600         // 2 h: max time a task may sit queued (S13)
const QUEUED_EXPIRE_BATCH_SIZE = 500      // rows per tick, keeps DB transactions short (S13)

export async function startSweeper(ctx: AbortSignal, db: Database): Promise<void> {
  const tick = setInterval(async () => {
    if (ctx.aborted) { clearInterval(tick); return; }
    await sweepStaleRunners(db);
    await sweepStaleTasks(db);
    await sweepExpiredQueuedTasks(db);
    await gcRunners(db);
  }, SWEEP_INTERVAL_MS);
}
```

Each constant is directly from S13 and carries the original design rationale as comments. You should treat these as *tunable defaults* — your production environment may warrant different values, but these are the values the source system was built and tested with.

### sweepStaleRunners — the heart of crash detection

A runner is considered stale when the database has not seen a heartbeat from it in more than `STALE_THRESHOLD_SECS` seconds. Why 150 seconds? The source (S13) explains the arithmetic: the DB heartbeat flush interval is 60 s, plus one daemon heartbeat cycle (~15 s), plus the scheduler tick (~30 s) = 105 s worst-case DB age for a completely healthy runner. 150 s leaves a 45 s buffer, and caps worst-case detection latency at `staleThreshold + sweepInterval = 180 s` (about 3 minutes).

```ts
// src/orchestrator/sweeper.ts (continued)

async function sweepStaleRunners(db: Database): Promise<void> {
  // 1. Find runners whose DB last_seen_at is older than the threshold.
  const candidates = await db.selectStaleOnlineRunners(STALE_THRESHOLD_SECS);
  if (candidates.length === 0) return;

  // 2. Cross-check against the liveness store (Redis) before flipping offline.
  //    The DB may lag; Redis has the hot heartbeat path.
  //    If the liveness store is unavailable, we fall through to DB-only behavior.
  const confirmed = await filterByLiveness(candidates, db.liveness);

  if (confirmed.length === 0) return;

  // 3. Atomically mark the confirmed stale runners offline.
  const offlined = await db.markRunnersOffline({
    ids: confirmed,
    staleSeconds: STALE_THRESHOLD_SECS,
  });

  if (offlined.length === 0) return; // raced — someone else got there first

  // 4. Clean up liveness-store keys so a stray Redis TTL doesn't resurface them.
  for (const runner of offlined) {
    db.liveness.forget(runner.id);
  }

  // 5. Fail any dispatched/running tasks that belong to those now-offline runners.
  const failedTasks = await db.failTasksForOfflineRunners();
  if (failedTasks.length > 0) {
    await handleFailedTasks(db, failedTasks);
  }
}
```

A key design choice here is the two-phase check (S13): we first query the database for stale candidates, then cross-check against the liveness store (backed by Redis) before actually flipping any runner offline. The DB is the durable record but is allowed to lag; Redis is the authority on the hot heartbeat path. If Redis is unavailable, we degrade gracefully and trust the DB stale window — matching the behavior before the Redis layer existed.

`handleFailedTasks` is the shared failure pipeline that fires `task:failed` events, reconciles the agent's status, rolls back the parent issue, and schedules automatic retries. We funnel through it rather than directly updating rows because the side-effects (event broadcast, retry scheduling, UI notification) must fire uniformly whether a task fails due to a crash, a timeout, or a manual cancel.

### filterByLiveness — cross-checking Redis

```ts
// src/orchestrator/sweeper.ts (continued)

async function filterByLiveness(
  candidates: Runner[],
  store: LivenessStore,
): Promise<string[]> {
  // If Redis is not configured, flip everything — original DB-only behavior.
  if (!store.available()) {
    return candidates.map(c => c.id);
  }

  const ids = candidates.map(c => c.id);
  const aliveMap = await store.isAliveBatch(ids);

  // Keep only runners that Redis also considers dead (or has no record of).
  return ids.filter(id => !aliveMap[id]);
}
```

You might wonder: what is a `LivenessStore`? It is a thin interface over a key-value store (typically Redis) where each runner writes a heartbeat key with a TTL. `isAliveBatch` does a batch `MGET` and returns which ids have an active key. If the store is unreachable, we fall back to the DB-stale path (S13).

### sweepStaleTasks — the safety net when the runner is still online

What if the runner is alive (heartbeating normally) but the agent process it started has hung? The runner's heartbeat will keep it from being flipped offline, but the task will never complete.

This is what `sweepStaleTasks` covers (S13). It fails any task that has been stuck in `dispatched` for more than `DISPATCH_TIMEOUT_SECS` (5 minutes — the `dispatched→running` transition should be near-instant), or in `running` for more than `RUNNING_TIMEOUT_SECS` (150 minutes — a generous server-side backstop for very long runs).

```ts
// src/orchestrator/sweeper.ts (continued)

async function sweepStaleTasks(db: Database): Promise<void> {
  const failedTasks = await db.failStaleTasks({
    dispatchTimeoutSecs: DISPATCH_TIMEOUT_SECS,
    runningTimeoutSecs: RUNNING_TIMEOUT_SECS,
  });
  if (failedTasks.length === 0) return;

  await handleFailedTasks(db, failedTasks);
}
```

The `runningTimeoutSecs` threshold (S13) is intentionally generous. The runner's own watchdog is responsible for detecting per-task idleness (no tool calls for N minutes); this server-side backstop is only for the case where the runner's process itself disappears without reporting. Setting it too aggressively would kill legitimate long-running analyses.

### sweepExpiredQueuedTasks — draining the backlog

There is a subtler scenario: a runner goes offline *after* tasks have already been queued for it. Those tasks are `queued` — not `in_progress` — so `sweepStaleRunners` will not touch them. They will wait in the queue forever, invisible to any other runner.

`sweepExpiredQueuedTasks` expires tasks that have been `queued` past `QUEUED_TTL_SECS` (2 hours, S13). Two hours is well above any reasonable "queued behind a long-running task" window, so we do not expire legitimately-pending work.

To keep DB transactions short when clearing a large backlog, the sweep processes at most `QUEUED_EXPIRE_BATCH_SIZE` (500) rows per tick (S13):

```ts
// src/orchestrator/sweeper.ts (continued)

async function sweepExpiredQueuedTasks(db: Database): Promise<void> {
  const failedTasks = await db.expireStaleQueuedTasks({
    ttlSecs: QUEUED_TTL_SECS,
    maxPerTick: QUEUED_EXPIRE_BATCH_SIZE,
  });
  if (failedTasks.length === 0) return;

  await handleFailedTasks(db, failedTasks);
}
```

### gcRunners — cleaning up long-dead runners

Over time, offline runners accumulate in the database. `gcRunners` deletes offline runners with no active agents that have exceeded `OFFLINE_RUNNER_TTL_SECS` (7 days, S13). Seven days gives an operator plenty of time to restart a daemon and re-claim ownership:

```ts
// src/orchestrator/sweeper.ts (continued)

async function gcRunners(db: Database): Promise<void> {
  const deleted = await db.deleteStaleOfflineRunners(OFFLINE_RUNNER_TTL_SECS);
  if (deleted.length === 0) return;

  // Notify connected frontends so they remove the stale runner from their lists.
  const workspaceIds = [...new Set(deleted.map(r => r.workspaceId))];
  for (const wsId of workspaceIds) {
    db.bus.publish({ type: "runner:gc", workspaceId: wsId });
  }
}
```

The FK constraint on agents uses `ON DELETE RESTRICT`, so the DB query must archive or remove any agents belonging to the runner before deleting the runner record itself (S13).

---

### Sweeper thresholds at a glance

| Constant | Value (S13) | What it governs |
|---|---|---|
| `SWEEP_INTERVAL_MS` | 30 s | How often the tick fires |
| `STALE_THRESHOLD_SECS` | 150 s | Heartbeat age that marks a runner stale |
| `DISPATCH_TIMEOUT_SECS` | 300 s (5 min) | Max time in `dispatched` state |
| `RUNNING_TIMEOUT_SECS` | 9 000 s (2.5 h) | Max time in `running` state |
| `QUEUED_TTL_SECS` | 7 200 s (2 h) | Max time a task may sit `queued` |
| `QUEUED_EXPIRE_BATCH_SIZE` | 500 rows | Max tasks expired per tick |
| `OFFLINE_RUNNER_TTL_SECS` | 604 800 s (7 d) | How long before an offline runner is GC'd |

These values form a layered defense: Redis heartbeat → DB stale window → per-state timeouts → queued TTL. Each layer catches a failure mode the previous one misses.

---

## Step 3 — Orphan recovery on restart

The sweeper handles the steady-state case. But there is a gap: when the *orchestrator itself* restarts, every task that was `in_progress` or `dispatched` at the moment of the restart is dangling. The sweeper will eventually detect these (after the stale threshold passes), but we can do better.

The source (S5) notes the design explicitly:

> "the runtime heartbeat sweeper takes up to 75s + the in-process task timeout (2.5h) to notice such tasks; the daemon itself knows the moment it comes back up, so we let it report orphan recovery."

Rather than waiting for the sweeper's slow clock, we let each runner call the orchestrator at startup and say: "I just came back — please recover any tasks you think I was running." The orchestrator atomically fails those tasks and immediately funnels them through the shared failure pipeline, so the UI sees the failure event and the retry is scheduled without delay.

```ts
// src/orchestrator/routes/runner-lifecycle.ts

/**
 * POST /api/runners/:runnerId/recover-orphans
 *
 * Called by the runner at startup. Atomically fails any dispatched/running
 * tasks the orchestrator believes belong to this runner — those are tasks
 * the previous runner process held when it crashed — and triggers the
 * standard failure pipeline for each so retries are scheduled and the UI
 * is notified immediately.
 */
async function recoverOrphanedTasks(
  req: Request,
  res: Response,
  db: Database,
): Promise<void> {
  const { runnerId } = req.params;

  // Atomically fail all dispatched/running tasks for this runner.
  // The query returns the rows that were actually transitioned so we can
  // funnel them through the failure pipeline.
  const orphanedTasks = await db.recoverOrphanedTasksForRunner(runnerId);

  // Run through the shared post-failure pipeline:
  //   task:failed events, agent reconcile, issue rollback, auto-retry.
  // This was previously a fast-path that bypassed those side effects,
  // leaving the UI stale when no retry was created (S5).
  const retried = await handleFailedTasks(db, orphanedTasks);

  if (orphanedTasks.length > 0) {
    console.info("recover-orphans completed", {
      runnerId,
      orphaned: orphanedTasks.length,
      retried,
    });
  }

  res.status(200).json({ orphaned: orphanedTasks.length, retried });
}
```

The crucial detail from S5: previously this was a "fast path" that bypassed `handleFailedTasks`. That meant when a task had exhausted its retry budget (or was non-retryable), the UI never saw the failure event, the issue stayed `in_progress`, and the user had no way to know. Routing through `handleFailedTasks` ensures every side-effect fires uniformly, regardless of whether a retry is created.

---

## Step 4 — Session and working-directory pinning

Now that tasks can be retried automatically, we face a new problem: how should the retry resume? If an agent was halfway through a multi-file refactor when the runner crashed, starting over from a blank session wastes the work already done and confuses the user.

The solution is **session pinning** (S5). As soon as the runner starts an agent and receives its first system message, it pins the agent's session id and working directory back to the task record. If the task is retried, the next runner reads these pinned values and resumes the existing conversation instead of starting fresh.

```ts
// src/orchestrator/routes/task-lifecycle.ts

interface PinTaskSessionRequest {
  sessionId?: string  // The agent's session identifier (e.g. Claude Code session id)
  workDir?: string    // The working directory the agent was using
}

/**
 * PATCH /api/tasks/:taskId/session
 *
 * Called by the runner right after the agent emits its first system message.
 * Persists the session id and work dir so a crash mid-run doesn't lose
 * the resume pointer needed to continue the conversation on the next attempt.
 * At least one of sessionId or workDir must be provided (S5).
 */
async function pinTaskSession(
  req: Request<{ taskId: string }, {}, PinTaskSessionRequest>,
  res: Response,
  db: Database,
): Promise<void> {
  const { taskId } = req.params;
  const { sessionId, workDir } = req.body;

  if (!sessionId && !workDir) {
    res.status(400).json({ error: "session_id or work_dir required" });
    return;
  }

  await db.updateTaskSession({ taskId, sessionId, workDir });
  res.status(204).end();
}
```

The runner calls this endpoint once, right after the first system message appears. From that point on, any automatic retry can inherit the session.

---

## Step 5 — Rerun semantics: fresh vs. inherited session

There are two different reasons a task might be retried, and they have opposite needs when it comes to session inheritance (S5):

**Automatic retry (infrastructure failure).** The runner crashed — the conversation itself was fine. We want the next attempt to continue from the same session. The prior output was not bad; the system just hiccupped.

**User-initiated rerun.** The user clicked "Rerun". They clicked it because they judged the prior output bad. Continuing from the same session would replay the same poisoned conversation state. The user wants a fresh start.

The source (S5) models this with a `force_fresh_session` flag on the task:

```ts
// src/orchestrator/routes/issue-lifecycle.ts

interface RerunIssueRequest {
  // If provided, target the agent that ran this specific past task
  // (rather than the issue's current assignee). Useful when the user
  // clicks "retry" on a specific row in the execution log.
  taskId?: string
}

/**
 * POST /api/issues/:taskId/rerun
 *
 * Manually re-enqueues an agent run for the issue.
 * Sets force_fresh_session=true on the new task so the runner skips the
 * (agentId, taskId) session-resume lookup and starts a clean session.
 *
 * Contrast with automatic retry (infrastructure failure): that path does NOT
 * set force_fresh_session, so the agent inherits the prior session and
 * resumes the conversation where it left off (S5).
 */
async function rerunIssue(
  req: Request<{ taskId: string }, {}, RerunIssueRequest>,
  res: Response,
  db: Database,
): Promise<void> {
  const { taskId } = req.params;
  const { taskId } = req.body ?? {};

  // Creates a new task row with force_fresh_session = true.
  // The runner's claim handler reads this flag and skips the session-resume
  // lookup, ensuring the agent opens a new conversation.
  const newTask = await db.enqueueRerun({
    taskId,
    sourceTaskId: taskId,     // target the specific past agent if provided
    forceFreshSession: true,  // user-initiated: always fresh (S5)
  });

  res.status(202).json(newTask);
}
```

To summarize the two paths:

| Trigger | `force_fresh_session` | Session behaviour |
|---|---|---|
| Automatic retry (infrastructure failure) | `false` | Runner inherits the pinned session; conversation continues |
| User-initiated rerun | `true` | Runner skips session-resume lookup; agent starts fresh |

This distinction matters because the runner's claim handler reads `force_fresh_session` before deciding whether to look up the `(agentId, taskId)` session. Getting it wrong in either direction produces a bad user experience: a poisoned session replayed on user-rerun, or a broken conversation resumed when the user just wanted recovery.

---

## Putting it all together

Here is the complete picture of the crash-recovery subsystem we have built:

```
Runner alive:  runner → heartbeat store (Redis, ~15 s interval)
               orchestrator ← sweeper tick (every 30 s)
                   └─ Redis check → DB check → mark offline if stale (150 s threshold)
                   └─ failTasksForOfflineRunners → handleFailedTasks
                   └─ sweepStaleTasks (dispatch 5 min, running 2.5 h timeouts)
                   └─ sweepExpiredQueuedTasks (queued TTL 2 h)
                   └─ gcRunners (offline > 7 days)

Runner restart:  runner → POST /runners/:id/recover-orphans (at startup)
                 orchestrator → recoverOrphanedTasksForRunner (atomic)
                             → handleFailedTasks (events + retries)

Task session:    runner → PATCH /tasks/:id/session (after first agent message)
                 runner (on retry) → reads pinned session_id + work_dir

User rerun:      user → POST /issues/:id/rerun
                 orchestrator → enqueueRerun(forceFreshSession: true)
                 runner → sees force_fresh_session, starts clean conversation
```

A task that was `in_progress` when a runner crashed will be recovered in one of two ways: immediately (if the runner restarts and calls the orphan-recovery endpoint) or within ~3 minutes (when the sweeper's stale threshold + tick interval elapses). Either path routes through `handleFailedTasks`, so retries, UI events, and issue-status reconciliation are always consistent.

---

## Try it yourself

Once you have the sweeper and orphan-recovery endpoint wired up, these exercises help you verify the behavior end-to-end:

**Exercise 1 — Kill a runner mid-task**

1. Start a runner and claim a task so it enters `in_progress`.
2. Kill the runner process without letting it deregister.
3. Wait for the sweeper's next tick (up to 30 s + 150 s stale threshold = ~3 minutes in the default configuration).
4. Observe in the database: the runner row should have `status = offline`; the task row should be `failed` with a failure reason of `agent_error` or similar; a retry task should be `queued` if the task had remaining attempts.

**Exercise 2 — Tune the stale threshold**

Lower `STALE_THRESHOLD_SECS` to 20 and `SWEEP_INTERVAL_MS` to 5000 in a dev environment. Kill a runner and observe how quickly the sweeper recovers the task. Then raise the threshold back — notice that very low thresholds can flip healthy runners offline if their heartbeat DB-flush lags under load.

**Exercise 3 — Force a fresh-session rerun**

1. Let an agent run to completion (or failure) with a session id pinned.
2. Call the rerun endpoint: `POST /api/issues/:id/rerun` with an empty body.
3. Inspect the new task row — it should have `force_fresh_session = true`.
4. Claim that task with a runner and verify the runner does not inherit the previous session id.

---

← Previous: [The Task Queue and Worker Claim Loop](./task-queue-and-claim-loop.md) · Next: [Agents as a Team: The Org Chart](../coordination/org-chart.md) →
