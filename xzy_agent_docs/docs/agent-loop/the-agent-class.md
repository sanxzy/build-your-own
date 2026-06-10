---
title: "The Agent Class: State Machine, Steering, and Lifecycle"
description: "How the Agent class wraps agentLoop to add stateful lifecycle management, subscriber dispatch, steering and follow-up queues, abort control, and idle-wait coordination."
category: agent-loop
type: tutorial
tags: [Agent class, state machine, subscriber, steering, follow-up queue, abort, waitForIdle, agent-core, lifecycle, event dispatch, QueueMode, PendingMessageQueue, AgentOptions, AgentState, prompt, continue, reset, activeRun, AbortController]
keywords: [stateful agent wrapper, agent lifecycle, event subscription, queue management, abort signal, idle coordination, agent concurrency, turn management, message queue drain]
sources: [S28]
---

**TL;DR** — `agentLoop` is a great mechanism for running one turn-by-turn conversation, but a real application needs more: it needs to observe every event, inject messages mid-run, queue work for later, and cancel a run in progress. The `Agent` class wraps `agentLoop` in a stateful object that provides all of these. By the end of this chapter you will understand the complete lifecycle of an `Agent` instance — from construction through subscriber wiring, prompt dispatch, mid-run steering, follow-up queuing, abort, and idle-wait.

# The Agent Class: State Machine, Steering, and Lifecycle

## Starting Point: What agentLoop Does Not Give You

The previous chapter introduced `agentLoop` and `agentLoopContinue` — the functions that drive one conversation round-trip with the model and execute any tool calls that come back (see [The Agent Loop: Turn-by-Turn LLM and Tool Execution](./the-agent-loop.md) for a full walkthrough). Those functions emit events while they run and resolve when the agent decides it is done.

That is the mechanism. But notice what `agentLoop` on its own cannot do:

- It does not hold the message history between calls. You pass it a transcript; it gives back events. You must reassemble the state.
- It does not let you react to events without polling. You pass a callback — but nothing coordinates multiple subscribers.
- It does not let you inject a message while the agent is mid-turn. You must wait until the function returns.
- It does not offer a clean abort or an easy way to know when everything has settled.

A single utility function for a single run is right for the job of running turns. A class is right for the job of managing ongoing agent state. That is exactly what `Agent` does.

Let's build up an understanding of the class piece by piece, starting with its state and working outward to the APIs that callers actually use.

---

## The Internal State

The `Agent` class stores all mutable runtime data in a private `_state` object of type `MutableAgentState`. Think of this as the agent's working memory — the single source of truth that all other APIs read and write.

```ts
// Simplified view of what _state holds
{
  systemPrompt: string;
  model: Model;
  thinkingLevel: "off" | ...;
  tools: AgentTool<any>[];
  messages: AgentMessage[];          // the growing transcript
  isStreaming: boolean;              // true while a run is in progress
  streamingMessage?: AgentMessage;  // the message currently being assembled
  pendingToolCalls: Set<string>;    // tool call IDs not yet settled
  errorMessage?: string;            // last error, if any
}
```

Two properties — `tools` and `messages` — are defensive-copied on every assignment. That means callers who hold a reference to the old array will not see the agent mutate it underneath them. This small detail matters once you have subscribers that read state from inside event handlers.

The `AgentState` type that the public `state` getter returns omits the mutable internals (`isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage` are re-exposed as read-only). The getter is a simple pass-through to `_state`:

```ts
get state(): AgentState {
  return this._state;
}
```

You can read `agent.state.messages` at any time; you cannot swap out the array directly.

---

## Construction: Wiring Options to Fields

You create an `Agent` by passing an `AgentOptions` object (all fields optional):

```ts
const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful coding assistant.",
    model: myModel,
    tools: [readFileTool, writeFileTool],
  },
  steeringMode: "one-at-a-time",  // default
  followUpMode: "one-at-a-time",  // default
  transport: "auto",              // default
  toolExecution: "parallel",      // default
});
```

