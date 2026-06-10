---
title: "Automation and Scheduling: Cron, Heartbeat, and Dreaming"
description: "How OpenClaw schedules agent runs via cron jobs and the heartbeat mechanism, including isolated sessions, queue interaction, and the dreaming memory subagent."
category: automation
type: explanation
tags:
  - automation
  - cron
  - cron tool
  - heartbeat
  - heartbeat_respond
  - scheduled runs
  - isolated session
  - global lane
  - dreaming
  - memory-core
  - standing orders
  - webhook
  - cron vs heartbeat
  - background jobs
  - recurring tasks
  - session target
  - failure alerts
  - stall watchdog
  - HEARTBEAT.md
  - DREAMS.md
  - MEMORY.md
  - cron schedule
  - cron expression
keywords:
  - agent scheduling
  - autonomous agent
  - periodic agent turn
  - main session cron
  - isolated cron run
  - cron lane
  - queue saturation
  - cron maxConcurrentRuns
  - heartbeat every
  - heartbeat activeHours
  - lightContext
  - isolatedSession
  - dreaming frequency
  - light REM deep phases
  - dream diary
  - standing authority
  - webhook trigger
sources: [S66, S67, S48, S116, S25, S137, S138]
---

**TL;DR** — OpenClaw offers two scheduled-automation surfaces: *cron*, which fires an agent run on a calendar expression, and *heartbeat*, which nudges the agent in its main session at a regular cadence. They are separate mechanisms suited to different jobs. This chapter explains when to use each, how both interact with the run queue, uses the `memory-core` dreaming subagent as a concrete recurring example, and briefly covers webhooks and standing orders as additional automation surfaces.

# Automation and Scheduling: Cron, Heartbeat, and Dreaming

We have spent the previous chapters building up a picture of how OpenClaw handles a message that arrives from a user. Now let's flip the question: what happens when the agent needs to do work that *nobody prompted*? Scheduled summaries, memory consolidation, inbox sweeps, health checks — all of this requires the agent to start a run on its own schedule.

OpenClaw gives us two mechanisms for that, and choosing the right one matters. Let's understand what each is before we look at the details.

---

## The two scheduling primitives

Think of it this way:

- **Cron is an alarm clock.** You set it for a specific time (or a repeating pattern), and when it goes off it fires a fully-specified agent task — isolated, with its own session, its own prompt, its own delivery route. The alarm doesn't care what else the agent is doing in conversation; it fires regardless.
- **Heartbeat is a periodic "anything to do?" check-in.** Every 30 minutes (by default), the runtime taps the agent on the shoulder inside its *main* session and says: "check your heartbeat checklist; if nothing needs attention, reply `HEARTBEAT_OK`." It is lightweight, conversational, and designed to surface things that need the agent's attention without creating a separate job.

The right choice usually comes down to one question: **does the task need its own fresh context, or does it belong in the ongoing conversation?** Isolated work with deterministic output (daily summaries, memory sweeps, reports) → cron. Lightweight check-ins and nudges that should feel like part of the conversation → heartbeat.

| | Cron | Heartbeat |
|---|---|---|
| What triggers it | Calendar expression or one-shot timestamp | Fixed interval (cadence timer) |
| Session | Fresh isolated session (or configurable — see below) | Agent's main session (default) |
| Creates background task record | Yes | No |
| Defers when cron lane is busy | — | Yes (always) |
| Best for | Reports, memory sweeps, scheduled tasks | Check-ins, nudges, lightweight polls |

---

## Cron: scheduled agent runs

### How cron works

The cron service runs **inside the Gateway process** — not inside the model. It persists all job definitions, runtime state, and run history in OpenClaw's shared SQLite state database (`state/openclaw.sqlite`), so a Gateway restart does not lose your schedules. When a job fires, the Gateway enqueues an agent turn and tracks it as a background task record.

### Schedule types

You express when a job fires with one of three schedule kinds:

| Kind | CLI flag | Example |
|---|---|---|
| One-shot timestamp | `--at` | `--at "20m"` or `--at "2026-02-01T16:00:00Z"` |
| Fixed interval | `--every` | `--every "1h"` |
| Cron expression | `--cron` | `--cron "0 7 * * 1-5"` (weekdays at 7 AM) |

