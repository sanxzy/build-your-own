---
title: "WebSockets II — The Live Board"
description: Build the browser-facing live-events WebSocket server with workspace-scoped auth, two auth paths (session vs agent API key), 30s ping/pong keepalive, and a message catalogue.
category: real-time
type: tutorial
tags:
  - live board
  - live events WebSocket
  - workspace-scoped subscribe
  - auth
  - board session
  - agent API key
  - JWT
  - ping pong
  - slow-client termination
  - upgrade rejection
  - WebSocket message catalogue
  - noServer mode
  - real-time streaming
  - SHA-256
  - bearer token
  - keepalive
  - WebSocketServer
  - subscribeWorkspaceLiveEvents
  - deployment mode
  - local_trusted
  - authenticated
keywords:
  - live dashboard WebSocket
  - browser WebSocket streaming
  - noServer WebSocket attach
  - hashed API key authentication
  - WebSocket pong timeout
  - terminate slow client
  - HTTP upgrade rejection
  - workspace event fanout
  - swarm live events
sources: [S34, S11]
---

**TL;DR** — We already have a WebSocket hub that wakes runners (P16). Now we need a second WebSocket endpoint that lets a browser dashboard *watch* what the swarm is doing — tasks moving, agents running, costs accumulating — in real time. This chapter builds that live-board server: we attach it to the HTTP server in `noServer` mode, authenticate upgrades via two paths (a human's session token or an agent's hashed API key), subscribe each connected client to its workspace's event stream, and keep connections alive with a 30-second ping/pong cycle that terminates any client that goes silent.

# WebSockets II — The Live Board

In [WebSockets I — The Runner Hub](./runner-hub.md) we built the WebSocket channel that runners use to receive task assignments and report heartbeats. That channel is internal: runners are trusted processes on known machines.

The live board is the outward-facing side. A human opens a dashboard in their browser. They want to see, right now, whether their agent just started a task, whether a run finished, what it cost. Polling would work — but it adds latency, hammers the database, and gives a jerky UI. A persistent WebSocket connection, fed by the same event bus that already exists on the server, gives us low-latency push with no polling.

This chapter walks through building that connection, step by step. We will need to solve four distinct problems:

1. How do we attach a second WebSocket server to the same HTTP server without a second port?
2. How do we authenticate the upgrade before the WebSocket handshake completes?
3. How do we route events to the right client without one workspace's traffic leaking to another?
4. How do we detect and clean up dead connections?

Let us tackle them in order.

---

## The event bus (quick recap from P16)

Before we subscribe anyone to anything, we need to understand what we are subscribing to. The Swarm server runs an in-process event bus — a synchronous pub/sub mechanism where any service can publish a domain event and any number of handlers receive it.

The bus (S11) holds two kinds of listeners:

- **Type-specific** — registered with `Subscribe(eventType, handler)`. A handler only sees events whose `Type` field matches.
- **Global** — registered with `SubscribeAll(handler)`. Called for every event regardless of type, after all type-specific handlers have run.

Every `Event` carries a `WorkspaceID` field that identifies which workspace the event belongs to. The live-events layer uses that field to route each published event to only the clients that belong to that workspace — so one workspace never sees another's data.

If you want to go deeper on the bus itself, see [WebSockets I — The Runner Hub](./runner-hub.md) for the full treatment of how the event pipeline connects runners and clients.

---

## Step 1 — Attaching a second WebSocket server without a second port

Our HTTP server is already listening. We could spin up a second `http.Server` on a different port, but that complicates deployment: another port to open, another TLS certificate to terminate, another load-balancer rule. Instead, we want to share the single existing HTTP listener and route specific upgrade requests to our new WebSocket server.

The `ws` library's `noServer: true` option is exactly this escape hatch. When you create a `WebSocketServer` with `noServer: true`, the server does *not* listen on any port itself. It only knows how to negotiate a WebSocket handshake once you hand it a raw HTTP upgrade request. You become responsible for two things: deciding which upgrades belong to this server, and calling `handleUpgrade` to hand off the socket.

