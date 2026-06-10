---
title: "Best Practices: Configuration, Security, Tool Policy, Sessions, Observability"
description: "Field-tested operating guidance for OpenClaw, organized by area: configuration, security, tool policy, sessions/queue, and observability."
category: reference
type: how-to
tags: [best-practices, configuration, security, tool-policy, sessions, observability, operations, dmPolicy, sandbox, dmScope, queue, logging, doctor, openclaw-security-audit, thinking, tool-profiles, redaction]
keywords: [hardening, operational guide, openclaw.json, deployment hygiene, access control, prompt injection, pairing, queue modes, session isolation, log redaction, doctor lint, run queue, concurrency, context budget]
sources: [S1, S19, S22, S37, S38, S39, S40, S42, S44, S47, S74, S77, S124]
---

**TL;DR** — This chapter collects field-tested operating guidance across five areas: writing stable configuration, hardening the security surface, setting a safe tool policy, tuning sessions and the run queue, and wiring up observability. Each recommendation starts from the failure it prevents, then shows the concrete config or command that addresses it.

# Best Practices: Configuration, Security, Tool Policy, Sessions, Observability

OpenClaw connects a language model to real messaging surfaces and real host tools. That combination is useful precisely because it has teeth — and "having teeth" means a misconfiguration can cause real consequences. The five sections below walk through each operational area the same way: here is the thing that goes wrong, here is what to do about it, here is how to verify it worked.

---

## Area 1: Configuration

### What can go wrong

Let's start with configuration, because getting it wrong here underlies most other failure modes. The Gateway's configuration file (`~/.openclaw/openclaw.json`) is large, and many settings interact. The most common failure modes are:

- Stale keys from an older version that silently stop doing what you expect.
- Unsafe config mutations made directly to the file that break schema validation.
- Config that works today but fails after an update because the schema evolved.

Think of `openclaw.json` as the schematics for a complex machine. If one wiring label changes without updating the diagram, you get surprising behavior, not an obvious error.

*If you haven't set up configuration yet, see the full reference at [Configuration System](../operations/18-configuration.md).*

### Use `openclaw doctor` after every config change

The `openclaw doctor` command is OpenClaw's repair and migration tool. It normalizes legacy keys, checks health, migrates stale config shapes, and provides actionable repair steps.

```bash
# Normal health check (interactive, human-readable)
openclaw doctor

# Apply recommended repairs automatically
openclaw doctor --fix

# Read-only check suitable for CI or automated preflight
openclaw doctor --lint
openclaw doctor --lint --json
```

The lint mode returns structured findings with a `checkId`, `severity`, and a `fixHint` per finding. Exit code `0` means no findings at or above the selected severity; `1` means findings exist; `2` is a runtime failure.

```bash
# Only fail on errors, not warnings
openclaw doctor --lint --severity-min error

# Narrow gate for a specific check
openclaw doctor --lint --only core/doctor/gateway-config
```

Run `openclaw doctor` after upgrades. Gateway startup explicitly refuses legacy config formats and will ask you to run `openclaw doctor --fix` rather than silently accepting a stale shape.

### Edit config through the CLI, not the raw file

The `openclaw config set` command applies additive, schema-validated writes and refuses destructive replacements unless you pass `--replace`:

```bash
# Safe: add a model entry without wiping existing entries
openclaw config set agents.defaults.models '{"openai/gpt-5.5": {"alias": "gpt"}}' --strict-json --merge

# Safe: add a custom provider
openclaw config set models.providers.my-proxy '{"baseUrl": "http://localhost:4000/v1"}' --strict-json --merge
```

Direct JSON editing is fine for big structural changes, but always run `openclaw doctor --lint` afterward to catch schema problems before the Gateway sees them.

### Keep file permissions tight

```bash
# Config should be readable only by your user
chmod 600 ~/.openclaw/openclaw.json
chmod 700 ~/.openclaw
```

`openclaw doctor` warns when these permissions are wider than `600`/`700` and can tighten them automatically with `--fix`.

### Use caret ranges in any `package.json` examples

When writing documentation or example configs that include internal package references, use caret ranges (`^0.1.0`) rather than exact pins. This keeps examples forward-compatible across minor updates.

### Verify config with `openclaw status`

```bash
openclaw status --all
```

