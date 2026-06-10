---
title: "Observability — Log Files, Observer Events, and Bundled Consumers"
description: "How Hermes surfaces what the agent did: four rotating log files, per-session tagging, the hermes logs CLI, and the observer hook system for metrics and tracing."
category: observability
type: explanation
tags:
  - observability
  - rotating log files
  - agent.log
  - errors.log
  - gateway.log
  - gui.log
  - RotatingFileHandler
  - RedactingFormatter
  - session_id tagging
  - hermes logs
  - observer hook system
  - Langfuse plugin
  - NeMo Relay plugin
  - no Prometheus
  - no Grafana
  - no distributed tracing
  - skills audit log
  - setup_logging
  - log level
  - log rotation
  - secret redaction
  - COMPONENT_PREFIXES
  - tail_log
  - follow mode
  - component filter
  - since filter
  - telemetry schema version
  - correlation IDs
  - hook callbacks
  - fail-open hooks
  - has_hook gating
  - ATOF
  - ATIF
  - NeMo Relay
  - Langfuse traces
keywords:
  - hermes observability
  - agent log files
  - session correlation
  - log filtering CLI
  - plugin telemetry
  - register_hook
  - pre_api_request
  - post_tool_call
  - on_session_start
  - HERMES_LANGFUSE_PUBLIC_KEY
  - HERMES_NEMO_RELAY_ATOF_ENABLED
sources: [S70, S60, S71]
---

**TL;DR** — Hermes does not ship a built-in Prometheus endpoint or a distributed tracing stack. What it does provide is a set of four rotating log files under `~/.hermes/logs/`, a per-session tagging mechanism that lets you trace one conversation across interleaved log lines, and the `hermes logs` CLI for browsing those files. If you need metrics, spans, or external trace export, you add that through the observer hook system — a plugin extension point that already has two bundled consumers: Langfuse and NeMo Relay. By the end of this chapter you will know what each log file captures, how to filter by session, and how to wire Langfuse or NeMo Relay for deeper telemetry.

# Observability — Log Files, Observer Events, and Bundled Consumers

## What the problem is, and what Hermes actually gives you

When you operate an agent, two questions come up quickly: *what did it do?* and *why did that call fail?* You need a record of activity, and you need a way to zoom in on one session without reading megabytes of interleaved output.

Hermes answers these with structured log files — not metrics dashboards. There is **no built-in Prometheus metrics endpoint**, no built-in Grafana integration, and no distributed tracing stack wired in by default. What you get out of the box is:

- **Four rotating log files**, each covering a different slice of the system.
- **Per-thread `[session_id]` tags** on every log line, so you can grep or filter for one session.
- **The `hermes logs` CLI**, which combines tailing, following, level filtering, session filtering, and component filtering in one command.
- **The observer hook system** as the extension point for anything beyond file logs: metrics, traces, span export, trajectory format — all of it comes from observer plugins.

Two observer plugins ship bundled: **Langfuse** (turn and call tracing) and **NeMo Relay** (ATOF/ATIF trajectory export). Neither is active by default; both are opt-in.

Let's walk through each layer in turn, starting with the logs themselves.

---

## The four rotating log files

### Why four files, not one?

A single log file is easy to produce but awkward to use. Errors get buried in INFO-level activity. Gateway connection events clutter agent-loop lines. During a debugging session you usually care about one slice at a time. Hermes separates concerns into four files, each under `~/.hermes/logs/`, created when `setup_logging()` runs at CLI or gateway startup.

A **rotating log file** is one that automatically rolls over to a new file once it reaches a size limit, keeping a small number of compressed backups. This prevents logs from filling your disk indefinitely. Hermes uses Python's `RotatingFileHandler` — a standard library class that handles the rollover — with a custom `RedactingFormatter` layered on top to scrub secrets from every line before it is written.

