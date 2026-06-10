---
title: "Glossary — LLM, Kanban, SQLite, WAL, CAS, Toolset, and More"
description: A plain-language reference defining every technical term used across the Hermes documentation library, with links to the chapters that teach each one in depth.
category: reference
type: reference
tags: [glossary, LLM, kanban, SQLite, WAL, async, OAuth, CAS, toolset, profile, FTS5, trigram, dispatcher, skill, session, iteration budget, context compression, swarm, blackboard, api_mode, CredentialPool, observer hook, middleware, MCP, ACP, HMAC, webhook, cron, DAG, delegation, gateway, platform adapter, DM pairing, provider, failover, cooldown, terminal backend, memory layer, context window, SessionDB, background thread, Skills Hub, Curator, plugin, AIAgent, run_conversation, IterationBudget, MemoryManager, ContextCompressor, ToolCallGuardrailController, create_swarm, dispatch_once, claim_task, VALID_STATUSES, PooledCredential, STATUS_DEAD]
keywords: [hermes glossary, agent terminology, kanban terms, SQLite WAL explanation, compare-and-swap, credential rotation, FTS5 full-text search, trigram tokenizer, swarm topology, blackboard pattern, observer pattern, middleware chain, agentskills.io, skill curator, background review, NFS WAL fallback, compression lock, parent session chain, api mode chat completions, fill first round robin, platform enum, DM pair code, HMAC webhook, cron scheduler tick, task status machine]
sources: [S1, S2, S15, S21, S24]
---

**TL;DR** — This page defines every technical term the Hermes documentation uses. You do not need to memorize it. Skim it now to get your bearings, then return here whenever a term you encounter in another chapter trips you up. Each entry gives you the plain-language meaning first, then tells you exactly how Hermes uses it and links to the chapter where it is taught in full.

# Glossary — LLM, Kanban, SQLite, WAL, CAS, Toolset, and More

Hermes draws on ideas from databases, operating systems, distributed systems, and machine learning. If you are new to one of those areas, that is fine — this page exists precisely so that every term is defined before you need it. Terms are grouped thematically rather than alphabetically so that related ideas sit next to each other; use your browser's search (`Ctrl/Cmd+F`) to jump to any specific word.

> **Prerequisite recap:** The rest of this library assumes you have read [What Hermes Is](../getting-started/what-is-hermes.md), which introduces Hermes's core identity: a self-improving AI agent with a closed learning loop. If you have not read it yet, that is the right first stop. The terms below will make more sense after that one-page introduction.

---

## The Agent and Its Loop

### AI agent

An **AI agent** is a program that takes input (usually a message from a person or another system), decides on a course of action, executes that action through tools, observes the result, and then decides what to do next — repeating until it reaches a goal or runs out of budget. Unlike a simple chatbot, which just produces text, an agent can write files, run shell commands, call APIs, and spawn other agents.

**In Hermes:** the central runtime object is the `AIAgent` class (`run_agent.py`). One `AIAgent` instance manages one conversation: it holds the message history, the iteration budget, the active toolset, and the current model configuration. See [The `AIAgent` Class and Conversation Loop](../core-runtime/aiagent-and-conversation-loop.md).

---

### LLM (Large Language Model)

An **LLM** is a neural network trained on large amounts of text. It learns statistical patterns in language well enough to generate coherent, contextually relevant responses. Common examples include models from Anthropic (Claude), OpenAI (GPT), and Google (Gemini). An LLM is the "brain" inside an agent: the agent calls the LLM with a prompt, receives a response (which may include instructions to call tools), and then acts on that response.

**In Hermes:** Hermes is provider-agnostic. It supports approximately 30 bundled LLM providers and routes to them through the configured `model.provider` and `model.model` settings. You switch providers with `hermes model` — no code changes required. See [Config-Driven Routing and API Modes](../providers/config-driven-routing-and-api-modes.md).

---

### tool / tool-calling

A **tool** is a function the LLM is allowed to call during a conversation turn. The LLM does not execute tools itself; it emits a structured "I want to call this function with these arguments" response, and the agent runtime executes the function and returns the result as a new message. This mechanism is sometimes called "function calling."

Tools give the LLM a way to affect the world: read a file, run a shell command, search the web, query a database, create a kanban task, or delegate work to another agent.

**In Hermes:** tools are registered in `tools/registry.py`. Each tool has a name, description (the LLM reads this to decide when to use it), and parameter schema. See [The `AIAgent` Class and Conversation Loop](../core-runtime/aiagent-and-conversation-loop.md).

