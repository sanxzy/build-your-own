---
title: "Squads: A Leader That Delegates"
description: Build a squad — a named group with a leader agent who evaluates incoming tasks and delegates them to members, with anti-loop dedup and a readiness gate.
category: coordination
type: tutorial
tags:
  - squad
  - leader
  - delegation
  - auto-membership
  - dispatch
  - anti-loop suppression
  - dedup
  - squad routing
  - leader evaluation
  - AgentReadiness
  - archived_at
  - runner online
  - admission gate
  - create_issue mode
  - run_only mode
  - squad member
  - task queue
  - org chart
  - agent readiness gate
  - pending task dedup
  - self-trigger guard
  - squad leader evaluation outcome
  - action
  - no_action
  - failed
  - squad status
  - working
  - idle
  - offline
  - unstable
  - archived
keywords:
  - squad-based routing
  - leader delegates to member
  - automatic leader membership
  - agent readiness check
  - infinite delegation loop prevention
  - squad trigger anti-loop
  - squad autopilot
  - squad leader task dedup
  - squad leader comment trigger
  - squad member role
  - squad assignee
sources: [S4, S7, S8]
---

**TL;DR** — Assigning a task directly to a specific agent is fragile: that agent might be offline, overloaded, or eventually replaced. A *squad* solves this by giving you a named group with a *leader* who receives every incoming task and decides where it goes. By the end of this chapter you will have built a squad with auto-membership, a delegation loop with outcome logging, an anti-loop guard that stops the leader from chasing its own comments, and a readiness gate that prevents dispatching to an archived or offline agent.

# Squads: A Leader That Delegates

## Why direct assignment breaks down

Suppose you are building a system where incoming issues get routed to an agent. You pick "Agent-7" — it knows the domain, it is online right now, it is a good choice. Six weeks later, Agent-7 is archived and replaced by a more capable model. Every issue still assigned to Agent-7 stalls.

Even when nothing changes, a single assignee is a single point of failure: if that agent's runner goes offline, all work queues. If the agent is busy, new tasks pile up without any opportunity for load balancing.

A *squad* breaks this coupling. Instead of assigning a task to a specific agent, you assign it to a squad — a named group. The squad has a *leader*: one designated agent whose job is to evaluate incoming tasks and route them to the right member. The leader is the group's decision maker; the members do the work.

This chapter builds the squad machinery from scratch. We will cover:

1. The squad model and why the leader is automatically a member.
2. How delegation works end-to-end: task arrives → leader wakes up → leader decides → outcome is recorded.
3. The anti-loop guard that prevents the leader from triggering itself on its own comments.
4. The readiness gate — the single check that determines whether any agent can accept work right now.

### Quick recap: the org chart and the task queue

Before we dive in, two concepts from earlier chapters:

**The org chart** (covered in [Agents as a Team: The Org Chart](./org-chart.md)) gives every agent a `reportsTo` parent. It establishes hierarchy for permissions and visibility, but it does not do routing — that is squads' job.

**The task queue and claim loop** (covered in [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md)) is how work reaches an agent: a task is enqueued with an `agentId`, the agent's runner polls the queue, claims the task, and executes it. Squads sit *upstream* of that queue: the squad leader is the entity that decides which `agentId` to enqueue a task for.

---

## Step 1 — The squad model

Let's start with what a squad actually is. A squad has a name, a description, optional instructions for its leader, and most importantly, a `leaderId` pointing to one agent in the workspace.

```ts
// Simplified view of the Squad record
interface Squad {
  id: string;
  workspaceId: string;
  name: string;
  description: string;
  instructions: string;
  leaderId: string;       // the agent who routes work
  creatorId: string;
  createdAt: string;
  updatedAt: string;
  archivedAt: string | null;  // null = active; non-null = archived
  archivedBy: string | null;
}
```

Each squad also has a *member list*. Members are the agents (and optionally workspace users) the leader can delegate to. A `SquadMember` record links a squad to one participant:

```ts
interface SquadMember {
  id: string;
  squadId: string;
  memberType: "agent" | "member";  // "agent" = AI agent; "member" = human workspace member
  memberId: string;
  role: string;        // e.g. "leader", or a custom role string for other members
  createdAt: string;
}
```