This output is pasteable and has secrets redacted. Prefer it over sharing raw log files when asking for help or troubleshooting.

---

## Area 2: Security

Security in OpenClaw is structured around three questions: who can talk to the bot, where the bot is allowed to act, and what the bot can touch. Notice that the failure mode is almost never a sophisticated exploit — it's "someone messaged the bot and the bot did what they asked." We'll work through each question in turn.

*For the full security model see [Security and Governance](../operations/20-security.md).*

### Run the security audit regularly

```bash
openclaw security audit
openclaw security audit --deep   # also runs a live Gateway probe
openclaw security audit --fix    # auto-corrects common misconfigs
```

The audit checks inbound access policies, tool blast radius, exec filesystem drift, network exposure, browser control exposure, local disk hygiene, and plugin policy. Each finding has a structured `checkId` (e.g. `gateway.bind_no_auth`, `tools.exec.security_full_configured`) so you can reference it exactly.

Run this especially after:
- Adding a new messaging channel.
- Widening a DM policy.
- Changing the gateway bind address.
- Installing a plugin.

### Lock down DM access before anything else

By default, DMs on Telegram, WhatsApp, Discord, Slack, iMessage, and Signal use `dmPolicy: "pairing"`. An unknown sender receives a short pairing code; the bot does not process their message until you approve them.

The risk of **not** doing this: any stranger who knows or guesses your bot's address can send it messages and attempt to trigger tool calls, exfiltrate context, or probe for credentials through prompt injection.

```json5
// openclaw.json — explicit pairing (also the default, but stating it is good hygiene)
{
  "channels": {
    "whatsapp": { "dmPolicy": "pairing" },
    "telegram": { "dmPolicy": "pairing" }
  }
}
```

Approve a pending pairing request:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <code>
```

If you want a truly open inbox (`dmPolicy: "open"`), you must also add `"*"` to the channel `allowFrom` list explicitly — the double opt-in is intentional. Treat `dmPolicy: "open"` as a last resort; prefer pairing or allowlists.

`openclaw doctor` will warn when DM policies are open without an allowlist.

### Isolate DM sessions when more than one person can message you

By default, all DMs share one session — that is fine when you are the only user. The moment a second person can DM your bot, you have a context-leakage problem: Alice's conversation is visible to Bob in the shared session context.

```json5
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

`dmScope` options:

| Value | Scope |
|---|---|
| `main` | All DMs share one session (default) |
| `per-peer` | One session per sender, across all channels |
| `per-channel-peer` | One session per channel + sender pair (recommended for multi-user) |
| `per-account-channel-peer` | One session per account + channel + sender (multi-account channels) |

Verify with `openclaw security audit`: it flags shared DM sessions when more than one sender has access.

### Start with a hardened baseline

The following template gives you a local-only Gateway with DM isolation and runtime tools disabled by default. Selectively re-enable what you actually need:

```json5
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "auth": { "mode": "token", "token": "replace-with-long-random-token" }
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "tools": {
    "profile": "messaging",
    "deny": ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    "fs": { "workspaceOnly": true },
    "exec": { "security": "deny", "ask": "always" },
    "elevated": { "enabled": false }
  },
  "channels": {
    "whatsapp": {
      "dmPolicy": "pairing",
      "groups": { "*": { "requireMention": true } }
    }
  }
}
```

Generate a gateway token if you don't have one:

```bash
openclaw doctor --generate-gateway-token
```

### Keep Gateway auth enabled — it is required by default

Gateway auth is fail-closed: if no valid auth path is configured, the Gateway refuses WebSocket connections. Onboarding generates a token for you. Do not disable it.

Auth modes:

| Mode | When to use |
|---|---|
| `token` | Recommended for most setups — shared bearer token |
| `password` | Alternative; prefer setting via `OPENCLAW_GATEWAY_PASSWORD` env var |
| `trusted-proxy` | When an identity-aware reverse proxy authenticates users |

If you use a reverse proxy (nginx, Caddy, Traefik), configure `gateway.trustedProxies` with your proxy IPs. Ensure your proxy **overwrites** `X-Forwarded-For` rather than appending to it:

```nginx
# Good: overwrites
proxy_set_header X-Forwarded-For $remote_addr;

# Bad: appends untrusted header
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### Use the strongest available model for tool-enabled agents

Prompt injection resistance varies significantly across model tiers. Smaller or older models are substantially more susceptible to instruction hijacking under adversarial prompts.

For any agent that runs tools, reads untrusted content (web pages, email, attachments), or accepts messages from more than one person: use the best modern instruction-hardened model you can access. If you must use a smaller model, reduce blast radius with sandboxing, read-only tools, and strict allowlists.

### Sandbox non-main sessions for group/channel safety

When a session is not your personal main DM session, it likely came from a group chat or a channel where other people are involved. Run those sessions in a sandbox:

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "backend": "docker"
      }
    }
  },
  "tools": {
    "sandbox": {
      "tools": {
        "allow": [
          "exec", "process", "read", "write", "edit",
          "sessions_list", "sessions_history", "sessions_send", "sessions_spawn"
        ],
        "deny": ["browser", "canvas", "nodes", "cron", "discord", "gateway"]
      }
    }
  }
}
```

`mode: "non-main"` sandboxes every session whose key is not `"main"`. Group and channel sessions get their own keys, so they are automatically sandboxed. Your personal DM session stays on the host.

Sandbox mode is `off` by default. When it is off and you set `tools.exec.host: "sandbox"`, that fails closed — no sandbox runtime is available. Set `host: "gateway"` explicitly if you want the tool to run on the gateway host without a sandbox.

### Keep sensitive keys out of workspace `.env` files

OpenClaw blocks provider credential environment variables (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, etc.) and any key starting with `OPENCLAW_*` from being inherited from workspace-local `.env` files. Those files often live next to agent code, get committed accidentally, or can be written by tools. Put provider credentials in:

- The Gateway process environment.
- `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`).
- The config `env` block.
- Or optional login-shell import.

### Deny control-plane tools for agents that handle untrusted content

Two built-in tools can make persistent config changes: `gateway` (can call `config.apply`, `config.patch`) and `cron` (can create scheduled jobs that keep running). For any agent or surface that handles untrusted content:

```json5
{
  "tools": {
    "deny": ["gateway", "cron", "sessions_spawn", "sessions_send"]
  }
}
```

### Enable log redaction

```json5
{
  "logging": {
    "redactSensitive": "tools"
  }
}
```

`tools` (the default) redacts sensitive tokens from tool-call events, `sessions_history` output, and diagnostics. Add custom patterns for your environment:

```json5
{
  "logging": {
    "redactSensitive": "tools",
    "redactPatterns": ["my-internal-hostname", "sk-[A-Za-z0-9]+"]
  }
}
```

`redactSensitive: "off"` does not make all safety surfaces emit raw secrets — certain surfaces (Control UI tool-call events, exec approval display) always redact.

---

## Area 3: Tool Policy

The failure mode we're guarding against here is straightforward: an over-permissive tool policy lets an agent run dangerous commands or access file system areas it should never touch. Once a tool is available to the model, it can be invoked — whether you intended that or not. So let's think carefully about what we actually need to enable.

Tool policy determines what the model can actually do. OpenClaw applies tool policy in layers, and the order matters: base profile → `tools.allow`/`tools.deny` → per-agent overrides.

*The full tool registration and policy system is described in [Tool System](../extending/12-tool-system.md).*

### Use a named profile as your starting point

Rather than building an allow/deny list from scratch, start with a named profile and layer on top:

| Profile | What it includes |
|---|---|
| `minimal` | `session_status` only |
| `coding` | Filesystem, runtime, web, sessions, memory, cron, image, video tools |
| `messaging` | Messaging tools + basic session tools |
| `full` | No restriction |

Local onboarding defaults to `tools.profile: "coding"`. For agents handling untrusted input, start with `messaging` and add only what you need:

```json5
{
  "tools": {
    "profile": "messaging",
    "allow": ["web_search", "memory_search"]
  }
}
```

### Know the tool groups

OpenClaw defines named groups you can reference in allow/deny lists:

| Group | Contains |
|---|---|
| `group:runtime` | `exec`, `process`, `code_execution` |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:web` | `web_search`, `x_search`, `web_fetch` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:media` | `image`, `image_generate`, `music_generate`, `video_generate`, `tts` |

Deny groups rather than individual tool names when you want to close a category entirely:

```json5
{
  "tools": {
    "deny": ["group:runtime", "group:fs", "group:automation"]
  }
}
```

