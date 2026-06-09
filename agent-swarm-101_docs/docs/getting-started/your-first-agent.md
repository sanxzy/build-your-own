---
title: "Your First Agent"
description: Run one task end-to-end on the built-in mock adapter — no API key needed — then optionally swap in a real OpenAI or Anthropic provider.
category: getting-started
type: tutorial
tags:
  [
    first agent,
    hello agent,
    mock adapter,
    mock provider,
    echo adapter,
    task enqueue,
    end-to-end,
    keyless,
    run,
    invoke,
    registerMockProvider,
    mockAssistantMessage,
    mockText,
    InvocationContext,
    AdapterResult,
    ANTHROPIC_API_KEY,
    OPENAI_API_KEY,
    opt-in real LLM,
    complete,
    getModel,
    provider config,
    process adapter,
    agent swarm tutorial,
  ]
keywords:
  [
    swarm tutorial,
    first run,
    no api key,
    canned response,
    scripted response,
    mock llm,
    mock provider,
    task lifecycle,
    enqueue task,
    runner invoke,
    local llm,
    openai compatible,
    anthropic compatible,
  ]
sources: [S27, S49, S43, S41]
---

**TL;DR** — We have a scaffolded repo but nothing is actually running an agent yet. In this chapter we wire up one agent backed by a mock adapter — which returns scripted responses with zero network calls and no API key — enqueue a single task, invoke the agent, and read the result. By the end you will have seen a complete end-to-end cycle. The final section shows the two-line change that replaces the mock with a real OpenAI-compatible or Anthropic provider.

# Your First Agent

When you finished setting up the scaffold (see [Prerequisites and Project Setup](./project-setup.md)), you had a working database, a migration, and a bare project skeleton. What you did not have was anything that could actually execute work. That is what we will build here.

We will stay keyless for the whole main walkthrough. The mock adapter is a deterministic, in-process provider that returns the exact responses you script — no network, no tokens, no cost. That makes it the right foundation for understanding the agent loop before you bring in a live model. The rest of this book defaults to the mock for the same reason.

## A quick recap of the moving parts

Before we write any code it helps to have the three key pieces in mind. If you read [What Is an Agent Swarm?](./what-is-a-swarm.md), these will be familiar; here is the one-sentence version of each.

- **An agent** is the record that describes one AI worker — its configuration and the adapter it runs through.
- **An adapter** is the execution interface the runner calls to do real work. It receives a context object and returns a result. (The full contract is in the next chapter, [The Adapter Interface](../the-agent/adapter-interface.md); for now we only need the part that lets us return a scripted reply.)
- **A task** is the unit of work the runner picks up. It carries the prompt or description of what needs doing, and its lifecycle runs from `queued` → `running` → `succeeded` / `failed`.

That is the entire cycle we are about to build.

## Step 1 — The problem: we have no execution path yet

We have a database and a project skeleton. But if we tried to "run an agent" right now we would have nowhere to plug in the execution logic. We need two things:

1. Something that can respond to a task without calling a real LLM.
2. A minimal runner loop that wires a task to that something and records the result.

Let's start with the mock adapter, because once that is in place the runner loop becomes obvious.

## Step 2 — Building the mock adapter

### What the mock provider does

The mock provider is an in-process provider that returns responses from a queue you populate yourself. When you call `setResponses([...])` on the registration, you are handing it a list of scripted replies. Each time the provider is asked for a completion, it shifts one reply off the front of that queue and returns it — no network, no tokens, zero cost.

The relevant facts about the queue (grounded in S49):

- If the queue is empty when a request arrives, the provider returns an error message with `errorMessage: "No more mock responses queued"`.
- `setResponses([...])` replaces the queue entirely.
- `appendResponses([...])` adds to the end of the existing queue.
- `registration.state.callCount` tracks how many times the provider has been called.
- Call `registration.unregister()` when you are done to remove the provider from the global registry.

To build a content block you pass to `mockAssistantMessage`, you use the helpers:

- `mockText(text)` — a plain text block.
- `mockThinking(text)` — a thinking/reasoning block.
- `mockToolCall(name, args)` — a tool-call block.

Here is the smallest mock adapter we can write. Create a new file at `src/adapters/mock.ts`:

Before we look at the code, a quick note on the import. `@swarm/llm` is the in-repo LLM toolkit package — the workspace peer to `@swarm/db` and `@swarm/shared` you saw in the project layout. Its full design is covered in [The LLM Adapter](../the-agent/llm-adapter.md), but for now the part we need is its built-in mock provider: a deterministic, in-process backend that lets you run an agent without registering an API key. You will flesh `@swarm/llm` out as the guide progresses; at this stage it ships ready to use out of the box.

