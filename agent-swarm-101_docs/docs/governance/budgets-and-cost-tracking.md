---
title: "Budgets and Cost Tracking"
description: Record cost events per run, aggregate spend by scope, and enforce soft-warning and hard-stop thresholds that auto-pause in-flight agent work.
category: governance
type: tutorial
tags:
  - budget
  - cost tracking
  - cost events
  - cost_events table
  - per-scope aggregation
  - workspace budget
  - agent budget
  - project budget
  - monthly UTC window
  - budgetStatusFromObserved
  - ok
  - warning
  - hard_stop
  - cancelWorkForScope
  - soft alert
  - hard stop
  - auto-pause
  - spend cap
  - BudgetPolicy
  - BudgetIncident
  - billed_cents
  - calendar_month_utc
  - lifetime budget
  - Drizzle
  - PostgreSQL
  - governance
  - orchestrator
keywords:
  - cost visibility
  - token cost
  - LLM spend
  - budget enforcement
  - agent pausing
  - budget thresholds
  - spend aggregation
  - invocation block
  - budget override
  - soft threshold
  - hard threshold
  - warnPercent
  - hardStopEnabled
  - observedAmount
  - budgetMonthlyCents
sources: [S38, S32, S18]
---

**TL;DR** — Agents that call paid LLM APIs can accumulate real costs silently. This chapter shows you how to record a cost event every time a run completes, aggregate those events into per-scope spend totals (workspace, agent, or project) over a monthly UTC window, and automatically pause in-flight work when spend hits a hard-stop threshold. By the end you will have a working budget enforcement loop: a run writes a cost event, the budget service evaluates the new total, issues a soft warning at a configurable threshold, and hard-stops the relevant scope when it reaches the cap.

# Budgets and Cost Tracking

Before we add any budget logic, let's be clear about why it matters. An LLM API call costs money. A scheduler that fires dozens of agents on overlapping cron windows, each processing a multi-turn task, can hit surprising spend in minutes. Without visibility into what each agent is spending, and without a hard ceiling, the system is a liability.

The approach we'll build is straightforward:

1. Every run writes a **cost event** — a structured record of provider, model, tokens, and cost in cents — linked to the agent, task, and project that generated the spend.
2. A budget service **aggregates** those events per scope (workspace, agent, or project) over a rolling monthly UTC window.
3. The service evaluates a **`budgetStatusFromObserved`** function to decide whether the scope is `ok`, approaching a `warning`, or at a `hard_stop`.
4. At `hard_stop`, a `cancelWorkForScope` hook pauses the scope and signals any in-flight runs to stop.

Let's build each piece.

## Prerequisites

**Runs and usage (P9).** Each time an agent is invoked, the orchestrator creates a *run* — a single execution window with a recorded start, finish, and outcome. The run also captures usage: input tokens, output tokens, and the cost in cents that the provider billed. That cost figure is what we'll persist into the `cost_events` table. See [A Run: Sessions, Usage, and Cost](../the-agent/a-run.md) for full detail on how runs report their usage.

**Tasks and projects (P10).** A *task* is the unit of work an agent is assigned. Tasks belong to *projects*, which in turn belong to a workspace. Because cost events carry foreign keys to both the task and project, we get per-project spend rollups for free. See [Modeling Tasks](../tasks-and-queue/modeling-tasks.md) for the task lifecycle.

---

## Step 1 — Recording cost events

When a run finishes, we need to persist what it cost. The question is: what exactly do we store?

We need enough to answer:
- How much did this specific run cost? (for audit and debugging)
- How much has this agent spent this month? (for per-agent budget)
- How much has this project spent overall? (for per-project budget)
- How much has the whole workspace spent this month? (for workspace-level cap)

The `cost_events` table — defined with Drizzle — stores exactly these fields. Here is the schema, brand-scrubbed for our Swarm system:

