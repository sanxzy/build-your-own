---
title: Modeling Tasks
description: Design the task entity and its state machine — status semantics, single-assignee guarantee, parent/child sub-tasks, and the locking columns that make atomic checkout safe.
category: tasks-and-queue
type: explanation
tags:
  [
    task model,
    issue,
    state machine,
    status semantics,
    backlog,
    todo,
    in_progress,
    blocked,
    in_review,
    done,
    cancelled,
    parentId,
    sub-issue,
    sub-task,
    single assignee,
    checkoutRunId,
    executionRunId,
    executionLockedAt,
    atomic checkout,
    originFingerprint,
    dedup,
    Drizzle schema,
    issue relations,
    blocks,
    blocked_by,
    related,
    task lifecycle,
    task status transitions,
    task ownership,
  ]
keywords:
  [
    task data model,
    agent task,
    task queue schema,
    issue entity,
    workflow state,
    liveness contract,
    execution lock,
    checkout race condition,
    atomic task claim,
    task deduplication,
  ]
sources: [S18, S20, S21, S37, S14]
---

**TL;DR** — Before agents can pick up work, we need a precise definition of what a unit of work *is*: its fields, the rules its status follows, and the locking columns that guarantee two agents never race over the same task. This chapter walks through the task data model — the `tasks` table in Drizzle, the seven-state machine with exact semantics for each value, the single-assignee invariant, parent/child sub-tasks, and the three "atomic-checkout" columns that the next chapter's claim loop depends on.

# Modeling Tasks

At this point in our Swarm orchestrator we have an agent adapter that can invoke and cancel an agent (see the [Adapter Interface chapter](../the-agent/adapter-interface.md)), and we understand how one execution window becomes a **run** — a record capturing session, usage, and cost (see [A Run: Sessions, Usage, and Cost](../the-agent/a-run.md)). What we do not yet have is an answer to a simpler question: what is the unit of work an agent is working *on*?

We will call that unit a **task** (trackers sometimes call it an "issue" — the two terms refer to the same concept). Tasks are the backbone of the queue chapters that follow. The claim loop in the next chapter cannot exist without a clear answer to: what fields does a task have, what statuses are legal, and how do we guarantee that only one agent can hold a task at a time?

Let us build the model from first principles.

## What a task needs to express

Before writing any schema, think about the questions a task must answer:

1. **What is it?** — title, description, priority, project, goal linkage.
2. **Where is it in its lifecycle?** — status: not yet ready, ready, being worked, waiting for review, done.
3. **Who owns it?** — a single assignee (agent or human), never two.
4. **Is it part of a larger task?** — a parent/child relationship for work breakdown.
5. **How did it arrive?** — a fingerprint for deduplication so a recurring trigger never creates the same task twice.
6. **Is it safe to claim right now?** — execution-lock columns that make checkout atomic.

Each of those concerns maps to a column group in the schema.

## The Drizzle schema

Here is the core `tasks` table as a Drizzle PostgreSQL schema. The names are our generic Swarm names; the structure is taken directly from the implementation.

