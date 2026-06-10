---
title: "The Gateway: Port 18789, Wire Protocol, and Node Pairing"
description: "How the Gateway process works — one multiplexed port, WebSocket RPC, the connect/req/res/event wire frames, loopback-default security, and node pairing."
category: gateway
type: explanation
tags: [gateway, port 18789, websocket, wire protocol, connect frame, req frame, res frame, event frame, node pairing, loopback, multiplexed, auth, handshake, mermaid, device token, hello-ok, control plane, pairing, protocol version, shared secret, operator, node role, UNAVAILABLE, AUTH_TOKEN_MISMATCH, PAIRING_REQUIRED]
keywords: [gateway process, websocket rpc, connect handshake, loopback bind, network exposure, device pairing approval, connect challenge, nonce, scope gating, gateway token, gateway password, trusted-proxy, openclaw gateway, 18789]
sources: [S15, S33, S34, S43, S80, S95, S96, S97, S115]
---

**TL;DR** — The Gateway is a single long-lived process that binds one port (default 18789) and multiplexes WebSocket RPC, OpenAI-compatible HTTP endpoints, the Control UI, and plugin routes on it. Every client speaks a three-frame wire protocol (`req`/`res`/`event`) and must complete a `connect` handshake before sending anything else. By default the Gateway binds only to loopback, which keeps it invisible to the network. This chapter makes those mechanics concrete so you can reason about what happens when a client connects, what goes wrong when auth fails, and how remote devices earn trust through node pairing.

# The Gateway: Port 18789, Wire Protocol, and Node Pairing

The [previous chapter](../getting-started/02-architecture.md) introduced the Gateway as the control plane — the single process that everything else connects to. Let's now look inside it: where it listens, what it serves on that single address, how clients talk to it, and why it starts life invisible to the network.

## One port, many surfaces

When you run `openclaw gateway`, the process binds to one address (by default `127.0.0.1:18789`) and serves everything through that one port. Think of port 18789 as a post office building with several service windows inside — the address on the envelope is always the same, but once you walk through the door you can reach different counters.

The surfaces multiplexed on that one port are:

| Surface | What it is |
|---|---|
| WebSocket RPC | The main control-plane API — all CLI, UI, and automation clients connect here |
| OpenAI-compatible HTTP | `/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses` — lets you point third-party tools at OpenClaw as if it were an OpenAI-compatible backend |
| Control UI | A Vite + Lit single-page app served at `http://127.0.0.1:18789/` — a browser-based dashboard for chat, config, channels, and cron jobs |
| Plugin HTTP routes | Optional plugin-registered HTTP surfaces such as the admin RPC route at `/api/v1/admin/rpc` (off by default) |
| Canvas host | `/__openclaw__/canvas/` and `/__openclaw__/a2ui/` — agent-editable HTML/JS surfaces |

The port and bind address are configurable. Resolution order:

```
--port flag  →  OPENCLAW_GATEWAY_PORT env var  →  gateway.port config  →  18789 (default)
--bind flag  →  gateway.bind config            →  loopback (default)
```

We'll come back to what "loopback" means and why it matters in the security section below.

## The wire protocol

Every client that wants to control the Gateway — the CLI, the macOS app, the Control UI, an iOS node, an automation script — connects over WebSocket and speaks the same three-frame JSON protocol. The frames are:

| Frame type | Direction | Shape | Purpose |
|---|---|---|---|
| `req` | client → gateway | `{type:"req", id, method, params}` | Call a method; expects a matching `res` |
| `res` | gateway → client | `{type:"res", id, ok, payload\|error}` | Reply to a `req` with success or failure |
| `event` | gateway → client | `{type:"event", event, payload, seq?, stateVersion?}` | Server-pushed notification; no reply expected |

The `id` field in `req` and `res` is how the client matches a response to the call that triggered it — like a ticket number at that post office counter.

Events carry an optional `seq` (monotonic sequence number per client connection) and `stateVersion` (gateway state generation counter). Events are not replayed after a gap; if you miss some, you have to refresh state with a fresh `health` or `system-presence` call.

These three frame types are the only building blocks. Everything else — chat messages, agent runs, presence updates, cron events, config changes — flows through them.

## The `connect` handshake: why it must come first

