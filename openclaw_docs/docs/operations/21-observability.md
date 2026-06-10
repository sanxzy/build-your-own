---
title: "Monitoring and Observability: Logs, Debug Flags, OTel, Prometheus, and Health Endpoints"
description: "How to observe a running OpenClaw Gateway: JSONL log rolling, CLI tailing, model-transport debug flags, subsystem diagnostics, OTel and Prometheus extensions, health probes, and openclaw doctor."
category: operations
type: explanation
tags:
  - observability
  - monitoring
  - JSONL logs
  - openclaw logs
  - log tailing
  - OPENCLAW_DEBUG_MODEL_TRANSPORT
  - OPENCLAW_DEBUG_MODEL_PAYLOAD
  - OPENCLAW_DEBUG_SSE
  - diagnostics
  - OpenTelemetry
  - Prometheus
  - healthz
  - readyz
  - openclaw doctor
  - debug flags
  - diagnostics-otel
  - diagnostics-prometheus
  - logging
  - log rotation
  - log rolling
  - file log
  - OTEL_EXPORTER_OTLP_ENDPOINT
  - subsystem diagnostics
  - OPENCLAW_DIAGNOSTICS
  - diagnostics.flags
  - stability recorder
  - gateway health
keywords:
  - openclaw observability
  - openclaw metrics
  - openclaw tracing
  - openclaw log file
  - openclaw debug transport
  - openclaw health check
  - readiness probe
  - liveness probe
  - openclaw doctor first-line diagnostics
  - model payload debug
sources: [S44, S45, S46, S50, S81, S110, S111, S132]
---

**TL;DR** — An OpenClaw Gateway writes structured JSONL logs you can tail live with `openclaw logs --follow`, emit targeted model-transport debug output without raising the global log level, and export full telemetry (metrics, traces, logs) via optional `diagnostics-otel` and `diagnostics-prometheus` extensions. Two HTTP endpoints tell a container orchestrator whether the process is alive (`/healthz`) or ready for traffic (`/readyz`). When something seems wrong, the first step is always `openclaw doctor`.

# Monitoring and Observability: Logs, Debug Flags, OTel, Prometheus, and Health Endpoints

Running an AI gateway is not quite like running a web server. A request can take seconds or minutes, span dozens of tool calls, and touch multiple providers. You need to know not only "did it reply?" but "what model was used, how many tokens did it consume, did a tool call block, and where in the run did latency appear?" This chapter walks through every observation surface OpenClaw provides, building from the simplest first step — reading a log file — up to full OTLP telemetry export.

Before we start, a quick recap of the prerequisite. Log levels and some diagnostic knobs are part of the broader configuration system, documented in [Configuration](./18-configuration.md). The env var `OPENCLAW_LOG_LEVEL` sets the baseline file-log level, and `~/.openclaw/openclaw.json` holds the `logging.*` block. We will refer back to those knobs as we go, but you do not need to re-read that chapter first.

---

## The log file: where everything lands

Let's start with the simplest surface: the log file. The Gateway writes one JSON-line (JSONL) log file per calendar day:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

The date uses the gateway host's local timezone. When the active file reaches `logging.maxFileBytes` (default: **100 MB**), the Gateway rotates it, keeps up to **five** numbered archives alongside the active file (e.g., `openclaw-YYYY-MM-DD.1.log`), and opens a fresh active file. No diagnostic output is ever suppressed — if the file rotates, writing continues immediately.

You can override the path and level in `~/.openclaw/openclaw.json`:

```json
{
  "logging": {
    "file": "/var/log/openclaw/gateway.log",
    "level": "info"
  }
}
```

### What a JSONL log record contains

Each line in the file is a self-contained JSON object. Beyond the per-event data (timestamp, level, subsystem, message text), file-log records include these **machine-filterable top-level fields** when the log call carries the relevant context:

| Field        | What it holds                                                    |
|--------------|------------------------------------------------------------------|
| `hostname`   | The gateway host's hostname                                      |
| `agent_id`   | The active agent ID when the log call carries agent context       |
| `session_id` | The active session ID or key when the log call carries session context |
| `channel`    | The active channel (e.g., `telegram`, `discord`) when carried   |
| `message`    | Flattened log message text, useful for full-text search          |
| `traceId`    | W3C trace ID when a valid diagnostic trace context is active     |
| `spanId`     | Span ID, same condition                                          |
| `parentSpanId` | Parent span ID, same condition                                 |
| `traceFlags` | OpenTelemetry trace flags, same condition                        |

The trace fields (`traceId`, `spanId`, etc.) appear only when a diagnostic trace context is active. When you also run the `diagnostics-otel` extension (described below), those fields let external log processors join a JSONL line with its corresponding OTLP span — correlating local file logs with provider `traceparent` propagation — without ever logging raw prompt or response content.

Talk, realtime voice, and managed-room records also flow through this same pipeline. They include event type, mode, transport, provider, and size/timing measurements, but omit transcript text, audio payloads, turn IDs, and call IDs.

### Tailing logs live

To follow the log file as the Gateway produces it:

```bash
openclaw logs --follow
```

The CLI connects to the running Gateway via RPC and streams parsed, structured output. On a TTY you get pretty, colorized, human-readable lines; in non-TTY mode (CI, scripts) you get plain text. Useful options:

```bash
openclaw logs --follow --local-time        # timestamps in your local timezone
openclaw logs --follow --json              # one JSON object per line (for piping)
openclaw logs --follow --plain             # force plain text in a TTY session
openclaw logs --follow --no-color          # strip ANSI colors
```

In JSON mode the CLI emits typed objects: `meta` (stream metadata), `log` (parsed entry), `notice` (truncation or rotation hints), and `raw` (an unparsed line). You can also filter by channel:

```bash
openclaw channels logs --channel telegram
```

If the Gateway is unreachable, `openclaw logs --follow` falls back to the local log file on Linux (via systemd journal by PID when available) and prints a hint:

```bash
openclaw doctor
```

That hint is the right instinct: when you are not sure what is wrong, `openclaw doctor` is always the first step. We will cover it at the end of this chapter.

### Log levels: file vs. console

These two surfaces are configured independently:

| Setting               | What it controls                      | Default |
|-----------------------|---------------------------------------|---------|
| `logging.level`       | Detail level written to the JSONL file | `info`  |
| `logging.consoleLevel`| What appears in the terminal          | `info`  |
| `OPENCLAW_LOG_LEVEL`  | Override for both, takes precedence over config | — |

`--verbose` (the CLI flag) only affects **console verbosity** and WebSocket protocol log style. It does **not** raise the file log level. To capture verbose-only information in the log file you need `logging.level: "debug"` or `"trace"`. The `debug` level writes extra detail on most subsystems; `trace` also includes diagnostic timing summaries for selected hot paths such as plugin tool factory preparation.

Console formatting is controlled separately:

| `logging.consoleStyle` | Output shape |
|------------------------|--------------|
| `pretty`               | Human-friendly, colorized, with timestamps (default) |
| `compact`              | Tighter format, better for long sessions |
| `json`                 | One JSON object per line, for log processors |

---

## Targeted model-transport debug flags

Here is the problem with `logging.level: "debug"`: it floods every subsystem. When you are debugging a specific provider call — say, an unexpected tool-routing choice or an unexplained latency spike — you want to see only what the model transport layer is doing, not the entire gateway.

OpenClaw solves this with a set of targeted environment flags that emit model-transport diagnostics at `info` level without touching the global log level.