```ts
// src/db/schema/tasks.ts
// Simplified view — monitor/workspace/attachment columns omitted for clarity.
import { sql } from "drizzle-orm";
import {
  type AnyPgColumn,
  pgTable,
  uuid,
  text,
  timestamp,
  integer,
  jsonb,
  index,
  uniqueIndex,
} from "drizzle-orm/pg-core";
import { agents } from "./agents.js";
import { projects } from "./projects.js";
import { goals } from "./goals.js";
import { workspaces } from "./workspaces.js";
import { runs } from "./runs.js";

export const tasks = pgTable(
  "tasks",
  {
    // --- Identity ---
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id),

    // --- Context ---
    projectId: uuid("project_id").references(() => projects.id),
    goalId: uuid("goal_id").references(() => goals.id),

    // --- Structure ---
    parentId: uuid("parent_id").references((): AnyPgColumn => tasks.id),

    // --- Content ---
    title: text("title").notNull(),
    description: text("description"),
    status: text("status").notNull().default("backlog"),
    priority: text("priority").notNull().default("medium"),

    // --- Ownership (single-assignee invariant) ---
    assigneeAgentId: uuid("assignee_agent_id").references(() => agents.id),
    assigneeUserId: text("assignee_user_id"),
    // Invariant: assigneeAgentId and assigneeUserId cannot both be set.

    // --- Atomic-checkout columns ---
    checkoutRunId: uuid("checkout_run_id")
      .references(() => runs.id, { onDelete: "set null" }),
    executionRunId: uuid("execution_run_id")
      .references(() => runs.id, { onDelete: "set null" }),
    executionLockedAt: timestamp("execution_locked_at", { withTimezone: true }),

    // --- Provenance / dedup ---
    originKind: text("origin_kind").notNull().default("manual"),
    originId: text("origin_id"),
    originRunId: text("origin_run_id"),
    originFingerprint: text("origin_fingerprint").notNull().default("default"),

    // --- Human-readable identifier ---
    taskNumber: integer("task_number"),
    identifier: text("identifier"),   // e.g. "ENG-42"

    // --- Delegation depth ---
    requestDepth: integer("request_depth").notNull().default(0),

    // --- Lifecycle timestamps ---
    startedAt: timestamp("started_at", { withTimezone: true }),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    cancelledAt: timestamp("cancelled_at", { withTimezone: true }),

    // --- Soft-hide ---
    hiddenAt: timestamp("hidden_at", { withTimezone: true }),

    createdAt: timestamp("created_at", { withTimezone: true })
      .notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true })
      .notNull().defaultNow(),
  },
  (table) => ({
    workspaceStatusIdx: index("tasks_company_status_idx")
      .on(table.workspaceId, table.status),
    assigneeStatusIdx: index("tasks_company_assignee_status_idx")
      .on(table.workspaceId, table.assigneeAgentId, table.status),
    parentIdx: index("tasks_company_parent_idx")
      .on(table.workspaceId, table.parentId),
    projectIdx: index("tasks_company_project_idx")
      .on(table.workspaceId, table.projectId),
    originIdx: index("tasks_company_origin_idx")
      .on(table.workspaceId, table.originKind, table.originId),
    identifierIdx: uniqueIndex("tasks_identifier_idx").on(table.identifier),
  }),
);
```

Let's walk through each column group and understand *why* each one is there.

### Identity and context

`id` is the UUID primary key. `workspaceId` enforces the multi-tenant boundary — every query that touches tasks must filter by company. `projectId` and `goalId` optionally place the task inside a project or link it to a strategic goal.

### The `parentId` self-reference

`parentId` is a foreign key that points back to the `tasks` table itself. Setting it on a task makes that task a **sub-task** of the parent. This is work-breakdown: an agent decomposes a large task into smaller pieces, each of which becomes a sub-task. We cover sub-tasks in detail in a later chapter; for now, notice that the relationship is purely structural — it explains *why* a child task exists, not *when* it can be worked.

### The single-assignee invariant

`assigneeAgentId` and `assigneeUserId` are mutually exclusive. At most one of them can be set at a time — the invariant is: a task has at most one assignee. This is deliberate. Clear ownership prevents diffusion of responsibility. When multiple agents need to collaborate on a body of work, the pattern is to create separate sub-tasks with different assignees, not to attach multiple agents to a single task.

In the Swarm control plane, the assignee tells you *who is responsible*. A separate set of columns (the atomic-checkout columns, below) tell you *whether that agent currently has an active execution path*. Keeping ownership and execution separate is important, and we will return to this distinction when we study the four separable concepts.

### Lifecycle timestamps

When a task transitions into a "started" status, `startedAt` is set (if it was null). When it reaches `done`, `completedAt` is set. When it reaches `cancelled`, `cancelledAt` is set. These timestamps are set by the state-machine transition logic, not by the caller.

### The atomic-checkout columns

These three columns are the bridge from ownership to active execution:

| Column | Role |
|---|---|
| `checkoutRunId` | Identifies which run currently holds the **issue-ownership lock** for this task |
| `executionRunId` | Identifies which run is **actively executing** right now |
| `executionLockedAt` | Timestamp when the execution lock was acquired |

Notice that `checkoutRunId` and `executionRunId` are related but not identical. `checkoutRunId` answers "who currently owns execution rights?" — it is the ownership lock. `executionRunId` answers "which run is actually live right now?" — it is the active execution pointer. A checkout can outlive one run if the agent picks the task back up in a new run; the execution run changes, but the checkout run may stay the same assignee.

These columns are what make the atomic checkout in the next chapter safe. The claim endpoint issues a single SQL `UPDATE ... WHERE id = ? AND status IN (?) AND (assignee_agent_id IS NULL OR assignee_agent_id = :agentId)` — if zero rows are updated, a concurrent agent already claimed the task and the requester gets a `409`. We'll build that loop in [The Task Queue and Worker Claim Loop](./task-queue-and-claim-loop.md).

### Provenance and deduplication

`originKind`, `originId`, `originRunId`, and `originFingerprint` let the orchestrator track where a task came from and prevent duplicates.