```ts
// src/adapters/mock.ts
//
// A zero-network adapter backed by the mock provider.
// Returns scripted responses for deterministic local runs.

import {
  registerMockProvider,  // registers a fresh mock provider in the global registry and returns a handle
  mockAssistantMessage,  // wraps one or more content blocks into a scripted assistant reply
  mockText,              // creates a plain-text content block to pass to mockAssistantMessage
  type MockProviderRegistration,  // the handle returned by registerMockProvider — owns the queue and getModel()
} from "@swarm/llm";

export interface MockAdapterConfig {
  /** One scripted reply per expected invocation. */
  responses: string[];
}

export interface MockAdapter {
  /** Execute one task against the mock provider. */
  invoke(prompt: string): Promise<{ text: string; callCount: number }>;
  /** Tear down the mock provider when you are finished. */
  dispose(): void;
}

export function createMockAdapter(config: MockAdapterConfig): MockAdapter {
  const registration: MockProviderRegistration = registerMockProvider();

  // Load all scripted replies into the queue up front.
  registration.setResponses(
    config.responses.map((reply) => mockAssistantMessage(mockText(reply)))
  );

  return {
    async invoke(prompt: string) {
      const model = registration.getModel();

      // complete() waits for the full reply — no streaming needed here.
      // The mock provider shifts the next scripted reply off the queue.
      const { complete } = await import("@swarm/llm");
      const response = await complete(model, {
        messages: [{ role: "user", content: prompt, timestamp: Date.now() }],
      });

      // Extract the text block from the response content.
      const textBlock = response.content.find((b) => b.type === "text");
      const text = textBlock?.type === "text" ? textBlock.text : "";

      return { text, callCount: registration.state.callCount };
    },

    dispose() {
      registration.unregister();
    },
  };
}
```

A few things to notice here:

- `registerMockProvider()` returns a `MockProviderRegistration` that owns the queue and exposes `getModel()`. The model it hands back is a `Model` object — the same shape any real provider would give you.
- We call the generic `complete()` function, not any provider-specific variant. That is the point of the provider abstraction: the caller does not need to know whether the model is mock or real.
- `response.content` is an array of content blocks. Text blocks have `type: "text"` and a `text` property. (In a real run there might also be `toolCall` or `thinking` blocks — but the mock only returns what we scripted.)

### Why not a lower-level approach?

You might wonder why we are going through the provider registry at all for a mock. The short answer is that the adapter contract the runner calls (covered in [The Adapter Interface](../the-agent/adapter-interface.md)) hands the adapter an `InvocationContext` that includes an `agent` and a `config` object. A real adapter unpacks `config.command` or `config.baseUrl` from that context. Our mock ignores `config` and instead uses pre-scripted responses. The pattern from S27 — `ctx.runId`, `ctx.agent`, `ctx.config`, `ctx.onLog` — is the full shape we will see in the next chapter. For now, the inner `invoke` is enough.

## Step 3 — Defining an agent record

An agent in Swarm is a configuration record that tells the runner which adapter to use, what prompt template to start with, and a few runtime limits. For our walkthrough we will keep this in memory rather than in the database — that is the smallest thing that works.

Create `src/agents/echo-agent.ts`:

```ts
// src/agents/echo-agent.ts
//
// A minimal in-memory agent definition.
// No database row needed for this walkthrough.

export interface AgentDefinition {
  id: string;
  name: string;
  /** Identifies which adapter to use. */
  adapterType: "mock" | "process" | "http";
  /** Adapter-specific settings. */
  adapterConfig: Record<string, unknown>;
}

// The agent we will run in this chapter.
export const echoAgent: AgentDefinition = {
  id: "agent-echo-01",
  name: "Echo Agent",
  adapterType: "mock",
  adapterConfig: {
    // Scripted replies — one per task we plan to enqueue.
    responses: ["Hello from the mock adapter. Task complete."],
  },
};
```

The `adapterConfig` is deliberately open — a real system would validate this against a schema per adapter type. For now it carries the one field our mock adapter reads: `responses`.

## Step 4 — Enqueueing a task

A task is the unit of work. In the full system a task lives in the database and has a rich state machine. For this walkthrough we represent it as a plain in-memory object so we can focus on the lifecycle, not the SQL.

Create `src/tasks/types.ts`:

```ts
// src/tasks/types.ts

export type TaskStatus = "queued" | "running" | "succeeded" | "failed";

export interface Task {
  id: string;
  agentId: string;
  prompt: string;
  status: TaskStatus;
  result?: string;
  errorMessage?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

And a tiny in-memory queue in `src/tasks/queue.ts`:

```ts
// src/tasks/queue.ts

