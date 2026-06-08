---
title: "The Agent Class: State Machine and Lifecycle"
description: "Wrap the agent loop in the Agent class — adding the state machine, event subscribers, steering and follow-up queues, abort control, and idle-wait coordination."
category: agent-core
type: tutorial
tags: [Agent class, state machine, subscriber, steering, follow-up queue, abort, waitForIdle, lifecycle, event dispatch, agent-core]
keywords: [Agent class, state machine, subscriber pattern, steering queue, abort coordination]
sources: [S20]
---

**TL;DR** — The raw agent loop is a function. The `Agent` class wraps it in a state machine with lifecycle management: you can `prompt()` it (which queues messages and starts the loop if idle), `abort()` it mid-turn, subscribe to events, steer its behavior with follow-up instructions, and `waitForIdle()` to coordinate with async workflows. We'll build each of these capabilities.

## The Agent class skeleton

Create `packages/agent-core/src/agent.ts`:

```ts
export class Agent {
  private state: AgentState = "idle";
  private context: AgentContext;
  private config: AgentLoopConfig;
  private subscribers = new Set<(event: AgentEvent) => void>();
  private messageQueue: AgentMessage[] = [];
  private steeringQueue: AgentMessage[] = [];
  private abortController: AbortController | null = null;
  private idleResolvers: Array<() => void> = [];
  private currentTurn: Promise<void> | null = null;
}
```

## The state machine

Three states, clear transitions:

```
     prompt()
  ┌─────────────┐
  │             ▼
  │    ┌──────────────┐
  │    │    idle       │◀────── turn completes ────┐
  │    └──────────────┘                           │
  │         │                                      │
  │         │ prompt()                             │
  │         ▼                                      │
  │    ┌──────────────┐     abort()         ┌──────────────┐
  │    │   running     │───────────────────▶│  stopping     │
  │    └──────────────┘                    └──────────────┘
  │         │                                      │
  │         │ turn completes naturally              │
  └─────────┘                                      │
                                                   │
                          turn completes (aborted) ─┘
```

The state check guards every operation:

```ts
prompt(messages: AgentMessage[]): void {
  if (this.state === "stopping") {
    // Queue the messages for after the current turn stops
    this.messageQueue.push(...messages);
    return;
  }

  this.messageQueue.push(...messages);
  if (this.state === "idle") {
    this.startLoop();
  }
  // If already running, messages will be picked up at the next
  // drain point — no action needed
}
```

## Prompting: adding messages to the queue

The `prompt()` method is the primary way to interact with the agent. It accepts one or more `AgentMessage` objects and either starts the loop (if idle) or queues them (if running):

```ts
prompt(...messages: AgentMessage[]): void {
  this.messageQueue.push(...messages);
  if (this.state === "idle") {
    this.startLoop();
  }
}
```

Multiple calls to `prompt()` while the agent is running all get queued. At the next turn boundary, the loop drains the queue according to the configured `QueueMode`.

## Steering: injecting instructions mid-turn

Sometimes you want to give the agent an instruction without it being a full user message. The `steer()` method injects a high-priority message that the loop picks up at the next opportunity:

```ts
steer(message: AgentMessage): void {
  this.steeringQueue.push(message);
}
```

Steering messages are injected before queued user messages. They're used for things like "stop what you're doing and focus on this instead" or "update your understanding of the project structure."

## Event subscribers

Any part of the system can observe agent activity by subscribing:

```ts
subscribe(fn: (event: AgentEvent) => void): () => void {
  this.subscribers.add(fn);
  return () => this.subscribers.delete(fn);
}

private emit(event: AgentEvent): void {
  for (const sub of this.subscribers) {
    try { sub(event); } catch { /* isolate subscriber errors */ }
  }
}
```

The `subscribe()` method returns an unsubscribe function — clean, leak-proof subscription management. Events include `turn_start`, `assistant_message`, `tool_execution_start`, `tool_execution_end`, `error`, and `turn_end`. The UI uses these to render streaming content and tool execution status. Extensions use them to trigger custom behavior.

## Abort control

The `abort()` method gracefully stops the current turn:

```ts
abort(): void {
  if (this.state !== "running") return;
  this.state = "stopping";
  this.abortController?.abort();
  // The loop will detect the abort and finish the current turn
  // with stopReason: "aborted"
}
```

Abort is cooperative, not forceful. The loop checks `signal.aborted` at turn boundaries and stops. Any in-flight HTTP request is cancelled via the `AbortSignal`, but tool execution continues until the current tool finishes (we don't want to leave the system in an inconsistent state).

## Waiting for idle

Async workflows need to know when the agent is done:

```ts
async waitForIdle(): Promise<void> {
  if (this.state === "idle") return;
  return new Promise(resolve => {
    this.idleResolvers.push(resolve);
  });
}

private transitionTo(newState: AgentState): void {
  this.state = newState;
  if (newState === "idle") {
    // Resolve all waiters
    for (const resolve of this.idleResolvers) resolve();
    this.idleResolvers = [];
    // If more messages were queued while we were running, start again
    if (this.messageQueue.length > 0 || this.steeringQueue.length > 0) {
      this.startLoop();
    }
  }
}
```

This is how tests wait for the agent to finish processing before making assertions.

## The loop starter

```ts
private startLoop(): void {
  this.state = "running";
  this.abortController = new AbortController();

  this.currentTurn = runAgentLoop(
    [], // messages come from the queue, not here
    this.context,
    this.config,
    (event) => this.emit(event),
    this.abortController.signal,
    this.config.streamFn,
  ).then(() => {
    this.transitionTo("idle");
  }).catch((err) => {
    this.emit({ type: "error", error: err });
    this.transitionTo("idle");
  });
}
```

## Using the Agent

Here's how the pieces fit together from the consumer's perspective:

```ts
const agent = new Agent({
  systemPrompt: "You are a helpful coding assistant.",
  tools: [readFileTool, writeFileTool, bashTool],
  model: getModel("claude-sonnet-4-6")!,
  queueMode: "all",
});

// Subscribe to streaming events
agent.subscribe((event) => {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta); // stream to terminal
  }
});

// Start a task
agent.prompt({
  role: "user",
  content: "Find the bug in src/auth.ts and fix it.",
  timestamp: Date.now(),
});

// Wait for completion
await agent.waitForIdle();
```

## What we've built

The `Agent` class wraps the raw loop in a clean, event-driven API:

- **State machine** (`idle → running → stopping → idle`) with guarded transitions
- **`prompt()`** — queue messages and start the loop if idle
- **`steer()`** — inject high-priority instructions
- **`subscribe()`** — observe events with leak-proof unsubscription
- **`abort()`** — cooperatively stop the current turn
- **`waitForIdle()`** — coordinate with async workflows

In the next chapter, we'll build the Agent Harness — the layer that adds compaction, session persistence, system prompt construction, and skill loading on top of the Agent class.

---

← Previous: [The Agent Loop](./the-agent-loop.md) · Next: [The Agent Harness: Compaction, Sessions, and Skills](./harness-compaction-sessions.md) →
