---
title: "The Task Queue and Worker Claim Loop"
description: Build the enqueue → claim → start pipeline with an atomic checkout so no two runners ever work the same task simultaneously.
category: tasks-and-queue
type: tutorial
tags:
  - task queue
  - claim loop
  - enqueue
  - claim
  - atomic checkout
  - 409 conflict
  - wakeup notifier
  - claimResponseRecoveryWindow
  - double-dispatch guard
  - EmptyClaimCache
  - in-memory cache
  - worker
  - runner
  - single assignee
  - task-available wakeup
  - FOR UPDATE SKIP LOCKED
  - queued
  - in_progress
  - checkout_run_id
  - execution_locked_at
  - stale lock reclaim
  - orphan recovery
  - SQLite
  - Drizzle
  - claim latency
keywords:
  - task dispatch
  - task steal prevention
  - worker concurrency
  - claim conflict
  - atomic SQL update
  - 409 checkout
  - runner poll loop
  - empty claim cache
  - invalidation version
  - wakeup notification
  - lease expiry
  - orphaned task
sources: [S18, S9, S5]
---

**TL;DR** — When many runners share one task queue, two can grab the same task unless the checkout step is atomic. This chapter builds the full pipeline: enqueue a task, claim it with a single SQL `UPDATE … WHERE` that stamps `checkout_run_id` and `execution_locked_at` while handing a `409` to the loser, then walk through two guards that keep a runner from firing the same task twice. By the end you will have a working claim loop backed by real concurrency safety, running on the same SQLite/Drizzle database you set up in the previous chapter.

# The Task Queue and Worker Claim Loop

Imagine you have ten runners — each is a background process that picks up tasks and invokes an agent adapter. You have one queue in the database. Every runner periodically asks: "Is there work for me?" If two runners ask at the same moment, and both get the same answer, you have a double-dispatch: one task, two concurrent agent runs, corrupt results.

That is the problem this chapter solves. We will build the claim pipeline in three stages:

1. **Enqueue** — insert a task row and broadcast that it is ready.
2. **Atomic checkout** — a single SQL `UPDATE … WHERE` that can only succeed for one claimant; everyone else gets a `409`.
3. **The worker claim loop** — how a runner polls (or is woken), attempts checkout, and on success calls the adapter.

Then we will add two guards that prevent a different class of double-fire: a **claim-response recovery window** and an **empty-claim cache** implemented entirely in process — no external service required.

---

## Prerequisites

Before going further, make sure you have the two concepts below in mind. If either is new, follow the linked chapter first — we will only recap here.

