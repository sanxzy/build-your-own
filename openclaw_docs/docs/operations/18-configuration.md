---
title: "Configuration System: openclaw.json, Zod Validation, and Hot Reload"
description: "How openclaw.json works, what happens when validation fails, all path and Gateway env vars, the four hot-reload modes, and openclaw doctor."
category: operations
type: explanation
tags: [configuration, openclaw.json, JSON5, Zod validation, hot reload, hybrid reload, OPENCLAW_CONFIG_PATH, OPENCLAW_STATE_DIR, OPENCLAW_GATEWAY_TOKEN, OPENCLAW_GATEWAY_PASSWORD, OPENCLAW_GATEWAY_PORT, OPENCLAW_GATEWAY_URL, OPENCLAW_GATEWAY_SECRET, OPENCLAW_HOME, OPENCLAW_INCLUDE_ROOTS, OPENCLAW_LOG_LEVEL, OPENCLAW_DIAGNOSTICS, OPENCLAW_DEBUG_MODEL_TRANSPORT, OPENCLAW_PROXY_URL, OPENCLAW_NO_RESPAWN, env vars, openclaw doctor, config schema, environment variables, validation, reload modes, config file, gateway config]
keywords: [config file location, openclaw config, gateway.reload, debounceMs, last-known-good, config validate, hot apply, restart required, path overrides, state dir, config path]
sources: [S35, S36, S37, S38, S39, S47, S120, S9]
---

**TL;DR** — OpenClaw reads a single JSON5 file at `~/.openclaw/openclaw.json` and validates every field against a Zod schema at startup. If the file is missing, OpenClaw runs on safe defaults. If a field is invalid, the Gateway refuses to start — and `openclaw doctor --fix` is the command that explains and repairs the problem. This chapter walks through the file's location, the validation contract, every path-override and Gateway environment variable, and the four hot-reload modes that control how live edits take effect without a manual restart.

# Configuration System: openclaw.json, Zod Validation, and Hot Reload

Before we look at what goes in the config file, let's establish why it exists and what problem it solves. When you run the OpenClaw Gateway, dozens of subsystems need to know: which models to use, which channels to listen on, where to store state, and who is allowed to connect. Rather than accepting these settings from dozens of separate files or command-line flags, OpenClaw reads a single optional config file and uses it as the authoritative source. Everything that is not in that file gets a safe default.

Two prerequisites shape how the Gateway reads this file:

- **The Gateway** — the long-running process that serves all agents and channels — reads the config at startup and can update itself when the file changes. See [Gateway](../gateway/03-gateway.md) for details on the Gateway process itself.
- **Agents** — runtime identities each with their own workspace — draw most of their defaults from the `agents.defaults.*` section. See [Agents](../agents/05-agents.md) for what these defaults configure.

## The Config File: Location and Format

The config file lives at `~/.openclaw/openclaw.json`. The extension says `.json` but OpenClaw actually accepts **JSON5**, a superset of JSON that allows comments and trailing commas. This means you can annotate your config without breaking it:

```json5
// ~/.openclaw/openclaw.json
{
  // The workspace the default agent uses
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },

  // Only accept WhatsApp DMs from this number
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

If the file is absent entirely, OpenClaw uses safe defaults and starts normally. You only need the file when you want to change something from those defaults.

> **Symlink note:** The config path must point at a regular file. If you keep the file outside the default state directory, point `OPENCLAW_CONFIG_PATH` directly at it. Symlinking `openclaw.json` is not supported for OpenClaw-owned writes because an atomic write may replace the path rather than follow the symlink.

## Zod Validation: What Happens When the File Is Wrong

OpenClaw validates the config file against a **Zod schema** — Zod is a TypeScript-first schema library that checks every field's type and value against a precise specification. The source confirms this: `src/config/zod-schema.ts` imports from `zod` and assembles the canonical schema. Zod is also used in the per-subsystem schema files (`zod-schema.agent-defaults.ts`, `zod-schema.session.ts`, `zod-schema.hooks.ts`, and others in `src/config/`).

Think of this validation like a strict border check at an airport: every item in your bag is inspected against a known list. If you carry something unexpected — an unrecognised key, a string where a number is required, a value outside the allowed set — the bag does not get through.

### What validation enforces

- **No unknown keys.** Every key you write must match the schema. The only root-level exception is `$schema` (a string that lets editors attach JSON Schema metadata).
- **Correct types.** A port number must be a number, not a string. A boolean field cannot be `"yes"`.
- **Allowed values.** Enum fields like `gateway.reload.mode` only accept the documented string values.

### What happens when validation fails at startup

1. The Gateway refuses to start.
2. Only diagnostic commands continue to work: `openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`.
3. You run `openclaw doctor` to see the exact validation issues.
4. You run `openclaw doctor --fix` (or `--yes`) to apply repairs.

This means that a typo in your config file does not cause mysterious runtime failures or half-started subsystems — it causes an immediate, legible refusal with a path to repair.

### The last-known-good copy

After every successful startup, OpenClaw keeps a trusted copy of the validated config. If a subsequent edit fails validation, the running Gateway **does not** restore this copy automatically — the reload is skipped and the runtime continues with the last accepted config. The last-known-good copy is there for `openclaw doctor --fix` to restore when you ask it to.

There is one guard: if the candidate config contains redacted secret placeholders such as `***`, it is not promoted to last-known-good even if it validates. This prevents accidentally treating a partially-redacted export as the live config.

### Editing config safely

You have four ways to edit the config:

| Method | When to use |
|---|---|
| `openclaw onboard` / `openclaw configure` | Interactive setup wizard; generates a valid config from prompts |
| `openclaw config set <path> <value>` | One-liner for a single field (`openclaw config set agents.defaults.heartbeat.every "2h"`) |
| Control UI Config tab at `http://127.0.0.1:18789` | Form-driven editor built from the live schema, including a raw JSON escape hatch |
| Direct file edit | Full control; the Gateway watches and reloads automatically |

The CLI commands (`config set`, `config get`, `config unset`) apply the same schema gate before writing, so they will not save an invalid value.

To inspect the canonical schema at any time:

```bash
openclaw config schema   # prints the full JSON Schema
```

## The $include Directive: Splitting Config into Multiple Files

For large configs, you can factor sections into separate files using `$include`:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/a.json5", "./clients/b.json5"],
  },
}
```

Key rules:
- A single-file include replaces the containing object.
- An array of files is deep-merged in order (later wins).
- Sibling keys override included values.
- Includes are confined to the directory containing `openclaw.json` by default. To include files from other directories, set `OPENCLAW_INCLUDE_ROOTS` (covered below).
- Includes are capped at 10 levels deep.
- When a top-level section is backed by a single-file include (e.g. `plugins: { $include: "./plugins.json5" }`), OpenClaw writes changes to that included file directly, leaving `openclaw.json` intact.

## Environment Variables

OpenClaw's environment variable system has a defined precedence order (highest to lowest):

1. Process environment (already set before OpenClaw starts)
2. `.env` in the current working directory
3. `~/.openclaw/.env`
4. The `env` block in `openclaw.json`

Neither the `.env` files nor the `env` config block override non-empty values already in the process environment.

You can also use `${VAR_NAME}` substitution inside any config string value:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
}
```

Only uppercase names matching `[A-Z_][A-Z0-9_]*` are substituted. A missing or empty variable throws an error at config load time.

### Path override variables

These variables redirect where OpenClaw stores state and reads config. They are useful when you run multiple Gateway instances, work in containers, or keep state on a separate volume.

| Variable | Default | Purpose |
|---|---|---|
| `OPENCLAW_STATE_DIR` | `~/.openclaw` | Root directory for all mutable state (sessions, logs, SQLite databases, credentials) |
| `OPENCLAW_CONFIG_PATH` | `~/.openclaw/openclaw.json` | Override the config file path; must point at a regular file, not a symlink |
| `OPENCLAW_HOME` | `~` (system home dir) | Override the home directory used for tilde expansion in all paths |
| `OPENCLAW_INCLUDE_ROOTS` | (none) | Colon-separated (POSIX) or semicolon-separated (Windows) extra directories from which `$include` may resolve files; each entry is tilde-expanded |

