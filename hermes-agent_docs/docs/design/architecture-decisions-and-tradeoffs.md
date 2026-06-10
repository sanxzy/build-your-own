---
title: Architecture Decisions and Tradeoffs
description: Why Hermes is shaped the way it is — six major design choices, the alternatives that existed, and what each costs and gains.
category: design
type: explanation
tags:
  - architecture decisions
  - tradeoffs
  - synchronous loop
  - async agent
  - predictability
  - debuggability
  - SQLite
  - WAL
  - portability
  - NFS fallback
  - five-layer memory
  - memory architecture
  - vector store
  - OS security boundary
  - in-process heuristics
  - config-driven routing
  - provider-agnostic
  - profiles
  - multi-tenancy
  - isolation
  - scalability
  - design rationale
  - hermes agent
  - nous research
keywords:
  - hermes design decisions
  - agent loop tradeoffs
  - sqlite wal agent
  - memory layer architecture
  - os boundary security
  - model selection routing
  - profile isolation multi-tenant
sources: [S1, S3, S4, S6, S7, S15]
---

**TL;DR** — Every design is a tradeoff: something gained, something given up. This chapter walks through six choices that define Hermes's shape — the synchronous loop, SQLite as the session store, five memory layers, OS-only security, config-driven model routing, and profiles as multi-tenancy — explaining the reasoning behind each, the alternatives that existed, and the honest costs an operator carries forward.

# Architecture Decisions and Tradeoffs

A system's architecture is the sum of its tradeoffs. Understanding *why* Hermes was built the way it was helps you judge whether those tradeoffs suit your situation — and tells you where to push if they do not.

This chapter covers six design decisions. For each one, we'll look at:

- The problem the choice was solving.
- What was chosen, and why.
- The alternative that was passed over.
- What the chosen path wins and what it costs.
- What the costs imply for scalability, maintainability, and operations.

At the end there is a summary comparison table.

---

## A quick vocabulary check

Before we dive in, a few terms that appear throughout:

- **Synchronous** (sync): operations happen one at a time, in order. You do step A, wait for it to finish, then do step B. Nothing runs in parallel.
- **Asynchronous** (async): operations can be handed off and run concurrently. You start step A, hand it to another thread or event loop, and the main program continues while A is running.
- **SQLite**: a self-contained database engine that stores everything in a single file on disk. No separate server process required.
- **WAL** (Write-Ahead Log): a SQLite mode where writes are first appended to a log file before being applied. This allows readers to continue reading the database while a write is in progress.
- **Vector store**: a database optimised for finding items by semantic similarity (how closely two pieces of text *mean* the same thing) rather than by exact text matching.
- **Multi-tenancy**: running multiple independent users or teams on the same system, with their data and behaviour kept separate.

---

## Decision 1 — Synchronous agent loop with interrupt checks

### The problem

An agent loop is the engine at the heart of Hermes: it fires an LLM call, receives a response, dispatches any tool calls, collects results, then repeats until the agent is done. The central design question is how that engine is structured. You could build it as a fully asynchronous system — every API call and tool execution runs concurrently on an event loop, and callbacks fire as results arrive. Or you could run the whole thing synchronously, turn by turn.

Hermes chose the synchronous path.

### What Hermes does

The `run_conversation()` function in `agent/conversation_loop.py` is the agent's main loop. It is a `while` loop — a plain sequential structure. Each iteration checks for an interrupt, consumes one unit of the iteration budget, fires one LLM API call, dispatches the resulting tool calls (up to `_MAX_TOOL_WORKERS = 8` concurrent threads for parallel tool execution, sourced from `run_agent.py`), collects results, appends them to the conversation history, and goes around again. The loop advances in lockstep:

```python
# Simplified view of the loop condition in agent/conversation_loop.py:
while (
    api_call_count < agent.max_iterations
    and agent.iteration_budget.remaining > 0
) or agent._budget_grace_call:

    if agent._interrupt_requested:
        interrupted = True
        break

    # ... fire one LLM call, dispatch tools, collect results ...
    api_call_count += 1
```

The interrupt check (`agent._interrupt_requested`) runs at the top of every iteration. That means a user-sent `/stop` can only land between iterations — not mid-API-call. The loop drains its current step, reaches the top, sees the flag, and exits cleanly.

Tool calls within a single iteration are dispatched to a thread pool (up to eight workers), so multiple tools that were requested in the same LLM response run in parallel. But the *loop itself* — the sequence of LLM call → tool results → next LLM call — is synchronous. There is one conversation turn happening at a time.

