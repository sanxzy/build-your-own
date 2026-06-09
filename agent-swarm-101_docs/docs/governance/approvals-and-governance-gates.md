---
title: "Approvals and Governance Gates"
description: Add human-in-the-loop governance to agent hire and strategy decisions using an optimistic-update resolution flow with a race re-fetch guard.
category: governance
type: tutorial
tags:
  - approvals
  - governance
  - hire approval
  - strategy approval
  - goal alignment
  - optimistic update
  - resolveApproval
  - canResolveStatuses
  - race re-fetch
  - approval gate
  - goal hierarchy
  - executive agent
  - engineer agent
  - pending
  - revision_requested
  - approve_ceo_strategy
  - hire_agent
  - notifyHireApproved
  - budget integration
  - concurrent resolution
  - approval state machine
keywords:
  - human-in-the-loop
  - approval workflow
  - concurrent approval guard
  - optimistic locking
  - governance gate
  - agent hiring workflow
  - CEO strategy approval
  - board approval
  - task hierarchy
  - goal traceability
sources: [S33, S18, S19]
---

**TL;DR** — Some decisions — hiring a new agent, committing to a strategy, approving large spend — are too consequential to leave fully autonomous. This chapter shows you how to add human-in-the-loop governance gates to those decisions. We'll build up the approval state machine, implement an optimistic-update resolution function that stays safe when two reviewers click at the same time, wire hire and strategy approval flows, and connect a post-approval budget check so an approved plan still respects spend caps.

# Approvals and Governance Gates

Autonomous agents are good at executing well-defined work. But some decisions sit above the execution plane. Hiring a new agent changes the org chart. Committing to a strategy determines what dozens of tasks will exist next week. Approving a plan that exceeds a monthly budget cap affects every agent's spend envelope.

These are decisions where a human should confirm before anything executes.

The previous governance control we added — budgets and spend tracking — lets you cap how much any agent can spend (see [Budgets and Cost Tracking](./budgets-and-cost-tracking.md)). Approvals sit one level higher: they gate whether an action is even permitted to start.

We also need to understand where approvals fit in the broader picture. Every piece of work in Swarm must trace back to a top-level goal through a chain of parent tasks. Approvals live at the goal and strategy layer: a human approves at the goal or strategy level, and agents execute beneath that approved structure. We'll return to this hierarchy at the end of the chapter after we have the mechanics in place.

## The Approval Entity and Its States

Let's start with the data. An approval is a record that asks: "should this action be allowed?" It sits in a pending state until a board operator (a human reviewer) resolves it.

The approval schema carries these fields (from the data model):

| Field | Type | Purpose |
|---|---|---|
| `id` | uuid | Unique identifier |
| `workspaceId` | uuid | Workspace scope |
| `type` | enum | What kind of decision this is (see below) |
| `requestedByAgentId` | uuid \| null | The agent that requested the action |
| `requestedByUserId` | uuid \| null | The user that requested the action |
| `status` | enum | Current state of the approval |
| `payload` | jsonb | The proposed action details (agent draft, strategy text, etc.) |
| `decisionNote` | text \| null | The reviewer's comment |
| `decidedByUserId` | uuid \| null | Which reviewer resolved it |
| `decidedAt` | timestamptz \| null | When the decision was made |

The `type` field tells you which governance gate is in play:

| Type value | What it gates |
|---|---|
| `hire_agent` | Creating a new agent in the org chart |
| `approve_ceo_strategy` | The executive agent's proposed strategic plan |
| `budget_override_required` | Spend that would exceed a budget cap |
| `request_board_approval` | Generic board escalation for anything else |

### States and Allowed Transitions

An approval moves through these states:

```
pending  ──────────────────────→  approved   (terminal)
   │                           
   ├──────────────────────────→  rejected   (terminal)
   │
   ├──────────────────────────→  cancelled  (terminal)
   │
   └──────→  revision_requested
                  │
                  └──→  pending  (after agent resubmits)
```

The state machine in plain language:

- A new approval starts as `pending`.
- A reviewer can move it to `approved` or `rejected` — both are terminal.
- A reviewer can also ask for changes by moving it to `revision_requested`.
- The requesting agent can then resubmit (moving it back to `pending`) with an updated payload.
- Any approval can be cancelled — also terminal.