Example — running a second Gateway instance with its own state and config:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/work.json \
OPENCLAW_STATE_DIR=~/.openclaw-work \
openclaw gateway --port 19001
```

The convenience flags `--dev` and `--profile <name>` are shorthand for this pattern (`--dev` uses `~/.openclaw-dev` + port `19001`; `--profile work` uses `~/.openclaw-work`).

### Gateway authentication and connection variables

These variables configure the Gateway's own authentication and how CLI commands locate a running Gateway. They are typically set in `~/.openclaw/.env` or in the service environment rather than in `openclaw.json` directly.

| Variable | Purpose |
|---|---|
| `OPENCLAW_GATEWAY_TOKEN` | Shared-secret bearer token for Gateway authentication. Set a strong random value (`openssl rand -hex 32`). Never use the documented placeholder value verbatim |
| `OPENCLAW_GATEWAY_PASSWORD` | Alternative auth mode (token or password, not both) |
| `OPENCLAW_GATEWAY_PORT` | Override the Gateway's TCP port. Precedence: `--port` CLI flag > this variable > `gateway.port` config > default `18789` |
| `OPENCLAW_GATEWAY_URL` | URL the CLI uses to locate a running Gateway (used by client-side commands) |
| `OPENCLAW_GATEWAY_SECRET` | Internal secret used by credential resolution flows; treated as a sensitive var and blocked from workspace dotenv injection |

> **Security note:** `OPENCLAW_GATEWAY_TOKEN` and `OPENCLAW_GATEWAY_PASSWORD` are mutually exclusive. If both are set, `gateway.auth.mode` must be set explicitly to `token` or `password` or Gateway startup fails. Non-loopback binds require Gateway auth; the onboarding wizard generates a token by default.

### Observability and process variables

These variables control diagnostics, logging verbosity, and process lifecycle.

| Variable | Purpose |
|---|---|
| `OPENCLAW_LOG_LEVEL` | Override the log level. Allowed values correspond to the Gateway's level names (e.g. `debug`, `info`, `warn`, `error`). Invalid values are ignored with a stderr warning |
| `OPENCLAW_DIAGNOSTICS` | Enable subsystem diagnostic flags. Accepts a comma-separated list of flag names (e.g. `telegram.http`), `*` or `1` for all flags, or `0`/`false`/`off` to disable all |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | Enable model transport debug logging (truthy value, e.g. `1`) |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD` | Set model payload debug verbosity: `summary` (sizes and counts), `tools` (tool schemas), `full-redacted` (full request with secrets masked) |
| `OPENCLAW_DEBUG_SSE` | Set SSE stream debug verbosity: `events` (all events) or `peek` (first few bytes) |
| `OPENCLAW_PROXY_URL` | Forward-proxy URL for all outbound HTTP(S) from the Gateway (config field `proxyUrl` takes precedence when set) |
| `OPENCLAW_NO_RESPAWN` | Set to `1` to disable the Gateway's automatic process respawn after updates or crashes. Useful on small VMs or managed container environments where the supervisor handles restarts. When set, `OPENCLAW_NO_RESPAWN=1` takes precedence over any inherited supervisor respawn hints |

Subsystem diagnostics (`OPENCLAW_DIAGNOSTICS`) are more granular than `OPENCLAW_LOG_LEVEL`. Where `OPENCLAW_LOG_LEVEL=debug` floods every component, `OPENCLAW_DIAGNOSTICS=telegram.http` enables detailed logging only for the Telegram HTTP layer, leaving everything else at its normal level. The same flags can also be set in `openclaw.json` under `diagnostics.flags`. See [Observability](./21-observability.md) for the full treatment of logs and diagnostics.

## Config Hot Reload

Once the Gateway is running, it watches `~/.openclaw/openclaw.json` for changes. When you save the file, the Gateway re-reads it, validates it, and decides what to do next — without you issuing any command.

An analogy: imagine a restaurant kitchen that can swap ingredients mid-service. Some swaps (a different garnish, a sauce tweak) go straight to the next plate. Others (replacing a gas stove with an induction one) require briefly halting service. Hot reload works the same way.

### The four reload modes

Set the mode in `openclaw.json` under `gateway.reload`:

```json5
{
  gateway: {
    reload: {
      mode: "hybrid",   // off | restart | hot | hybrid (default)
      debounceMs: 300,  // wait this many ms for edit churn to settle
    },
  },
}
```