```ts
// src/db/schema/cost_events.ts
import { pgTable, uuid, text, timestamp, integer, index } from "drizzle-orm/pg-core";
import { workspaces } from "./workspaces.js";
import { agents } from "./agents.js";
import { tasks } from "./tasks.js";      // maps to "issues" in the origin schema
import { projects } from "./projects.js";
import { goals } from "./goals.js";
import { runs } from "./runs.js";        // maps to "heartbeat_runs"

export const costEvents = pgTable(
  "cost_events",
  {
    id:         uuid("id").primaryKey().defaultRandom(),

    // Workspace scope — every event belongs to exactly one workspace
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id),

    // Who generated the cost
    agentId:    uuid("agent_id").notNull().references(() => agents.id),

    // Optional — link to the task, project, goal, and run that generated the cost
    taskId:     uuid("task_id").references(() => tasks.id),
    projectId:  uuid("project_id").references(() => projects.id),
    goalId:     uuid("goal_id").references(() => goals.id),
    runId:      uuid("run_id").references(() => runs.id),

    // What provider/model/billing category was used
    billingCode:  text("billing_code"),
    provider:     text("provider").notNull(),
    biller:       text("biller").notNull().default("unknown"),
    billingType:  text("billing_type").notNull().default("unknown"),
    model:        text("model").notNull(),

    // Token breakdown
    inputTokens:        integer("input_tokens").notNull().default(0),
    cachedInputTokens:  integer("cached_input_tokens").notNull().default(0),
    outputTokens:       integer("output_tokens").notNull().default(0),

    // The cost itself, stored as integer cents to avoid floating-point rounding
    costCents:    integer("cost_cents").notNull(),

    // When the cost was incurred (used for windowed aggregation)
    occurredAt:   timestamp("occurred_at", { withTimezone: true }).notNull(),
    createdAt:    timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    // Fast workspace-level spend rollup
    workspaceOccurredIdx: index("cost_events_workspace_occurred_idx")
      .on(table.workspaceId, table.occurredAt),

    // Fast per-agent spend rollup within a workspace
    workspaceAgentOccurredIdx: index("cost_events_workspace_agent_occurred_idx")
      .on(table.workspaceId, table.agentId, table.occurredAt),

    // Per-provider reporting
    workspaceProviderOccurredIdx: index("cost_events_workspace_provider_occurred_idx")
      .on(table.workspaceId, table.provider, table.occurredAt),

    // Per-biller reporting
    workspaceBillerOccurredIdx: index("cost_events_workspace_biller_occurred_idx")
      .on(table.workspaceId, table.biller, table.occurredAt),

    // Look up all cost events for a given run
    workspaceRunIdx: index("cost_events_workspace_run_idx")
      .on(table.workspaceId, table.runId),
  }),
);
```

A few design decisions are worth noticing here:

- **`costCents` is an integer.** Storing money as integer cents sidesteps floating-point rounding when summing thousands of events.
- **`occurredAt` is separate from `createdAt`.** The run might finish at 23:59 UTC and the record inserted a second later. Budget windows are evaluated against `occurredAt`, so the cost lands in the correct calendar window.
- **Composite indexes.** Queries that sum spend per workspace, per agent, and per provider use `occurredAt` as the second key. The indexes match the exact query pattern the budget service will run.
- **`taskId`, `projectId`, `goalId`, and `runId` are nullable.** A workspace-level cost event that doesn't trace to a specific task is valid — but we need at least `workspaceId` and `agentId`.

### Writing a cost event from a run

When a run completes and reports its usage back to the orchestrator (see [A Run: Sessions, Usage, and Cost](../the-agent/a-run.md)), the orchestrator inserts a cost event. Here is the insertion — simplified to show the essential fields:

```ts
// src/orchestrator/cost.ts  (simplified)
import { db } from "@swarm/db";
import { costEvents } from "@swarm/db/schema";

export async function recordRunCost(opts: {
  workspaceId: string;
  agentId: string;
  runId: string;
  taskId?: string | null;
  projectId?: string | null;
  provider: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  costCents: number;
  occurredAt: Date;
}) {
  await db.insert(costEvents).values({
    workspaceId:  opts.workspaceId,
    agentId:      opts.agentId,
    runId:        opts.runId,
    taskId:       opts.taskId ?? null,
    projectId:    opts.projectId ?? null,
    provider:     opts.provider,
    model:        opts.model,
    inputTokens:  opts.inputTokens,
    outputTokens: opts.outputTokens,
    costCents:    opts.costCents,
    occurredAt:   opts.occurredAt,
  });
}
```

After inserting the record, the orchestrator should evaluate the applicable budget policies — we'll wire that in Step 4.

---

