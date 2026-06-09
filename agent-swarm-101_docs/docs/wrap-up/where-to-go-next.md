---
title: "Where to Go Next"
description: An honest survey of what the guide left out — horizontal scaling, auth hardening, observability, production Postgres, and operating patterns — with pointers for each.
category: wrap-up
type: explanation
tags: [next steps, horizontal scaling, security, authentication, JWT, observability, tracing, production Postgres, multi-tenancy, pgvector, vector search, gaps, scaling, hardening, deployment, rate limiting, activity log, runner, orchestrator, sweeper, liveness, Redis, concurrency limits, heartbeat, session tokens, SHA-256, connection pooling, migrations, autonomous loop, event-driven loop, safety-first loop]
keywords: [where to go next, production deployment, scale out, auth hardening, secret rotation, structured logging, distributed tracing, metrics, pgvector semantic search, multi-tenant, Swarm production checklist, known gaps]
sources: [S13, S41, S3]
---

**TL;DR** — You have built a working Swarm: an orchestrator that stores tasks and manages agents, runners that claim and execute work, real-time WebSocket updates, and adapters that front real AI CLIs. This chapter names the five areas the guide deliberately left out — horizontal scaling, security hardening, observability, production database operation, and runtime operating patterns — says exactly what each area involves, and tells you where to look next. It also lists honest gaps in the guide itself.

# Where to Go Next

Congratulations on reaching this point. The system you built in the previous chapters — the full Swarm — is not a toy. It has a real task queue, a real state machine, a real adapter layer, and real WebSocket delivery. A single-machine version of it will serve a small team today.

But a single machine is also where most guides stop, and that is exactly the gap production punishes. This chapter is that gap made visible.

We will walk through five areas the guide deliberately deferred, explain what you would need to add in each, and point you toward the patterns that real deployments use — grounded where possible in the source material behind this guide, and framed as honest general guidance where the sources are silent.

---

## Picking up where we left off

If you have not yet read [Putting It All Together](./putting-it-all-together.md), do that first. That chapter walks the complete, integrated system: orchestrator process, runner daemon, task queue, real-time bus, and the adapter wiring. Everything in this chapter assumes that system as the baseline.

---

## 1. Horizontal scaling

### What the guide covered

The guide runs one orchestrator and one runner on the same machine. The runner polls the orchestrator for tasks, claims them atomically, and executes them. The orchestrator holds a SQLite database and an in-process event bus.

### What a real deployment adds

Once you want more than one runner — or more than one orchestrator — you run into three coordination problems.

**Multiple runners.** The runner itself already knows it can run many tasks at once; the configuration key for that cap is `SWARM_DAEMON_MAX_CONCURRENT_TASKS`, which defaults to `20` in the reference implementation (S3). When you add a second runner, the only new requirement is that task claiming remains atomic in the database — and your SQL already handles that with `FOR UPDATE SKIP LOCKED`. Each runner is stateless between polls; you can add runners without coordinating them.

**Multiple orchestrators.** Running two orchestrators against the same database is harder. Each orchestrator process carries a background sweeper that periodically marks stale runners offline and fails orphaned tasks. If both sweepers run concurrently against the same rows, you get races. The reference sweeper in S13 runs on a 30-second tick with carefully-chosen thresholds:

```go
// From the production sweeper (S13) — these constants show the
// reasoning behind each timeout value.

// sweepInterval is how often we check for stale runtimes and tasks.
sweepInterval = 30 * time.Second

// staleThresholdSeconds marks runtimes offline if no heartbeat for
// this long. Must be strictly greater than the DB flush interval
// (60s) plus one daemon heartbeat cycle (~15s) plus the liveness
// scheduler tick interval (~30s). 150s leaves a 45s buffer above
// the 105s worst-case DB age.
staleThresholdSeconds = 150.0

// dispatchTimeoutSeconds fails tasks stuck in 'dispatched' beyond
// this. The dispatched→running transition should be near-instant,
// so 5 minutes means something went wrong.
dispatchTimeoutSeconds = 300.0

// queuedTTLSeconds expires tasks sitting in 'queued' without being
// claimed. 2 hours is conservatively above any reasonable 'queued
// behind a long-running task' window for an online runtime.
queuedTTLSeconds = 2 * 3600.0

// queuedExpireBatchSize caps how many queued rows one sweeper tick
// transitions to failed, so the sweep transaction stays short even
// against a large historical backlog.
queuedExpireBatchSize = 500
```

