---
title: "Process and HTTP Adapters"
description: "Build two adapter variants — a process adapter that spawns a local CLI agent and an HTTP adapter that fires a webhook — and see how the SwarmAdapter interface generalises to any execution model."
category: the-agent
type: tutorial
tags:
  - process adapter
  - HTTP adapter
  - webhook adapter
  - spawn
  - child process
  - cwd
  - env
  - timeout
  - exit code
  - outbound fetch
  - fire and forget
  - AbortController
  - agentId
  - runId
  - adapter authoring
  - no-remote-git rule
  - SwarmAdapter
  - InvocationContext
  - AdapterResult
  - AdapterAgent
  - timeoutSec
  - graceSec
  - timeoutMs
  - payloadTemplate
  - invoke
keywords:
  - run CLI as agent
  - spawn subprocess
  - process adapter implementation
  - HTTP webhook agent
  - fetch with abort
  - adapter state persistence
  - local working directory adapter
  - git push forbidden adapter
  - adapter cwd invariant
  - adapter authoring contract
  - child_process spawn
  - asString asNumber config coercion
  - buildSwarmEnv environment variables
sources: [S27, S28, S29, S23]
---

**TL;DR** — The `SwarmAdapter` interface is not limited to LLMs. In this chapter we build two more adapters: a *process adapter* that spawns any local program as the agent, and an *HTTP adapter* that fires a webhook and treats a 2xx response as success. By the end you will understand how either execution model maps onto the same result type, and you will know the authoring rules that keep adapter code safe to deploy alongside other runs.

# Process and HTTP Adapters

## Where we are

In [The Adapter Interface](./adapter-interface.md) we defined the `SwarmAdapter` contract. To recap: every adapter must implement three methods — `invoke`, `status`, and `cancel`. The one we care about for writing output is `invoke`:

```ts
invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>
```

`agent` carries the agent's `id`, `workspaceId`, `name`, and `adapterConfig` (the backend-specific settings). `context` carries the `runId`, the `runtime` session state, and a `config` map of per-run settings the operator provided. The return type, `AdapterResult`, is the unified shape every adapter returns: required fields `exitCode` and `timedOut`, plus optional fields like `usage`, `sessionId`, `summary`, and `costUsd` that adapters report when relevant.

In [The Mock Adapter](./mock-adapter.md) we implemented the simplest possible version of that contract by returning a hard-coded result without touching anything external.

Now we want to do something real. Two motivations pull in different directions:

- **Local programs** — an agent might be a CLI you already have installed, such as Claude Code or Codex CLI. You want to hand it a prompt, wait for it to finish, and collect its output. That is exactly what a *process adapter* does.
- **Remote services** — an agent might live entirely behind an HTTP API: a webhook endpoint you control, a serverless function, or a hosted service. You do not want to wait for it to finish; firing the request and getting a 2xx back is enough. That is the *HTTP adapter*.

Both fit inside the same `AdapterResult` envelope. Let's build each one in turn.

---

## Part 1 — The process adapter

### The problem: running a program and knowing when it succeeded

Spawning a child process sounds straightforward, but there are three things you need to handle cleanly:

1. **Configuration** — which binary to run, what arguments, which directory, which environment variables, how long to wait before giving up.
2. **Timeout** — if the agent hangs, you need to kill it and return a result that says it timed out rather than letting the run block forever.
3. **Exit code** — a non-zero exit code means the agent reported failure; you need to carry the code and captured output back in the result.

Let's work through each concern.

### Step 1 — Small helpers for reading config safely

The `context.config` field arriving in `invoke` is typed as `Record<string, unknown>` — the orchestrator passes it through without inspecting it. That means every value is `unknown` until we check it. We need a handful of narrow coercion helpers that extract a field, verify its type, and fall back to a safe default when the field is missing or the wrong type.

Here they are in full — they are trivial, so we define them once at the top of the adapter file:

