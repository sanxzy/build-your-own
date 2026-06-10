# OpenClaw Glossary

> This is the top-level mirror of the canonical glossary chapter, [27 · Glossary](docs/reference/27-glossary.md). It is kept here so the library exposes a `GLOSSARY.md` at its root. For the chapter in reading order, see the [reading map](index.md).

**TL;DR** — This page collects every OpenClaw-specific term you will encounter across the library and defines each one in plain language. If you hit an unfamiliar word in any chapter, look it up here and follow the pointer to the chapter that covers it in depth.


Every chapter in this library introduces terms as they first appear. This glossary is a fast lookup companion: definitions are concise and practical, and each entry links to the chapter where the concept is explored in full context.

Entries are alphabetized. Where a term has a precise source definition, that definition is the basis here; inferred meanings are noted. Related terms cross-reference one another.

---

## A

**Active memory**
A plugin (`@openclaw/active-memory`) that runs a blocking memory subagent *before* eligible replies, querying the memory store and injecting relevant recall into the current turn's context; it operates in addition to the exclusive memory plugin slot, not inside it.
→ [Memory System](docs/memory/10-memory-system.md)

**Agent**
A fully scoped AI runtime with its own workspace directory, bootstrap files, per-agent auth profiles, session store, and agent loop; the Gateway may host one or many agents simultaneously, each identified by an `agentId`.
→ [Agents](docs/agents/05-agents.md)

**Agent loop**
The full sequence of work OpenClaw performs for a single inbound prompt: intake → context assembly → model inference → tool execution → streaming replies → persistence; it is the authoritative path that turns a message into actions and a final reply.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

**Agent workspace**
The on-disk directory (`agents.defaults.workspace`, default `~/.openclaw/workspace`) that serves as the agent's working directory for tools and context; it contains the bootstrap files, skill folder, daily memory notes, and session transcripts.
→ [Agents](docs/agents/05-agents.md)

**`agent.wait`**
A Gateway RPC that blocks until a named run (identified by `runId`) reaches a terminal state and returns `{ status, startedAt, endedAt, error? }`; the default wait timeout is 30 seconds, independent of the agent runtime timeout.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

**`agentId`**
The string identifier for one agent scope; in single-agent mode it defaults to `main`; in multi-agent mode each entry in `agents.list` carries a distinct id.
→ [Multi-Agent Coordination](docs/coordination/16-multi-agent.md)

**Auth profile**
A named credential set — type `api_key`, `token`, or `oauth` — stored per-agent at `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` and used for model or service authentication; `api_key` and `token` profiles are portable across agents, while `oauth` profiles are not (single-use refresh risk).
→ [Configuration System](docs/operations/18-configuration.md)

---

## B

**Binding**
A routing rule that maps a `(channel, accountId, peer)` tuple to a specific `agentId`; the most specific matching binding wins, following the order: exact peer → parentPeer → guildId+roles → guildId → teamId → accountId → channel-level → default agent.
→ [Multi-Agent Coordination](docs/coordination/16-multi-agent.md)

