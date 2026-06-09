---
title: "Putting It All Together"
description: "Wire every part of the Swarm — enqueue a task, watch squad delegation, claim, execute via the LLM adapter, track cost, and stream results to the live board."
category: wrap-up
type: tutorial
tags:
  [
    capstone,
    end-to-end,
    multi-agent,
    squad delegation,
    enqueue,
    claim loop,
    LLM adapter,
    mock adapter,
    live board,
    WebSocket,
    budget,
    cost tracking,
    schedule,
    governance,
    synthesis,
    swarm run,
    runner hub,
    task queue,
    org chart,
    squads,
    adapter registry,
    crash recovery,
    approval gate,
    atomic checkout,
    heartbeat,
    cost events,
    PostgreSQL,
    Drizzle,
    ws,
  ]
keywords:
  [
    orchestrator boot,
    full swarm run,
    end-to-end agent tutorial,
    agent task lifecycle,
    task claim,
    squad leader delegate,
    LLM invocation,
    token usage,
    spend cap,
    live event stream,
    runner daemon,
    scheduler trigger,
  ]
sources: [S1, S9, S10, S18, S34, S38, S41]
---

**TL;DR** — Every chapter in this book built one piece of the Swarm: the adapter, the queue, the org chart, the hub, the live board, budgets, and governance. In this final chapter we trace one complete run from start to finish — a scheduled "summarise new issues" task that the squad leader delegates to a worker, the runner claims over WebSocket, executes through the LLM adapter, has its cost recorded and budget checked, and whose progress streams live to the board. By the end you will see how every module fits together as a working system.

# Putting It All Together

You built the pieces separately, chapter by chapter. Now let's watch them work together.

Here is the scenario we will trace:

> A daily schedule fires at 09:00. It creates a "Summarise new issues" task and assigns it to the **Research squad**. The squad leader reads the task and delegates it to a member agent. The runner (a background process on your machine) is woken over the runner hub, atomically claims the task, and executes it through the LLM adapter. The run records token usage and a cost event; the budget is checked. Every state change streams live to the board over a WebSocket.

We will work through seven steps that correspond directly to the seven parts of the book. At each step we will show the wiring code (using `@swarm/*` modules you built), describe what happens at runtime, and note which earlier chapter owns that piece.

---

## Before we start: a one-minute recap of every module

Rather than re-teach any chapter in depth, here is a one-line summary and a link for each:

| Module | What it does | Chapter |
| --- | --- | --- |
| An **agent** | An AI worker with an adapter and config; belongs to the org chart | [What Is an Agent Swarm?](../getting-started/what-is-a-swarm.md) |
| **First agent / mock adapter** | A runnable agent that responds with scripted text — no LLM key needed | [Your First Agent](../getting-started/your-first-agent.md) |
| **Adapter interface** | The `invoke / status / cancel` contract every adapter implements | [The Adapter Interface](../the-agent/adapter-interface.md) |
| **Mock adapter** | Implements the interface without calling any LLM | [The Mock Adapter](../the-agent/mock-adapter.md) |
| **LLM adapter** | Implements the interface by calling Claude or another model via API | [The LLM Adapter](../the-agent/llm-adapter.md) |
| **Adapter registry** | Selects the right adapter at claim time by adapter type string | [The Adapter Registry](../the-agent/adapter-registry.md) |
| **Task model** | The `task` row, its status machine, and priority field | [Modeling Tasks](../tasks-and-queue/modeling-tasks.md) |
| **Claim loop** | Atomic PostgreSQL `UPDATE … WHERE status = 'queued'` + wakeup channel | [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md) |
| **Crash recovery** | Sweeper reclaims dispatched tasks whose lease has expired | [Crash Recovery and Liveness](../tasks-and-queue/crash-recovery-and-liveness.md) |
| **Org chart** | `reportsTo` self-FK that places every agent in a tree | [The Org Chart](../coordination/org-chart.md) |
| **Squads** | A named group with a leader agent who delegates work to members | [Squads](../coordination/squads.md) |
| **Agent-to-agent comms** | Leader creates a sub-task assigned to a member; sub-task is enqueued and claimed independently | [Agent-to-Agent Communication](../coordination/agent-to-agent-communication.md) |
| **Runner hub (WS I)** | The WebSocket hub the orchestrator uses to push wakeup hints to daemons | [WebSockets I — The Runner Hub](../real-time/runner-hub.md) |
| **Live board (WS II)** | The WebSocket the browser subscribes to for live task/agent events | [WebSockets II — The Live Board](../real-time/live-board.md) |
| **Scheduler loop** | Reads cron schedules from the DB and creates tasks when they fire | [The Scheduler Loop](../scheduling/scheduler-loop.md) |
| **Budgets and cost tracking** | `cost_events` table + monthly rollup + hard-pause enforcement | [Budgets and Cost Tracking](../governance/budgets-and-cost-tracking.md) |
| **Approvals and governance** | An `approvals` row that pauses a task branch until the board decides | [Approvals and Governance Gates](../governance/approvals-and-governance-gates.md) |