**Task schema and checkout columns.** Each task row carries three columns that the checkout step reads and writes atomically: `status` (which must equal `queued` for a claim to succeed), `checkout_run_id` (stamped with the claiming runner's run id), and `execution_locked_at` (the lock timestamp). These are introduced fully in [Modeling Tasks](./modeling-tasks.md). We will reference them throughout this chapter without re-defining them.

**Runners and adapters.** A runner is the worker process that connects to the orchestrator, claims tasks from the queue, and calls `adapter.invoke(agent, context)` to execute the agent. The adapter interface (`invoke` / `status` / `cancel`) is covered in [Your First Agent](../getting-started/your-first-agent.md). Here we focus on the claim loop that sits in front of that call.

---

## Step 1 — Enqueue: putting a task on the queue

The queue starts empty. Something has to put a task there. In Swarm, the orchestrator does this whenever an agent-assigned task needs to run — for example when an issue is created and assigned, when a comment mentions an agent, or when a scheduled trigger fires.

The insert itself is straightforward: create a row with `status = 'queued'`, stamp `agent_id`, `runtime_id`, and any link ids (issue, chat session, etc.).

```ts
// src/orchestrator/service/task.ts  (simplified view of enqueueTaskForIssue)

async function enqueueTaskForIssue(
  db: Database,
  issue: Issue,
  triggerCommentId?: string,
): Promise<TaskQueueRow> {
  if (!issue.assigneeId) {
    throw new Error("issue has no assignee");
  }

  const agent = await db.getAgent(issue.assigneeId);
  if (agent.archivedAt) throw new Error("agent is archived");
  if (!agent.runtimeId)  throw new Error("agent has no runtime");

  const task = await db.createAgentTask({
    agentId:          issue.assigneeId,
    runtimeId:        agent.runtimeId,
    taskId:          issue.id,
    priority:         priorityToInt(issue.priority),
    triggerCommentId: triggerCommentId ?? null,
  });

  // Broadcast "queued" before kicking the daemon — ordering matters.
  // The queued event must reach the UI before the dispatch event does,
  // otherwise the UI briefly shows "running" without having seen "queued".
  broadcastTaskEvent("task:queued", task);
  notifyTaskEnqueued(task); // invalidates empty-claim cache + kicks the runner
  return task;
}
```

The last call, `notifyTaskEnqueued`, does two things we will explain in detail later. For now, think of it as a hint to idle runners: "there is work available, come check."

Notice that the function validates the agent is not archived and has a runtime before inserting. A task with no runner will sit in `queued` forever — better to reject up front.

---

## Step 2 — The atomic checkout: one winner, everyone else gets 409

Now we have a task row with `status = 'queued'`. Multiple runners are polling. We need exactly one to succeed.

### Why a normal `SELECT` then `UPDATE` is not safe

You might think: "Each runner SELECTs the task, checks it is still `queued`, then UPDATEs it." But between the SELECT and the UPDATE, another runner can sneak in and do the same SELECT. Both see `queued`, both proceed to UPDATE, and you have two runners on the same task. The check-then-act window is a classic TOCTOU (time-of-check/time-of-use) race.

The fix is to collapse the check and the update into a **single statement with a conditional WHERE clause**. The database executes this atomically in one row lock:

```sql
-- The atomic checkout query (Drizzle equivalent shown below)
UPDATE agent_task_queue
SET
  status              = 'in_progress',
  checkout_run_id     = :runId,
  execution_locked_at = now(),
  started_at          = COALESCE(started_at, now())
WHERE
  id     = :taskId
  AND status IN ('queued', 'todo', 'backlog', 'blocked', 'in_review')
  AND (assignee_agent_id IS NULL OR assignee_agent_id = :agentId)
RETURNING *;
```

If this UPDATE touches zero rows — because `status` was already `in_progress` or a terminal state, or `checkout_run_id` was already set to someone else — we know we lost the race. In that case the server returns `409 Conflict` with the current owner and status.

### The checkout endpoint contract (from S18)

The orchestrator exposes this as a REST endpoint:

```
POST /issues/:taskId/checkout
```

Request body:

```json
{
  "agentId": "<uuid>",
  "expectedStatuses": ["todo", "backlog", "blocked", "in_review"]
}
```

Server behavior, verbatim from the spec (S18 §10.4.1):

1. Execute a **single SQL UPDATE** with `WHERE id = ? AND status IN (?) AND (assignee_agent_id IS NULL OR assignee_agent_id = :agentId)`.
2. If the updated row count is **0**, return **`409`** with the current owner and status.
3. On success, set `assignee_agent_id`, `status = in_progress`, and `started_at`.

The invariant is: **single assignee only**. A task in `in_progress` must have an assignee; there can never be two concurrent claimants.

### Drizzle implementation

Here is how that translates in TypeScript with Drizzle ORM:

```ts
// src/orchestrator/db/queries/claim.ts

import { eq, sql } from "drizzle-orm";
import { tasks } from "../schema";
import type { PgDatabase } from "drizzle-orm/pg-core";

export interface CheckoutResult {
  task: typeof tasks.$inferSelect;
}

export class CheckoutConflictError extends Error {
  constructor(
    public readonly currentStatus: string,
    public readonly currentOwnerId: string | null,
  ) {
    super(`checkout conflict: status=${currentStatus} owner=${currentOwnerId ?? "none"}`);
  }
}

export async function atomicCheckout(
  db: PgDatabase<any>,
  taskId: string,
  agentId: string,
  runId: string,
  expectedStatuses: string[],
): Promise<CheckoutResult> {
  const [updated] = await db
    .update(tasks)
    .set({
      status:             "in_progress",
      checkoutRunId:      runId,
      executionLockedAt:  sql`now()`,
      startedAt:          sql`COALESCE(started_at, now())`,
    })
    .where(
      sql`
        id = ${taskId}
        AND status = ANY(${expectedStatuses})
        AND (assignee_agent_id IS NULL OR assignee_agent_id = ${agentId})
      `,
    )
    .returning();

  if (!updated) {
    // Another runner won the race; fetch current state for the 409 body.
    const current = await db.query.tasks.findFirst({
      where: eq(tasks.id, taskId),
    });
    throw new CheckoutConflictError(
      current?.status ?? "unknown",
      current?.assigneeAgentId ?? null,
    );
  }

  return { task: updated };
}
```

The HTTP handler wraps this and translates `CheckoutConflictError` to a `409` response:

```ts
// src/orchestrator/handler/task_lifecycle.ts

import { CheckoutConflictError, atomicCheckout } from "../db/queries/claim";

export async function handleCheckout(req: Request, res: Response) {
  const { taskId } = req.params;
  const { agentId, expectedStatuses } = req.body;

  try {
    const { task } = await atomicCheckout(
      db,
      taskId,
      agentId,
      req.runId,              // stamped by auth middleware
      expectedStatuses ?? ["todo", "backlog", "blocked", "in_review"],
    );
    return res.status(200).json({ task });
  } catch (err) {
    if (err instanceof CheckoutConflictError) {
      return res.status(409).json({
        error:   "checkout_conflict",
        status:  err.currentStatus,
        ownerId: err.currentOwnerId,
      });
    }
    throw err;
  }
}
```

The runner gets a `409`, logs it, and moves on. The winning runner gets a `200` and proceeds to call the adapter.

---

## Step 3 — The worker claim loop

We now have an atomic checkout. Let us build the runner side: the loop that polls for available tasks and drives them through claim → start → invoke.

### The three-phase claim

The full claim for a runtime (a group of agent runners sharing one daemon process) has three phases:

| Phase | What happens |
|---|---|
| **Reclaim stale locks** | Check for tasks whose `execution_locked_at` has expired — a lock claimed but never completed. Clear those locks so another runner can pick them up (see §Step 4). |
| **Empty-cache check** | Ask the in-process cache: "Did we recently verify this runtime has no queued tasks?" If yes, skip the DB entirely. |
| **Claim from DB** | List queued candidates for this runtime; try to claim each agent's next task via `ClaimTask`. |

Here is the claim loop in TypeScript:

```ts
// src/orchestrator/service/claim.ts  (simplified view of claimTaskForRuntime)

export async function claimTaskForRuntime(
  db: Database,
  emptyClaim: EmptyClaimCache | null,
  runtimeId: string,
): Promise<TaskQueueRow | null> {
  // Phase 1: reclaim any task whose execution lock has expired for this runtime.
  // A lock is stale when execution_locked_at is older than the recovery window
  // and the task has not reached a terminal state — the runner that held the lock
  // either crashed or lost its connection before completing. Clear those locks
  // so the work is not permanently lost.
  const stale = await reclaimStaleLock(db, runtimeId);
  if (stale) {
    return stale;
  }

  // Phase 2: check the empty-claim cache before touching the DB.
  if (emptyClaim?.isEmpty(runtimeId)) {
    return null; // fast path: we know there is nothing to claim
  }

  // Phase 3: sample the invalidation version BEFORE the SELECT.
  // If an enqueue happens between here and MarkEmpty below, the bumped
  // version makes the cached empty verdict stale, and the next poll will
  // go through to the DB.
  const preSelectVersion = emptyClaim?.currentVersion(runtimeId) ?? 0n;

  const candidates = await db.listQueuedCandidatesByRuntime(runtimeId);
  if (candidates.length === 0) {
    emptyClaim?.markEmpty(runtimeId, preSelectVersion);
    return null;
  }

  // Try agents in order, one at a time, until one claim succeeds.
  const tried = new Set<string>();
  for (const candidate of candidates) {
    if (tried.has(candidate.agentId)) continue;
    tried.add(candidate.agentId);

    const task = await claimTask(db, candidate.agentId);
    if (task && task.runtimeId === runtimeId) {
      return task;
    }
  }
  return null;
}
```

### ClaimTask: one agent's atomic pick

`claimTask` handles the per-agent step: verify the agent has capacity, then run the `ClaimAgentTask` query. Internally, the query can use `SELECT … FOR UPDATE SKIP LOCKED` — a technique for picking the next claimable row without blocking concurrent callers — before executing the atomic `UPDATE` on the chosen row:

```ts
// src/orchestrator/service/claim.ts  (simplified view of claimTask)

export async function claimTask(
  db: Database,
  agentId: string,
): Promise<TaskQueueRow | null> {
  const agent = await db.getAgent(agentId);

  const running = await db.countRunningTasks(agentId);
  if (running >= agent.maxConcurrentTasks) {
    return null; // agent is at capacity
  }

  // ClaimAgentTask selects the next queued row for this agent and atomically
  // sets status = 'in_progress', checkout_run_id, and execution_locked_at.
  // It returns null if nothing is queued for this agent right now.
  const task = await db.claimAgentTask(agentId);
  if (!task) {
    return null; // nothing in queue for this agent
  }

  reconcileAgentStatus(db, agentId);
  broadcastTaskDispatch(task);
  return task;
}
```

### The start call and session pinning

After a successful claim the task is `in_progress` — `checkout_run_id` and `execution_locked_at` are already stamped. The runner now calls the adapter. As soon as the agent emits its first output, the runner can **pin the session ID** — the identifier the agent runtime uses to resume a conversation — via `POST /tasks/:id/pin-session` (S5). The runner should call this before awaiting completion, so a crash mid-run does not lose the resume pointer:

```ts
// src/runner/loop.ts  (illustrative; translated from the daemon pattern in S9/S5)

async function runnerLoop(runtimeId: string) {
  while (true) {
    await sleepOrWakeup(POLL_INTERVAL_MS); // woken early by NotifyTaskAvailable

    const task = await claimTaskForRuntime(db, emptyClaim, runtimeId);
    if (!task) continue;

    // task.status is now 'in_progress'; checkout_run_id and
    // execution_locked_at are stamped by the atomic checkout.
    const adapter = resolveAdapter(task.agentId);
    adapter.on("firstMessage", async ({ sessionId, workDir }) => {
      // Pin session early so a crash mid-run does not lose the resume pointer.
      await orchestratorClient.post(`/tasks/${task.id}/pin-session`, {
        sessionId,
        workDir,
      });
    });

    adapter.invoke(task).then(async (result) => {
      await orchestratorClient.post(`/tasks/${task.id}/complete`, { result });
    }).catch(async (err) => {
      await orchestratorClient.post(`/tasks/${task.id}/fail`, { error: err.message });
    });
    // Do not await the invoke — continue polling for more tasks in parallel.
  }
}
```

Notice we do not `await` the `invoke` call before continuing the loop. The runner can claim another task for a different agent while the first is still executing, up to each agent's `maxConcurrentTasks` limit.

---

## Step 4 — Double-dispatch guards

Two scenarios can cause a task to be worked twice even with the atomic checkout in place.

### Guard 1: The claim-response recovery window (stale lock reclaim)

Here is the sequence for the first scenario:

1. Runner A claims a task → `status = 'in_progress'`, `checkout_run_id` = runner A's id, `execution_locked_at` = now.
2. Runner A's process crashes or its network connection to the orchestrator drops before any progress is recorded.
3. The task sits `in_progress` with a stale lock — `execution_locked_at` is old, the runner that held it is gone.

The fix (grounded in S5 and S18) is a **stale-lock reclaim**: a task whose `execution_locked_at` is older than a threshold and has not reached a terminal state is considered orphaned. Another runner can clear the lock by resetting `checkout_run_id` to its own id, refreshing `execution_locked_at` to now, and re-delivering the task.

The threshold must exceed the combined client timeout for the claim call plus any start acknowledgement latency plus scheduling slack, so an in-flight run cannot be accidentally pre-empted. A value around 90 seconds is typically sufficient:

```ts
// src/orchestrator/service/claim.ts

// CLAIM_RESPONSE_RECOVERY_WINDOW_SEC must exceed the daemon client timeout for
// the claim round-trip plus scheduling slack, so an active run's lock cannot
// be mistakenly cleared. Source: S9 comment on claimResponseRecoveryWindow.
const CLAIM_RESPONSE_RECOVERY_WINDOW_SEC = 90;

async function reclaimStaleLock(
  db: Database,
  runtimeId: string,
): Promise<TaskQueueRow | null> {
  // A stale lock is one where execution_locked_at is older than the recovery
  // window and the task has not completed. We reclaim it by refreshing the
  // lock columns (checkout_run_id = this runner, execution_locked_at = now)
  // so the task can be worked again without being reset to 'queued'.
  return db.reclaimStaleLockedTaskForRuntime(
    runtimeId,
    CLAIM_RESPONSE_RECOVERY_WINDOW_SEC,
  );
}
```

The SQL that backs this is a targeted `UPDATE` that **re-delivers** the task by refreshing the lock columns in place:

```sql
-- Reclaims a task whose execution lock has expired.
-- We re-stamp checkout_run_id and execution_locked_at rather than
-- resetting status to 'queued', so the task stays in_progress and
-- the runner that picks it up can resume where the prior attempt left off.
UPDATE agent_task_queue
SET
  checkout_run_id     = :newRunId,
  execution_locked_at = now()
WHERE id = (
  SELECT atq.id FROM agent_task_queue atq
  WHERE  atq.runtime_id          = :runtimeId
    AND  atq.status              = 'in_progress'
    AND  atq.execution_locked_at < now() - make_interval(secs => :recoverySecs)
    AND  atq.completed_at        IS NULL
  ORDER BY atq.priority DESC, atq.execution_locked_at ASC
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

Two things are worth noticing here:

- The task **stays `in_progress`** — the mechanism refreshes `checkout_run_id` and `execution_locked_at` without touching `status`. The new runner picks up from where the old one left off (or starts fresh if no session was pinned).
- The age-check column is `execution_locked_at`, the same column the original atomic checkout stamped. There is no separate `dispatched_at` timestamp; the lock lifecycle lives entirely in `checkout_run_id` + `execution_locked_at`.

This reclaim runs **before** the empty-claim cache check in `claimTaskForRuntime`. A stale lock means the task is `in_progress`, not `queued`, so the empty-queued cache cannot represent recoverability — we must check for expired locks on every poll cycle.

For the full orphan-recovery story — crash detection, atomically failing all tasks a dead runtime owned, and the `/recover-orphans` endpoint that a daemon calls at startup — see [Crash Recovery and Liveness](./crash-recovery-and-liveness.md). The `reclaimStaleLock` path above handles only the narrow per-task lease expiry; the startup orphan sweep (S5) handles the broader "runtime died mid-run" case.

### Guard 2: The empty-claim cache (avoid hammering the DB)

In the steady state, most poll cycles for an idle runtime produce zero tasks. Without a cache, each runner hits the database on every tick. With many runners across many runtimes, this becomes significant read pressure.

The `EmptyClaimCache` caches the negative result: "this runtime has no queued tasks right now." Here is how we implement it entirely in process — no external service required:

```ts
// src/orchestrator/service/empty_claim_cache.ts

// How this works:
//   - For each runtime we store two things in memory:
//       emptyVersion: the version under which we last confirmed "empty"
//       currentVersion: a monotonic counter bumped on every enqueue
//   - isEmpty() returns true only when emptyVersion === currentVersion
//     (i.e., no enqueue has fired since we last confirmed emptiness).
//   - markEmpty() writes emptyVersion = observedVersion.
//   - bump() increments currentVersion, making any in-flight markEmpty stale.
//   - Entries expire after EMPTY_CLAIM_CACHE_TTL_MS so a crash or missed
//     invalidation does not keep the cache wrong forever.

export const EMPTY_CLAIM_CACHE_TTL_MS = 3 * 60 * 1000; // 3 minutes

interface CacheEntry {
  emptyVersion: bigint;   // version when "empty" was last confirmed
  expiresAt:    number;   // Date.now() + TTL — safety net for missed bumps
}

export class EmptyClaimCache {
  private readonly versions  = new Map<string, bigint>();     // currentVersion per runtimeId
  private readonly empties   = new Map<string, CacheEntry>(); // empty verdicts per runtimeId

  // Read the current invalidation version BEFORE the SELECT.
  // Pass the returned value to markEmpty after confirming emptiness.
  currentVersion(runtimeId: string): bigint {
    return this.versions.get(runtimeId) ?? 0n;
  }

  // Returns true only when: (a) an empty verdict is cached AND
  // (b) it was written under the current version (i.e., no enqueue
  // has happened since the last DB check) AND (c) it has not expired.
  isEmpty(runtimeId: string): boolean {
    const entry = this.empties.get(runtimeId);
    if (!entry) return false;
    if (Date.now() > entry.expiresAt) {
      this.empties.delete(runtimeId);
      return false;
    }
    return entry.emptyVersion === this.currentVersion(runtimeId);
  }

  // Store "no tasks" tagged with the version observed before the SELECT.
  // A concurrent bump() makes the next reader reject this entry.
  markEmpty(runtimeId: string, observedVersion: bigint): void {
    this.empties.set(runtimeId, {
      emptyVersion: observedVersion,
      expiresAt:    Date.now() + EMPTY_CLAIM_CACHE_TTL_MS,
    });
  }

  // Called by every enqueue path BEFORE the wakeup signal.
  // Increments the version so any in-flight markEmpty call
  // writes a now-stale version that isEmpty will reject.
  bump(runtimeId: string): void {
    this.versions.set(runtimeId, (this.versions.get(runtimeId) ?? 0n) + 1n);
  }
}
```

The **version tag** is the critical detail. Consider this race without it:

```
T1 claim:   SELECT → empty
            (GC pause, slow network…)
T2 enqueue: INSERT row
            bump version  (v0 → v1)
            wakeup signal
T1 claim:   markEmpty("no tasks")       ← stale!
T3 claim:   isEmpty() = true            ← wrong! task never claimed
```

With the version tag, T3 reads version `v1` but the cached entry holds `v0`, sees they differ, treats it as a cache miss, and hits the DB. The task gets claimed.

The full sequence in `claimTaskForRuntime` is:

```
1. preSelectVersion = currentVersion(runtimeId)    // read version first
2. candidates = SELECT queued tasks                // DB read
3. if candidates empty:
     markEmpty(runtimeId, preSelectVersion)        // tag with pre-select version
4. otherwise: try to claim
```

If an enqueue fires between steps 1 and 3, it bumps the version. Step 3's `markEmpty` writes a now-stale version. The next `isEmpty` call sees version mismatch → cache miss → DB hit. The task is claimed within one poll cycle.

> A `null` / unconfigured `EmptyClaimCache` is safe: the optional-chaining calls in `claimTaskForRuntime` (`emptyClaim?.isEmpty(...)`) become no-ops, and the claim loop falls through to the DB on every cycle. This is the correct default for single-process development.

> **Optional production upgrade — shared cache across multiple orchestrator instances.** The in-process cache above works correctly in a single-process deployment (one orchestrator process, one or more runner threads). If you later scale to multiple orchestrator processes behind a load balancer, each process holds its own independent cache, and a `bump()` in process A is invisible to processes B and C. In that case you can move the version counter and empty verdict to a shared store — Redis is a natural fit (`INCR` for the version key, `SET … EX` for the empty verdict). The interface and algorithm stay identical; only the backing storage changes. Chapter [Where to Go Next](../wrap-up/where-to-go-next.md) covers scaling considerations.

---

## Putting it all together: the full enqueue → claim → start flow

Here is the complete happy-path sequence for a single task:

```
Orchestrator                        Runner
    │                                  │
    │  INSERT task (status=queued)      │
    │  broadcastTaskEvent "queued"      │
    │  EmptyClaimCache.bump()           │
    │  Wakeup.NotifyTaskAvailable() ───►│ (woken from sleepOrWakeup)
    │                                  │
    │◄── POST /issues/:id/checkout ─────│ (claimTaskForRuntime → atomicCheckout)
    │  UPDATE … SET                     │
    │    status = 'in_progress'         │
    │    checkout_run_id = runId        │
    │    execution_locked_at = now()    │
    │──► 200 { task } ─────────────────►│
    │                                  │
    │                        adapter.invoke(agent, ctx)
    │                        (first message) →
    │◄── POST /tasks/:id/pin-session ───│
    │  UPDATE session_id, work_dir      │
    │──► 204 ──────────────────────────►│
    │                                  │
    │◄── POST /tasks/:id/complete ──────│
    │  UPDATE … SET status='done'       │
    │  broadcastTaskEvent "completed"   │
    │──► 200 ──────────────────────────►│
```

A losing runner at the claim step gets zero rows from `ClaimAgentTask`; it simply returns `null` from `claimTaskForRuntime` and waits for the next poll.

A runner that claims but crashes before making progress will have its task reclaimed after the recovery window (≈90 seconds) by the next runner that calls `reclaimStaleLock`. The P10 schema columns — `checkout_run_id` and `execution_locked_at` — carry the full lock lifecycle from first claim through expiry and re-claim. There is no separate timestamp column for this purpose.

---

## Try it yourself

The concepts above are most concrete when you can observe them directly. Here are three experiments to try once you have the orchestrator running locally.

> **Experiment 1 — Watch the 409 happen**
>
> Start two runner processes against one orchestrator. Create a task with `POST /workspaces/:id/issues`. In a tight loop from both runners, call `POST /issues/:id/checkout` simultaneously (use `curl --parallel` or two terminals). One should return `200`, the other `409`. Inspect the `409` body — it will tell you the current owner and status.

> **Experiment 2 — Observe stale-lock reclaim**
>
> Claim a task from runner A (it should now be `in_progress` with `checkout_run_id` and `execution_locked_at` set). Kill runner A without completing the task. Wait 91 seconds. On the next poll from runner B, the `reclaimStaleLock` path should pick it up and return it to runner B. You can confirm by checking `execution_locked_at` and `checkout_run_id` in the database before and after — `checkout_run_id` should switch from runner A's id to runner B's id, and `execution_locked_at` should be refreshed to the current time.

> **Experiment 3 — Log claim latency**
>
> Instrument your `claimTaskForRuntime` function with `Date.now()` timestamps at the three phases (stale-lock check, cache check, DB select). Under load, the empty-cache hit rate should be close to 100% for idle runtimes, and you should see occasional DB hits only after an enqueue event.

---

← Previous: [Modeling Tasks](./modeling-tasks.md) · Next: [Crash Recovery and Liveness](./crash-recovery-and-liveness.md) →
