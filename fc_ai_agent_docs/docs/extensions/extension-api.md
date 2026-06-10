---
title: "The Extension API: Handlers, Context, and Events"
description: "Design the Extension API surface — the context object, handler registration for all hook types, ToolDefinition for custom tools, and registerProvider for custom LLM backends."
category: extensions
type: tutorial
tags: [ExtensionAPI, extension, handler, context, ToolDefinition, event types, registerTool, registerProvider, session_start, tool_call, extensions, plugin system]
keywords: [extension API, plugin system, custom tools, custom providers, event handlers]
sources: [S50]
---

**TL;DR** — The hooks system gives us event interception. The Extension API wraps it in a first-class plugin interface: extensions are self-contained modules that register tools, providers, slash commands, and hook handlers through a typed `ExtensionAPI` object. We'll design the API surface and build the minimal "hello world" extension to establish the pattern.

## The ExtensionAPI interface

Create `packages/coding-agent/src/core/hooks/extension-api.ts`. The `ExtensionAPI` is what every extension receives:

```ts
interface ExtensionAPI {
  // Metadata
  name: string;
  version: string;

  // Tool registration
  registerTool(tool: AgentTool): void;

  // Provider registration
  registerProvider(provider: ModelProvider): void;

  // Slash command registration
  registerCommand(command: SlashCommand): void;

  // Hook registration
  on<K extends HookType>(hook: K, handler: HookHandler<K>): () => void;

  // State
  getState(): ExtensionState;
  setState(state: Partial<ExtensionState>): void;

  // Session access (read-only)
  getSession(): SessionInfo;

  // Subscriptions to agent events
  subscribe(fn: (event: AgentEvent) => void): () => void;

  // Logging
  log(level: "info" | "warn" | "error", message: string): void;
}
```

## Extension structure

An extension is a TypeScript module that exports an `activate` function:

```ts
// my-extension.ts
import type { ExtensionAPI } from "coding-agent/hooks";

export function activate(api: ExtensionAPI): void {
  // Register a custom tool
  api.registerTool({
    name: "weather",
    description: "Get the current weather for a city",
    parameters: Type.Object({
      city: Type.String(),
    }),
    execute: async (args) => {
      const weather = await fetchWeather(args.city);
      return {
        content: [{ type: "text", text: `${args.city}: ${weather.temp}°C, ${weather.conditions}` }],
        isError: false,
      };
    },
  });

  // Register a hook
  api.on("PreToolUse", (ctx) => {
    if (ctx.toolName === "bash" && ctx.args.command.includes("production")) {
      return { block: true, reason: "Production commands require confirmation" };
    }
  });

  // Register a slash command
  api.registerCommand({
    name: "weather",
    description: "Check the weather",
    handler: async (args) => {
      const weather = await fetchWeather(args.trim() || "London");
      return `${args.trim() || "London"}: ${weather.temp}°C`;
    },
  });

  api.log("info", "Weather extension activated");
}

export function deactivate(): void {
  // Cleanup: close connections, stop timers, etc.
}
```

## The hello-world extension

The simplest possible extension — just to establish the pattern:

```ts
// hello-world.ts
import type { ExtensionAPI } from "coding-agent/hooks";

export function activate(api: ExtensionAPI): void {
  api.registerCommand({
    name: "hello",
    description: "Say hello",
    handler: async () => `Hello from ${api.name} v${api.version}!`,
  });

  api.on("SessionStart", (ctx) => {
    api.log("info", `Session ${ctx.sessionId} started in ${ctx.cwd}`);
  });
}
```

When the user types `/hello`, the agent responds with the extension's name and version.

## Extension context

Each extension gets its own isolated `ExtensionState`:

```ts
interface ExtensionState {
  data: Record<string, unknown>;
}

// In the extension:
const counters = api.getState().data;
counters.toolCalls = (counters.toolCalls as number ?? 0) + 1;
api.setState({ data: counters });
```

State is scoped to the extension — one extension can't read another's state. This prevents accidental coupling while allowing extensions to maintain their own data.

## Extension lifecycle

```
load → activate(api) → [runs for session duration] → deactivate()
```

Extensions are activated when the session starts and deactivated when it ends. The `deactivate()` export is optional — use it for cleanup (closing network connections, stopping timers, persisting state).

## What we've built

- **ExtensionAPI** — typed surface for registering tools, providers, commands, and hooks
- **Extension structure** — `activate(api)` + optional `deactivate()`
- **Isolated state** — per-extension data scoping
- **Hello-world example** — the minimal pattern every extension follows

---

← Previous: [The CLI Entry Point](../coding-agent/cli-entry-point.md) · Next: [Loading and Running Extensions](./extension-loader.md) →