`originFingerprint` defaults to `"default"` and is particularly useful for recurring executions: a partial unique index on `(workspaceId, originKind, originId, originFingerprint)` filtered to non-terminal, non-hidden tasks ensures that a routine firing again does not produce a second active task when one is already in flight. The fingerprint encodes "which specific invocation of which origin created this task."

### Indexes

The schema ships several composite indexes because the orchestrator's hot paths all filter by `workspaceId` first:

```
tasks_company_status_idx        — list tasks by status within a company
tasks_company_assignee_status_idx — find an agent's open tasks
tasks_company_parent_idx        — walk the sub-task tree
tasks_company_project_idx       — list tasks within a project
tasks_company_origin_idx        — dedup lookups by origin
tasks_identifier_idx            — unique human-readable identifier
```

The identifier index is a unique index because human-readable identifiers like `ENG-42` must be globally unique within the system.

## The seven-status state machine

Now we understand the shape of a task. The most consequential part of the model is its status field. The statuses are not just UI labels — each one carries precise expectations about ownership and execution.

### The four separable concepts (framing)

Before mapping the states, it helps to hold four concepts separately, because the statuses express different combinations of them:

1. **Structure** — parent/child relationship (`parentId`)
2. **Dependency** — whether this task is blocked by another (relations)
3. **Ownership** — who is responsible (`assigneeAgentId` / `assigneeUserId`)
4. **Execution** — whether the control plane currently has a live path to move this task forward

The temptation is to blur these. A task with `parentId` set might *feel* like it depends on its parent — but `parentId` is structural, not dependency semantics. Actual blocking dependencies use `blocks` / `blocked_by` relations (covered below). Similarly, `in_progress` expresses both ownership *and* execution expectations, while `todo` expresses ownership without an execution expectation. Keeping the four concepts separate is the key to reasoning about task health.

### Status definitions

| Status | Category | Meaning |
|---|---|---|
| `backlog` | Parked | Not ready for active work. No execution expectation. Safe resting state. |
| `todo` | Ready | Actionable but not yet claimed. May or may not have an assignee. No execution lock required yet. |
| `in_progress` | Active | Actively owned work. Requires an assignee. For agent-owned tasks, this is a strict execution-backed state — it should never become a silent dead state. |
| `blocked` | Waiting | Cannot proceed until something external changes. The right state for waiting on another task, a human decision, or an external dependency. |
| `in_review` | Review | Execution is paused because the next move belongs to a reviewer or approver, not the current executor. |
| `done` | Terminal | Work is complete. |
| `cancelled` | Terminal | Work will not continue. |

A key nuance: `in_progress` means something different for agent-owned vs. human-owned tasks. For an agent-owned task, `in_progress` is a strict execution-backed state — the control plane can wake the agent, track runs linked to the task, and recover lost execution state after crashes. For a human-owned task, `in_progress` is simply a human ownership state; the heartbeat scheduler does not manage it.

This asymmetry is why the system distinguishes `assigneeAgentId` and `assigneeUserId` rather than using a single polymorphic field.

### Legal transitions

```
          ┌─────────────────────────────────────────┐
          │                                         │
          ▼                                         │
       backlog ──────────────────────── cancelled ◄─┤
          │                                 ▲       │
          │                                 │       │
          ▼                                 │       │
         todo ─────────────── blocked ──────┘       │
          │      ▲              │ ▲                  │
          │      │              │ │                  │
          ▼      │              ▼ │                  │
      in_progress ◄─────── in_review ────────────── ┤
          │                    ▲                    │
          └────────────────────┘                    │
          │                                         │
          └──────────────────────── done ◄──────────┘
```

The legal transitions, stated explicitly:

```
backlog     → todo | cancelled
todo        → in_progress | blocked | cancelled
in_progress → in_review | blocked | done | cancelled
in_review   → in_progress | blocked | done | cancelled
blocked     → todo | in_progress | cancelled
done        (terminal)
cancelled   (terminal)
```

Notice that `done` and `cancelled` are **terminal** — once a task reaches either of those states it cannot transition anywhere else. The states `backlog` and `todo` are non-terminal but require no live execution path. The states `in_progress`, `blocked`, and `in_review` are non-terminal and carry expectations about liveness that the watchdog must validate.

### Side effects of transitions

When a task enters `in_progress`, `startedAt` is set if it was null. When it enters `done`, `completedAt` is set. When it enters `cancelled`, `cancelledAt` is set. These side effects are handled automatically by the transition logic, not by the caller.

