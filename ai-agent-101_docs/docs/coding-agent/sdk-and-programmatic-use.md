---
title: "The SDK: createAgentSession and Programmatic Use"
description: How to embed the coding agent in your own Node.js program using createAgentSession, from the minimal two-line setup to fully explicit configuration.
category: coding-agent
type: tutorial
tags: [SDK, createAgentSession, programmatic, options, minimal, full-control, Node.js, embedding, coding-agent, integration, API, examples, AuthStorage, ModelRegistry, ResourceLoader, SessionManager, SettingsManager, tools, customTools, noTools, thinkingLevel, subscribe, events, agentDir, cwd]
keywords: [programmatic agent, library embedding, agent session factory, SDK entry point, no CLI, custom auth, in-memory session, tool allowlist]
sources: [S66, S74, S75, S76]
---

**TL;DR** — `createAgentSession` is the single function that assembles every manager and service the coding agent needs and hands back a ready `AgentSession`. This chapter walks from the simplest possible usage — three lines and a prompt — to a fully explicit "full-control" configuration that bypasses all file-system discovery. By the end you will know every option `createAgentSession` accepts, when to reach for each one, and how to subscribe to the events it emits.

# The SDK: createAgentSession and Programmatic Use

## When the CLI is not what you want

The CLI we explored in the [previous chapter](./cli-entry-point.md) is great for running the agent interactively. But sometimes you want the agent as a library component inside a larger program — a web server that answers coding questions, a CI step that auto-generates tests, a desktop app that drives an embedded terminal. Launching a subprocess and scraping its output would be fragile and slow.

The coding agent exposes a single function — `createAgentSession` — that wires up every internal service (authentication, model resolution, tool registration, session persistence) and returns a ready-to-use `AgentSession`. You call it from your own TypeScript or JavaScript code, and you stay in full control of the process.

### What createAgentSession assembles

Before diving in, a quick recap of what `createAgentSession` puts together. If you have read the [AgentSession chapter](./agent-session-core.md), you will recognise these pieces; if not, here is what each service does:

- **`AuthStorage`** — stores and retrieves API credentials (keys, OAuth tokens). Backed by a JSON file on disk, or an in-memory store for testing.
- **`ModelRegistry`** — knows which provider models are available and how to obtain their API keys from `AuthStorage`.
- **`ResourceLoader`** — discovers and loads skills (prompt snippets), extensions (plug-in code), context files (`AGENTS.md`), and the system prompt. Can also be replaced with a plain object if you want zero discovery.
- **`SessionManager`** — handles the conversation history — persisting it to disk between runs, or keeping it in memory.
- **`SettingsManager`** — holds configuration knobs (compaction, retry policy, HTTP timeouts, model defaults).

`createAgentSession` creates whichever of these you did not supply yourself, wires them together, instantiates the underlying `Agent` and session, and returns the assembled `AgentSession` along with a `LoadExtensionsResult` and an optional `modelFallbackMessage` (a warning string if the saved model could not be restored).

```ts
// Return type of createAgentSession
interface CreateAgentSessionResult {
  session: AgentSession;
  extensionsResult: LoadExtensionsResult;
  modelFallbackMessage?: string;
}
```

Now let's start from the smallest thing that works.

## Step 1 — The minimal example

The simplest possible usage passes no options at all:

```ts
// Simplified from examples/sdk/01-minimal.ts
import { createAgentSession } from "coding-agent";

const { session } = await createAgentSession();

try {
  session.subscribe((event) => {
    if (
      event.type === "message_update" &&
      event.assistantMessageEvent.type === "text_delta"
    ) {
      process.stdout.write(event.assistantMessageEvent.delta);
    }
  });

  await session.prompt("What files are in the current directory?");

  session.state.messages.forEach((msg) => {
    console.log(msg);
  });
  console.log();
} finally {
  session.dispose();
}
```

Three things happen in this tiny program:

1. **`createAgentSession()` with no arguments** — every service is created with its default: `AuthStorage` reads from `~/.xzy/auth.json`, `ModelRegistry` is built from that storage, `ResourceLoader` scans `process.cwd()` and `~/.xzy/` for skills and extensions, and `SessionManager` persists the conversation to disk under `process.cwd()`.

2. **`session.subscribe(callback)`** — registers a listener for events the session emits. Here we print every streamed text delta to stdout, so the assistant's reply appears incrementally as it arrives.

3. **`session.prompt("…")`** — sends a user message and runs the agent loop to completion. The loop may invoke tools (reading files, running bash commands) before producing the final text reply. `prompt` is an `async` function; `await` it to know when the turn is finished.

4. **`session.dispose()`** in the `finally` block — cleans up internal resources (event subscriptions, open handles).

That is a working coding-agent session in under twenty lines. But those defaults carry assumptions — credentials in `~/.xzy/`, sessions persisted to disk, the default built-in tools active — that may not suit every embedding context. Let's understand what we can change.

## Step 2 — Understanding the options

`createAgentSession` accepts a single options object of type `CreateAgentSessionOptions`. Every field is optional. Here is the full table:

| Option | Type | Default | What it controls |
|---|---|---|---|
| `cwd` | `string` | `process.cwd()` | Working directory for project-local discovery (skills, session files, settings) |
| `agentDir` | `string` | `~/.xzy/` | Global config directory for auth, models config, and fallback skills |
| `authStorage` | `AuthStorage` | `AuthStorage.create(agentDir/auth.json)` | Credential storage for API keys and OAuth tokens |
| `modelRegistry` | `ModelRegistry` | `ModelRegistry.create(authStorage, agentDir/models.json)` | Registry of available provider models |
| `model` | `Model<any>` | From settings, else first available | Which LLM model to use |
| `thinkingLevel` | `ThinkingLevel` | From settings, else `'medium'` (clamped to model capability) | Extended thinking budget: `'off'`, `'low'`, `'medium'`, `'high'` |
| `scopedModels` | `Array<{ model, thinkingLevel? }>` | — | Models available for cycling (Ctrl+P in interactive mode) |
| `noTools` | `'all' \| 'builtin'` | — | Suppress tool defaults: `'all'` disables every tool; `'builtin'` disables the four built-in tools but keeps extension/custom tools |
| `tools` | `string[]` | `["read", "bash", "edit", "write"]` | Explicit allowlist of tool names; overrides `noTools` when provided |
| `excludeTools` | `string[]` | — | Denylist of tool names; applied after `tools` when both are provided |
| `customTools` | `ToolDefinition[]` | `[]` | Additional tool definitions to register alongside built-in tools |
| `resourceLoader` | `ResourceLoader` | `new DefaultResourceLoader({ cwd, agentDir, settingsManager })` | Provides skills, extensions, prompts, themes, system prompt, and context files |
| `sessionManager` | `SessionManager` | `SessionManager.create(cwd, sessionDir)` | Manages conversation history persistence |
| `settingsManager` | `SettingsManager` | `SettingsManager.create(cwd, agentDir)` | Configuration knobs: compaction, retry, timeouts, defaults |
| `sessionStartEvent` | `SessionStartEvent` | — | Metadata passed to extension runtime at startup |

A few of these need more explanation — let's look at the patterns that come up most often, then see them all put together in the full-control example.

### Tool control

By default the session enables four built-in tools: `read`, `bash`, `edit`, `write`. The `tools` option replaces that list with an explicit allowlist:

```ts
// Read-only: agent can read and search but not modify files or run commands
const { session } = await createAgentSession({
  tools: ["read", "grep", "find", "ls"],
});
```

If you want to keep the default built-ins but block one specifically, use `excludeTools`:

```ts
const { session } = await createAgentSession({
  excludeTools: ["bash"], // keeps read, edit, write; removes bash
});
```

`noTools: "all"` starts with nothing enabled — useful when you provide only `customTools` and want no built-in tool active at all.

### The thinking level

`thinkingLevel` controls how much extended reasoning budget the model is given before it answers. Valid values are `'off'`, `'low'`, `'medium'`, and `'high'`. The value is automatically clamped to what the chosen model actually supports — so passing `'high'` with a model that only supports `'off'` silently becomes `'off'` rather than erroring.