```ts
// src/adapters/shared/config.ts — small config-coercion helpers

/** Return value as a string, or the fallback when it is missing/non-string. */
function asString(value: unknown, fallback: string): string {
  return typeof value === "string" ? value : fallback;
}

/** Return value as a number, or the fallback when it is missing/non-number. */
function asNumber(value: unknown, fallback: number): number {
  return typeof value === "number" ? value : fallback;
}

/** Return value as a string[], or [] when it is missing or not an array of strings. */
function asStringArray(value: unknown): string[] {
  if (!Array.isArray(value)) return [];
  return value.filter((v): v is string => typeof v === "string");
}

/** Return value as Record<string, unknown>, or {} when it is missing or not a plain object. */
function parseObject(value: unknown): Record<string, unknown> {
  if (value !== null && typeof value === "object" && !Array.isArray(value)) {
    return value as Record<string, unknown>;
  }
  return {};
}
```

These four functions appear throughout the process and HTTP adapters. Every config field goes through one of them so a misconfigured agent config produces a predictable fallback, not a runtime crash.

### Step 2 — Reading the process adapter's configuration

With those helpers in place, we can read the process adapter's configuration from `context.config`:

```ts
// src/adapters/process/invoke.ts — configuration extraction (simplified view)
import type { AdapterAgent, InvocationContext, AdapterResult } from "../types.js";

export async function invoke(
  agent: AdapterAgent,
  context: InvocationContext,
): Promise<AdapterResult> {
  const { runId, config } = context;

  // command is required — no sensible default exists
  const command = asString(config.command, "");
  if (!command) throw new Error("Process adapter missing command");

  const args       = asStringArray(config.args);         // extra CLI flags; defaults to []
  const cwd        = asString(config.cwd, process.cwd()); // working directory; defaults to the server's cwd
  const envConfig  = parseObject(config.env);            // extra env vars to inject; defaults to {}

  const timeoutSec = asNumber(config.timeoutSec, 0);     // 0 means no timeout
  const graceSec   = asNumber(config.graceSec, 15);      // seconds after SIGTERM before SIGKILL
  // ...
}
```

A few things to notice:

- `command` has no default. If the operator forgets it, we throw immediately rather than silently running nothing.
- `cwd` falls back to the server's own working directory, which is safe because the runner sets its cwd to the workspace directory before invoking the adapter.
- `timeoutSec: 0` means "no timeout" — the process runs until it exits on its own.
- `graceSec` controls how long we wait between sending SIGTERM and sending SIGKILL. The default of 15 seconds gives the child time to flush buffers and exit cleanly.

### Step 3 — Building the environment

The agent process needs context about the run. We build an environment by starting from the process's own environment, then stamping in the Swarm identity variables the runner needs the child to see (the agent id, the run id, and similar platform keys), and finally layering in any custom keys the operator supplied in `config.env`.

Here is a minimal `buildSwarmEnv` that produces those identity variables:

```ts
// Produces the platform env block the child process should inherit.
// agent.id and agent.workspaceId are the only fields we stamp unconditionally.
function buildSwarmEnv(agent: AdapterAgent): Record<string, string> {
  return {
    SWARM_AGENT_ID: agent.id,
    SWARM_COMPANY_ID: agent.workspaceId,
  };
}
```

And a guard that makes sure `PATH` is always present (without it, the child may fail to find its own dependencies):

```ts
// Merge envBase into the process environment and ensure PATH is always defined.
function ensurePathInEnv(
  envBase: Record<string, string | undefined>,
): Record<string, string> {
  const result: Record<string, string> = {};
  for (const [k, v] of Object.entries(envBase)) {
    if (typeof v === "string") result[k] = v;
  }
  if (!result["PATH"] && process.env["PATH"]) {
    result["PATH"] = process.env["PATH"];
  }
  return result;
}
```

Putting them together:

```ts
// Building the merged environment inside invoke() (simplified view)
const swarmVars = buildSwarmEnv(agent);
const operatorVars: Record<string, string> = {};
for (const [k, v] of Object.entries(envConfig)) {
  if (typeof v === "string") operatorVars[k] = v;
}

const runtimeEnv = ensurePathInEnv({
  ...process.env,     // inherit the server's full environment
  ...swarmVars,       // stamp in the agent/run identity
  ...operatorVars,    // let the operator override or add anything else
});
```