### Model-transport flags

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
```

Set `OPENCLAW_DEBUG_MODEL_TRANSPORT=1` to emit request-start, fetch-response, SDK headers, first-streaming-event, stream-completion, and transport errors at `info` level. These appear in the normal log file and in `openclaw logs --follow`, so you do not need to change any config or restart with a different log level.

Combine with a payload flag to see more of the request shape:

```bash
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=summary openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted openclaw gateway
```

| `OPENCLAW_DEBUG_MODEL_PAYLOAD` value | What appears in the log |
|--------------------------------------|-------------------------|
| `summary`    | A bounded summary of the request payload (no raw content) |
| `tools`      | All model-facing tool names in the payload summary |
| `full-redacted` | A redacted, capped JSON snapshot of the full request payload |

**On `full-redacted` specifically:** the redaction applies the same `logging.redactSensitive` policy that covers console, file-log, OTLP log-record, and session-transcript sinks. Secrets, API keys, and values matching `logging.redactPatterns` are masked before the snapshot is written. However, **prompt and message text may still be present**. The "redacted" label means secrets are removed — it does not mean conversation content is stripped. Use this flag only while actively debugging, not in a long-running production gateway.

For SSE stream inspection:

```bash
OPENCLAW_DEBUG_SSE=events openclaw gateway   # first-event + stream-completion timing
OPENCLAW_DEBUG_SSE=peek   openclaw gateway   # also emits first 5 redacted SSE payloads, capped per event
```

One more flag for code-mode debugging:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 openclaw gateway  # emits code-mode model-surface diagnostics
```

All of these flags route output through normal OpenClaw logging, so `openclaw logs --follow` and the Control UI Logs tab show them automatically. Without the flags, the same diagnostics are still recorded at `debug` level, which means raising `logging.level` to `debug` gives you the same data — but also everything else at debug level.

---

## Subsystem-scoped diagnostics flags

Now we hit a more interesting problem. You have a conversation stuck somewhere in the Telegram channel. You have raised `OPENCLAW_LOG_LEVEL=debug`, but the log is noisy and the signal is buried. What you want is: *extra diagnostic output scoped to one subsystem, without touching anything else*.

That is what `diagnostics.flags` — and its env-var companion `OPENCLAW_DIAGNOSTICS` — provides.

Think of it this way: `OPENCLAW_LOG_LEVEL=debug` is like turning up the volume on every instrument in an orchestra simultaneously. `diagnostics.flags` is like asking the brass section to play louder, leaving everything else unchanged.

### Enabling diagnostics flags

In `~/.openclaw/openclaw.json`:

```json5
{
  "diagnostics": {
    "enabled": true,
    "flags": ["telegram.http"]
  }
}
```

Or as a one-off environment override (no config change, no restart):

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

Flags are **case-insensitive** and support **wildcards**:

| Flag pattern | What it targets |
|---|---|
| `telegram.http` | Telegram HTTP transport diagnostics |
| `gateway.*` | All gateway-namespace diagnostics |
| `*` | Every subsystem (use sparingly) |
| `profiler` | Profiler diagnostics |
| `timeline` | Timeline diagnostics |

Flag output lands in the standard log file (`logging.file`) and is subject to `logging.redactSensitive`. No separate file, no separate process — it is only more structured detail routed through the existing pipeline.

### What diagnostics flags give you that `OPENCLAW_LOG_LEVEL` alone does not

`OPENCLAW_LOG_LEVEL=debug` raises the floor for *everything* — every subsystem emits debug output, which can be thousands of lines per second on a busy gateway. Diagnostics flags let you:

1. **Target a specific subsystem** without changing anything else. If Telegram's HTTP transport is misbehaving, you get Telegram HTTP diagnostics without debug noise from session management, context assembly, or tool execution.
2. **Activate per-subsystem diagnostic modes** that are not merely "log at debug level" — they surface structured operational data (HTTP request metadata, gateway-level protocol timing, profiling events) that the debug log level does not produce.
3. **Leave the file log level unchanged** so existing log-based alerting and monitoring keep working without suddenly drowning in debug events.

In short: `OPENCLAW_LOG_LEVEL=debug` is a blunt instrument; `diagnostics.flags` is surgical.