Keep this table handy as we move through the steps.

---

## Step 1 — Boot the orchestrator and start the runner

The problem: all the modules we built live in separate files. We need one entry point that wires them together and starts listening, and we need the runner process to connect to it.

### 1a — The orchestrator process

The orchestrator process owns four long-running concerns:

1. A PostgreSQL connection (the DB underpins every module).
2. The **runner hub** — a WebSocket server daemons connect to for wakeup hints (built in [the runner hub chapter](../real-time/runner-hub.md)).
3. The **live events server** — a second WebSocket endpoint the board browser subscribes to (built in [the live board chapter](../real-time/live-board.md)).
4. The **scheduler loop** — a background loop that fires tasks from cron schedules (built in [the scheduler loop chapter](../scheduling/scheduler-loop.md)).

Here is the wiring:

```ts
// src/orchestrator/main.ts — simplified view of the boot sequence

import { createDb } from "@swarm/db";
import { TaskService } from "@swarm/core/task-service";
import { Hub } from "@swarm/realtime/hub";
import { EventBus } from "@swarm/events/bus";
import { setupLiveEventsWebSocketServer } from "@swarm/realtime/live-events-ws";
import { SchedulerLoop } from "@swarm/scheduling/scheduler-loop";
import { AdapterRegistry } from "@swarm/adapters/registry";
import { BudgetEnforcer } from "@swarm/governance/budget-enforcer";

async function main() {
  const db = createDb(process.env.DATABASE_URL);

  // --- runner hub (WebSocket, daemons connect here) ---
  // The hub keeps a map: runnerID → Set<client>.
  // When a task is enqueued, notifyTaskAvailable() pushes a
  // "runner:task_available" frame to any connected daemon that
  // owns that runtime.  (S10 — hub.go)
  const hub = new Hub();
  const bus = new EventBus();

  // Wire the event bus to the hub: when the task service publishes
  // "task:available", the hub pushes a wakeup frame to the runner.
  bus.subscribe("task:available", (e) => {
    if (e.runnerID) hub.notifyTaskAvailable(e.runnerID, e.taskID ?? "");
  });

  const taskService = new TaskService({ db, hub, bus });

  // --- live events WebSocket (browser board subscribes here) ---
  //   Path: /api/workspaces/:workspaceId/events/ws
  //   Auth: bearer agent API key OR session cookie.
  //   On connection: subscribe to company-scoped live events.
  //   (S34 — live-events-ws.ts)
  setupLiveEventsWebSocketServer(httpServer, db, {
    deploymentMode: process.env.DEPLOYMENT_MODE ?? "local_trusted",
  });

  // --- scheduler loop ---
  //   Reads schedules from the DB; on each cron tick, creates a task
  //   and calls taskService.enqueue().  (S12 — scheduler-loop chapter)
  const scheduler = new SchedulerLoop({ db, taskService });
  scheduler.start();

  // --- budget enforcer ---
  //   Checked on every cost event; at 100 % it pauses the agent.  (S18)
  const budgetEnforcer = new BudgetEnforcer({ db });

  httpServer.listen(8080, () =>
    console.log("orchestrator listening on :8080")
  );
}
```