The sweeper is also where **Redis-backed liveness** fits in. In the reference (S13), when a runner sends a heartbeat the server stores a short-TTL key in Redis alongside the database row. When the sweeper runs, it does a batched Redis `MGET` before flipping any candidate to offline — if Redis says a runner is still alive, the sweeper skips it even though the database row looks stale (because the database write is deliberately deferred and batched). When Redis is unavailable or errors, the sweeper falls back gracefully to trusting the database timestamp alone:

```ts
// Simplified view of filterStaleRuntimesByLiveness (S13)
async function filterStaleRuntimesByLiveness(
  candidates: Runtime[],
  liveness: LivenessStore,
): Promise<string[]> {
  if (!liveness.available()) {
    // Redis unavailable — degrade to DB-only behaviour.
    return allIds(candidates);
  }

  // isAliveBatch does a single Redis MGET for all candidate IDs,
  // returning a Map<id, boolean> (true = heartbeat key still present).
  let aliveById: Map<string, boolean>;
  try {
    aliveById = await liveness.isAliveBatch(candidates.map((r) => r.id));
  } catch {
    // Store hiccup — this tick falls back to DB-only.
    return allIds(candidates);
  }

  // Only flip runners Redis says are dead.
  return deadIds(candidates, aliveById);
}
```

This pattern — fast in-memory liveness authority with a slower durable fallback — is the canonical approach for scaling the sweeper across orchestrator instances without double-flipping runners.

### The concrete next step

1. Move the database from SQLite to Postgres (see §4 below).
2. Add a Redis instance and implement the liveness store interface: `Available()`, `IsAliveBatch()`, `Forget()`.
3. Guard the sweeper with a distributed advisory lock so only one orchestrator instance runs each sweep tick (Postgres `pg_try_advisory_lock` works well here).
4. Scale runners independently — they are already stateless.

---

## 2. Security and auth hardening

### What the guide covered

The guide introduced WebSocket connections secured with session tokens, and API keys stored as SHA-256 hashes in the database (so the server never holds the plain-text key). Agents authenticate their runs by presenting a token the orchestrator validates before accepting task updates.

### What a real deployment adds

Session tokens and hashed keys are the right foundation, but production deployments add several layers on top.

**Real authentication with JWT.** The session tokens in the guide are opaque random strings the server looks up on every request. A production system typically issues short-lived JWT access tokens (signed with a private key, verified without a database round-trip) alongside longer-lived refresh tokens that are stored and can be revoked. The orchestrator's WebSocket handshake validates the JWT signature before upgrading the connection.

**Secret rotation.** API keys for agents should expire. Rotating a key means: generate a new key, store its SHA-256 hash, mark the old one as expired (not deleted — you need the audit trail), and let the agent re-authenticate. A key management endpoint should limit the blast radius of a leaked key.

**Rate limiting.** Unauthenticated endpoints (login, registration, the task-claim path) need rate limiting. Without it, a compromised runner can flood the task queue. A simple token-bucket per source IP at the reverse proxy layer is a reasonable first step.

**Least-privilege agent permissions.** Adapters run unsandboxed on the host machine — the reference documentation makes this explicit (S41): *"Local CLI adapters run unsandboxed on the host machine. That means: prompt instructions matter, configured credentials/env vars are sensitive, working directory permissions matter. Start with least privilege where possible."* In practice this means: give each runner a dedicated OS user with no write access outside its workspace root, and never inject secrets into adapter arguments that end up in process listings.

### The concrete next step

1. Replace opaque session tokens with a JWT library (e.g. `golang-jwt/jwt` for Go or `jsonwebtoken` for Node.js). Use short expiry (15 minutes) with a refresh path.
2. Add a `revoked_at` column to your API-key table and check it on every key validation.
3. Put a reverse proxy (Nginx, Caddy, or a cloud load balancer) in front of the orchestrator and configure rate limiting there.
4. Review adapter working-directory permissions — the runner already creates an isolated workspace directory per task (S3); make sure the OS user running the runner cannot write outside that root.

---

## 3. Observability

### What the guide covered

The guide writes structured logs at key lifecycle points — task claimed, run started, run completed — and the activity log table gives you a per-task audit trail. The real-time event bus lets you watch the system live in a connected client.

### What a real deployment adds

Logs and an activity table are a starting point, but three concerns compound as you add runners:

**Distributed tracing.** A task flows through several hops: HTTP claim → queue → runner → adapter → result callback. When a task fails, you want to reconstruct the full chain in one view. OpenTelemetry trace propagation passes a `trace-id` header through each hop. Every log line and DB write stamps that ID. A collector (Jaeger, Tempo) stitches them back into a timeline.

**Metrics.** Counts and rates matter more than individual log lines at scale: queue depth over time, claim-to-start latency, run duration by adapter type, sweep tick duration, Redis liveness cache hit rate. A Prometheus-compatible `/metrics` endpoint on the orchestrator exposes these; Grafana visualises them.

**The activity log as an audit trail.** Every chapter that writes to the task queue also writes an activity-log entry with the actor, the action, and a timestamp. In production this table becomes your compliance trail — especially important for multi-tenant deployments where one workspace's agents must never see another workspace's activity. Make sure every activity-log query filters by `workspace_id`.

### The concrete next step

1. Instrument the task lifecycle with OpenTelemetry spans: one span per task from queue write to result write, with child spans for each hop.
2. Add a `/metrics` endpoint exposing at minimum: `swarm_queue_depth`, `swarm_run_duration_seconds`, `swarm_sweep_tick_duration_seconds`.
3. Verify every activity-log query has a `workspace_id` predicate — an accidental cross-workspace read in a shared deployment is a data-isolation bug.

---

## 4. Production database

### What the guide covered

The guide uses SQLite for the database, which is the right choice for getting started: zero infrastructure, one file, easy to reset. The schema was designed with Postgres compatibility in mind from chapter two, so the one-line driver swap from the earlier setup chapter (`better-sqlite3` → `pg` with `DATABASE_URL`) is intentional.

### What a real deployment adds

**Connection pooling.** Postgres is not SQLite — it has a finite number of connections (typically 100 by default). Every orchestrator process and every migration tool needs to route through a pool. PgBouncer in transaction mode is the standard choice for server-side pooling; it sits between your application and Postgres and multiplexes hundreds of application connections onto a small pool of Postgres server connections.

**Migrations in CI.** The guide runs migrations manually. In production, migrations run as a step in your deployment pipeline before the new binary starts. The migration tool (Flyway, golang-migrate, or the custom `migrate-up` target in the reference build — S3) runs with a lock so two deploys cannot apply the same migration twice.

**pgvector for semantic task search.** If you want to query tasks by meaning rather than keyword — "find all tasks similar to this one" — `pgvector` adds a vector column to the tasks table and an approximate-nearest-neighbour index. The `pgvector/pgvector:pg17` image used in the reference CI (S3) ships the extension already installed; you enable it with `CREATE EXTENSION IF NOT EXISTS vector`.

<!-- GAP: The guide does not specify how task embeddings are generated or which embedding model is assumed; source silent on this detail. -->

A `pgvector` search query adds an operator the guide does not cover:

```sql
-- Approximate nearest-neighbour search by cosine distance
SELECT id, title
FROM tasks
ORDER BY embedding <=> $1          -- <=> is the cosine-distance operator
LIMIT 10;
```

The `embedding` column would be of type `vector(1536)` (or whatever dimensionality your embedding model produces).

### The concrete next step

1. Switch `DATABASE_URL` to point at a Postgres instance (the one-line swap from setup chapter two).
2. Put PgBouncer in front and set `pool_mode = transaction`.
3. Add a migration step to your deploy script: `migrate-up` before the binary starts.
4. If you want semantic search: `CREATE EXTENSION IF NOT EXISTS vector;`, add an `embedding vector(N)` column to `tasks`, populate it from an embedding API call when tasks are created, and add an IVFFlat or HNSW index.

---

## 5. Operating patterns

### What the guide covered

The guide builds the machinery — the poll loop, the claim step, the adapter invocation, the run lifecycle — but does not tell you how to configure that machinery for different types of work.

### The three patterns from the reference documentation

The reference agent runtime documentation (S41) describes three operating patterns. They apply directly to how you configure your runners and prompt your agents.

| Pattern | Timer interval | Assignment wakeup | Best for |
|---|---|---|---|
| **Autonomous loop** | Short (e.g. 300s) | Enabled | Agents that should act continuously, leave durable progress, and self-direct |
| **Event-driven loop** | Long or disabled | Enabled | Agents that wait for assigned work; handoffs via comments and sub-tasks |
| **Safety-first loop** | Short | Optional | Untrusted or experimental agents; short timeout, conservative prompt, cancel quickly |