<!-- GAP: The exact clamping logic (which models support which levels) is in `clampThinkingLevel` from `llm-toolkit`; the source is silent on the specific capability matrix for each provider model — needed for a precise capability table -->

## Step 3 — The full-control example

Now let's see what it looks like when you want to replace every default. The scenario is an embedded agent that must not touch the user's real credential files, must use a specific model, and should hold conversation history only in memory (useful in tests or stateless servers).

```ts
// Adapted from examples/sdk/12-full-control.ts
// Genericised: package names use "coding-agent" and "llm-toolkit"
import { getModel } from "llm-toolkit";
import {
  AuthStorage,
  createAgentSession,
  createExtensionRuntime,
  ModelRegistry,
  type ResourceLoader,
  SessionManager,
  SettingsManager,
} from "coding-agent";

// 1. Custom auth: isolated credential file, not ~/.xzy/auth.json
const authStorage = AuthStorage.create("/tmp/my-agent/auth.json");

// 2. Inject API key at runtime (not written to disk)
if (process.env.MY_ANTHROPIC_KEY) {
  authStorage.setRuntimeApiKey("anthropic", process.env.MY_ANTHROPIC_KEY);
}

// 3. In-memory model registry — no models.json file
const modelRegistry = ModelRegistry.inMemory(authStorage);

// 4. Explicit model selection
const model = getModel("anthropic", "claude-sonnet-4-20250514");
if (!model) throw new Error("Model not found");

// 5. In-memory settings with specific overrides
const settingsManager = SettingsManager.inMemory({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 2 },
});

const cwd = process.cwd();

// 6. Inline resource loader — no file-system discovery at all
const resourceLoader: ResourceLoader = {
  getExtensions: () => ({
    extensions: [],
    errors: [],
    runtime: createExtensionRuntime(),
  }),
  getSkills: () => ({ skills: [], diagnostics: [] }),
  getPrompts: () => ({ prompts: [], diagnostics: [] }),
  getThemes: () => ({ themes: [], diagnostics: [] }),
  getAgentsFiles: () => ({ agentsFiles: [] }),
  getSystemPrompt: () =>
    `You are a minimal assistant.\nAvailable: read, bash. Be concise.`,
  getAppendSystemPrompt: () => [],
  extendResources: () => {},
  reload: async () => {},
};

// 7. Assemble — everything explicit
const { session } = await createAgentSession({
  cwd,
  agentDir: "/tmp/my-agent",
  model,
  thinkingLevel: "off",
  authStorage,
  modelRegistry,
  resourceLoader,
  tools: ["read", "bash"],
  sessionManager: SessionManager.inMemory(cwd),
  settingsManager,
});

try {
  session.subscribe((event) => {
    if (
      event.type === "message_update" &&
      event.assistantMessageEvent.type === "text_delta"
    ) {
      process.stdout.write(event.assistantMessageEvent.delta);
    }
  });

  await session.prompt("List files in the current directory.");
  console.log();
} finally {
  session.dispose();
}
```

Let's walk through what each decision buys you:

**Steps 1–2: Isolated auth.** `AuthStorage.create("/tmp/my-agent/auth.json")` points credentials at a path your application owns, not the user's home directory. `setRuntimeApiKey` injects a key from an environment variable without persisting it — no credentials written to disk.

**Step 3: In-memory model registry.** `ModelRegistry.inMemory(authStorage)` creates a registry that holds its model list in RAM, consulting `authStorage` for API keys. Useful when you do not want to read or write `~/.xzy/models.json`.

**Step 4: Explicit model.** `getModel("anthropic", "claude-sonnet-4-20250514")` returns the named model descriptor from the `llm-toolkit` package's built-in catalogue. You always know exactly which model is running — no fallback discovery.

**Step 5: In-memory settings.** `SettingsManager.inMemory({ … })` creates settings entirely in RAM, starting from the overrides you provide. Here we disable compaction (the process that summarises long conversations to save context) and cap retries at 2.

