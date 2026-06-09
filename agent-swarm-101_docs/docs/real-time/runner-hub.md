---
title: "WebSockets I — The Runner Hub"
description: Build the server-side WebSocket hub that runners connect to: a live runner registry, task-available wakeup signals, and a per-client dedup ring buffer.
category: real-time
type: tutorial
tags:
  - WebSocket
  - runner hub
  - runner WebSocket
  - runtime registry
  - task-available wakeup
  - heartbeat
  - ping pong
  - dedup ring buffer
  - RunnerIdentity
  - parseRunnerIDs
  - slow-client eviction
  - Hub
  - ws
  - wakeup coalescing
  - event bus
  - pub/sub
  - keepalive
  - write deadline
  - read deadline
  - pong handler
keywords:
  - push notification
  - replace polling
  - WebSocket upgrade
  - connection registry
  - best-effort delivery
  - client eviction
  - duplicate suppression
  - ring buffer
  - event deduplication
  - task dispatch
  - gorilla websocket
  - ws npm
sources: [S10, S6, S11]
---

**TL;DR** — Polling the task queue every few seconds works, but it wastes resources and adds latency. This chapter builds the **runner hub**: a server-side WebSocket registry that tracks which runners are online, wires into the event bus so a task-available notification travels to the right runner the instant work is enqueued, and protects against duplicate delivery with a per-client ring buffer. By the end you will have a working hub that runners can connect to and be woken up through.

# WebSockets I — The Runner Hub

## The problem with polling

Back in [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md), runners discover work by periodically asking the orchestrator "do you have anything for me?" — a polling loop on a short interval. Polling is reliable: even if a notification is missed, the next tick picks the task up. But it has two costs.

First, there is latency. A task that lands a millisecond after a poll cycle sits idle until the next tick — tens of seconds in the worst case. Second, every connected runner is firing HTTP requests even when there is nothing to do, which adds load to both the runner and the orchestrator.

The fix is to **push** a notification to the runner the moment a task is ready. The runner does not need to change its claim logic at all — we are not replacing the poll loop, we are giving it a doorbell. When the doorbell rings the runner runs the claim loop immediately; when it does not ring, the loop fires on its normal cadence as a backstop.

WebSockets are the right transport for this: they give us a persistent, bidirectional channel that the orchestrator can write to at any time. Let's build the hub that manages those channels.

## Step 1 — The handshake and runner identity

The first concern is: how does the orchestrator know which runner is connecting, and which tasks it can be woken up for?

A runner manages one or more **runner IDs** — the unique identifiers of the agent runtimes it owns. When it opens the WebSocket connection it tells the orchestrator which runner IDs it is registering for, by passing them as query parameters (`?runner_id=abc&runner_id=def` or as a comma-separated `?runner_ids=abc,def`). The HTTP handler reads these, deduplicates them, and checks that the authenticated caller actually owns each one before the upgrade happens.

Here is the identity-parsing helper that handles both parameter forms and silently drops duplicates:

```ts
// src/orchestrator/realtime/parse-runner-ids.ts
// Simplified view — the real implementation accepts both ?runner_id (repeatable)
// and ?runner_ids (comma-separated) query params, deduplicating inline.

export function parseRunnerIDs(url: URL): string[] {
  const seen = new Set<string>();
  const out: string[] = [];

  const add = (raw: string) => {
    for (const part of raw.split(",")) {
      const id = part.trim();
      if (id && !seen.has(id)) {
        seen.add(id);
        out.push(id);
      }
    }
  };

  for (const raw of url.searchParams.getAll("runner_id")) add(raw);
  for (const raw of url.searchParams.getAll("runner_ids")) add(raw);

  return out;
}
```

Once the IDs are parsed and ownership is verified, we package everything the hub needs to know about the connection into a `RunnerIdentity` value object:

```ts
// src/orchestrator/realtime/runner-identity.ts

export interface RunnerIdentity {
  runnerAgentID: string;   // the authenticated agent making the connection
  userID: string;
  workspaceID: string;
  runnerIDs: string[];     // the runner runtime IDs this connection covers
  clientVersion: string;
}
```

The HTTP upgrade handler — the function your router calls when a client hits the `/runner/ws` endpoint — validates ownership and then passes this identity object to the hub's `handleWebSocket` method:

