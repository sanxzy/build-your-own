---
title: "Storage and Persistence: SQLite, JSONL Sessions, and the Workspace"
description: "How OpenClaw organizes its data: two SQLite databases, JSONL session transcripts, workspace files, and what the backup command covers."
category: operations
type: explanation
tags:
  [
    storage,
    SQLite,
    openclaw.sqlite,
    openclaw-agent.sqlite,
    JSONL,
    session transcript,
    session index,
    sessions.json,
    workspace,
    auth_profile_store,
    auth_profile_state,
    Kysely,
    persistence,
    shared DB,
    per-agent DB,
    backup,
    state directory,
    OPENCLAW_STATE_DIR,
    named product artifact,
    credentials,
    plugin KV,
    data storage model,
  ]
keywords:
  [
    openclaw storage,
    sqlite database location,
    session jsonl file,
    openclaw backup,
    state directory layout,
    agent workspace files,
    auth profiles sqlite,
    session index schema,
    openclaw data model,
  ]
sources: [S3, S18, S19, S20, S49]
---

**TL;DR** — OpenClaw keeps all of its runtime state in exactly two SQLite databases: a shared one for gateway-wide data and a per-agent one for each agent's private state. Session conversation history lives outside SQLite as JSONL transcript files — a deliberate choice that makes them named, portable artifacts. This chapter explains what goes where, why the split exists, how sessions are stored and indexed, and what the `openclaw backup` command covers.

# Storage and Persistence: SQLite, JSONL Sessions, and the Workspace

Every time OpenClaw receives a message, runs a tool, or updates an auth credential, something needs to be written to disk. Understanding where that data lands — and why it's organized the way it is — helps you reason about backup, migration, and failure modes.

We'll build up the picture one layer at a time: first the storage policy, then the two databases, then the JSONL session layer that sits alongside them, then the workspace, and finally what the `openclaw backup` command captures.

## The storage policy: SQLite for runtime state, files only for named artifacts

Before looking at any specific file, it helps to understand the rule that governs the whole system.

OpenClaw's architecture rule is: **SQLite is the canonical store for all OpenClaw-owned runtime state.** This covers caches, queues, registries, indexes, cursors, checkpoints, and plugin scratch data. If a piece of data is app state, it belongs in SQLite.

The rule has a deliberate carve-out: **file storage is allowed only when the data is a named product artifact** — an import/export record, a user attachment, a log, a backup, or something an external tool expects to find as a file. The session transcript is the clearest example of this carve-out, and we'll come back to why it qualifies.

This policy, stated in the codebase's architecture guide, exists to prevent a class of bugs where the runtime has to consult scattered JSON/JSONL/TXT sidecar files to reconstruct its own state. It also means there's no "if SQLite fails, fall back to JSON" branching in the runtime — the only migration path for legacy file stores is through `openclaw doctor --fix`.

## Two SQLite databases: shared and per-agent

Rather than one monolithic database, OpenClaw uses two. Understanding the split makes the paths below easier to hold in your head.

