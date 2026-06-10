---
title: "Design Decisions and Tradeoffs: SQLite, Exclusive Memory Slot, Loopback, In-Process Plugins"
description: "The reasoning behind four load-bearing OpenClaw architecture choices — and what each one trades away."
category: reference
type: explanation
tags: [design, tradeoffs, architecture, sqlite, security, plugins, memory-slot, loopback, in-process, storage, gateway, sandboxing, exclusive-slot]
keywords: [why sqlite, why loopback default, in-process plugin security, exclusive memory plugin, plugin trust model, gateway bind loopback, runtime state store, jsonl sessions]
sources: [S2, S3, S40, S42]
---

**TL;DR** — OpenClaw makes four non-obvious architectural choices that shape the entire system: it uses SQLite (not a separate database service) as the canonical runtime-state store; it restricts the memory plugin slot to one active plugin at a time; it binds the Gateway to loopback by default rather than all network interfaces; and it runs plugins in-process with the Gateway rather than in isolated subprocesses. This chapter walks through each decision — the problem it solves, the choice, and what it trades away.

# Design Decisions and Tradeoffs: SQLite, Exclusive Memory Slot, Loopback, In-Process Plugins

Before we dig into the decisions themselves, let's set the stage. A quick orientation for each concept involved:

- **The Gateway** is the long-running process that is the control plane for all agents, channels, and plugins. It binds to a port (default 18789) and receives messages from chat apps. We cover it in detail in [High-Level Architecture](../getting-started/02-architecture.md).
- **The memory plugin slot** is a special configuration point where exactly one memory plugin can be active at a time. The memory system is introduced in [Memory System](../memory/10-memory-system.md).
- **Plugins** are packaged runtime capabilities — they register tools, providers, channels, hooks, and services. Their structure is explained in [Plugins, Skills, and Tools](../extending/11-plugins-skills-tools.md).
- **Storage** in OpenClaw has two canonical forms: SQLite databases for runtime state, and JSONL files for session transcripts. The storage layer is introduced in [Storage and Persistence](../operations/19-storage.md).
- **Sandbox mode** runs tool execution inside isolated containers (Docker by default) to limit what the agent can touch. The security model is in [Security and Governance](../operations/20-security.md).

With those concepts placed, let's look at each architectural decision in turn.

---

## Decision 1: SQLite as the canonical runtime-state store

### The problem

OpenClaw needs to persist a range of runtime state: which plugins are installed and what their keys are, a task registry, a delivery queue, a model-catalog cache, per-agent auth profiles, and plugin key-value scratch data. The naive path is to reach for whatever storage the OS provides — JSON files, scattered text files, a mix of both.

That path creates a serious problem over time. Once there are multiple file formats, every migration, repair operation, and new feature has to touch all of them. There is no single place to query "what is the current state of the system?" Consistency guarantees are informal; if the process crashes mid-write, two files may disagree. And adding new state always tempts a developer to invent a new file rather than use an existing store — so the system accumulates debt.

### The choice

The AGENTS.md engineering guide states the rule directly:

> Storage default: SQLite only. Do not add JSON/JSONL/TXT/sidecar files for OpenClaw-owned runtime state, caches, queues, registries, indexes, cursors, checkpoints, or plugin scratch data.

And it gives the allocation:

> Use the shared state DB (`state/openclaw.sqlite`) for global runtime state and plugin KV data. Use the per-agent DB (`agents/<agentId>/agent/openclaw-agent.sqlite`) for agent-scoped state/cache. Use a dedicated SQLite DB only when schema, volume, or lifecycle clearly does not fit those stores.

There are two databases, not one:

| Database | Path | Holds |
|---|---|---|
| Shared state DB | `<stateDir>/state/openclaw.sqlite` | Plugin KV, task registry, delivery queue, model-catalog cache, installed-plugin index |
| Per-agent DB | `<stateDir>/agents/<agentId>/agent/openclaw-agent.sqlite` | Agent private state, auth profile tables (`auth_profile_store`, `auth_profile_state`) |

Queries go through Kysely helpers — Kysely is a third-party type-safe SQL query-builder library for TypeScript that generates SQL from typed method chains rather than raw strings — not raw SQL strings (except schema DDL, migrations, and low-level DB bootstrap).