The spread order matters: operator-supplied variables win over the Swarm defaults, and both win over the server's own environment for any key that overlaps.

### Step 4 — Spawning the child and handling the three outcomes

Now we can spawn the process. The core is Node's built-in `child_process.spawn`, wrapped to collect output, enforce the timeout, and resolve with a structured result. Here is a minimal `runChildProcess` that captures exactly what we need:

```ts
import { spawn } from "child_process";

interface ChildResult {
  exitCode: number | null;
  signal:   string | null;
  timedOut: boolean;
  stdout:   string;
  stderr:   string;
}

/**
 * Spawn command with args, collect stdout/stderr, enforce a timeout.
 * Resolves (never rejects) with a ChildResult.
 */
async function runChildProcess(
  runId:      string,
  command:    string,
  args:       string[],
  opts: {
    cwd:        string;
    env:        Record<string, string>;
    timeoutSec: number;   // 0 = no timeout
    graceSec:   number;
    onLog?:     (line: string) => void;
  },
): Promise<ChildResult> {
  return new Promise((resolve) => {
    const proc = spawn(command, args, {
      cwd: opts.cwd,
      env: opts.env,
      stdio: ["ignore", "pipe", "pipe"],
    });

    let stdout = "";
    let stderr = "";
    let timedOut = false;
    let killTimer: ReturnType<typeof setTimeout> | null = null;
    let graceTimer: ReturnType<typeof setTimeout> | null = null;

    // Enforce the timeout: SIGTERM first, SIGKILL after graceSec.
    if (opts.timeoutSec > 0) {
      killTimer = setTimeout(() => {
        timedOut = true;
        proc.kill("SIGTERM");
        graceTimer = setTimeout(() => proc.kill("SIGKILL"), opts.graceSec * 1000);
      }, opts.timeoutSec * 1000);
    }

    proc.stdout.on("data", (chunk: Buffer) => {
      const line = chunk.toString();
      stdout += line;
      opts.onLog?.(line);
    });
    proc.stderr.on("data", (chunk: Buffer) => {
      const line = chunk.toString();
      stderr += line;
      opts.onLog?.(line);
    });

    proc.on("close", (code, signal) => {
      if (killTimer)  clearTimeout(killTimer);
      if (graceTimer) clearTimeout(graceTimer);
      resolve({
        exitCode: code,
        signal:   signal,
        timedOut,
        stdout,
        stderr,
      });
    });
  });
}
```

The `onLog` callback is optional — when the orchestrator provides it, each line of stdout/stderr flows to the run's live log view as the process runs.

With `runChildProcess` in hand, invoking the child and mapping the three possible outcomes is straightforward:

```ts
const proc = await runChildProcess(runId, command, args, {
  cwd,
  env: runtimeEnv,
  timeoutSec,
  graceSec,
  onLog: context.onLog,
});
```

There are exactly three outcomes we need to map to `AdapterResult`:

**Outcome A — timeout:**

```ts
if (proc.timedOut) {
  return {
    exitCode: proc.exitCode,
    timedOut: true,
    errorMessage: `Timed out after ${timeoutSec}s`,
  };
}
```

**Outcome B — non-zero exit:**

```ts
if ((proc.exitCode ?? 0) !== 0) {
  return {
    exitCode: proc.exitCode,
    timedOut: false,
    errorMessage: `Process exited with code ${proc.exitCode ?? -1}`,
    resultJson: {
      stdout: proc.stdout,
      stderr: proc.stderr,
    },
  };
}
```

Note that we still capture `stdout` and `stderr` on failure — the caller often needs them to diagnose the error.

**Outcome C — success (exit code 0):**