> **What `RedactingFormatter` does.** Before writing any log record, it runs the formatted text through a regex-based secret scanner that recognises known API-key prefixes (like `sk-`, `sk-ant-`, `AKIA…`, `ghp_`, `hf_`, Stripe keys, and many others). Short tokens under 18 characters are fully replaced with `[REDACTED]`; longer tokens keep the first 6 and last 4 characters for debuggability — enough to identify a key without leaking it. The redactor is also applied to query-string parameters and JSON body keys named `password`, `api_key`, `client_secret`, `token`, and similar. This runs at log-write time, so secrets never reach disk regardless of which handler receives the record.

Here is the complete table of the four files:

| File | Level threshold | Max size | Backups kept | What lands here |
|---|---|---|---|---|
| `agent.log` | INFO and above | 5 MB (default) | 3 | Everything — the catch-all main activity log: agent loop, tool calls, session events, memory, cron |
| `errors.log` | WARNING and above | 2 MB | 2 | Errors and warnings only; the first place to look after a failure |
| `gateway.log` | INFO and above | 5 MB | 3 | Gateway-only records: platform adapters, session management, slash commands, delivery routing; created only when `mode="gateway"` |
| `gui.log` | INFO and above | 10 MB | 5 | Dashboard and TUI-gateway records: `web_server`, `pty_bridge`, `tui_gateway.*`, `uvicorn`; created only when `mode="gui"` |

`agent.log` is always created. `gateway.log` and `gui.log` are created conditionally: the gateway process creates `gateway.log` when it starts, and the dashboard/TUI backend creates `gui.log`. If you are running Hermes purely as a CLI agent with no gateway, you will have `agent.log` and `errors.log` only.

**Component routing** — `gateway.log` does not duplicate everything; it only receives records whose Python logger name starts with `gateway` or `hermes_plugins`. The `gui.log` receives records from `hermes_cli.web_server`, `hermes_cli.pty_bridge`, `tui_gateway.*`, and `uvicorn.*`. `agent.log` receives everything, making it the complete record.

### Log line format

Every line written by `RedactingFormatter` follows this template (from `_LOG_FORMAT` in `hermes_logging.py`):

```
%(asctime)s %(levelname)s%(session_tag)s %(name)s: %(message)s
```

A real line looks like:

```text
2026-06-10 14:23:05,412 INFO [abc123] tools.terminal_tool: running command: ls -la
```

The `[abc123]` fragment is the `session_tag` — we will look at how it gets injected next.

---

## Per-thread `[session_id]` tagging

### The problem this solves

Hermes can be handling multiple concurrent sessions — for example, a gateway serving two Telegram conversations at once. Both sessions write to the same `agent.log`. If you grep for a bug, you see interleaved lines from both sessions and have no way to separate them without an identifier on each line.

### How session tagging works

At the top of every `run_conversation()` call, Hermes calls `set_session_context(session_id)`. This stores the session ID in **thread-local storage** — a Python mechanism that gives each thread its own private copy of a variable. When the conversation ends (or is interrupted), `clear_session_context()` removes it.

Rather than attaching a filter to each handler, Hermes replaces the global Python **log record factory** (the function that creates every `LogRecord` object before a handler ever sees it). The custom factory reads the thread-local `session_id` and injects it as a `session_tag` field on the record. Because this runs for every record in the process, it works with all handlers — including third-party ones — and the `%(session_tag)s` token in the format string is always available.

```python
# Simplified view of the session record factory (from hermes_logging.py)
def _session_record_factory(*args, **kwargs):
    record = current_factory(*args, **kwargs)
    sid = getattr(_session_context, "session_id", None)
    record.session_tag = f" [{sid}]" if sid else ""
    return record
```

The result: every line emitted on a session's thread carries `[<session_id>]` in the formatted output, and lines emitted outside a session carry nothing. You can now filter `agent.log` to one session using any text tool.

---

## The `hermes logs` CLI

### The problem with raw `tail`

You can always open `~/.hermes/logs/agent.log` with `tail` or `grep`, but that quickly becomes unwieldy. You want to show only WARNING-level lines, or only lines from one session, or only gateway-related lines — possibly all at once and following in real time. The `hermes logs` command provides that as first-class flags.

### Basic usage