Think of this like a well-organized workshop: instead of leaving parts on every surface, everything goes in one of two labelled drawers. The shared drawer holds what the whole shop needs; the personal drawer holds what only one workstation needs. When you need to find something, you know exactly which drawer to check.

### What it trades away

SQLite is an excellent fit for a single-process, single-host deployment — and that is OpenClaw's intended deployment model. But SQLite does not support concurrent writes from multiple processes the way a networked database server does. The AGENTS.md guide acknowledges this implicitly by framing it as a single-process architecture: the Gateway reads and writes these databases; it is not designed for multiple Gateway instances sharing one SQLite file across a network.

The AGENTS.md guide also imposes a strict migration discipline as the price of this simplicity:

> Persistent user state gets one migration owner. Doctor migrates, verifies, and then runtime assumes the new shape. No dual-write, read-through fallback, lazy import, or "if SQLite fails use JSON" branches.

If a config change makes existing data invalid, `openclaw doctor --fix` must migrate it before the runtime ever touches it. (`openclaw doctor` is OpenClaw's built-in diagnostic command; the `--fix` flag tells it to apply any pending migrations and repairs rather than merely reporting them.) The runtime reads only the canonical, current shape — there is no legacy fallback in the steady-state code path.

**What SQLite does not own:** JSONL session files. Session transcripts — the record of what was said and done in a conversation — live at `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`. This is intentional: a JSONL file is a named product artifact (a user-readable, appendable conversation record), not app state or a cache. The AGENTS.md guide distinguishes these clearly:

> File storage must be a named product artifact: import/export, user attachment, log, backup, or external tool contract. If it is app state or cache, it belongs in SQLite.

The JSONL session file exists because it is the session record itself — not because it is a convenient place to store data. Runtime-owned state (indexes, cursors, queues) goes in SQLite; the human-readable session transcript stays in JSONL.

---

## Decision 2: The memory plugin slot is exclusive

### The problem

OpenClaw ships multiple memory implementations: `memory-core` (a dreaming subagent with read/write memory tools), `memory-lancedb` (a vector database with read/write/forget tools), and `memory-wiki` (a knowledge vault with search and retrieval tools). These are all different shapes of "memory" — they each register tools and hook into the agent's context assembly in different ways.

What would happen if two of them were active simultaneously? Both might try to inject context into the system prompt. Both might register tools that share names or overlap in semantics. The dreaming subagent in `memory-core` might write to a store that `memory-lancedb` is simultaneously indexing. The source does not detail the exact failure modes, but the architectural choice is that the ambiguity itself is the problem: if both are active, neither can reason about what the other is doing, and the resulting behavior is unpredictable.

### The choice

VISION.md states this directly:

> Memory is a special plugin slot where only one memory plugin can be active at a time.

<!-- GAP: exact config key name (e.g. plugins.slots.memory) is not confirmed in the four assigned sources (S2, S3, S40, S42); it appears only in plugin manifests assigned elsewhere -->

The memory slot holds at most one active memory plugin. When you install `memory-lancedb`, it occupies this slot; `memory-core` is no longer active until you switch back. This is not a technical limitation of the plugin system — it is a deliberate design rule.

Think of it like a turntable cartridge: the slot is fixed, and only one needle can touch the record at a time. You choose the needle that fits your collection.

### What it trades away

The exclusive slot means you cannot simultaneously use the dreaming subagent from `memory-core` *and* the vector recall from `memory-lancedb`. If you want long-term recall and a background consolidation process, you must choose one implementation that provides both, or accept that the consolidation runs separately as a cron-triggered operation.

VISION.md notes the long-term direction but is explicit about where things stand today:

> Today we ship multiple memory options; over time we plan to converge on one recommended default path.

The source does not describe any mechanism for composing memory plugins. The current tradeoff is clarity and predictability over flexibility: one active memory system means one predictable set of tools in the model's context, one context-injection path, and one place to debug if memory retrieval goes wrong.

<!-- GAP: whether any memory plugin (e.g. memory-wiki) sits "beside" the exclusive slot rather than in it, and the specific invocation details, are not confirmed in the four assigned sources (S2, S3, S40, S42); those details come from plugin manifests not assigned to this page -->

---

## Decision 3: The Gateway binds to loopback by default

### The problem

The Gateway exposes a substantial attack surface. Its single port (default 18789) multiplexes WebSocket RPC, the Control UI (an SPA), and HTTP APIs including OpenAI-compatible endpoints. Anything reachable on this port can interact with the agent, see the Control UI, and (with valid credentials) make persistent control-plane changes.

If the Gateway bound to `0.0.0.0` (all network interfaces) by default, it would be reachable from the local network, and potentially from the internet, the moment it started — before the operator had set up auth, before DM pairing was configured, before any of the hardening described in the security documentation was in place. A first-run user who has not yet read the security docs would have an exposed Gateway.

### The choice

S40 (the security index) states:

> `gateway.bind: "loopback"` (default): only local clients can connect.

And further:

> Never expose the Gateway unauthenticated on `0.0.0.0`.

The Gateway listens on `127.0.0.1` (loopback) by default. Local clients — the OpenClaw CLI, the Control UI loaded in a local browser, plugins running in the same process — can connect. Remote clients cannot, unless the operator explicitly changes `gateway.bind` to `"lan"`, `"tailnet"`, or `"custom"`.

This is a "fail closed" posture: the safest configuration is the default configuration. An operator who wants network access must make an active choice and accept the surface they are opening.

S40 also explains the security motivation: the design aims to stay capable for real work while making risky paths explicit and operator-controlled.

The recommended path for exposing the Gateway to remote clients is Tailscale Serve, not a direct LAN bind:

> Prefer Tailscale Serve over LAN binds (Serve keeps the Gateway on loopback, and Tailscale handles access).

With Tailscale Serve, the Gateway itself never leaves loopback. Tailscale handles the remote transport and identity, and OpenClaw can verify the Tailscale-injected identity headers (`tailscale-user-login`) when `gateway.auth.allowTailscale` is enabled.

### What it trades away

Loopback-default means that the first time a remote client (a phone app, another machine on the LAN, a cloud deployment) tries to reach the Gateway, it will fail — because the Gateway is not listening on that interface yet. The operator must consciously open the Gateway to the network.

S40 documents what changes when you move beyond loopback:

- Non-loopback binds expand the attack surface. Only use them with gateway auth (shared token, password, or correctly configured trusted proxy) and a real firewall.
- Docker port publishing bypasses `iptables INPUT` rules, so container-based deployments need `DOCKER-USER` chain rules in addition to host firewall rules.
- If you bind to LAN, firewall the port to a tight allowlist of source IPs; do not port-forward it broadly.

There is also a proxy subtlety. When the Gateway detects proxy headers (`X-Forwarded-For`) from an address not listed in `gateway.trustedProxies`, it will not treat the connection as a local client. S40 notes:

> If gateway auth is disabled, those connections are rejected. This prevents authentication bypass where proxied connections would otherwise appear to come from localhost and receive automatic trust.

So the loopback default is not only about the bind address — it is a trust model: connections from loopback are treated as local-operator connections. Moving beyond loopback requires explicitly modeling and configuring that expanded trust.

---

## Decision 4: Plugins run in-process with the Gateway

### The problem

Plugins in OpenClaw can register tools, providers, channels, hooks, and services. They have deep access to the runtime: they intercept the agent loop, they can modify tool parameters before execution, they can inject context, they can talk to external services. This level of integration requires that a plugin can call into core runtime objects — the agent loop, the session state, the tool registry — synchronously and at low latency.

One alternative would be to run each plugin in its own subprocess (or even a separate container), communicating via IPC or an HTTP API. This would give strong process isolation: a crashed plugin would not crash the Gateway; a malicious plugin could only do what the IPC protocol allowed.

### The choice

S40 states this plainly:

> Plugins run **in-process** with the Gateway. Treat them as trusted code.

VISION.md elaborates on the reasoning:

> Use code plugins when the capability needs runtime hooks, providers, channels, tools, or other in-process extension points.

The "other in-process extension points" phrasing is key: the full plugin API — hooks, providers, channels — only makes sense when the plugin shares a process with the runtime it is extending. An out-of-process plugin can call remote APIs; it cannot register a `before_tool_call` hook that synchronously rewrites parameters before the model call happens, because there is no shared call stack.

The consequence is explicit in S40:

> Only install plugins from sources you trust.

And:

> If you install or update plugins (`openclaw plugins install <package>`, `openclaw plugins update <id>`), treat it like running untrusted code.

The source does not document a subprocess or sandboxed plugin runtime. There is no "plugin sandbox" separate from the Gateway process.

### What it trades away

Running plugins in-process means a plugin with a bug can crash the Gateway process. A plugin with malicious intent has access to everything the Gateway process can reach: the filesystem (subject to tool policy), the SQLite databases, the agent sessions, the loaded configuration, the network (subject to network policy).

S40 provides specific guidance on how to manage this:

> - Only install plugins from sources you trust.
> - Prefer explicit `plugins.allow` allowlists.
> - Review plugin config before enabling.
> - Restart the Gateway after plugin changes.
> - Prefer pinned, exact versions (`@scope/pkg@1.2.3`), and inspect the unpacked code on disk before enabling.

The operator-level mitigation is supply chain discipline: treat plugin installation like running untrusted code, because that is exactly what it is. This is a familiar model (npm, browser extensions, VS Code extensions all run in-process with trust delegated to the operator) but it does mean the Gateway's security boundary is only as strong as the plugins loaded into it.

VISION.md describes a preference ordering that acknowledges this:

> Prefer bundle-style plugins when they can express the capability. They have a smaller, more stable interface and better security boundaries. Use code plugins when the capability needs runtime hooks, providers, channels, tools, or other in-process extension points.

Bundle-style plugins (which package skills, MCP servers, and configuration, without running arbitrary code hooks) have a smaller blast radius than code plugins. The architecture steers operators toward bundle-style first, and requires code plugins only when the capability genuinely needs in-process runtime access.

---

## Decisions at a glance

| Decision | Problem solved | Choice | Key tradeoff |
|---|---|---|---|
| SQLite as runtime-state store | Scattered file formats, no single query surface, crash-inconsistency risk | All runtime state goes to one of two SQLite DBs; JSONL is reserved for session transcripts as named artifacts | Not suitable for multi-process or multi-host Gateway deployments |
| Exclusive memory plugin slot | Unpredictable behavior when two memory plugins compete for the same context-injection and tool-registration surface | The memory slot holds at most one active memory plugin | Cannot compose dreaming subagent with vector recall simultaneously |
| Gateway binds to loopback by default | A first-run Gateway exposed to the network before auth/pairing is configured is an immediate risk | `gateway.bind: "loopback"` by default; network access requires explicit operator opt-in | Remote clients cannot connect until the operator changes `gateway.bind` |
| Plugins run in-process | Full runtime hook API (hooks, providers, channels) requires sharing a call stack with the Gateway | Plugins run in the Gateway process; they are trusted code | A buggy or malicious plugin can affect the full Gateway process |

---

## A note on what is not here

<!-- GAP: The source documentation does not provide explicit rationale text for the SQLite-over-service choice beyond the architectural rules in AGENTS.md. The following framing ("single-process, single-host") is inferred from the stated constraint against multi-process access and the documented storage rules, not from a dedicated design-rationale document. -->

The sources — VISION.md, AGENTS.md, the security index, and the sandboxing document — state each rule and its immediate enforcement mechanism clearly. What they generally do not provide is the historical narrative behind each decision ("we tried X and it failed because Y"). Where this chapter says "the problem was…" it is reconstructing the motivation from the rule's shape and from what the source documents say the rule prevents. The rationale for SQLite, in particular, is architectural inference from the documented engineering policy; a dedicated "why SQLite?" design document does not appear in the assigned sources.

The tradeoffs described here are grounded: they follow directly from what the source says the rule is and what the alternative would require. No benchmarks, throughput numbers, or "scales to N" claims are made, because the sources do not make them.

---

← Previous: [End-to-End Walkthroughs](./24-walkthroughs.md) · Next: [Best Practices](./26-best-practices.md) →