Here is the most important rule of the protocol: **the first frame any client sends must be a `connect` request.** The Gateway closes the connection immediately if it receives anything else before a successful handshake.

Why? Because the Gateway needs to know who is asking before it will serve anything. The `connect` frame carries auth credentials, a protocol version range, a client description, and device identity. The Gateway uses all of these to decide whether to admit the connection and what it is allowed to do.

There is also a pre-`connect` challenge step. The Gateway sends a `connect.challenge` event immediately when a WebSocket opens — before the client sends anything:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

The client must sign this nonce with its device private key and include the signature in the `connect` request. This prevents replay attacks — a captured `connect` frame cannot be reused because the nonce changes each time.

### Anatomy of a `connect` request

Here is an operator client (CLI or Control UI) connecting:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

The key fields to understand:

- **`minProtocol` / `maxProtocol`** — the client's supported protocol version range. The current protocol is version 4. If the server's version falls outside the range the client sends, the handshake is rejected. Think of it like negotiating which edition of a board game you're playing before you sit down.
- **`role`** — either `"operator"` (control-plane clients: CLI, UI, automation) or `"node"` (capability hosts: iOS, Android, headless devices with camera/screen/location/voice). The role determines what the client can do.
- **`scopes`** — the permissions the client is requesting within its role. Common operator scopes: `operator.read`, `operator.write`, `operator.admin`, `operator.pairing`.
- **`auth`** — carries `token`, `password`, `deviceToken`, or `bootstrapToken` depending on the configured auth mode.
- **`device`** — the client's stable device identity: its public key, the nonce from the challenge, and a signature. The `nonce` here must match the one the Gateway sent in the `connect.challenge`.

A node client uses the same frame shape but declares `"role": "node"` and fills `caps`, `commands`, and `permissions` with the device capabilities it can provide (for example `"caps": ["camera", "canvas", "screen", "location", "voice"]`).

### The `hello-ok` response

If the handshake succeeds, the Gateway responds with a `res` frame whose payload type is `"hello-ok"`:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

The `hello-ok` tells the client everything it needs to know to behave correctly:
- `protocol` — the version the server settled on.
- `features.methods` and `features.events` — a discovery list of what this Gateway supports. Not every callable method appears here; treat it as a capability hint.
- `snapshot` — a bundle of current state (presence, health, etc.) so the client doesn't need to make extra calls to get initial state.
- `auth` — the negotiated role and scopes. When a device token is issued, `auth.deviceToken` is also included here; the client should persist it for future connects.
- `policy` — the Gateway's limits that the client must respect: max frame size (25 MB default), max buffered bytes (50 MB default), and tick interval (used for liveness checks).

If the Gateway is still finishing startup, it may return a retryable `UNAVAILABLE` error with `details.reason: "startup-sidecars"` and a `retryAfterMs`. Clients should retry within their connection budget rather than treating this as a permanent failure.

### Sequence diagram for a full connect

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Gateway-->>Client: event: connect.challenge {nonce, ts}
    Note over Client: Client signs nonce with device key

    Client->>Gateway: req: connect {role, auth, device{id, publicKey, signature, nonce}, minProtocol, maxProtocol, ...}

    alt Auth + pairing OK
        Gateway-->>Client: res: connect {ok:true, payload:{type:"hello-ok", protocol, snapshot, auth, policy}}
        Note over Client: Connection accepted — client can now send req frames
        Gateway-->>Client: event: presence (initial snapshot)
        Gateway-->>Client: event: tick (periodic heartbeat)
        Client->>Gateway: req: agent {agentId, message, ...}
        Gateway-->>Client: res: agent {status:"accepted", runId}
        Gateway-->>Client: event: agent (streaming chunks)
        Gateway-->>Client: res: agent {status:"ok", summary}
    else Auth failed (wrong token or password)
        Gateway-->>Client: res: connect {ok:false, error:{code:"AUTH_TOKEN_MISMATCH", ...}}
        Note over Gateway: Connection closed
    else Pairing required (new device)
        Gateway-->>Client: res: connect {ok:false, error:{code:"PAIRING_REQUIRED", ...}}
        Note over Gateway: Pending pairing request stored; connection closed
    else Protocol version mismatch
        Gateway-->>Client: res: connect {ok:false, error:{code:"PROTOCOL_MISMATCH", ...}}
        Note over Gateway: Connection closed
    end