Timestamps without a timezone are treated as UTC. Add `--tz America/New_York` for local wall-clock scheduling. Recurring top-of-hour expressions are automatically staggered by up to 5 minutes to reduce load spikes; use `--exact` to force precise timing.

> **Cron expression gotcha.** When both the day-of-month and day-of-week fields are non-wildcard in a cron expression, OpenClaw (via the croner library) matches when *either* field matches — not both. `0 9 15 * 1` fires on every 15th of the month *and* on every Monday at 9 AM, not only on Mondays that fall on the 15th. To require both conditions, use the `+` day-of-week modifier: `0 9 15 * +1`.

### Session types for cron jobs

This is the key question the commission asks us to nail down directly: **does a cron run use the agent's normal `main` session, or does it get its own isolated session?**

The answer is: it depends on the `--session` flag, and `isolated` is the most common choice for scheduled background work. Here are all four options:

| `--session` value | Runs in | Best for |
|---|---|---|
| `isolated` | Fresh `cron:<jobId>` session per run | Reports, background chores, memory sweeps |
| `main` | Dedicated cron wake lane (not the chat lane) | Reminders, system events |
| `current` | Bound to the session active at creation time | Context-aware recurring work |
| `session:custom-id` | Persistent named session | Workflows that build on prior history |

**Isolated jobs get a truly fresh session each run.** The source is explicit about what "fresh" means here: a new transcript/session id for each run. The job does not inherit ambient conversation context from an older cron row — no channel/group routing, no send or queue policy, no elevation or origin from prior turns. This is intentional: you want the memory sweep to start clean, not colored by whatever conversation was happening last night.

**Main-session jobs** use a cron-owned run lane and optionally wake the heartbeat (via `--wake now` or `--wake next-heartbeat`). They can use the target main session's last delivery context for replies, but they do *not* append routine cron turns to the human chat lane and do not extend daily/idle reset freshness for the target session.

### Does an isolated cron run wait in the queue or get dropped?

This is a load-bearing question. The answer, verified from the source: **it waits in queue**.

Isolated cron runs use the queue's dedicated `cron-nested` execution lane internally. The `cron.maxConcurrentRuns` configuration key (default: 8) limits how many scheduled runs can execute concurrently. When a cron job fires while the lane is at capacity, the run waits for a slot to open rather than being dropped. Raising `maxConcurrentRuns` lets independent cron LLM runs progress in parallel instead of only starting their outer cron wrappers.

The global lane (which caps overall concurrency via `agents.defaults.maxConcurrent`) also applies — see the run queue chapter for a full explanation of the lane hierarchy.

A quick recap for readers arriving at this chapter first: the **global lane** is OpenClaw's cross-session concurrency pool, capped by `agents.defaults.maxConcurrent` (default 4 for main runs, 8 for subagent runs). Any run — from a user message or a cron job — must acquire a slot in this pool before the agent loop begins. For more on how lanes work, see [Run Queue and Concurrency](../agents/08-run-queue.md).

### Creating and managing cron jobs

Let's walk through a few concrete examples.

**One-shot reminder:**

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

This fires once in 20 minutes, injects a system event into the main session's cron wake lane, and one-shot jobs auto-delete after success by default.

**Recurring isolated job:**