## Step 2 — Aggregating spend per scope

Now that cost events are flowing into the database, we need to answer "how much has scope X spent this window?" The scope can be:

| Scope | What it covers |
|---|---|
| `workspace` | The entire multi-tenant workspace — all agents, all projects |
| `agent` | One specific agent, across all its tasks |
| `project` | One specific project, across all agents working on it |

And the window can be:

| Window kind | Meaning |
|---|---|
| `calendar_month_utc` | From `UTC(year, month, 1)` to `UTC(year, month+1, 1)` — a rolling calendar month |
| `lifetime` | From epoch to far future — the full history of the scope |

The default for workspace and agent budgets is `calendar_month_utc`. Project budgets default to `lifetime` because projects often span multiple months, and a per-project cap makes sense against total project spend, not a single month.

Here is how the budget service computes the monthly UTC window boundaries:

```ts
// src/services/budgets.ts  (from source)
function currentUtcMonthWindow(now = new Date()) {
  const year  = now.getUTCFullYear();
  const month = now.getUTCMonth();
  const start = new Date(Date.UTC(year, month,     1, 0, 0, 0, 0));
  const end   = new Date(Date.UTC(year, month + 1, 1, 0, 0, 0, 0));
  return { start, end };
}

function resolveWindow(windowKind: "calendar_month_utc" | "lifetime", now = new Date()) {
  if (windowKind === "lifetime") {
    return {
      start: new Date(Date.UTC(1970, 0, 1, 0, 0, 0, 0)),
      end:   new Date(Date.UTC(9999, 0, 1, 0, 0, 0, 0)),
    };
  }
  return currentUtcMonthWindow(now);
}
```

Notice that both boundaries are UTC midnight. A `calendar_month_utc` window always starts on the first of the month and ends on the first of the next month, so events exactly on the boundary fall cleanly into one side.

### The aggregation query

Given a policy (which carries `workspaceId`, `scopeType`, `scopeId`, `windowKind`), the service counts total `costCents` over the window:

```ts
// src/services/budgets.ts  (simplified view of computeObservedAmount)
import { and, eq, gte, lt, sql } from "drizzle-orm";
import { costEvents } from "@swarm/db/schema";

async function computeObservedAmount(
  db: Db,
  policy: {
    workspaceId: string;
    scopeType:   "workspace" | "agent" | "project";
    scopeId:     string;
    windowKind:  "calendar_month_utc" | "lifetime";
    metric:      string;
  },
): Promise<number> {
  // Currently the only supported metric is billed_cents
  if (policy.metric !== "billed_cents") return 0;

  const conditions = [eq(costEvents.workspaceId, policy.workspaceId)];

  // Narrow to the correct scope
  if (policy.scopeType === "agent") {
    conditions.push(eq(costEvents.agentId, policy.scopeId));
  }
  if (policy.scopeType === "project") {
    conditions.push(eq(costEvents.projectId, policy.scopeId));
  }
  // workspace scope: no extra filter — all events for the workspace are in scope

  // Apply the time window for calendar_month_utc budgets
  const { start, end } = resolveWindow(policy.windowKind);
  if (policy.windowKind === "calendar_month_utc") {
    conditions.push(gte(costEvents.occurredAt, start));
    conditions.push(lt(costEvents.occurredAt, end));
  }

  const [row] = await db
    .select({
      total: sql<number>`coalesce(sum(${costEvents.costCents}), 0)::double precision`,
    })
    .from(costEvents)
    .where(and(...conditions));

  return Number(row?.total ?? 0);
}
```

You might wonder: if we have hundreds of thousands of cost events, won't a full `SUM` on every budget check be slow? For V1, the composite indexes we defined in Step 1 make this query fast enough. The window filter (`occurredAt >= start AND occurredAt < end`) combined with the workspace and agent keys hits a narrow B-tree range. If query latency becomes a concern at scale, pre-aggregated rollup tables can be added without changing the policy interface.

---

## Step 3 — Evaluating thresholds

Now we know how much a scope has spent. We need to decide what that means relative to its budget. The function `budgetStatusFromObserved` encodes the two-tier threshold model:

```ts
// src/services/budgets.ts  (from source)
function budgetStatusFromObserved(
  observedAmount: number,   // how much has been spent (cents)
  amount: number,           // the budget cap (cents)
  warnPercent: number,      // the soft-alert threshold, e.g. 80
): "ok" | "warning" | "hard_stop" {
  if (amount <= 0) return "ok";                              // no active budget → always ok
  if (observedAmount >= amount) return "hard_stop";          // at or over cap
  if (observedAmount >= Math.ceil((amount * warnPercent) / 100)) return "warning"; // soft zone
  return "ok";
}
```

Let's trace through an example. Suppose an agent has a monthly budget of 1000 cents (US$10.00) and the default `warnPercent` of 80:

| Observed spend (cents) | Status |
|---|---|
| 0–799 | `ok` |
| 800–999 | `warning` (≥ 80% of 1000) |
| 1000+ | `hard_stop` (≥ 100% of 1000) |

The `warnPercent` default is **80** (as set in the source when a new policy is inserted without an explicit `warnPercent`). The thresholds are configurable per policy: you can set `warnPercent: 90` for a looser warning, or `warnPercent: 50` for an early heads-up.

### Budget layers

The spec defines three independent budget layers, each evaluated separately:

```
Workspace budget  →  caps total monthly spend across all agents
        ↓
Agent budget      →  caps spend for one agent regardless of project
        ↓
Project budget    →  caps cumulative spend within a specific project
```

Each layer carries its own `BudgetPolicy` row. When a cost event arrives, the budget service checks **all three** policy types relevant to that event and enforces whichever one trips first. This means an agent can be stopped by any of: its own agent cap, the workspace cap, or the cap on the project it was working in.

A `BudgetPolicy` record holds:

| Field | Meaning |
|---|---|
| `scopeType` | `"workspace"`, `"agent"`, or `"project"` |
| `scopeId` | UUID of the workspace, agent, or project |
| `metric` | `"billed_cents"` (the only metric in V1) |
| `windowKind` | `"calendar_month_utc"` or `"lifetime"` |
| `amount` | Budget cap in integer cents |
| `warnPercent` | Soft-alert percentage (default 80) |
| `hardStopEnabled` | Whether the hard stop fires (default `true`) |
| `notifyEnabled` | Whether soft-alert incidents are created (default `true`) |
| `isActive` | Whether the policy is currently active |

---

## Step 4 — Enforcing the budget

Knowing the status is `warning` or `hard_stop` is only useful if we act on it. The budget service fires enforcement inside `evaluateCostEvent`, which runs after every cost event is written. Let's walk through what it does:

```ts
// src/services/budgets.ts  (simplified view of evaluateCostEvent)
async function evaluateCostEvent(event: CostEvent) {
  // 1. Fetch all active policies for this workspace that might apply
  const relevantPolicies = await getActivePoliciesForEvent(event);

  for (const policy of relevantPolicies) {
    if (policy.metric !== "billed_cents" || policy.amount <= 0) continue;

    const observedAmount = await computeObservedAmount(db, policy);
    const softThreshold  = Math.ceil((policy.amount * policy.warnPercent) / 100);

    // 2. Soft alert — create an incident if we've crossed the warn threshold
    if (policy.notifyEnabled && observedAmount >= softThreshold) {
      await createIncidentIfNeeded(policy, "soft", observedAmount);
      // (Incident creation is idempotent — duplicate incidents for the same
      //  window are suppressed.)
    }

    // 3. Hard stop — pause the scope and cancel in-flight work
    if (policy.hardStopEnabled && observedAmount >= policy.amount) {
      await resolveOpenSoftIncidents(policy.id);      // promote soft → resolved
      await createIncidentIfNeeded(policy, "hard", observedAmount);
      await pauseAndCancelScopeForBudget(policy);     // the key enforcement step
    }
  }
}
```

The two enforcement paths are:

**Soft alert.** A `BudgetIncident` record is written with `thresholdType: "soft"`. The orchestrator's notification layer picks this up and surfaces it to the board operator. Work continues — this is a warning, not a stop.