```ts
return {
  exitCode: proc.exitCode,
  timedOut: false,
  summary:  proc.stdout.trim().slice(0, 500) || undefined,
  resultJson: {
    stdout: proc.stdout,
    stderr: proc.stderr,
  },
};
```

A success result has no `errorMessage`. The caller treats the absence of `errorMessage` as the happy-path signal. We populate `summary` with the first 500 characters of stdout so the run's dashboard entry shows what the agent reported.

### The complete process adapter

Putting everything together in one file so you can follow it end-to-end:

```ts
// src/adapters/process/invoke.ts
import { spawn } from "child_process";
import type { AdapterAgent, InvocationContext, AdapterResult } from "../types.js";

// ── Config coercion helpers ──────────────────────────────────────────────────

function asString(value: unknown, fallback: string): string {
  return typeof value === "string" ? value : fallback;
}

function asNumber(value: unknown, fallback: number): number {
  return typeof value === "number" ? value : fallback;
}

function asStringArray(value: unknown): string[] {
  if (!Array.isArray(value)) return [];
  return value.filter((v): v is string => typeof v === "string");
}

function parseObject(value: unknown): Record<string, unknown> {
  if (value !== null && typeof value === "object" && !Array.isArray(value)) {
    return value as Record<string, unknown>;
  }
  return {};
}

// ── Environment helpers ──────────────────────────────────────────────────────

function buildSwarmEnv(agent: AdapterAgent): Record<string, string> {
  return {
    SWARM_AGENT_ID:   agent.id,
    SWARM_COMPANY_ID: agent.workspaceId,
  };
}

function ensurePathInEnv(
  envBase: Record<string, string | undefined>,
): Record<string, string> {
  const result: Record<string, string> = {};
  for (const [k, v] of Object.entries(envBase)) {
    if (typeof v === "string") result[k] = v;
  }
  if (!result["PATH"] && process.env["PATH"]) {
    result["PATH"] = process.env["PATH"];
  }
  return result;
}

// ── Child process runner ─────────────────────────────────────────────────────

interface ChildResult {
  exitCode: number | null;
  signal:   string | null;
  timedOut: boolean;
  stdout:   string;
  stderr:   string;
}

async function runChildProcess(
  runId:   string,
  command: string,
  args:    string[],
  opts: {
    cwd:        string;
    env:        Record<string, string>;
    timeoutSec: number;
    graceSec:   number;
    onLog?:     (line: string) => void;
  },
): Promise<ChildResult> {
  return new Promise((resolve) => {
    const proc = spawn(command, args, {
      cwd:   opts.cwd,
      env:   opts.env,
      stdio: ["ignore", "pipe", "pipe"],
    });

    let stdout   = "";
    let stderr   = "";
    let timedOut = false;
    let killTimer:  ReturnType<typeof setTimeout> | null = null;
    let graceTimer: ReturnType<typeof setTimeout> | null = null;

    if (opts.timeoutSec > 0) {
      killTimer = setTimeout(() => {
        timedOut = true;
        proc.kill("SIGTERM");
        graceTimer = setTimeout(() => proc.kill("SIGKILL"), opts.graceSec * 1000);
      }, opts.timeoutSec * 1000);
    }

    proc.stdout.on("data", (chunk: Buffer) => {
      const line = chunk.toString();
      stdout += line;
      opts.onLog?.(line);
    });
    proc.stderr.on("data", (chunk: Buffer) => {
      const line = chunk.toString();
      stderr += line;
      opts.onLog?.(line);
    });

    proc.on("close", (code, signal) => {
      if (killTimer)  clearTimeout(killTimer);
      if (graceTimer) clearTimeout(graceTimer);
      resolve({ exitCode: code, signal, timedOut, stdout, stderr });
    });
  });
}

// ── The adapter's invoke method ──────────────────────────────────────────────

export async function invoke(
  agent:   AdapterAgent,
  context: InvocationContext,
): Promise<AdapterResult> {
  const { runId, config } = context;

  const command    = asString(config.command, "");
  if (!command) throw new Error("Process adapter missing command");

  const args       = asStringArray(config.args);
  const cwd        = asString(config.cwd, process.cwd());
  const envConfig  = parseObject(config.env);
  const timeoutSec = asNumber(config.timeoutSec, 0);
  const graceSec   = asNumber(config.graceSec, 15);

  const swarmVars    = buildSwarmEnv(agent);
  const operatorVars: Record<string, string> = {};
  for (const [k, v] of Object.entries(envConfig)) {
    if (typeof v === "string") operatorVars[k] = v;
  }
  const runtimeEnv = ensurePathInEnv({
    ...process.env,
    ...swarmVars,
    ...operatorVars,
  });

  const proc = await runChildProcess(runId, command, args, {
    cwd,
    env:        runtimeEnv,
    timeoutSec,
    graceSec,
    onLog:      context.onLog,
  });

  if (proc.timedOut) {
    return {
      exitCode:     proc.exitCode,
      timedOut:     true,
      errorMessage: `Timed out after ${timeoutSec}s`,
    };
  }

  if ((proc.exitCode ?? 0) !== 0) {
    return {
      exitCode:     proc.exitCode,
      timedOut:     false,
      errorMessage: `Process exited with code ${proc.exitCode ?? -1}`,
      resultJson:   { stdout: proc.stdout, stderr: proc.stderr },
    };
  }

  return {
    exitCode:   proc.exitCode,
    timedOut:   false,
    summary:    proc.stdout.trim().slice(0, 500) || undefined,
    resultJson: { stdout: proc.stdout, stderr: proc.stderr },
  };
}
```

