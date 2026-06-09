---
title: "The Schedule Data Model"
description: "Design the four-table schedule schema — immutable revisions, cron + timezone triggers, and the concurrency and catch-up policies that govern missed ticks."
category: scheduling
type: explanation
tags:
  - schedule
  - routine
  - cron
  - cron expression
  - timezone
  - revision
  - concurrency policy
  - coalesce_if_active
  - catch-up policy
  - skip_missed
  - enqueue_missed_with_cap
  - MAX_CATCH_UP_RUNS
  - variables
  - dispatchFingerprint
  - coalescedIntoRunId
  - trigger
  - nextRunAt
  - signingMode
  - replayWindowSec
  - Drizzle schema
  - PostgreSQL
  - immutable snapshots
  - audit trail
  - idempotency
  - webhook trigger
  - manual trigger
  - schedule trigger
  - skip_if_active
  - always_enqueue
keywords:
  - recurring agent work
  - scheduled tasks
  - autopilot
  - cron job agent
  - timezone-aware scheduling
  - catch-up runs
  - missed ticks
  - deduplication fingerprint
  - coalesce runs
  - schedule revisions
  - tick-safe dispatch
sources: [S39, S31, S14]
---

**TL;DR** — Swarm represents recurring agent work as a *schedule* (what some systems call a routine or autopilot). The schema spans four tables: a mutable `schedules` header, immutable `revisions` that capture every configuration change, `triggers` that hold the cron expression and timezone, and `runs` that record every firing. Two per-schedule policies — *concurrency* and *catch-up* — determine what happens when a previous run is still in flight or when ticks were missed during downtime.

# The Schedule Data Model

Some agent work needs no human prompt. A daily standup summary, a weekly cost report, a nightly code audit — these should happen on a timer, reliably, even while the orchestrator was briefly offline. To support that, Swarm introduces the *schedule* entity.

This chapter explains the four-table schema behind schedules, the rationale for each design decision, and the two policies you configure per schedule to control parallel execution and missed-tick recovery.

We will refer back to two related concepts introduced earlier:

- **Tasks** (see [Modeling Tasks](../tasks-and-queue/modeling-tasks.md)) — a schedule fires by creating a task; understanding the task lifecycle helps make sense of what `dispatchFingerprint` and `coalescedIntoRunId` protect.
- **Drizzle schema patterns** (see [Prerequisites and Project Setup](../getting-started/project-setup.md)) — the schemas below use the same `pgTable` / `uuid` / `timestamp` primitives introduced there.

---

## Why four tables instead of one

The first question is: why not store everything in a single `schedules` table?

The answer is mutability. A schedule can change: its cron expression can be updated, its assignee can change, its variables can be edited. If a run happened *before* that change, we want to know exactly what configuration produced it. We also want to roll back to any past state.

That constraint pulls us toward two tables — a mutable header and immutable revision snapshots — and then the trigger and run concerns naturally live in their own tables for query efficiency and separate lifecycles.

Let's build this up one table at a time.

---

## Table 1 — `schedules`

The `schedules` table is the mutable header: it holds the current live configuration and the pointers into the other tables.

```ts
// Simplified view of the schedules table (Drizzle, PostgreSQL)
export const schedules = pgTable(
  "schedules",
  {
    id: uuid("id").primaryKey().defaultRandom(),

    // Multi-tenant scoping — every schedule belongs to one workspace
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),

    // Optional project + goal context
    projectId: uuid("project_id").references(() => projects.id, { onDelete: "cascade" }),
    goalId: uuid("goal_id").references(() => goals.id, { onDelete: "set null" }),
    parentTaskId: uuid("parent_task_id").references(() => tasks.id, { onDelete: "set null" }),

    // What the schedule produces
    title: text("title").notNull(),
    description: text("description"),
    assigneeAgentId: uuid("assignee_agent_id").references(() => agents.id),
    priority: text("priority").notNull().default("medium"),
    status: text("status").notNull().default("active"),

    // The two governance policies (explained in detail below)
    concurrencyPolicy: text("concurrency_policy").notNull().default("coalesce_if_active"),
    catchUpPolicy: text("catch_up_policy").notNull().default("skip_missed"),

    // Typed variables for template interpolation
    variables: jsonb("variables").$type<ScheduleVariable[]>().notNull().default([]),
    env: jsonb("env").$type<ScheduleEnvConfig>(),

    // Pointers to the latest revision — updated on every config change
    latestRevisionId: uuid("latest_revision_id"),
    latestRevisionNumber: integer("latest_revision_number").notNull().default(1),

    // Audit timestamps
    lastTriggeredAt: timestamp("last_triggered_at", { withTimezone: true }),
    lastEnqueuedAt: timestamp("last_enqueued_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    workspaceStatusIdx: index("schedules_workspace_status_idx").on(table.workspaceId, table.status),
    workspaceAssigneeIdx: index("schedules_workspace_assignee_idx").on(table.workspaceId, table.assigneeAgentId),
    workspaceProjectIdx: index("schedules_workspace_project_idx").on(table.workspaceId, table.projectId),
  }),
);
```

