---
title: "The Adapter Interface"
description: "The three-method contract every adapter must implement, the shared result type it returns, and why a single interface lets the orchestrator drive any backend."
category: the-agent
type: explanation
tags:
  [
    adapter interface,
    invoke,
    status,
    cancel,
    SwarmAdapter,
    AdapterResult,
    UsageSummary,
    unified result type,
    polymorphism,
    one orchestrator many backends,
    AdapterAgent,
    AdapterRuntime,
    AdapterResult,
    agent backend,
    process adapter,
    http adapter,
    mock adapter,
    LLM provider interface,
    exitCode,
    costUsd,
    sessionId,
    provider,
    model,
    summary,
  ]
keywords:
  [
    agent adapter contract,
    adapter abstraction,
    backend-agnostic orchestration,
    execution result shape,
    token usage tracking,
    structured adapter result,
    adapter polymorphism,
    swarm adapter typescript,
  ]
sources: [S18, S24, S25, S44]
---

**TL;DR** — The orchestrator never knows whether it is waking a local Claude CLI, an HTTP webhook, or a child process. That is by design: every backend implements the same three-method `SwarmAdapter` interface — `invoke`, `status`, `cancel` — and returns the same `AdapterResult` shape. This chapter explains why that contract exists, defines it precisely, and shows how the same "one interface, many implementations" idea applies to the LLM provider layer too.

# The Adapter Interface

## The problem: the orchestrator drives very different things

When the orchestrator schedules a task for an agent, it needs to do three things:

1. **Start the work** — tell the agent "your next heartbeat is ready."
2. **Check whether the work is still running** — is the process still alive? did the webhook succeed?
3. **Stop it if needed** — the board operator paused the agent mid-run.

The tricky part is that "the agent" might be a long-lived child process on the local machine, an HTTP endpoint that farms work to a remote system, or a local CLI session that talks to an LLM. Each of these has a completely different underlying mechanism.

If the orchestrator contained `if adapterType === "process"` branches scattered across its heartbeat, status-polling, and cancellation logic, adding a new backend would mean editing the orchestrator itself. That is the classic sign that an abstraction is missing.

The solution is a single interface that all backends satisfy — a contract so narrow that the orchestrator needs to know nothing about the implementation behind it.

## The three-method contract

The interface the orchestrator relies on has exactly three methods:

```ts
// src/adapters/types.ts
// Simplified view of the SwarmAdapter interface

interface SwarmAdapter {
  invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;
}
```

Let's look at each method and why it exists.

### `invoke` — start the work

`invoke` is called when the scheduler decides it is time for an agent to execute its next heartbeat. It receives two things:

- `agent` — a minimal description of the agent: its `id`, the `workspaceId` it belongs to, its `name`, and its `adapterConfig` (the backend-specific settings, such as which command to run or which URL to call).
- `context` — the execution context: a `runId` that uniquely identifies this execution window, plus the agent's session state and task assignments.

It returns a `Promise<AdapterResult>` — the unified result shape we'll define in the next section.

Here is the `AdapterAgent` type that `invoke` receives:

```ts
// The minimal description of an agent that every adapter works with.
// Adapters must not depend on the full DB row — only this shape is guaranteed.
interface AdapterAgent {
  id: string;
  workspaceId: string;
  name: string;
  adapterType: string | null;
  adapterConfig: unknown; // parsed from the agent's adapter_config column
}
```

The `adapterConfig` field is `unknown` at the interface level — each adapter interprets it as its own config shape (process command + args, HTTP URL + headers, etc.). The orchestrator hands it through without inspecting it.

### `status` — check a running execution

Some invocations are fire-and-forget (the HTTP adapter sends a request and the result arrives via callback); others are long-lived processes. The `status` method lets the orchestrator ask "is this run still going?" without knowing what mechanism it is polling.

```ts
// A heartbeat run — one execution window for one agent.
// The adapter uses run.id and run.externalRunId to look up the running job.
interface HeartbeatRun {
  id: string;
  externalRunId: string | null; // set by the adapter at invoke time, e.g. a PID or a remote job id
}

type RunStatus = "queued" | "running" | "succeeded" | "failed" | "cancelled" | "timed_out";
```

