---
title: "A Run: Sessions, Usage, and Cost"
description: "What the run entity records for every agent execution window — fields, lifecycle, session IDs that enable resumption, and how usage and cost flow from the adapter result into the database."
category: the-agent
type: explanation
tags: [run, session, heartbeat run, invocationSource, retry chain, usage summary, cost, costUsd, sessionId, log storage, resume session, liveness tracking, contextSnapshot, run log, status, livenessState, retryOfRunId, scheduledRetryAt, logStore, logRef, usageJson, resultJson, billingType, provider, model, AdapterResult, UsageSummary, SwarmAdapter, invoke]
keywords: [execution window, run record, run lifecycle, run cost event, drizzle schema, token tracking, session resumption, session before after, run liveness, continuation attempt, process tracking, log compression, cost ledger]
sources: [S24, S30, S36]
---

**TL;DR** — Every time an agent works, the orchestrator writes a **run** — a durable record of one execution window. This chapter explains what a run captures (its schema fields, lifecycle statuses, and session identifiers), how token usage and cost flow from the adapter's return value into the run record and the cost ledger, and how a session ID stored on the run lets a later execution resume exactly where the previous one left off.

# A Run: Sessions, Usage, and Cost

When an agent does something — answers a task, wakes up on a schedule, continues an interrupted session — the orchestrator creates one **run** record. That record answers three questions at any point in time:

1. What is happening right now (status, live output position)?
2. What did it cost and which tokens were consumed?
3. Where should the next execution pick up the conversation?

Let's walk through each of those questions, building up the full picture of the run entity along the way.

---

## What triggers a run, and what gets recorded immediately

### The invocation source

Before an adapter ever runs, the orchestrator inserts a row into the `runs` table (the schema uses the table name `heartbeat_runs` — you may encounter that term in the codebase). The very first field to understand is `invocationSource`:

```ts
// Simplified view of the run insert at enqueue time
await db.insert(heartbeatRuns).values({
  workspaceId: agent.workspaceId,
  agentId,
  invocationSource: source,   // e.g. "on_demand" | "automation"
  triggerDetail,              // e.g. "manual" | "system"
  status: "queued",
  wakeupRequestId: wakeupRequest.id,
  contextSnapshot: enrichedContextSnapshot,
  sessionIdBefore: sessionBefore,
  continuationAttempt,
});
```

`invocationSource` records *why* the run was created. The schema defines its default as `"on_demand"` (S36); the service sets it to `"automation"` when the orchestrator fires the run programmatically (for example, on a scheduled trigger or as part of a retry chain). `triggerDetail` adds a finer label: `"manual"` when a human requested it, `"system"` when the orchestrator itself decided.

`continuationAttempt` starts at 0 for a fresh run and increments each time a completed run spawns a follow-up within the same task scope (for example, when a max-turn exhaustion triggers an automatic continuation).

### The context snapshot

Also written at insert time is `contextSnapshot` — a JSON blob that carries the task context under which this run was queued:

```ts
// Keys stored inside contextSnapshot (drawn from heartbeat.ts query aliases)
{
  taskId: string | null,      // which task this run is working on
  taskId: string | null,       // alias for taskId
  taskKey: string | null,      // human-readable key, e.g. "PROJ-42"
  commentId: string | null,    // the triggering comment, if any
  wakeCommentId: string | null,
  wakeReason: string | null,   // e.g. "task_blockers_resolved"
  wakeSource: string | null,
}
```

The context snapshot is written once at enqueue and may be updated mid-run (for example, if the adapter discovers runtime services). It is the run's "why am I here?" memo. The orchestrator reads it back during execution to derive the task key, resolve the associated task, and decide whether to post result comments.

---

## The run's lifecycle: from queued to terminal

Once enqueued, a run moves through a series of statuses. The schema defines the field as `status text not null default 'queued'`, and the service tracks these statuses (S30):

| Status | Meaning |
|---|---|
| `queued` | Waiting for the runner to claim it |
| `running` | The adapter is actively executing |
| `scheduled_retry` | Waiting for a deferred retry (e.g. transient upstream error) |
| `succeeded` | The adapter finished with exit code 0 and no error |
| `failed` | The adapter reported an error or exited non-zero |
| `timed_out` | The run exceeded its allowed time window |
| `cancelled` | Cancelled before or during execution |