**Autonomous loop.** You enable a short timer and keep assignment wakeups on (S41, §7.1). The agent prompt tells the agent to act in the same heartbeat, leave durable progress in the task record, and mark blocked work with an explicit reason. You watch run logs and iterate the prompt. This pattern consumes the most tokens and requires the most prompt tuning.

**Event-driven loop.** You disable the timer or set it long, and rely entirely on assignment wakeups (S41, §7.2). Agents wake when someone assigns work to them, do that work, and go back to sleep. Handoffs happen through sub-task creation or comments. This pattern burns fewer tokens and is easier to reason about — but requires that someone (human or another agent) always routes the next piece of work.

**Safety-first loop.** You set a short timeout, write a conservative prompt, and monitor errors actively (S41, §7.3). When an agent drifts — repeated failures, unexpected outputs — you cancel the run and reset its session. This is the right starting point for any new agent whose behaviour you do not yet trust.

### The concrete next step

1. Start every new agent in safety-first mode until you trust its behaviour.
2. Move an agent to event-driven once you know what triggers it and that its outputs are reliable.
3. Reserve autonomous mode for agents that have proven stable in event-driven mode and that genuinely need to act without being asked.

---

## Known gaps in this guide

No guide covers everything, and this one is no exception. The following areas are either deliberately simplified or not covered:

- **Runner garbage collection.** A production runner periodically scans its workspace root and reclaims disk space from completed, cancelled, or orphaned task directories. The reference implementation (S3) has configurable GC with three modes (full cleanup, orphan cleanup, artifact-only cleanup) and TTLs like `SWARM_GC_TTL=24h`. The guide does not implement GC; your runner will fill disk on a long-lived deployment.
- **Multi-tenancy enforcement.** The guide has a single workspace. The real isolation requirement — every database query filtered by `workspace_id`, membership checks on every request, the `X-Workspace-ID` header routing scheme — is mentioned in context but not built end-to-end.
- **Secret management.** The guide stores API keys as SHA-256 hashes, which is correct for verification but does not cover secrets the agents themselves need (e.g. `ANTHROPIC_API_KEY` in an adapter's environment). In production these live in a secret store (Vault, AWS Secrets Manager, or Kubernetes Secrets) and are injected into the adapter environment at run time, not baked into configuration files.
- **Deployment.** The guide says nothing about containerisation, health endpoints, or rolling restarts. The runner daemon needs a `/health` endpoint and a clean-shutdown handler that drains in-flight runs before the process exits.
- **Cost accounting.** The guide writes token-usage fields to the run record, but does not implement budget caps or spend alerts per agent or per workspace.

These are gaps, not mistakes. Each is a real topic with a body of literature; treating them as an exercise for the reader is honest, not lazy.

---

## Stretch projects — keep building

If you want to go deeper, here are self-contained projects that extend the system you built:

1. **Add a new adapter.** The adapter interface is three methods: `invoke`, `status`, `cancel`. Write a `http` adapter that `POST`s task context to an arbitrary endpoint and polls for a result. This is the same shape as the `process` adapter but without a local subprocess.

2. **Add Prometheus metrics.** Instrument the task state machine: a counter for each state transition, a histogram for run duration. Wire a `/metrics` endpoint and point a local Prometheus scrape config at it.

3. **Deploy with Postgres.** Switch `DATABASE_URL` to a real Postgres instance, run the migrations, and verify that the sweeper's advisory-lock guard prevents double-sweeps when you start two orchestrator processes.

4. **Implement session resume.** The reference agent runtime (S41, §4) stores a session ID after each run so the next heartbeat can resume the same conversation. Add a `session_id` column to your runs table and pass it back to the adapter on subsequent invocations.

5. **Add pgvector task search.** Install the `vector` extension, add an `embedding` column to tasks, call an embedding API when tasks are created, and implement a "find similar tasks" query using the `<=>` cosine-distance operator.

---

You built a real agent orchestration system from scratch. The chapters in this guide are not a demonstration — they are working implementations of a task queue, a state machine, a real-time event bus, an adapter abstraction, and a sweeper that handles crashes gracefully. That is a real foundation. What you do with it is up to you.

---

← Previous: [Putting It All Together](./putting-it-all-together.md)