```ts
// src/orchestrator/handler/runner-ws.ts  (simplified)
import { WebSocket, WebSocketServer } from "ws";
import type { IncomingMessage, ServerResponse } from "http";
import { parseRunnerIDs } from "../realtime/parse-runner-ids.js";
import { requireRunnerAccess } from "../middleware/runner-auth.js";
import type { Hub } from "../realtime/hub.js";

export function makeRunnerWebSocketHandler(hub: Hub) {
  return async function runnerWebSocket(
    req: IncomingMessage,
    res: ServerResponse
  ) {
    if (!hub) {
      res.writeHead(503);
      res.end(JSON.stringify({ error: "runner websocket unavailable" }));
      return;
    }

    const url = new URL(req.url ?? "/", "http://host");
    const runnerIDs = parseRunnerIDs(url);
    if (runnerIDs.length === 0) {
      res.writeHead(400);
      res.end(JSON.stringify({ error: "runner_ids required" }));
      return;
    }

    // Verify the authenticated caller owns every runner ID they listed.
    for (const runnerID of runnerIDs) {
      const ok = await requireRunnerAccess(req, runnerID);
      if (!ok) {
        res.writeHead(404);
        res.end(JSON.stringify({ error: "runner not found" }));
        return;
      }
    }

    // All IDs verified — perform the WebSocket upgrade.
    hub.handleWebSocket(req, res, {
      runnerAgentID: req.runnerAgentID,  // set by auth middleware
      userID: req.userID,
      workspaceID: req.workspaceID,
      runnerIDs,
      clientVersion: req.headers["x-client-version"] as string ?? "",
    });
  };
}
```

Notice that the hub itself does no authentication — it receives a fully-validated identity. Keeping auth out of the hub means the hub stays a pure connection-management concern.

## Step 2 — The hub registry

Now we can focus on the hub itself. Its core job is to maintain a live map from **runner ID → set of open connections** so that when a task becomes available for runner `abc`, the hub can find the right socket in O(1) without scanning every connection.

A runner can open more than one connection for the same runner ID (for example, if a crash left a stale socket open while a new one arrives). The registry therefore maps each runner ID to a *set* of clients.

```ts
// src/orchestrator/realtime/hub.ts

import { WebSocket } from "ws";
import type { IncomingMessage, ServerResponse } from "http";
import type { RunnerIdentity } from "./runner-identity.js";

// A single connected client.
interface Client {
  ws: WebSocket;
  identity: RunnerIdentity;
  // The set of runner IDs this connection covers, for fast O(1) lookup
  // when unregistering.
  runtimes: Set<string>;
  // Outbound message buffer — filled by notifyFrame, drained by the write loop.
  send: Array<Buffer>;
}

export class Hub {
  // All live clients.
  private clients = new Set<Client>();
  // Index: runnerID → set of clients watching that runner.
  private byRunner = new Map<string, Set<Client>>();

  private register(c: Client): void {
    this.clients.add(c);
    for (const runnerID of c.runtimes) {
      if (!this.byRunner.has(runnerID)) {
        this.byRunner.set(runnerID, new Set());
      }
      this.byRunner.get(runnerID)!.add(c);
    }
  }

  private unregister(c: Client): void {
    if (!this.clients.has(c)) return;
    this.clients.delete(c);
    for (const runnerID of c.runtimes) {
      const conns = this.byRunner.get(runnerID);
      if (conns) {
        conns.delete(c);
        if (conns.size === 0) this.byRunner.delete(runnerID);
      }
    }
  }

  handleWebSocket(
    req: IncomingMessage,
    res: ServerResponse,
    identity: RunnerIdentity
  ): void {
    if (identity.runnerIDs.length === 0) {
      res.writeHead(400);
      res.end(JSON.stringify({ error: "runner_ids required" }));
      return;
    }

    const runtimes = new Set(identity.runnerIDs.filter(Boolean));
    if (runtimes.size === 0) {
      res.writeHead(400);
      res.end(JSON.stringify({ error: "runner_ids required" }));
      return;
    }

    // Upgrade to WebSocket — the `ws` library handles the HTTP handshake.
    // (wss is the WebSocketServer instance created when the hub is constructed)
    this.wss.handleUpgrade(req, req.socket, Buffer.alloc(0), (ws) => {
      const c: Client = {
        ws,
        identity,
        runtimes,
        send: [],
      };
      this.register(c);
      this.startPumps(c);
    });
  }
}
```

With this registry in place, `hub.byRunner.get("runner-abc")` returns every socket currently watching runner `abc`. That is what `notifyTaskAvailable` will use in the next step.

## Step 3 — Task-available wakeup

We have connected sockets and a registry. Now we need a way for the rest of the orchestrator — specifically the part that enqueues tasks — to say "hey hub, runner `abc` has work."