```bash
# Show the last 50 lines of agent.log (the default)
hermes logs

# Follow agent.log in real time — Ctrl+C to stop
hermes logs -f

# Show the last 50 lines of errors.log
hermes logs errors

# Show the last 100 lines of gateway.log
hermes logs gateway -n 100

# Follow gui.log
hermes logs gui -f

# List all available log files with sizes and ages
hermes logs list
```

The first positional argument chooses which log to read: `agent` (default), `errors`, `gateway`, or `gui`. All other arguments are filters.

### Flags reference

| Flag | Short | Purpose | Example |
|---|---|---|---|
| `--follow` | `-f` | Keep reading new lines as they arrive (Ctrl+C to stop) | `hermes logs -f` |
| `--lines N` | `-n N` | Number of recent lines to show (default: 50) | `hermes logs -n 200` |
| `--level LEVEL` | — | Minimum log level: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` | `hermes logs --level WARNING` |
| `--session ID` | — | Filter to lines containing this session ID as a substring | `hermes logs --session abc123` |
| `--since TIME` | — | Show lines from the last N seconds/minutes/hours/days (e.g. `30m`, `1h`, `2d`) | `hermes logs --since 1h` |
| `--component NAME` | — | Filter to one component bucket: `gateway`, `agent`, `tools`, `cli`, `cron`, `gui` | `hermes logs --component tools` |

Flags compose freely:

```bash
# Follow agent.log, show only WARNING+, limit to one session
hermes logs --follow --level WARNING --session abc123

# Show the last hour of gateway activity, then keep following
hermes logs gateway --since 1h -f

# Find all tool errors in the last two days
hermes logs --since 2d --level ERROR --component tools
```

### Worked example: tracing a single session

Suppose a session with ID `s-7f2a` produced an unexpected result. Here is how we follow it from start to finish:

```bash
# Step 1 — glance at recent errors to spot the failure
hermes logs errors --since 2h

# Step 2 — zoom in on that session across the full activity log
hermes logs --session s-7f2a

# Step 3 — follow live if the session is still active
hermes logs -f --session s-7f2a
```

The `--session` flag does a substring match, so `--session s-7f2a` will match any line containing `[s-7f2a]`. If you only know a partial session ID (perhaps from a user report), a prefix is enough.

---

## The observer hook system — the primary extension point for metrics and tracing

Now we have structured logs. But suppose you want to count how many LLM calls a session makes, measure per-tool latencies, or export structured span data to an external tracing system. Log files are not the right shape for that — they are text, not structured events.

This is where the **observer hook system** comes in. It is the only supported observability extension point beyond file logs.

### What observer hooks are (brief recap)

Observer hooks are an extension mechanism that lets plugins register callbacks on named lifecycle events — `pre_api_request`, `post_tool_call`, `on_session_start`, and so on. The agent emits these events as it runs; registered callbacks receive a payload of structured fields. For a full treatment of the plugin and hook registration model, see the canonical page on [plugin system and observer hooks](../extensions/plugin-system-and-observer-hooks.md). Here we focus on how observer hooks feed the observability use case.

The key properties for telemetry:

- **Read-only by default.** Observer hook callbacks receive event data but do not normally change agent behavior. The agent keeps running regardless of what the callback does.
- **Fail-open.** If a callback raises an exception, Hermes logs a warning and continues. A broken observability plugin does not stop the agent.
- **`has_hook` gating.** Building the sanitized request/response payloads is not free. Hermes only constructs those payloads when at least one plugin has registered the relevant hook — so the uninstrumented path stays cheap.
- **Stable correlation IDs.** Every payload includes `session_id`, `turn_id`, `api_request_id`, and `tool_call_id` so that a plugin can join events from multiple hooks into coherent spans without relying on callback order.
- **Schema version.** Every payload carries `telemetry_schema_version = "hermes.observer.v1"` so consumers can handle version transitions.

### The six observer event families

Observer hooks are grouped into six families, each covering a different lifecycle boundary:

| Family | Hooks | What they cover |
|---|---|---|
| Session lifecycle | `on_session_start`, `on_session_end`, `on_session_finalize`, `on_session_reset` | Conversation boundaries and session identity transitions |
| Turn-scoped LLM | `pre_llm_call`, `post_llm_call` | One user turn from first LLM call through final assistant response |
| Request-scoped API | `pre_api_request`, `post_api_request`, `api_request_error` | Individual provider attempts inside the agent loop |
| Tool lifecycle | `pre_tool_call`, `post_tool_call`, `transform_tool_result` | Individual tool dispatches, including blocked and cancelled outcomes |
| Approval lifecycle | `pre_approval_request`, `post_approval_response` | Dangerous-command approval prompts and user choices |
| Subagent lifecycle | `subagent_start`, `subagent_stop` | Delegated child agents and their parent linkage |

A few hooks have optional behavior-affecting return values (notably `pre_tool_call` can block a tool by returning `{"action": "block", "message": "..."}`), but for observability purposes you ignore the return values and treat every hook as a notification.

### What you do NOT get out of the box

To be explicit: if you start Hermes with no plugins enabled, you have four log files and nothing else. There is:

- No Prometheus `/metrics` endpoint.
- No Grafana dashboard wired in.
- No OpenTelemetry collector integration active.
- No span or trace export running.

All of that becomes available by enabling an observer plugin. The two bundled options are described next.

---

## Bundled observer plugin: Langfuse

Langfuse is an open-source LLM observability platform. The Hermes Langfuse plugin registers hooks on `pre_api_request`, `post_api_request`, `pre_llm_call`, `post_llm_call`, `pre_tool_call`, and `post_tool_call`, and maps them to Langfuse traces, generations, and spans.

The plugin **ships bundled but is opt-in** — it does not load until you enable it. If the Langfuse SDK is not installed or credentials are missing, the plugin fails open silently: hooks are registered but all callbacks are no-ops.

### Enabling Langfuse

```bash
# Option A — interactive wizard (walks through credential entry + SDK install)
hermes tools   # then navigate to → Langfuse Observability