### Configuration reference

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `command` | `string` | — (required) | Binary name or absolute path to execute |
| `args` | `string[]` | `[]` | Extra CLI arguments passed after `command` |
| `cwd` | `string` | `process.cwd()` | Working directory for the child process |
| `env` | `Record<string, string>` | `{}` | Extra env vars merged on top of Swarm base env |
| `timeoutSec` | `number` | `0` (no timeout) | Kill the process if it exceeds this many seconds |
| `graceSec` | `number` | `15` | Seconds between SIGTERM and SIGKILL |

### A note on richer local CLI adapters

The process adapter above is intentionally minimal — it is the right base for any program that reads from stdin or acts on env vars and exits with a meaningful code. Real CLI agents such as Claude Code or Codex CLI demand more from their adapter:

- **Session resume** — the CLI stores a session id after each run; the next run passes `--resume <session-id>` to continue the conversation rather than starting fresh.
- **Stream parsing** — the CLI emits newline-delimited JSON; the adapter parses that stream to extract a structured result, token usage, and cost rather than treating stdout as an opaque blob.
- **Execution targets** — the CLI can run locally, over SSH, or inside a sandbox container; the adapter must sync the working directory to the remote before the run and sync changes back afterwards.

Those concerns layer on top of the same `runChildProcess` foundation we built here. S29 shows how a production local-CLI adapter handles them. We keep that detail separate because it belongs with the specific CLI it wraps, not with the general process adapter pattern.

---

## Part 2 — The HTTP adapter

### The problem: triggering an agent that lives behind a URL

Some agents are not a binary you run locally — they are an HTTP endpoint. You send a request describing the task, the endpoint processes it asynchronously, and a 2xx response tells you the agent received and accepted the work. This is the fire-and-forget webhook pattern.

The HTTP adapter's job is:

1. **Build the request** — merge a configurable payload template with the run's identity fields (`agentId`, `runId`, `context`).
2. **Send it** — use `fetch` with a configurable method and headers.
3. **Enforce a timeout** — if the endpoint is slow to respond, abort the request and return a timeout result.
4. **Interpret the response** — 2xx means success; anything else throws so the run records a failure.

### Step 1 — Reading the configuration

We reuse the same four config helpers from the process adapter (`asString`, `asNumber`, `parseObject`):