Notice that the wiring is mechanical: we create the DB, hand it to the services that need it, and start the loops. No module reaches outside its own layer.

### 1b — Starting the runner

The runner is a separate process — the one the book built in [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md) and extended with WebSocket support in [WebSockets I — The Runner Hub](../real-time/runner-hub.md). It connects to the hub's WebSocket endpoint and enters the claim loop.

**Recall from the runner hub chapter:** the runner calls `hub.handleWebSocket(req, res, identity)`, where `identity` carries the runner's `runnerIDs` — the runtime identifiers it owns. After the upgrade, the hub registers the connection in its `byRunner` map so future wakeup frames can reach it.

In practice you start the runner as a separate Node.js process pointing at the orchestrator:

```ts
// src/runner/main.ts — simplified view of runner startup
import { startRunner } from "@swarm/runner";

startRunner({
  serverUrl:  "ws://localhost:8080/runner/ws",
  runnerIds:  [process.env.RUNNER_ID ?? "rt_abc123"],
  token:      process.env.SWARM_RUNNER_TOKEN,
});
```

`startRunner` opens the WebSocket connection — using the `ws` library, the same one the hub uses — then enters the dual-path loop: it listens for `runner:task_available` push frames and also fires the claim loop on a periodic tick as a backstop (described in [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md)).

Once connected, the runner appears in `hub.byRunner["rt_abc123"]`. Any wakeup the server sends for that runner ID lands in the runner's WebSocket read loop.

**The dual-path design.** Push hint plus HTTP claim is the design established in [the runner hub chapter](../real-time/runner-hub.md): the push eliminates latency on the hot path, but the periodic poll remains the correctness backstop. The runner does not need to change its claim logic — the wakeup just tells it "now is a good time."

---

## Step 2 — The schedule fires and a task is enqueued

The problem: we want the "Summarise new issues" work to happen every morning at 09:00 without a human pressing a button.

The **scheduler loop** (built in [the scheduler loop chapter](../scheduling/scheduler-loop.md)) reads a `schedules` table whose rows look like:

```ts
{
  id: "sched_daily_summary",
  cronExpr: "0 9 * * *",   // 09:00 every day
  agentId: "agent_research_squad",   // assigned to the squad
  title:   "Summarise new issues",
  enabled: true,
}
```

When the cron expression fires the scheduler calls `taskService.enqueue()` with the issue title and the squad as assignee. Under the hood, `EnqueueTaskForIssue` (S9) performs these actions in order:

1. Validates the agent is not archived and has a runtime.
2. Calls `Queries.CreateAgentTask(...)` to insert the new `queued` row into `agent_task_queue`.
3. Broadcasts a `task:queued` event to all live-board subscribers for this workspace.
4. Calls `NotifyTaskEnqueued()`, which calls `hub.notifyTaskAvailable(runtimeID, taskID)` to push a wakeup frame to the runner daemon immediately — rather than waiting for the next periodic poll cycle.

```ts
// Simplified view of the broadcast-then-notify ordering (S9 — task.go)
taskService.broadcastTaskEvent(ctx, EventTaskQueued, task);   // live board first
taskService.notifyTaskEnqueued(ctx, task);                    // runner hub second
```

The ordering matters: the live board sees `task:queued` before the daemon can claim and transition the task to `dispatched`, so board observers always see the queued state before the dispatched state. The source comments in S9 document this invariant explicitly ("Order matters: broadcast first, notify daemon second").

At this point the task row looks like:

```
status: "queued"
agent_id: <research-squad-leader>
runtime_id: rt_abc123
priority: 2   (medium)
```

---

## Step 3 — The squad leader delegates to a member