### The leader is automatically a member

Here is the first important detail: **when you create a squad, the leader is immediately added as a member with the role `"leader"`**. You do not have to do this manually — the orchestrator does it for you.

```ts
// Pseudocode: what happens inside createSquad()
async function createSquad(params: {
  workspaceId: string;
  name: string;
  description: string;
  leaderId: string;
}): Promise<Squad> {
  // Verify the leader is a real agent in this workspace before proceeding.
  const leader = await db.getAgentInWorkspace({
    id: params.leaderId,
    workspaceId: params.workspaceId,
  });
  // Throws if not found — you cannot appoint an agent that does not exist
  // or belongs to a different workspace.

  const squad = await db.createSquad(params);

  // Auto-add the leader as a member with role "leader".
  await db.addSquadMember({
    squadId: squad.id,
    memberType: "agent",
    memberId: params.leaderId,
    role: "leader",
  });

  return squad;
}
```

Why auto-add? Because the leader *is* a member of the group — if you list all squad members you expect to see the leader there. This also ensures the leader can never be left out of its own squad accidentally.

The same auto-add logic applies when you **update** the leader field: if the new leader is not already in the member list, they are added automatically. Conversely, you cannot remove the current leader from the member list without first changing the leader:

```
Error: cannot remove the squad leader; change leader first
```

This constraint keeps the squad in a valid state: there is always exactly one leader agent, and that agent is always a member.

### The member status snapshot

When you need a live view of which members are available, the system derives a status for each member by combining runtime health and active-task signals. The five buckets are:

| Status | Meaning |
|---|---|
| `working` | Agent has at least one active (dispatched or running) task |
| `idle` | Agent's runner is `online` but has no active tasks |
| `unstable` | Runner went offline within the last 5 minutes — transient drop |
| `offline` | Runner has not been seen for more than 5 minutes |
| `archived` | Agent's `archived_at` is set — shown in lists but never dispatched to |

`archived` always wins: an agent with `archived_at` set reports `archived` regardless of any leftover runner row or task. The `working` bucket comes next, because an agent mid-task is occupied even if the runner briefly flaps.

---

## Step 2 — Delegation: a task arrives and the leader evaluates it

Now that we have a squad with members, let's trace what happens when a task is assigned to the squad.

### The trigger points

The leader is woken up in two situations:

1. **A task is assigned to the squad** (the issue moves out of the backlog, or is created pre-assigned).
2. **A new comment is posted on a task already assigned to the squad**.

In both cases the orchestrator runs a readiness check on the leader (more on that in Step 4), then enqueues a task for the leader — *not* for a member. The leader's job is to evaluate the situation and decide what to do next.

```ts
// Pseudocode: deciding whether to wake the leader on assignment
function shouldEnqueueLeaderOnAssign(issue: Issue): boolean {
  // Backlog issues are a parking lot — don't disturb the leader.
  if (issue.status === "backlog") return false;

  return isSquadLeaderReady(issue);
}
```

### The leader evaluation record

Once the leader finishes its turn, it reports back. The three possible outcomes are:

| Outcome | Meaning |
|---|---|
| `action` | The leader did something — delegated to a member, posted a comment, changed status, etc. |
| `no_action` | The leader reviewed the situation and decided nothing needed to be done. |
| `failed` | The leader attempted to act but encountered an error. |

The orchestrator records this outcome in the activity log as a `squad_leader_evaluated` event:

```ts
// Pseudocode: recording the leader's outcome
async function recordLeaderEvaluation(params: {
  taskId: string;
  taskId: string;
  outcome: "action" | "no_action" | "failed";
  reason: string;  // short plain-text explanation from the leader
}): Promise<ActivityEntry> {
  // Security: only the squad's designated leader agent may call this.
  // The orchestrator checks the caller's identity against squad.leaderId.

  const details = {
    squadId: squad.id,
    taskId: params.taskId,
    outcome: params.outcome,
    reason: params.reason,
  };

  return db.createActivity({
    workspaceId: issue.workspaceId,
    taskId: params.taskId,
    actorType: "agent",
    actorId: squad.leaderId,
    action: "squad_leader_evaluated",
    details: JSON.stringify(details),
  });
}
```