The `externalRunId` is the adapter's bookmark into its own world — a process ID for the process adapter, a remote job identifier for an HTTP adapter. The orchestrator stores whatever the adapter put there and hands it back on every `status` call.

### `cancel` — stop a running execution

When the board pauses an agent or the scheduler detects a run has exceeded its time budget, it calls `cancel`. The adapter is responsible for stopping whatever it started: sending SIGTERM to a child process (then SIGKILL after a grace period), or calling a remote cancellation endpoint. The orchestrator does not know how — it calls `cancel` and trusts the adapter to handle it.

```ts
cancel(run: HeartbeatRun): Promise<void>;
// Returns when cancellation has been initiated (not necessarily completed).
```

## The unified result type

`invoke` returns an `AdapterResult`. This is the shape every adapter must produce, regardless of whether it ran a process, called an HTTP endpoint, or simulated work.

```ts
// The result every adapter returns from invoke().
// All fields except exitCode and timedOut are optional — an adapter reports
// only what it has.
interface AdapterResult {
  // Execution outcome
  exitCode: number | null;     // process exit code; null for non-process adapters
  signal: string | null;       // Unix signal name if the process was killed
  timedOut: boolean;           // true if the run exceeded its time limit

  // Error details (optional)
  errorMessage?: string | null;
  errorCode?: string | null;
  errorFamily?: "transient_upstream" | null; // classifies recoverable upstream errors

  // LLM usage (optional — reported by adapters that talk to an LLM)
  usage?: UsageSummary;

  // Session continuity (optional — for adapters that resume sessions)
  sessionId?: string | null;    // legacy single-session id
  sessionParams?: Record<string, unknown> | null; // structured session state
  sessionDisplayId?: string | null; // human-readable session label

  // Billing and model identity (optional)
  provider?: string | null;    // e.g. "anthropic", "openai"
  model?: string | null;       // e.g. "claude-opus-4-5"
  costUsd?: number | null;     // total cost for this run in US dollars

  // Work output (optional)
  summary?: string | null;     // plain-text description of what the agent did
  resultJson?: Record<string, unknown> | null; // structured output, if any
}
```

You might expect the result to be much simpler — just a success/failure flag. But notice what would be lost:

- **Usage and cost** — the orchestrator needs `usage` and `costUsd` to write a `cost_event` record, roll up agent spending, and enforce budget limits.
- **Session continuity** — if the agent was mid-conversation with an LLM, `sessionParams` lets the next run resume from the same context window rather than starting fresh.
- **Provider and model identity** — the orchestrator stores which model the agent used so the board can see it in the cost dashboard.
- **`errorFamily`** — a `"transient_upstream"` error (e.g. a rate-limit or a temporary network failure) is treated differently from a logic error; the orchestrator can schedule a retry after `retryNotBefore` rather than immediately marking the agent as `error`.

The `UsageSummary` sub-type that `usage` refers to:

```ts
interface UsageSummary {
  inputTokens: number;
  outputTokens: number;
  cachedInputTokens?: number; // tokens served from the provider's prompt cache
}
```

All three counts are non-negative integers. `cachedInputTokens` is optional because not every provider reports cache hits.

## A complete picture: what an adapter looks like

Now we can see what we are building toward. Here is the full `SwarmAdapter` interface with both supporting types assembled:

```ts
// src/adapters/types.ts

export interface AdapterAgent {
  id: string;
  workspaceId: string;
  name: string;
  adapterType: string | null;
  adapterConfig: unknown;
}

export interface AdapterRuntime {
  sessionId: string | null;
  sessionParams: Record<string, unknown> | null;
  sessionDisplayId: string | null;
  taskKey: string | null;
}

export interface UsageSummary {
  inputTokens: number;
  outputTokens: number;
  cachedInputTokens?: number;
}

export interface AdapterResult {
  exitCode: number | null;
  signal: string | null;
  timedOut: boolean;
  errorMessage?: string | null;
  errorCode?: string | null;
  errorFamily?: "transient_upstream" | null;
  retryNotBefore?: string | null; // ISO 8601 timestamp
  usage?: UsageSummary;
  sessionId?: string | null;
  sessionParams?: Record<string, unknown> | null;
  sessionDisplayId?: string | null;
  provider?: string | null;
  model?: string | null;
  costUsd?: number | null;
  summary?: string | null;
  resultJson?: Record<string, unknown> | null;
}

export interface HeartbeatRun {
  id: string;
  externalRunId: string | null;
}

export type RunStatus =
  | "queued"
  | "running"
  | "succeeded"
  | "failed"
  | "cancelled"
  | "timed_out";

export interface InvocationContext {
  runId: string;
  agent: AdapterAgent;
  runtime: AdapterRuntime;
  config: Record<string, unknown>;
  context: Record<string, unknown>;
}

// The contract every adapter must satisfy.
export interface SwarmAdapter {
  invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;
}
```

This is the complete surface the orchestrator relies on. Everything else — how to spawn a child process, how to sign an HTTP request, how to resume a Claude session — lives inside the adapter, invisible to the caller.

## What implementations look like: a forward map

The interface is defined once. Every backend that plugs in must satisfy it:

| Adapter | `invoke` | `status` | `cancel` |
|---|---|---|---|
| **Mock** | Returns a fixed `AdapterResult` immediately; useful for tests | Always returns `"succeeded"` | No-op |
| **Process** | Spawns a child process; saves PID as `externalRunId` | Checks whether the PID is still alive | Sends SIGTERM, then SIGKILL after grace period |
| **HTTP** | Sends an outbound HTTP POST to a configured URL; non-2xx means failure | Polls a remote status endpoint, or awaits an async callback | Calls a remote cancel endpoint |
| **LLM CLI** | Launches a local CLI tool (e.g. Claude Code, Codex) with a prepared prompt; streams stdout to run logs | Checks the process lifecycle | Sends SIGTERM to the CLI process |

The orchestrator calls exactly the same three methods regardless of which row applies. Swapping one backend for another is a configuration change, not a code change.

## The same idea, one layer down: the LLM provider interface

The adapter interface solves the "one orchestrator, many agent backends" problem. If you look one level deeper — at what happens inside an LLM CLI adapter when it actually calls a language model — you find the same pattern applied again.

The LLM toolkit that the adapter uses defines a unified `StreamFunction` type:

```ts
// Simplified view of the LLM provider interface
type StreamFunction<TApi extends Api = Api> = (
  model: Model<TApi>,
  context: Context,
  options?: StreamOptions,
) => AssistantMessageEventStream;
```

The `StreamFunction` is a single calling convention that every provider implementation satisfies — Anthropic, OpenAI, Google, Groq, and many others. The adapter (or the CLI tool the adapter launches) does not branch on which provider is configured; it calls `streamFn(model, context)` and the matching provider implementation handles the HTTP details.

The analogy is direct:

| Layer | Interface | Implementations |
|---|---|---|
| Orchestrator → agent backends | `SwarmAdapter` (`invoke` / `status` / `cancel`) | Mock, Process, HTTP, LLM CLI |
| LLM adapter → model providers | `StreamFunction` (`model`, `context`) | Anthropic, OpenAI, Google, Groq, … |

Both layers use the same design principle: agree on a narrow contract, put the per-implementation complexity behind it, and let the caller stay oblivious.

This is why you can build a system where the board operator switches an agent from a Claude-backed adapter to a Gemini-backed adapter by changing a config field — neither the orchestrator nor the agent-facing code changes, only the implementation behind the interface does.

## Why this matters for what comes next

In the chapters that follow, we will implement each concrete adapter. Each one is a TypeScript class (or module) that exports a single object implementing `SwarmAdapter`. Because they share a contract, we can test them against the same harness, register them in the same registry, and let the orchestrator route to any of them identically.

When you read [The Mock Adapter](./mock-adapter.md) next, you will see that the minimal implementation is only a handful of lines — the interface is intentionally thin. The complexity lives in the process and HTTP adapters, but the contract they all share remains the same three methods.

---

← Previous: [Your First Agent](../getting-started/your-first-agent.md) · Next: [The Mock Adapter](./mock-adapter.md) →