Note: `deny: ["write"]` does **not** deny `apply_patch`. To block all file mutation, deny `group:fs` or list each tool explicitly: `["write", "edit", "apply_patch"]`.

### Use `toolsBySender` for per-sender restrictions

`tools.toolsBySender` applies defense-in-depth on top of channel access control. Keys use explicit prefixes:

```json5
{
  "tools": {
    "toolsBySender": {
      "channel:discord:1234567890123": { "alsoAllow": ["group:fs"] },
      "id:guest-user-id": { "deny": ["group:runtime", "group:fs"] },
      "*": { "deny": ["exec", "process", "write", "edit", "apply_patch"] }
    }
  }
}
```

The `"*"` wildcard acts as a default for any sender not matched by a more specific key. Use it to set a restrictive floor and lift restrictions only for trusted identities.

### Set a sessions visibility scope

`tools.sessions.visibility` controls which sessions the session tools (`sessions_list`, `sessions_history`, `sessions_send`) can target:

| Value | Scope |
|---|---|
| `self` | Only the current session key |
| `tree` | Current session + spawned subagents (default) |
| `agent` | Any session under the current agent id |
| `all` | Any session (cross-agent targeting also requires `tools.agentToAgent`) |

`tree` is the right default for most agents. Use `agent` only when you have a deliberate reason to let an agent see all its own sessions. `all` is almost never the right choice for an agent that handles untrusted input.

### Configure thinking level defaults explicitly

Thinking level controls how much extended reasoning the model uses. If you don't set it, the provider default applies, which may be more expensive than you want for routine workloads.

*Thinking directives are covered in detail in [Model Integration](../models/15-model-integration.md).*

Set a global default for all agents:

```json5
{
  "agents": {
    "defaults": {
      "model": {
        "thinkingDefault": "low"
      }
    }
  }
}
```

Thinking level resolution order (the first match wins):

1. Inline directive on the message (e.g. `/think high`).
2. Stored session override.
3. Per-agent default (`agents.list[].thinkingDefault`).
4. Global default (`agents.defaults.thinkingDefault`).
5. Provider-declared fallback, or `medium` for reasoning-capable models.

Use `/think off` for bulk-processing agents that do not need extended reasoning. Use `adaptive` for Claude 4.6 models when you want the provider to decide how much thinking the task warrants.

### Enable loop detection for long-running agentic runs

Tool-loop detection is disabled by default. Enable it for agents doing long autonomous work:

```json5
{
  "tools": {
    "loopDetection": {
      "enabled": true,
      "historySize": 30,
      "warningThreshold": 10,
      "criticalThreshold": 20,
      "globalCircuitBreakerThreshold": 30
    }
  }
}
```

The thresholds must be strictly ordered: `warningThreshold < criticalThreshold < globalCircuitBreakerThreshold`. Config validation rejects invalid orderings.

---

## Area 4: Sessions and Queue

A session in OpenClaw is a named conversation context — the bucket into which related messages are routed and history is accumulated. Sessions are stored in SQLite runtime state; the full message transcript is written as a JSONL file. Once you understand what a session is, you'll want to think about three things: when it resets, how much storage it can consume, and what happens when messages arrive faster than the agent can reply.

*Sessions are introduced in [Sessions](../agents/07-sessions.md). The run queue and concurrency system is described in [Run Queue and Concurrency](../agents/08-run-queue.md).*

### Set an idle reset for unattended agents

By default, sessions reset at 4:00 AM on the gateway host (daily reset). For agents that run unattended or accumulate long histories, add an idle reset too:

```json5
{
  "session": {
    "reset": {
      "idleMinutes": 120
    }
  }
}
```

When both daily and idle resets are configured, whichever expires first wins. Idle freshness is based on the last real user/channel interaction — heartbeat, cron, and exec system events do not extend idle lifetime.

Sessions with an active provider-owned CLI session are not cut by the implicit daily reset. Use `/reset` or configure `session.reset` explicitly when those sessions should expire on a timer.

### Tune session maintenance to prevent storage growth

OpenClaw bounds session storage automatically. The default is `enforce` mode, which applies cleanup during maintenance:

```json5
{
  "session": {
    "maintenance": {
      "mode": "enforce",
      "pruneAfter": "30d",
      "maxEntries": 500
    }
  }
}
```