```ts
// src/adapters/http/invoke.ts — configuration extraction (simplified view)
import type { AdapterAgent, InvocationContext, AdapterResult } from "../types.js";

export async function invoke(
  agent:   AdapterAgent,
  context: InvocationContext,
): Promise<AdapterResult> {
  const { runId, config } = context;

  const url = asString(config.url, "");
  if (!url) throw new Error("HTTP adapter missing url");

  const method          = asString(config.method, "POST");     // default POST
  const timeoutMs       = asNumber(config.timeoutMs, 0);       // 0 = no timeout
  const headers         = parseObject(config.headers) as Record<string, string>;
  const payloadTemplate = parseObject(config.payloadTemplate); // base fields to merge
  // ...
}
```

`url` is required for the same reason `command` was required in the process adapter — there is no safe default. `method` defaults to `"POST"` because that is the conventional choice for webhook triggers.

### Step 2 — Building the request body

Here is the key design decision: we always inject the run's identity into the body, regardless of what the operator put in `payloadTemplate`.

```ts
const body = {
  ...payloadTemplate,   // operator-defined fields come first
  agentId: agent.id,   // then we stamp the agent identity
  runId,               // and the run id
  context,             // and the full execution context
};
```

The spread order matters: `agentId`, `runId`, and `context` always override anything in `payloadTemplate` that uses those keys. The endpoint can rely on these fields being present in every invocation.

### Step 3 — Setting up the AbortController timeout

JavaScript's `fetch` does not have a built-in timeout. We use `AbortController` to cancel the request if it takes too long:

```ts
const controller = new AbortController();
const timer = timeoutMs > 0
  ? setTimeout(() => controller.abort(), timeoutMs)
  : null;
```

We only create the timer when `timeoutMs > 0`. When it is zero, we pass no signal to `fetch` and let the request run until the network resolves it.

### Step 4 — Sending the request and mapping outcomes

```ts
try {
  const res = await fetch(url, {
    method,
    headers: {
      "content-type": "application/json",
      ...headers,            // custom headers can override content-type if needed
    },
    body: JSON.stringify(body),
    ...(timer ? { signal: controller.signal } : {}),
  });

  if (!res.ok) {
    // !res.ok covers 4xx, 5xx — throw so the run records failure
    throw new Error(`HTTP invoke failed with status ${res.status}`);
  }

  return {
    exitCode: 0,
    timedOut: false,
    summary:  `HTTP ${method} ${url}`,
  };
} catch (err) {
  if (timer && err instanceof Error && err.name === "AbortError") {
    return {
      exitCode:     null,
      timedOut:     true,
      errorMessage: `HTTP ${method} ${url} timed out after ${timeoutMs}ms`,
      errorCode:    "timeout",
    };
  }
  throw err;  // re-throw anything else (network error, non-2xx, etc.)
} finally {
  if (timer) clearTimeout(timer);  // always clean up the timer
}
```

There are three outcomes:

| Outcome | Condition | What we return |
|---------|-----------|----------------|
| **Success** | `res.ok` (2xx) | `{ exitCode: 0, timedOut: false, summary: "HTTP POST <url>" }` |
| **Timeout** | `AbortError` caught and timer was set | `{ exitCode: null, timedOut: true, errorCode: "timeout" }` |
| **Failure** | Non-2xx or other network error | re-thrown (the run records the error) |

You might wonder why a non-2xx throws rather than returning a shaped result like the process adapter does. The reason is that `fetch` does not throw on non-2xx by design — `res.ok` is the check — so we throw manually to propagate the failure through the same channel as real network errors. The orchestrator's error handler will capture it and record the status code.

### The complete HTTP adapter