```ts
// Simplified view of the initial setup
import { createRequire } from "node:module";

const require = createRequire(import.meta.url);
const { WebSocketServer } = require("ws") as {
  WebSocketServer: new (opts: { noServer: boolean }) => WsServer;
};

const wss = new WebSocketServer({ noServer: true });
```

`wss` is now a WebSocket server that owns no port. It sits idle until we feed it an upgrade.

We intercept upgrades by listening on the `"upgrade"` event of the existing `HttpServer`:

```ts
server.on("upgrade", (req, socket, head) => {
  // req  — the incoming HTTP request that triggered the upgrade
  // socket — the raw TCP duplex stream
  // head — any bytes already buffered after the HTTP headers
  //
  // We decide here whether this upgrade is for us.
});
```

The browser will connect to a URL like `/api/workspaces/<workspaceId>/events/ws`. Let's parse that path first. We use a helper that extracts the workspace identifier from the pathname and returns `null` for any path that does not match:

```ts
function parseWorkspaceId(pathname: string): string | null {
  // Match /api/workspaces/<id>/events/ws
  const match = pathname.match(/^\/api\/workspaces\/([^/]+)\/events\/ws$/);
  if (!match) return null;
  try {
    return decodeURIComponent(match[1] ?? "");
  } catch {
    return null;
  }
}
```

> Each subscription is scoped to a single workspace, so a connected client only ever receives events that belong to its own workspace.

Inside the `"upgrade"` handler we call this helper. If the path does not match our pattern, we let the socket fall through (or destroy it, if nothing else will handle it):

```ts
server.on("upgrade", (req, socket, head) => {
  if (!req.url) {
    rejectUpgrade(socket, "400 Bad Request", "missing url");
    return;
  }

  const url = new URL(req.url, "http://localhost");
  const workspaceId = parseWorkspaceId(url.pathname);
  if (!workspaceId) {
    socket.destroy();  // Not our endpoint — drop silently
    return;
  }

  // It is our endpoint — authenticate, then hand off
});
```

Now we have a problem: we need to authenticate the client *before* completing the handshake, because `handleUpgrade` completes the handshake immediately. If we let an unauthorized client through to `handleUpgrade`, we have already upgraded the connection before we can reject it, and a WebSocket close code is far less clear than an HTTP 403.

---

## Step 2 — Rejecting unauthorized upgrades with a proper HTTP status line

The WebSocket handshake is just an HTTP `101 Switching Protocols` response. Until we send that response the connection is still a plain HTTP stream, and we can write any HTTP status line we want and then close the socket.

That is what `rejectUpgrade` does:

```ts
function rejectUpgrade(socket: Duplex, statusLine: string, message: string): void {
  // Strip any newlines from the message to prevent header injection
  const safe = message.replace(/[\r\n]+/g, " ").trim();
  socket.write(
    `HTTP/1.1 ${statusLine}\r\nConnection: close\r\nContent-Type: text/plain\r\n\r\n${safe}`
  );
  socket.destroy();
}
```

This writes a minimal HTTP response directly to the duplex stream — no framework, no `res.send()`. The browser receives a real HTTP 403 (or 400, or 500) and its `WebSocket` constructor fires an `onerror` event with the status code. That makes debugging much cleaner than a silent connection drop.

We call it in three situations:

| HTTP status | When |
|---|---|
| `400 Bad Request` | The request URL is missing |
| `403 Forbidden` | Authorization failed (bad token, wrong workspace) |
| `500 Internal Server Error` | The auth check itself threw an unexpected error |

---

## Step 3 — Two auth paths: session token vs agent API key

The live board is used by two distinct actor types.

- A **board client** — a human logged in through the browser. Their identity is proven by a session cookie (JWT or equivalent), managed by the server's auth layer.
- An **agent client** — an automated process (a runner, a script, an integration) that carries an API key issued from the workspace settings page.

Both must be scoped to a single workspace. A board client who is a member of Workspace A must not receive Workspace B's events, even if they somehow supply Workspace B's ID in the URL.

Let's walk through the authorization function:

```ts
async function authorizeUpgrade(
  db: Db,
  req: IncomingMessage,
  workspaceId: string,
  url: URL,
  opts: {
    deploymentMode: DeploymentMode;
    resolveSessionFromHeaders?: (headers: Headers) => Promise<BetterAuthSessionResult | null>;
  },
): Promise<UpgradeContext | null> {
  // ...
}
```

It returns an `UpgradeContext` on success, or `null` on failure. The context carries three fields:

```ts
interface UpgradeContext {
  workspaceId: string;   // the workspace this client belongs to
  actorType: "board" | "agent";
  actorId: string;     // userId for board, agentId for agent
}
```

### Token extraction

The function first looks for a bearer token. It checks two places: the `Authorization` header, and a `token` query parameter (useful in environments where custom headers are difficult to set on WebSocket connections):

```ts
const queryToken = url.searchParams.get("token")?.trim() ?? "";
const authToken = parseBearerToken(req.headers.authorization);
const token = authToken ?? (queryToken.length > 0 ? queryToken : null);
```

`parseBearerToken` strips the `Bearer ` prefix and returns `null` for anything that does not match the expected format.

### Path A — No token: session auth (board client)

If no token is present at all, we are dealing with a browser that relies on cookies for auth. What we do next depends on the server's `deploymentMode`:

```ts
if (!token) {
  if (opts.deploymentMode === "local_trusted") {
    // Fully local, no auth — treat every connection as a trusted board client
    return { workspaceId: workspaceId, actorType: "board", actorId: "board" };
  }

  if (opts.deploymentMode !== "authenticated" || !opts.resolveSessionFromHeaders) {
    return null;  // Unknown mode or no session resolver — reject
  }

  const session = await opts.resolveSessionFromHeaders(headersFromIncomingMessage(req));
  const userId = session?.user?.id;
  if (!userId) return null;

  // Check workspace membership
  const [roleRow, memberships] = await Promise.all([
    db.select(/* instance admin role */).where(/* userId */),
    db.select(/* workspace memberships */).where(/* userId, workspaceId, active */),
  ]);

  const hasAccess = roleRow || memberships.some((row) => row.workspaceId === workspaceId);
  if (!hasAccess) return null;

  return { workspaceId: workspaceId, actorType: "board", actorId: userId };
}
```

Two things to notice:

- `local_trusted` mode short-circuits — no database query, no session check. This is for local-only deployments where all users are trusted by definition.
- In `authenticated` mode the server resolves the session from the request headers (the cookie is there), then does a membership check. Instance admins (`instanceUserRoles`) can access any workspace; regular users need a row in `workspaceMemberships` for this specific workspace.

### Path B — Token present: agent API key (agent client)

If a token *is* present, we treat it as an agent API key. Agent API keys are stored as SHA-256 hashes in the database — the raw key is never stored, only the hash. So we hash the incoming token and look it up:

```ts
function hashToken(token: string): string {
  return createHash("sha256").update(token).digest("hex");
}

// Inside authorizeUpgrade, after the !token block:
const tokenHash = hashToken(token);
const key = await db
  .select()
  .from(agentApiKeys)
  .where(and(eq(agentApiKeys.keyHash, tokenHash), isNull(agentApiKeys.revokedAt)))
  .then((rows) => rows[0] ?? null);

if (!key || key.workspaceId !== workspaceId) {
  return null;  // Key not found, revoked, or belongs to a different workspace
}

// Stamp the key's lastUsedAt so the workspace can see recent usage
await db
  .update(agentApiKeys)
  .set({ lastUsedAt: new Date() })
  .where(eq(agentApiKeys.id, key.id));

return { workspaceId: workspaceId, actorType: "agent", actorId: key.agentId };
```

The workspace scope check (`key.workspaceId !== workspaceId`) is critical. A valid API key from Workspace A cannot be used to subscribe to Workspace B's event stream — the workspace IDs must match.

### Wiring it together

Back in the `"upgrade"` handler, once we have the workspace ID we run the auth check and either reject or proceed:

```ts
server.on("upgrade", (req, socket, head) => {
  // ... (url parsing and workspaceId extraction from Step 1) ...

  void authorizeUpgrade(db, req, workspaceId, url, opts)
    .then((context) => {
      if (!context) {
        rejectUpgrade(socket, "403 Forbidden", "forbidden");
        return;
      }

      // Attach the context to the request so the "connection" handler can read it
      (req as IncomingMessageWithContext).upgradeContext = context;

      wss.handleUpgrade(req, socket, head, (ws) => {
        wss.emit("connection", ws, req);
      });
    })
    .catch((err) => {
      logger.error({ err, path: req.url }, "failed websocket upgrade authorization");
      rejectUpgrade(socket, "500 Internal Server Error", "upgrade failed");
    });
});
```

We attach the resolved `UpgradeContext` to the request object before calling `handleUpgrade`. That way, when the `"connection"` event fires on `wss`, the handler can read the context off the request without re-running the auth query.

---

## Step 4 — Subscribe and push: connecting a client to its workspace's events

Once the handshake is complete, the `"connection"` event fires on `wss`. Now we need to:

1. Read the `UpgradeContext` we attached in Step 3.
2. Subscribe this socket to the workspace's live-event stream.
3. Push each event to the socket as a JSON string.
4. Clean up when the socket closes.

```ts
wss.on("connection", (socket: WsSocket, req: IncomingMessage) => {
  const context = (req as IncomingMessageWithContext).upgradeContext;
  if (!context) {
    // Should never happen — handleUpgrade only fires after auth — but guard anyway
    socket.close(1008, "missing context");
    return;
  }

  const unsubscribe = subscribeWorkspaceLiveEvents(context.workspaceId, (event) => {
    if (socket.readyState !== WebSocket.OPEN) return;
    socket.send(JSON.stringify(event));
  });

  cleanupByClient.set(socket, unsubscribe);
  aliveByClient.set(socket, true);  // covered in Step 5

  socket.on("close", () => {
    const cleanup = cleanupByClient.get(socket);
    if (cleanup) cleanup();
    cleanupByClient.delete(socket);
    aliveByClient.delete(socket);
  });

  socket.on("error", (err) => {
    logger.warn({ err, workspaceId: context.workspaceId }, "live websocket client error");
  });
});
```

`subscribeWorkspaceLiveEvents` is a function provided by the live-events service layer. It registers a callback on the event bus for this workspace's events and returns an `unsubscribe` function. We store that unsubscribe function in `cleanupByClient` — a `Map<WsSocket, () => void>` — keyed by the socket. When the socket closes, we call it immediately to avoid a memory leak.

The guard `socket.readyState !== WebSocket.OPEN` protects against the race where an event arrives between the close handshake starting and the `"close"` event firing.

---

## Step 5 — Keepalive: 30-second ping/pong and slow-client termination

Now we have a working subscription — but what happens when a client's network goes away silently? TCP keepalives exist, but they operate at the OS level with configurable timeouts that can be very long. The WebSocket protocol has its own application-level keepalive: the `PING`/`PONG` frame pair.

The server sends a `PING` frame; a well-behaved client responds with a `PONG` frame. If the `PONG` never arrives, the client is dead (or stuck) and we should terminate the connection to free its subscription.

We use two server-scoped maps to track liveness:

```ts
const cleanupByClient = new Map<WsSocket, () => void>();
const aliveByClient = new Map<WsSocket, boolean>();
```

`aliveByClient` maps each socket to a boolean. The logic is:

- At connection time, set it to `true`.
- When we receive a `"pong"` from the client, set it back to `true`.
- Every 30 seconds, iterate all clients. For each one:
  - If `alive` is `false`, the client missed the previous ping — terminate it.
  - If `alive` is `true`, set it to `false` and send a new ping.

```ts
const pingInterval = setInterval(() => {
  for (const socket of wss.clients) {
    if (!aliveByClient.get(socket)) {
      socket.terminate();  // No pong received — connection is dead
      continue;
    }
    aliveByClient.set(socket, false);  // Assume dead until pong arrives
    socket.ping();
  }
}, 30_000);
```