A few details worth noting:

- `status` starts at `"active"` but will fall back to `"paused"` if no assignee agent is set at creation time — a schedule with no one to run it cannot be active.
- `latestRevisionId` + `latestRevisionNumber` are pointer columns updated whenever a revision is appended. They let the scheduler look up the current snapshot in O(1) without walking revision history.
- `lastTriggeredAt` / `lastEnqueuedAt` are convenience timestamps: the UI reads `lastTriggeredAt` to show "last fired X minutes ago" and `lastEnqueuedAt` to show the last time a task was actually created (distinct from a coalesced or skipped tick).

---

## Table 2 — `schedule_revisions` (immutable snapshots)

Every time the schedule's configuration changes — any field update, any trigger added or removed — the orchestrator appends a new row to `schedule_revisions`. Existing rows are **never mutated**. This gives us:

1. A full audit trail of who changed what and when.
2. The ability to reproduce exactly what configuration a past run used.
3. The ability to roll back to any earlier configuration.

```ts
// Simplified view of the schedule_revisions table
export const scheduleRevisions = pgTable(
  "schedule_revisions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
    scheduleId: uuid("schedule_id").notNull().references(() => schedules.id, { onDelete: "cascade" }),

    // Monotonically increasing within a schedule; the unique index enforces this
    revisionNumber: integer("revision_number").notNull(),

    // Human-readable title + description from this revision's snapshot
    title: text("title").notNull(),
    description: text("description"),

    // The full immutable snapshot: schedule fields + all triggers at this point in time
    snapshot: jsonb("snapshot").$type<ScheduleRevisionSnapshotV1>().notNull(),
    changeSummary: text("change_summary"),

    // Self-referential for restore operations: "restored from revision #3"
    restoredFromRevisionId: uuid("restored_from_revision_id").references(
      (): AnyPgColumn => scheduleRevisions.id,
      { onDelete: "set null" },
    ),

    // Who created this revision
    createdByAgentId: uuid("created_by_agent_id").references(() => agents.id, { onDelete: "set null" }),
    createdByUserId: text("created_by_user_id"),
    createdByRunId: uuid("created_by_run_id").references(() => scheduleRuns.id, { onDelete: "set null" }),

    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    // A schedule can have at most one row per revision number
    revisionUq: uniqueIndex("schedule_revisions_revision_uq").on(table.scheduleId, table.revisionNumber),
    workspaceScheduleCreatedIdx: index("schedule_revisions_created_idx").on(
      table.workspaceId,
      table.scheduleId,
      table.createdAt,
    ),
  }),
);
```

The `snapshot` column deserves special attention. It is a JSONB blob typed as `ScheduleRevisionSnapshotV1` and contains a complete snapshot of the schedule's fields *and* all its triggers at the moment the revision was created. When the scheduler loop fires a tick, it reads `schedules.latestRevisionId`, fetches that one revision row, and has everything it needs — no join necessary, and no risk of reading a half-updated state.

The `version: 1` field inside the snapshot allows forward-compatible schema evolution: if we need to add a new field, we can introduce `ScheduleRevisionSnapshotV2` and migrate.

The `restoredFromRevisionId` self-reference documents restore operations: if a user rolls back to revision 3 and that creates revision 7, revision 7's `restoredFromRevisionId` points at revision 3.

---

## Table 3 — `schedule_triggers`

