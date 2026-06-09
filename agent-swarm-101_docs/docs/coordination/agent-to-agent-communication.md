---
title: "Agent-to-Agent Communication"
description: How agents communicate by creating work — spawning sub-tasks via parentId, tracking delegation depth with requestDepth, and declaring blockers via task relations.
category: coordination
type: tutorial
tags:
  - sub-task
  - parent task
  - child task
  - parentId
  - request depth
  - requestDepth
  - blocker
  - blocked_by
  - blocks
  - relations
  - task relations
  - agent communication
  - delegation depth
  - sub-issue creation
  - task originator
  - issue relations
  - IssueRelation
  - taskRelations
  - MAX_ISSUE_REQUEST_DEPTH
  - Drizzle
  - PostgreSQL
  - agent delegation
  - squad leader
  - enqueue task
  - IsLeaderTask
  - task handoff
  - task lifecycle
keywords:
  - agent spawns task
  - parent child tasks
  - task dependency
  - blocking task
  - task nesting
  - delegation chain
  - agent work handoff
  - swarm coordination
  - agent orchestration
sources: [S4, S37, S20]
---

**TL;DR** — Agents in the Swarm do not call each other directly. When agent A needs agent B to do something, A creates a task and assigns it to B. This chapter walks through the three mechanics that make this work: the `parentId` self-reference that links child tasks to the parent that spawned them, the `requestDepth` counter that bounds how deep the delegation chain can grow, and the `taskRelations` table that lets any task declare itself blocked by another. By the end you will be able to model agent-to-agent work handoffs and express dependencies between tasks.

# Agent-to-Agent Communication

## There is no RPC between agents

When you first think about getting one agent to coordinate with another, the obvious instinct is to add a function call: the leader calls a method on the member, passes it the work, and waits for a result. That works fine when everything runs in a single process, but Swarm's agents run in separate runner processes — possibly on separate machines — and they communicate with the orchestrator over HTTP or WebSocket. There is no shared address space, no direct method call.

So how does agent A hand work to agent B?

It creates a task.

The leader does not call the member; it writes a new row into the task queue, sets `assigneeAgentId` to the member's ID, and returns. The member's runner picks up that task on its next claim cycle and executes it. The whole interaction is a data write followed by a data read, mediated by the orchestrator.

This pattern has a convenient side-effect: every handoff is durable. If the member's runner crashes mid-execution, the task is still in the database. The orchestrator can retry it, reassign it, or surface it as blocked — none of which would be possible with a direct call.

We saw a version of this in the [previous chapter on squads](./squads.md): a squad leader receives a task for the squad, decides which member should handle a piece of it, and creates a new task for that member. The mechanic behind that delegation is what we will now build up step by step.

## Step 1 — Linking child tasks to their parent with `parentId`

Let us start with the simplest case: a single agent decides that a task is too large to do in one pass and wants to break it into smaller pieces.

The task schema (in the Drizzle/PostgreSQL schema for the `issues` table, S37) carries a self-referencing foreign key:

```ts
// Simplified view of the issues table — showing the parentId relationship
import { pgTable, uuid, text, integer, timestamp } from "drizzle-orm/pg-core";
import type { AnyPgColumn } from "drizzle-orm/pg-core";

export const issues = pgTable("issues", {
  id:       uuid("id").primaryKey().defaultRandom(),
  title:    text("title").notNull(),
  status:   text("status").notNull().default("backlog"),
  priority: text("priority").notNull().default("medium"),

  // Self-reference: set this to make the task a child of another task
  parentId: uuid("parent_id").references((): AnyPgColumn => issues.id),

  // Who the work is assigned to
  assigneeAgentId: uuid("assignee_agent_id"),

  // How deep in the delegation chain this task sits (default 0 = top-level)
  requestDepth: integer("request_depth").notNull().default(0),

  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

The `: AnyPgColumn` type annotation on the column reference is how Drizzle handles circular references — a table cannot reference itself until TypeScript can see the full type, so the reference is wrapped in a thunk (`(): AnyPgColumn => issues.id`).

When an agent wants to break its current task into sub-tasks, it creates new tasks and sets `parentId` to the current task's `id`. From that point on, the hierarchy is queryable: you can find all children of a task by filtering `WHERE parent_id = $1`, and you can walk up the chain by following `parentId` until you reach a row where it is `NULL`.

### What the sub-task creation call looks like

In your orchestrator's issue-creation API handler, creating a sub-task means including `parentId` in the request body. The `buildSubIssueDefaults` helper (S37 / Drizzle companion code) shows what fields the system inherits automatically when a parent is set:

```ts
// Simplified view of buildSubIssueDefaults — what a child task copies from its parent
function buildSubIssueDefaults(parent: {
  id: string;
  identifier: string | null;
  title: string;
  projectId: string | null;
  assigneeAgentId: string | null;
}) {
  return {
    parentId: parent.id,           // the link that makes this a child
    parentIdentifier: parent.identifier ?? undefined,
    parentTitle: parent.title,
    ...(parent.projectId ? { projectId: parent.projectId } : {}),
    ...(parent.assigneeAgentId ? { assigneeAgentId: parent.assigneeAgentId } : {}),
  };
}
```

A sub-task inherits `projectId` from its parent at creation time (S20). It does **not** inherit `assigneeAgentId` automatically — the caller decides whether to assign the sub-task to the same agent, a different one, or leave it unassigned for the leader to route (see [Squads: A Leader That Delegates](./squads.md)).

> **Key takeaway.** The parent/child relationship is entirely expressed through data. The parent agent creates a task with `parentId` set and then continues (or completes) its own work. There is no blocking synchronous call.

## Step 2 — Bounding the chain with `requestDepth`

Sub-tasks can themselves have sub-tasks. A task at depth 0 spawns children at depth 1; those children might spawn grandchildren at depth 2. This is useful — a large project can decompose recursively. But left unchecked, an agent that recursively spawns sub-tasks that each spawn sub-tasks can run up a bill quickly and crash the system under load.

The `requestDepth` column on every task exists to track where in the chain a task sits and to bound how deep it can go.

### How it works

Every top-level task starts with `requestDepth = 0` (the column default in S37). When an agent creates a sub-task, it is the caller's responsibility to set `requestDepth` on the new task to `parentDepth + 1`.

The schema declares `requestDepth` as a non-negative integer with a default of 0:

```ts
requestDepth: integer("request_depth").notNull().default(0),
```

The validation layer clamps any submitted value to the configured maximum, ensuring a misbehaving agent cannot write an arbitrarily large depth:

```ts
// From the shared validators (S37 companion)
// Incoming requestDepth values are clamped — never accepted raw
const taskRequestDepthInputSchema = z
  .number()
  .int()
  .nonnegative()
  .transform((value) => clampIssueRequestDepth(value));
