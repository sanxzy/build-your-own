---
title: Build Your Own AI Agent Swarm
description: A dependency-ordered, beginner-friendly course that teaches you to build a multi-agent orchestration system from scratch — agents and adapters, a task queue, coordination, real-time streaming, scheduling, and governance.
category: overview
type: explanation
tags: [ai agent swarm, multi-agent, orchestration, adapter, task queue, squad, websocket, scheduler, budget, governance, build from scratch, tutorial, course, index]
keywords: [how to build an agent swarm, multi-agent orchestration, agent control plane, task queue, agent adapter, from scratch]
---

**TL;DR** — This library teaches you to build **Swarm**, a system that runs many AI agents as a coordinated team, one layer at a time. You start by running a single agent on a keyless mock adapter, then give it a real LLM, then build the task queue that lets many agents share work safely, then coordination, real-time streaming, scheduling, and spend governance — until you have a complete, working agent swarm. No prior AI experience is assumed: if you can read TypeScript, you can follow along, and every chapter runs without an API key thanks to the built-in mock adapter.

## What you'll build

By the end you will have built, from scratch and in dependency order:

- An **agent abstraction** — one adapter interface (`invoke` / `status` / `cancel`) behind which a mock agent, a real LLM (OpenAI- or Anthropic-compatible), a local CLI, and a webhook all look the same.
- A **task queue** with an atomic claim loop, so many runners share one queue without ever double-working a task — plus crash recovery and liveness.
- **Coordination** — an org chart, squads led by a delegating leader, and agent-to-agent communication through sub-tasks.
- **Real-time streaming** over WebSockets — a runner hub that wakes workers the instant work appears, and a live board that streams progress to a dashboard.
- **Scheduling** — recurring agent work driven by cron, with concurrency and catch-up policies.
- **Governance** — per-scope budgets with soft alerts and hard stops, and human-in-the-loop approval gates.

## How to read this

Read the chapters **in order** — each builds on the ones before it, foundation-first. You set up one repository in Chapter 2 and grow it across the whole book; every chapter opens with the problem the previous one couldn't yet solve. Every chapter is self-contained, so you can also revisit any single topic later. The default examples run on a **mock adapter** with no API key; swapping in a real provider is always an opt-in step.

You need: basic TypeScript, Node.js 22 or later, and curiosity. That's it.

## The reading path

### 1. Getting started — the mental model and your first running agent
- [What Is an Agent Swarm?](docs/getting-started/what-is-a-swarm.md) — orchestrator, runner, agent, and task: the four roles, and why a control plane matters.
- [Prerequisites and Project Setup](docs/getting-started/project-setup.md) — scaffold the repo: TypeScript, SQLite + Drizzle (Postgres as a one-line swap), and your provider config.
- [Your First Agent](docs/getting-started/your-first-agent.md) — run one task end-to-end on the keyless mock adapter; optionally swap in a real LLM.

### 2. The agent — one interface, many backends
- [The Adapter Interface](docs/the-agent/adapter-interface.md) — the three-method contract every agent backend satisfies.
- [The Mock Adapter](docs/the-agent/mock-adapter.md) — a keyless, scripted agent: the workhorse for every chapter.
- [The LLM Adapter](docs/the-agent/llm-adapter.md) — a real agent built on an OpenAI- or Anthropic-compatible provider config, with streaming and cost.
- [Process and HTTP Adapters](docs/the-agent/process-and-http-adapters.md) — run a local CLI agent, or a fire-and-forget webhook agent.
- [The Adapter Registry](docs/the-agent/adapter-registry.md) — register and look up adapters (and providers) at runtime.
- [A Run: Sessions, Usage, and Cost](docs/the-agent/a-run.md) — what gets recorded each time an agent works.

### 3. Tasks and the queue — sharing work safely
- [Modeling Tasks](docs/tasks-and-queue/modeling-tasks.md) — the task data model and its status state machine.
- [The Task Queue and Worker Claim Loop](docs/tasks-and-queue/task-queue-and-claim-loop.md) — enqueue, then claim with an atomic checkout (409 on conflict).
- [Crash Recovery and Liveness](docs/tasks-and-queue/crash-recovery-and-liveness.md) — detect dead runners and recover orphaned work.

### 4. Coordination — agents as a team
- [Agents as a Team: The Org Chart](docs/coordination/org-chart.md) — a `reportsTo` hierarchy and workspace-scoped isolation.
- [Squads: A Leader That Delegates](docs/coordination/squads.md) — assign to a squad; a leader routes work to members.
- [Agent-to-Agent Communication](docs/coordination/agent-to-agent-communication.md) — sub-tasks, request depth, and blockers.

### 5. Real-time — watching the swarm work
- [WebSockets I — The Runner Hub](docs/real-time/runner-hub.md) — push work to runners the instant it appears.
- [WebSockets II — The Live Board](docs/real-time/live-board.md) — stream progress to a dashboard, with auth and a message catalogue.

### 6. Scheduling — recurring work
- [The Schedule Data Model](docs/scheduling/schedule-data-model.md) — cron triggers, revisions, and concurrency/catch-up policies.
- [The Scheduler Loop](docs/scheduling/scheduler-loop.md) — fire due triggers exactly once, even across restarts.

### 7. Governance — control spend and consequential decisions
- [Budgets and Cost Tracking](docs/governance/budgets-and-cost-tracking.md) — cost events, per-scope budgets, soft alerts and hard stops.
- [Approvals and Governance Gates](docs/governance/approvals-and-governance-gates.md) — human-in-the-loop approval for hiring and strategy.

### 8. Wrap-up
- [Putting It All Together](docs/wrap-up/putting-it-all-together.md) — run one realistic scenario end-to-end through the whole swarm.
- [Where to Go Next](docs/wrap-up/where-to-go-next.md) — an honest survey of what production adds: scaling, security, observability.

---

Start at **[Chapter 1: What Is an Agent Swarm?](docs/getting-started/what-is-a-swarm.md)** and follow the prev/next links at the bottom of each chapter to read it like a book.