And in the `"connection"` handler we wire the `"pong"` listener:

```ts
socket.on("pong", () => {
  aliveByClient.set(socket, true);
});
```

`socket.terminate()` is a hard close — it does not send a WebSocket close frame, it just destroys the underlying TCP connection. That is appropriate here: if the client is not responding to pings, a graceful close would also not be received. The `"close"` event still fires on the socket after `terminate()`, so the cleanup path in Step 4 runs correctly.

When the WebSocket server itself shuts down, we clear the interval to avoid it firing against an empty (or garbage-collected) `wss`:

```ts
wss.on("close", () => {
  clearInterval(pingInterval);
});
```

---

## Step 6 — Putting it all together: `setupLiveEventsWebSocketServer`

The full initialization wraps everything above into a single exported function. The caller passes the HTTP server, the database handle, and the options (deployment mode + session resolver):

```ts
export function setupLiveEventsWebSocketServer(
  server: HttpServer,
  db: Db,
  opts: {
    deploymentMode: DeploymentMode;
    resolveSessionFromHeaders?: (headers: Headers) => Promise<BetterAuthSessionResult | null>;
  },
): WsServer {
  const wss = new WebSocketServer({ noServer: true });
  const cleanupByClient = new Map<WsSocket, () => void>();
  const aliveByClient = new Map<WsSocket, boolean>();

  // --- Step 5: ping interval ---
  const pingInterval = setInterval(() => {
    for (const socket of wss.clients) {
      if (!aliveByClient.get(socket)) {
        socket.terminate();
        continue;
      }
      aliveByClient.set(socket, false);
      socket.ping();
    }
  }, 30_000);

  // --- Step 4: subscribe + push on connection ---
  wss.on("connection", (socket, req) => {
    const context = (req as IncomingMessageWithContext).upgradeContext;
    if (!context) {
      socket.close(1008, "missing context");
      return;
    }

    const unsubscribe = subscribeWorkspaceLiveEvents(context.workspaceId, (event) => {
      if (socket.readyState !== WebSocket.OPEN) return;
      socket.send(JSON.stringify(event));
    });

    cleanupByClient.set(socket, unsubscribe);
    aliveByClient.set(socket, true);

    socket.on("pong", () => { aliveByClient.set(socket, true); });
    socket.on("close", () => {
      cleanupByClient.get(socket)?.();
      cleanupByClient.delete(socket);
      aliveByClient.delete(socket);
    });
    socket.on("error", (err) => {
      logger.warn({ err, workspaceId: context.workspaceId }, "live websocket client error");
    });
  });

  wss.on("close", () => clearInterval(pingInterval));

  // --- Steps 1–3: upgrade interception and auth ---
  server.on("upgrade", (req, socket, head) => {
    if (!req.url) {
      rejectUpgrade(socket, "400 Bad Request", "missing url");
      return;
    }

    const url = new URL(req.url, "http://localhost");
    const workspaceId = parseWorkspaceId(url.pathname);
    if (!workspaceId) {
      socket.destroy();
      return;
    }

    void authorizeUpgrade(db, req, workspaceId, url, opts)
      .then((context) => {
        if (!context) {
          rejectUpgrade(socket, "403 Forbidden", "forbidden");
          return;
        }
        (req as IncomingMessageWithContext).upgradeContext = context;
        wss.handleUpgrade(req, socket, head, (ws) => {
          wss.emit("connection", ws, req);
        });
      })
      .catch((err) => {
        logger.error({ err, path: req.url }, "failed websocket upgrade authorization");
        rejectUpgrade(socket, "500 Internal Server Error", "upgrade failed");
      });
  });

  return wss;
}
```

Call this once at server startup, after the HTTP server is created but before it starts listening:

```ts
// src/server.ts (illustrative)
const server = app.listen(PORT);
setupLiveEventsWebSocketServer(server, db, {
  deploymentMode: config.deploymentMode,
  resolveSessionFromHeaders: auth.resolveSessionFromHeaders,
});
```

---

## Message catalogue