The `schedule_triggers` table holds the *activation conditions* for a schedule. A single schedule can have multiple triggers — for example, a cron trigger that fires Monday mornings plus a webhook trigger so an external CI system can also kick it off.

```ts
// Simplified view of the schedule_triggers table
export const scheduleTriggers = pgTable(
  "schedule_triggers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
    scheduleId: uuid("schedule_id").notNull().references(() => schedules.id, { onDelete: "cascade" }),

    // Discriminator: "schedule" | "webhook" | "manual"
    kind: text("kind").notNull(),
    label: text("label"),
    enabled: boolean("enabled").notNull().default(true),

    // Cron-trigger fields (null on webhook/manual triggers)
    cronExpression: text("cron_expression"),
    timezone: text("timezone"),
    nextRunAt: timestamp("next_run_at", { withTimezone: true }),
    lastFiredAt: timestamp("last_fired_at", { withTimezone: true }),

    // Webhook-trigger fields
    publicId: text("public_id"),          // stable URL token
    secretId: uuid("secret_id").references(() => workspaceSecrets.id, { onDelete: "set null" }),
    signingMode: text("signing_mode"),    // "none" | "bearer" | "github_hmac" | default HMAC
    replayWindowSec: integer("replay_window_sec"),
    lastRotatedAt: timestamp("last_rotated_at", { withTimezone: true }),

    lastResult: text("last_result"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    workspaceScheduleIdx: index("schedule_triggers_workspace_schedule_idx").on(
      table.workspaceId,
      table.scheduleId,
    ),
    // The scheduler loop queries "WHERE kind='schedule' AND nextRunAt <= NOW()"
    nextRunIdx: index("schedule_triggers_next_run_idx").on(table.nextRunAt),
    publicIdUq: uniqueIndex("schedule_triggers_public_id_uq").on(table.publicId),
  }),
);
```

### Trigger kinds

| `kind` | What fires it | Key fields |
|--------|--------------|------------|
| `schedule` | The scheduler loop, on a cron tick | `cronExpression`, `timezone`, `nextRunAt` |
| `webhook` | An inbound HTTP POST to the trigger's public URL | `publicId`, `secretId`, `signingMode`, `replayWindowSec` |
| `manual` | A user or agent explicitly calling the run API | none (no scheduling state) |

### Cron + timezone + `nextRunAt`

For a `schedule`-kind trigger, three fields work together:

- **`cronExpression`** — a five-field cron string (`"0 9 * * 1"` for "every Monday at 09:00"). The orchestrator validates it at create/update time and rejects invalid expressions before they reach the database.
- **`timezone`** — an IANA timezone string (e.g. `"America/New_York"`). The orchestrator validates it using `Intl.DateTimeFormat` and throws if the string is unrecognised. All cron matching happens in this timezone, so "9am Monday" means 9am in the configured zone regardless of the server's local time.
- **`nextRunAt`** — a pre-computed UTC timestamp of the next tick. The scheduler loop selects `WHERE nextRunAt <= NOW()` to find due triggers; it does not parse cron expressions on every query. After claiming a tick, it computes the next `nextRunAt` and writes it back.

You might wonder: why store `nextRunAt` at all instead of computing it on every scheduler pass? Two reasons. First, the scheduler can use a single indexed `lte` query — no per-row expression evaluation. Second, the optimistic-update pattern used for tick claiming (compare-and-swap on `nextRunAt`) prevents double-dispatch across concurrent scheduler instances.

### Webhook signing modes

A webhook trigger generates a per-trigger secret stored in `workspaceSecrets`. When an inbound request arrives, the orchestrator verifies it using the `signingMode`:

| `signingMode` | Verification method |
|---|---|
| `none` | The `publicId` in the URL acts as the shared secret; no body signing |
| `bearer` | `Authorization: Bearer <secret>` header, constant-time compare |
| `github_hmac` | HMAC-SHA-256 over the raw body; accepts `X-Hub-Signature-256` (GitHub/Sentry convention) |
| *(default)* | Timestamp + HMAC-SHA-256: `sha256(timestamp + "." + rawBody)`, with a replay window |

The `replayWindowSec` field (default 300 seconds when null) defines how wide the timestamp tolerance is for the default HMAC mode, protecting against replay attacks.

---

## Table 4 — `schedule_runs`