```

## Failure paths

### Failed handshake: wrong auth

If the client sends an incorrect token or password, the Gateway returns an error `res` and closes the connection. The error includes `error.details.code` and a `recommendedNextStep` hint:

- `AUTH_TOKEN_MISMATCH` — the token was not recognized. On trusted endpoints (loopback or `wss://` with a pinned TLS fingerprint), clients may attempt one bounded retry with a stored device token. If that also fails, the client should stop automatic reconnect loops and prompt the operator.
- `AUTH_SCOPE_MISMATCH` — the device token was recognized but its granted scopes do not cover what was requested. The operator needs to re-pair or approve the scope change explicitly.

### Pre-connect frames rejected

If a client sends any frame before `connect` completes, the Gateway closes the connection immediately with a hard close. Invalid or non-JSON first frames are treated the same way. There is no way to recover from this on the same connection — the client must reconnect and start the handshake again.

### UNAVAILABLE during startup

If the Gateway returns `UNAVAILABLE` with `details.reason: "startup-sidecars"`, the process is still initializing background services. Clients should wait for `retryAfterMs` and retry, not surface this as a terminal failure.

## Loopback-default security model

Now let's talk about why the Gateway binds to `127.0.0.1` (loopback) by default.

Loopback is a special network address that only processes running on the same machine can reach. Nothing from your local network, let alone the internet, can connect to it. Think of it like an internal intercom in a building — people outside the building cannot ring it, regardless of what they try.

This default is deliberate. OpenClaw is a personal assistant that runs on your machine. The things that connect to the Gateway are — by default — other programs on the same machine: your CLI, your local browser, your macOS app. There is no reason to expose it to the network unless you explicitly want remote access.

The bind mode is controlled by `gateway.bind` (or the `--bind` CLI flag):

| Mode | What it binds to | Who can connect |
|---|---|---|
| `loopback` (default) | `127.0.0.1` | Processes on the same machine only |
| `lan` | All local interfaces | Devices on your local network |
| `tailnet` | Tailscale interface | Devices on your Tailscale network |
| `all` | All interfaces | Anyone with network access to the machine |

**When you expose the Gateway to the network**, several things change:

1. **Auth becomes mandatory.** The Gateway refuses to bind to a non-loopback interface without a valid auth path (token, password, or trusted-proxy). The startup log will show `refusing to bind gateway ... without auth` if you try.
2. **Pairing requires explicit approval.** On loopback, a direct local connection from the same machine can be auto-approved. On the network, every new device must go through the manual pairing flow.
3. **The attack surface grows.** Any machine that can reach your network can attempt to connect. Shared-secret auth helps, but network exposure adds risk.

The recommended approach for remote access is a Tailscale or VPN tunnel — this keeps the Gateway on loopback while giving authorized remote devices a path to it. See [Security](../operations/20-security.md) for the full runbook on network exposure.

## Auth modes

The Gateway supports four auth modes, controlled by `gateway.auth.mode`:

| Mode | How it works | When to use |
|---|---|---|
| `token` | Client sends `connect.params.auth.token`; Gateway checks it against `gateway.auth.token` or `OPENCLAW_GATEWAY_TOKEN` | Default for most setups |
| `password` | Client sends `connect.params.auth.password`; Gateway checks against `gateway.auth.password` or `OPENCLAW_GATEWAY_PASSWORD` | Human-typed auth |
| `trusted-proxy` | Auth satisfied from request headers (for reverse proxy setups) | Non-loopback with a trusted ingress proxy |
| `none` | No shared-secret check at all | Private-ingress setups where network path is the trust boundary |

A special identity-bearing mode: Tailscale Serve (`gateway.auth.allowTailscale: true`) verifies auth from Tailscale's forwarded identity headers instead of a shared secret. The Gateway verifies the identity by resolving the client IP with `tailscale whois` before accepting.

> **Important:** Never expose `gateway.auth.mode: "none"` on a public or untrusted ingress. The auth check is the only gate between the network and the control plane.

If you configure both `gateway.auth.token` and `gateway.auth.password` without setting `gateway.auth.mode`, the Gateway refuses to start and reports an explicit error:

```
Invalid config: gateway.auth.token and gateway.auth.password are both configured,
but gateway.auth.mode is unset. Set gateway.auth.mode to token or password.
```

This prevents the resolver from guessing which credential you intended.

## Device pairing

When a client connects for the first time with a new device identity, the Gateway does not automatically trust it — even if the auth token is correct. It requires a separate **pairing approval** step. This is a second layer of trust on top of shared-secret auth.

Think of it like a keycard building. The shared secret (`auth.token`) is proof you know the building's code. Pairing approval is the front desk recognizing your face and issuing you a personal badge. On subsequent visits you show the badge and skip the desk.

Here is how the flow works:

1. Client sends `connect` with a new `device.id`.
2. The Gateway stores a pending pairing request and emits a `device.pair.requested` event to any connected operators.
3. An operator approves the request:
   ```bash
   openclaw devices list
   openclaw devices approve <requestId>
   ```
4. The Gateway issues a **device token** for this device. The client receives it in `hello-ok.auth.deviceToken` and persists it.
5. On future connects, the client includes the stored device token in `connect.params.auth.deviceToken` and skips the approval flow.

If the client sends a `connect` with an unrecognized device and pairing is required, the Gateway returns `PAIRING_REQUIRED` and closes the connection. The `error.details` include `recommendedNextStep: "wait_then_retry"` and `retryable: true` — the client should keep reconnecting with the same credentials until an operator approves the request.

### Loopback auto-approval

Direct local loopback connections (from `127.0.0.1` or `localhost`) can be auto-approved for device pairing. This keeps same-machine UX smooth — the CLI and local Control UI don't require manual approval when connecting from the same host.

Important nuance: if the loopback connection carries `Forwarded`, `X-Forwarded-*`, or `X-Real-IP` headers (as a reverse proxy would add), the Gateway treats it as a non-loopback connection and requires explicit approval. The loopback auto-approve path requires that the raw socket **and** the forwarded headers both indicate the same machine.

Tailnet and LAN connects — even from the same machine via a Tailscale address — are treated as remote and require explicit approval.

### Trusted same-process backend clients

Internal Gateway-owned subsystems (for example, the subagent session update path) use a narrow backend self-connect path with `client.id: "gateway-client"` and `client.mode: "backend"`. These clients may omit `device` when authenticating with the shared gateway token on direct loopback. This path is reserved for the Gateway's own internal control-plane work and is not available to external clients.

### Node pairing (capability hosts)

The pairing flow above covers *device* pairing — the trust that grants a client entry via the WebSocket handshake. Separately, OpenClaw has a concept of **node pairing** for capability hosts (iOS, Android, headless devices).

A node (a device with `role: "node"`) that wants to expose commands like `camera.snap` or `screen.record` to the Gateway's tool system goes through node pairing, which is managed via `node.pair.*` RPC methods and stored separately from device pairing records.

The key distinction: device pairing gates the WebSocket handshake itself. Node pairing gates whether the node's declared commands are available as tools after the handshake is already complete.

Node pairing works like this:

1. A node connects and its device pairing is handled (same flow as above).
2. The node calls `node.pair.request` — this is a separate RPC, not part of `connect`.
3. The Gateway stores a pending node pairing request and emits `node.pair.requested`.
4. An operator approves with:
   ```bash
   openclaw nodes pending
   openclaw nodes approve <requestId>
   ```
5. Once approved, the node's declared commands become available subject to the Gateway's command policy (`gateway.nodes.allowCommands`, `gateway.nodes.denyCommands`).

> **Important:** Before approval, all commands from the node are filtered and will not execute. Commands queued before approval are **dropped**, not deferred. Node pairing is required in addition to device pairing for nodes to expose their commands.

### Node pairing rejection

If an operator rejects a node pairing request (`openclaw nodes reject <requestId>`), the Gateway emits `node.pair.resolved` with a rejected status. The node remains connected (device pairing is unaffected) but its commands stay unavailable. Pending requests expire after **5 minutes** if neither approved nor rejected.

### Pairing storage

Both pairing stores live under the Gateway state directory (default `~/.openclaw`):

```
~/.openclaw/nodes/paired.json    ← approved node pairings + tokens
~/.openclaw/nodes/pending.json   ← pending node pairing requests
```