The first three are "live" statuses — the run is still in motion. The last four are terminal. Once in a terminal status, the run is never modified for execution purposes, though the liveness classifier may update `livenessState` and `livenessReason` afterward.

### Process tracking fields

When the adapter spawns a child process, the orchestrator records that process's identity on the run row. This lets the runner detect a "detached" process — one whose in-memory handle was lost but whose OS process is still alive:

```ts
// Written via onSpawn callback when the adapter starts a child process
{
  processPid: number,
  processGroupId: number | null,
  processStartedAt: timestamp,
}
```

These fields (from S36) are only meaningful while `status = "running"`. After the run finishes they become historical markers used for post-mortem analysis.

---

## Log storage: capturing what the agent produced

Every byte of agent output needs a home. The orchestrator opens a log handle at the start of execution and writes chunks to it as they arrive. Once the run finishes, the log is finalized and its location recorded on the run row (S30):

```ts
// Written at log-open time (run start)
{
  logStore: handle.store,   // backend type, e.g. "local_file"
  logRef: handle.logRef,    // opaque reference to the log object
}

// Written at log-finalize time (run end)
{
  logBytes: number,         // total bytes stored
  logSha256: string | null, // SHA-256 integrity hash
  logCompressed: boolean,   // whether the stored data is compressed (default false)
}
```

`logStore` and `logRef` together let the orchestrator re-open the log for replay on demand. The `logSha256` field uses standard SHA-256 so the server can verify log integrity before serving it.

The run also keeps inline excerpts for fast UI display without fetching the full log:

```ts
{
  stdoutExcerpt: string | null,
  stderrExcerpt: string | null,
}
```

During execution, the orchestrator also tracks the live output position — the last byte offset and timestamp the runner has advanced to — so that watchdog and liveness logic can tell whether the agent is still producing output:

```ts
{
  lastOutputAt: timestamp,
  lastOutputSeq: integer,   // default 0
  lastOutputStream: text,   // "stdout" | "stderr"
  lastOutputBytes: bigint,
}
```

---

## Usage, cost, and the adapter result

Now we get to what makes a run financially meaningful. The adapter's result — the object it returns after execution — carries both token usage and monetary cost. Let's see where those values come from before following them into the database.

### A quick recap of `AdapterResult` and `UsageSummary`

The **Adapter Interface** (see [The Adapter Interface](./adapter-interface.md)) defines `AdapterResult` as the return type of every adapter's `invoke` method. The full definition lives in P4; the fields that matter for cost and usage tracking are (S24):

| Field | Type | Notes |
|---|---|---|
| `exitCode` | `number \| null` | Process exit code; `null` for non-process adapters |
| `timedOut` | `boolean` | `true` if the run exceeded its time limit |
| `usage` | `UsageSummary?` | Token counts, when reported by the adapter |
| `costUsd` | `number \| null \|` optional | Total cost for this run in US dollars |
| `provider` | `string \| null \|` optional | e.g. `"anthropic"`, `"openai"` |
| `model` | `string \| null \|` optional | e.g. `"claude-opus-4-5"` |
| `sessionId` | `string \| null \|` optional | Legacy single-session identifier |
| `sessionParams` | `Record<string, unknown> \| null \|` optional | Structured session state |
| `sessionDisplayId` | `string \| null \|` optional | Human-readable session label |