### The event bus

We do not want the task-enqueueing code to have a direct import-time dependency on the hub. Instead, the orchestrator uses a lightweight **in-process event bus**: a pub/sub mechanism where any part of the server can publish an event, and any subscriber that registered for that event type is called.

Here is the full bus:

```ts
// src/orchestrator/events/bus.ts

export interface SwarmEvent {
  type: string;           // e.g. "task:available", "task:status-changed"
  workspaceID: string;
  actorType: "member" | "agent" | "system";
  actorID: string;
  payload: unknown;       // JSON-serializable, same shape as WebSocket payloads

  // Optional routing hints for the fanout layer:
  taskID?: string;
  runnerID?: string;
}

type EventHandler = (e: SwarmEvent) => void;

export class EventBus {
  private listeners = new Map<string, EventHandler[]>();
  private globalHandlers: EventHandler[] = [];

  // Register a handler for one event type.
  subscribe(eventType: string, handler: EventHandler): void {
    const list = this.listeners.get(eventType) ?? [];
    list.push(handler);
    this.listeners.set(eventType, list);
  }

  // Register a handler that receives ALL events, regardless of type.
  // Global handlers fire after type-specific handlers.
  subscribeAll(handler: EventHandler): void {
    this.globalHandlers.push(handler);
  }

  // Dispatch to all registered handlers. Handlers are called synchronously
  // in registration order. A panic/throw in one handler does not prevent
  // the others from executing.
  publish(e: SwarmEvent): void {
    const specific = this.listeners.get(e.type) ?? [];
    const globals = this.globalHandlers;

    for (const h of [...specific, ...globals]) {
      try {
        h(e);
      } catch (err) {
        console.error("panic in event listener", { eventType: e.type, err });
      }
    }
  }
}
```

When a task is enqueued, the task service publishes a `"task:available"` event. The hub subscribes to that event type and calls `notifyTaskAvailable` with the runner ID from the payload.

### Wiring the hub to the bus

```ts
// In your server bootstrap / dependency-injection setup:

const bus = new EventBus();
const hub = new Hub();

bus.subscribe("task:available", (e) => {
  if (e.runnerID) {
    hub.notifyTaskAvailable(e.runnerID, e.taskID ?? "");
  }
});
```

### What notifyTaskAvailable does

`notifyTaskAvailable` is intentionally **best-effort**. It looks up the clients for the given runner ID, constructs a small JSON frame, and tries to write it to each client's send channel. If the send buffer is full (a slow client), the client is evicted rather than blocking the caller:

```ts
// src/orchestrator/realtime/hub.ts  (continued)

notifyTaskAvailable(runnerID: string, taskID: string): void {
  if (!runnerID) return;

  const frame = JSON.stringify({
    type: "runner:task_available",
    payload: { runnerID, taskID },
  });

  this.notifyFrame(runnerID, Buffer.from(frame), /* eventID */ taskID);
}

private notifyFrame(
  runnerID: string,
  data: Buffer,
  eventID: string
): { delivered: boolean; deduped: boolean } {
  const clients = this.byRunner.get(runnerID);
  if (!clients) return { delivered: false, deduped: false };

  let delivered = false;
  let deduped = false;
  const slow: Client[] = [];

  for (const c of clients) {
    // Check the dedup ring buffer (explained in step 5).
    if (!this.markSeen(c, eventID)) {
      deduped = true;
      continue;
    }
    // Non-blocking enqueue to the client's send buffer.
    if (c.send.length < SEND_BUFFER_HIGH_WATER) {
      c.send.push(data);
      delivered = true;
    } else {
      slow.push(c);
    }
  }

  // Evict slow clients — don't block the notification path waiting for them.
  for (const c of slow) {
    this.unregister(c);
    c.ws.close();
  }

  return { delivered, deduped };
}
```

The runner receives the `runner:task_available` frame and immediately runs the claim loop from [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md). Remember: the claim loop is already built to be idempotent and safe to call at any time. The wakeup just tells it "now is a good time."

**Why best-effort?** If the runner is offline, the notification is silently dropped. When the runner comes back online, its periodic poll backstop will pick up the pending task. This keeps the orchestrator's hot path free of blocking writes.

## Step 4 — Heartbeats and keepalive

WebSocket connections idle when there is nothing to send. TCP keep-alive is unreliable across proxies and load balancers, so we need application-level liveness probes. The hub uses the WebSocket **ping/pong** mechanism for this.

The constants that drive the timing come from the source and are worth understanding:

```ts
// src/orchestrator/realtime/hub.ts

const WRITE_WAIT_MS  = 10_000;   // max time to write any single frame
const PONG_WAIT_MS   = 60_000;   // if no pong in this window, the connection is dead
// pingPeriod is (pongWait * 9/10) — fire pings slightly before the deadline
const PING_PERIOD_MS = (PONG_WAIT_MS * 9) / 10;  // 54 000 ms
```

`PONG_WAIT_MS` is the read deadline: the server expects a pong within 60 seconds of a ping. `PING_PERIOD_MS` is set to 90% of that window so the ping always fires before the deadline expires.

Each client has two goroutine-equivalent loops: a **read pump** that receives incoming frames and resets the pong deadline when a pong arrives, and a **write pump** that drains the send queue and fires periodic pings.

```ts
// src/orchestrator/realtime/hub.ts  (simplified pump sketch)

private startPumps(c: Client): void {
  const ws = c.ws;

  // Read pump: reset the read deadline whenever a pong arrives.
  ws.on("pong", () => {
    // Pong received — the connection is alive; reset the deadline.
    // (In Node.js you would use a timeout handle rather than a true deadline.)
    resetReadDeadline(c);
  });

  ws.on("message", (raw: Buffer) => {
    this.handleFrame(c, raw);
  });

  ws.on("close", () => {
    this.unregister(c);
    clearTimers(c);
  });

  ws.on("error", () => {
    this.unregister(c);
    clearTimers(c);
    ws.close();
  });

  // Write pump: drain the send queue and send periodic pings.
  const pingTimer = setInterval(() => {
    ws.ping();  // WebSocket PING frame — runner must respond with PONG
  }, PING_PERIOD_MS);

  // Store pingTimer handle on c so we can clear it on unregister.
  (c as any).__pingTimer = pingTimer;
}
```

When the write pump has a message to send, it sets a write deadline so a slow network connection cannot hold the server indefinitely:

```ts
private flushSend(c: Client): void {
  const message = c.send.shift();
  if (!message) return;

  c.ws.send(message, { binary: false }, (err) => {
    if (err) {
      // Write failed — treat as a dead connection.
      this.unregister(c);
      c.ws.close();
    }
  });
}
```

The read loop also sets the initial read deadline and applies a hard limit on inbound frame size:

```ts
// Inbound frame size limit — guards against a runner sending huge payloads.
ws.setMaxPayload(4096);
```

When the `pong` handler fires, the read deadline clock resets. If no pong arrives within `PONG_WAIT_MS`, the TCP connection is closed and the client is unregistered.

### Slow-client eviction