The outcome log is how you audit squad behaviour over time. If the leader keeps returning `no_action`, the squad's instructions may need tuning. If it returns `failed`, there is likely a delegation or tool error to investigate.

---

## Step 3 — Anti-loop suppression and dedup

Here is the problem we have created. The leader wakes up when a comment appears. The leader evaluates the situation and posts its own comment ("I am delegating this to Agent-B"). That comment is new activity on the issue — which would wake the leader *again*. And Agent-B's response would wake the leader *again*. We have an infinite delegation loop.

We need two guards: one to stop the leader from triggering itself on its own comments, and one to avoid queuing duplicate leader tasks.

### Guard 1: the self-trigger check

Before enqueueing a leader task in response to a comment, the orchestrator asks: *did the leader just post this comment while acting in its leader role?* If so, skip.

The key word is "while acting in its leader role." An agent can wear two hats: it might be both the squad leader and a delegated worker on the same issue. If it posts a comment while working as a *worker*, the leader role *should* still wake up (a worker update is a real signal that may need coordination). Only a comment posted *as the leader* should suppress the next leader trigger.

The orchestrator resolves this by looking at the agent's most recent task on the issue and checking whether that task was a leader task:

```ts
// Pseudocode: self-trigger guard
function shouldEnqueueLeaderOnComment(
  issue: Issue,
  commentContent: string,
  authorType: "agent" | "member",
  authorId: string,
): boolean {
  // If the author is the leader AND their last task on this issue was a leader task,
  // they are posting from the leader role — skip to prevent a self-trigger loop.
  if (
    authorType === "agent" &&
    authorId === squad.leaderId &&
    lastTaskWasLeader(issue.id, squad.leaderId)
  ) {
    return false;
  }

  // If a human member @mentioned someone explicitly, they are doing their own
  // routing — the leader would just observe and record no_action. Skip.
  // Note: issue cross-reference mentions (mention://issue/...) are NOT routing
  // signals and do not suppress the leader.
  if (authorType === "member" && commentMentionsAnyone(commentContent)) {
    return false;
  }

  // Finally, verify the leader agent is ready (not archived, runner online).
  return isSquadLeaderReady(issue);
}
```

Notice the `commentMentionsAnyone` check: when a human posts a comment that `@mentions` an agent, another member, a squad, or `@all`, they are directing work themselves. The leader waking up in that case would just observe and do nothing — wasted compute. We skip it.

What counts as a routing mention? Any `@Name` link to an agent, member, squad, or `@all`. Cross-references to *other issues* (`mention://issue/...`) are not routing signals; they are context links.

### Guard 2: dedup before enqueue

Even with the self-trigger guard, there is a race: two events arrive at nearly the same time (say, a status change and a comment), and both pass the trigger checks. We are about to enqueue two leader tasks for the same issue. One will do useful work; the other will find nothing left to do and record `no_action` at best — or create a confused second delegation at worst.

The fix is simple: before enqueuing a leader task, check whether there is already a pending task for this leader on this issue. If yes, skip.

```ts
// Pseudocode: dedup before enqueue
async function enqueueLeaderTask(issue: Issue): Promise<void> {
  // Gate: is there already a pending (not-yet-claimed) leader task?
  const hasPending = await db.hasPendingTaskForIssueAndAgent({
    taskId: issue.id,
    agentId: squad.leaderId,
  });
  if (hasPending) {
    return; // let the existing task handle it
  }

  await taskService.enqueueTaskForSquadLeader(issue, squad.leaderId);
}
```

Together, the self-trigger guard and the dedup check contain the loop:

```
Comment arrives
  └─ Is author the leader acting in leader role? → skip
  └─ Is author a member who @mentioned someone? → skip
  └─ Is there already a pending leader task? → skip
  └─ Is the leader ready? → proceed → enqueue leader task
```

---

## Step 4 — The readiness gate

We have referenced `isSquadLeaderReady` several times. Let's now build it properly.

The readiness gate answers one question: *can this agent accept new work right now?* The answer is `true` only when all three conditions hold:

1. `archived_at IS NULL` — the agent has not been archived.
2. The agent has a runner bound to it (`runtimeId IS NOT NULL`).
3. That runner's status is `"online"`.

```ts
// Pseudocode: AgentReadiness — the single readiness gate
//
// This is intentionally the only place this logic lives.
// The same function is called by:
//   - the squad-leader pre-enqueue check (comment + assign paths)
//   - the autopilot admission gate (before creating a run)
//   - the squad-leader runtime check inside run_only dispatch
//
// Keeping three code paths aligned matters: if one starts accepting
// "starting" runners while another does not, you get a bug that only
// surfaces when the same squad is triggered through two different entry
// points at once.
async function agentReadiness(
  agentId: string,
): Promise<{ ready: boolean; reason: string }> {
  const agent = await db.getAgent(agentId);

  if (agent.archivedAt !== null) {
    return { ready: false, reason: "agent is archived" };
  }
  if (!agent.runtimeId) {
    return { ready: false, reason: "agent has no runtime bound" };
  }

  const runtime = await db.getAgentRuntime(agent.runtimeId);

  if (runtime.status !== "online") {
    return { ready: false, reason: `agent runtime is ${runtime.status}` };
  }

  return { ready: true, reason: "" };
}
```

The reason string is machine-readable: the autopilot system stores it in `failure_reason` on a skipped run, and dashboards group skipped runs by this string to surface patterns ("squad leader agent has no runtime bound — did someone forget to start the daemon?").

### The gate is shared across all dispatch paths

You might wonder: why is this in one function instead of inline checks scattered across the codebase? Because there are three separate paths that all need the same answer:

| Path | When it runs |
|---|---|
| Squad comment trigger | Before enqueuing a leader task on comment |
| Squad assign trigger | Before enqueuing a leader task on issue assign |
| Autopilot admission gate | Before creating a run for any scheduled/triggered autopilot |

If each path had its own copy of the logic, they could drift. One might start permitting a runner in `"starting"` state while another stays strict. The bug would only appear when the same squad is triggered through two different code paths at the same time — hard to reproduce, easy to miss.

Centralising in `AgentReadiness` means touching the function moves all three paths together.

### Resolving the leader for squad autopilots

Schedules (autopilots) can also be assigned to a squad instead of a specific agent. When the schedule fires, the system resolves the leader and applies the same readiness gate before doing any work:

```ts
// Pseudocode: resolving who actually does the work for an autopilot
async function resolveAutopilotLeader(autopilot: Autopilot): Promise<Agent> {
  if (autopilot.assigneeType === "squad") {
    const squad = await db.getSquad(autopilot.assigneeId);
    if (squad.archivedAt !== null) {
      throw new Error("squad is archived");
    }
    // The squad's leader is the executing agent.
    return db.getAgent(squad.leaderId);
  }
  // Direct agent assignment — no indirection.
  return db.getAgent(autopilot.assigneeId);
}
```

An archived squad fails immediately with `"squad is archived"` rather than silently dispatching to nothing.

### Two execution modes for schedules

Schedules have two modes that affect what happens after the leader is resolved:

| Mode | What happens |
|---|---|
| `create_issue` | A new issue is created and assigned to the squad; the existing squad-assignment listener chain fires, waking the leader via the normal path |
| `run_only` | A task is enqueued directly for the leader, bypassing issue creation |

In `create_issue` mode, the issue is created with `assigneeType: "squad"` and `assigneeId: <squadId>`. The same event listener that handles a human manually assigning an issue to a squad then fires — no special squad-aware code is needed in the schedule dispatcher.

In `run_only` mode, `AgentReadiness` is checked a second time just before the task is created: even if the leader was ready at admission time, a few milliseconds later the runner could have gone offline. We fail closed rather than enqueue a doomed task.

---

## Step 5 — Putting it all together

Let's review the complete path a task takes when it is assigned to a squad:

```
Issue assigned to squad
  │
  ├─ status === "backlog"? → stop (parking lot, no action)
  │
  ├─ AgentReadiness(squad.leader)?
  │     ├─ archived → stop
  │     ├─ no runner → stop
  │     ├─ runner offline → stop
  │     └─ runner online → continue
  │
  ├─ hasPendingTask(issue, leader)? → stop (dedup)
  │
  └─ enqueueTaskForSquadLeader(issue, leader)
         │
         └─ leader's runner claims the task
               │
               └─ leader evaluates, then calls recordLeaderEvaluation:
                     outcome: "action" | "no_action" | "failed"
                     reason: "delegated to Agent-B" | "issue needs more info" | ...
```

