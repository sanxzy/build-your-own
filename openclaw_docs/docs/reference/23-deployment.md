---
title: "Deployment and Lifecycle: Install, Daemon Setup, Docker, and Hosted Options"
description: "How to install OpenClaw, configure a supervisor daemon, containerize with Docker, and deploy to Fly.io, Render, or any VPS — with OPENCLAW_NO_RESPAWN explained."
category: reference
type: how-to
tags: [deployment, install, npm install, openclaw onboard, daemon, launchd, systemd, Windows Scheduled Task, Docker, docker-compose, volumes, Fly.io, Render, VPS, OPENCLAW_NO_RESPAWN, openclaw gateway status, openclaw dashboard, hosted options, multi-stage build, non-root, build args, OPENCLAW_EXTENSIONS, OPENCLAW_INSTALL_BROWSER, OPENCLAW_INSTALL_DOCKER_CLI]
keywords: [openclaw setup, install openclaw, daemon setup, containerize openclaw, cloud deployment, LaunchAgent, ai.openclaw.gateway, openclaw-gateway service, openclaw-cli service, docker compose, persistent volume, state directory, onboarding wizard]
sources: [S74, S75, S76, S77, S78, S79, S11, S12, S13, S14, S127, S128, S80, S125, S133]
---

**TL;DR** — This chapter walks through the full deployment sequence: install the
OpenClaw CLI, run onboarding to configure the Gateway and set up an
auto-start supervisor, verify the Gateway is alive, and open the Control UI
dashboard. We then cover containerized deployments (Docker multi-stage image,
Docker Compose) and three hosted options (Fly.io, Render, VPS with systemd).

# Deployment and Lifecycle: Install, Daemon Setup, Docker, and Hosted Options

The **Gateway** is the single long-running process at the center of OpenClaw —
it owns all agent sessions, channel connections, and state. Think of it like a
router in your home: nothing else works if it is not running. Every other step
in this chapter is either about getting the Gateway started or making sure it
stays running when you close your terminal or reboot the machine.

Before diving in, recall two concepts you've already met in earlier chapters:

- The **Gateway** (see [The Gateway](../gateway/03-gateway.md)) is the control
  plane that listens on port 18789 and routes messages between channels and
  agents.