```ts
// src/adapters/http/invoke.ts
import type { AdapterAgent, InvocationContext, AdapterResult } from "../types.js";

// ── Config coercion helpers (same as process adapter) ────────────────────────

function asString(value: unknown, fallback: string): string {
  return typeof value === "string" ? value : fallback;
}

function asNumber(value: unknown, fallback: number): number {
  return typeof value === "number" ? value : fallback;
}

function parseObject(value: unknown): Record<string, unknown> {
  if (value !== null && typeof value === "object" && !Array.isArray(value)) {
    return value as Record<string, unknown>;
  }
  return {};
}

// ── The adapter's invoke method ──────────────────────────────────────────────

export async function invoke(
  agent:   AdapterAgent,
  context: InvocationContext,
): Promise<AdapterResult> {
  const { runId, config } = context;

  const url = asString(config.url, "");
  if (!url) throw new Error("HTTP adapter missing url");

  const method          = asString(config.method, "POST");
  const timeoutMs       = asNumber(config.timeoutMs, 0);
  const headers         = parseObject(config.headers) as Record<string, string>;
  const payloadTemplate = parseObject(config.payloadTemplate);

  const body = {
    ...payloadTemplate,
    agentId: agent.id,
    runId,
    context,
  };

  const controller = new AbortController();
  const timer = timeoutMs > 0
    ? setTimeout(() => controller.abort(), timeoutMs)
    : null;

  try {
    const res = await fetch(url, {
      method,
      headers: {
        "content-type": "application/json",
        ...headers,
      },
      body: JSON.stringify(body),
      ...(timer ? { signal: controller.signal } : {}),
    });

    if (!res.ok) {
      throw new Error(`HTTP invoke failed with status ${res.status}`);
    }

    return {
      exitCode: 0,
      timedOut: false,
      summary:  `HTTP ${method} ${url}`,
    };
  } catch (err) {
    if (timer && err instanceof Error && err.name === "AbortError") {
      return {
        exitCode:     null,
        timedOut:     true,
        errorMessage: `HTTP ${method} ${url} timed out after ${timeoutMs}ms`,
        errorCode:    "timeout",
      };
    }
    throw err;
  } finally {
    if (timer) clearTimeout(timer);
  }
}
```

### Configuration reference

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `url` | `string` | — (required) | Target URL to POST (or method of choice) to |
| `method` | `string` | `"POST"` | HTTP method |
| `timeoutMs` | `number` | `0` (no timeout) | Abort the request after this many milliseconds |
| `headers` | `Record<string, string>` | `{}` | Extra headers; merged after `content-type: application/json` |
| `payloadTemplate` | `Record<string, unknown>` | `{}` | Base body fields; `agentId`, `runId`, `context` are always appended |

---

## Part 3 — The adapter authoring contract

We have now built two adapters. Before we ship either one, there is an authoring contract you must understand — it is what keeps adapter code safe when it runs alongside many other agents, possibly on different hosts or in isolated workspaces.

### State lives in the local working directory

An adapter runs as part of a *run* — a single heartbeat of an agent. When that run finishes, the orchestrator needs to carry any state the agent produced forward to the next run. The only reliable mechanism for this is the **local working directory (cwd)** that the runner prepared for the run.

The contract is: if an agent produces output (commits, files, reports) that future runs need, that output must be present in the local cwd by the time `invoke()` returns. The runner handles syncing that directory to wherever the agent ran (local disk, SSH host, sandbox container) and syncing changes back afterwards. Your adapter code does not need to do the sync — but it must not bypass the mechanism.

### The no-remote-git rule

Here is the rule that comes up most often in practice:

> **An adapter must never `git push` from inside its runtime code.** Never assume the local worktree has any `git remote` configured.

Why does this matter? The runner resolves a local execution workspace (a worktree) for each run. It syncs that workspace to wherever the agent runs and syncs changes back when the run finishes. If the agent commits locally and then pushes to a remote, the local worktree is no longer the source of truth. The next run's sync will diverge. Dependent tasks gated on the local worktree being up-to-date will stall. Isolated workspaces that have no remote configured at all will fail outright.

The safe pattern is: commit locally, let the runner's workspace sync carry the commit forward.

If you are building an adapter that runs the agent on a different host (over SSH, inside a sandbox, in a remote container), use the round-trip helpers the framework provides: bundle the local cwd to the remote before the run, and sync remote-side changes — including new commits — back into the local cwd after the run. Both operations happen with no `git remote` configured.