Device pairing records (for operators, the Control UI, and app clients) are stored in the same JSON-file pattern as node pairing — under `~/.openclaw/devices/`:

```
~/.openclaw/devices/paired.json    ← approved device pairings + tokens
~/.openclaw/devices/pending.json   ← pending device pairing requests
```

All four pairing files are sensitive; treat them like credentials.

### Auto-approval for trusted CIDRs

For private node networks where the Gateway already trusts the network path, you can opt in to automatic node pairing approval for specific IP ranges:

```json
{
  "gateway": {
    "nodes": {
      "pairing": {
        "autoApproveCidrs": ["192.168.1.0/24"]
      }
    }
  }
}
```

This applies only to fresh `role: node` device pairing with no requested scopes. Operator, browser, Control UI, and scope-upgrading nodes remain manual. There is no blanket LAN or private-network auto-approve mode — CIDRs must be explicit.

## Scope gating on broadcast events

One more detail worth knowing: server-pushed events are not broadcast indiscriminately to every connected client. The Gateway scope-gates events based on the client's negotiated scopes:

- **Chat, agent, and tool-result events** require at least `operator.read`. Clients without this scope skip these frames entirely.
- **Plugin-defined `plugin.*` broadcasts** require `operator.write` or `operator.admin`, depending on how the plugin registered them.
- **Transport events** (`heartbeat`, `presence`, `tick`, connect/disconnect lifecycle) are unrestricted — every authenticated session sees them.

Each client connection maintains its own monotonic sequence counter so broadcasts preserve ordering on that socket even when different clients see different filtered subsets of the event stream.

## Putting it together: the full lifecycle of a request

Let's trace one complete interaction to see how the pieces connect. After a successful `connect`, the client sends an `agent` request to ask an agent a question:

```json
{
  "type": "req",
  "id": "req-001",
  "method": "agent",
  "params": {
    "agentId": "default",
    "message": "What's the weather like?"
  }
}
```

The Gateway responds immediately with an acceptance ack:

```json
{
  "type": "res",
  "id": "req-001",
  "ok": true,
  "payload": { "status": "accepted", "runId": "run-abc" }
}
```

Notice the response comes back right away — the Gateway does not wait for the agent to finish. This is by design: agent runs can take seconds or minutes, and blocking the WebSocket connection for that duration would make the client unusable. Instead, the client gets a `runId` immediately, and the actual agent output streams in as `agent` events:

```json
{
  "type": "event",
  "event": "agent",
  "payload": { "runId": "run-abc", "delta": "The weather is ..." }
}
```

When the run finishes, a final `res` arrives:

```json
{
  "type": "res",
  "id": "req-001",
  "ok": true,
  "payload": { "status": "ok", "runId": "run-abc", "summary": "…" }
}
```

This two-stage pattern — immediate ack, then streaming events, then final response — is how every agent run works through the Gateway.

## Quick reference: client constants

These defaults come from the Gateway client implementation and the protocol version constants:

| Constant | Default | What it governs |
|---|---|---|
| Protocol version | `4` | Current wire protocol generation |
| Request timeout (per RPC) | `30 000` ms | How long a client waits for a `res` before giving up |
| Connect challenge timeout | `15 000` ms | Budget for the pre-`connect` nonce exchange |
| Initial reconnect backoff | `1 000` ms | First wait before reconnecting after a disconnect |
| Max reconnect backoff | `30 000` ms | Cap on exponential backoff |
| Max payload (pre-`hello-ok`) | `64` KiB | Frame size cap before the handshake completes |
| Max payload (post-`hello-ok`) | `25` MB | From `hello-ok.policy.maxPayload`; client must honor this |

After receiving `hello-ok`, clients should switch to the policy limits the Gateway advertised in that response rather than the pre-handshake defaults. The server-advertised `policy.tickIntervalMs` also governs liveness: a client will close the connection (WebSocket close code `4000`) if it hears nothing for `tickIntervalMs * 2` milliseconds.

---

See also: [Security](../operations/20-security.md) for the full network exposure runbook, Gateway authentication modes, sandbox configuration, and secret redaction.

---

← Previous: [High-Level Architecture: Four Layers and the Gateway Control Plane](../getting-started/02-architecture.md) · Next: [Channels: Message Surfaces, Session Grammar, and DM Pairing](../channels/04-channels.md) →