```bash
openclaw cron create "0 7 * * *" \
  "Summarize overnight updates." \
  --name "Morning brief" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

This fires every day at 7 AM Pacific in a fresh isolated session. When the agent finishes, the final reply is delivered to a Slack channel via the `--announce` delivery mode (which posts the final text if the agent didn't send it directly).

**Inspect your jobs:**

```bash
openclaw cron list
openclaw cron get <job-id>
openclaw cron show <job-id>          # includes resolved delivery route
openclaw cron runs --id <job-id>     # run history
```

**Force a job to run now:**

```bash
openclaw cron run <jobId>                          # enqueue immediately
openclaw cron run <jobId> --wait --wait-timeout 10m  # block until done
```

### Delivery modes

After an isolated run finishes, the cron service needs to know what to do with the result:

| Mode | What happens |
|---|---|
| `announce` | Post the final reply to the configured channel/target if the agent didn't send it directly |
| `webhook` | POST the finished event payload to a URL |
| `none` | No runner fallback delivery (agent can still use the `message` tool if a chat route exists) |

### Cron failure paths

Cron has a layered failure model worth understanding.

**Failure alerts.** When a run fails, OpenClaw can notify you. Configure a global destination with `cron.failureDestination` in `openclaw.json`, or override per job with `job.delivery.failureDestination`. If neither is set and the job already delivers via `announce`, failure notifications fall back to that primary announce target.

**Stall watchdogs.** If an isolated agent-turn stalls before the runner starts or before the first model call, cron records a phase-specific timeout message such as `"setup timed out before runner start"` or `"stalled before first model call (last phase: context-engine)"`. These watchdogs cover both embedded providers and CLI-backed providers and are capped independently from long `timeoutSeconds` values, so cold-start and auth failures surface quickly.

**Timeout handling.** When an isolated agent-turn reaches `timeoutSeconds`, cron aborts the underlying agent run and gives it a short cleanup window. If the run does not drain, Gateway-owned cleanup force-clears that run's session ownership before cron records the timeout, preventing queued chat work from being left behind a stale processing session.

**Retry policy.** For transient errors (rate limit, overload, network, server error), cron retries up to 3 times with exponential backoff: 60 s, 120 s, 300 s. For recurring jobs, exponential backoff (30 s to 60 min) applies between retries, resetting after a successful run. Permanent errors disable the job immediately.

**Model preflight.** Before an isolated run starts, OpenClaw checks reachable local provider endpoints for configured Ollama, vLLM, or LM Studio servers. If the endpoint is down, the run is recorded as `skipped` with a clear provider error instead of starting a failed model call. Skipped runs do not increment execution-error backoff.

Here is a Mermaid diagram showing the execution flow for an isolated cron run:

```mermaid
sequenceDiagram
    participant Timer as Cron Timer
    participant Queue as Run Queue (cron-nested lane)
    participant Loop as Agent Loop
    participant Delivery as Delivery

    Timer->>Queue: Job fires, enqueue isolated run
    Queue-->>Timer: Queued (waits if lane saturated)
    Queue->>Loop: Slot available, start agent turn
    Loop->>Loop: Fresh isolated session (cron:<jobId>)
    Loop->>Loop: Context assembly, model call, tool execution
    alt Run succeeds
        Loop->>Delivery: Final reply ready
        Delivery->>Delivery: announce / webhook / none
    else Run stalls or times out
        Loop->>Queue: Watchdog fires, abort and cleanup
        Queue->>Queue: Record timeout, increment error counter
        Queue->>Delivery: failureDestination notification
    end
```

### Configuration reference

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 8,        // limits both scheduled dispatch and isolated agent execution
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    sessionRetention: "24h",     // prunes isolated run-session entries
    runLog: { maxBytes: "2mb", keepLines: 2000 },
  },
}
```

To disable cron entirely: `cron.enabled: false` or `OPENCLAW_SKIP_CRON=1`.

---

## Heartbeat: periodic check-ins in the main session

### How heartbeat works

Heartbeat is fundamentally different from cron. It does not create a new isolated session or a background task record. Instead, every `every` interval (default: 30 minutes), the runtime sends a prompt into the **agent's main session** and waits for a reply. The agent checks its `HEARTBEAT.md` checklist (if present), decides whether anything needs attention, and either replies `HEARTBEAT_OK` (suppressed silently) or sends an alert.

Think of it this way: cron is a separate alarm that spawns a fresh context; heartbeat is a colleague tapping you on the shoulder while you're at your desk.

The default prompt body (configurable via `agents.defaults.heartbeat.prompt`):

```
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly.
Do not infer or repeat old tasks from prior chats.
If nothing needs attention, reply HEARTBEAT_OK.
```

The `HEARTBEAT_OK` token is the agent's "all clear." OpenClaw strips it and drops the reply if the remaining content is 300 characters or fewer (`ackMaxChars` default). Only alert content (non-OK replies) is delivered externally.

### The `heartbeat_respond` tool

Tool-capable heartbeat runs can use the structured `heartbeat_respond` tool instead of returning plain text:

- `heartbeat_respond({ notify: false })` — silent acknowledgement, equivalent to `HEARTBEAT_OK`
- `heartbeat_respond({ notify: true, notificationText: "..." })` — explicit alert with structured payload

When the structured tool response is present, it takes precedence over the text fallback.