# Option B — manual
pip install langfuse
hermes plugins enable observability/langfuse
```

### Required environment variables

Set these in `~/.hermes/.env` (or export them before running Hermes):

```bash
HERMES_LANGFUSE_PUBLIC_KEY=pk-lf-...
HERMES_LANGFUSE_SECRET_KEY=sk-lf-...
HERMES_LANGFUSE_BASE_URL=https://cloud.langfuse.com   # or your self-hosted URL
```

### Verifying it works

```bash
hermes plugins list          # look for: observability/langfuse  enabled
hermes chat -q "hello"       # then check Langfuse for a "Hermes turn" trace
```

### Optional tuning

```bash
HERMES_LANGFUSE_ENV=production        # environment label in Langfuse
HERMES_LANGFUSE_RELEASE=v1.0.0        # release tag
HERMES_LANGFUSE_SAMPLE_RATE=0.5       # sample 50% of traces (1.0 = all)
HERMES_LANGFUSE_MAX_CHARS=12000       # max characters per payload field (default: 12000)
HERMES_LANGFUSE_DEBUG=true            # verbose plugin logging to agent.log
```

### Disabling

```bash
hermes plugins disable observability/langfuse
```

---

## Bundled observer plugin: NeMo Relay

NeMo Relay is NVIDIA's runtime layer for agent execution boundaries. The Hermes NeMo Relay plugin maps the full observer hook contract to NeMo Relay scopes, LLM spans, tool spans, mark events, and two trajectory export formats:

- **ATOF** (Agent Trajectory Observability Format) — a JSONL event stream, one event per line, useful for offline debugging and inspection.
- **ATIF** (Agent Trajectory Interchange Format) — a structured JSON trajectory per session, designed for replay, evaluation, and harness analysis; supports nested subagent embedding.

The plugin registers the complete set of observer hooks: all six session hooks, both LLM hooks, all three API hooks, all three tool hooks, both approval hooks, and both subagent hooks.

Like the Langfuse plugin, NeMo Relay fails open: if the `nemo-relay` package is not installed, the plugin loads but all export callbacks are no-ops.

### Enabling NeMo Relay

```bash
hermes plugins enable observability/nemo_relay
```

Note: this must be run against the same `HERMES_HOME` that the agent will use. Setting `HERMES_NEMO_RELAY_*` environment variables alone does not enable the plugin.

### Configuring ATOF and ATIF export

```bash
# Enable ATOF export (JSONL event stream)
export HERMES_NEMO_RELAY_ATOF_ENABLED=1
export HERMES_NEMO_RELAY_ATOF_OUTPUT_DIRECTORY=.nemo-relay/atof