The problem: the "Research squad" is a group — not a single agent. Work assigned to a squad is received by the leader, who then routes it to the right member. We built this in [the squads chapter](../coordination/squads.md) and [the agent-to-agent communication chapter](../coordination/agent-to-agent-communication.md).

A squad has:
- A **leader agent** — the agent whose `id` is stored as `squad.leaderId`.
- **Member agents** — agents who belong to the squad and can receive sub-tasks.

**Recall from the agent-to-agent communication chapter:** agents do not call each other directly. When the leader wants to hand work to a member, it calls the orchestrator API to create a new task (`POST /issues` with `parentId` set to the current task's `id` and `assigneeAgentId` set to the member). The new task enters the queue independently and the member's runner picks it up on its next claim cycle.

When the scheduler enqueues the "Summarise new issues" task, it is addressed to the squad's leader (because the task creation sets `isLeaderTask = true` on the task row). The leader agent wakes, reads the task, and decides which member is best suited. It communicates that decision by creating a child task assigned to the member:

```ts
// Leader creates a sub-task for the member agent
// (the mechanic built in the agent-to-agent communication chapter)
await orchestratorClient.createIssue({
  title:           "Summarise all issues opened in the last 24 hours and post a digest",
  parentId:        currentTask.taskId,           // link to the parent task
  assigneeAgentId: "analyst-1",                   // route to the member
  requestDepth:    currentTask.requestDepth + 1,  // increment the delegation depth counter
});
```

The orchestrator receives this and calls `EnqueueTaskForMention` (S9) to enqueue the member's task:

```ts
// Simplified view of EnqueueTaskForMention (S9 — task.go, translated to TS)
async function enqueueTaskForMention(
  issue: Issue,
  agentId: string,
  triggerCommentId: string,
): Promise<AgentTaskQueue> {
  return enqueueMentionTask(issue, agentId, triggerCommentId, /* isLeader */ false, /* forceFreshSession */ false);
}
```

This inserts a second task row — same issue, different `agent_id` — with `isLeaderTask = false`. The task for `analyst-1` enters the queue with status `queued` and its own `triggerCommentId` pointing at the parent task. The runner hub is notified again.

Now there are two tasks for this issue: the leader's (which it has already started and will complete once it delegates) and the worker's (freshly queued).

---

## Step 4 — The runner is woken, claims the task, and executes it

This step has three sub-parts: wakeup, atomic claim, and execution.

### 4a — Wakeup over the runner hub

The hub pushed a `runner:task_available` frame the moment `analyst-1`'s task was enqueued in Step 3. Inside the hub (built in [the runner hub chapter](../real-time/runner-hub.md)), `notifyTaskAvailable` looks up `byRunner[runtimeID]`, builds a small JSON frame, and enqueues it to each matching client's send buffer:

```ts
// Simplified view of the push path (S10 — hub.ts)
notifyTaskAvailable(runnerID: string, taskID: string): void {
  if (!runnerID) return;

  const frame = JSON.stringify({
    type: "runner:task_available",
    payload: { runnerID, taskID },
  });

  this.notifyFrame(runnerID, Buffer.from(frame), /* eventID */ taskID);
}
```

The frame carries the `runnerID` and `taskID`. The runner's read pump — inside the `ws` WebSocket connection — receives this frame and immediately calls the claim endpoint rather than waiting for the next scheduled poll.

The comment in S10 is clear about the design intent: "Messages are best-effort wakeup hints; the daemon still uses HTTP claim for correctness." The push just eliminates the polling latency.

### 4b — Atomic claim

The runner calls the task service's claim function (S9). The function:

1. **Reclaim check**: first calls `ReclaimStaleDispatchedTaskForRuntime` — if a task was dispatched but the previous runner crashed before starting it, this reclaims it rather than leaving it stuck. This is the crash-recovery mechanism built in [the crash recovery chapter](../tasks-and-queue/crash-recovery-and-liveness.md).
2. **Empty-claim fast path**: checks the `EmptyClaimCache` — if a recent check already confirmed no queued tasks for this runtime, return immediately without hitting PostgreSQL.
3. **List and claim loop**: fetches `queued` candidate tasks for the runtime, then for each unique agent calls `ClaimTask` — which internally calls `Queries.ClaimAgentTask(ctx, agentID)`, an `UPDATE … WHERE status = 'queued' … RETURNING` that atomically moves the row to `dispatched`.

```ts
// Simplified view — ClaimTask (S9 — task.go, translated to TS)
async function claimTask(agentId: string): Promise<AgentTaskQueue | null> {
  // Check agent capacity
  const running = await queries.countRunningTasks(agentId);
  if (running >= agent.maxConcurrentTasks) {
    return null; // no capacity
  }

  // Atomic claim: UPDATE … WHERE status = 'queued' RETURNING …
  const task = await queries.claimAgentTask(agentId);
  if (!task) return null; // nothing available (ErrNoRows equivalent)

  taskService.captureTaskDispatched(task);
  taskService.reconcileAgentStatus(agentId);  // marks agent "working"
  taskService.broadcastTaskDispatch(task);    // pushes task:dispatch to live board
  return task;
}
```

If two runners race for the same task, only one `RETURNING` row comes back — the loser gets no row and moves on. Concurrency is handled entirely in the database.

After a successful claim the task row is:

```
status: "dispatched"
agent_id: <analyst-1>
dispatched_at: <now>
```

The runner then calls `POST /tasks/:id/start` which transitions the row to `running` and broadcasts `task:running` to the live board.

### 4c — Execution via the adapter registry

Now the runner needs to actually run the agent. It asks the **adapter registry** (built in [the adapter registry chapter](../the-agent/adapter-registry.md)) which adapter to use:

```ts
// Simplified view — src/runner/executor.ts
import { adapterRegistry } from "@swarm/adapters/registry";

const task = await claimTask(runnerId);
if (!task) return;

const adapter = adapterRegistry.get(task.agentAdapterType);
// Returns the LLM adapter if adapterType = "claude-cli",
// the mock adapter if adapterType = "mock", etc.

const result: AdapterResult = await adapter.invoke(agent, {
  runId:       task.id,
  prompt:      buildPrompt(task),
  environment: task.env,
});
```

**Recall from the adapter interface chapter:** every adapter — mock or LLM — satisfies the same `SwarmAdapter` contract:

```ts
interface SwarmAdapter {
  invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;
}
```

`AdapterResult` is the unified result shape that every adapter returns from `invoke`. It includes the output text and, for real LLM adapters, token counts that the runner will report as a cost event.

**For a real LLM run** the adapter type is `"claude-cli"` and the LLM adapter (built in [the LLM adapter chapter](../the-agent/llm-adapter.md)) spawns the `claude` CLI, passes it the prompt, streams stdout, and returns the output plus token usage wrapped in `AdapterResult`.

**For a keyless practice run** use `"mock"` — the mock adapter (built in [the mock adapter chapter](../the-agent/mock-adapter.md)) returns a scripted response immediately without any network calls. This is how you can run the complete loop without an `ANTHROPIC_API_KEY`.

The runtime operating patterns from S41 describe four wakeup sources — `timer`, `assignment`, `on_demand`, and `automation` — and note that if an agent is already running, new wakeups are coalesced rather than launching duplicate runs.

---

## Step 5 — Cost event recorded and budget checked

The problem: the run consumed tokens. We need to record the spend and check whether the agent has exceeded its monthly budget. This is the system built in [the budgets and cost tracking chapter](../governance/budgets-and-cost-tracking.md).

When the LLM adapter returns, the runner reports token usage to the orchestrator:

```ts
// Simplified view — src/runner/executor.ts (after invoke returns)
await reportCostEvent({
  agentId:      agent.id,
  workspaceId:  task.workspaceId,
  taskId:       task.id,
  provider:     "anthropic",
  model:        "claude-sonnet-4-5",
  inputTokens:  result.usage.inputTokens,
  outputTokens: result.usage.outputTokens,
  costCents:    result.usage.costCents,
  occurredAt:   new Date().toISOString(),
});
```

The cost event lands in the `cost_events` table. The schema (S38) captures:

```ts
// Simplified from packages/db/src/schema/cost_events.ts (S38)
export const costEvents = pgTable("cost_events", {
  id:          uuid("id").primaryKey().defaultRandom(),
  workspaceId:   uuid("workspace_id").notNull(),
  agentId:     uuid("agent_id").notNull(),
  taskId:     uuid("task_id"),
  projectId:   uuid("project_id"),
  goalId:      uuid("goal_id"),
  heartbeatRunId: uuid("heartbeat_run_id"),
  billingCode: text("billing_code"),
  provider:    text("provider").notNull(),
  biller:      text("biller").notNull().default("unknown"),
  billingType: text("billing_type").notNull().default("unknown"),
  model:       text("model").notNull(),
  inputTokens: integer("input_tokens").notNull().default(0),
  cachedInputTokens: integer("cached_input_tokens").notNull().default(0),
  outputTokens: integer("output_tokens").notNull().default(0),
  costCents:   integer("cost_cents").notNull(),
  occurredAt:  timestamp("occurred_at", { withTimezone: true }).notNull(),
});
```

Three composite indexes (S38) keep rollup queries fast:

| Index | Purpose |
| --- | --- |
| `(workspaceId, occurredAt)` | Company-wide spend for the current month |
| `(workspaceId, agentId, occurredAt)` | Per-agent monthly spend |
| `(workspaceId, provider, occurredAt)` | Spend breakdown by LLM provider |

After inserting the cost event the budget enforcer calls `evaluateCostEvent`. **Recall from the [budgets and cost tracking chapter](../governance/budgets-and-cost-tracking.md):** the system does not compare two agent-row columns directly. Instead it aggregates all `cost_events` for the relevant scope over the current UTC calendar-month window (`SUM(costCents) WHERE agentId = … AND occurredAt >= monthStart`), then passes that total to `budgetStatusFromObserved`, which returns one of three statuses:

| Status | Meaning |
|---|---|
| `ok` | Spend is below the soft-alert threshold |
| `warning` | Spend has crossed the configurable `warnPercent` (default 80 %) of the cap; a `BudgetIncident` with `thresholdType: "soft"` is created; work continues |
| `hard_stop` | Spend has reached the cap; `pauseAndCancelScopeForBudget` fires |

```ts
// Simplified view — evaluateCostEvent called after every cost event insert (S18)
const observedAmount = await computeObservedAmount(db, {
  workspaceId: event.workspaceId,
  scopeType:   "agent",
  scopeId:     event.agentId,
  windowKind:  "calendar_month_utc",
  metric:      "billed_cents",
});

const status = budgetStatusFromObserved(
  observedAmount,
  policy.amount,     // cap in cents
  policy.warnPercent // default 80
);

if (status === "warning") {
  await createIncidentIfNeeded(policy, "soft", observedAmount);
  // Work continues — this is a notification, not a stop.
}

if (status === "hard_stop") {
  await createIncidentIfNeeded(policy, "hard", observedAmount);
  await pauseAndCancelScopeForBudget(policy);
  // pauseAndCancelScopeForBudget writes agents.status = "paused",
  // agents.pauseReason = "budget", then fires the cancelWorkForScope
  // hook to terminate any in-flight adapter runs.
}
```

At `hard_stop`, two things happen: the agent row is updated to `status = "paused", pauseReason = "budget"`, and the `cancelWorkForScope` hook signals any in-flight adapter runs to stop. The scheduler's `getInvocationBlock` check then prevents new runs from starting for the paused agent.

The board operator resumes the agent by raising the policy's `amount` above the current observed spend — the budget service validates the new cap exceeds what was already spent, clears `pauseReason`, and the agent becomes available for the next scheduler tick (S18).

---

## Step 6 — Progress and completion stream to the live board

The problem: the board operator wants to see the run progress without polling. Every state change we made in Steps 2–5 already published events to the live events bus. Let's trace exactly what the browser receives.

The live events WebSocket server (S34) handles the `/api/workspaces/:workspaceId/events/ws` endpoint. On connection it calls `subscribeWorkspaceLiveEvents(workspaceId, callback)` — a function (built in [the live board chapter](../real-time/live-board.md)) that registers the socket for all events scoped to that workspace. Every event emitted anywhere in the orchestrator — via `broadcastTaskEvent`, `publishAgentStatus`, the budget enforcer's activity event — routes through this bus.

```ts
// Simplified from server/src/realtime/live-events-ws.ts (S34)
wss.on("connection", (socket, req) => {
  const { workspaceId } = req.upgradeContext;

  // Subscribe: every company-scoped event pushes to this socket.
  const unsubscribe = subscribeWorkspaceLiveEvents(workspaceId, (event) => {
    if (socket.readyState !== WebSocket.OPEN) return;
    socket.send(JSON.stringify(event));
  });

  socket.on("close", () => unsubscribe());
});
```

The browser connects with a bearer token (agent API key) or a session cookie. The auth path (S34, `authorizeUpgrade`) supports both. In `local_trusted` mode no token is required.

For our "Summarise new issues" run, the board subscriber receives this stream of events in order:

```
task:queued        — scheduler fires, task row created for squad leader
task:queued        — leader creates sub-task for analyst-1
task:dispatch      — runner atomically claims analyst-1's task
task:running       — runner calls /tasks/:id/start
task:progress      — (optional) periodic heartbeat from the run
task:completed     — run finishes, output posted as comment
agent:status       — analyst-1 reconciled back to "available"
```

The board re-renders each event as it arrives — no page reload, no polling.

The WebSocket server pings every 30 seconds (S34) to detect stale connections, and automatically terminates unresponsive clients. If the board browser loses the connection it reconnects and receives events from the current state.

---

## Step 7 — Optional branches: approval gate and crash recovery

The main path ends at Step 6. Two branches you can explore without new concepts:

### Approval gate (P21)

The spec (S18, §12) defines an `approvals` table with types including `request_board_approval`. If the analyst's task were "Deploy to production" rather than "Summarise issues", you could require board approval before the runner is allowed to start the task. The approval gate (built in [the approvals chapter](../governance/approvals-and-governance-gates.md)) works by:

1. The agent posts an `approval(type=request_board_approval, status=pending)` row before transitioning the issue to `in_progress`.
2. The orchestrator refuses to let the task proceed until the approval reaches `approved`.
3. The board approves in the UI; the approval status transitions and the task resumes.

No new wiring is needed — the approval rows share the same PostgreSQL DB and the same live events bus. The board sees a `pending_approval` card arrive on the live board; clicking "Approve" pushes it back through the same event stream.

### Crash recovery (P12)

What if the runner process died after claiming the task but before completing it? The crash recovery sweeper (built in [the crash recovery chapter](../tasks-and-queue/crash-recovery-and-liveness.md)) handles this. Inside the claim function the first thing it does is check for stale dispatched tasks:

```ts
// Simplified view — checking for stale dispatched tasks before touching the queue (S9 — task.go)
const stale = await queries.reclaimStaleDispatchedTaskForRuntime({
  runtimeId:         runtimeId,
  claimRecoverySecs: 90,  // claimResponseRecoveryWindow = 90 seconds (S9, line 88)
});
if (stale) {
  // The runner just reclaimed a task the previous instance left behind.
  return stale;
}
```

`claimResponseRecoveryWindow` is 90 seconds (S9, line 88). If a task was dispatched but never transitioned to `running` within that window, the next claim call atomically moves it back to `queued` and the current runner picks it up. The board observer sees the status flip: `dispatched → queued → dispatched → running`.

---

## The architecture you built

Let's pause and look at what the system does end-to-end:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Orchestrator process                                               │
│                                                                     │
│  ┌──────────────┐   enqueue()   ┌──────────────────────────────┐   │
│  │  Scheduler   │──────────────>│  Task Service                │   │
│  │  Loop        │               │  (claim / complete / fail)   │   │
│  └──────────────┘               └──────────┬───────────────────┘   │
│                                            │ broadcastTaskEvent()   │
│  ┌──────────────┐  notifyTaskAvailable()   │                        │
│  │  Runner Hub  │<─────────────────────────┘                        │
│  │  (WS, daemons│                          │ publishEvent()         │
│  │   connect)   │                          ▼                        │
│  └──────┬───────┘               ┌──────────────────────────────┐   │
│         │ runner:task_available  │  Live Events Bus             │   │
│         │                       │  (company-scoped fan-out)    │   │
│         ▼                       └──────────┬───────────────────┘   │
│  ┌──────────────┐                          │ push JSON event        │
│  │  Runner      │ claim (HTTP POST)        ▼                        │
│  │  process     │──────────────>  ┌────────────────────┐           │
│  │  (your       │                 │  Live Board WS     │           │
│  │   machine)   │                 │  (browsers connect)│           │
│  └──────┬───────┘                 └────────────────────┘           │
│         │ invoke()                                                   │
│         ▼                         ┌────────────────────┐           │
│  ┌──────────────┐  cost event     │  Budget Enforcer   │           │
│  │  Adapter     │────────────────>│  (pause on 100 %)  │           │
│  │  Registry    │                 └────────────────────┘           │
│  │  (LLM/mock)  │                                                   │
│  └──────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
                     ▲ squad delegation (sub-task enqueue via createIssue)
                     └── Org Chart / Squads (coordination layer)
```

Each box corresponds to one book section. The arrows are real function calls, WebSocket frames, or database writes you traced in the chapters.

---

## Try it yourself

Now that you can see the whole path, here are three experiments that each teach something new:

### Swap in the real LLM adapter

In Step 4c we used `"mock"` as the adapter type. To run against a real model:

1. Set `ANTHROPIC_API_KEY` in your runner's environment.
2. Change the agent's `adapterType` to `"claude-cli"` (and configure `cwd`, `timeoutSec`).
3. Enqueue a new task and watch the token counts appear in the cost event row.

Note: the adapter interface does not change — the mock and LLM adapters both implement `invoke / status / cancel` and return the same `AdapterResult` shape. Only the registry lookup changes. (S41 documents this for Claude: "If `ANTHROPIC_API_KEY` is set in adapter env or host environment, Claude uses API-key auth instead of subscription login.")

### Add a second squad member and watch load spread

1. Add `analyst-2` to the Research squad with the same adapter type and a different `maxConcurrentTasks`.
2. Create ten tasks in quick succession assigned to the squad.
3. Watch the live board: the squad leader will distribute sub-tasks across both members. The claim loop respects each agent's `maxConcurrentTasks` limit, so work spreads rather than piling up on one runner.

### Trip the budget mid-run

1. Set `analyst-1`'s `budgetMonthlyCents` to a very small value (e.g. 1 cent = `1`).
2. Enqueue a task that requires an LLM call.
3. The first cost event will push `spentMonthlyCents` past the limit.
4. Watch the live board receive the `agent:status` event with status `"paused"` and the budget-exceeded activity event.
5. Resume the agent by raising the budget in the UI and calling `POST /agents/:id/resume`.

This exercises the budget enforcement loop (S18, §13.2) end-to-end without any code changes.

---

## What comes next

You have now traced a complete Swarm run — from schedule through squad delegation, claim, execution, cost tracking, and live streaming. Every component you built is load-bearing in that path.

The next and final chapter points you toward the parts of the system worth extending: adding a new adapter type, hooking in a custom governance rule, building a richer board UI, and hardening the scheduler for production.

---

← Previous: [Approvals and Governance Gates](../governance/approvals-and-governance-gates.md) · Next: [Where to Go Next](./where-to-go-next.md) →