---

## Redaction: what gets masked

OpenClaw applies a redaction policy before log or transcript output leaves the process. This covers:

- Console output
- The JSONL log file
- OTLP log records
- Session transcript text
- Control UI tool-call event payloads (tool-start args, partial/final results, exec output, patch summaries)

Configuration:

```json
{
  "logging": {
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

| `redactSensitive` value | Effect |
|---|---|
| `tools` (default) | Redact common API credentials, payment-credential fields, and matches from `redactPatterns` |
| `off` | Disable the general log/transcript redaction policy |

Setting `off` does **not** disable safety-boundary redaction. The following surfaces always redact regardless of the setting: Control UI tool-call events, `sessions_history` tool output, diagnostics support exports, provider error observations, exec approval command display, and Gateway WebSocket protocol logs. Custom `logging.redactPatterns` can add project-specific patterns on top of those surfaces.

Default patterns cover common API key shapes, bearer headers, PEM blocks, popular token prefixes, and payment credential field names. Matched values are masked by keeping the first 6 and last 4 characters when the value is at least 18 characters long; shorter values become `***`.

---

## OpenTelemetry export: `diagnostics-otel`

So far we have talked about local log files and debug flags — observation tools you use while sitting at a terminal. For production deployments you want to push metrics, traces, and logs to a collector or backend (Grafana, Datadog, Honeycomb, New Relic, Tempo) so you can build dashboards and alerts without SSHing into the gateway host.

OpenClaw exports diagnostics through the official `diagnostics-otel` plugin using **OTLP/HTTP (protobuf)**. Any collector that accepts OTLP/HTTP works without code changes.

### How it fits together

The Gateway and its bundled plugins emit **diagnostics events** — structured, in-process records for model runs, message flow, sessions, queues, and tool executions. The `diagnostics-otel` plugin subscribes to those events and exports them as OpenTelemetry **metrics**, **traces**, and **logs** over OTLP/HTTP. Exporters only attach when both `diagnostics.enabled: true` and the plugin are active, so the in-process cost stays near zero by default.

### Installing and enabling

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
openclaw plugins enable diagnostics-otel
```

Then configure in `~/.openclaw/openclaw.json`:

```json5
{
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": { "enabled": true }
    }
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw-gateway",
      "traces": true,
      "metrics": true,
      "logs": false,
      "sampleRate": 1.0,
      "flushIntervalMs": 60000
    }
  }
}
```

> **Note:** `protocol` currently supports `http/protobuf` only. The `grpc` value is accepted but ignored.

### What gets exported

| Signal | What it contains |
|--------|------------------|
| **Metrics** | Counters and histograms for token usage, cost, run duration, failover, skill usage, message flow, queue lanes, session state and recovery, tool execution, oversized payloads, exec, and memory pressure |
| **Traces** | Spans for model usage, model calls, harness lifecycle, skill usage, tool execution, exec, webhook/message processing, context assembly, and tool loops |
| **Logs** | Structured JSONL file-log records exported over OTLP when `diagnostics.otel.logs: true`; log bodies are withheld unless `captureContent` is explicitly enabled |

Toggle each signal independently. Traces and metrics default to `true` when `diagnostics.otel.enabled` is true. Logs default to `false`.

### Environment variable overrides

| Variable | Purpose |
|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Override `diagnostics.otel.endpoint` |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Signal-specific traces endpoint |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | Signal-specific metrics endpoint |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | Signal-specific logs endpoint |
| `OTEL_SERVICE_NAME` | Override `diagnostics.otel.serviceName` |
| `OTEL_SEMCONV_STABILITY_OPT_IN` | Set to `gen_ai_latest_experimental` for the latest GenAI inference span shape |
| `OPENCLAW_OTEL_PRELOADED` | Set to `1` when another process already registered the global OpenTelemetry SDK |

Signal-specific config wins over signal-specific env, which wins over the shared endpoint.