And when a comment arrives:

```
Comment on squad-assigned issue
  │
  ├─ authorType === "agent" AND authorId === squad.leaderId
  │   AND lastTaskWasLeader(issue, leader)? → stop (self-trigger guard)
  │
  ├─ authorType === "member" AND commentMentionsAnyone(content)? → stop (human routing)
  │
  ├─ AgentReadiness(squad.leader)? → stop if not ready
  │
  ├─ hasPendingTask(issue, leader)? → stop (dedup)
  │
  └─ enqueueTaskForSquadLeader(issue, leader)
```

---

## Archiving a squad

When a squad is archived, two transfers happen before the archive flag is set:

1. **Issue reassignment.** All issues currently assigned to the squad are reassigned to the leader agent directly. The squad becomes absent as an assignee; the leader carries the work forward.

2. **Schedule reassignment.** All schedules (`autopilots`) pointing at the squad are rewritten to point at the leader agent directly. This prevents every future schedule tick from hitting `"squad is archived"` and recording a skipped run.

After archiving, `ArchivedAt` is set on the squad record. Any code path that tries to dispatch through the archived squad hits the `errSquadArchived` sentinel and records a skip rather than a failure — it is the expected state, not an error.

---

## Try it yourself

Here is a worked example you can follow using the HTTP API the book has built. We will:

1. Create a squad through the `POST /workspaces/:workspaceId/squads` endpoint and see the auto-membership take effect.
2. Add worker agents as members with `POST /workspaces/:workspaceId/squads/:id/members`.
3. Create an issue assigned to the squad and watch the delegation path fire.
4. Archive a member agent and verify the readiness gate blocks dispatch to it.
5. Read the leader's evaluation log from the activity feed.

Throughout these examples, replace `<workspace-id>`, `<leader-id>`, `<member-a-id>`, `<member-b-id>`, `<squad-id>`, and `<issue-id>` with the real UUIDs returned at each step.

### Step 1 — Create the squad

`POST /workspaces/:workspaceId/squads` accepts `name`, `description`, and `leader_id`. The handler (the `createSquad` logic we traced in Step 1 above) inserts the squad row and then immediately inserts a `SquadMember` row for the leader with `role: "leader"` — no second request needed.

```bash
curl -X POST https://localhost:8080/workspaces/<workspace-id>/squads \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Triage Squad",
    "description": "Routes incoming support issues",
    "leader_id": "<leader-id>"
  }'
```

The response includes the full squad record. The `member_count` field will already be `1` and `member_preview` will show the leader — proof that auto-membership ran.

```json
{
  "id": "<squad-id>",
  "name": "Triage Squad",
  "leader_id": "<leader-id>",
  "member_count": 1,
  "member_preview": [
    { "member_type": "agent", "member_id": "<leader-id>", "role": "leader" }
  ]
}
```

### Step 2 — Add worker members

`POST /workspaces/:workspaceId/squads/:id/members` accepts `member_type`, `member_id`, and `role`. Add the two worker agents:

```bash
# Add Member-A
curl -X POST https://localhost:8080/workspaces/<workspace-id>/squads/<squad-id>/members \
  -H "Content-Type: application/json" \
  -d '{ "member_type": "agent", "member_id": "<member-a-id>", "role": "worker" }'

# Add Member-B
curl -X POST https://localhost:8080/workspaces/<workspace-id>/squads/<squad-id>/members \
  -H "Content-Type: application/json" \
  -d '{ "member_type": "agent", "member_id": "<member-b-id>", "role": "worker" }'
```

To see all three members together, call `GET /workspaces/:workspaceId/squads/:id/members`. You should see three entries: the leader (added automatically) plus the two workers.

### Step 3 — Assign an issue to the squad and observe delegation