Think of it like a hospital: the shared database is the hospital's central records system (scheduling, drug inventory, shared infrastructure) while the per-agent database is each patient's personal chart (private to that patient's care team).

### The shared state database

**Path:** `<stateDir>/state/openclaw.sqlite`

`<stateDir>` defaults to `~/.openclaw/`. You can override it with the `OPENCLAW_STATE_DIR` environment variable — see the [Configuration chapter](./18-configuration.md) for details on how that variable is resolved.

This database holds data that belongs to the **Gateway process as a whole**, not to any single agent. The architecture rule names it explicitly: it holds "global runtime state and plugin KV data." This covers caches, queues, registries, indexes, cursors, checkpoints, and plugin scratch data — anything the gateway owns at runtime that is not scoped to a single agent.

All access to this database goes through **Kysely** — a TypeScript query builder — rather than raw SQL strings. This is an architectural constraint: raw SQL appears only in DDL, migrations, and a narrow set of low-level bootstrapping operations.

### The per-agent database

**Path:** `<stateDir>/agents/<agentId>/agent/openclaw-agent.sqlite`

Each agent gets its own SQLite database, stored inside that agent's `agent/` directory. This database holds data that is **private to one agent's scope**: agent-local state, caches, and — most importantly for operations — the auth profile tables.

The auth profile system uses two tables in this per-agent database:

| Table | What it holds |
|---|---|
| `auth_profile_store` | Serialized auth credential payloads (API keys, OAuth tokens) |
| `auth_profile_state` | Runtime state associated with those credentials (e.g. token freshness) |

The auth profile tables are confirmed in source (`src/agents/auth-profiles/sqlite.ts`). That file opens `openclaw-agent.sqlite` from the agent's directory and queries `auth_profile_store` and `auth_profile_state` directly through Kysely. The two-table design keeps the credential payload (what you configured) separate from the runtime state (what happened when you used it), so a transient state write never accidentally overwrites a credential.

> **Why two databases rather than one?** Separating shared from per-agent state lets the Gateway manage each agent's lifecycle independently. You can delete an agent and its database without touching global state. You can also move an agent's directory to a different machine and carry its private state with it. A single shared database would make both of these operations more complex.

## The JSONL session layer: named artifacts alongside the databases

Now we reach the deliberate exception to the SQLite rule.

Session transcripts — the full conversation history including messages, tool calls, and tool results — are stored as **JSONL files** (`<sessionId>.jsonl`), not in SQLite. This is intentional and the architecture rule explains why: file storage is permitted when the data is a **named product artifact**.

A session transcript qualifies because it is something a user would reasonably want to export, inspect, or restore as a discrete document. It is not transient app state; it is a durable record of a conversation.

Think of it this way: SQLite is the filing cabinet in the back office, handling the operational data that keeps the system running. The JSONL transcript is the printed record of a conversation, filed by date — it has its own identity and could be handed to someone without needing the whole filing cabinet.

### Where session files live

All session data for a given agent lives under:

```
~/.openclaw/agents/<agentId>/sessions/
```

Within that directory, there are two kinds of objects:

**The session index** (`sessions.json`) — a JSON file that maps session keys to metadata. It is the lightweight directory of all sessions; the runtime consults it to find which transcript belongs to a conversation and to check reset timestamps. It holds three timestamps per session that serve distinct roles:

| Field | What it tracks |
|---|---|
| `sessionStartedAt` | When the current `sessionId` began; the **daily reset** uses this. |
| `lastInteractionAt` | When the last real user/channel interaction happened; the **idle reset** uses this. System events like heartbeats do not update this field, so they don't extend the idle window. |
| `updatedAt` | The most recent store-row mutation; used for listing and pruning, but **not** authoritative for any reset logic. |

**Transcript files** (`<sessionId>.jsonl`) — one JSONL file per session. These are append-only in normal operation and use a tree structure: every entry has an `id` and a `parentId`, which is how compaction checkpoints and branch navigations are represented without rewriting the whole file.

A Telegram topic session produces a slightly different filename:
```
<sessionId>-topic-<threadId>.jsonl
```

### What's inside a transcript file

The first line of every transcript is a **session header** — a JSON object with `type: "session"` that records the `id`, the working directory (`cwd`), a `timestamp`, and optionally a `parentSession` reference if this session was forked.

After the header, each subsequent line is an **entry** of one of several types:

| Entry type | What it records |
|---|---|
| `message` | User message, assistant reply, or tool result — what goes into the model's context |
| `custom_message` | Extension-injected content that does enter model context but may be hidden from UI |
| `custom` | Extension state that does **not** enter model context |
| `compaction` | A persisted compaction summary, marking how far back the conversation was summarized and what `firstKeptEntryId` was preserved |
| `branch_summary` | A summary written when navigating a tree branch |

The transcript therefore holds raw messages **and** tool calls **and** tool results. When the agent loop reaches its persistence step, it appends the full turn — the user prompt, each tool call the model made, each tool result returned, and the final assistant reply — to this file.

### Is there any data that persists outside SQLite and JSONL?

Yes, three categories:

1. **Workspace files** — the `~/.openclaw/workspace/` directory and its Markdown files (`AGENTS.md`, `SOUL.md`, `USER.md`, etc.) are ordinary files. They are not app state; they are operator-authored content that the agent reads. See the [Agents chapter](../agents/05-agents.md) for the full workspace file inventory.

2. **Credentials directory** — channel and provider credentials live in `~/.openclaw/credentials/`. These are separate from the per-agent auth profile databases.

3. **The config file** — `~/.openclaw/openclaw.json` is a plain JSON file, not a database record.

Everything else that OpenClaw owns at runtime is in one of the two SQLite databases or in JSONL transcripts.

## The workspace directory

The agent workspace — described in detail in the [Agents chapter](../agents/05-agents.md) — is a directory (`~/.openclaw/workspace` by default) that holds the agent's Markdown instruction files and memory files. A brief recap for context here:

It is intentionally **not** a database. These files are meant to be human-readable and human-editable. The architecture treats them as operator-authored content, not as runtime state, which is why they're exempt from the "SQLite for runtime state" rule.

The workspace and the state directory are separate:

```
~/.openclaw/
├── openclaw.json           ← config
├── credentials/            ← channel/provider credentials
├── state/
│   └── openclaw.sqlite     ← shared gateway state database
├── agents/
│   └── <agentId>/
│       ├── agent/
│       │   ├── openclaw-agent.sqlite   ← per-agent state + auth profiles
│       │   └── auth-profiles.json      ← legacy plaintext auth (migration target)
│       └── sessions/
│           ├── sessions.json           ← session index
│           └── <sessionId>.jsonl       ← transcript per session
└── workspace/              ← agent workspace files (AGENTS.md, SOUL.md, etc.)
```

> **Note on `auth-profiles.json`:** The secrets documentation references `auth-profiles.json` as a path that may still contain plaintext credentials in older installs. The runtime's canonical auth profile store is the `auth_profile_store`/`auth_profile_state` tables in `openclaw-agent.sqlite`. The `openclaw secrets configure --apply` workflow scrubs the legacy plaintext file once SecretRefs are in place.

## Session maintenance: keeping the sessions directory bounded

Left unchecked, the `sessions/` directory accumulates indefinitely. OpenClaw has automatic maintenance controls under `session.maintenance` in `openclaw.json`.

The defaults are:

```json5
{
  session: {
    maintenance: {
      mode: "enforce",  // "warn" to preview without mutating
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

`mode: "enforce"` applies cleanup automatically during normal Gateway writes. `mode: "warn"` reports what would be cleaned without touching any files — useful before you first apply maintenance to an existing install.

You can also run maintenance on demand:

```bash
# Preview what would be cleaned
openclaw sessions cleanup --dry-run

# Apply the configured cap immediately
openclaw sessions cleanup --enforce
```

Additional controls you can add:

| Key | What it does |
|---|---|
| `resetArchiveRetention` | How long to keep `*.reset.<timestamp>` transcript archives (default: same as `pruneAfter`; `false` disables cleanup) |
| `maxDiskBytes` | Optional total sessions-directory disk budget |
| `highWaterBytes` | Target after cleanup (default: 80% of `maxDiskBytes`) |

Maintenance respects external conversation pointers: group sessions and thread-scoped chat sessions are preserved, while synthetic entries for cron jobs, heartbeats, ACP, and subagent runs can age out.

One note about write locks: because transcript mutations serialize using a per-file write lock, a heavily loaded agent that runs slow pre-compaction housekeeping can cause lock contention. The default lock acquisition timeout is 60,000 ms. If you see "busy-session" errors on a slow machine, `session.writeLock.acquireTimeoutMs` can be raised.

## Cron session retention

Cron runs produce their own isolated sessions. These have a separate retention control:

```json5
{
  cron: {
    sessionRetention: "24h",    // default; false disables pruning
    runLog: {
      keepLines: 2000,          // SQLite run-history rows per cron job
    },
  },
}
```

The `keepLines` limit applies to rows in the shared state database's cron run-log table — another example of runtime state going to SQLite, not to a file.

## The `openclaw backup` command

Now we reach the question of backup. This is a separate matter from maintenance: maintenance bounds what stays on disk; backup is about archiving what you have.

OpenClaw's backup command is documented in `docs/cli/backup.md`, which is the source for all of the following section. The command accepts several forms:

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
```

### What gets backed up

`openclaw backup create` builds a timestamped `.tar.gz` archive. The sources it includes are:

1. **The state directory** (`~/.openclaw/` or wherever `OPENCLAW_STATE_DIR` points) — this covers `openclaw.sqlite`, all per-agent `openclaw-agent.sqlite` databases, and the `sessions/` directory with transcripts. Auth profiles under `agents/<agentId>/agent/auth-profiles.json` are also in scope.

2. **The active config file** (`~/.openclaw/openclaw.json`).

3. **The `credentials/` directory**, when it exists outside the state directory.

4. **Workspace directories** discovered from the current config (unless you pass `--no-include-workspace`).

OpenClaw avoids duplicating sources that are already nested inside the state directory. If your workspace or credentials live under `~/.openclaw/`, they are captured by the state directory entry and not added again.

### What gets intentionally skipped

During archive creation, OpenClaw skips files that have no restoration value:

- Active agent session transcripts (appended to during live runs — the archived version would be stale by the time you restore)
- Cron run logs
- Rolling log files
- Delivery queues
- Socket, PID, and temp files
- Plugin `node_modules/` trees (these are rebuildable install artifacts)

The JSON output includes a `skippedVolatileCount` field so automation can see how many files were intentionally omitted.

### Verifying and restoring

```bash
# Verify an existing archive
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz

# Create and immediately verify
openclaw backup create --verify
```

The verify step checks that the archive contains exactly one root manifest, rejects traversal-style paths inside the tarball, and confirms every manifest-declared payload is present.

After restoring an archive, reinstall plugins whose `node_modules/` were skipped:

```bash
openclaw plugins update <id>
# or
openclaw plugins install <spec> --force
```

> **Session transcripts are not backed up during live runs.** The backup command skips active session transcripts because they are being written to during an active Gateway session. To capture a complete transcript archive, either stop the Gateway before running `openclaw backup create`, or accept that the most recent turns of any active session may not appear in the backup.

### Workspace git backup (recommended alongside `openclaw backup`)

The `openclaw backup create` command captures the workspace, but the recommended ongoing approach for workspace files is a **private git repository**. This is documented in the agent-workspace source:

```bash
# If the workspace isn't already a git repo
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

Then add a private remote (GitHub, GitLab, or similar) and push. This gives you incremental history for memory files and workspace instructions — something a tarball backup does not provide.

Keep secrets out of the workspace repo. The `agents/<agentId>/agent/` directory (which holds `auth-profiles.json` and `openclaw-agent.sqlite`) should never be committed.

### Storage locations at a glance

| What | Default path | DB or file? |
|---|---|---|
| Shared gateway DB | `~/.openclaw/state/openclaw.sqlite` | SQLite (Kysely) |
| Per-agent private DB | `~/.openclaw/agents/<id>/agent/openclaw-agent.sqlite` | SQLite (Kysely) |
| Session index | `~/.openclaw/agents/<id>/sessions/sessions.json` | JSON file |
| Session transcripts | `~/.openclaw/agents/<id>/sessions/<sessionId>.jsonl` | JSONL file (named artifact) |
| Workspace files | `~/.openclaw/workspace/` | Plain files |
| Config | `~/.openclaw/openclaw.json` | JSON file |
| Credentials | `~/.openclaw/credentials/` | Files |

## Summary

OpenClaw's storage model rests on one principle: runtime state belongs in SQLite, and files are only for named artifacts or operator-authored content. That gives you two databases — the shared gateway database and the per-agent private database — alongside JSONL transcripts that earn their file status by being importable, exportable conversation records. The workspace directory is deliberately outside both because it holds human-readable instructions, not machine state.

The `openclaw backup` command captures all of this (minus volatile live files and rebuildable `node_modules/`) into a verifiable tarball. For ongoing workspace history, a private git repository complements the tarball approach.

---

← Previous: [Configuration System: openclaw.json, Zod Validation, and Hot Reload](./18-configuration.md) · Next: [Security and Governance: Pairing, Auth Modes, Sandbox, and Network Policy](./20-security.md) →
