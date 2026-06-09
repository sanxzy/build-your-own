# Build an AI Agent Swarm from Scratch

A complete, beginner-friendly course that teaches you to build **Swarm** — a system that runs many AI agents as a coordinated team — from the ground up, one layer at a time. No prior AI experience required. If you can read TypeScript, you can follow along.

This is a **documentation library**: 23 self-contained, step-by-step chapters arranged as a single guided reading path, from "what even is an agent swarm?" all the way to a real orchestration system with adapters, a task queue, squads, real-time streaming, scheduling, and spend governance.

> Everything here is written to be followed **without any source repository open**. Each chapter introduces every term it uses, shows complete examples, and links back to the concepts it builds on. Every chapter runs **without an API key** thanks to a built-in mock adapter — swapping in a real LLM is always an opt-in step.

---

## Who this is for

- Students and junior developers who want to understand how multi-agent systems actually work — not just use one.
- Anyone who learns best by **building it themselves**, in small motivated steps.
- Engineers evaluating how to orchestrate a fleet of AI agents in their own product.

You need: basic TypeScript, Node.js 22 or later, and curiosity. That's it.

## What you'll build

By the end you will have built, in strict dependency order:

- An **agent abstraction** — one adapter interface (`invoke` / `status` / `cancel`) behind which a mock agent, a real LLM (OpenAI- or Anthropic-compatible), a local CLI, and a webhook all look the same.
- A **task queue** with an atomic claim loop so many runners share one queue without ever double-working a task, plus crash recovery and liveness.
- **Coordination** — an org chart, squads led by a delegating leader, and agent-to-agent communication through sub-tasks.
- **Real-time streaming** over WebSockets — a runner hub that wakes workers instantly, and a live board that streams progress to a dashboard.
- **Scheduling** — recurring agent work driven by cron, with concurrency and catch-up policies.
- **Governance** — per-scope budgets with soft alerts and hard stops, and human-in-the-loop approval gates.

## How to read it

Read the chapters **in order** — each builds on the ones before it. The system is taught foundation-first, because that is how the parts depend on one another:

```
the swarm            (Putting It All Together — composes everything)
├── the agent        (adapters: mock → LLM → process/HTTP → registry)   ← taught first
├── tasks & queue    (model, atomic claim loop, crash recovery)         ← builds on the agent
├── coordination     (org chart, squads, agent-to-agent comms)          ← builds on the queue
├── real-time        (runner hub + live board over WebSockets)          ← builds on the queue
├── scheduling       (schedule model + the scheduler loop)              ← builds on coordination
└── governance       (budgets & cost, approval gates)                   ← builds on runs + tasks
```

Start at **[the landing page](index.md)** for the full linked map, or jump straight into [Chapter 1](docs/getting-started/what-is-a-swarm.md) and walk down the list below. Every chapter ends with prev/next links so you can read it like a book.

## Table of contents

### 1. Getting started
1. [What Is an Agent Swarm?](docs/getting-started/what-is-a-swarm.md)
2. [Prerequisites and Project Setup](docs/getting-started/project-setup.md)
3. [Your First Agent](docs/getting-started/your-first-agent.md)

### 2. The agent
4. [The Adapter Interface](docs/the-agent/adapter-interface.md)
5. [The Mock Adapter](docs/the-agent/mock-adapter.md)
6. [The LLM Adapter](docs/the-agent/llm-adapter.md)
7. [Process and HTTP Adapters](docs/the-agent/process-and-http-adapters.md)
8. [The Adapter Registry](docs/the-agent/adapter-registry.md)
9. [A Run: Sessions, Usage, and Cost](docs/the-agent/a-run.md)

### 3. Tasks and the queue
10. [Modeling Tasks](docs/tasks-and-queue/modeling-tasks.md)
11. [The Task Queue and Worker Claim Loop](docs/tasks-and-queue/task-queue-and-claim-loop.md)
12. [Crash Recovery and Liveness](docs/tasks-and-queue/crash-recovery-and-liveness.md)

### 4. Coordination
13. [Agents as a Team: The Org Chart](docs/coordination/org-chart.md)
14. [Squads: A Leader That Delegates](docs/coordination/squads.md)
15. [Agent-to-Agent Communication](docs/coordination/agent-to-agent-communication.md)

### 5. Real-time
16. [WebSockets I — The Runner Hub](docs/real-time/runner-hub.md)
17. [WebSockets II — The Live Board](docs/real-time/live-board.md)

### 6. Scheduling
18. [The Schedule Data Model](docs/scheduling/schedule-data-model.md)
19. [The Scheduler Loop](docs/scheduling/scheduler-loop.md)

### 7. Governance
20. [Budgets and Cost Tracking](docs/governance/budgets-and-cost-tracking.md)
21. [Approvals and Governance Gates](docs/governance/approvals-and-governance-gates.md)

### 8. Wrap-up
22. [Putting It All Together](docs/wrap-up/putting-it-all-together.md)
23. [Where to Go Next](docs/wrap-up/where-to-go-next.md)

## How it's taught

- **You build it, step by step.** Each chapter starts from the problem it solves, then arrives at the solution — never a finished-code dump.
- **Self-contained chapters.** Every term is introduced on first use; you can read any page without a source repository open.
- **Runs keyless.** A built-in mock adapter means every chapter runs with no API key and no spend; a real OpenAI- or Anthropic-compatible provider is always opt-in.
- **Brand- and version-agnostic.** Examples use generic names and minimal starting versions, so the concepts are yours to reuse.