**Hard stop.** Three things happen atomically:
1. Open soft incidents for the policy are marked resolved (they're superseded by the hard incident).
2. A `BudgetIncident` record is written with `thresholdType: "hard"`.
3. `pauseAndCancelScopeForBudget(policy)` runs — which we'll examine next.

### What `pauseAndCancelScopeForBudget` does

```ts
// src/services/budgets.ts  (from source, simplified)
async function pauseAndCancelScopeForBudget(policy: PolicyRow) {
  // Step A: mark the database record as paused with reason "budget"
  await pauseScopeForBudget(policy);

  // Step B: call the cancelWorkForScope hook so in-flight runs are signalled
  await hooks.cancelWorkForScope?.({
    workspaceId: policy.workspaceId,
    scopeType:   policy.scopeType as BudgetScopeType,
    scopeId:     policy.scopeId,
  });
}
```

The pause writes depend on the scope type:

| Scope | What gets written |
|---|---|
| `agent` | `agents.status = "paused"`, `agents.pauseReason = "budget"`, `agents.pausedAt = now` (only if agent is currently `active`, `idle`, `running`, or `error`) |
| `project` | `projects.pauseReason = "budget"`, `projects.pausedAt = now` |
| `workspace` | `workspaces.status = "paused"`, `workspaces.pauseReason = "budget"`, `workspaces.pausedAt = now` |

After the database records are updated, the `cancelWorkForScope` hook fires. This hook is provided by the caller when constructing the budget service — it is a seam that connects the budget system to the run/adapter layer. A typical implementation cancels any in-flight runs for the paused scope by calling the adapter's `cancel()` method for each active run.

### How the scheduler respects budget pauses

You might wonder: what stops the scheduler from starting *new* runs for the paused agent immediately? The scheduler checks the agent's status before each invocation attempt. An agent with `status = "paused"` is skipped. The `getInvocationBlock` function in the budget service makes this explicit — it returns a blocking reason for any of three conditions:

```ts
// Invocation is blocked when ANY of these is true:
// 1. The workspace itself is paused with pauseReason = "budget"
// 2. The agent is paused with pauseReason = "budget"
// 3. The project the task belongs to is paused with pauseReason = "budget"
```

The scheduler calls `getInvocationBlock` before firing a run. If a block is returned, the run is not started and the block reason is surfaced to the board operator.

### Resuming after a hard stop

Work does not resume automatically. A board operator must either:

- **Raise the budget** — update the policy's `amount` to a value above the current `observedAmount`. The budget service will call `resumeScopeFromBudget`, which clears `status`/`pauseReason`/`pausedAt` for the scope, resolves open incidents, and marks any associated approvals. After this, the scheduler can invoke the agent again on the next tick.
- **Dismiss the incident** — mark the incident as dismissed without raising the cap. The scope stays paused until the board takes further action.

The `resolveIncident` flow checks that the new amount exceeds the observed amount before applying the raise:

```ts
// Simplified view of incident resolution
if (input.action === "raise_budget_and_resume") {
  const currentObserved = await computeObservedAmount(db, policy);
  if (nextAmount <= currentObserved) {
    throw unprocessable("New budget must exceed current observed spend");
  }
  // ... update policy amount, resume scope, resolve incidents
}
```

This guard prevents the operator from accidentally setting a new cap that is already blown, which would immediately re-trigger the hard stop.

---

## Putting it together — the full enforcement loop

Here is the end-to-end flow from a run completing to enforcement firing:

```
Run completes
    ↓
recordRunCost()  →  INSERT INTO cost_events (agent, task, project, tokens, costCents, occurredAt)
    ↓
evaluateCostEvent(event)
    ↓  for each relevant policy:
    ├─ computeObservedAmount()   →  SUM(costCents) WHERE workspace + scope + window
    ├─ budgetStatusFromObserved()  →  ok / warning / hard_stop
    │
    ├─ [warning]  createIncidentIfNeeded(policy, "soft")
    │                 →  INSERT budget_incidents (thresholdType="soft", status="open")
    │
    └─ [hard_stop]  createIncidentIfNeeded(policy, "hard")
                    pauseAndCancelScopeForBudget(policy)
                        ├─ UPDATE agents SET status="paused", pauseReason="budget"
                        └─ cancelWorkForScope hook  →  adapter.cancel() for active runs
```

The scheduler's `getInvocationBlock` check sits at the entry point of the invocation path. Any time it finds `pauseReason = "budget"` on the workspace, agent, or project, it returns a block, and no new run is started.

---

## Try it yourself

The following exercise walks through the full loop using the mock adapter. The mock adapter supports configurable per-run costs, which lets us drive a budget to exhaustion in a controlled way.

**Setup: create a workspace and an agent with a small budget**

```ts
// 1. Create a workspace and agent (see earlier chapters for the full setup)
// 2. Set a budget policy for the agent: 200 cents = US$2.00, warn at 80%
await budgetService.upsertPolicy(workspaceId, {
  scopeType:       "agent",
  scopeId:         agentId,
  metric:          "billed_cents",
  windowKind:      "calendar_month_utc",
  amount:          200,   // $2.00 in cents
  warnPercent:     80,
  hardStopEnabled: true,
  notifyEnabled:   true,
});
```

**Step 1: run the agent until the soft warning fires**

Configure the mock adapter to report 50 cents per run. After the third run (150 cents observed, 75% utilization), the fourth run (200 cents would exceed 80%) should push the status into `warning`.

```ts
// Record four cost events and evaluate after each
for (const costCents of [50, 50, 50, 20]) {
  await recordRunCost({ workspaceId, agentId, costCents, occurredAt: new Date(), ... });
  await budgetService.evaluateCostEvent({ workspaceId, agentId, costCents, ... });
}

// After the fourth event, observed = 170 cents → 85% of 200 → status: warning
const summary = await budgetService.buildPolicySummary(policy);
console.log(summary.status);            // "warning"
console.log(summary.utilizationPercent); // 85.00
```

You should see an open `BudgetIncident` with `thresholdType: "soft"`.

**Step 2: push to hard stop**

Add one more event that brings the total to 200 cents or beyond:

```ts
await recordRunCost({ workspaceId, agentId, costCents: 30, occurredAt: new Date(), ... });
await budgetService.evaluateCostEvent({ workspaceId, agentId, costCents: 30, ... });
// Observed = 200 cents → hard_stop
```

Check the agent's status in the database:

```sql
SELECT status, pause_reason FROM agents WHERE id = '<agentId>';
-- status: paused, pause_reason: budget
```

Attempt to start a new run — `getInvocationBlock` should return a block reason.

**Step 3: raise the cap and watch work resume**

```ts
await budgetService.resolveIncident(workspaceId, hardIncidentId, {
  action: "raise_budget_and_resume",
  amount: 500,  // raise to $5.00
}, boardUserId);
```

The agent's status returns to `idle`. The next scheduler tick will invoke it normally.

**Variation: aggregate spend per project**

Create a project budget and assign a few tasks to it. Record cost events with `projectId` set. The aggregation query will sum only events for that project:

```ts
await budgetService.upsertPolicy(workspaceId, {
  scopeType:  "project",
  scopeId:    projectId,
  windowKind: "lifetime",   // project budgets default to lifetime
  amount:     5000,         // $50.00 cumulative
  warnPercent: 80,
});
```

When the project's cumulative spend reaches 4000 cents, the soft warning fires. At 5000 cents, the project is paused and any agent working on a task in that project will be blocked from starting new runs for it.

---

## What we've built

| Concern | Mechanism |
|---|---|
| Per-run cost recording | `cost_events` table: provider, model, tokens, `costCents`, `occurredAt` |
| Monthly windowing | `currentUtcMonthWindow` — calendar-aligned UTC boundaries |
| Per-scope aggregation | `computeObservedAmount` — `SUM(costCents)` with composite index on workspace + scope + `occurredAt` |
| Threshold evaluation | `budgetStatusFromObserved` — `ok` / `warning` / `hard_stop` based on configurable `warnPercent` |
| Soft alert | `BudgetIncident` with `thresholdType: "soft"` — work continues, board is notified |
| Hard stop | `pauseAndCancelScopeForBudget` — scope paused in DB, `cancelWorkForScope` hook fires to terminate in-flight runs |
| New-invocation guard | `getInvocationBlock` — scheduler checks for paused scopes before each run |
| Recovery | `raise_budget_and_resume` on the incident — new cap must exceed observed spend; scope resumes automatically |

The next chapter adds a second governance layer on top of this one: approval gates that route certain agent actions — including budget overrides — through a board-operator review queue.

---

← Previous: [The Scheduler Loop](../scheduling/scheduler-loop.md) · Next: [Approvals and Governance Gates](./approvals-and-governance-gates.md) →