Now here is the key constraint that shapes the resolution code: **only `pending` and `revision_requested` approvals can be approved or rejected.** Once an approval has been approved, rejected, or cancelled, no further resolution is possible.

In code, this constraint is captured in a single set:

```ts
// Simplified view of the approval service constructor
const canResolveStatuses = new Set(["pending", "revision_requested"]);
const resolvableStatuses = Array.from(canResolveStatuses);
```

We'll use `canResolveStatuses` to guard every resolution attempt.

## Resolving an Approval — and the Concurrent-Reviewer Problem

Now we need a function to resolve an approval. At first glance, the logic looks like:

1. Load the approval.
2. Check that its current status is resolvable.
3. Update it to `approved` or `rejected`.

That works for a single reviewer. But what happens when two board operators are looking at the same approval at the same time? Reviewer A and Reviewer B both see `status: pending`. Both click "Approve" within milliseconds of each other. Without a guard, both updates succeed and we get duplicate side effects — two agent creations, two hire notifications, two budget policies.

We need to make the update atomic: only the first write wins, and the second write detects that the race has already been resolved.

### The Optimistic Update Pattern

The solution is an **optimistic update** guarded by the status we read. Instead of a bare `UPDATE approvals SET status = ? WHERE id = ?`, we add the status constraint to the WHERE clause:

```ts
UPDATE approvals
SET status = ?, decided_by_user_id = ?, decision_note = ?, decided_at = ?
WHERE id = ? AND status IN ('pending', 'revision_requested')
```

If the row has already been updated by a concurrent resolver, the `status IN (...)` predicate will not match and the update returns zero rows. The second reviewer's write simply does nothing.

### The Race Re-fetch Guard