### Privacy model for OTLP export

Raw prompt and tool content is **not** exported by default. Spans carry bounded identifiers (channel, provider, model, error category, hash-only request IDs, tool source, tool owner, skill name/source) and never include prompt text, response text, tool inputs, tool outputs, skill file paths, or session keys. OTLP log records keep severity, logger, code location, trusted trace context, and sanitized attributes — the raw log message body is exported only when `diagnostics.otel.captureContent` is explicitly `true`.

To opt into content capture for specific classes:

```json5
{
  "diagnostics": {
    "otel": {
      "captureContent": {
        "inputMessages": true,
        "outputMessages": false,
        "toolInputs": false,
        "toolOutputs": false,
        "systemPrompt": false,
        "toolDefinitions": false
      }
    }
  }
}
```

Each subkey is opt-in independently. Enable content capture only when your collector and retention policy are approved for prompt, response, tool, or system-prompt text.

### Sampling and flushing

- **Traces:** `diagnostics.otel.sampleRate` (root-span only, `0.0` drops all, `1.0` keeps all).
- **Metrics:** exported on `diagnostics.otel.flushIntervalMs` (minimum 1000 ms).
- **Logs:** respect `logging.level` (the file-log level). High-volume installs should prefer OTLP collector-side sampling/filtering over local sampling.

---

## Prometheus metrics: `diagnostics-prometheus`

If your stack is already built around Prometheus and Grafana, you may not want to run an OTLP collector at all. The `diagnostics-prometheus` plugin exposes a pull endpoint that Prometheus can scrape directly.

The endpoint is:

```text
GET /api/diagnostics/prometheus
```

Content type: `text/plain; version=0.0.4; charset=utf-8` (standard Prometheus exposition format).

**The route uses Gateway authentication (operator scope).** Do not expose it as a public unauthenticated `/metrics` endpoint. Scrape it through the same auth path you use for other operator APIs.

### Installing and enabling

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-prometheus
openclaw plugins enable diagnostics-prometheus
```

Config:

```json5
{
  "plugins": {
    "allow": ["diagnostics-prometheus"],
    "entries": {
      "diagnostics-prometheus": { "enabled": true }
    }
  },
  "diagnostics": {
    "enabled": true
  }
}
```

Restart the Gateway so the HTTP route registers at plugin startup.

Scrape it:

```bash
curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
  http://127.0.0.1:18789/api/diagnostics/prometheus
```

Prometheus scrape config:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: openclaw
    scrape_interval: 30s
    metrics_path: /api/diagnostics/prometheus
    authorization:
      credentials_file: /etc/prometheus/openclaw-gateway-token
    static_configs:
      - targets: ["openclaw-gateway:18789"]
```

> **Note:** `diagnostics.enabled: true` is required. Without it, the plugin still registers the HTTP route but no diagnostic events flow into the exporter, so the response is empty.

### Selected metrics and PromQL recipes

The plugin exports counters, gauges, and histograms covering runs, model calls, token usage, cost, tool execution, session state, queue depth, and liveness. A few useful starting queries:

```promql
# Token usage per minute, split by provider
sum by (provider) (rate(openclaw_model_tokens_total[1m]))

# Spend (USD) over the last hour, by model
sum by (model) (increase(openclaw_model_cost_usd_total[1h]))

# 95th percentile model run duration
histogram_quantile(
  0.95,
  sum by (le, provider, model)
    (rate(openclaw_run_duration_seconds_bucket[5m]))
)

# Queue wait time (95p under 2 s)
histogram_quantile(
  0.95,
  sum by (le, lane) (rate(openclaw_queue_lane_wait_seconds_bucket[5m]))
) < 2

# Cardinality alarm: dropped series
increase(openclaw_prometheus_series_dropped_total[15m]) > 0
```

### Label policy and cardinality cap