When a client is connected and subscribed, it receives a stream of JSON-encoded events. Each message is a JSON string sent via `socket.send(JSON.stringify(event))` — that is, one JSON object per WebSocket text frame.

The shape of each event comes from the event bus's `Event` struct (S11):

```ts
// The envelope every message follows
interface LiveEvent {
  Type: string;         // event discriminator, e.g. "issue:created"
  WorkspaceID: string;  // the workspace this event belongs to
  ActorType: string;    // "member", "agent", or "system"
  ActorID: string;      // the id of the actor that caused the event
  Payload: unknown;     // JSON-serializable, shape varies by Type
  TaskID?: string;      // optional — set when the event relates to a specific task
  ChatSessionID?: string; // optional — set when related to a chat session
}
```

The `Type` field is the discriminator the client should switch on. The event bus defines the type string; the `Payload` shape is whatever the publishing service puts there.

<!-- GAP: Specific Type strings and their Payload shapes (e.g. "task:updated", "run:started", "run:finished", "cost:event") are not directly enumerated in S34 or S11. S11 gives the envelope shape and shows example type strings ("issue:created", "inbox:new") as doc-comments, but does not list the full catalogue of types emitted by the live-events service layer. Mark the type-string list as a gap. -->

The following table shows the type strings visible in S11's doc-comments. Additional types are emitted by the live-events service (not directly in these sources) and should be treated as illustrative examples:

| `Type` string | When it fires | Payload notes |
|---|---|---|
| `issue:created` | A new task is created in the workspace | <!-- GAP: Payload shape not in S34/S11 — source silent --> |
| `inbox:new` | A new inbox notification arrives | <!-- GAP: Payload shape not in S34/S11 — source silent --> |

> For the complete list of live-event types and their payload shapes, consult the live-events service layer (`src/services/live-events.ts` in your Swarm deployment). The envelope above is stable; the `Type` catalogue grows as new domain events are added.

### Handling messages in the browser

From a browser DevTools console (or a small script), connecting looks like this:

```ts
// Substitute your workspace ID and obtain a valid token from the session
const workspaceId = "ws_abc123";

// Board client: rely on session cookie — no explicit token needed if authenticated mode
const ws = new WebSocket(
  `ws://localhost:3000/api/workspaces/${workspaceId}/events/ws`
);

// Agent client: pass the API key as a bearer token via query param
// (browser WebSocket API cannot set custom headers)
const ws = new WebSocket(
  `ws://localhost:3000/api/workspaces/${workspaceId}/events/ws?token=<your-api-key>`
);

ws.onmessage = (e) => {
  const event = JSON.parse(e.data);
  console.log(event.Type, event.Payload);
};

ws.onerror = (e) => {
  // A 403 on upgrade appears here — check the Network tab for the status code
  console.error("WS error", e);
};
```

---

## Try it yourself

Here are three concrete experiments you can run once the server is running locally.

### 1. Watch events appear on task enqueue

Open a browser tab to your dashboard and open DevTools → Console. Paste the board-client connection snippet above (adjust the workspace ID). Create or update a task via the API or UI. You should see a JSON object logged for each domain event the action triggers.

### 2. Drop the pong and watch termination

Inspect the live connection from the Node.js side (or add a temporary log in the `"pong"` listener). You can simulate a dead client by monkey-patching the pong out:

```ts
// Wrap the pong listener after connecting (Node.js ws client only)
ws._socket.removeAllListeners("data");
// Now wait up to 60 seconds — two ping cycles — and watch the server terminate the socket
```

After at most two 30-second cycles, the server calls `socket.terminate()`. The client's `ws.onclose` fires with a code of `1006` (abnormal closure — no close frame was sent by the server before the TCP connection dropped).

### 3. Attempt a cross-workspace connection with a valid key

Obtain an API key from Workspace A. Attempt to connect to Workspace B's event stream using that key. You should receive an HTTP 403 in the WebSocket upgrade response — not a connected socket that then streams no events, but a hard rejection before the handshake completes.

---

← Previous: [WebSockets I — The Runner Hub](./runner-hub.md) · Next: [The Schedule Data Model](../scheduling/schedule-data-model.md) →
