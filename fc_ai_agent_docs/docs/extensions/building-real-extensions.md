---
title: "Building Real Extensions: Tools, State, Commands, and Hooks"
description: "Build progressively richer extensions — a security permission gate, a todo tracker with custom tool and TUI renderer, a custom compaction strategy, and an EventBus for extension-to-extension communication."
category: extensions
type: tutorial
tags: [extension, permission gate, tool hook, state, slash command, renderer, session_start, custom compaction, event bus, pub-sub, dynamic resources, extensions, examples]
keywords: [security extension, todo extension, permission gate, event bus, custom compaction extension, TUI renderer]
sources: [S50]
---

**TL;DR** — We'll build three real extensions that demonstrate the full power of the Extension API: a **security gate** that blocks dangerous bash commands, a **todo tracker** with a custom tool + slash command + TUI renderer, and an **EventBus** that lets extensions communicate through pub/sub. Each builds on the previous one, showing progressively more advanced patterns.

## Extension 1: Security permission gate

The security gate intercepts bash commands and blocks dangerous patterns:

```ts
// security-gate.ts
import type { ExtensionAPI } from "coding-agent/hooks";

const DANGEROUS_PATTERNS = [
  /rm\s+-rf\s+\//,                          // rm -rf /
  /sudo\s+rm/,                               // sudo rm
  />\s*\/dev\/sd[a-z]/,                      // overwrite disk
  /curl.*\|\s*(ba)?sh/,                      // curl pipe bash
  /chmod\s+777/,                             // world-writable
  /git\s+push\s+--force.*origin\s+main/,    // force push main
];

export function activate(api: ExtensionAPI): void {
  const blocked: Array<{ command: string; reason: string; timestamp: number }> = [];

  api.on("PreToolUse", (ctx) => {
    if (ctx.toolName !== "bash") return;

    const command = ctx.args.command as string;
    for (const pattern of DANGEROUS_PATTERNS) {
      if (pattern.test(command)) {
        blocked.push({ command, reason: pattern.toString(), timestamp: Date.now() });
        return {
          block: true,
          reason: `Security gate blocked: ${command}\nPattern: ${pattern}`,
        };
      }
    }
  });

  api.registerCommand({
    name: "security-log",
    description: "Show blocked commands",
    handler: async () => {
      if (blocked.length === 0) return "No commands blocked this session.";
      return blocked.map(b =>
        `[${new Date(b.timestamp).toISOString()}] ${b.command}`
      ).join("\n");
    },
  });

  api.log("info", `Security gate active — ${DANGEROUS_PATTERNS.length} patterns`);
}
```

## Extension 2: Todo tracker

A full-featured extension with custom tool, slash command, state persistence, and a TUI renderer:

```ts
// todo-tracker.ts
import type { ExtensionAPI } from "coding-agent/hooks";

interface Todo {
  id: string;
  text: string;
  done: boolean;
  createdAt: number;
}

export function activate(api: ExtensionAPI): void {
  // Initialize state
  const state = api.getState().data;
  state.todos = state.todos ?? [];

  // Register the todo tool — the agent can use it
  api.registerTool({
    name: "todo",
    description: "Add, complete, or list todo items.",
    parameters: Type.Object({
      action: Type.Union([Type.Literal("add"), Type.Literal("done"), Type.Literal("list")]),
      text: Type.Optional(Type.String()),
      id: Type.Optional(Type.String()),
    }),

    execute: async (args) => {
      const todos = state.todos as Todo[];

      switch (args.action) {
        case "add": {
          const todo: Todo = {
            id: String(todos.length + 1),
            text: args.text ?? "Untitled",
            done: false,
            createdAt: Date.now(),
          };
          todos.push(todo);
          api.setState({ data: { todos } });
          return {
            content: [{ type: "text", text: `Added todo #${todo.id}: ${todo.text}` }],
            isError: false,
          };
        }

        case "done": {
          const todo = todos.find(t => t.id === args.id);
          if (!todo) return { content: [{ type: "text", text: `Todo #${args.id} not found` }], isError: true };
          todo.done = true;
          api.setState({ data: { todos } });
          return {
            content: [{ type: "text", text: `Completed todo #${todo.id}: ${todo.text}` }],
            isError: false,
          };
        }

        case "list": {
          if (todos.length === 0) return { content: [{ type: "text", text: "No todos yet." }], isError: false };
          const lines = todos.map(t =>
            `${t.done ? "✅" : "⬜"} #${t.id}: ${t.text}`
          );
          return { content: [{ type: "text", text: lines.join("\n") }], isError: false };
        }
      }
    },
  });

  // Register slash command — the user can use it directly
  api.registerCommand({
    name: "todos",
    description: "Show todo list",
    handler: async () => {
      const todos = state.todos as Todo[];
      if (todos.length === 0) return "No todos yet.";
      return todos.map(t => `${t.done ? "✅" : "⬜"} #${t.id}: ${t.text}`).join("\n");
    },
  });

  // Register a TUI renderer for the todo list
  api.registerRenderer("todos", () => {
    const todos = state.todos as Todo[];
    const pending = todos.filter(t => !t.done).length;
    return `📋 Todos: ${pending}/${todos.length} remaining`;
  });

  // Auto-persist todos on session end
  api.on("SessionEnd", () => {
    api.log("info", `Session ended with ${state.todos.length} todos`);
  });

  api.log("info", "Todo tracker activated");
}
```

The agent can now use the `todo` tool to track tasks during a coding session. The user can type `/todos` to see the list. The status bar shows pending count.

## Extension 3: EventBus for extension-to-extension communication

Extensions are isolated by default. The EventBus lets them communicate through typed pub/sub:

```ts
// event-bus.ts
type EventHandler = (data: unknown) => void;

class EventBus {
  private channels = new Map<string, Set<EventHandler>>();

  publish(channel: string, data: unknown): void {
    const handlers = this.channels.get(channel);
    if (!handlers) return;
    for (const handler of handlers) {
      try { handler(data); } catch { /* isolate */ }
    }
  }

  subscribe(channel: string, handler: EventHandler): () => void {
    const handlers = this.channels.get(channel) ?? new Set();
    handlers.add(handler);
    this.channels.set(channel, handlers);
    return () => handlers.delete(handler);
  }
}

// Exposed via ExtensionAPI:
// api.eventBus.publish("todo:added", { id: "1", text: "Fix auth bug" });
// api.eventBus.subscribe("todo:added", (data) => { ... });
```

Use case: the security gate publishes `"command:blocked"` events, and a notification extension subscribes to show desktop notifications for blocked commands.

## What we've built

Three real extensions demonstrating the full API surface:
- **Security gate** — PreToolUse hooks, pattern matching, command logging
- **Todo tracker** — custom tool, slash command, state persistence, TUI renderer
- **EventBus** — inter-extension pub/sub with error isolation

---

← Previous: [Loading and Running Extensions](./extension-loader.md) · Next: [Multi-Agent Orchestration: Fan-Out with Subagents](./multi-agent-orchestration.md) →