### Key configuration options

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",              // cadence; use "0m" to disable
        target: "last",            // where to deliver alerts: "last", "none", or a channel id
        lightContext: true,        // only inject HEARTBEAT.md, skip other bootstrap files
        isolatedSession: true,     // run each heartbeat in a fresh session (reduces token cost dramatically)
        skipWhenBusy: false,       // if true, defer when this agent's subagent/nested lanes are busy
        activeHours: {
          start: "08:00",
          end: "22:00",
          timezone: "America/New_York",
        },
        timeoutSeconds: 600,       // max seconds per heartbeat turn
      },
    },
  },
}
```

Key behaviors from the source:

- **`lightContext: true`** restricts bootstrap file injection to only `HEARTBEAT.md`, skipping `AGENTS.md`, `SOUL.md`, and other workspace files. This dramatically reduces per-heartbeat token cost.
- **`isolatedSession: true`** runs each heartbeat in a fresh session — the same isolation pattern as cron `sessionTarget: "isolated"`. The source notes this can reduce per-run token cost from ~100K (with full conversation history) to ~2–5K. Delivery routing still uses main session context.
- **Cron lanes always defer heartbeats**, even without `skipWhenBusy`, so local-model hosts do not run cron and heartbeat prompts simultaneously.
- **`activeHours`** restricts heartbeats to a time window. Outside the window, heartbeats are skipped until the next tick inside it. Do not set `start` and `end` to the same time — that creates a zero-width window and heartbeats are always skipped.
- Heartbeat interval default is `30m`, but is `1h` when Anthropic OAuth/token auth (including Claude CLI reuse) is the detected auth mode.

### The HEARTBEAT.md file

`HEARTBEAT.md` is an optional file in the agent workspace. If present, the default prompt instructs the agent to read it. Think of it as your heartbeat checklist: a short, stable list the agent consults every 30 minutes.

```markdown
# Heartbeat checklist

- Quick scan: anything urgent in inboxes?
- If it's daytime, do a lightweight check-in if nothing else is pending.
- If a task is blocked, write down *what is missing* and ask next time.
```

`HEARTBEAT.md` also supports a structured `tasks:` block for interval-based checks within a single heartbeat file:

```markdown
tasks:

- name: inbox-triage
  interval: 30m
  prompt: "Check for urgent unread emails and flag anything time sensitive."
- name: calendar-scan
  interval: 2h
  prompt: "Check for upcoming meetings that need prep or follow-up."
```

When `tasks:` blocks are present, OpenClaw parses them and only includes *due* tasks in the heartbeat prompt for that tick. If no tasks are due, the heartbeat is skipped entirely to avoid a wasted model call (`reason=no-tasks-due`). This lets one file hold several periodic checks without paying for all of them every tick.

> **Empty file warning.** If `HEARTBEAT.md` exists but contains only blank lines, headings, comments, or empty checklist stubs, OpenClaw skips the heartbeat run (`reason=empty-heartbeat-file`). If the file is missing entirely, the heartbeat still runs.

### Heartbeat failure paths

- If the main queue, target session lane, cron lane, or an active cron job is busy, the heartbeat is skipped and retried at the next tick. It does not wait in a queue.
- If all three visibility flags (`showOk`, `showAlerts`, `useIndicator`) are false, OpenClaw skips the heartbeat run entirely — no model call is made.
- Heartbeat-only replies do **not** keep the session alive. Idle expiry uses `lastInteractionAt` from the last real user/channel message; daily expiry uses `sessionStartedAt`.

---

## Dreaming: the concrete cron example

Now let's make cron concrete. The `memory-core` plugin's *dreaming* subagent is the best example of scheduled background automation in the entire OpenClaw codebase. It shows exactly how cron, isolated sessions, and structured background work fit together.

A quick recap for readers who haven't read the memory chapter yet: `memory-core` is OpenClaw's default memory plugin. It provides `memory_get` and `memory_search` tools for the agent, and it manages `MEMORY.md` (the agent's curated long-term note file) and daily memory notes under `memory/YYYY-MM-DD.md`. For the full memory system, see [Memory System](../memory/10-memory-system.md).

The dreaming subagent is **opt-in and disabled by default**. When enabled, `memory-core` automatically registers one cron job that runs a full memory consolidation sweep. The agent doesn't need to think about it — the schedule is managed by the plugin.

### The three-phase model

Dreaming runs three cooperative phases in order: light → REM → deep.

| Phase | Purpose | Writes to `MEMORY.md`? |
|---|---|---|
| Light | Ingest recent daily memory signals, dedupe, stage candidates | No |
| REM | Extract patterns and reflective themes from recent traces | No |
| Deep | Score and promote durable candidates using weighted ranking | Yes |

Think of it like real sleep: the light and REM phases process and sort; the deep phase makes the permanent decision about what to remember.

**Light phase** reads from short-term recall state, recent daily memory files, and redacted session transcripts when available. It writes a managed `## Light Sleep` block to `DREAMS.md` (the human-readable dream diary file) and records reinforcement signals for later deep ranking.