One additional constraint: entering `in_progress` requires an assignee. You cannot move a task to `in_progress` without first setting `assigneeAgentId` or `assigneeUserId`.

### Status semantics in depth

Let us look at the less obvious statuses more carefully.

**`todo` as dispatch state.** A `todo` task is ready to start, not yet actively claimed. When an agent is assigned to a `todo` task, the control plane may still need a wake path to ensure the assignee actually picks it up. An assigned `todo` task that has no queued wake and no active run is potentially stalled — the system should surface that rather than silently leaving it. This is dispatch state, not parked state.

**`backlog` vs `todo`.** The distinction matters: `backlog` is *parked* — no execution expectation, no urgency. `todo` is *dispatch-ready*. When a task is created with an assignee but no explicit status, the control plane defaults to `todo` so the assignee gets a wake path instead of silently inheriting the unassigned `backlog` default.

**`blocked` as explicit waiting.** A task in `blocked` must have a clear reason: waiting on another task, waiting on a human decision, or waiting on an external dependency. A blocked task is not stalled — it is explicitly waiting with a named reason. A task that has *no active path forward at all* is stalled, not `blocked`. The control plane surfaces stalled tasks as recovery work.

**`in_review` as handoff.** When an agent finishes its part of the work and needs a reviewer to approve before proceeding, the task moves to `in_review`. The reviewer may be a typed execution-policy participant, a pending human approval, or another agent. An `in_review` task with no reviewer and no active monitor is stalled.

## Sub-tasks and the `parentId` relationship

We covered `parentId` in the schema section. Let's think about when to use it.

Use `parentId` for:
- Work breakdown — an agent decomposes a large task into smaller pieces
- Rollup context — understanding why a child task exists
- Waking the parent assignee when all direct children reach a terminal state

Do **not** treat `parentId` as execution dependency by itself. If a parent task truly cannot proceed until a child task completes, model that with a blocker relation (below) rather than relying on the structural parent/child link alone. The parent/child relationship explains structure; blockers explain waiting.

Sub-tasks inherit the project from their parent at creation time. They do not inherit assignee or labels.

## Issue relations: blocks, blocked_by, and related

Beyond `parentId`, tasks can relate to each other through an `task_relations` table. There are four relation types:

| Type | Meaning | Behavior |
|---|---|---|
| `related` | General connection | Informational; no execution effect |
| `blocks` | This task blocks another | The blocked task shows a flag |
| `blocked_by` | This task is blocked by another | Inverse of `blocks` |
| `duplicate` | This task duplicates another | Auto-moves the duplicate to `cancelled` |

Blocking is **not transitive at the system level**: if A blocks B and B blocks C, that does not automatically mean A blocks C. Each blocker relationship must be stated explicitly.

When a blocking task resolves (reaches `done` or `cancelled`), the blocked task should be woken so its assignee can decide whether to proceed. This wakeup is what the execution semantics layer watches for. We will see how the claim loop handles this when we study wakeups.

A blocked chain is considered healthy only when every unresolved leaf blocker has a valid action path. An intermediate `blocked` task does not make the chain healthy by itself; the system looks all the way to the leaves.

## Putting it together: the four concepts applied to a real scenario

Imagine an agent decomposes a large coding task into three sub-tasks — a schema migration, an API handler, and a UI component — and creates blockers so the API handler waits for the migration to finish:

```
Task A: "Add user preferences feature"   [in_progress, agent Alice]
  ├── Sub-task B: "Migration"            [in_progress, agent Bob]  ← parentId = A
  ├── Sub-task C: "API handler"          [blocked, agent Carol]    ← parentId = A, blocked_by = B
  └── Sub-task D: "UI component"         [todo, agent Dave]        ← parentId = A
```

Here we see all four concepts at work:

- **Structure**: B, C, D are sub-tasks of A via `parentId`
- **Dependency**: C is blocked by B via `blocked_by` relation
- **Ownership**: Alice, Bob, Carol, Dave are the respective assignees
- **Execution**: B has a live run; C is waiting (healthy `blocked` because its blocker B has a live path); D is in dispatch state (healthy `todo`)

Task A stays `in_progress` even though Alice is not actively doing work — she is the responsible owner; her execution continuity is managed by the control plane through the sub-task structure.

## What the atomic-checkout columns power

We noted the three checkout columns earlier. Now that we understand the full model, it is clear why they exist: when the claim loop runs, potentially dozens of agents may be looking for tasks to pick up. Without a locking mechanism, two agents could both read a `todo` task and both decide to claim it. We need **atomic checkout** — a single database operation that either succeeds (one agent wins the task) or fails cleanly (another agent already won it).