Create an issue in the workspace and set `assignee_type: "squad"` with `assignee_id` pointing at the squad. The issue-assign listener we traced in Step 2 runs immediately: it calls `shouldEnqueueSquadLeaderOnAssign`, checks `AgentReadiness` for the leader, and — if the leader's runner is online — inserts a leader task into the task queue.

```bash
curl -X POST https://localhost:8080/workspaces/<workspace-id>/issues \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Investigate login failure",
    "status": "todo",
    "assignee_type": "squad",
    "assignee_id": "<squad-id>"
  }'
```

What to watch for: the leader agent's runner should claim the task shortly after the issue is created. Once the leader has evaluated the situation and called `RecordSquadLeaderEvaluation` (Step 5 below tells you how to verify this), a `squad_leader_evaluated` activity entry will appear on the issue.

> If the leader's runner is not online, `shouldEnqueueSquadLeaderOnAssign` returns `false` and no task is created. Bring the leader's daemon online first, then reassign the issue to trigger the path again.

### Step 4 — Archive a member agent and verify the readiness gate

Archiving an agent sets `archived_at` on its row. Call the archive endpoint for Member-A:

```bash
curl -X DELETE https://localhost:8080/workspaces/<workspace-id>/agents/<member-a-id>
```

Now check the live member status for the squad:

```bash
curl https://localhost:8080/workspaces/<workspace-id>/squads/<squad-id>/members/status
```

Member-A should appear in the response with `"status": "archived"` — the `deriveSquadMemberStatus` function we saw in Step 1 evaluates `archived_at` first and returns `"archived"` unconditionally, regardless of any leftover runtime row.

```json
{
  "members": [
    { "member_type": "agent", "member_id": "<leader-id>",   "status": "idle" },
    { "member_type": "agent", "member_id": "<member-a-id>", "status": "archived" },
    { "member_type": "agent", "member_id": "<member-b-id>", "status": "idle" }
  ]
}
```

On the next leader evaluation, `AgentReadiness` will return `ready: false, reason: "agent is archived"` for Member-A. The leader should route any new delegation to Member-B instead — inspect the delegation comment it posts to confirm.

### Step 5 — Read the leader evaluation log

Once the leader agent has finished evaluating and called `RecordSquadLeaderEvaluation`, the activity log for the issue will contain a `squad_leader_evaluated` entry. Fetch the activity feed for the issue:

```bash
curl https://localhost:8080/workspaces/<workspace-id>/issues/<issue-id>/activity
```

Look for an entry with `"action": "squad_leader_evaluated"` in the response. The `details` field will contain the outcome and the leader's plain-text reason:

```json
{
  "action": "squad_leader_evaluated",
  "actor_type": "agent",
  "actor_id": "<leader-id>",
  "details": {
    "squad_id": "<squad-id>",
    "task_id": "<task-id>",
    "outcome": "action",
    "reason": "delegated to Member-B: domain expertise match"
  }
}
```

If `outcome` is `no_action`, the leader found nothing to do on this pass. If it is `failed`, the leader hit an error — the `reason` field will say more. Both are normal operating states; the log is how you tune the leader's instructions over time.

> **Wrapping this in a CLI.** Everything above is a plain HTTP call. If you want a `swarm squad create` shorthand, you could write a small CLI that wraps these endpoints — but the logic lives in the server, not in a CLI binary, so the exercises above reflect what the system actually does.

---

## Summary

A squad gives you a named group with a designated leader agent who owns every routing decision. Here is what we built:

- **Auto-membership:** the leader is always in the member list; removing or replacing the leader keeps the member list consistent.
- **Delegation loop:** task arrives → readiness check → dedup check → leader task enqueued → leader evaluates → outcome recorded.
- **Anti-loop suppression:** the leader does not re-trigger on comments it posted *while acting as leader*; human-routed comments (with `@mentions`) are also skipped.
- **Dedup:** if a pending leader task already exists for the issue, no second task is created.
- **`AgentReadiness`:** the single readiness gate — `archived_at IS NULL`, runner bound, runner `online` — shared by squad dispatch, assignment triggers, and schedule admission.

---

← Previous: [Agents as a Team: The Org Chart](./org-chart.md) · Next: [Agent-to-Agent Communication](./agent-to-agent-communication.md) →