Prometheus labels stay bounded and low-cardinality. Raw diagnostic identifiers (`runId`, `sessionKey`, `callId`, message IDs) never appear as label values. Labels exceeding the character policy are replaced with `unknown`, `other`, or `none`.

The exporter caps retained time series at **2048** across counters, gauges, and histograms combined. New series beyond the cap are dropped and `openclaw_prometheus_series_dropped_total` increments. Watch this counter — a climbing value means a label upstream is leaking high-cardinality values. The exporter never lifts the cap automatically; fix the source.

### Prometheus vs. OpenTelemetry: choosing a path

| | `diagnostics-prometheus` | `diagnostics-otel` |
|---|---|---|
| Model | Pull (Prometheus scrapes) | Push (OpenClaw sends OTLP) |
| External collector required | No | Yes (or OTLP-native backend) |
| Signals | Metrics only | Metrics + Traces + Logs |
| Auth | Gateway operator scope | OTLP endpoint credentials |
| Best for | Stacks standardized on Prometheus + Grafana | Full observability backends (Grafana, Datadog, Honeycomb, etc.) |

You can run both independently, or bridge `diagnostics-otel` to Prometheus through an OpenTelemetry Collector's `prometheus` or `prometheusremotewrite` exporter.

---

## Health endpoints: liveness vs. readiness

When you deploy the Gateway in a container or behind a load balancer, the orchestrator needs two answers:

- "Is the process alive?" — the **liveness** probe.
- "Can it handle traffic right now?" — the **readiness** probe.

Think of it like a restaurant. A liveness probe is a pulse check: the chef is present and breathing. A readiness probe is the "Open" sign on the door: the kitchen is set up, the staff are in place, and the restaurant is ready to seat customers. You can be alive but not yet ready for business.

OpenClaw exposes both as unauthenticated HTTP endpoints on the same port as the Gateway (default 18789):

<!-- GAP: docs/gateway/health.md (S132) covers CLI health commands but does not explicitly define the HTTP /healthz and /readyz endpoint semantics; the following distinction is sourced from the broader corpus (docs/cli/gateway.md, docs/install/docker.md) which are outside assigned sources; marking accordingly -->

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz    # readiness
```

| Endpoint | What it checks | When it returns 200 |
|---|---|---|
| `GET /healthz` | Liveness: can the HTTP server accept a connection? | As soon as the HTTP server is up and able to answer |
| `GET /readyz` | Readiness: is the Gateway ready for traffic? | After startup plugin sidecars, channels, and configured hooks have finished settling |

The distinction matters during startup and restart. The liveness probe returns 200 quickly; the readiness probe stays "not ready" while the Gateway is still loading plugins and connecting channels. A container orchestrator that uses only `healthz` may start routing traffic before the Gateway can actually handle it. A readiness probe on `readyz` prevents that.

In Docker deployments, the built-in `HEALTHCHECK` directive in the OpenClaw image pings `/healthz`. For Kubernetes, configure a separate readiness probe on `/readyz` and a liveness probe on `/healthz`.

Local and authenticated detailed readiness responses include an `eventLoop` diagnostic block with event-loop delay, event-loop utilization, CPU core ratio, and a `degraded` flag.

The CLI health commands (`openclaw health` and `openclaw health --verbose`) use the Gateway's RPC health snapshot rather than these HTTP probes directly. For programmatic health checks in scripts:

```bash
# liveness: exits non-zero if the server is unreachable
curl -fsS http://127.0.0.1:18789/healthz