# Enable ATIF export (structured trajectory per session)
export HERMES_NEMO_RELAY_ATIF_ENABLED=1
export HERMES_NEMO_RELAY_ATIF_OUTPUT_DIRECTORY=.nemo-relay/atif
```

For structured multi-component configuration, you can point to a `plugins.toml` file:

```bash
export HERMES_NEMO_RELAY_PLUGINS_TOML=.nemo-relay/plugins.toml
```

Minimal `plugins.toml`:

```toml
version = 1

[[components]]
kind = "observability"
enabled = true

[components.config]
version = 1

[components.config.atof]
enabled = true
output_directory = ".nemo-relay/atof"
filename = "events.jsonl"
mode = "overwrite"

[components.config.atif]
enabled = true
output_directory = ".nemo-relay/atif"
filename_template = "trajectory-{session_id}.json"
agent_name = "Hermes Agent"
agent_version = "local"
```

When `plugins.toml` initialises successfully, NeMo Relay owns the exporter lifecycle through that config and the direct `HERMES_NEMO_RELAY_ATOF_*` fallback variables are skipped.

### Worked example: exporting an ATOF trace

Let's enable ATOF export for a single run, then read the resulting event stream:

```bash
# Enable the plugin once
hermes plugins enable observability/nemo_relay

# Configure ATOF export for this run
export HERMES_NEMO_RELAY_ATOF_ENABLED=1
export HERMES_NEMO_RELAY_ATOF_OUTPUT_DIRECTORY=/tmp/hermes-trace/atof
export HERMES_NEMO_RELAY_ATOF_FILENAME=events.jsonl
export HERMES_NEMO_RELAY_ATOF_MODE=overwrite

# Run a quick conversation
hermes chat --query "List the files in the current directory" --toolsets file --max-turns 4 --quiet

# Inspect the trace
cat /tmp/hermes-trace/atof/events.jsonl
```

Each line in `events.jsonl` is a JSON object with a `kind` field (`scope`, `mark`), a `category` (`llm`, `tool`, `agent`), and a `metadata` block containing `session_id` and other correlation fields. You can join tool spans to LLM spans using the shared `session_id` and `turn_id`.

---

## The skills install audit log

There is one more observability record worth knowing about, separate from the rotating logs: the skills install audit log at `~/.hermes/skills/.hub/audit.log`. Every time the Skills Hub installs, updates, or removes a skill — including the trust-level assigned and the content hash recorded — it appends an entry to this file. This is a plain append-only text log, not a rotating handler.

If you are auditing what skills a running agent has access to, or investigating whether a skill was replaced or tampered with between runs, the audit log is the record to check.

---

## The observability pipeline at a glance

Let's put the two layers together as a diagram — first, how log records flow from components into the four files:

```mermaid
flowchart TD
    A[Agent loop\nagent.*\nrun_agent.*] --> R[LogRecord factory\ninjects session_tag]
    B[Gateway components\ngateway.*\nhermes_plugins.*] --> R
    C[GUI / dashboard\nweb_server\ntui_gateway.*\nuvicorn.*] --> R
    D[Tools\ntools.*] --> R
    R --> RF[RedactingFormatter\nscrubs secrets]
    RF --> AL[agent.log\nINFO+\n5 MB / 3 backups]
    RF --> EL[errors.log\nWARNING+\n2 MB / 2 backups]
    RF -->|mode=gateway| GL[gateway.log\nINFO+ gateway only\n5 MB / 3 backups]
    RF -->|mode=gui| UL[gui.log\nINFO+ gui only\n10 MB / 5 backups]