- **Configuration and environment variables** (see [Configuration](../operations/18-configuration.md))
  like `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, and `OPENCLAW_GATEWAY_TOKEN`
  control where state lives and how the Gateway authenticates clients.

## Step 1: Choose an install path

There are four ways to get the `openclaw` CLI onto your machine. We start with
the two you are most likely to use:

| Method | Best for |
|---|---|
| Installer script (curl or PowerShell) | First install on macOS, Linux, or Windows — handles Node automatically |
| `npm install -g openclaw@latest` | If you already manage Node yourself |
| `pnpm add -g openclaw@latest` | pnpm users (also run `pnpm approve-builds -g` after install) |
| `bun add -g openclaw@latest` | Bun users (CLI only; Node remains the recommended Gateway runtime) |

### Installer script (recommended first install)

The installer detects your OS, installs Node if it is missing, and launches
onboarding when it finishes:

```bash
# macOS / Linux / WSL2
curl -fsSL https://openclaw.ai/install.sh | bash
```

```powershell
# Windows (PowerShell)
iwr -useb https://openclaw.ai/install.ps1 | iex
```

To install without immediately running onboarding, add `--no-onboard`:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### npm global install

If you prefer to manage Node yourself and already have Node 22.19+ or 24:

```bash
npm install -g openclaw@latest
```

After install, verify the CLI is reachable:

```bash
openclaw --version
```

If you get "command not found," the global npm bin directory is not in your
`PATH`. Find it and add it:

```bash
npm prefix -g        # prints something like /usr/local or /home/user/.npm-global
# Add $(npm prefix -g)/bin to your $PATH in ~/.zshrc or ~/.bashrc, then restart
```

## Step 2: Run onboarding and install the daemon

Here is the problem we need to solve: even if the CLI is installed, the Gateway
process is not running yet. And even if you start it manually, it will stop the
moment your terminal closes or your machine reboots.

`openclaw onboard --install-daemon` solves both in one step. It is a guided
wizard that:

1. **Model and auth** — prompts you to choose a provider (Anthropic, OpenAI,
   Google, etc.) and enter your API key, which it stores in an auth profile.
2. **Workspace** — sets the agent workspace location (default `~/.openclaw/workspace`)
   and seeds bootstrap files (`AGENTS.md`, `IDENTITY.md`, `USER.md`).
3. **Gateway** — configures the port (default 18789), bind address, and auth
   token.
4. **Channels** — optionally connects a messaging channel (Telegram, Discord,
   WhatsApp, etc.).
5. **Daemon** — registers the Gateway as a system supervisor service so it
   starts automatically at login (or boot). This is the `--install-daemon` part.
6. **Health check** — starts the Gateway and confirms it is reachable.
7. **Skills** — installs recommended skills.

```bash
openclaw onboard --install-daemon
```

The wizard takes roughly two minutes. It asks one question at a time. At the
end you will have a running Gateway and a supervisor entry that restarts it if
it crashes.

Think of the daemon entry like a smoke detector: it runs silently in the
background and restarts itself if something goes wrong, without you having to
intervene.

### What supervisor does `--install-daemon` register?

The supervisor type depends on your operating system:

| OS | Supervisor | Label / unit name |
|---|---|---|
| macOS | launchd LaunchAgent | `ai.openclaw.gateway` |
| Linux / WSL2 | systemd user service | `openclaw-gateway.service` |
| Windows | Scheduled Task (per-user Startup-folder fallback if task creation is denied) | depends on profile |

The wizard renders the current canonical unit for you and loads it. You do not
need to write plist or service files by hand unless you want a customised setup.

> **Re-running onboarding** does not wipe existing config unless you pass
> `--reset`. It is safe to re-run to update credentials, change a channel, or
> reinstall the daemon after a manual unload.

## Step 3: Verify the Gateway is running

The Gateway is now under supervisor control. Let's confirm it came up
correctly:

```bash
openclaw gateway status
```

You should see the Gateway listening on port 18789. If you do not, check:

```bash
openclaw doctor
```

`openclaw doctor` (introduced in the [Configuration chapter](../operations/18-configuration.md))
audits config validity, auth profiles, and Gateway reachability, and prints
actionable guidance. It is always the first diagnostic step.

## Step 4: Open the Control UI dashboard

The Control UI — also called the **OpenClaw dashboard** — is a small single-page
web app served by the Gateway itself at `http://127.0.0.1:18789/`. Think of it
as the admin panel for your agent: you can chat, review sessions, manage cron
jobs, edit config, and tail logs, all from the browser.

```bash
openclaw dashboard
```

This opens the Control UI in your default browser. If it loads, the full
installation is working.

For the first connection from a new browser or device, the Gateway requires a
one-time pairing approval:

```bash
openclaw devices list          # see the pending request ID
openclaw devices approve <id>  # approve it
```

Local loopback connections (`127.0.0.1`) are auto-approved, so on the same
machine you typically will not see this prompt.

We now have the minimal working setup: CLI installed → Gateway running under a
supervisor → dashboard accessible. The rest of this chapter covers more advanced
deployment shapes.

---

## Supervisor reference: managing the Gateway process

Whether you used `openclaw onboard --install-daemon` or want to manage the
daemon manually, here are the platform-specific commands.

### macOS (launchd)

OpenClaw installs a per-user LaunchAgent. The label is `ai.openclaw.gateway`
(or `ai.openclaw.<profile>` when running a named profile).

```bash
# Check status
openclaw gateway status

# Stop the Gateway service
launchctl bootout gui/$UID/ai.openclaw.gateway

# Start / restart
launchctl kickstart -k gui/$UID/ai.openclaw.gateway

# Install (if not already registered)
openclaw gateway install
```