```

<!-- GAP: MAX_ISSUE_REQUEST_DEPTH exact value — the constant is defined in the shared package (found as 1024 in peripheral source) but is not stated in the assigned sources S4, S37, or S20; mark as implementation-specific -->

The precise cap is implementation-defined. What matters architecturally is:

1. Every task knows its depth.
2. Incoming values are clamped, so a runaway agent cannot inject an unbounded depth.
3. Your orchestrator can enforce a business-level limit *before* creating the sub-task — for example, refusing to create tasks beyond depth 5 for a particular project or agent tier.

### Applying the depth check before spawning

In practice, an agent's tool or CLI command for creating a sub-task should read the current task's `requestDepth` and add 1:

```ts
// Pseudocode for an agent tool that creates a sub-task
async function createSubTask(
  orchestratorClient: OrchestratorClient,
  currentTask: { id: string; requestDepth: number },
  subTaskTitle: string,
  assigneeAgentId: string,
) {
  const newDepth = currentTask.requestDepth + 1;

  // Let the validator/orchestrator enforce the system cap;
  // the agent can add its own earlier guard if needed.
  return orchestratorClient.createIssue({
    title: subTaskTitle,
    parentId: currentTask.id,
    assigneeAgentId,
    requestDepth: newDepth,
  });
}
```

When the orchestrator receives this request, the validation layer clamps `newDepth` if it exceeds the system maximum and rejects the creation entirely if the clamped value equals the current one — effectively blocking the creation when the chain is already at the limit.

> You might wonder: "Why not just count levels by walking the parent chain at query time?" Walking up the chain on every creation is a query-time cost proportional to depth; storing it flat makes the check a single integer comparison. The trade-off is that callers must maintain it accurately.

## Step 3 — Declaring blockers with task relations

Sub-tasks address one kind of inter-task coordination: decomposition. But sometimes two tasks at the *same* level have a dependency: task A cannot start until task B completes. That is not a parent/child relationship — it is a dependency edge.

The `task_relations` table stores these edges. Each row records that one task stands in a particular relationship to another:

```ts
// Simplified view of the taskRelations schema (S37)
export const taskRelations = pgTable(
  "task_relations",
  {
    id:             uuid("id").primaryKey().defaultRandom(),
    workspaceId:      uuid("workspace_id").notNull(),
    taskId:        uuid("task_id").notNull().references(() => issues.id, { onDelete: "cascade" }),
    relatedIssueId: uuid("related_task_id").notNull().references(() => issues.id, { onDelete: "cascade" }),
    type:           text("type").$type<"blocks">().notNull(),
    createdByAgentId: uuid("created_by_agent_id"),
    createdAt:      timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    // Ensure (company, taskId, relatedIssueId, type) is unique —
    // you cannot declare the same edge twice
    workspaceEdgeUq: uniqueIndex("task_relations_workspace_edge_uq").on(
      table.workspaceId,
      table.taskId,
      table.relatedIssueId,
      table.type,
    ),
  }),
);
```

The schema's `type` column currently stores `"blocks"` — meaning "the `taskId` task blocks the `relatedIssueId` task". The inverse (`blocked_by`) is the same edge read from the other direction.

### The four relation types (per S20)

The target data model (S20) defines four relation types that the system can express between tasks:

| Type | Meaning | Behavior |
|---|---|---|
| `blocks` | `taskId` blocks `relatedIssueId` | The blocked task shows a flag; it should not start |
| `blocked_by` | `taskId` is blocked by `relatedIssueId` | Inverse of `blocks`; same physical row read from the other side |
| `related` | General informational connection | No execution effect; visible in the UI |
| `duplicate` | `taskId` duplicates `relatedIssueId` | The duplicate task is auto-moved to a cancelled state |

> **Note on implementation vs. target.** The S37 schema (the implemented Drizzle table) defines the `type` column as `.$type<"blocks">()`, which constrains it to the string `"blocks"`. S20 is explicitly a *target model* ("some of this is already implemented, some is aspirational"). When building, confirm which relation types your deployed schema supports before using them in agent logic.

### Declaring a blocker

When agent A creates task T-B and knows that T-B cannot start until T-A is complete, it inserts a relation row:

```ts
// Agent A declares: "T-B is blocked by T-A"
await orchestratorClient.createIssueRelation({
  taskId: taskB.id,          // the task that is blocked
  relatedIssueId: taskA.id,   // the task doing the blocking
  type: "blocked_by",
});
```

Or, equivalently, from the blocker's perspective:

```ts
// Alternatively: "T-A blocks T-B"
await orchestratorClient.createIssueRelation({
  taskId: taskA.id,
  relatedIssueId: taskB.id,
  type: "blocks",
});
```

Both express the same dependency. The unique constraint on `(workspaceId, taskId, relatedIssueId, type)` prevents the same edge being declared twice.

### What happens to a blocked task

A task that carries a `blocked_by` relation is shown with a visual flag in the orchestrator UI (S20). In terms of execution: the runner should check whether the assigned task has any unresolved blockers before starting work. A blocked task should remain in the queue without being claimed until the blocking task completes.

According to S20, when the blocking issue is resolved (transitions to a completed state), the relation becomes informational — the flag turns green — and the previously blocked task is free to be claimed.

<!-- GAP: exact runtime enforcement of "blocked" status — S20 describes the flag behaviour and the resolved-blocker state change, but does not specify whether the claim step in the runner actively queries task_relations or relies on a status field; source silent on the mechanism -->

Note also that blocking is **not transitive** at the system level (S20): if T-A blocks T-B, and T-B blocks T-C, the system does not automatically infer that T-A blocks T-C. Each edge must be declared explicitly.

## Step 4 — The leader-task flag and the handoff pattern

We saw in [Squads: A Leader That Delegates](./squads.md) that a squad leader is triggered by an incoming task and then creates child tasks for its members. Let us look at the exact mechanic.

When the orchestrator enqueues a task for the squad's leader agent, it sets `is_leader_task = true` on the queue row (S4, the `AgentTaskQueue` model):

```ts
// From the AgentTaskQueue model (S4, translated to TS)
interface AgentTaskQueue {
  id: string;
  agentId: string;
  taskId: string | null;
  priority: number;
  isLeaderTask: boolean;    // true when this task was enqueued for a squad leader role
  parentTaskId: string | null; // set when this task was spawned by another task
  // ... other fields
}
```

The `isLeaderTask` flag serves a specific purpose: **self-trigger prevention**. An agent can be both the squad leader and a squad member. When such an agent posts a comment on an issue while acting in the *leader* role, the system needs to know not to immediately re-trigger the leader for that same comment (which would create an infinite loop). The `is_leader_task` field on the most recent task for that agent+issue pair is the signal the orchestrator checks (S4, `lastTaskWasLeader`).

The `parentTaskId` field on the queue row is the complementary piece: when the leader creates a child task for a member, that child's queue row carries the leader's task ID as `parentTaskId`. This makes the chain traceable at the task-queue level, independent of the `parentId` link on the issue itself.

### The handoff in sequence

Let's trace a complete delegation:

```
1. Human posts a comment on issue SWARM-42 (assigned to squad "Backend Squad")
2. Orchestrator: shouldEnqueueSquadLeaderOnComment → true
3. Orchestrator: enqueueSquadLeaderTask — creates AgentTaskQueue row
      agentId = leader.id
      taskId = SWARM-42.id
      isLeaderTask = true