```

Now the observer hook layer — showing what is built-in versus what requires a plugin:

```mermaid
flowchart LR
    subgraph builtin["Built-in (no plugin needed)"]
        L1[agent.log]
        L2[errors.log]
        L3[gateway.log]
        L4[gui.log]
    end
    subgraph plugins["Via observer plugins (opt-in)"]
        P1[Langfuse plugin\nturns · spans · tool calls]
        P2[NeMo Relay plugin\nATOF JSONL · ATIF trajectories]
        P3[Custom plugin\nwrite your own observer]
    end
    Agent[Hermes agent loop] --> builtin
    Agent -->|observer hooks| PluginManager
    PluginManager --> P1
    PluginManager --> P2
    PluginManager --> P3
```

---

## Edge cases and failure modes

### A secret that `RedactingFormatter` scrubs

Suppose a tool's output accidentally includes an Anthropic API key. The raw string would look like:

```text
Exported key: sk-ant-api03-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

`RedactingFormatter` matches the `sk-ant-` prefix pattern. Because the token is longer than 18 characters, it is partially redacted, preserving the first 6 and last 4 characters of the token body:

```text
Exported key: sk-ant-[REDACTED:sk-ant-...XXXX]
```

The log file receives the redacted form. The full key is never written to disk.

If you need the raw value to appear in logs for development purposes (for example, when working on the redactor itself), you can set `security.redact_secrets: false` in `config.yaml` or `HERMES_REDACT_SECRETS=false` in `~/.hermes/.env`. Hermes logs a warning at startup when redaction is disabled, so operators notice the downgrade.

### What you do NOT get without a plugin

Without an observer plugin enabled, you have no structured span data, no token-usage time series, no per-tool latency metrics, and no trace export. The `hermes logs` CLI gives you text logs you can grep; it does not compute aggregates or feed external dashboards. If your operator workflow expects a Prometheus scrape endpoint or a Grafana datasource, you need to build or configure an observer plugin that exports those formats — none ship pre-wired.

### Observer callback failure

If an observer plugin's callback raises an uncaught exception, Hermes catches it, logs a warning to `agent.log`, and continues. The agent loop is not interrupted. This "fail-open" design means a misconfigured or buggy observability plugin cannot take down a running agent. The cost is that you may silently lose telemetry events for the failed callback; check `agent.log` for `[observer hook error]` entries if a plugin seems to stop producing data.

### Log file not found

If you run `hermes logs` before the agent has ever run, the log files do not exist yet:

```bash
hermes logs
# Log file not found: /Users/you/.hermes/logs/agent.log
# (Logs are created when Hermes runs — try 'hermes chat' first)
```

Run any `hermes chat` command to trigger `setup_logging()` and create the files.

### External log rotation

If an external tool (such as `logrotate`) renames or replaces a log file while Hermes is running, the internal `_ManagedRotatingFileHandler` detects the inode change on the next write and reopens the file at the expected path. Log records continue flowing to the new file without restarting the process. This is handled transparently by comparing the open file descriptor's inode against the current inode of `baseFilename` before each write.

---

## Summary

Hermes observability is two layers, and keeping them distinct prevents confusion:

1. **File logs** — always on, no configuration required, four rotating files under `~/.hermes/logs/`, with `[session_id]` tagging on every line and secret scrubbing before write. Browse them with `hermes logs` using any combination of `--follow`, `--level`, `--session`, `--since`, and `--component`.

2. **Observer hooks** — the extension point for everything beyond file logs. Metrics, spans, trace export, and trajectory recording all come through plugins that register callbacks on lifecycle events. The two bundled options are `observability/langfuse` (Langfuse traces) and `observability/nemo_relay` (ATOF/ATIF export). Neither is active without explicit `hermes plugins enable`.

---

← Previous: [Terminal Backends and the Mixture-of-Agents Tool](../interfaces/terminal-backends-and-moa.md) · Next: [Scenario 1 — From Conversation to Skill Creation](../scenarios/single-agent-conversation-to-skill.md) →