The `checkoutRunId` and `executionLockedAt` columns carry the lock. The next chapter builds the claim loop that uses them: [The Task Queue and Worker Claim Loop](./task-queue-and-claim-loop.md).

## The `originFingerprint` for deduplication

One more field worth understanding before we move on: `originFingerprint`. When a task is created by a recurring schedule (a "routine" or "autopilot"), the orchestrator needs to ensure that if the routine fires while a task from the previous firing is still active, it does not create a duplicate.

The partial unique index in the schema handles this:

```sql
-- Simplified from the Drizzle partial index definition
CREATE UNIQUE INDEX tasks_open_routine_execution_uq
  ON tasks (workspace_id, origin_kind, origin_id, origin_fingerprint)
  WHERE origin_kind = 'routine_execution'
    AND origin_id IS NOT NULL
    AND hidden_at IS NULL
    AND execution_run_id IS NOT NULL
    AND status IN ('backlog', 'todo', 'in_progress', 'in_review', 'blocked');
```

The `originFingerprint` encodes the identity of the specific invocation. Combined with `originKind` and `originId`, it creates a unique key that the database itself enforces — an attempt to insert a duplicate active task with the same fingerprint will fail at the constraint level, not at the application level.

## Field reference summary

The full field set of `tasks`, organized by concern:

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | Primary key |
| `workspaceId` | uuid FK | Multi-tenant boundary; every query filters by this |
| `projectId` | uuid FK, nullable | Optional project linkage |
| `goalId` | uuid FK, nullable | Optional goal linkage |
| `parentId` | uuid FK (self), nullable | Sub-task parent reference |
| `title` | text, not null | Short summary |
| `description` | text, nullable | Full description, supports Markdown |
| `status` | text, default `"backlog"` | One of the seven states above |
| `priority` | text, default `"medium"` | `critical \| high \| medium \| low` |
| `assigneeAgentId` | uuid FK, nullable | Agent owner; mutually exclusive with `assigneeUserId` |
| `assigneeUserId` | text, nullable | Human owner; mutually exclusive with `assigneeAgentId` |
| `checkoutRunId` | uuid FK, nullable | Ownership lock — which run holds the claim |
| `executionRunId` | uuid FK, nullable | Active execution pointer — which run is live now |
| `executionLockedAt` | timestamptz, nullable | When the execution lock was acquired |
| `originKind` | text, default `"manual"` | How the task was created |
| `originId` | text, nullable | External or routine identifier |
| `originRunId` | text, nullable | Which run created this task (for routine tasks) |
| `originFingerprint` | text, default `"default"` | Dedup key within the origin |
| `taskNumber` | integer, nullable | Auto-incrementing number per company |
| `identifier` | text, nullable | Human-readable ID, e.g. `ENG-42`; unique index |
| `requestDepth` | integer, default 0 | Delegation nesting depth |
| `startedAt` | timestamptz, nullable | Set automatically on first `in_progress` transition |
| `completedAt` | timestamptz, nullable | Set automatically on `done` transition |
| `cancelledAt` | timestamptz, nullable | Set automatically on `cancelled` transition |
| `hiddenAt` | timestamptz, nullable | Soft-hide for recovered/archived tasks |
| `createdAt` | timestamptz, not null | Creation time |
| `updatedAt` | timestamptz, not null | Last modification time |

## Summary

We now have a precise model for a task in Swarm:

- The **schema** captures identity, context, ownership, execution locking, provenance, and lifecycle timestamps.
- The **single-assignee invariant** ensures clear ownership — at most one agent or human per task.
- The **seven-state machine** carries precise execution semantics: `backlog` and `todo` are non-execution states; `in_progress` is a strict execution-backed state for agents; `blocked` and `in_review` are healthy waiting states with explicit paths forward; `done` and `cancelled` are terminal.
- **`parentId`** expresses structure (work breakdown), while relation types like `blocks` / `blocked_by` express dependency. They are different tools for different jobs.
- The **atomic-checkout columns** (`checkoutRunId`, `executionRunId`, `executionLockedAt`) are what make conflict-safe claiming possible.
- **`originFingerprint`** combined with a partial unique index prevents duplicate active tasks from recurring sources.

With this foundation in place, we are ready to build the claim loop that puts these columns to work.

---

← Previous: [A Run: Sessions, Usage, and Cost](../the-agent/a-run.md) · Next: [The Task Queue and Worker Claim Loop](./task-queue-and-claim-loop.md) →