**REM phase** builds theme and reflection summaries from recent short-term traces. It writes a managed `## REM Sleep` block to `DREAMS.md` and records REM reinforcement signals used by deep ranking. Like light phase, it never writes to `MEMORY.md`.

**Deep phase** decides what becomes long-term memory. It ranks candidates using six weighted signals:

| Signal | Weight | Description |
|---|---|---|
| Relevance | 0.30 | Average retrieval quality for the entry |
| Frequency | 0.24 | How many short-term signals the entry accumulated |
| Query diversity | 0.15 | Distinct query/day contexts that surfaced it |
| Recency | 0.15 | Time-decayed freshness score |
| Consolidation | 0.10 | Multi-day recurrence strength |
| Conceptual richness | 0.06 | Concept-tag density from snippet/path |

Candidates must pass `minScore`, `minRecallCount`, and `minUniqueQueries` gates. Entries that pass are appended to `MEMORY.md` — the durable long-term note file the agent reads at the start of every DM session.

### Default dreaming schedule

```json5
// plugins.entries.memory-core.config.dreaming
{
  "dreaming": {
    "enabled": true,
    "frequency": "0 3 * * *"    // default: 3 AM every night
  }
}
```

By default the sweep runs at 3 AM. You can change the cadence:

```json5
{
  "dreaming": {
    "enabled": true,
    "timezone": "America/Los_Angeles",
    "frequency": "0 */6 * * *"   // every 6 hours
  }
}
```

The sweep includes the primary runtime workspace and any configured agent workspaces, deduped by path, so subagent workspace fan-out does not exclude the main agent's `DREAMS.md` and memory state.

### The Dream Diary

After each phase produces enough material, `memory-core` runs a best-effort background subagent turn and appends a short narrative entry to `DREAMS.md`. This diary is for human reading in the Dreams UI — it is not a promotion source. Only grounded memory snippets from the deep phase are eligible to promote into `MEMORY.md`.

### What dreaming writes

| Location | What it contains |
|---|---|
| `MEMORY.md` | Promoted long-term entries (deep phase only) |
| `DREAMS.md` | Human-readable diary entries (light, REM, deep summaries) |
| `memory/.dreams/` | Machine state: recall store, phase signals, ingestion checkpoints, locks |
| `memory/dreaming/<phase>/YYYY-MM-DD.md` | Optional per-phase report files |

### Blocked dreaming

If `openclaw memory status` reports `Dreaming status: blocked`, the managed cron job exists but the default agent heartbeat is not firing. The source notes: check that heartbeat is enabled for the default agent and that its target is not `none`, then run `openclaw memory status --deep` after the next heartbeat interval.

### Slash commands and CLI

```bash
/dreaming on
/dreaming off
/dreaming status
```

```bash
openclaw memory promote           # preview what would be promoted
openclaw memory promote --apply   # apply promotions
openclaw memory status --deep     # show phase-level status
```

---

## Webhooks: inbound HTTP triggers

Webhooks are the third way to trigger an agent run without a user message. Instead of a time-based schedule, an external system POSTs an HTTP request to the Gateway.

Enable webhooks in `openclaw.json`:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",             // dedicated subpath; "/" is rejected
  },
}
```

Every request must include the hook token:

```
Authorization: Bearer <token>
```

The two primary webhook endpoints are:

**`POST /hooks/wake`** — enqueue a system event for the main session:

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"New email received","mode":"now"}'
```

`mode` is `"now"` (immediate) or `"next-heartbeat"` (deferred to the next heartbeat tick).