The constructor assigns each option to a corresponding public field (e.g. `this.streamFn`, `this.beforeToolCall`) or creates a private queue object. If you omit an option the constructor picks a sensible default:

| Option | Default |
|---|---|
| `convertToLlm` | Filters to `user`, `assistant`, and `toolResult` messages |
| `streamFn` | `streamSimple` from `llm-toolkit` |
| `steeringMode` | `"one-at-a-time"` |
| `followUpMode` | `"one-at-a-time"` |
| `transport` | `"auto"` |
| `toolExecution` | `"parallel"` |

The full option set is wider — including `getApiKey`, `onPayload`, `onResponse`, `beforeToolCall`, `afterToolCall`, `prepareNextTurn`, `sessionId`, `thinkingBudgets`, `maxRetryDelayMs`, and `transformContext`. We will encounter most of these in context as we walk through the APIs below.

---

## Subscribers: Watching Every Event

The first thing a caller typically does after constructing an `Agent` is attach one or more subscribers. A subscriber is a function that receives every `AgentEvent` the underlying loop emits:

```ts
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === "message_end") {
    console.log("New message:", event.message.content);
  }
  if (event.type === "tool_execution_start") {
    console.log("Running tool:", event.toolCallId);
  }
});

// Later, remove the listener:
unsubscribe();
```

`subscribe` returns an unsubscribe function. Internally the class holds a `Set` of listeners:

```ts
private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void> | void>();

subscribe(listener: ...): () => void {
  this.listeners.add(listener);
  return () => this.listeners.delete(listener);
}
```

Two things are important about how listeners are invoked. First, they receive the active `AbortSignal` for the current run — so if the run is cancelled, your listener knows immediately and can bail out of any long work it is doing. Second, listener promises are **awaited in subscription order**: the class waits for each listener to settle before moving on. This means `agent_end` — the final event — is not considered fully settled until every listener for it has resolved.

This has a direct consequence for `waitForIdle()`, which we will see shortly.

---

## The Lifecycle: activeRun and runWithLifecycle

Every call to `prompt()` or `continue()` eventually reaches a private method called `runWithLifecycle`. This is the state machine. Let's trace what it does:

```ts
// Simplified view of runWithLifecycle
private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
  // 1. Guard: reject concurrent runs
  if (this.activeRun) {
    throw new Error("Agent is already processing.");
  }

  // 2. Create the AbortController and a manually-resolved Promise
  const abortController = new AbortController();
  let resolvePromise = () => {};
  const promise = new Promise<void>((resolve) => {
    resolvePromise = resolve;
  });
  this.activeRun = { promise, resolve: resolvePromise, abortController };

  // 3. Mark as running
  this._state.isStreaming = true;
  this._state.streamingMessage = undefined;
  this._state.errorMessage = undefined;

  try {
    await executor(abortController.signal);
  } catch (error) {
    await this.handleRunFailure(error, abortController.signal.aborted);
  } finally {
    this.finishRun();
  }
}
```

The `activeRun` object is the heartbeat of the class. While it is set, the agent is busy. It carries three things:

| Field | Purpose |
|---|---|
| `promise` | Resolved only after `finishRun()` clears state — this is what `waitForIdle()` returns |
| `resolve` | Called by `finishRun()` to unblock waiters |
| `abortController` | Used by `abort()` to cancel the run |

The pattern of storing a manually-resolved promise is worth noticing: `waitForIdle()` returns `this.activeRun?.promise ?? Promise.resolve()`. If nothing is running, it resolves immediately. If a run is in progress, the caller awaits the promise and wakes up only after `finishRun()` has fully torn down the active run — which happens *after* all `agent_end` listeners settle.

---

## Sending a Prompt

Once you have an agent and at least one subscriber wired, you send messages with `prompt()`:

```ts
// Three equivalent ways to start a run:
await agent.prompt("List all .ts files in src/");
await agent.prompt({ role: "user", content: [...], timestamp: Date.now() });
await agent.prompt([message1, message2]);
```