**Bootstrap file**
A workspace file (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`) that is injected into the system prompt's Project Context on the *first turn* of a new session; blank files are skipped and large files are truncated with a marker.
→ [Agents](docs/agents/05-agents.md) · [System Prompt and Context](docs/agents/09-system-prompt.md)

**`buildAgentSystemPrompt`**
The internal function that assembles the OpenClaw system prompt from explicit inputs (tools, skills, bootstrap files, sandbox state, provider contributions); it is a pure renderer and does not read global config directly.
→ [System Prompt and Context](docs/agents/09-system-prompt.md)

---

## C

**Channel**
A registered message surface — such as Telegram, Discord, WhatsApp, Signal, or iMessage — implemented as a plugin that maps inbound messages to sessions and delivers outbound replies; channels are distinct from tools in that they are the *entry points* for user messages, not callable functions the model invokes.
→ [Channels](docs/channels/04-channels.md)

**ClawHub**
The public plugin and skill discovery registry at `clawhub.ai`; operator-facing install commands use `clawhub:<pkg>` style slugs; publishers host and maintain plugins in their own repositories.
→ [Plugins, Skills, and Tools](docs/extending/11-plugins-skills-tools.md)

**Compaction**
The process of summarizing older conversation messages when the context window approaches its limit; tool call/result pairs are kept together, and a memory-flush turn runs before compaction so important facts are written to disk before the summary replaces the raw history.
→ [System Prompt and Context](docs/agents/09-system-prompt.md)

**`connect` frame**
The mandatory first WebSocket frame a client must send after the Gateway issues a `connect.challenge` event; it carries the client identity, auth credentials, role, and a signed nonce; any non-JSON or non-`connect` first frame results in a hard close.
→ [Gateway](docs/gateway/03-gateway.md)

**Cron job**
A scheduled agent run configured via the `cron` built-in tool; cron runs receive a fresh isolated session per run and enter the global lane; the scheduler starts a per-run timeout timer and aborts the underlying run at the configured deadline.
→ [Automation and Scheduling](docs/automation/17-automation.md)

---

## D

**Device token**
A per-device credential issued by the Gateway after a successful `connect` handshake with pairing approval; the token is returned in `hello-ok.auth.deviceToken` and should be persisted by the client for subsequent reconnects.
→ [Gateway](docs/gateway/03-gateway.md)

**DM pairing**
The default security model (`dmPolicy="pairing"`) for personal channels; unknown senders receive a short pairing code and the agent does not process their message until the operator approves with `openclaw pairing approve <channel> <code>`.
→ [Security and Governance](docs/operations/20-security.md)

**`dmScope`**
A session-routing configuration option that controls how direct-message conversations are mapped to sessions; the four values are `main` (all DMs share one session, the default), `per-peer` (isolate by sender), `per-channel-peer` (isolate by channel + sender, recommended for multi-user setups), and `per-account-channel-peer` (isolate by account + channel + sender).
→ [Sessions](docs/agents/07-sessions.md)

**Dreaming subagent**
A cron-scheduled background process in `memory-core` that consolidates short-term memory signals into long-term memory (`MEMORY.md`) through scored light/REM/deep phases; it is disabled by default and, when enabled, auto-manages one recurring cron job.
→ [Memory System](docs/memory/10-memory-system.md) · [Automation and Scheduling](docs/automation/17-automation.md)

---

## E

**Effective tool policy**
The resolved set of tools the model is permitted to see for a given run; it is computed before the model call by combining the active auth profile, `tools.allow`/`tools.deny` config, provider restrictions, sandbox state, channel permissions, and plugin availability; tools excluded by policy are never sent to the model.
→ [Tool System](docs/extending/12-tool-system.md)

---

## G

**Gateway**
The single long-running process that is the control plane for all agents and channels; it exposes a multiplexed WebSocket and HTTP API on port 18789 (default) and manages provider connections, session routing, tool dispatch, and plugin lifecycle.
→ [Gateway](docs/gateway/03-gateway.md) · [High-Level Architecture](docs/getting-started/02-architecture.md)

**Global lane**
The cross-session concurrency pool that caps total active runs across all sessions; the default `main` lane allows up to `agents.defaults.maxConcurrent` concurrent runs (default 4 for inbound runs, 8 for subagent runs); cron and subagent runs use separate lanes with their own caps.
→ [Run Queue and Concurrency](docs/agents/08-run-queue.md)

---

## H

**Harness**
The runtime implementation that executes an agent's turns; the default embedded harness (`openclaw`) is built into the Gateway and handles model discovery, tool wiring, prompt assembly, session management, and channel delivery as one integrated surface; CLI backend harnesses (such as `claude-cli`) delegate execution to an external process.
→ [Agents](docs/agents/05-agents.md)

**Heartbeat**
A configured interval that fires the `heartbeat_respond` built-in tool, giving the agent an opportunity to act (check tasks, write memory notes, send a report) without an inbound user message; heartbeat runs enter the global lane and wait in queue if the lane is saturated.
→ [Automation and Scheduling](docs/automation/17-automation.md)

**`hello-ok`**
The Gateway's response payload to a successful `connect` handshake; it contains the negotiated protocol version, server metadata, feature discovery lists (`features.methods`, `features.events`), connection policy limits (`maxPayload`, `maxBufferedBytes`, `tickIntervalMs`), and the negotiated auth role and scopes.
→ [Gateway](docs/gateway/03-gateway.md)

**Hook**
A lifecycle event callback registered by a plugin via `api.on(hookName, handler, opts?)` to intercept the agent loop, tool calls, message flow, session lifecycle, or Gateway startup; multiple hooks on the same event run in descending numeric `priority` order.
→ [Agent Loop Hooks](docs/extending/14-hooks.md)

---

## J

**JSONL session transcript**
The per-session conversation record stored at `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`; each line is a JSON record representing one message or tool call/result; session metadata is tracked in a parallel index file (`sessions.json`).
→ [Sessions](docs/agents/07-sessions.md) · [Storage and Persistence](docs/operations/19-storage.md)

---

## M

**Memory plugin slot**
The exclusive plugin position for the active memory implementation; only one memory plugin can occupy this slot at a time (`plugins.slots.memory`); the two confirmed slot plugin ids are `memory-core` (default, bundled) and `memory-lancedb` (LanceDB vector DB); QMD and Honcho are memory *backends* (alternative storage engines for recall), not slot-selectable plugin ids — they integrate as backend options within the memory system rather than as named slot plugins.
→ [Memory System](docs/memory/10-memory-system.md)

**`MEMORY.md`**
The curated long-term memory file in the agent workspace; it holds durable facts, preferences, and decisions and is loaded at the start of every DM session as part of bootstrap; daily notes and detailed observations belong in `memory/YYYY-MM-DD.md` files instead.
→ [Memory System](docs/memory/10-memory-system.md)

**Model ref**
A `provider/model` string (for example `anthropic/claude-opus-4-6`) that identifies a specific AI model via a named provider; if the provider segment is omitted, OpenClaw tries an alias, then a unique configured-provider match, then the configured default provider.
→ [AI Model Integration](docs/models/15-model-integration.md)

---

## N

**Node**
A companion device — macOS, iOS, Android, or a headless process — that connects to the Gateway WebSocket with `role: "node"`, declares capability claims (`caps`, `commands`, `permissions`), and can receive commands like `canvas.navigate`, `camera.snap`, or `screen.record`; pairing approval is required for new device IDs.
→ [Gateway](docs/gateway/03-gateway.md)

**Node pairing**
The device-identity handshake a node completes before the Gateway issues a device token; the node includes a stable `device.id` derived from a keypair fingerprint and signs the server-provided challenge nonce; new device IDs require explicit operator approval.
→ [Gateway](docs/gateway/03-gateway.md)

**`NO_REPLY`**
A reserved output token; when the agent's final reply consists solely of `NO_REPLY` (or `no_reply`), OpenClaw filters it from all outgoing payloads and no visible reply is delivered to the channel.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

---

## O

**`openclaw doctor`**
The canonical health-check and configuration migration command; it surfaces risky or misconfigured settings, detects stale or invalid config keys, and with `--fix` rewrites the config to the current canonical format; core-owned migrations run in core doctor code; plugin-owned migrations run in each plugin's doctor contract.
→ [Configuration System](docs/operations/18-configuration.md) · [Best Practices](docs/reference/26-best-practices.md)

---

## P

**Plugin**
A packaged runtime capability that can register tools, providers, channels, hooks, skills, and services; plugins are declared via an `openclaw.plugin.json` manifest and run in-process with the Gateway; two styles exist: *code plugins* (runtime hooks/providers/channels/tools) and *bundle-style plugins* (skills, MCP servers, config).
→ [Plugins, Skills, and Tools](docs/extending/11-plugins-skills-tools.md)

**Port 18789**
The default TCP port the Gateway binds to; it is shared by WebSocket connections (the wire protocol), HTTP endpoints (Control UI, OpenAI-compatible API), and plugin-registered HTTP routes; the bind host defaults to `127.0.0.1` (loopback only).
→ [Gateway](docs/gateway/03-gateway.md)

---

## Q

**Queue mode**
The policy that governs what happens to a new prompt that arrives while a session already has an active run; the four modes are `steer` (inject into the active run after current tool calls finish — the default), `followup` (wait for the current run to end), `collect` (coalesce queued messages into a single followup turn), and `interrupt` (abort the active run).
→ [Run Queue and Concurrency](docs/agents/08-run-queue.md)

---

## R

**Run**
A single execution of the agent loop initiated by one inbound prompt (or a cron/heartbeat trigger); each run is assigned a `runId` returned immediately in `{ runId, acceptedAt }` when the `agent` RPC is called; runs are serialized per session lane and capped globally.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

**`runId`**
The unique identifier for one agent loop execution; returned immediately in the `agent` RPC response (`{ runId, acceptedAt }`) so callers can track streaming events and use `agent.wait` without blocking on the full run.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

---

## S

**Sandbox**
An execution environment for non-`main` sessions; when `agents.defaults.sandbox.mode: "non-main"` is set, tool calls in non-main sessions run inside an isolated container (Docker default backend); the default sandbox tool policy allows bash/read/write/edit/sessions operations and denies browser/canvas/cron/gateway tools.
→ [Security and Governance](docs/operations/20-security.md)

**Session**
A JSONL conversation record scoped to a channel/peer/account combination and identified by a session key; sessions track `sessionStartedAt`, `lastInteractionAt`, and `updatedAt` in the index file; DM sessions default to a shared `main` key unless `dmScope` is configured otherwise.
→ [Sessions](docs/agents/07-sessions.md)

**Session key**
The string that uniquely identifies one session scope, used as the lane key for serialization and as the filename base for the JSONL transcript; for DM sessions in single-agent mode it takes the form `agent:main:<mainKey>`.
→ [Sessions](docs/agents/07-sessions.md) · [Run Queue and Concurrency](docs/agents/08-run-queue.md)

**Session lane**
The per-session serialization queue that ensures only one agent run is active for a given session key at a time; a second run for the same session waits in the lane until the first completes (behavior depends on the queue mode).
→ [Run Queue and Concurrency](docs/agents/08-run-queue.md)

**Skill**
A `SKILL.md` instruction pack — YAML frontmatter plus a Markdown body — that is injected into the system prompt as compact XML to shape how the agent behaves; skills do not add new callable functions (that is what tools do), and they follow a six-level loading precedence from workspace to bundled.
→ [Skills](docs/extending/13-skills.md)

**`SKILL.md`**
The file format for a skill; at minimum it requires `name` and `description` frontmatter fields; optional fields include `user-invocable`, `disable-model-invocation`, `command-dispatch`, and `metadata.openclaw` gating (requiring binaries, env vars, or config keys).
→ [Skills](docs/extending/13-skills.md)

**Soul**
The `SOUL.md` workspace file that defines the agent's persona, tone, and behavioral boundaries; it is injected into the system prompt as a bootstrap file on the first turn of each session.
→ [Agents](docs/agents/05-agents.md) · [System Prompt and Context](docs/agents/09-system-prompt.md)

**SQLite (shared and per-agent)**
The canonical stores for OpenClaw-owned runtime state; the shared database at `<stateDir>/state/openclaw.sqlite` holds plugin KV, task registry, delivery queue, model-catalog cache, and installed-plugin index; the per-agent database at `<stateDir>/agents/<agentId>/agent/openclaw-agent.sqlite` holds agent-private state and auth profile tables.
→ [Storage and Persistence](docs/operations/19-storage.md)

**Subagent**
A subordinate agent run spawned by a parent run via `sessions_spawn`; the `subagents` tool lists and inspects spawned sub-agent runs owned by the requester session — it is a status-listing tool, not a spawn tool; subagent runs enter the global lane as a subagent run (capped separately from main runs), execute their own agent loop, and announce their result back to the requester session.
→ [Multi-Agent Coordination](docs/coordination/16-multi-agent.md)

**System prompt**
The OpenClaw-assembled context block sent to the model at the start of each turn; it contains fixed sections (Tooling, Execution Bias, Safety, Skills, Workspace, Bootstrap files, Sandbox, Runtime, Reasoning, etc.) assembled by `buildAgentSystemPrompt`; the model never sees a raw provider default prompt.
→ [System Prompt and Context](docs/agents/09-system-prompt.md)

---

## T

**`ThinkingLevel`**
A configuration value that controls how much extended reasoning the model performs before responding; the eight levels are `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, and `max`; `adaptive` is the default for Anthropic Claude 4.6 models (provider-managed adaptive thinking); the `/think <level>` chat command changes the level for the current session.
→ [AI Model Integration](docs/models/15-model-integration.md)

**Tool**
A typed callable function the model can invoke during a run; each tool has a name, description, and parameter schema; the model sees only the tools included in the effective tool policy for that run.
→ [Tool System](docs/extending/12-tool-system.md)

**Tool policy**
See *Effective tool policy*.

---

## W

**Wire protocol**
The JSON-over-WebSocket message format used between the Gateway and all clients on port 18789; three frame types exist: `req` (`{type:"req", id, method, params}`), `res` (`{type:"res", id, ok, payload|error}`), and `event` (`{type:"event", event, payload, seq?, stateVersion?}`); the first frame on any new connection must be `connect`.
→ [Gateway](docs/gateway/03-gateway.md)

**Write lock (session)**
A file-based process-aware lock acquired before any transcript write, compaction, or truncation; writers wait up to `session.writeLock.acquireTimeoutMs` (default 60 000 ms) before reporting the session as busy; the lock is non-reentrant by default.
→ [The Agent Loop](docs/agents/06-agent-loop.md)

---

---

Return to the [reading map](index.md) · canonical chapter: [27 · Glossary](docs/reference/27-glossary.md)