---

### toolset

A **toolset** is a named group of related tools that can be enabled or disabled together. Instead of configuring tools one by one, you configure toolsets — for example, enabling `coding` gives the agent file-editing and shell tools, while enabling `kanban` gives it task-management tools.

**In Hermes:** toolsets are defined in the `TOOLSETS` dictionary in `toolsets.py`. You configure active toolsets per platform in `config.yaml` or interactively with `hermes tools`. See [Iteration Budget and Toolsets](../core-runtime/iteration-budget-and-toolsets.md).

---

### agent loop / `run_conversation()`

The **agent loop** is the repeated cycle an `AIAgent` runs to handle a single user request: check for interrupts → consume one iteration from the budget → call the LLM → parse the response → dispatch tool calls → loop back. It continues until the LLM produces a final answer (no more tool calls), the iteration budget is exhausted, or an interrupt is received.

`run_conversation()` is the Python function that implements this loop (`run_agent.py`).

**In Hermes:** the loop is where every tool call, model call, and memory operation happens. Understanding it is the foundation for understanding everything else. See [The `AIAgent` Class and Conversation Loop](../core-runtime/aiagent-and-conversation-loop.md).

---

### iteration budget / `IterationBudget`

An **iteration** is one pass through the agent loop — one LLM call plus whatever tool calls the LLM requested. An **iteration budget** is a cap on how many iterations the agent is allowed before it must stop and return control to the user.

The budget prevents an agent from looping indefinitely on a stuck task. When the budget reaches zero, the agent gets one final "grace call" to produce a summary before stopping.

**In Hermes:** `IterationBudget` is the class (`agent/iteration_budget.py`) that tracks this. The grace call is `_budget_grace_call`. See [Iteration Budget and Toolsets](../core-runtime/iteration-budget-and-toolsets.md).

---

## Memory

### memory layer

A **memory layer** is one of the distinct systems Hermes uses to persist information across time. Different layers have different scopes (within a turn, across turns, across sessions) and different retrieval mechanisms (exact lookup, similarity search, full-text search).

**In Hermes:** there are five memory layers:

| Layer | What it holds | Scope |
|---|---|---|
| In-context conversation window | The live message history the LLM sees | Current turn |
| `ContextCompressor` | A searchable compressed representation of prior messages | Current session |
| `MemoryManager` | Long-term facts about the user and world | Across sessions |
| `SessionDB` | Full conversation archive, all sessions | Permanent |
| Skills | Procedural instructions learned from experience | Permanent |

See [The Five Memory Layers](../memory/five-memory-layers.md) for the full treatment.

---

### context window

The **context window** is the maximum amount of text (measured in tokens — roughly word-pieces) an LLM can process at once. Everything the LLM "sees" in a single call — the system prompt, the conversation history, the tool results — must fit inside this window.

As a conversation grows, it approaches the context window limit. This is the problem that context compression exists to solve.

**In Hermes:** the context window is managed by the `ContextEngine` class. When it fills, `ContextCompressor` steps in. See [The Five Memory Layers](../memory/five-memory-layers.md).

---

### context compression / LCM

**Context compression** is the process of replacing a growing conversation history with a compact, searchable summary so that the conversation can continue without hitting the context window limit. The compressed history is not discarded — it is stored in a retrieval layer so the agent can look up earlier details on demand.

**LCM** (Layered Context Manager) is what Hermes calls its compression approach. After compression, the agent can use three tools to work with the compressed history: `lcm_grep` (keyword search), `lcm_describe` (get a summary of a portion), and `lcm_expand` (retrieve the full text of a section). When compression triggers, Hermes splits the current session into a child session linked to the parent by `parent_session_id`, and acquires a 300-second compression lock to prevent two processes from compressing the same session simultaneously.

**In Hermes:** implemented in `agent/context_compressor.py` and `agent/context_engine.py`. See [Context Compressor and LCM](../memory/context-compressor-and-lcm.md).

---

### `MemoryManager`

`MemoryManager` is Hermes's orchestrator for long-term memory. Before each conversation turn, it calls `prefetch_all()` to pull relevant memories from external providers and inject them into the system prompt inside `<memory-context>` tags. After each turn, it calls `sync_all()` on a background thread to update the external providers with new information.