The overloads all normalise to an `AgentMessage[]` and then call `runPromptMessages`, which calls `runAgentLoop` with a snapshot of the current state:

```ts
private async runPromptMessages(messages: AgentMessage[], options = {}): Promise<void> {
  await this.runWithLifecycle(async (signal) => {
    await runAgentLoop(
      messages,
      this.createContextSnapshot(),   // systemPrompt + messages + tools, frozen at call time
      this.createLoopConfig(options), // model, hooks, queue drain functions
      (event) => this.processEvents(event),
      signal,
      this.streamFn,
    );
  });
}
```

Notice `createContextSnapshot()` — it copies the current messages and tools arrays at the moment the run starts, so mutations during the run do not affect the loop's view of the transcript mid-flight.

If you call `prompt()` while a run is already active, the class throws immediately:

```
"Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion."
```

That error message is itself the documentation for what to do next. Let's look at those two alternatives.

---

## Steering: Injecting Messages Mid-Run

Suppose the agent is mid-turn — it called a tool, is waiting for results, and you want to tell it something new before it continues. You do not want to wait; you want to inject a message that will be picked up as soon as the current assistant turn finishes.

That is what the **steering queue** is for:

```ts
agent.steer({
  role: "user",
  content: [{ type: "text", text: "Actually, focus only on TypeScript files." }],
  timestamp: Date.now(),
});
```

`steer()` pushes the message into a `PendingMessageQueue` held by the class. The queue is polled between turns via the `getSteeringMessages` callback wired into `AgentLoopConfig`:

```ts
getSteeringMessages: async () => {
  if (skipInitialSteeringPoll) {
    skipInitialSteeringPoll = false;
    return [];
  }
  return this.steeringQueue.drain();
},
```

The `skipInitialSteeringPoll` flag is used when the agent is already continuing from a steering message — it avoids double-draining on the very first turn of that sub-run. After that, the queue is drained once per inter-turn checkpoint.

You can also clear all pending steering messages if you change your mind:

```ts
agent.clearSteeringQueue();
```

---

## Follow-Up Messages: Work After the Agent Stops

The steering queue delivers messages between turns within an ongoing run. But what if the agent has already decided it is done — it emitted `agent_end` — and you want to trigger a second task?

That is the **follow-up queue**:

```ts
agent.followUp({
  role: "user",
  content: [{ type: "text", text: "Now summarise your findings." }],
  timestamp: Date.now(),
});
```

The follow-up queue is also drained between turns, but it is polled only after the steering queue is empty and the agent would otherwise stop. In practice this means: steering messages keep the current conversation going; follow-up messages start a new one when the current one finishes.

The `continue()` method shows this clearly. When called after the last message is an assistant message (i.e. the agent has stopped), it checks the steering queue first, then the follow-up queue:

```ts
async continue(): Promise<void> {
  // ...
  if (lastMessage.role === "assistant") {
    const queuedSteering = this.steeringQueue.drain();
    if (queuedSteering.length > 0) {
      await this.runPromptMessages(queuedSteering, { skipInitialSteeringPoll: true });
      return;
    }

    const queuedFollowUps = this.followUpQueue.drain();
    if (queuedFollowUps.length > 0) {
      await this.runPromptMessages(queuedFollowUps);
      return;
    }

    throw new Error("Cannot continue from message role: assistant");
  }

  await this.runContinuation();
}
```

If the last message is a user or tool-result message (the loop stopped mid-conversation for some reason), `continue()` calls `runAgentLoopContinue` to resume from the existing transcript without adding new messages.

---

## QueueMode: One at a Time vs All

Both queues support two drain modes controlled by `QueueMode`:

| Mode | Behaviour |
|---|---|
| `"one-at-a-time"` (default) | Each drain call removes and returns **the first** queued message only |
| `"all"` | Each drain call removes and returns **all** queued messages at once |