An `UPDATE` that returns zero rows is ambiguous on its own. It could mean:
- The row was already resolved by a concurrent reviewer with the same outcome (both wanted to approve — that's fine, we can return the existing result), or
- The row was already resolved with a conflicting outcome (one approved, one tried to reject), or
- The row never existed.

To distinguish these cases, we re-fetch after a zero-row update. Here is the full `resolveApproval` function:

```ts
// From the approval service — brand-agnostic version
type ApprovalRecord = typeof approvals.$inferSelect;
type ResolutionResult = { approval: ApprovalRecord; applied: boolean };

async function resolveApproval(
  id: string,
  targetStatus: "approved" | "rejected",
  decidedByUserId: string,
  decisionNote: string | null | undefined,
): Promise<ResolutionResult> {
  // Step 1: Fetch the current record
  const existing = await getExistingApproval(id); // throws 404 if missing

  // Step 2: Pre-check — is this approval even resolvable?
  if (!canResolveStatuses.has(existing.status)) {
    if (existing.status === targetStatus) {
      // Already resolved to the same outcome — idempotent, no work to do
      return { approval: existing, applied: false };
    }
    throw unprocessable(
      `Only pending or revision requested approvals can be ${
        targetStatus === "approved" ? "approved" : "rejected"
      }`,
    );
  }

  // Step 3: Optimistic UPDATE — only succeeds if status is still resolvable
  const now = new Date();
  const updated = await db
    .update(approvals)
    .set({
      status: targetStatus,
      decidedByUserId,
      decisionNote: decisionNote ?? null,
      decidedAt: now,
      updatedAt: now,
    })
    .where(
      and(
        eq(approvals.id, id),
        inArray(approvals.status, resolvableStatuses), // ← the race guard
      ),
    )
    .returning()
    .then((rows) => rows[0] ?? null);

  if (updated) {
    // Our write won the race
    return { approval: updated, applied: true };
  }

  // Step 4: Zero rows updated — race re-fetch to find out why
  const latest = await getExistingApproval(id);
  if (latest.status === targetStatus) {
    // Concurrent resolver got here first with the same outcome — acceptable
    return { approval: latest, applied: false };
  }

  // Conflicting concurrent resolution — surface the error
  throw unprocessable(
    `Only pending or revision requested approvals can be ${
      targetStatus === "approved" ? "approved" : "rejected"
    }`,
  );
}
```

Notice the return type: `{ approval, applied }`. The `applied` boolean tells the caller whether the current call performed the resolution or whether a concurrent resolver already did. The caller uses this to decide whether to trigger side effects — we should only send a hire notification once, not twice.

This pattern will feel familiar if you've worked through the task checkout mechanism elsewhere in this library. Both use the same idea: guard the write with the pre-condition in the WHERE clause, then re-fetch on a miss to distinguish a race from an error.

## Approval Gates in Action

With `resolveApproval` in place, we can build the two most important gates: hiring and strategy.

### Gate 1: Hire Approval

Hiring an agent into the org chart is irreversible in most practical senses: you configure an adapter, allocate a monthly budget, and notify systems that a new agent is active. That's why the hire flow goes through an approval gate before any of that happens.

The flow from the spec:

1. An agent (or board operator) creates an approval record with `type: "hire_agent"` and a `payload` containing the agent's proposed configuration: name, role, title, reporting line, adapter type and config, and requested budget.
2. The approval sits in `pending` until a board operator acts.
3. On approval, the server creates the agent row (or activates a draft agent), upserts a budget policy if the hire includes a budget, and fires a hire notification.
4. On rejection, if a draft agent exists in the payload, it is terminated.

Here is the approve handler:

```ts
approve: async (id: string, decidedByUserId: string, decisionNote?: string | null) => {
  // Resolve the approval — safe against concurrent reviewers
  const { approval: updated, applied } = await resolveApproval(
    id,
    "approved",
    decidedByUserId,
    decisionNote,
  );

  let hireApprovedAgentId: string | null = null;
  const now = new Date();

  // Only run side effects if this call actually applied the resolution
  if (applied && updated.type === "hire_agent") {
    const payload = updated.payload as Record<string, unknown>;
    const payloadAgentId = typeof payload.agentId === "string" ? payload.agentId : null;

    if (payloadAgentId) {
      // A draft agent was created at request time — activate it
      await agentsSvc.activatePendingApproval(payloadAgentId);
      hireApprovedAgentId = payloadAgentId;
    } else {
      // No draft — create the agent fresh from the payload
      const created = await agentsSvc.create(updated.workspaceId, {
        name: String(payload.name ?? "New Agent"),
        role: String(payload.role ?? "general"),
        title: typeof payload.title === "string" ? payload.title : null,
        reportsTo: typeof payload.reportsTo === "string" ? payload.reportsTo : null,
        capabilities: typeof payload.capabilities === "string" ? payload.capabilities : null,
        adapterType: String(payload.adapterType ?? "process"),
        adapterConfig:
          typeof payload.adapterConfig === "object" && payload.adapterConfig !== null
            ? (payload.adapterConfig as Record<string, unknown>)
            : {},
        budgetMonthlyCents:
          typeof payload.budgetMonthlyCents === "number" ? payload.budgetMonthlyCents : 0,
        metadata:
          typeof payload.metadata === "object" && payload.metadata !== null
            ? (payload.metadata as Record<string, unknown>)
            : null,
        status: "idle",
        spentMonthlyCents: 0,
        permissions: undefined,
        lastHeartbeatAt: null,
      });
      hireApprovedAgentId = created?.id ?? null;
    }

    if (hireApprovedAgentId) {
      // Post-approval budget check: if the hire requested a budget, enforce it now
      const budgetMonthlyCents =
        typeof payload.budgetMonthlyCents === "number" ? payload.budgetMonthlyCents : 0;
      if (budgetMonthlyCents > 0) {
        await budgets.upsertPolicy(
          updated.workspaceId,
          {
            scopeType: "agent",
            scopeId: hireApprovedAgentId,
            amount: budgetMonthlyCents,
            windowKind: "calendar_month_utc",
          },
          decidedByUserId,
        );
      }

      // Notify downstream systems that the hire is approved
      void notifyHireApproved(db, {
        workspaceId: updated.workspaceId,
        agentId: hireApprovedAgentId,
        source: "approval",
        sourceId: id,
        approvedAt: now,
      }).catch(() => {});
    }
  }

  return { approval: updated, applied };
},
```

A few things to notice here:

**The `applied` guard is load-bearing.** All side effects — creating the agent, setting the budget, sending the notification — are wrapped in `if (applied && updated.type === "hire_agent")`. If a concurrent reviewer already resolved this approval, `applied` is `false` and we skip every side effect. The second call returns the existing resolved approval without duplicating any work.

**The budget policy is set at approval time, not at request time.** An agent can propose a monthly budget in its hire request payload, but the budget policy only becomes active after the board approves the hire. This is intentional: the board's approval is also the moment they confirm the spend envelope. (If you need a refresher on budget policies and how `upsertPolicy` works, see [Budgets and Cost Tracking](./budgets-and-cost-tracking.md).)

**The notification is fire-and-forget.** `notifyHireApproved` is called with `void` and `.catch(() => {})`. A notification failure should not roll back a successful hire.

The reject handler mirrors the same structure but terminates any draft agent instead:

```ts
reject: async (id: string, decidedByUserId: string, decisionNote?: string | null) => {
  const { approval: updated, applied } = await resolveApproval(
    id,
    "rejected",
    decidedByUserId,
    decisionNote,
  );

  if (applied && updated.type === "hire_agent") {
    const payload = updated.payload as Record<string, unknown>;
    const payloadAgentId = typeof payload.agentId === "string" ? payload.agentId : null;
    if (payloadAgentId) {
      await agentsSvc.terminate(payloadAgentId);
    }
  }

  return { approval: updated, applied };
},
```

### Gate 2: Strategy Approval

The second major gate is the executive agent's strategy. Before a swarm can execute any significant plan, the board reviews and approves it.

The flow:

1. The executive (CEO) agent proposes a strategy as an approval with `type: "approve_ceo_strategy"`. The payload contains the plan text, proposed structure, and high-level tasks.
2. The board reviews.
3. **Before the first strategy approval**, the executive agent may only draft tasks. It cannot transition any task to an active execution state.
4. Once the board approves, that execution lock lifts and delegated work can begin.

This constraint is spelled out in the spec:

> Before first strategy approval, CEO may only draft tasks, not transition them to active execution states.

From a data perspective, the strategy approval uses the same `resolveApproval` function — the mechanics are identical to hire approval. What changes is what happens after:

- A hire approval creates an agent.
- A strategy approval lifts an execution gate on the executive agent's pending tasks.

The separation is deliberate: the board approves the goal, the agents execute beneath it. Which brings us to the bigger picture.

## Goal Alignment: Why Approvals Sit at the Top

Approvals do not exist in isolation. They fit into a broader structure: in Swarm, every piece of work must trace back to the workspace's top-level goal.

The task hierarchy looks like this (from the product definition):

```
Company goal: "Build the #1 AI note-taking app to $1M MRR in 3 months"
  └── Strategic goal: "Grow new signups by 100 users"
        └── Project goal: "Create Facebook ads"
              └── Task: "Research Facebook ads competitors use"
```

Every task exists in service of a parent, all the way to the root goal. This is what keeps agents aligned — they can always answer "why am I doing this?"

The goal schema supports this hierarchy with a `level` field and a `parentId` self-reference:

| Level | Meaning |
|---|---|
| `company` | The root goal — at least one required per workspace |
| `team` | Mid-level objective owned by a team or squad |
| `agent` | An individual agent's current focus area |
| `task` | A specific unit of work |

Approvals connect to this hierarchy at the top two levels: a strategy approval gates execution at the `company` or `team` level. A hire approval adds an agent who will then own goals at the `agent` level.

### Executive and Engineer Agents

Two agent personas illustrate how goal alignment plays out at runtime:

**Executive agent** (e.g., a CEO role): On each heartbeat, reviews what the team is doing, checks metrics, reprioritizes if needed, and assigns new strategic initiatives. The executive agent's entire existence is structured around the board-approved strategy. Before that approval, it can only prepare — draft tasks, outline plans. After approval, it directs the work.

**Engineer agent**: On each heartbeat, checks assigned tasks, picks the highest priority, and works it. The engineer operates beneath the approved strategy. It doesn't need to see the approval flow; it receives tasks that already exist because a strategy was approved and an executive decomposed that strategy into work items.

The board-level approval gate is what makes it safe to let the engineer agent run autonomously. The consequential decisions — what to build, who to hire, how much to spend — have already been reviewed by a human. The engineer executes within those boundaries.

## The Revision Loop

Not every approval is a binary approve/reject. Sometimes a reviewer needs changes before they can approve. The revision flow:

1. Reviewer calls `requestRevision` on a `pending` approval — status moves to `revision_requested`, and the reviewer's note explains what needs to change.
2. The requesting agent reads the note, updates its proposal, and calls `resubmit` — status returns to `pending` with the new payload.
3. The reviewer acts again.

The `requestRevision` function only accepts `pending` approvals — you cannot request revision on something already in `revision_requested`:

```ts
requestRevision: async (id, decidedByUserId, decisionNote?) => {
  const existing = await getExistingApproval(id);
  if (existing.status !== "pending") {
    throw unprocessable("Only pending approvals can request revision");
  }
  // ... update to revision_requested
},
```

And `resubmit` only accepts `revision_requested` approvals — you cannot resubmit something that is still pending:

```ts
resubmit: async (id, payload?) => {
  const existing = await getExistingApproval(id);
  if (existing.status !== "revision_requested") {
    throw unprocessable("Only revision requested approvals can be resubmitted");
  }
  // ... reset to pending, clearing the previous decision fields
  // decisionNote → null, decidedByUserId → null, decidedAt → null
},
```

Notice that `resubmit` clears `decisionNote`, `decidedByUserId`, and `decidedAt` — the previous reviewer's note is erased when the agent submits a new proposal. The new version starts fresh.

## Who Can Request vs. Who Can Resolve

The permission matrix is worth being explicit about:

| Action | Board operator | Agent |
|---|---|---|
| Create (hire/strategy) approval request | Yes (direct hire also allowed) | Yes — via approval request |
| Approve or reject an approval | Yes | No |
| Request revision | Yes | No |
| Resubmit after revision request | No | Yes |
| Bypass approval gates | No | No |

Agents cannot bypass approval gates. The spec is explicit: agents "cannot bypass approval gates." A board operator can create agents directly without going through an approval — but even direct creation is logged as a governance action.

## Connecting the Pieces: A Hire with Budget Check

Let's walk the whole flow end-to-end so each step is concrete.

A software project needs a new engineer. An executive agent wants to hire one:

```ts
// Agent: submit a hire approval request
const hireRequest = await approvalService.create(workspaceId, {
  type: "hire_agent",
  requestedByAgentId: executiveAgentId,
  status: "pending",
  payload: {
    name: "Backend Engineer",
    role: "engineer",
    title: "Senior Backend Engineer",
    reportsTo: executiveAgentId,
    capabilities: "TypeScript, PostgreSQL, REST API design",
    adapterType: "process",
    adapterConfig: { command: "claude", args: ["--dangerouslySkipPermissions"] },
    budgetMonthlyCents: 5000, // $50.00/month requested
  },
});
// hireRequest.status === "pending"
```

A board operator reviews the request and approves:

```ts
// Board operator: approve the hire
const result = await approvalService.approve(hireRequest.id, boardUserId, "Looks good.");
// result.applied === true  (we were first)
// result.approval.status === "approved"
```

What happened inside `approve`:

1. `resolveApproval` ran the guarded UPDATE — our write won.
2. `applied === true`, so side effects run.
3. `agentsSvc.create(...)` created the agent with `status: "idle"`.
4. `budgets.upsertPolicy(...)` set a `calendar_month_utc` budget of 5000 cents for the new agent (because `budgetMonthlyCents > 0`).
5. `notifyHireApproved(...)` fired asynchronously.

Now suppose a second reviewer also clicked "Approve" a millisecond later:

```ts
const result2 = await approvalService.approve(hireRequest.id, secondUserId);
// result2.applied === false  (concurrent resolver won)
// result2.approval.status === "approved"  (shows the resolved state)
// No agent was created again. No second budget policy. No duplicate notification.
```

The race re-fetch saw `status === "approved"` matching the target — returned `applied: false` without error, without duplication.

## Try It Yourself

Here are three exercises to solidify the concepts from this chapter:

**1. Gate agent hire behind an approval.**
Modify the agent creation endpoint so it routes through an approval request when the workspace has `requireBoardApprovalForNewAgents: true`. An agent that tries to create a subordinate directly should get a 403 and find a `pending` approval in the queue instead.

**2. Simulate two concurrent reviewers.**
Write an integration test that fires two `approve` calls for the same approval ID in parallel (e.g., with `Promise.all`). Assert that exactly one returns `applied: true`, the other returns `applied: false`, and the agent was created exactly once.

**3. Require approval when a plan exceeds a budget threshold.**
Before creating a strategy approval, inspect the payload for total projected cost. If it exceeds the workspace's monthly budget cap, automatically set the approval's type to `budget_override_required` and add a note explaining the projected overage. The board reviewer then sees the budget context alongside the strategy.

---

← Previous: [Budgets and Cost Tracking](./budgets-and-cost-tracking.md) · Next: [Putting It All Together](../wrap-up/putting-it-all-together.md) →