| Mode | Behavior |
|---|---|
| `hybrid` **(default)** | Hot-applies safe changes in-process. Automatically restarts the Gateway when a change requires it. Best for most deployments |
| `hot` | Hot-applies safe changes only. Logs a warning when a restart-required change is detected — you handle the restart manually |
| `restart` | Restarts the Gateway process on any config change, safe or not |
| `off` | Disables file watching entirely. Changes take effect only on the next manual restart |

### What hot-applies vs. what needs a restart

This is the core practical question: "if I change this field, will my agent sessions be interrupted?"

| Config area | Fields | Restart needed? |
|---|---|---|
| Channels | `channels.*`, `web` | No |
| Agent and model settings | `agent`, `agents`, `models`, `routing` | No |
| Automation | `hooks`, `cron`, `agent.heartbeat` | No |
| Sessions and messages | `session`, `messages` | No |
| Tools, browser, skills, MCP | `tools`, `browser`, `skills`, `mcp`, `audio`, `talk` | No |
| UI, logging, identity, bindings | `ui`, `logging`, `identity`, `bindings` | No |
| **Gateway server settings** | `gateway.*` (port, bind, auth, TLS, HTTP) | **Yes** |
| **Infrastructure** | `discovery`, `plugins` | **Yes** |

> **Exceptions:** `gateway.reload` and `gateway.remote` do not trigger a restart when changed — they are read from the new config immediately.

**Concrete example (hot-applies — no restart):** You add a new skill to `agents.defaults.skills`. The Gateway reads the new config, updates the skill roster in memory, and the change takes effect on the next agent turn. Active sessions are unaffected.

**Concrete example (needs restart):** You change `gateway.port` from `18789` to `19001`. The Gateway's TCP listener is already bound to port `18789`. A new port requires the listener to be torn down and re-established, which means a process restart. In `hybrid` mode this restart happens automatically; in `hot` mode you would see a warning and need to run `openclaw gateway restart` yourself.

### The debounce window

When you save a file, most editors do a temp-write/rename dance. The `debounceMs` value (default `300`) makes the Gateway wait for edit churn to settle before reading the final version. Without this, a save operation that writes a temp file and then moves it into place would trigger two reloads (one for the partial write, one for the final file). If you run the Gateway on a slow or heavily loaded host and see spurious reload warnings, increasing `debounceMs` can help.

### When the incoming edit is invalid

The watcher treats direct file edits as untrusted input. If the new file fails validation, the reload is skipped. The current runtime keeps the last accepted config. You will see a message like `config reload skipped (invalid config)`. Run `openclaw config validate` to inspect the issue, then `openclaw doctor --fix` to repair it.

## openclaw doctor: The Canonical Health Check

`openclaw doctor` is the first command to run when something is wrong with your configuration or Gateway state. Think of it as a mechanic who checks your car and tells you exactly which parts need attention — then offers to fix them.

Run it without flags for an interactive human-readable health report:

```bash
openclaw doctor
```

### The key flags

| Command | Behavior |
|---|---|
| `openclaw doctor` | Interactive health report; prompts before making any changes |
| `openclaw doctor --fix` | Apply recommended repairs without prompting (repairs + restarts where safe) |
| `openclaw doctor --yes` | Accept all defaults without prompting, including restarts |
| `openclaw doctor --lint` | Read-only structured output for CI; never writes, never prompts; exits non-zero when findings meet the threshold |
| `openclaw doctor --fix --force` | Also apply aggressive repairs such as overwriting customised supervisor configs |
| `openclaw doctor --non-interactive` | Apply safe config-only migrations (no restarts, no service changes) |

### What doctor checks and repairs

Doctor covers a wide range of checks. The ones most relevant to configuration are:

**Config validation and migrations.** When your config uses deprecated keys (e.g. the old `routing.allowFrom` instead of `channels.whatsapp.allowFrom`), the Gateway refuses to start and asks you to run `openclaw doctor`. Doctor explains which keys are legacy, shows you the migration it intends to apply, and rewrites `openclaw.json` with the updated schema. Current migrations include routing, channel, model, TTS, browser, and cron schema changes. Gateway startup refuses legacy config and does not rewrite `openclaw.json` itself — that is `openclaw doctor`'s job.