You set the mode at construction time or switch it on the fly:

```ts
// At construction:
const agent = new Agent({ steeringMode: "all", followUpMode: "one-at-a-time" });

// Or later:
agent.steeringMode = "all";
agent.followUpMode = "one-at-a-time";
```

`"one-at-a-time"` is the right choice when each message should give the agent a chance to respond before the next arrives — for example, a sequence of user instructions you want the agent to process one by one. `"all"` makes sense when you want to batch-inject several context messages as a group before the agent proceeds.

Internally, the `PendingMessageQueue` class handles both modes in its `drain()` method:

```ts
drain(): AgentMessage[] {
  if (this.mode === "all") {
    const drained = this.messages.slice();
    this.messages = [];
    return drained;
  }
  // "one-at-a-time"
  const first = this.messages[0];
  if (!first) return [];
  this.messages = this.messages.slice(1);
  return [first];
}
```

---

## Abort: Cancelling a Run in Progress

To stop a run immediately:

```ts
agent.abort();
```

Internally this calls `this.activeRun?.abortController.abort()`. The `AbortSignal` that was passed into `runAgentLoop` and into every subscriber is now in an aborted state. The loop stops at its next signal-check point, and all listeners that are watching the signal can clean up.

When a run is aborted, `handleRunFailure` synthesises a terminal message with `stopReason: "aborted"` and emits `message_start`, `message_end`, `turn_end`, and `agent_end` events — so subscribers always see a complete event sequence even when the run did not finish naturally.

You can also read the active signal at any time without aborting it:

```ts
const signal = agent.signal; // AbortSignal | undefined
```

`undefined` means the agent is idle.

---

## waitForIdle: Knowing When Everything Has Settled

After calling `prompt()` or `abort()`, you may need to wait until the agent (and all its listeners) are fully done before proceeding:

```ts
await agent.waitForIdle();
console.log("Agent is idle. Final messages:", agent.state.messages);
```

`waitForIdle()` returns the `promise` from the current `activeRun`, or `Promise.resolve()` if no run is active. The promise resolves only after `finishRun()` executes — and `finishRun()` runs in the `finally` block of `runWithLifecycle`, which means it runs *after every listener for `agent_end` has settled*.

This sequencing guarantee is important: if your `agent_end` listener writes a file or updates a database, `waitForIdle()` will not resolve until that work is done.

---

## Event Processing: Keeping State in Sync

As the loop emits events, the class reduces them into `_state` and then fans them out to subscribers. Let's look at the full set of events and what each one does to the state:

| Event | State change |
|---|---|
| `message_start` | `streamingMessage` is set to the new (partial) message |
| `message_update` | `streamingMessage` is updated with accumulated content |
| `message_end` | `streamingMessage` cleared; message pushed to `messages` array |
| `tool_execution_start` | Tool call ID added to `pendingToolCalls` |
| `tool_execution_end` | Tool call ID removed from `pendingToolCalls` |
| `turn_end` | If the message has an error, `errorMessage` is set |
| `agent_end` | `streamingMessage` cleared |

After updating state, `processEvents` iterates the `listeners` set and awaits each one in order:

```ts
for (const listener of this.listeners) {
  await listener(event, signal);
}
```

The fact that this is sequential — not `Promise.all` — means listeners fire in the order they were registered, and a slow listener delays later ones. This is deliberate: it avoids race conditions between listeners that both read and write shared state.

---

## reset(): Starting Over

To wipe the transcript and all runtime state without creating a new `Agent` instance:

```ts
agent.reset();
```

This clears `messages`, `isStreaming`, `streamingMessage`, `pendingToolCalls`, `errorMessage`, the follow-up queue, and the steering queue. The agent's configuration (tools, system prompt, model, hooks) is left untouched. This is useful when you want to reuse an agent with the same settings for a fresh conversation.

---

## Putting It All Together: A Complete Example

Here is a self-contained walkthrough that uses everything we have covered: construction, subscription, a prompt, mid-run steering, a follow-up, and idle-wait:

```ts
import { Agent } from "agent-core";

// 1. Create the agent with initial state
const agent = new Agent({
  initialState: {
    systemPrompt: "You are a file-system assistant. Answer concisely.",
    model: myModel,
    tools: [listFilesTool, readFileTool],
  },
});

// 2. Subscribe to events
const unsubscribe = agent.subscribe(async (event, signal) => {
  switch (event.type) {
    case "message_start":
      process.stdout.write("\nAssistant: ");
      break;
    case "message_update":
      // stream partial text to terminal
      for (const part of event.message.content) {
        if (part.type === "text") process.stdout.write(part.text);
      }
      break;
    case "message_end":
      process.stdout.write("\n");
      break;
    case "agent_end":
      console.log("[run complete]");
      break;
  }
});

// 3. Queue a follow-up before we even start
agent.followUp({
  role: "user",
  content: [{ type: "text", text: "Now summarise what you found in one sentence." }],
  timestamp: Date.now(),
});

// 4. Start the initial prompt (non-blocking — we'll await below)
const runPromise = agent.prompt("List all TypeScript files in src/");

// 5. Steer mid-run (safe to call while the agent is working)
setTimeout(() => {
  agent.steer({
    role: "user",
    content: [{ type: "text", text: "Include file sizes in your response." }],
    timestamp: Date.now(),
  });
}, 500);

// 6. Wait for everything — initial run + steering + follow-up — to finish
await agent.waitForIdle();

// 7. Inspect the final transcript
console.log("Total messages:", agent.state.messages.length);

// 8. Clean up the subscriber
unsubscribe();
```

A few points to notice:

- We called `agent.waitForIdle()` rather than awaiting `runPromise`. Both eventually resolve, but `waitForIdle()` also captures any follow-up runs that the queue triggered, which `runPromise` does not.
- The steering message was enqueued from outside the run using `steer()`. The loop picks it up at the next inter-turn checkpoint without any action on our part.
- The follow-up was enqueued before `prompt()` was even called. It sits in the queue and fires once the initial run completes and the agent would otherwise go idle.

---

## The Distinction Between Loop and Class

It is worth being explicit about what each layer owns:

| Concern | Owned by |
|---|---|
| LLM call, token streaming | `agentLoop` / `agentLoopContinue` |
| Tool execution logic | `agentLoop` |
| Turn-to-turn sequencing within one run | `agentLoop` |
| Persistent transcript across runs | `Agent` |
| Subscriber fan-out | `Agent` |
| Steering and follow-up queuing | `Agent` |
| Abort control | `Agent` |
| Idle-wait coordination | `Agent` |
| Concurrent-run prevention | `Agent` |

The loop is mechanism; the class is policy. You can use the loop directly for single-shot automation. You reach for the class when a caller (a UI, a test harness, an orchestrator) needs to observe and control a long-running agent over time.

---

## Summary

The `Agent` class wraps the stateless `agentLoop` function in a stateful object with a well-defined lifecycle:

1. **Construct** with `AgentOptions`; defaults cover the common case.
2. **Subscribe** to receive every `AgentEvent` with `subscribe()`.
3. **Send a prompt** with `prompt()` or resume with `continue()`.
4. **Steer** mid-run by enqueuing messages with `steer()`.
5. **Queue follow-ups** with `followUp()` to chain work after the current run stops.
6. **Abort** with `abort()` to cancel a run; listeners still see a complete event sequence.
7. **Await idle** with `waitForIdle()` to know when all listeners have settled.
8. **Reset** with `reset()` to start fresh without rebuilding the agent.

The next chapter looks at the harness layer that sits above `Agent`, adding session persistence, context compaction, and skill loading.

---

← Previous: [The Agent Loop: Turn-by-Turn LLM and Tool Execution](./the-agent-loop.md) · Next: [The Agent Harness: Compaction, Session Storage, and Skills](./harness-session-and-compaction.md) →