A failed sync-back is a run-level error. Do not swallow restore errors — let them surface so the orchestrator can gate dependent tasks appropriately.

### The static check

A static analysis script scans all adapter source files for unapproved `git push` invocations and fails the CI `policy` job if it finds one. If you have a legitimate reason to push (this is rare), add an explicit opt-in comment on the line:

```ts
// swarm:allow-git-push: <reason why this push is safe>
await exec("git push origin main");
```

The comment makes the exception visible in code review rather than silent.

### Summary of adapter authoring rules

| Rule | Why |
|------|-----|
| State that future runs need must be in the local cwd when `invoke()` returns | The runner syncs cwd forward; a remote push bypasses that |
| Never `git push` from adapter runtime code | Diverges the local worktree from the source of truth |
| Never assume a `git remote` is configured | Isolated workspaces may have none |
| Failed workspace sync-back is a run-level error — propagate it | Downstream tasks are gated on successful finalize |
| To opt in to a push, add an explicit `// swarm:allow-git-push: <reason>` comment | Keeps exceptions visible in review |

---

## How both adapters satisfy the same interface

It is worth stepping back to see that both adapters return the same `AdapterResult` shape we defined in [The Adapter Interface](./adapter-interface.md):

| Field | Process adapter | HTTP adapter |
|-------|-----------------|--------------|
| `exitCode` | child's exit code | `0` on success; `null` on timeout |
| `timedOut` | `true` if `timeoutSec` exceeded | `true` if `timeoutMs` exceeded |
| `errorMessage` | set on timeout or non-zero exit | set on timeout |
| `resultJson` | `{ stdout, stderr }` on non-zero exit or success | — (not set on success) |
| `summary` | first 500 chars of stdout on success | `"HTTP POST <url>"` on success |

The orchestrator does not need to know which adapter type ran. It reads the same fields and records the same run-level outcome regardless.

---

## Try it yourself

### Wrap a shell one-liner as a process agent

Call the process adapter's `invoke` function directly — the same method the runner calls at runtime — with a minimal `AdapterAgent` and `InvocationContext`:

```ts
import { invoke } from "./src/adapters/process/invoke.js";
import type { AdapterAgent, InvocationContext } from "./src/adapters/types.js";

const agent: AdapterAgent = {
  id: "agent-local-shell",
  workspaceId: "ws-demo",
  name: "Shell one-liner",
  adapterConfig: {},
};

const context: InvocationContext = {
  runId: "run-001",
  config: {
    command: "bash",
    args: ["-c", "echo hello from the agent"],
    timeoutSec: 10,
  },
  onLog: (line) => process.stdout.write(line),
};

const result = await invoke(agent, context);
console.log(result);
// { exitCode: 0, timedOut: false, summary: "hello from the agent", resultJson: { ... } }
```

Run this file with `node --loader ts-node/esm exercise.ts` (or your project's usual TypeScript runner). You should see "hello from the agent" arrive through `onLog` in real time and the final `result.exitCode` be `0`. (You could wrap this in a CLI subcommand later once you have a command-line entry point.)

### Point the HTTP adapter at a local echo server

Start a local echo server in one terminal:

```bash
npx @nicolo-ribaudo/http-echo-server 3999
```

Then configure an agent with:

```yaml
adapter: http
config:
  url: http://localhost:3999/agent-hook
  method: POST
  timeoutMs: 5000
  payloadTemplate:
    source: swarm
```

Trigger a run. The echo server will log the full JSON body — you will see `agentId`, `runId`, and `context` injected alongside your `source` field.

### Add a per-adapter timeout and see what happens

Set `timeoutSec: 1` on the process adapter and run a command that takes longer than a second (e.g., `sleep 5`). The result will have `timedOut: true` and `errorMessage: "Timed out after 1s"`. The same experiment works with `timeoutMs: 100` on the HTTP adapter pointed at a slow endpoint.

---

← Previous: [The LLM Adapter](./llm-adapter.md) · Next: [The Adapter Registry](./adapter-registry.md) →