**Auth-profile format repair.** If `auth-profiles.json` is in the legacy flat format (`{ "provider": { "apiKey": "..." } }`), `openclaw doctor --fix` rewrites it to canonical `provider:default` API-key profiles with a backup.

**Last-known-good restoration.** `openclaw doctor --fix` can restore the last-known-good config copy when the current file is broken.

**State integrity.** Doctor checks that the state directory exists and is writable, that session directories are present, and that config file permissions are tight (`600` on local installs). It warns when state is on an iCloud or SD-card path, where I/O can cause lock races.

**Gateway health.** Doctor checks whether the Gateway service is installed and running, and offers to restart it when it looks unhealthy. It also checks for port collisions on port `18789`.

**OAuth and model auth.** Doctor inspects OAuth profiles, warns when tokens are expiring, and can refresh them when safe.

**Security posture.** Doctor emits warnings when a channel has an open DM policy, when hook tokens reuse Gateway auth secrets, or when a dangerous config pattern is detected.

**Skills readiness.** Doctor reports which skills are eligible, blocked, or missing their required binaries/env vars, and `--fix` can disable unavailable skills in `skills.entries`.

**Plugin install hygiene.** Doctor removes stale plugin dependency staging state and can reinstall missing downloadable plugins referenced in config.

### Lint mode for CI

`openclaw doctor --lint` is designed for automated environments. It is read-only — it never writes, never prompts, and never restarts anything. It emits structured findings and exits with code `0` (no findings at or above the threshold) or `1` (one or more findings met the threshold):

```bash
openclaw doctor --lint                            # structured human output
openclaw doctor --lint --json                     # machine-readable JSON
openclaw doctor --lint --severity-min warning     # only show warnings and above
openclaw doctor --lint --only core/doctor/gateway-config  # narrow check
```

The JSON output includes `ok`, `checksRun`, `checksSkipped`, and a `findings` array with `checkId`, `severity`, `message`, and optional `path`, `line`, `column`, and `fixHint` fields.

### Failure path: config validation failure at startup

When you see the Gateway fail to start with an invalid config message, here is the sequence:

```
1. openclaw doctor          → see the exact validation issues listed
2. openclaw config validate → confirm the specific errors
3. openclaw doctor --fix    → let doctor repair what it can (migrations,
                               format fixes, last-known-good restore)
4. openclaw gateway start   → restart after config is repaired
```

If doctor cannot automatically repair the issue (e.g. you wrote an invalid value that has no migration), the output will show the exact field path and what is wrong, so you can edit the file directly.

## Config in Practice: A Complete Minimal Example

Let's trace through what a fresh minimal config looks like and what happens when we run the Gateway with it.

We start with nothing: the file does not exist. The Gateway starts on safe defaults. We decide we want Telegram and a specific model:

```json5
// ~/.openclaw/openclaw.json — step 1: add Telegram
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "${TELEGRAM_BOT_TOKEN}",
      dmPolicy: "pairing",  // unknown senders get a one-time pairing code
    },
  },
}
```

The Gateway's watcher detects the new file, validates it (all fields are valid, `TELEGRAM_BOT_TOKEN` is in the process env), and hot-applies the Telegram channel — no restart needed.

Now we add a model:

```json5
// ~/.openclaw/openclaw.json — step 2: add model
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "${TELEGRAM_BOT_TOKEN}",
      dmPolicy: "pairing",
    },
  },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
    },
  },
}
```

`agents.*` hot-applies. The change takes effect on the next agent run without a restart.

Now we decide to change the Gateway port from `18789` to `19001`:

```json5
{
  // ... same channels and agents config ...
  gateway: { port: 19001 },
}
```

In `hybrid` mode (the default), the watcher detects that `gateway.port` is a restart-required field and automatically restarts the Gateway process on the new port. In `hot` mode, you would see a warning and need to restart manually.

---

← Previous: [Automation and Scheduling: Cron, Heartbeat, and Dreaming](../automation/17-automation.md) · Next: [Storage and Persistence: SQLite, JSONL Sessions, and the Workspace](./19-storage.md) →