If the Gateway keeps silently stopping on macOS after a period of inactivity,
this is typically a macOS Maintenance Sleep issue interacting with launchd's
respawn-protection gate. The platform documentation at [docs.openclaw.ai](https://docs.openclaw.ai/gateway/troubleshooting)
covers the specific workaround.

### Linux / WSL2 (systemd user service)

OpenClaw installs a systemd **user** service by default. If you need a system
service for a shared or always-on server without a logged-in user session, use
`sudo systemctl edit openclaw-gateway.service` instead of the user path.

```bash
# Check status
systemctl --user status openclaw-gateway.service

# Start / stop / restart
systemctl --user start openclaw-gateway.service
systemctl --user stop openclaw-gateway.service
systemctl --user restart openclaw-gateway.service

# Enable auto-start on login
systemctl --user enable openclaw-gateway.service
```

A minimal hand-written user unit (for custom setups where `openclaw gateway install`
did not run) looks like this:

```ini
# ~/.config/systemd/user/openclaw-gateway.service
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group

[Install]
WantedBy=default.target
```

Enable it:

```bash
systemctl --user enable --now openclaw-gateway.service
```

### Windows

On Windows, `openclaw onboard --install-daemon` registers a Scheduled Task. If
Task Scheduler creation is denied, it falls back to a per-user Startup-folder
entry. Check Gateway status the same way as on other platforms:

```bash
openclaw gateway status
```

### `OPENCLAW_NO_RESPAWN=1` — why small VMs need it

By default, the Gateway uses a process respawn strategy during routine restarts:
it spawns a new process, hands off state, and exits the old one. On a fast
desktop machine, this is fine — the handoff is fast and PID tracking is simple.

On a small VM (1–2 GB RAM, shared CPU), that extra process spawn during handoff
can temporarily spike memory high enough to trigger the Linux OOM killer, which
kills one of the two Gateway processes at the worst possible moment. You also
get an extra process in PID tracking that can confuse some supervisor setups.

`OPENCLAW_NO_RESPAWN=1` prevents that respawn — routine Gateway restarts stay
in-process rather than spawning a second process:

```bash
# Add to ~/.bashrc or the systemd unit's [Service] section
export OPENCLAW_NO_RESPAWN=1
```

For VPS deployments, the source documentation recommends adding it alongside a
Node module compile cache to speed up cold starts on low-power hosts:

```bash
grep -q 'OPENCLAW_NO_RESPAWN' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

When using the `--install-daemon` systemd path, inject these into the unit:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

---

## Docker deployment

Docker is optional — use it when you want the Gateway isolated in a container
or when you are deploying to a host that should not have Node installed directly.
If you are running on your own machine and want the fastest workflow, the
supervisor approach above is simpler.

### How the multi-stage build works

The OpenClaw `Dockerfile` uses a multi-stage build to keep the runtime image
small. The build argument `OPENCLAW_EXTENSIONS` lets you bake in plugin
dependencies at image-build time instead of installing them at runtime:

```
Stage 1 (workspace-deps):  Copies only package.json manifests → fast dependency stage
Stage 2 (build):           Runs pnpm install + pnpm build + pnpm ui:build
Stage 3 (runtime-assets):  Prunes dev dependencies, strips .d.ts and .map files
Final stage (base-runtime): node:24-bookworm-slim, non-root 'node' user (uid 1000), tini as PID 1
```

The final image runs as the non-root `node` user (uid 1000). This matters for
volume permissions: the directories you mount must be owned by uid 1000 on the
host, or you will see `EACCES` errors on startup.

### Build arguments

| Build arg | Purpose |
|---|---|
| `OPENCLAW_EXTENSIONS` | Space- or comma-separated plugin directory names to bake in (e.g. `"diagnostics-otel matrix"`) |
| `OPENCLAW_INSTALL_BROWSER` | Set to `1` to bake Playwright Chromium into the image (adds ~300 MB; skips the 60–90 s browser install at first run) |
| `OPENCLAW_INSTALL_DOCKER_CLI` | Set to `1` to bake the Docker CLI (~50 MB); required when using `agents.defaults.sandbox` with the Docker backend |
| `OPENCLAW_IMAGE_APT_PACKAGES` | Space-separated apt packages to install (e.g. `"git curl jq"`) |
| `OPENCLAW_IMAGE_PIP_PACKAGES` | Space-separated Python packages (e.g. `"requests humanize"`); pin package versions |

Building the image locally:

```bash
docker build -t openclaw:local \
  --build-arg OPENCLAW_EXTENSIONS="diagnostics-otel" \
  -f Dockerfile .
```

Using a pre-built image from the GitHub Container Registry:

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
```

Common image tags: `main`, `latest`.

### Volumes and state persistence

This is the most important thing to understand about the Docker setup: the
Gateway stores all of its state (config, sessions, credentials, workspace files)
in a single home directory on the container filesystem. If you do not mount that
directory to a host path, everything is lost when the container is replaced.

The Docker Compose file mounts three host paths into the container:

| Host path variable | Mounts to (in container) | What it holds |
|---|---|---|
| `OPENCLAW_CONFIG_DIR` (default: `~/.openclaw`) | `/home/node/.openclaw` | `openclaw.json`, per-agent databases, session JSONL files, installed plugin records |
| `OPENCLAW_WORKSPACE_DIR` (default: `~/.openclaw/workspace`) | `/home/node/.openclaw/workspace` | Agent workspace files (`AGENTS.md`, `MEMORY.md`, `SOUL.md`, etc.) |
| `OPENCLAW_AUTH_PROFILE_SECRET_DIR` (default: `~/.openclaw-auth-profile-secrets`) | `/home/node/.config/openclaw` | Local encryption key for OAuth-backed auth profile token material |

When any of these variables is unset, the Compose file falls back to paths under
`$HOME`, or `/tmp` when `HOME` itself is also unset, so `docker compose up` does
not fail on bare environments.

> **The state volume** (`/home/node/.openclaw`) is the critical one. Mounting
> it to a persistent host directory is what makes the Gateway's sessions, config,
> and agent state survive container replacement. Without it you lose everything
> on `docker compose down`.

If you see permission errors like `blocked plugin candidate: suspicious ownership`,
fix the host-side ownership:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

### Health check endpoints

The Gateway exposes two auth-free probe endpoints for container orchestration:

```bash
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz    # readiness
```

The Docker image includes a built-in `HEALTHCHECK` instruction that pings
`/healthz` every three minutes. If it fails repeatedly, Docker marks the
container `unhealthy`.

### Docker Compose services

The bundled `docker-compose.yml` defines two services:

**`openclaw-gateway`** — the main Gateway process. Publishes port 18789 to the
host. Starts with `--bind lan` so the host browser can reach it at
`http://127.0.0.1:18789/`. Security hardening: drops `NET_RAW` and `NET_ADMIN`
capabilities, sets `no-new-privileges`.

**`openclaw-cli`** — a sidecar service that shares the network namespace of
`openclaw-gateway` (`network_mode: "service:openclaw-gateway"`). Because it
shares the network namespace, it can reach the Gateway over `127.0.0.1` without
an extra port publication. Use it for any CLI commands you need to run after the
Gateway container is already up:

```bash
docker compose run --rm openclaw-cli gateway status
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
```

Think of `openclaw-cli` as a screwdriver that shares the same workbench as
`openclaw-gateway` — it reaches the same ports, the same files, and the same
config, but it is not always running.

**Important:** pre-start onboarding and first-time config writes must run
through `openclaw-gateway` with `--no-deps --entrypoint node`, before the
Compose stack is up:

```bash
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
```

Once the Gateway container is running, switch to `openclaw-cli` for all
subsequent CLI commands.

### Using the setup script

The fastest path to a running Docker Gateway is the bundled setup script, which
handles build, onboarding, and stack startup in sequence:

```bash
./scripts/docker/setup.sh
```

The script will:
- Build the image (or pull `$OPENCLAW_IMAGE` if set)
- Run onboarding, prompt for provider API keys, and write a gateway token to `.env`
- Start `docker compose up -d openclaw-gateway`

After setup, open the Control UI and paste the configured shared secret from
`.env` into the Settings panel:

```bash
docker compose run --rm openclaw-cli dashboard --no-open
# prints the dashboard URL with the token encoded
```

### Enabling agent sandbox (Docker backend)

If you want agent tool executions isolated in separate Docker containers, you
need to mount the Docker socket and install the Docker CLI inside the image:

```bash
# In docker-compose.yml, uncomment:
#   - /var/run/docker.sock:/var/run/docker.sock
# and the group_add: ["${DOCKER_GID:-999}"] block

# Build with Docker CLI baked in:
docker build --build-arg OPENCLAW_INSTALL_DOCKER_CLI=1 -t openclaw:local .
```

Sandbox mode config:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",  // off | non-main | all
        scope: "agent",    // session | agent | shared
      },
    },
  },
}
```

See [Security and Governance](../operations/20-security.md) for full sandbox
exposure hardening guidance.

### LAN bind and local providers

The Compose setup defaults to `--bind lan` (`gateway.bind: "lan"`) so the host
browser can reach the published port. If you run local AI providers (LM Studio,
Ollama) on the host machine, reach them via `host.docker.internal` inside the
container:

| Provider | Host URL | Docker URL |
|---|---|---|
| LM Studio | `http://127.0.0.1:1234` | `http://host.docker.internal:1234` |
| Ollama | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