`UsageSummary` carries three non-negative integer counts: `inputTokens`, `outputTokens`, and optionally `cachedInputTokens` (tokens served from the provider's prompt cache).

The **LLM adapter** (see [The LLM Adapter](./llm-adapter.md)) is the primary adapter that produces real values for `usage` and `costUsd` — it parses them from the LLM provider's response and returns them in the `AdapterResult`.

### Building `usageJson` from the adapter result

After the adapter returns, the orchestrator assembles a `usageJson` blob to persist on the run row (S30):

```ts
// Simplified view of how usageJson is composed (heartbeat.ts)
const rawUsage = normalizeUsageTotals(adapterResult.usage);
// normalizeUsageTotals clamps floats to non-negative integers

const usageJson =
  normalizedUsage || adapterResult.costUsd != null
    ? {
        ...normalizedUsage,        // inputTokens, outputTokens, cachedInputTokens
        rawInputTokens: rawUsage?.inputTokens,
        rawCachedInputTokens: rawUsage?.cachedInputTokens,
        rawOutputTokens: rawUsage?.outputTokens,
        provider: adapterResult.provider ?? "unknown",
        biller: resolveLedgerBiller(adapterResult),
        model: adapterResult.model ?? "unknown",
        ...(adapterResult.costUsd != null ? { costUsd: adapterResult.costUsd } : {}),
        billingType: normalizeLedgerBillingType(adapterResult.billingType),
        sessionReused: /* whether a session was resumed */,
        freshSession: /* whether this was a new session */,
        // ...
      }
    : null;
```

Notice that the orchestrator stores both `rawInputTokens` and `inputTokens`. The raw counts are what the adapter reported directly; the normalized counts may have been adjusted (for example, when usage is derived from session-level totals rather than a single-run report). The field `usageSource: "session_delta"` is set in `usageJson` when the orchestrator computes usage by subtracting session cumulative totals rather than using a per-run figure reported directly by the adapter (S30).

The full assembled `usageJson` is persisted on the run record and can be queried via Drizzle's JSON path operators (S30 shows the orchestrator extracting `costUsd` from `resultJson ->> 'costUsd'`).

### Writing the cost event

Usage and cost are recorded in two places: in-line on the run row (via `usageJson`) and as a separate **cost event** row in the cost ledger. The orchestrator calls `costService` after every run finishes (S30):

```ts
// Simplified view of updateRuntimeState in heartbeat.ts
const billingType = normalizeLedgerBillingType(result.billingType);
const additionalCostCents = normalizeBilledCostCents(result.costUsd, billingType);
// normalizeBilledCostCents returns 0 when billingType is "subscription_included"
// or when costUsd is not a finite number

if (additionalCostCents > 0 || hasTokenUsage) {
  await costs.createEvent(agent.workspaceId, {
    heartbeatRunId: run.id,
    agentId: agent.id,
    taskId: ledgerScope.taskId,   // resolved from contextSnapshot
    projectId: ledgerScope.projectId,
    provider,
    biller,
    billingType,
    model: result.model ?? "unknown",
    inputTokens,
    cachedInputTokens,
    outputTokens,
    costCents: additionalCostCents,
    occurredAt: new Date(),
  });
}
```

Two things to notice here:

1. `normalizeBilledCostCents` converts the USD float to integer cents, clamped at zero — and returns `0` when `billingType` is `"subscription_included"`, because subscription-plan usage has no incremental cost to record.
2. The cost event references the run's `id` (`heartbeatRunId`) so the budget and reporting layers can group spend by run, agent, task, or project.

The cost event chain is what feeds the budget cap system. You will see this explored further in the budgets chapter (P20).

---

## Session IDs: how a run knows where to resume

This is the piece that makes multi-turn agents practical. A conversation with an LLM model often has a session ID — an opaque string the provider assigns so that subsequent requests can continue the same context window. The orchestrator tracks session identity across runs using two fields on the run row (S36):

| Field | Content |
|---|---|
| `sessionIdBefore` | The session display ID that was active *when this run was queued* |
| `sessionIdAfter` | The session display ID produced *after* the adapter finished |

At insert time, `sessionIdBefore` is set to the session that was current for the agent (from the agent runtime state). At finish time, `sessionIdAfter` is set to whatever the adapter returned in `AdapterResult.sessionId` / `sessionDisplayId` (S30):

```ts
// At run finish
await setRunStatus(run.id, status, {
  // ...
  sessionIdAfter: nextSessionState.displayId ?? nextSessionState.legacySessionId,
});
```

`AdapterResult` carries three session-related fields (S24):

| Field | Purpose |
|---|---|
| `sessionId` | Legacy single-string session identifier |
| `sessionParams` | Structured session parameters (preferred) |
| `sessionDisplayId` | Human-readable version of the session identity |

The orchestrator normalizes these into `nextSessionState.displayId` / `nextSessionState.legacySessionId` before persisting them (S30).

### Resuming with a previous run's session

When a new run is queued for the same task scope and there is already a stored session, the orchestrator looks up the most recent session associated with that task and passes it to the adapter as the starting session context. This is the mechanism that lets an agent say "carry on from where we left off."

The explicit resume path works like this (S30):

```ts
// Simplified view of buildExplicitResumeSessionOverride
// Returns the session to pass to the adapter, or null if none
function buildExplicitResumeSessionOverride(input: {
  resumeFromRunId: string;
  resumeRunSessionIdBefore: string | null;
  resumeRunSessionIdAfter: string | null;
  resumeRunSessionParams: Record<string, unknown> | null;
  taskSession: ResumeSessionRow | null;
  sessionCodec: AdapterSessionCodec;
}) {
  // Prefer the structured params from the task session if still current;
  // otherwise fall back to the run's own sessionIdAfter / sessionIdBefore
  // ...
  return sessionDisplayId || sessionParams
    ? { sessionDisplayId, sessionParams }
    : null;
}
```

The `AdapterSessionCodec` (defined on the `ServerAdapterModule`, S24) handles serialisation — different adapters encode their session identity differently, and the codec normalises them to and from `Record<string, unknown>`.

You will notice that `AdapterResult` also carries a `clearSession?: boolean` field (S24). When the adapter sets this to `true`, the orchestrator discards the stored session for the task rather than persisting it. This is used when an adapter detects that its session context has become invalid (for example, the provider has expired the conversation).

---

## Liveness: is the run still making progress?

A run in the `running` status is not always healthy. The orchestrator maintains a parallel assessment of run health in `livenessState` and `livenessReason` (S36):

```ts
// From the Drizzle schema
livenessState: text("liveness_state"),
livenessReason: text("liveness_reason"),
```

After the adapter finishes — whether successfully or not — a liveness classifier reads the run's result and assigns a liveness state. The schema also indexes on `livenessState` for company-wide liveness dashboards:

```ts
// From heartbeat_runs.ts schema (S36)
workspaceLivenessIdx: index("heartbeat_runs_company_liveness_idx").on(
  table.workspaceId,
  table.livenessState,
  table.createdAt,
),
```

Liveness tracking is explored in depth in the recovery chapter (P12). For now, understand that `livenessState` is the orchestrator's assessment of *whether this run produced meaningful work*, distinct from `status`, which is purely about execution outcome.

---

## Retry chains

When a run fails and the failure is recoverable — for example, a transient upstream error from the LLM provider — the orchestrator may schedule a retry. The retry is a **new run row**, and it references the original via `retryOfRunId` (S36):

```ts
retryOfRunId: uuid("retry_of_run_id").references(
  (): AnyPgColumn => heartbeatRuns.id,
  { onDelete: "set null" },
),
```

The cascade rule is `onDelete: "set null"` — if the parent run row is deleted, the retry pointer becomes null rather than cascading the delete. The retry schedule itself is tracked in three fields (S36):

| Field | Purpose |
|---|---|
| `scheduledRetryAt` | When the retry run should be promoted to `queued` |
| `scheduledRetryAttempt` | Which attempt this is (0 = not a retry) |
| `scheduledRetryReason` | Why the retry was scheduled, e.g. `"transient_failure"` |

The transient retry delays are bounded — the service defines a fixed progression of delays (S30):

```ts
// heartbeat.ts — bounded retry delay schedule
export const BOUNDED_TRANSIENT_HEARTBEAT_RETRY_DELAYS_MS = [
  2 * 60 * 1000,    // attempt 1: ~2 min
  10 * 60 * 1000,   // attempt 2: ~10 min
  30 * 60 * 1000,   // attempt 3: ~30 min
  2 * 60 * 60 * 1000, // attempt 4: ~2 hrs
] as const;
```

Each delay has ±25% jitter applied, and retries stop after four attempts. After that, the run remains in a terminal failed state.

---

## The full run record at a glance

Here is a consolidated reference for the fields we have covered, grouped by concern. All columns come from the Drizzle schema in S36 unless otherwise noted.

### Identity and scope

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` | Primary key, random UUID |
| `workspaceId` | `uuid` | Workspace (multi-tenant scoping key) |
| `agentId` | `uuid` | The agent that ran |

### Invocation

| Column | Type | Default | Notes |
|---|---|---|---|
| `invocationSource` | `text` | `"on_demand"` | Why the run was created (`"automation"`, `"on_demand"`, etc.) |
| `triggerDetail` | `text` | — | Fine-grained trigger label (`"manual"`, `"system"`, `"ping"`, `"callback"`) |
| `wakeupRequestId` | `uuid` | — | Links to the wakeup request that spawned this run |
| `continuationAttempt` | `integer` | `0` | Counts within-task continuation restarts |

### Lifecycle

| Column | Type | Default | Notes |
|---|---|---|---|
| `status` | `text` | `"queued"` | `queued` → `running` → terminal |
| `startedAt` | `timestamp` | — | Set when the run transitions to `running` |
| `finishedAt` | `timestamp` | — | Set on terminal transition |
| `error` | `text` | — | Human-readable error message |
| `errorCode` | `text` | — | Machine error code (e.g. `"timeout"`, `"adapter_failed"`) |
| `exitCode` | `integer` | — | OS process exit code from the adapter |
| `signal` | `text` | — | OS signal if the process was signalled |

### Process tracking

| Column | Type | Notes |
|---|---|---|
| `processPid` | `integer` | OS PID of the agent child process |
| `processGroupId` | `integer` | Process group, for clean teardown |
| `processStartedAt` | `timestamp` | When the process was observed to start |

### Log storage

| Column | Type | Notes |
|---|---|---|
| `logStore` | `text` | Backend type for the log (e.g. `"local_file"`) |
| `logRef` | `text` | Opaque reference to the log object within `logStore` |
| `logBytes` | `bigint` | Total stored bytes |
| `logSha256` | `text` | SHA-256 integrity hash |
| `logCompressed` | `boolean` | Default `false` |
| `stdoutExcerpt` | `text` | Inline excerpt for fast UI |
| `stderrExcerpt` | `text` | Inline excerpt for fast UI |

### Live output tracking

| Column | Type | Notes |
|---|---|---|
| `lastOutputAt` | `timestamp` | Timestamp of most recent output chunk |
| `lastOutputSeq` | `integer` | Monotonic sequence number for last output |
| `lastOutputStream` | `text` | `"stdout"` or `"stderr"` |
| `lastOutputBytes` | `bigint` | Cumulative byte position at last flush |

### Usage and cost

| Column | Type | Notes |
|---|---|---|
| `usageJson` | `jsonb` | Token counts, provider, model, cost, session flags |
| `resultJson` | `jsonb` | Adapter result metadata, stop metadata, recovery info |

The `usageJson` blob contains (from the adapter result, S30): `inputTokens`, `outputTokens`, `cachedInputTokens`, `provider`, `model`, `biller`, `billingType`, `costUsd`, and session tracking flags (`sessionReused`, `freshSession`, `sessionRotated`).

### Session continuity

| Column | Type | Notes |
|---|---|---|
| `sessionIdBefore` | `text` | Session display ID active at run start |
| `sessionIdAfter` | `text` | Session display ID produced by this run |

### Context snapshot

| Column | Type | Notes |
|---|---|---|
| `contextSnapshot` | `jsonb` | Task ID, comment ID, wake reason, runtime services |

### Liveness

| Column | Type | Notes |
|---|---|---|
| `livenessState` | `text` | Classifier output (e.g. `"plan_only"`, `"empty_response"`) |
| `livenessReason` | `text` | Human explanation for the liveness state |
| `lastUsefulActionAt` | `timestamp` | When the agent last did substantive work |
| `nextAction` | `text` | What the liveness classifier expects to happen next |

### Retry chain

| Column | Type | Notes |
|---|---|---|
| `retryOfRunId` | `uuid` | Points to the run this is a retry of (`null` if original) |
| `processLossRetryCount` | `integer` | Default `0`; counts process-loss recoveries |
| `scheduledRetryAt` | `timestamp` | When to promote the run from `scheduled_retry` to `queued` |
| `scheduledRetryAttempt` | `integer` | Default `0` |
| `scheduledRetryReason` | `text` | Why the retry was scheduled |

---

## Where runs show up next

The run entity connects to several other parts of the system:

- **Budget enforcement (P20)** — `costService.createEvent` is called with each run's `heartbeatRunId`. Budget caps are checked against cumulative cost events, so a run that pushes an agent over its spend limit triggers a cancellation on the *next* enqueue attempt.
- **Liveness and recovery (P12)** — `livenessState` and the live output tracking fields feed the watchdog that detects stuck runs and decides whether to intervene (for example, by queuing a continuation or a recovery run).
- **The Adapter Registry (P8)** — The registry selected the adapter that produced the `AdapterResult` we've been following through the run finalization path. See [The Adapter Registry](./adapter-registry.md) for how adapter selection works.

---

← Previous: [The Adapter Registry](./adapter-registry.md) · Next: [Modeling Tasks](../tasks-and-queue/modeling-tasks.md) →