4. Leader runner claims the task and evaluates it
5. Leader decides member Agent-B should handle a sub-piece
6. Leader calls the orchestrator API:
      POST /issues  { title: "...", parentId: SWARM-42.id, assigneeAgentId: agentB.id,
                      requestDepth: currentTask.requestDepth + 1 }
7. New issue SWARM-43 created with parentId = SWARM-42.id
8. Orchestrator enqueues task for Agent-B (isLeaderTask = false, parentTaskId = leader task id)
9. Agent-B's runner claims the task and executes
10. Agent-B completes → SWARM-43 status → "done"
11. (Parent SWARM-42 can react: if all children are done, the leader may close SWARM-42)
```

<!-- GAP: auto-close of parent when all children complete — S20 mentions "sub-issue auto-close: when parent completes, remaining sub-issues auto-complete" but does not specify a mechanism for the reverse (child completion triggering parent close); S4 and S37 are also silent on this -->

Steps 1–4 come directly from S4's `shouldEnqueueSquadLeaderOnComment` and `enqueueSquadLeaderTask`. Steps 6–10 come from the `parentId` and `requestDepth` mechanics in S37. Step 11 is partially grounded: S20 mentions sub-issue auto-close when the *parent* completes, but the reverse direction — a parent reacting to child completion — is not described in the assigned sources.

## Try it yourself

Here are three exercises to cement these mechanics.

**1. Spawn a sub-task from an agent.**

Write a small agent tool that, when invoked on an existing task, calls `POST /issues` with:
- `parentId` set to the current task's `id`
- `requestDepth` set to `currentTask.requestDepth + 1`
- `assigneeAgentId` set to a second agent's ID

Verify in the database that the new task's `parentId` column references the original task.

**2. Cap the delegation depth.**

Add a guard in the agent tool: if `currentTask.requestDepth >= MAX_DEPTH` (choose a small value like `3` for testing), return an error instead of creating the sub-task. Observe that a chain of three agents each spawning a child terminates at depth 3 and the fourth would-be child task is never created.

**3. Mark a task `blocked_by` another and watch it wait.**

Create two tasks, T-A and T-B. Insert a row into `task_relations` with `taskId = T-B.id`, `relatedIssueId = T-A.id`, `type = "blocks"`. Check T-B's detail view — the blocker flag should be visible. Now transition T-A to `done`. Verify that the blocker flag on T-B changes state (implementation-dependent per your runner's blocker-check logic).

---

← Previous: [Squads: A Leader That Delegates](./squads.md) · Next: [WebSockets I — The Runner Hub](../real-time/runner-hub.md) →