**In Hermes:** implemented in `agent/memory_manager.py`. External memory providers (Honcho, Mem0, Hindsight) plug in through the `MemoryProvider` abstract base class. See [The Five Memory Layers](../memory/five-memory-layers.md).

---

### session

A **session** is one continuous conversation between you and the agent, recorded from start to finish. It has a unique `session_id`, a source (which platform or interface started it), a model, and a full record of every message. When context compression triggers, the old session ends and a new child session begins, but both are linked via `parent_session_id` so the history chain is preserved.

**In Hermes:** sessions are stored in `SessionDB`, the SQLite database at `~/.hermes/state.db`. See [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

---

## Databases and Storage

### SQLite

**SQLite** is a file-based relational database. Unlike server databases (PostgreSQL, MySQL), SQLite stores the entire database in a single file on disk. You do not need to run a separate server process — the library is embedded directly into the application. This makes it ideal for local-first tools like Hermes.

**In Hermes:** Hermes uses SQLite for two key stores: `state.db` (sessions and messages, managed by `SessionDB`) and `kanban.db` (tasks and boards, managed by `hermes_cli/kanban_db.py`). The schema version of `state.db` is currently 15 (`SCHEMA_VERSION = 15`). See [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

---

### WAL (write-ahead logging)

**Write-ahead logging** is a durability technique used by SQLite. Instead of writing changes directly to the database file, SQLite first appends each change to a separate "write-ahead log" (WAL) file. Reads can proceed from the main file while writes happen in the WAL. Periodically, the WAL is checkpointed — written back into the main database file.

The benefit of WAL mode is that multiple readers can access the database at the same time without blocking each other, and a single writer does not block readers. This matters for Hermes because the gateway, the CLI, and background workers may all read the session database concurrently.

**The catch:** WAL mode requires shared-memory coordination that does not work reliably on network filesystems (NFS, SMB/CIFS, some FUSE mounts). Hermes handles this automatically via `apply_wal_with_fallback()` in `hermes_state.py`: if setting WAL mode fails with a "locking protocol" error, it falls back to the older `journal_mode=DELETE`. Functionality is preserved; concurrent-reader performance is reduced. See [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

---

### FTS5

**FTS5** (Full-Text Search version 5) is a SQLite extension that creates a virtual table you can query for words across all stored documents, much like a search engine index. It tokenizes text into words and builds an inverted index so that "find all messages containing the word 'authentication'" is fast regardless of how many messages exist.

**In Hermes:** `SessionDB` creates an FTS5 virtual table called `messages_fts` so you can search your entire conversation history. The table indexes message content plus tool names and tool call arguments, so you can find not just things the agent said but also things it did. See [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

---

### trigram

A **trigram** is a sequence of three consecutive characters. A **trigram tokenizer** breaks text into every possible three-character window (e.g., "hello" → "hel", "ell", "llo") instead of splitting on word boundaries. This means you can search for substrings within words, which is essential for languages like Chinese, Japanese, and Korean that do not use spaces between words.

**In Hermes:** `SessionDB` maintains a second FTS5 table called `messages_fts_trigram` using `tokenize='trigram'`. This table supports CJK substring search and partial-word matches that the word-boundary tokenizer in `messages_fts` would miss. See [SessionDB, FTS, and Search](../memory/sessiondb-fts-and-search.md).

---

## Profiles and Home Directory

### profile

A **profile** is a completely isolated Hermes identity: its own configuration file, API keys, memory, skills, session history, and kanban board participation. Each profile lives in its own subdirectory under `~/.hermes/profiles/<name>/`. Running `hermes -p <name>` starts Hermes with that profile's settings and state, completely separate from the default profile.

Profiles let you run multiple specialized agents on the same machine — for example, a coding agent and a research agent — without their memories or configurations mixing.

**In Hermes:** profile directories follow the same layout as the default home (`config.yaml`, `.env`, `skills/`, `sessions/`). The kanban board is shared across profiles by design: it lives at the parent directory level so any profile's worker can claim tasks. See [Home Directory and Profiles](../persistence/home-directory-and-profiles.md).

---

## Autonomy

### cron

**Cron** is the Unix convention for scheduling recurring tasks using time patterns. A cron expression like `0 9 * * 1-5` means "9:00 AM every weekday." Hermes has a built-in cron scheduler that fires tasks on a schedule without you being present.

**In Hermes:** the cron scheduler runs a `tick()` loop (`cron/scheduler.py`). On each tick it checks which jobs are due, starts agent sessions to handle them, and delivers the output to any configured platform (Telegram, email, etc.). The inactivity timeout for cron sessions defaults to 600 seconds (set by `HERMES_CRON_TIMEOUT`; `0` means unlimited). See [Cron Scheduler](../autonomy/cron-scheduler.md).

---

### webhook

A **webhook** is an HTTP callback: instead of polling "did something happen?", you register a URL and the external service calls that URL when an event occurs. Hermes can receive webhooks from GitHub or from any system that can make authenticated HTTP POST requests.

**In Hermes:** webhook triggers use HMAC authentication (see [HMAC](#hmac) below) to verify that the request genuinely came from the expected sender. See [Cron Scheduler](../autonomy/cron-scheduler.md).

---

### HMAC

**HMAC** (Hash-based Message Authentication Code) is a technique for verifying that a message came from someone who knows a shared secret key, and that the message was not tampered with in transit. The sender computes a hash of the message combined with the secret key and attaches it to the request. The receiver recomputes the hash and compares — if they match, the message is authentic.

**In Hermes:** webhook requests are authenticated with HMAC so that only senders who know the shared secret can trigger an agent run.

---

## Multi-Agent Coordination

### kanban

**Kanban** is a project-management method, originally from manufacturing, that represents work items as cards moving across columns on a board (typically: "to do" → "in progress" → "done"). In software, a kanban board is a shared coordination surface: team members (or agents) pick up cards, work on them, and mark them done.

**In Hermes:** the kanban board is used to coordinate multi-agent work. Tasks are represented as rows in a SQLite database, not physical cards. Agents claim tasks, work on them, and update their status. See [Kanban Dispatch](../multi-agent/kanban-dispatch.md).

---

### board

A **board** is one isolated kanban workspace with its own SQLite database. The default board lives at `~/.hermes/kanban.db`. Additional boards live at `~/.hermes/kanban/boards/<slug>/kanban.db` and are used to separate unrelated streams of work (e.g., one per project or repository).

**In Hermes:** you select the active board with `hermes kanban boards switch <slug>` or the `HERMES_KANBAN_BOARD` environment variable. Workers are pinned to the board that dispatched them via injected environment variables. See [Kanban Dispatch](../multi-agent/kanban-dispatch.md).

---

### task status / the nine statuses

Every task on a kanban board has a **status** that describes where it is in its lifecycle. Hermes defines exactly nine valid statuses in `VALID_STATUSES` (`hermes_cli/kanban_db.py`, line 100):

| Status | Meaning |
|---|---|
| `triage` | A rough idea, not yet decomposed or dependency-checked |
| `todo` | Dependencies not yet satisfied — waiting for parent tasks |
| `scheduled` | Waiting for a cron-specified time window |
| `ready` | All parents done; a dispatcher can claim and start it |
| `running` | A worker has claimed it and is executing |
| `blocked` | Explicitly paused by a `kanban_block` tool call; needs human or orchestrator attention |
| `review` | Awaiting a review gate before marking complete |
| `done` | Completed by a `kanban_complete` tool call |
| `archived` | Retained for history but no longer active |

See [The Nine-Status State Machine](../task-lifecycle/nine-status-state-machine.md) and [Kanban Dispatch](../multi-agent/kanban-dispatch.md).

---

### DAG (Directed Acyclic Graph)

A **DAG** is a set of nodes connected by directed edges (arrows that go one way) with no cycles (you can never follow arrows and end up back where you started). In task management, a DAG represents dependencies: task B depends on task A, so the arrow goes A → B, meaning A must complete before B can start.

**In Hermes:** tasks are linked in a parent/child DAG via the `task_links` table in `kanban.db`. The dispatcher uses `recompute_ready()` to promote tasks from `todo` to `ready` when all their parent tasks are in `done`. See [The Nine-Status State Machine](../task-lifecycle/nine-status-state-machine.md).

---

### dispatcher / `dispatch_once()`

The **dispatcher** is the component that watches the kanban board and starts worker agents to handle tasks that are ready. On every tick it calls `dispatch_once()` (`hermes_cli/kanban_db.py`, line 6025), which: (1) reclaims stale or crashed workers, (2) promotes `todo` tasks to `ready` via `recompute_ready()`, and (3) atomically claims each ready task and spawns a worker agent.

Only one gateway process should run the dispatcher (`kanban.dispatch_in_gateway: true` in `config.yaml`). A second gateway process with this flag set would race the first for claims, causing double-spawn.

**In Hermes:** the dispatcher runs in a background loop inside the gateway process. See [Kanban Dispatch](../multi-agent/kanban-dispatch.md) and [Cron Scheduler](../autonomy/cron-scheduler.md).

---

### CAS (compare-and-swap)

**Compare-and-swap** is an atomic database operation: "update this row IF it currently has value X; if it has any other value, do nothing and tell me." It is the building block for safe concurrent claim operations. Because SQLite serializes writers, a CAS operation is guaranteed to be atomic — at most one agent can win any given claim.

**In Hermes:** the `claim_task()` function uses CAS on `tasks.status` to atomically transition a task from `ready` to `running`. Any agent that tries to claim the same task after another agent has already claimed it observes zero affected rows and moves on. This is why there is no need for distributed-lock machinery. See [Kanban Dispatch](../multi-agent/kanban-dispatch.md).

---

### claim / claim TTL

When an agent **claims** a task, it writes its identity and a timestamp into the task's `claim_lock` field, transitioning the task to `running`. The claim is valid for a limited time — the **claim TTL** (time-to-live). If the worker does not finish or send a heartbeat within that window, the next dispatcher tick reclaims the task and returns it to `ready`.

**In Hermes:** the default claim TTL is 15 minutes (`DEFAULT_CLAIM_TTL_SECONDS = 15 * 60` in `hermes_cli/kanban_db.py`). Workers on long tasks should call `heartbeat_claim()` periodically. The TTL can be overridden with the `HERMES_KANBAN_CLAIM_TTL_SECONDS` environment variable. See [Kanban Dispatch](../multi-agent/kanban-dispatch.md).

---

### delegation

**Delegation** is the act of spawning a child agent — a sub-agent with its own isolated context and a restricted toolset — to handle a sub-task, then waiting for it to finish and using its result. Delegation is in-process: the child `AIAgent` runs inside the same Python process as the parent.

Delegation is different from kanban dispatch. Delegation is synchronous and in-process; kanban is asynchronous and cross-process. See [In-Process Delegation](../multi-agent/in-process-delegation.md).

**In Hermes:** delegation is performed with the `delegate_task` tool. The depth of nesting is capped at `delegation.max_spawn_depth` (default 1). See [In-Process Delegation](../multi-agent/in-process-delegation.md).

---

### swarm / `create_swarm()`

A **swarm** is a structured multi-agent topology created in one call: a planning root agent, a set of parallel worker agents, a verifier agent, and a synthesizer agent. All participants coordinate through the kanban board. The root task carries a `[swarm:blackboard]` comment that all workers and the synthesizer can read to share intermediate findings.

`create_swarm()` is the function in `hermes_cli/kanban_swarm.py` that creates this topology. It returns a `SwarmCreated` object with `root_id`, `worker_ids`, `verifier_id`, and `synthesizer_id`.

**In Hermes:** swarms are the way Hermes parallelizes work across many agents with a coordination structure. See [Swarm Topologies](../multi-agent/swarm-topologies.md).

---

### blackboard

In multi-agent systems, a **blackboard** is a shared data structure that all agents can read and write, used to share intermediate results without direct agent-to-agent communication. Agents post findings to the blackboard; other agents read them.

**In Hermes:** the swarm blackboard is implemented as a specially-formatted comment on the root kanban task, identified by the `[swarm:blackboard]` marker. Any worker can update it with `kanban_complete` carrying structured JSON, and the synthesizer reads it to assemble the final result. See [Swarm Topologies](../multi-agent/swarm-topologies.md).

---

## Providers and Credentials

### provider

A **provider** is a service that hosts LLM models and exposes an API. Examples include Anthropic, OpenAI, OpenRouter, Google Gemini, AWS Bedrock, and Nous Research's own portal. Hermes bundles support for approximately 30 providers and lets you switch between them with `hermes model` — no code changes required.

**In Hermes:** each provider has a `ProviderProfile` dataclass (`providers/base.py`, line 38) and a plugin directory under `plugins/model-providers/<name>/`. See [Config-Driven Routing and API Modes](../providers/config-driven-routing-and-api-modes.md).

---

### `api_mode`

`api_mode` tells Hermes which wire protocol to use when talking to a provider. Four values are supported:

| `api_mode` | Protocol | When to use |
|---|---|---|
| `chat_completions` | OpenAI-compatible REST | Default; works with most providers |
| `anthropic_messages` | Anthropic Messages API | Anthropic-specific features (prompt caching, extended thinking) |
| `codex_responses` | OpenAI Codex Responses API | OpenAI Codex / o-series models |
| `bedrock_converse` | AWS Bedrock Converse API | Models accessed through AWS Bedrock |

**In Hermes:** `api_mode` is determined by `determine_api_mode()` in `hermes_cli/model_switch.py`. See [Config-Driven Routing and API Modes](../providers/config-driven-routing-and-api-modes.md).

---

### `CredentialPool`

The `CredentialPool` class (`agent/credential_pool.py`, line 449) manages a collection of credentials for a single provider. When one credential fails or is rate-limited, the pool rotates to the next available one — automatically, without interrupting the conversation.

Four rotation **strategies** are supported:

| Strategy | Behaviour |
|---|---|
| `fill_first` | Always use the first available credential; fall over only on failure (default) |
| `round_robin` | Cycle through credentials evenly |
| `random` | Pick a credential at random |
| `least_used` | Always pick the credential with the fewest total requests |

**In Hermes:** each credential in the pool has a status. Transient failures put a credential into `exhausted` state with a cooldown. Permanent failures (e.g., a revoked OAuth token) set it to `STATUS_DEAD`, excluding it from rotation permanently. See [Credential Pool and Failover](../providers/credential-pool-and-failover.md).

---

### failover

**Failover** is the automatic switch to a backup credential or provider when the current one fails. The switch happens without user intervention and without losing the conversation.

**In Hermes:** failover is driven by `CredentialPool`. When a credential returns an error, `FailoverReason` classifies it (rate limit, auth failure, content policy, etc.) and decides whether to retry with a different credential or give up. See [Credential Pool and Failover](../providers/credential-pool-and-failover.md).

---

### cooldown

A **cooldown** is a waiting period after a failure, during which a credential is excluded from rotation. Cooldown prevents Hermes from hammering a rate-limited API key repeatedly.

**In Hermes:** cooldown duration depends on the HTTP error code:

| Error | Cooldown |
|---|---|
| 401 (auth failure) | 5 minutes |
| 429 (rate limited) | 1 hour |
| 402 (billing/quota) | 1 hour |
| Other | 1 hour |

Provider-supplied `reset_at` timestamps override these defaults when present. See [Credential Pool and Failover](../providers/credential-pool-and-failover.md).

---

### `STATUS_DEAD`

`STATUS_DEAD` (`"dead"`) is a terminal credential state in `CredentialPool`. A credential in this state will never recover on its own — for example, a revoked OAuth token. Dead credentials are excluded from rotation unconditionally and are only cleared when a fresh authentication explicitly replaces the stored token.

**In Hermes:** defined in `agent/credential_pool.py`, line 63. See [Credential Pool and Failover](../providers/credential-pool-and-failover.md).

---

### OAuth

**OAuth** (Open Authorization) is an industry-standard protocol that lets a user grant a third-party application access to their account without sharing their password. The user logs in on the provider's website and the provider gives the application a time-limited token. When the token expires, the application uses a "refresh token" to get a new one without requiring the user to log in again.

**In Hermes:** some providers (Anthropic, OpenAI Codex, Google) support OAuth-based authentication in addition to raw API keys. The `CredentialPool` handles token refresh automatically. See [OS Boundary and Isolation Postures](../security/os-boundary-and-isolation-postures.md).

---

## Gateway and Platforms

### gateway

The **gateway** is the Hermes process that connects the agent to external messaging platforms (Telegram, Discord, Slack, etc.) and handles incoming messages. It routes messages to the agent, routes the agent's responses back to the originating platform, and also runs background tasks like the kanban dispatcher and cron scheduler.

**In Hermes:** start the gateway with `hermes gateway start`. It uses platform-specific adapters to speak each platform's protocol. See [OS Boundary and Isolation Postures](../security/os-boundary-and-isolation-postures.md).

---

### platform adapter

A **platform adapter** translates between Hermes's internal message format and the protocol of one external platform. Each platform (Telegram, Discord, Slack, WhatsApp, Signal, etc.) has its own adapter in `gateway/platforms/`. Adapters handle authentication, formatting, file upload, and delivery confirmations.

**In Hermes:** the `Platform` enum in `gateway/config.py` lists all supported platforms. New platforms can be added by following `gateway/platforms/ADDING_A_PLATFORM.md`.

---

### DM pairing

**DM pairing** is how you authorize a new messaging platform account to talk to your Hermes agent. Hermes generates an 8-character one-time code that expires after 1 hour. You send that code to the bot on the new platform. If it matches, the account is paired and allowed to start conversations.

**In Hermes:** implemented in `gateway/pairing.py`. At most 3 pending codes can exist per platform at once. After 5 failed approval attempts, the platform is locked out. See [OS Boundary and Isolation Postures](../security/os-boundary-and-isolation-postures.md).

---

### async / background thread

**Async** (asynchronous) execution means work happens in the background, concurrently with other work, rather than blocking the caller until it completes. A **background thread** is a lightweight unit of execution that runs concurrently with the main thread.

**In Hermes:** several operations run in background threads so they do not slow down the conversation: `MemoryManager.sync_all()` saves memories after each turn, WAL checkpointing runs periodically, the Curator reviews skills without interrupting active conversations, and the kanban dispatcher tick runs on a timer. Up to 8 tool calls can execute concurrently on separate threads (`_MAX_TOOL_WORKERS = 8`). See [The `AIAgent` Class and Conversation Loop](../core-runtime/aiagent-and-conversation-loop.md).

---

## Skills and Learning

### skill

A **skill** is a Markdown document that teaches the agent a reusable procedure. Skills live in `~/.hermes/skills/<name>/SKILL.md`. When the agent needs to perform a complex task it has done before, it can read the relevant skill and follow it — rather than rediscovering the approach from scratch.

Skills are the "long-term procedural memory" layer of Hermes's learning loop. The agent creates skills autonomously after complex tasks and refines them over time.

**In Hermes:** the agent manages skills through three tools: `skills_list` (browse available skills), `skill_view` (read a skill's content), and `skill_manage` (create, edit, or delete a skill). See [Skill Structure and Tools](../skills/skill-structure-and-tools.md).

---

### Curator

The **Curator** is an autonomous background agent that periodically reviews the skills the main agent has created. It checks for outdated skills, consolidates overlapping skills, patches errors, pins high-quality skills, and archives stale ones. It runs on an inactivity-triggered schedule: if the main agent has been idle for a minimum period, the Curator starts a background review.

**In Hermes:** implemented in `agent/background_review.py` (spawns the review thread) and `hermes_cli/curator.py` (CLI access). The Curator is a separate concern from the cron scheduler — it is triggered by conversation inactivity, not by a clock. See [Skill Structure and Tools](../skills/skill-structure-and-tools.md).

---

### Skills Hub

The **Skills Hub** is a community registry of installable skills, hosted at agentskills.io. You install a skill from the hub with `hermes skills install <identifier>`. Installed skills are scanned by Skills Guard before installation. Trust levels govern how much trust an installed skill receives:

| Trust level | Source |
|---|---|
| `builtin` | Bundled with Hermes |
| `trusted` | Explicitly trusted by the operator |
| `community` | Installed from the hub or an external source |

**In Hermes:** install audit logs are recorded at `~/.hermes/skills/.hub/audit.log`. See [Skill Structure and Tools](../skills/skill-structure-and-tools.md).

---

## Extensions

### plugin

A **plugin** is an external Python package or directory that extends Hermes's behavior. Plugins can provide new LLM providers, new memory backends, new observability integrations, or new tools. They are discovered from standard locations and loaded by `PluginManager`.

**In Hermes:** see [Plugin System and Observer Hooks](../extensions/plugin-system-and-observer-hooks.md).

---

### observer hook

An **observer hook** is a callback that Hermes calls when internal events occur — a tool is called, a message is sent, a session ends. Your code can register a function to be called at these moments to log, record, or react to events without modifying Hermes's core. Hooks follow a schema version (`hermes.observer.v1`) defined in `hermes_cli/middleware.py`.

Observer hooks are **fail-open**: if your hook raises an exception, Hermes logs the error and continues — the hook failure does not block the agent. A `has_hook` guard means Hermes only evaluates hook payload building when at least one hook is registered, keeping the overhead close to zero when no hooks are active.

**In Hermes:** implemented in `hermes_cli/plugins.py` and `hermes_cli/middleware.py`. See [Plugin System and Observer Hooks](../extensions/plugin-system-and-observer-hooks.md).

---

### middleware

**Middleware** is code that intercepts a request or event, processes it (possibly modifying it), and passes it along. In Hermes, middleware can intercept four event kinds: incoming messages, outgoing messages, tool calls, and tool results. Unlike observer hooks (read-only), middleware can transform the data it receives. Middleware follows schema version `hermes.middleware.v1`.

**In Hermes:** defined in `hermes_cli/middleware.py`. See [Plugin System and Observer Hooks](../extensions/plugin-system-and-observer-hooks.md).

---

### MCP (Model Context Protocol)

**MCP** is an open standard protocol (by Anthropic) that defines how AI agents communicate with external tools and data sources. An MCP server exposes tools; an MCP client (like Hermes) calls those tools using a standardized JSON schema. This means you can connect Hermes to any MCP-compatible tool server without writing custom integration code.

**In Hermes:** Hermes can act as both an MCP client (connecting to external MCP servers for additional tools) and an MCP server (exposing 10 Hermes tools via `hermes mcp serve`, implemented in `mcp_serve.py`). See [Plugin System and Observer Hooks](../extensions/plugin-system-and-observer-hooks.md).

---

### ACP (Agent Communication Protocol)

**ACP** is an open standard for agent-to-agent communication — how one AI agent calls another, passes tasks, and receives results. It generalizes what tool-calling does for functions to what agents do for other agents.

**In Hermes:** ACP support is in `acp_adapter/`. The agent is registered as `hermes-agent` in `acp_registry/agent.json`. <!-- GAP: exact ACP version or spec URL — sources do not specify; source S2 mentions ACP adapter but does not include spec details -->

---

### terminal backend

A **terminal backend** is the execution environment where the agent runs shell commands. By default, commands run locally on the same machine as Hermes. Non-default backends run commands in an isolated environment:

| Backend | Where commands run |
|---|---|
| `local` | Your local machine (default) |
| `docker` | A Docker container |
| `ssh` | A remote server via SSH |
| `singularity` | A Singularity container |
| `modal` | Modal serverless cloud |
| `daytona` | Daytona cloud development environment |

Using a non-default backend is the primary way to run LLM-emitted commands in a sandboxed environment. The OS — not Hermes — is the isolation boundary. See [OS Boundary and Isolation Postures](../security/os-boundary-and-isolation-postures.md).

---

## Quick-Look Index

If you are searching for a specific term, this table lists every defined term and which section of this page covers it:

| Term | Section |
|---|---|
| ACP | Extensions |
| agent loop | The Agent and Its Loop |
| AI agent | The Agent and Its Loop |
| `api_mode` | Providers and Credentials |
| async / background thread | Gateway and Platforms |
| blackboard | Multi-Agent Coordination |
| board | Multi-Agent Coordination |
| CAS (compare-and-swap) | Multi-Agent Coordination |
| claim / claim TTL | Multi-Agent Coordination |
| context compression / LCM | Memory |
| context window | Memory |
| cooldown | Providers and Credentials |
| `CredentialPool` | Providers and Credentials |
| cron | Autonomy |
| Curator | Skills and Learning |
| DAG | Multi-Agent Coordination |
| delegation | Multi-Agent Coordination |
| dispatcher / `dispatch_once()` | Multi-Agent Coordination |
| DM pairing | Gateway and Platforms |
| failover | Providers and Credentials |
| FTS5 | Databases and Storage |
| gateway | Gateway and Platforms |
| HMAC | Autonomy |
| iteration budget | The Agent and Its Loop |
| kanban | Multi-Agent Coordination |
| LLM | The Agent and Its Loop |
| MCP | Extensions |
| memory layer | Memory |
| `MemoryManager` | Memory |
| middleware | Extensions |
| OAuth | Providers and Credentials |
| observer hook | Extensions |
| platform adapter | Gateway and Platforms |
| plugin | Extensions |
| profile | Profiles and Home Directory |
| provider | Providers and Credentials |
| `run_conversation()` | The Agent and Its Loop |
| session | Memory |
| skill | Skills and Learning |
| Skills Hub | Skills and Learning |
| SQLite | Databases and Storage |
| `STATUS_DEAD` | Providers and Credentials |
| swarm | Multi-Agent Coordination |
| task status / nine statuses | Multi-Agent Coordination |
| terminal backend | Gateway and Platforms |
| tool / tool-calling | The Agent and Its Loop |
| toolset | The Agent and Its Loop |
| trigram | Databases and Storage |
| WAL (write-ahead logging) | Databases and Storage |
| webhook | Autonomy |

---

← Previous: [What Hermes Is](../getting-started/what-is-hermes.md) · Next: [What This Guide Covers](../getting-started/scope-and-coverage.md) →