Every firing of a schedule — whether it created a task, was coalesced into an existing run, or was skipped — produces a row in `schedule_runs`. This is the execution ledger.

```ts
// Simplified view of the schedule_runs table
export const scheduleRuns = pgTable(
  "schedule_runs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
    scheduleId: uuid("schedule_id").notNull().references(() => schedules.id, { onDelete: "cascade" }),
    triggerId: uuid("trigger_id").references(() => scheduleTriggers.id, { onDelete: "set null" }),

    // Where the firing came from: "schedule" | "webhook" | "manual" | "api"
    source: text("source").notNull(),

    // Lifecycle: "received" → "task_created" | "coalesced" | "skipped" | "failed" | "completed"
    status: text("status").notNull().default("received"),

    triggeredAt: timestamp("triggered_at", { withTimezone: true }).notNull().defaultNow(),

    // Which revision was active when this run was dispatched
    scheduleRevisionId: uuid("schedule_revision_id").references(
      () => scheduleRevisions.id,
      { onDelete: "set null" },
    ),

    idempotencyKey: text("idempotency_key"),
    triggerPayload: jsonb("trigger_payload").$type<Record<string, unknown>>(),

    // Deduplication — see below
    dispatchFingerprint: text("dispatch_fingerprint"),
    coalescedIntoRunId: uuid("coalesced_into_run_id"),

    linkedTaskId: uuid("linked_task_id").references(() => tasks.id, { onDelete: "set null" }),
    failureReason: text("failure_reason"),
    completedAt: timestamp("completed_at", { withTimezone: true }),

    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    workspaceScheduleIdx: index("schedule_runs_workspace_schedule_idx").on(
      table.workspaceId,
      table.scheduleId,
      table.createdAt,
    ),
    dispatchFingerprintIdx: index("schedule_runs_dispatch_fingerprint_idx").on(
      table.scheduleId,
      table.dispatchFingerprint,
    ),
    idempotencyIdx: index("schedule_runs_idempotency_idx").on(table.triggerId, table.idempotencyKey),
  }),
);
```

The `scheduleRevisionId` column links each run to the exact configuration snapshot that produced it. If someone changes the cron expression next week, old runs still point at the revision that was active when they fired.

---

## Concurrency policy — what to do when the previous run is still active

Imagine your schedule fires every five minutes, but one execution takes twelve minutes to complete. When the next tick arrives, the previous task is still in progress. What should happen?

The `concurrencyPolicy` field on `schedules` decides. The three values are:

| `concurrencyPolicy` | Behaviour |
|---|---|
| `coalesce_if_active` | If a live task exists, mark this run as `"coalesced"` and point `coalescedIntoRunId` at the original run. No new task is created. This is the **default**. |
| `skip_if_active` | If a live task exists, mark this run as `"skipped"`. No new task is created, and the run is not linked to the existing one. |
| `always_enqueue` | Always create a new task, regardless of whether one is already running. |

The `coalesce_if_active` default is deliberately conservative. For most recurring work — daily reports, periodic audits — it makes no sense to run two simultaneous instances. The newer firing is recorded (so the history is honest) but does not create duplicate work.

When a run is coalesced, the `schedule_runs` row records it: `status = "coalesced"` and `coalescedIntoRunId` holds the ID of the run that was already live. If a human triggered the run manually, the orchestrator additionally marks the existing task as "seen" in the user's inbox so they notice it is running.

```
schedule_runs row (coalesced case):
  id:                  <new run id>
  status:              "coalesced"
  coalescedIntoRunId:  <id of the run that was already in flight>
  linkedTaskId:        <id of the already-running task>
  completedAt:         <now — immediately finalized>
```

The `skip_if_active` value is for cases where you want a clean "did not run" record without linking to the existing run — for example, a heartbeat-style probe where the in-flight version is already handling it and you do not want the new tick associated with it.

`always_enqueue` is for inherently parallel work — a schedule that generates independent batch items — where concurrent executions are expected and desired.

---

## Catch-up policy — what to do about missed ticks

Now consider downtime. The orchestrator restarts after six hours of maintenance. During that window, a schedule that runs every hour would have missed six ticks. Should those six runs fire now?

The `catchUpPolicy` field governs this. The two values are:

| `catchUpPolicy` | Behaviour |
|---|---|
| `skip_missed` | Compute the *next* future tick from now and advance `nextRunAt` forward. No catch-up runs fire. This is the **default**. |
| `enqueue_missed_with_cap` | Replay every tick that was missed, up to `MAX_CATCH_UP_RUNS` (25) total. The cap prevents a long outage from flooding the queue. |

The `skip_missed` default exists for a good reason: most recurring work is time-relative. If you missed the Tuesday morning report, re-running it on Tuesday afternoon using stale data produces a confusing or wrong result. Skipping is usually the right answer.

The `enqueue_missed_with_cap` value is for genuinely event-like work where each firing is independent and still meaningful after a delay — for example, syncing a feed that accumulates items. The cap `MAX_CATCH_UP_RUNS = 25` is a tunable constant in the scheduler service; it exists to prevent an orchestrator that was offline for a week from overwhelming the task queue when it comes back.

Here is how the scheduler loop reads these two values together (simplified from the `tickScheduledTriggers` function in the scheduler service):

```ts
// Simplified view of the tick-claiming logic inside tickScheduledTriggers()
let runCount = 1;
let claimedNextRunAt = nextCronTickInTimeZone(expression, timezone, now);

if (catchUpPolicy === "enqueue_missed_with_cap") {
  let cursor: Date | null = trigger.nextRunAt;
  runCount = 0;
  while (cursor && cursor <= now && runCount < MAX_CATCH_UP_RUNS) {
    runCount += 1;
    claimedNextRunAt = nextCronTickInTimeZone(expression, timezone, cursor);
    cursor = claimedNextRunAt;
  }
}

// Claim the tick atomically before dispatching — compare-and-swap on nextRunAt
const claimed = await db.update(scheduleTriggers)
  .set({ nextRunAt: claimedNextRunAt, updatedAt: new Date() })
  .where(and(
    eq(scheduleTriggers.id, trigger.id),
    eq(scheduleTriggers.nextRunAt, trigger.nextRunAt), // optimistic lock
  ))
  .returning({ id: scheduleTriggers.id })
  .then(rows => rows[0] ?? null);

if (!claimed) continue; // Another scheduler instance got there first

// Fire runCount times (1 for skip_missed, up to 25 for enqueue_missed_with_cap)
for (let i = 0; i < runCount; i++) {
  await dispatchScheduleRun({ schedule, trigger, source: "schedule" });
}
```

Notice the optimistic lock: the `WHERE nextRunAt = <original value>` condition means the update only succeeds if no other instance already claimed this tick. If two scheduler instances are running simultaneously, only one gets `claimed !== null` and proceeds. The other simply skips this row. This makes tick dispatch safe across restarts or horizontal scale-out without requiring distributed locks.

---

## Deduplication — `dispatchFingerprint` and `coalescedIntoRunId`

The final concern is: what if a tick is dispatched, a task is created, but then the scheduler crashes before writing the run status? On restart, it would try to fire again. Without protection, you would get duplicate tasks.

Swarm prevents this through **dispatch fingerprinting**. Before creating a task, the orchestrator computes a SHA-256 hash over the inputs that uniquely identify this dispatch:

```ts
// Inputs to the dispatch fingerprint (simplified)
const fingerprint = sha256(JSON.stringify({
  payload:                     triggerPayload,          // resolved variables + trigger body
  projectId,
  assigneeAgentId,
  scheduleRevisionId:          schedule.latestRevisionId,
  scheduleEnvFingerprint:      sha256(env),             // hash of env config
  executionWorkspaceId,
  title,                                                // after variable interpolation
  description,
}));
```

This fingerprint is stored on both the `schedule_runs` row (`dispatchFingerprint`) and — once the task is created — on the task itself (`originFingerprint`). Before creating a new task, the orchestrator looks for a live task with a matching fingerprint:

```ts
// If a live task with this fingerprint already exists, coalesce instead of creating a duplicate
const existing = await findLiveExecutionTask(schedule, fingerprint);
if (existing && concurrencyPolicy !== "always_enqueue") {
  await finalizeRun(run.id, {
    status: "coalesced",
    linkedTaskId: existing.id,
    coalescedIntoRunId: existing.originRunId,
    completedAt: now,
  });
  return;
}
```