Use `mode: "warn"` first to see what would be cleaned without mutating anything. Apply the cap immediately with:

```bash
openclaw sessions cleanup --enforce
```

Preview what would be removed:

```bash
openclaw sessions cleanup --dry-run
```

### Choose a queue mode that matches your workflow

The queue controls what happens when a second message arrives while a session already has an active agent run — like messages arriving faster than replies. Think of it as a message-handling policy at the front desk: does the new message get routed to interrupt, wait its turn, or get batched together?

| Mode | Behavior |
|---|---|
| `steer` | Inject the message into the active run (default) |
| `followup` | Enqueue for a separate agent turn after the current run ends |
| `collect` | Coalesce queued messages into a single followup turn |
| `interrupt` | Abort the active run and start fresh with the newest message |

The global default is `steer` with `debounceMs: 500`, `cap: 20`, `drop: "summarize"`. Configure it globally or per channel:

```json5
{
  "messages": {
    "queue": {
      "mode": "steer",
      "debounceMs": 500,
      "cap": 20,
      "drop": "summarize",
      "byChannel": {
        "discord": "collect"
      }
    }
  }
}
```

For a user at a keyboard, `steer` is usually right — it lets you refine your request mid-run. For webhook-driven automation, `collect` or `followup` prevents a burst of events from each spawning a separate agent run.

Queue mode can also be set per-session from chat:

```bash
/queue collect
/queue collect debounce:0.5s cap:25 drop:summarize
/queue reset  # clear session override, inherit config default
```

### Tune concurrency to match your hardware

The default concurrency caps are:

- `main` lane: 4 parallel runs.
- `subagent` lane: 8 parallel runs.
- Cron and nested: tracked separately so background jobs don't block inbound replies.

Raise or lower the main cap at `agents.defaults.maxConcurrent`:

```json5
{
  "agents": {
    "defaults": {
      "model": {
        "maxConcurrent": 3
      }
    }
  }
}
```

Per-session lanes guarantee only one agent run touches a given session at a time, regardless of the global cap.

### Watch for stuck sessions

When verbose logging is enabled, queued runs emit a notice if they waited more than roughly 2 seconds before starting. For persistent stuck sessions, enable diagnostics:

```json5
{
  "diagnostics": {
    "stuckSessionWarnMs": 30000
  }
}
```

Sessions past that threshold are classified:
- `session.long_running` — active work is happening, but slowly.
- `session.stalled` — active work but no recent progress.
- `session.stuck` — recoverable stale session bookkeeping.

If commands seem stuck: enable verbose logs and look for `"queued for ...ms"` lines to confirm the queue is draining.

### Configure compaction for long-running sessions

Compaction summarizes the conversation history when the context window approaches its limit — like condensing a long email thread into a short summary at the top, so the conversation can keep going.

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "identifierPolicy": "strict",
        "notifyUser": true,
        "truncateAfterCompaction": true
      }
    }
  }
}
```

- `mode: "safeguard"` uses chunked summarization for long histories, which is more reliable than the default single-pass mode.
- `identifierPolicy: "strict"` prepends guidance to preserve opaque identifiers (deployment IDs, ticket IDs, host:port pairs) during summarization.
- `notifyUser: true` sends brief notices when compaction starts and completes so you're not surprised by the history change.
- `truncateAfterCompaction: true` rotates to a smaller successor JSONL file after successful compaction.

---

## Area 5: Observability

A common trap here: running OpenClaw with `--verbose` feels like you're capturing everything, but that ad-hoc console output disappears when you close the terminal and is not the same as a durable, structured file log. We'll look at how to wire up both surfaces so you have what you need when something goes wrong.

*The full observability surface — structured events, the Debug UI, and log correlation — is introduced in [Observability](../operations/21-observability.md).*

### Understand the two log surfaces

OpenClaw has two distinct log outputs:

| Surface | What it is | How to control |
|---|---|---|
| Console output | What you see in the terminal or Debug UI | `--verbose`, `logging.consoleLevel`, `logging.consoleStyle` |
| File logs | JSON-lines files on disk | `logging.level`, `logging.file` |

`--verbose` only affects **console verbosity**. To capture verbose-level detail in file logs, set `logging.level: "debug"` or `"trace"`:

```json5
{
  "logging": {
    "level": "debug",
    "consoleLevel": "info",
    "consoleStyle": "pretty"
  }
}
```

The default rolling log file is at `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (one file per day, rotating at 100 MB). Change the path with `logging.file`.