For a deeper look at how the loop runs from a reader's perspective, see [AIAgent and the Conversation Loop](../core-runtime/aiagent-and-conversation-loop.md).

### The alternative: a fully async event-driven loop

The main alternative is an async-native loop built on Python's `asyncio` event loop. In that model, the agent would `await` each LLM call, and other coroutines (interrupt handlers, streaming callbacks, memory sync) could run on the same thread while the LLM is thinking. Many modern agent frameworks take this approach.

### Tradeoff analysis

| What synchronous wins | What synchronous costs |
|---|---|
| A linear call stack — you can set a breakpoint at any point in the loop and inspect full state | Cannot interleave work on other tasks while waiting for one LLM call (but Hermes uses separate processes/profiles for multi-task work) |
| Deterministic ordering: tool calls from the same LLM response always complete before the next LLM call | Interrupt latency: a `/stop` takes until the current iteration completes to take effect |
| No need for `asyncio.Lock` or `asyncio.Queue` to protect conversation history mutations | An async-native approach could, in principle, pipeline the memory sync with the next LLM call, reducing latency slightly |
| Exceptions propagate naturally up the call stack without requiring `asyncio.gather` error handling | Synchronous tool dispatch means the loop thread blocks if the thread pool is full |
| Full compatibility with third-party synchronous libraries and debuggers | |

The key insight is that the bottleneck in an agent loop is almost always the LLM API call itself — measured in seconds. Shaving microseconds from the Python call overhead with async does not improve user-perceived latency. What does matter is being able to debug, reason about, and interrupt the loop reliably. The synchronous choice serves those priorities.

For operations: this means Hermes scales horizontally (run multiple profiles, each with its own synchronous loop) rather than vertically (one loop juggling many concurrent sessions). If you need many simultaneous users, you run many profiles or instances.

---

## Decision 2 — SQLite with WAL as the primary session store

### The problem

Every conversation turn — user message, model response, tool calls, results — needs to be persisted. It needs to be searchable (so the agent can recall past sessions). It needs to be readable by multiple processes simultaneously (the CLI, the gateway, and background workers may all be reading session data at the same time). And it needs to work on a $5 VPS with no infrastructure beyond what comes on the machine.

### What Hermes does

The `SessionDB` class (`hermes_state.py`) stores all session data in a single SQLite file: `~/.hermes/state.db` (or the equivalent profile-aware path). The schema is at version 15 (`SCHEMA_VERSION = 15`). SQLite runs in WAL mode, set by `apply_wal_with_fallback()`. WAL allows one writer and multiple concurrent readers without blocking either, which is the right shape for a gateway that may be streaming to Telegram while the CLI is browsing history.

The `SessionDB` also builds two FTS5 (full-text search) virtual tables: a word-boundary index and a trigram index. These back the session-search feature that lets the agent search its own past conversations — a core part of the closed learning loop.

```python
# hermes_state.py — the schema version constant and the WAL helper:
SCHEMA_VERSION = 15

def apply_wal_with_fallback(conn, *, db_label="state.db") -> str:
    """Set journal_mode=WAL, falling back to DELETE on NFS/SMB/FUSE."""
    # ... tries WAL, catches OperationalError("locking protocol"),
    # falls back to journal_mode=DELETE if WAL is unsupported.
```

The WAL fallback is real operational engineering: some deployment environments — network-mounted home directories (NFS/SMB/CIFS), some FUSE mounts, WSL1 — do not support the shared-memory coordination WAL requires. Rather than failing at startup, `apply_wal_with_fallback()` detects the error (an `OperationalError: locking protocol`) and falls back to `journal_mode=DELETE`, the pre-WAL default. Concurrency drops (readers are blocked during writes), but the feature works. The warning fires once per process per database, so NFS users do not get their logs spammed.