The `coalescedIntoRunId` on the `schedule_runs` row then points at the run that originally created the still-live task. This means the run history is consistent: you can always trace "this tick did not create a new task because that run was already handling the same work."

The fingerprint is deterministic — the same inputs produce the same hash — so it also handles the case where the server restarts mid-dispatch: the second attempt computes the same fingerprint, finds the task it created in the first attempt, and coalesces rather than duplicating.

---

## Variables — parameterising schedule execution

A schedule can declare typed `variables` that are interpolated into the task `title` and `description` before dispatch. For example:

```ts
// A schedule variable definition (typed as ScheduleVariable)
{
  name: "report_period",
  label: "Report Period",
  type: "select",
  options: ["daily", "weekly", "monthly"],
  defaultValue: "weekly",
  required: true,
}
```

The `type` field can be `"text"`, `"number"`, `"boolean"`, or `"select"`. Required variables that have no default value cannot be used on cron triggers — the orchestrator rejects this at creation time because the scheduler loop has no way to supply the value.

Variable values are resolved at dispatch time in this priority order:

1. **Automatic variables** — values the orchestrator derives from execution context (e.g. the current workspace branch). These cannot be overridden by callers.
2. **Provided values** — variables passed in the webhook payload or the manual-run API call.
3. **Default values** — the `defaultValue` from the variable definition.

The resolved variable map is merged into the trigger payload and stored on the `schedule_runs` row, so the exact values used for every firing are preserved in the run history.

---

## The Go model — how another implementation expresses the same concepts

The Go-based reference implementation (S14) expresses the same data in its `AutopilotTrigger` struct:

```go
// Translated from the AutopilotTrigger struct in S14
type ScheduleTrigger struct {
    ID             string    `json:"id"`
    ScheduleID     string    `json:"schedule_id"`
    Kind           string    `json:"kind"`           // "schedule" | "webhook"
    Enabled        bool      `json:"enabled"`
    CronExpression *string   `json:"cron_expression"` // nil for non-cron triggers
    Timezone       *string   `json:"timezone"`
    NextRunAt      time.Time `json:"next_run_at"`
    WebhookToken   *string   `json:"webhook_token"`   // equivalent to publicId
    SigningSecret  *string   `json:"signing_secret"`
    EventFilters   []byte    `json:"event_filters"`   // JSON-encoded filter rules
    LastFiredAt    time.Time `json:"last_fired_at"`
    Provider       string    `json:"provider"`
}
```

The field mapping is close: `CronExpression`/`Timezone`/`NextRunAt` correspond directly. The Go version uses a simpler `WebhookToken` (a raw string) whereas the TypeScript version stores a `publicId` plus a separate secret in `workspaceSecrets`. The Go version carries `EventFilters` (a JSON blob for provider-specific event routing) and a `Provider` discriminator — the TypeScript version encodes the equivalent information in `signingMode` and the trigger kind.

Both approaches reach the same logical model: a trigger has a kind, optionally a cron schedule in a named timezone, and optionally a webhook token with signing configuration.

---

## Schema summary

| Table | Mutable? | Purpose |
|---|---|---|
| `schedules` | Yes | Mutable header; holds current policies, assignee, variables, pointer to latest revision |
| `schedule_revisions` | No (append-only) | Immutable snapshot of every configuration state; used for audit trail and rollback |
| `schedule_triggers` | Yes | Activation conditions — cron (with `nextRunAt`), webhook, manual |
| `schedule_runs` | Append + terminal updates | Execution ledger; every firing recorded with status, fingerprint, linked task |

| Policy | Default | Other values | Question it answers |
|---|---|---|---|
| `concurrencyPolicy` | `coalesce_if_active` | `skip_if_active`, `always_enqueue` | What to do if the previous run is still active |
| `catchUpPolicy` | `skip_missed` | `enqueue_missed_with_cap` | What to do about ticks missed during downtime |

---

## What comes next

We now have the full schema. What we have not yet seen is the *loop* that reads it on a timer, claims due triggers, applies both policies, and dispatches runs. That is the subject of the next chapter.

← Previous: [WebSockets II — The Live Board](../real-time/live-board.md) · Next: [The Scheduler Loop](./scheduler-loop.md) →