### Tail logs in real time

```bash
# Tail via CLI
openclaw logs --follow

# Tail via Gateway debug mode
openclaw gateway --verbose
openclaw gateway --verbose --ws-log compact
```

The Control UI Logs tab also tails the log file via the gateway (`logs.tail` RPC).

### Use `openclaw doctor --lint` in CI

For automated gates (pre-deploy, post-update checks), `--lint` mode is read-only, structured, and exits non-zero when findings exist:

```bash
# In a CI script
openclaw doctor --lint --json --severity-min error
# exit 0: clean; exit 1: error-level findings; exit 2: runtime failure
```

Combine with `--only <checkId>` for narrow gates around specific config surfaces.

### Run `openclaw doctor` after every update

Beyond the CI gate, run the interactive doctor after every manual update. It catches:

- Gateway service supervisor mismatches (stale port pins, proxy environment drift).
- Stale session lock files from abnormal exits.
- Model auth health (expiring OAuth tokens, billing-disabled profiles).
- Legacy plugin manifest contract keys that need migration.
- Bootstrap file size warnings (files near or over the context budget).

```bash
openclaw doctor
```

For Docker-based installs: review [Deployment and Lifecycle](./23-deployment.md) for container-specific doctor caveats.

### Verify health with `openclaw status`

```bash
# Current session, model, and gateway health
openclaw status

# Full picture with secrets redacted (safe to share)
openclaw status --all
```

For channel-specific health:

```bash
openclaw channels status --probe
```

### Inspect sessions to diagnose context issues

```bash
# List active sessions
openclaw sessions --json --active 60

# Inspect a specific session from chat
/status        # context usage, model, and toggles
/context list  # what is in the system prompt
```

If the model seems to have forgotten recent context, `/status` will show context usage as a fraction of the model's context window — high context usage is usually what triggers compaction or context pruning.

### Log rotation and retention

Active log files rotate at `logging.maxFileBytes` (default: 100 MB), keeping up to five numbered archives. For long-running production setups:

```json5
{
  "logging": {
    "file": "/var/log/openclaw/openclaw.log",
    "level": "info",
    "maxFileBytes": 52428800
  }
}
```

Prune old session transcripts when you don't need long retention:

```bash
openclaw sessions cleanup --dry-run    # preview what would be pruned
openclaw sessions cleanup --enforce    # apply the configured maxEntries cap
```

---

## Quick-reference tables

### Security priority order (audit findings)

| Priority | Finding type | Action |
|---|---|---|
| 1 | "open" + tools enabled | Lock down DMs/groups first, then tighten tool policy |
| 2 | Public network exposure | Fix immediately (LAN bind, Funnel, missing auth) |
| 3 | Browser control remote exposure | Treat as operator access: tailnet-only |
| 4 | Permissions | State/config/credentials not group/world-readable |
| 5 | Plugins | Only load what you explicitly trust |
| 6 | Model choice | Use modern, instruction-hardened models for tool-enabled agents |

### Common env vars

| Variable | Effect |
|---|---|
| `OPENCLAW_GATEWAY_PORT` | Override the gateway port (default `18789`) |
| `OPENCLAW_GATEWAY_PASSWORD` | Set gateway password auth |
| `OPENCLAW_STATE_DIR` | Override the state directory (default `~/.openclaw`) |
| `OPENCLAW_WORKSPACE_DIR` | Override the default workspace root |
| `OPENCLAW_DISABLE_BONJOUR` | Set to `1` to disable mDNS discovery |

### Session dmScope decision guide

| Situation | Recommended `dmScope` |
|---|---|
| Single user, one channel | `main` (default) |
| Multiple users, single account | `per-channel-peer` |
| Multiple users, multi-account channel | `per-account-channel-peer` |
| Same person on multiple channels | `per-peer` + `session.identityLinks` (config that links multiple channel identities to one logical user, so the same person on Telegram and WhatsApp shares a session) |

---

← Previous: [Design Decisions and Tradeoffs: SQLite, Exclusive Memory Slot, Loopback, In-Process Plugins](./25-design-decisions.md) · Next: [Glossary: OpenClaw Vocabulary Reference](./27-glossary.md) →