For more detail on `SessionDB` and its FTS5 search capabilities, see [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

### The alternative: a server-side database

The natural alternative is a networked database server — PostgreSQL, MySQL, or a cloud-managed equivalent. A server-side database offers higher write throughput, more sophisticated query planning, and better concurrency primitives. It is the standard choice for production web applications.

### Tradeoff analysis

| What SQLite + WAL wins | What SQLite + WAL costs |
|---|---|
| Zero infrastructure: nothing to install, no port to open, no service to manage | Write throughput is limited by a single writer; not suited for very high-volume concurrent writes |
| The whole session store moves with `~/.hermes/` — a profile is a directory you can copy, archive, or delete | WAL mode does not work on NFS/SMB/FUSE (the fallback handles it but loses concurrency) |
| Works offline; no network required | Multi-host deployments that need a shared session store cannot use the local file |
| The FTS5 and trigram search indexes are built in — no separate search engine required | Schema migrations must be backward-compatible; `SCHEMA_VERSION = 15` reflects a long evolution |
| Backup is `cp state.db backup.db` — no pg_dump required | |
| Consistent with the "runs anywhere" identity: a $5 VPS has no room for a PostgreSQL server | |

For operations: SQLite is an excellent choice for single-machine or single-profile deployments. If you need to share session history across multiple machines, you need either a shared network filesystem (which loses WAL concurrency) or a different persistence strategy. The profile model (each profile is a self-contained directory) scales horizontally — one profile per machine, coordinated via the kanban board — rather than via a shared session database.

---

## Decision 3 — Five-layer memory architecture

### The problem

Memory in an AI agent is genuinely hard. You need something the agent can see *right now* in its context window. You need something that persists across turns within a session. You need something that persists *across sessions* — so the agent remembers facts from a conversation three weeks ago. You need something that survives in a form the agent can act on procedurally. And you need all of this to stay within the LLM's context budget, which is finite and expensive.

A single vector store — a database where you embed everything as a high-dimensional vector and search by similarity — sounds like a clean solution. But it turns out to be insufficient on its own.

### What Hermes does

Hermes separates memory into five distinct layers, each with its own persistence boundary and retrieval mechanism:

1. **In-context conversation window** — the live message history the model reads on every call. Ephemeral: exists for the duration of the current session only.
2. **`ContextCompressor`** (`agent/context_compressor.py`) — when the conversation grows too long, the compressor summarises older turns and stores them as a compressed context block. The agent retains `lcm_grep`, `lcm_describe`, and `lcm_expand` tools to navigate compressed history. When compression triggers, a new session is created with a `parent_session_id` chain linking back to the original.
3. **`MemoryManager`** (`agent/memory_manager.py`) — the orchestrator that runs `prefetch_all()` before each turn (injecting recalled facts into a `<memory-context>` block) and `sync_all()` after each turn (on a background thread). Connects to external memory providers: Honcho, Hindsight, or Mem0.
4. **`SessionDB`** — the full conversation archive. Searchable across all past sessions via FTS5 and trigram indexes. The agent can search its own history explicitly when it needs to recall something specific.
5. **Skills** — persistent learned procedures. When the agent creates a skill via `skill_manage`, that knowledge survives indefinitely as a file in `~/.hermes/skills/`. Skills are the highest-fidelity form of memory: they encode not only facts but reusable procedures.

This architecture is the structural reason for Hermes's defining capability: the closed learning loop. Conversation → `ContextCompressor` keeps context manageable → `MemoryManager` persists important facts → `SessionDB` makes all history searchable → skills encode durable procedures the Curator maintains.

<!-- GAP: The brief does not state a specific numeric token budget per layer or a specific compression threshold; source S7 and S1 describe the mechanism but do not give a universal default token count that triggers compression — needed for a precise "compression fires at N tokens" claim. -->

### The alternative: a single vector store

The simpler alternative is to embed every message into a vector store and retrieve the top-K most relevant past messages by semantic similarity. This is how many agent memory systems are built.

### Tradeoff analysis

| What five layers win | What five layers cost |
|---|---|
| Each layer is matched to a retrieval problem: exact-phrase search (FTS5), semantic similarity (MemoryManager/external provider), procedural recall (skills) | Five layers to reason about, configure, and maintain |
| The learning loop is architecturally visible: skills are the durable output; the other layers feed their creation | External memory providers (Honcho, Mem0, Hindsight) require setup and API keys |
| The agent can selectively search past conversations rather than always loading a context window full of retrieved fragments | If `MemoryManager` is misconfigured, recall quality degrades silently rather than failing loudly |
| Compression keeps context costs predictable — the window stays bounded even in very long sessions | The `parent_session_id` chain from compression must be followed correctly to reconstruct full history |
| Skills provide recall without token cost at inference time (loaded at startup, not retrieved per turn) | |

For operations: the five-layer architecture rewards configuration effort. An operator running Hermes for personal use may be fine with the built-in layer only. A team deployment, or a research use case that needs rich cross-session recall, should configure an external memory provider. The layers are not redundant — they complement each other. Removing any one degrades a specific kind of recall.

---

## Decision 4 — OS as the only security boundary

### The problem

An AI agent that can run shell commands, write files, and call APIs is operating at a high privilege level. It receives input from sources that may be attacker-influenced — injected content in a web page it fetches, a malicious email in its inbox, a crafted gateway message. How do you constrain what it can do?

The honest answer is uncomfortable: any mechanism *inside* the agent process is operating on attacker-influenced strings, and a sophisticated attacker will find a way around it. Hermes's security policy names this directly.

### What Hermes does

The `SECURITY.md` policy states the foundational principle verbatim:

> **The only security boundary against an adversarial LLM is the operating system. Nothing inside the agent process constitutes containment — not the approval gate, not output redaction, not any pattern scanner, not any tool allowlist.**

Hermes supports two OS-level isolation postures, each addressing a different threat:

**Terminal-backend isolation.** A non-default terminal backend (Docker, SSH, Singularity, Modal, Daytona) runs LLM-emitted shell commands inside a container, remote host, or cloud sandbox. What this *confines*: anything the agent does by issuing shell or file operations. What this does *not* confine: everything the agent does in its own Python process — plugin loading, skill loading, MCP subprocess spawning, the code-execution tool.

**Whole-process wrapping.** The entire agent process tree runs inside a sandbox — Hermes's own Docker image and Compose setup, or NVIDIA OpenShell, which adds per-session sandboxes, declarative L7 egress policy, and process/syscall policy. This is the supported posture when the agent ingests content from surfaces the operator does not control.

The in-process tools (`SECURITY.md` §2.4) are explicitly labelled *heuristics*:
- The **approval gate** detects common destructive shell patterns and prompts the operator. "Shell is Turing-complete; a denylist over shell strings is structurally incomplete."
- **Output redaction** strips secret-like patterns from display. A motivated output producer will defeat it.
- **Skills Guard** scans installable skill content for injection patterns. It is a review aid; the boundary for third-party skills is operator review before install.

For the security posture in full detail, see [OS Boundary and Isolation Postures](../security/os-boundary-and-isolation-postures.md).

### The alternative: in-process sandboxing

The alternative — and the path many agent frameworks take — is to build more elaborate in-process mechanisms: Python `RestrictedPython` sandboxes, finer-grained tool permission systems, runtime call graph analysis, or intent-classification filters. These are more transparent to users (no container overhead), easier to configure, and require less infrastructure.

### Tradeoff analysis

| What OS-only boundary wins | What OS-only boundary costs |
|---|---|
| The guarantee is actually load-bearing: an OS-level sandbox confines the agent regardless of how clever the LLM output is | Requires real infrastructure: a container runtime, a remote execution environment, or access to OpenShell |
| The security model is honest — no false assurance that in-process filtering is "secure" | In-process heuristics still need to exist as accident-prevention; their limited scope must be explained to operators |
| Operators can reason clearly: if you run the local backend with untrusted input, you have accepted the trust envelope of the OS user | Running the default local backend on untrusted workloads is an out-of-scope posture, which shifts responsibility to operators |
| Whole-process wrapping confines plugin loading, MCP subprocesses, and skill loading — code paths that terminal-backend isolation leaves exposed | Docker Compose setup adds operational overhead for operators who only want to use Hermes locally |

The honest framing demands something of operators: you must choose a posture deliberately. The documentation and `SECURITY.md` §3.2 make clear that "running the default local backend with untrusted input surfaces" is outside the supported security posture. This is not a gap — it is an explicit stance. Operators who run Hermes in a shared or production setting *must* use a whole-process sandbox.

For operations: the choice of isolation posture is the most important deployment decision. Match the posture to the trust level of the content the agent will ingest.

---

## Decision 5 — Provider-agnostic, config-driven model selection

### The problem

Different LLM providers have different pricing, performance, context limits, and availability. A researcher might want to benchmark multiple providers against the same task. A team might switch providers as pricing changes. An operator running the agent autonomously might want to use a cheaper model for routine tasks and a more capable model for complex ones. How do you handle that without baking provider-specific code into the agent?

### What Hermes does

Model selection is fully config-driven. The operator sets `model.provider` and `model.model` in `config.yaml`, and `api_mode` to select the transport protocol. Hermes bundles approximately 30 providers as plugins under `plugins/model-providers/`, each providing a `ProviderProfile` dataclass. Switching providers is a config file edit and a `hermes model` command — no code changes.

The `CredentialPool` (`agent/credential_pool.py`) handles failover: it holds multiple API keys for the same provider and rotates among them using one of four strategies (`fill_first`, `round_robin`, `random`, `least_used`). If a key hits a 401 error, the pool places it in a 5-minute cooldown. A 429 or 402 triggers a 1-hour cooldown. Keys marked `STATUS_DEAD` are excluded from rotation permanently.

The four `api_mode` values correspond to distinct wire protocols:

| `api_mode` | Used for |
|---|---|
| `chat_completions` | Default; the OpenAI-compatible chat completions endpoint |
| `anthropic_messages` | Anthropic native Messages API |
| `codex_responses` | OpenAI Codex Responses API |
| `bedrock_converse` | AWS Bedrock Converse API (via `boto3`) |

Provider-specific adapters exist for Gemini (bypasses the OpenAI-compat layer entirely), Bedrock (uses `boto3`, not HTTP), and Anthropic (three auth variants: API key, OAuth setup-token, Claude Code credential). None of this is visible to the operator under normal use — `api_mode` selects the right adapter automatically.

For the full routing mechanism, see [Config-Driven Routing and API Modes](../providers/config-driven-routing-and-api-modes.md).

### The alternative: a learned router

The alternative is a dynamic, learned router that observes task characteristics and automatically selects the best model for each request — routing simple questions to a cheap model and complex tool-calling tasks to a more capable one, based on feedback signals.

### Tradeoff analysis

| What config-driven routing wins | What config-driven routing costs |
|---|---|
| Behaviour is fully predictable: the same task always uses the same model unless the operator changes the config | Operator must make routing decisions explicitly; there is no automatic "use the cheapest model that can handle this" |
| No data required to train a router; works from the first request | Adapting to new models or providers still requires a config change, not improved routing data |
| The operator retains full control over cost and capability tradeoffs — no model makes routing decisions autonomously | Sophisticated cost optimisation (route by task type, time of day, credit balance) must be built outside Hermes |
| Provider failover via `CredentialPool` is transparent: a failed key does not surface as a user-visible error | |
| `STATUS_DEAD` ensures permanently invalid keys never cause retries | |

The config-driven approach is consistent with the "provider-agnostic" identity. The claim "switch with `hermes model`, no code changes" is only true if the routing layer itself is purely config-driven. A learned router would introduce implicit coupling to training data and historical task distributions, making the provider-switching story much messier.

For operations: the config-driven approach rewards operators who invest in configuring their credential pool with multiple keys. Redundancy and cost control come from pool strategy configuration, not from automatic routing.

---

## Decision 6 — Profiles as the multi-tenancy mechanism

### The problem

Hermes is described as a "single-tenant personal agent" in `SECURITY.md`, but there are many legitimate use cases for running multiple independent agent instances: a developer who wants separate agents for work and personal use; a team that needs each member's agent to have its own memory, keys, and skills; a multi-agent system where each kanban worker profile should not be able to see another's conversation history or credentials.

How do you provide that isolation without building a user/organisation management layer?

### What Hermes does

A **profile** in Hermes is a named override of `HERMES_HOME`. The `_apply_profile_override()` function in `hermes_cli/main.py` sets the `HERMES_HOME` environment variable before any module imports, so all `get_hermes_home()` calls throughout the codebase automatically scope to the active profile's directory.

A profile named `coder` lives at `~/.hermes/profiles/coder/`. It has its own `config.yaml`, `.env`, `skills/`, `sessions/`, `kanban/`, `cron/`, and `logs/`. From inside the profile, this is `~/.hermes/` — the profile is completely transparent to all the code that uses `get_hermes_home()`.

```bash
# Start an agent session in the "coder" profile:
hermes -p coder

# List all profiles from any active profile:
hermes profile list
```

You can run multiple profiles simultaneously, each with a different provider, model, skills set, and gateway configuration. Kanban workers are isolated at the board level via the `HERMES_KANBAN_BOARD` environment variable, which pins each spawned worker to its own board — workers cannot see tasks on other boards.

For profiles in more depth, see the chapter on [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md) for state isolation, and [Config-Driven Routing](../providers/config-driven-routing-and-api-modes.md) for per-profile provider configuration.

### The alternative: a user/organisation model

The standard alternative is a proper user and organisation abstraction: a user table, per-user credential scopes, per-org role-based access control, and a shared database with row-level security. This is the architecture of multi-tenant SaaS products.

### Tradeoff analysis

| What profiles win | What profiles cost |
|---|---|
| No user management infrastructure: no database of users, no auth service, no permission system | Each profile is a full Hermes instance; they do not share credentials, plugins, or configuration |
| Isolation is complete at the OS level: one profile cannot read another's `state.db` or `.env` without OS-level access | Sharing a skill across profiles requires copying it to each profile's `skills/` directory (or using a hub-installed shared skill) |
| Creating a new isolated agent is `hermes profile create <name>` — no database migration | For very large teams, managing dozens of profiles is operational overhead |
| Profile-as-directory means backup, archival, and deletion are simple file operations | There is no per-user permission model within a single agent; all callers on the same allowlist have equal access |
| Profiles compose cleanly with kanban boards: a worker profile is a profile running the kanban dispatcher role | |

The profile mechanism is consistent with Hermes's "runs anywhere" identity. A user/org model requires a running database server, a migration story, and a credential management system. Profiles require only a filesystem. On a $5 VPS, the profile model is the only practical option.

For operations: if you genuinely need fine-grained per-user access control within a single deployment — for example, giving one user access to certain tools but not others — you are at the edge of what profiles support. Each user should get their own profile (and therefore their own Hermes instance), and authorization is enforced at the gateway adapter's allowlist level, not inside the agent. This is an honest limitation: Hermes does not model per-caller capabilities inside a single adapter.

---

## Summary decision table

| Decision | Chosen | Alternative passed over | Primary wins | Primary costs |
|---|---|---|---|---|
| Agent loop structure | Synchronous with interrupt checks | Fully async event loop | Predictable call stack, easy debugging, clean interrupt | No concurrent sessions per instance; interrupt waits for current iteration |
| Session persistence | SQLite + WAL (`SessionDB`) | Server-side database (PostgreSQL etc.) | Zero infrastructure, portable, built-in FTS search | Single writer; WAL unsupported on NFS (fallback exists) |
| Memory architecture | Five distinct layers | Single vector store | Matched retrieval per problem type; enables the learning loop | Configuration complexity; external providers need setup |
| Security boundary | OS isolation (terminal backend or whole-process wrap) | In-process sandboxing | Load-bearing containment; honest guarantee | Requires infrastructure; operator must choose posture deliberately |
| Model selection | Config-driven (`model.provider` + `api_mode` + `CredentialPool`) | Learned dynamic router | Predictable, controllable, provider-agnostic from day one | No automatic cost optimisation; operator carries routing decisions |
| Multi-tenancy | Profiles (`HERMES_HOME` override, one directory per instance) | User/org model with RBAC | Zero infrastructure; OS-level isolation; composable with kanban boards | No per-caller permission model within one agent |

---

## How the decisions fit together

These six decisions are not independent choices — they reinforce each other.

**SQLite portability enables the profile model.** If the session store required a server, each profile would need a server. Because it is a file, a profile is a directory, and you can spin up or tear down a profile without infrastructure changes.

**The profile model enables the "runs anywhere" identity.** A $5 VPS can run ten profiles with no external dependencies. A GPU cluster can run them at higher throughput. Serverless deployments can spin profiles up on demand. None of this requires a backend service.

**Config-driven routing makes provider-agnostic credible.** If routing were learned or dynamic, switching providers would require retraining the router or accumulating new data. Because it is pure config, you switch by editing one line.

**The OS security boundary is honest precisely because in-process heuristics are not.** Labelling the approval gate and Skills Guard as heuristics — not boundaries — is what allows the OS posture statement to be believed. A system that overstates the strength of its in-process tools trains operators to trust them in contexts where they should not.

**The synchronous loop and the five-layer memory architecture both serve the learning loop.** The loop's predictable ordering means `MemoryManager.sync_all()` runs reliably after every turn. Skills are created at well-defined points within the loop. There are no races between memory writes and the next turn's reads. The closed learning loop that defines Hermes's identity depends on this orderly execution.

---

← Previous: [Scenario 5 — Human-in-the-Loop Approval Workflow](../scenarios/human-in-the-loop-approval.md) · Next: [Best Practices — Delegation, Budgets, Credentials, Memory, and Skill Authoring](./best-practices.md) →