**`POST /hooks/agent`** — run an isolated agent turn:

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.4"}'
```

Fields: `message` (required), `name`, `agentId`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `fallbacks`, `thinking`, `timeoutSeconds`.

**Security notes.** Keep hook endpoints behind loopback, a Tailscale network, or a trusted reverse proxy. Use a dedicated hook token — do not reuse gateway auth tokens. Set `hooks.allowedAgentIds` to limit which agent a hook can target. Set `hooks.allowRequestSessionKey=false` unless you require caller-selected sessions.

---

## Standing orders: authority without per-task prompting

Standing orders are a documentation convention — not a distinct technical mechanism — that sits on top of cron jobs. The idea is to define **permanent operating authority** for the agent inside your workspace files (`AGENTS.md` is the recommended location, because it is auto-injected every session).

Where cron defines *when* a task fires, a standing order defines *what the agent is authorized to do* when that task fires.

```markdown
## Program: Weekly Status Report

**Authority:** Compile data, generate report, deliver to stakeholders
**Trigger:** Every Friday at 4 PM (enforced via cron job)
**Approval gate:** None for standard reports. Flag anomalies for human review.
**Escalation:** If data source unavailable or metrics look unusual (>2σ from norm)

### Execution steps
1. Pull metrics from configured sources
2. Compare to prior week and targets
3. Generate report in Reports/weekly/YYYY-MM-DD.md
4. Deliver summary via configured channel
5. Log completion to Agent/Logs/
```

The cron job's `--message` then references the standing order rather than duplicating it:

```bash
openclaw cron add \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel imessage \
  --to "+1XXXXXXXXXX" \
  --message "Execute daily inbox triage per standing orders."
```

Standing orders work well when you combine them with an explicit execute-verify-report discipline: every task step should execute the work, verify the result, and report what was done. This pattern prevents the common failure mode of an agent acknowledging a task without completing it.

Best practices from the source:
- Start with narrow authority and expand as trust builds.
- Every program needs a "when to stop and ask" clause.
- Put standing orders in `AGENTS.md` to guarantee they are loaded every session — the workspace bootstrap automatically injects `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, and `MEMORY.md`, but not arbitrary files in subdirectories.
- Without cron enforcement, standing orders become suggestions; pair them with a scheduled job.

---

## Cron vs heartbeat: the decision guide

Let's bring this together. The two mechanisms cover different needs:

**Use cron when:**
- The task needs a clean, isolated context (no conversation history bleeding in)
- The task produces a durable artifact (a report, a memory update, a file)
- The task should run on a fixed calendar schedule regardless of conversation state
- You want full control over the model, prompt, delivery route, and timeout for that specific task
- Failure should be tracked, alerted, and retried independently

**Use heartbeat when:**
- You want the agent to check in as part of its ongoing conversation
- The check is lightweight and conversational ("is there anything I should know?")
- You want `HEARTBEAT.md` as a simple, operator-editable checklist
- The agent should surface alerts but stay silent when nothing needs attention
- You want it to defer gracefully when the agent is busy with other work

**Use both when:**
- You want heartbeat for lightweight reactive alerting *and* cron for heavier scheduled tasks (the dreaming example: heartbeat provides the general "anything to do?" cycle; dreaming cron handles the heavy nightly sweep)

---

## Troubleshooting

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

**Cron not firing:**
- Check `cron.enabled` and `OPENCLAW_SKIP_CRON` env var.
- Confirm the Gateway is running continuously.
- For `--cron` schedules, verify `--tz` against the host timezone.

**Cron fired but no delivery:**
- `delivery.mode = "none"` means no runner fallback delivery — that is expected.
- Delivery target missing or invalid means outbound was skipped.
- If the isolated run returned only `NO_REPLY`, OpenClaw suppresses all delivery paths.

**Heartbeat not running:**
- Check that `agents.defaults.heartbeat.every` is not `"0m"`.
- If `HEARTBEAT.md` is effectively empty, runs are skipped (`reason=empty-heartbeat-file`).
- If all visibility flags are false (`showOk`, `showAlerts`, `useIndicator`), the heartbeat is skipped (`reason=alerts-disabled`).
- If cron work is active, heartbeat defers until it clears.

**Dreaming blocked:**
- `openclaw memory status` shows `Dreaming status: blocked` when the managed cron job exists but heartbeat is not firing.
- Ensure heartbeat is enabled for the default agent and `target` is not `"none"`.
- Run `openclaw memory status --deep` after the next heartbeat interval.

---

← Previous: [Multi-Agent Coordination: Bindings, Specificity Rules, and Subagent Calls](../coordination/16-multi-agent.md) · Next: [Configuration System: openclaw.json, Zod Validation, and Hot Reload](../operations/18-configuration.md) →