The `docker-compose.yml` already maps `host.docker.internal` to the Docker
host gateway for Linux Docker Engine.

---

## Hosted deployment options

### Fly.io

OpenClaw ships a `fly.toml` that configures a Fly Machine with the necessary
settings. The key entries:

```toml
[build]
dockerfile = "Dockerfile"

[env]
NODE_ENV        = "production"
OPENCLAW_STATE_DIR = "/data"          # state lives on the persistent volume

[processes]
app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

[http_service]
internal_port   = 3000
force_https     = true
auto_stop_machines = false            # keep running for persistent connections
min_machines_running = 1

[[vm]]
size   = "shared-cpu-2x"
memory = "2048mb"

[mounts]
source      = "openclaw_data"
destination = "/data"               # Fly persistent volume mounted here
```

Notice that state is directed to `/data` (the Fly persistent volume) rather
than `/home/node/.openclaw`. When you deploy, create the volume first:

```bash
fly volumes create openclaw_data --size 10 --region iad
fly deploy
```

### Render

OpenClaw ships a `render.yaml` for Render.com deployments:

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true       # Render generates a random token
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

Render generates an `OPENCLAW_GATEWAY_TOKEN` for you automatically. After
deploy, retrieve it from the Render environment panel and use it to authenticate
the Control UI and CLI connections.