import type { Task } from "./types.js";

const tasks: Task[] = [];

export function enqueueTask(agentId: string, prompt: string): Task {
  const now = new Date();
  const task: Task = {
    id: `task-${Date.now()}`,
    agentId,
    prompt,
    status: "queued",
    createdAt: now,
    updatedAt: now,
  };
  tasks.push(task);
  return task;
}

export function nextTask(): Task | undefined {
  return tasks.find((t) => t.status === "queued");
}

export function updateTask(id: string, patch: Partial<Task>): void {
  const task = tasks.find((t) => t.id === id);
  if (task) {
    Object.assign(task, patch, { updatedAt: new Date() });
  }
}
```

Now we can `enqueueTask("agent-echo-01", "Say hello.")` and have a `queued` row ready for the runner.

## Step 5 — The runner loop: connecting task to adapter

The runner is the piece that claims a queued task, calls the agent's adapter, and records the result. In the full system the runner is a long-running daemon that polls or receives push notifications. Here we write a single-iteration version that runs once and exits — the simplest thing that demonstrates the full cycle.

Create `src/runner/run-once.ts`:

```ts
// src/runner/run-once.ts
//
// Run exactly one queued task, then exit.
// Demonstrates the full task → adapter → result cycle.

import { echoAgent } from "../agents/echo-agent.js";
import { createMockAdapter } from "../adapters/mock.js";
import { enqueueTask, nextTask, updateTask } from "../tasks/queue.js";

async function runOnce() {
  // 1. Enqueue the task.
  const task = enqueueTask(echoAgent.id, "Say hello.");
  console.log(`Enqueued task ${task.id} — status: ${task.status}`);

  // 2. Claim the task.
  const claimed = nextTask();
  if (!claimed) {
    console.error("No queued tasks found.");
    process.exit(1);
  }
  updateTask(claimed.id, { status: "running" });
  console.log(`Running task ${claimed.id}…`);

  // 3. Build the adapter from the agent config.
  //    In the full system the runner resolves the adapter type from a registry.
  //    Here we wire it directly because we only have one adapter type.
  const adapter = createMockAdapter(
    echoAgent.adapterConfig as { responses: string[] }
  );

  // 4. Invoke the adapter with the task prompt.
  //    This is the pattern from the adapter execution contract (S27):
  //    the runner passes context, the adapter returns a result.
  try {
    const result = await adapter.invoke(claimed.prompt);

    updateTask(claimed.id, {
      status: "succeeded",
      result: result.text,
    });

    console.log(`Task ${claimed.id} succeeded.`);
    console.log(`Result: "${result.text}"`);
    console.log(`Provider call count: ${result.callCount}`);
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    updateTask(claimed.id, { status: "failed", errorMessage: message });
    console.error(`Task ${claimed.id} failed: ${message}`);
  } finally {
    // 5. Release the provider when done.
    adapter.dispose();
  }
}

runOnce().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Walk through the numbered comments and you will see the canonical adapter invocation pattern grounded in S27: the runner builds a context (here just the prompt), passes it to `adapter.invoke`, and interprets the result — checking for success or failure before writing the final status back.

## Step 6 — Run it

Add a script entry to `package.json` (the scaffold already has a `scripts` section):

```json
{
  "scripts": {
    "run-once": "tsx src/runner/run-once.ts"
  }
}
```

Now run it:

```bash
npm run run-once
```

You should see output like this:

```
Enqueued task task-1749123456789 — status: queued
Running task task-1749123456789…
Task task-1749123456789 succeeded.
Result: "Hello from the mock adapter. Task complete."
Provider call count: 1
```

No network was touched. No API key was needed. The mock provider shifted the scripted reply off its queue, the runner recorded the result, and the task moved to `succeeded`. That is the complete end-to-end cycle.

> **What just happened under the hood?** The mock provider's `complete()` call went through the same path a real provider would: build a `Context`, pass it to the registered stream function, collect the response. The difference is that the mock's stream function reads from the in-process queue instead of making an HTTP request. The `callCount` went from 0 to 1 because exactly one completion request was made.

## Step 7 — Optionally swap in a real LLM

The rest of this book defaults to the mock so every example runs keyless. But it is worth seeing how little changes when you point at a real provider.

### With an OpenAI-compatible endpoint

If you have an OpenAI key (or a local OpenAI-compatible gateway — Ollama, LM Studio, vLLM, etc.), set the environment variable and build a real model:

```bash
export OPENAI_API_KEY="sk-..."
```

