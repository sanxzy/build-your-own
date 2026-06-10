---
title: "The Hooks System: Intercepting Every Agent Event"
description: "Build the hooks system — the event pipeline where extensions can intercept PreToolUse, PostToolUse, SessionStart, SessionEnd, Stop, UserPromptSubmit, PreCompact, and Notification events."
category: coding-agent
type: tutorial
tags: [hooks, PreToolUse, PostToolUse, SessionStart, SessionEnd, Stop, UserPromptSubmit, PreCompact, Notification, event interception, lifecycle, coding-agent]
keywords: [hook system, event pipeline, tool interception, session hooks, extension events]
sources: [S50]
---

**TL;DR** — Extensions need to hook into the agent's lifecycle. We'll build an event pipeline with eight hook types: **PreToolUse** (block or modify tool calls before execution), **PostToolUse** (modify results after execution), **SessionStart** (initialize on session creation), **SessionEnd** (clean up), **Stop** (handle shutdown), **UserPromptSubmit** (transform user input), **PreCompact** (customize compaction), and **Notification** (respond to external events). Each hook has a typed context and return value.

## The hook architecture

Create `packages/coding-agent/src/core/hooks/index.ts`. Hooks are registered by name and called in order:

```ts
type HookHandler<Context, Result> = (ctx: Context) => Result | Promise<Result>;

class HookSystem {
  private hooks = new Map<string, HookHandler<any, any>[]>();

  register<C, R>(name: string, handler: HookHandler<C, R>): () => void {
    const handlers = this.hooks.get(name) ?? [];
    handlers.push(handler);
    this.hooks.set(name, handlers);
    return () => {
      const idx = handlers.indexOf(handler);
      if (idx >= 0) handlers.splice(idx, 1);
    };
  }

  async run<C, R>(name: string, context: C, defaultResult: R): Promise<R> {
    const handlers = this.hooks.get(name) ?? [];
    let result = defaultResult;

    for (const handler of handlers) {
      try {
        const handlerResult = await handler(context);
        if (handlerResult !== undefined) {
          result = { ...result, ...handlerResult }; // merge semantics
        }
      } catch (err) {
        console.error(`Hook ${name} handler error:`, err);
        // Isolate hook errors — never crash the agent
      }
    }

    return result;
  }
}
```

Key design decisions:
- **Merge semantics** — multiple handlers for the same hook merge their results (the last handler to set a field wins).
- **Error isolation** — a broken hook handler never crashes the agent loop.
- **Unsubscribe pattern** — `register()` returns a cleanup function.

## The eight hook types

### PreToolUse

Called before every tool execution. Handlers can block the tool or modify its arguments:

```ts
interface PreToolUseContext {
  toolName: string;
  toolCallId: string;
  args: unknown;
  assistantMessage: AssistantMessage;
  agentContext: AgentContext;
}

interface PreToolUseResult {
  block?: boolean;
  reason?: string;
  modifiedArgs?: unknown;
}
```

Use case: a security extension that blocks `rm -rf /` or `curl | bash`:

```ts
hooks.register("PreToolUse", (ctx: PreToolUseContext) => {
  if (ctx.toolName === "bash" && isDangerousCommand(ctx.args.command)) {
    return { block: true, reason: "Dangerous command blocked by security extension" };
  }
});
```

### PostToolUse

Called after every tool execution. Handlers can modify results, override the error flag, or signal early termination:

```ts
interface PostToolUseContext {
  toolName: string;
  toolCallId: string;
  result: AgentToolResult;
  assistantMessage: AssistantMessage;
  agentContext: AgentContext;
}

interface PostToolUseResult {
  content?: (TextContent | ImageContent)[];
  details?: unknown;
  isError?: boolean;
  terminate?: boolean;
}
```

Use case: an extension that post-processes bash output to redact secrets.

### SessionStart / SessionEnd

Called when a session is created or destroyed:

```ts
interface SessionStartContext {
  sessionId: string;
  cwd: string;
  modelId: string;
}

interface SessionEndContext {
  sessionId: string;
  messageCount: number;
  totalCost: number;
}
```

Use case: logging extensions that track session metrics.

### Stop

Called when the agent process receives SIGTERM/SIGINT. Handlers can delay shutdown for cleanup:

```ts
interface StopContext {
  signal: string;
  sessionId?: string;
}
```

### UserPromptSubmit

Called when the user submits a message. Handlers can transform the prompt:

```ts
interface UserPromptSubmitContext {
  prompt: string;
  sessionId?: string;
}

interface UserPromptSubmitResult {
  prompt?: string;       // transformed prompt
  additionalContext?: string;  // injected context
}
```

### PreCompact

Called before context compaction. Handlers can customize the compaction strategy:

```ts
interface PreCompactContext {
  messages: AgentMessage[];
  estimatedTokens: number;
  contextWindow: number;
}
```

### Notification

Called for external events (file changes, network events, timer expirations):

```ts
interface NotificationContext {
  type: string;
  data: unknown;
}
```

## Using hooks in AgentSession

The `AgentSession` wires hooks into the agent lifecycle:

```ts
class AgentSession {
  private hookSystem = new HookSystem();

  constructor() {
    this.harness.agent.subscribe(async (event) => {
      if (event.type === "tool_execution_start") {
        const hookResult = await this.hookSystem.run("PreToolUse", {
          toolName: event.toolCall.name,
          toolCallId: event.toolCall.id,
          args: event.args,
          assistantMessage: event.assistantMessage,
          agentContext: this.harness.agent.context,
        }, {});
        if (hookResult.block) { /* prevent execution */ }
        if (hookResult.modifiedArgs) { /* use modified args */ }
      }
    });
  }
}
```

## What we've built

The hooks system is the extensibility backbone. It gives extensions a typed, safe way to intercept every significant agent event without monkey-patching or forking.

---

← Previous: [Built-In Coding Tools](./built-in-tools.md) · Next: [System Prompt Construction and Skill Loading](./system-prompt-and-skills.md) →