### VPS (any Linux server)

Running OpenClaw on a Linux VPS follows the same install → onboard → daemon
sequence from Steps 1–3 above, with two additional concerns:

**Harden admin access first.** Before installing anything, decide how you will
administer the box. If you want Tailnet-only admin access, install Tailscale
first and verify you can SSH over the Tailscale IP before restricting public SSH.

**Keep the Gateway on loopback and use SSH tunnels or Tailscale Serve for
dashboard access.** The safe default is `gateway.bind: "loopback"`. If you
bind to `lan` or `tailnet`, require token or password auth.

```bash
# On your laptop — forward the remote Gateway port over SSH
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
# Then open http://127.0.0.1:18789/ locally
```

Or use Tailscale Serve for a persistent HTTPS URL across reboots:

```bash
openclaw gateway --tailscale serve
# Opens https://<magicdns>/ on your tailnet
```

For VPS deployments, also apply the `OPENCLAW_NO_RESPAWN=1` tuning from the
supervisor section above, especially on hosts with 1–2 GB RAM.

**VPS provider options** the source documentation lists include Railway,
Northflank, DigitalOcean, Oracle Cloud Free, Hetzner, Fly.io, Render, GCP,
Azure, and others. Each has a dedicated setup guide at
[docs.openclaw.ai/vps](https://docs.openclaw.ai/vps).

For remote access hardening details — token auth, Tailscale identity headers,
SSRF protection — see [Security and Governance](../operations/20-security.md).

---

## Keeping OpenClaw up to date

The recommended update path on any deployment:

```bash
openclaw update
```

This detects your install type (npm or git), fetches the latest version, runs
`openclaw doctor`, and restarts the Gateway. Use `--dry-run` to preview changes
first.

After updating:

```bash
openclaw doctor          # migrates config, verifies health
openclaw gateway restart # picks up the new install
openclaw health          # confirms the running version
```

For manual npm updates on a supervised install, stop the Gateway first to avoid
the package manager partially replacing files while the Gateway is still loading
them:

```bash
openclaw gateway stop
npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

---

## Common failure paths

| Symptom | Likely cause | Remedy |
|---|---|---|
| `openclaw: command not found` after install | npm global bin not in `$PATH` | Add `$(npm prefix -g)/bin` to `$PATH` in `~/.zshrc` or `~/.bashrc` |
| `openclaw gateway status` shows not running | Daemon not installed or failed to start | Run `openclaw doctor`; check `systemctl --user status openclaw-gateway` on Linux |
| Container exits immediately | Missing state volume or `EACCES` on `/home/node/.openclaw` | Check `docker logs openclaw-gateway`; fix host directory ownership to uid 1000 |
| Docker healthcheck reports `unhealthy` | Gateway not listening on `127.0.0.1:18789` inside container | Verify `OPENCLAW_GATEWAY_BIND=lan` is set; check for port conflict |
| OOM kill (exit 137) during `docker build` | Host has < 2 GB RAM | Use a host with at least 2 GB; `pnpm install` needs it |
| Control UI shows "pairing required" | First connection from new browser | Run `openclaw devices list` and `openclaw devices approve <id>` |
| VPS Gateway crashes and won't restart cleanly | Respawn causes two processes, OOM kills one | Set `OPENCLAW_NO_RESPAWN=1` in the systemd unit |

---

← Previous: [Project Structure: Monorepo Layout, 21 Packages, and Extension Inventory](./22-project-structure.md) · Next: [End-to-End Walkthroughs: DM Conversation, Cron Run, and Subagent Coordination](./24-walkthroughs.md) →