**Step 6: Inline resource loader.** This is the most complete escape hatch — it replaces all resource discovery with values you supply directly. Instead of creating a `DefaultResourceLoader` that scans the file system for skills, extensions, and context files, we provide a plain object that implements the `ResourceLoader` interface. Every method returns an empty result, and `getSystemPrompt` returns a hard-coded string. The session will have exactly the system prompt we specify and no discovered resources.

**Step 7: In-memory session.** `SessionManager.inMemory(cwd)` keeps the conversation history only in RAM — when the process exits, nothing is written to disk. Ideal for stateless servers or test fixtures.

### What changes between minimal and full-control

| Aspect | Minimal | Full control |
|---|---|---|
| Auth file | `~/.xzy/auth.json` | `/tmp/my-agent/auth.json` |
| Model | From settings / first available | Explicit: `claude-sonnet-4-20250514` |
| Model registry | Reads `~/.xzy/models.json` | In-memory: `ModelRegistry.inMemory` |
| Settings | Reads disk settings | In-memory: `SettingsManager.inMemory` |
| Skills & prompts | Scanned from `cwd` and `~/.xzy/` | Empty: inline `ResourceLoader` |
| Session history | Persisted to disk | In-memory: `SessionManager.inMemory` |
| Tools | `read`, `bash`, `edit`, `write` | `read`, `bash` only |
| `thinkingLevel` | From settings / `'medium'` | `'off'` |

## Step 4 — Subscribing to events

After `createAgentSession` returns, the `session` object emits events as the agent works. You subscribe with `session.subscribe(callback)`. The callback receives a typed event union; a minimal switch over the most common event types:

```ts
session.subscribe((event) => {
  switch (event.type) {
    case "message_update":
      // Streamed text from the assistant
      if (event.assistantMessageEvent.type === "text_delta") {
        process.stdout.write(event.assistantMessageEvent.delta);
      }
      break;
    case "tool_execution_start":
      // Agent is about to run a tool
      console.log(`Tool: ${event.toolName}`);
      break;
    case "tool_execution_end":
      // Tool finished, result available
      console.log(`Result: ${event.result}`);
      break;
    case "agent_end":
      // The full turn is complete
      console.log("Done");
      break;
  }
});
```

Call `subscribe` before `session.prompt(…)` — events fire during the prompt call, so a subscription registered after the fact will miss early events.

<!-- GAP: The full set of event type discriminants beyond the four shown here is not enumerated in the assigned sources; source S74 lists only these four examples — needed to document every event the session can emit -->

## Putting it all together

The decision tree for choosing your configuration:

1. **Exploring or prototyping?** Call `createAgentSession()` with no arguments. Everything auto-discovers from the current directory and `~/.xzy/`.
2. **Need a specific model?** Pass `model: getModel("anthropic", "…")` and optionally `thinkingLevel`.
3. **Want read-only access?** Pass `tools: ["read", "grep", "find", "ls"]` to restrict the tool set.
4. **Need isolated credentials?** Pass `authStorage: AuthStorage.create("/your/path/auth.json")` and inject keys with `authStorage.setRuntimeApiKey`.
5. **Stateless or testing context?** Pass `sessionManager: SessionManager.inMemory(cwd)` to keep history in RAM.
6. **Fully embedded, no discovery?** Implement the `ResourceLoader` interface inline (as in the full-control example) and pass every manager explicitly.

In each case, `createAgentSession` does the same work: it wires the managers together, resolves the initial model and thinking level, registers the tools, restores any previous session state, and hands back a ready `AgentSession`. You call `session.prompt(…)` to run a turn, observe events via `session.subscribe`, and call `session.dispose()` to clean up when you are done.

The next chapter covers how the package manager and configuration migrations work underneath these settings — particularly how the agent handles upgrading persisted state across versions.

---

← Previous: [The CLI Entry Point: Argument Parsing and Mode Selection](./cli-entry-point.md) · Next: [Package Management and Config Migrations](./package-manager-and-migrations.md) →