You might wonder: what if a runner's TCP buffer fills up and it cannot receive data? Without a write deadline, the server's `ws.send()` call would block, tying up a goroutine (or in Node.js, the event loop's I/O queue). The hub avoids this in two places:

1. In `notifyFrame`, if the client's send buffer is above the high-water mark, the client is evicted immediately instead of enqueueing more work.
2. In the write pump, the write deadline (`WRITE_WAIT_MS = 10 s`) ensures a blocked write does not stall indefinitely.

Once a client is evicted, the runner's reconnect logic brings it back, and the poll backstop covers the gap.

## Step 5 — The dedup ring buffer

We now have a hub that sends wakeup frames. But consider this scenario: a burst of ten tasks lands in rapid succession for the same runner. The hub fires `notifyTaskAvailable` ten times. The runner does not need ten wakeups — one is enough to trigger the claim loop, which drains as many tasks as it can in a single pass.

More importantly, consider network retransmissions or reconnects where the same event ID might be delivered more than once. Sending the same task notification twice could confuse a runner that tracks received event IDs.

The solution is a **per-client dedup ring buffer**. Each client maintains two data structures:

- A hash set (`seenIDs`) for O(1) membership tests: "have I seen event ID `X` before?"
- A ring list (`seenList`) to track insertion order so we know which entry to evict when the buffer is full.

When a new event ID arrives, `markSeen` checks the set. If the ID is already there, the frame is skipped (deduplicated). If not, the ID is added to both the set and the list. If the list grows beyond the capacity (128 in the source, tunable to your traffic profile), the oldest entry is dropped from both structures, making room for the new one.

```ts
// src/orchestrator/realtime/hub.ts

const DEDUP_RING_CAPACITY = 128;  // tunable: larger = fewer re-deliveries on long bursts

// markSeen returns true if the event should be delivered (not seen before).
// An empty eventID disables dedup and always returns true.
private markSeen(c: Client, eventID: string): boolean {
  if (!eventID) return true;   // no ID → always deliver

  if (!c.seenIDs) {
    c.seenIDs = new Set<string>();
    c.seenList = [];
  }

  if (c.seenIDs.has(eventID)) {
    return false;  // already delivered — skip
  }

  c.seenIDs.add(eventID);
  c.seenList.push(eventID);

  // If we have exceeded capacity, evict the oldest entry.
  if (c.seenList.length > DEDUP_RING_CAPACITY) {
    const drop = c.seenList.shift()!;
    c.seenIDs.delete(drop);
  }

  return true;  // deliver this event
}
```

Note the mutex (`dedupMu`) in the Go source: `markSeen` is called from `notifyFrame`, which may be invoked from multiple goroutines simultaneously. In a Node.js event-loop model you do not need a mutex for the ring buffer itself (the event loop is single-threaded), but if you ever call `notifyFrame` from worker threads, you would need a similar guard.

Why a ring buffer specifically? Because it gives bounded memory with no configuration: the oldest entries fall off automatically as new ones arrive. A naive "only deduplicate IDs we have seen in the last N seconds" approach would require a timer-driven sweep. The ring buffer deduplicates within the last N *deliveries* instead, which is exactly what we need — we care about recent duplicates, not time-bounded ones.

## Putting it all together

Let's look at the complete lifecycle of one task-available notification flowing through the hub:

```
1. Task enqueued
        │
        ▼
2. TaskService.publish("task:available", { runnerID: "abc", taskID: "t-1" })
        │
        ▼ (synchronous, in-process)
3. EventBus dispatches to hub's subscriber
        │
        ▼
4. hub.notifyTaskAvailable("abc", "t-1")
        │
        ▼
5. notifyFrame looks up byRunner["abc"] → { clientA, clientB }
        │
        ├─ clientA.markSeen("t-1") → true  → enqueue frame to clientA.send
        └─ clientB.markSeen("t-1") → true  → enqueue frame to clientB.send
        │
        ▼
6. Each client's write pump drains the send queue → ws.send(frame)
        │
        ▼
7. Runner receives { type: "runner:task_available", payload: { runnerID: "abc", taskID: "t-1" } }
        │
        ▼
8. Runner runs the claim loop immediately
```

If a second `"task:available"` event for `taskID: "t-1"` fires before the first is consumed (burst scenario), step 5 returns `markSeen("t-1") → false` for both clients, and the frame is dropped silently. One wakeup is sufficient.

## Reference: hub constants and tunables

| Constant | Source value | What it controls |
|---|---|---|
| `WRITE_WAIT_MS` | 10 000 ms | Maximum time allowed to write a single frame |
| `PONG_WAIT_MS` | 60 000 ms | Read deadline; connection declared dead if no pong |
| `PING_PERIOD_MS` | 54 000 ms (= pongWait × 9/10) | How often to send a WebSocket PING frame |
| `DEDUP_RING_CAPACITY` | 128 | Per-client dedup buffer size (entries, not bytes) |
| `SEND_BUFFER_HIGH_WATER` | 16 (send chan buffer in source) | Enqueued-but-unsent frames before slow-client eviction |

All of these are tunable. A high-throughput deployment with bursty task arrivals may benefit from a larger ring capacity; a deployment behind a proxy with aggressive idle timeouts may need a shorter `PONG_WAIT_MS`.

## Try it yourself

Once you have the hub wired into your server, here are three exercises that sharpen your intuition:

**1. Watch a runner be woken on enqueue.** Connect a test runner via `wscat` or a small script, then enqueue a task for its runner ID from a second terminal. Observe the `runner:task_available` frame arriving on the WebSocket connection without any polling.

**2. Verify the poll backstop still works.** Kill the WebSocket connection while a task is queued. Wait for the runner's periodic poll tick (from [The Task Queue and Worker Claim Loop](../tasks-and-queue/task-queue-and-claim-loop.md)) to fire. The task should still be claimed, demonstrating that the wakeup is an optimisation, not a correctness requirement.

**3. Observe dedup in action.** Temporarily shrink `DEDUP_RING_CAPACITY` to 2, then enqueue three tasks in rapid succession for the same runner ID with the same `taskID`. Add a log line inside `markSeen` to print whether each call returned `true` or `false`. You will see the first delivery go through and subsequent identical IDs be suppressed.

---

← Previous: [Agent-to-Agent Communication](../coordination/agent-to-agent-communication.md) · Next: [WebSockets II — The Live Board](./live-board.md) →