# readiness: check after startup before routing traffic
curl -fsS http://127.0.0.1:18789/readyz
```

---

## `openclaw doctor`: first-line diagnostics

We have covered log files, debug flags, subsystem diagnostics, OTel, Prometheus, and health endpoints. When something goes wrong in production, the instinct is often to jump straight to log inspection or to add more debug flags. Resist that instinct.

`openclaw doctor` is designed to be the **first thing you run**. It checks health, inspects config, detects stale state, verifies auth profiles, audits supervisor configuration, reports pending security warnings, and provides actionable repair steps — all in a single command:

```bash
openclaw doctor
```

### What doctor checks

Doctor is a broad repair-and-inspection tool. Key checks relevant to observability:

- **Gateway health and restart**: runs a health probe; offers to restart when the Gateway looks unhealthy.
- **Config normalization and migrations**: detects legacy config keys and offers to migrate them.
- **Session lock cleanup**: finds stale lock files left by abnormal exits.
- **State integrity**: verifies the state directory, session files, and transcript files are intact and writable.
- **Model auth health**: inspects OAuth token expiry, cooldowns, and disabled auth profiles.
- **Plugin status**: counts enabled/disabled/errored plugins; reports load-time warnings.
- **Channel status warnings**: runs a channel-status probe and reports connectivity issues.
- **Supervisor config audit**: checks launchd/systemd/schtasks config for outdated defaults.

If the CLI log tail falls back to a static file and prints a hint to run `openclaw doctor`, that is the system telling you it cannot reach the Gateway. Doctor will tell you why.

### Headless and automation modes

```bash
openclaw doctor           # interactive: prompts before writing
openclaw doctor --fix     # apply recommended repairs without prompts
openclaw doctor --lint    # read-only: structured findings for CI/preflight
openclaw doctor --yes     # accept defaults without prompting
openclaw doctor --deep    # also scan system services for extra gateway installs
```

`--lint` mode is the CI-friendly variant. It never writes config or state, emits structured JSON findings, and exits non-zero when findings meet the selected severity threshold:

```bash
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
```

### Diagnostics export for bug reports

When you have reproduced a problem and need to file a bug report, run:

```bash
openclaw gateway diagnostics export
```

This creates a zip file containing: a human-readable `summary.md`, a machine-readable `diagnostics.json`, sanitized config shape, sanitized log summaries and redacted recent log lines, a Gateway status/health snapshot, and the newest persisted stability bundle (when available). The export is designed to be shareable: chat text, webhook bodies, tool outputs, credentials, account IDs, and secret values are omitted or redacted.

You can also trigger this from within a conversation:

```bash
# Send this as a chat message to the agent:
/diagnostics bad tool choice
```

The agent will ask for one explicit exec approval, run the export, and reply with the bundle path and a summary.

The Gateway also records a bounded, payload-free **stability stream** by default when `diagnostics.enabled: true`. Inspect it live:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
```

After a crash or restart failure, inspect the persisted bundle:

```bash
openclaw gateway stability --bundle latest
```

Persisted bundles live under `~/.openclaw/logs/stability/` when events exist.

---

## Putting it together: a troubleshooting sequence

When something is not working as expected:

1. **Start with `openclaw doctor`** — it will catch most common issues (stale config, unhealthy gateway, auth drift, channel connectivity) and tell you what to fix.
2. **Tail the log file** — `openclaw logs --follow` gives you a live stream of structured events. Look for `error` or `warn` lines near the time of the failure.
3. **Add targeted debug flags** — if you suspect a specific subsystem, use `OPENCLAW_DEBUG_MODEL_TRANSPORT=1` or `OPENCLAW_DIAGNOSTICS=<subsystem>` to get more detail without flooding the log.
4. **Use `openclaw health --verbose`** — for a live channel probe, checking per-account connectivity and recent session activity.
5. **Export a diagnostics bundle** — `openclaw gateway diagnostics export` when you need to share findings or inspect a past incident.
6. **Escalate to OTLP or Prometheus** — for sustained production monitoring, wire up `diagnostics-otel` or `diagnostics-prometheus` so you have dashboards and alerting before the next incident.

---

← Previous: [Security and Governance: Pairing, Auth Modes, Sandbox, and Network Policy](./20-security.md) · Next: [Project Structure: Monorepo Layout, 21 Packages, and Extension Inventory](../reference/22-project-structure.md) →