Then replace the mock adapter with a real one. The provider client supports custom model objects for local gateways (S43):

```ts
// src/adapters/openai-adapter.ts (simplified — for illustration)
import { complete, type Model } from "@swarm/llm";

const model: Model<"openai-completions"> = {
  id: "gpt-4o-mini",
  name: "GPT-4o Mini",
  api: "openai-completions",
  provider: "openai",
  baseUrl: "https://api.openai.com/v1",
  reasoning: false,
  input: ["text"],
  cost: { input: 0.15, output: 0.6, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 16384,
};

// For a local Ollama gateway, point baseUrl at http://localhost:11434/v1
// and set apiKey to "dummy" — Ollama does not require a real key.

export async function invokeOpenAI(prompt: string): Promise<string> {
  const response = await complete(
    model,
    {
      messages: [{ role: "user", content: prompt, timestamp: Date.now() }],
    },
    { apiKey: process.env.OPENAI_API_KEY }
  );
  const textBlock = response.content.find((b) => b.type === "text");
  return textBlock?.type === "text" ? textBlock.text : "";
}
```

Note that for Ollama or other local gateways you can pass `apiKey: "dummy"` — those servers do not validate keys (S43).

### With an Anthropic-compatible endpoint

For Anthropic (Claude), set `ANTHROPIC_API_KEY` and use the `anthropic-messages` API (S43, S41):

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

```ts
// src/adapters/anthropic-adapter.ts (simplified — for illustration)
import { complete, type Model } from "@swarm/llm";

const model: Model<"anthropic-messages"> = {
  id: "claude-3-5-haiku-20241022",
  name: "Claude 3.5 Haiku",
  api: "anthropic-messages",
  provider: "anthropic",
  baseUrl: "https://api.anthropic.com",
  reasoning: false,
  input: ["text", "image"],
  cost: { input: 0.8, output: 4, cacheRead: 0.08, cacheWrite: 1 },
  contextWindow: 200000,
  maxTokens: 8192,
};

export async function invokeAnthropic(prompt: string): Promise<string> {
  const response = await complete(
    model,
    {
      messages: [{ role: "user", content: prompt, timestamp: Date.now() }],
    }
    // ANTHROPIC_API_KEY is picked up automatically from the environment (S43).
  );
  const textBlock = response.content.find((b) => b.type === "text");
  return textBlock?.type === "text" ? textBlock.text : "";
}
```

> **Claude-specific note (S41):** If `ANTHROPIC_API_KEY` is set in the environment, the adapter uses API-key auth. If you are running a local Claude CLI that uses subscription login instead, omit the env var. The system surfaces this as a warning in environment tests, not a hard error.

### What changes and what stays the same

The key insight is that `complete(model, context)` is the same call regardless of provider. The runner loop does not change at all. The only things that change are:

| What | Mock | Real LLM |
| ---------------------- | ---------------------------------------- | ----------------------------------------- |
| Provider registration | `registerMockProvider()` | Built-in (auto-registered at import time) |
| Model object | `registration.getModel()` | `getModel('openai', 'gpt-4o-mini')` or a custom `Model<...>` object |
| API key | None | `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` |
| Network call | No | Yes |
| Cost | Zero (cost fields are `0`) | Billed by the provider |
| Scripted responses | Yes — `setResponses([...])` | No — the model responds |

The rest of this book will use the mock unless a chapter specifically covers real-LLM integration.

## Try it yourself

Here are four experiments you can run from what you have built:

1. **Change the scripted reply.** In `echoAgent.adapterConfig.responses`, replace `"Hello from the mock adapter. Task complete."` with your own string. Re-run `npm run run-once` and confirm the new text appears.

2. **Enqueue two tasks.** Add a second `"Your second scripted reply."` to the `responses` array and call `enqueueTask` twice before starting the runner. Add a loop around the `nextTask()` / `updateTask()` / `invoke()` block and watch both tasks complete in sequence.

3. **Observe the empty-queue error.** Remove one `responses` entry so there are fewer scripted replies than tasks. Run again and read the error message that comes back from the mock provider: `"No more mock responses queued"`. This is the guard S49 specifies — the provider does not hang or crash, it returns a structured error.

4. **Point at a local OpenAI-compatible gateway.** If you have Ollama running locally, build the custom model object with `baseUrl: "http://localhost:11434/v1"` and `apiKey: "dummy"` as shown in the step above. Swap the `invokeOpenAI` call into your runner and run it against a real locally-hosted model.

---

← Previous: [Prerequisites and Project Setup](./project-setup.md) · Next: [The Adapter Interface](../the-agent/adapter-interface.md) →
