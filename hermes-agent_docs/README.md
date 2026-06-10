# Understand and Operate Hermes Agent — From Zero

A complete, beginner-first course on **Hermes Agent v0.16.0**, the self-improving AI agent built by **Nous Research** (MIT-licensed). It teaches every real subsystem — what it is, why it was designed that way, how its parts interact, and how to operate it end to end — one motivated step at a time. No prior agent experience required. If you have used an LLM chat app and can read a little Python and YAML, you can follow along.

This is a **documentation library**: 42 self-contained, step-by-step chapters arranged as a single dependency-ordered reading path, from "what even is an AI agent?" all the way to multi-agent kanban boards, swarm topologies, the credential failover system, and the closed learning loop that defines Hermes.

> Everything here is written to be followed **without the Hermes source repository open**. Each chapter introduces every term on first use, shows complete examples, includes a diagram at every multi-step mechanism, and links back to the concepts it builds on. Every technical claim is traced to the Hermes v0.16.0 source.

---

## Who this is for

- High-school students, beginners, and junior developers who want to understand how a real AI agent works — not just use one.
- Anyone who learns best by following a **motivated walkthrough**, where each step starts from the problem it solves.
- Engineers evaluating how to deploy, extend, or operate Hermes safely.

You need: curiosity, a little command-line comfort, and the willingness to meet a genuinely complex system one layer at a time.

## What you'll be able to do

By the end you will be able to:

- Explain what makes Hermes a **self-improving** agent — its closed **learning loop** (creates skills from experience, improves them in use, nudges itself to persist knowledge, searches its own past, models the user across sessions).
- Configure a profile, connect a provider, and run a conversation from scratch.
- Create a kanban board, decompose and dispatch tasks, and trace how workers claim and run them.
- Choose correctly between `delegate_task`, kanban dispatch, and swarm topologies.
- Name the five memory layers and explain how each persists across turns and sessions.
- Extend Hermes with an observer plugin or middleware against the right schema version.
- Explain why the **operating system** — not any in-process mechanism — is the only real security boundary.
- Recover from budget exhaustion, context overflow, credential cooldown, a stale cron run, and a crashed worker.

## How to read it

Read the chapters **in order** — each builds only on the ones before it. The clusters are taught foundation-first, because that is how they depend on one another:

```
identity → core runtime → memory → persistence → autonomy → tools
→ multi-agent → task lifecycle → providers → skills → gateway
→ security → extensions → interfaces → observability → scenarios → design
```

Start at the **[landing page](index.md)** for the full linked map, or jump straight into [Chapter 1](docs/getting-started/what-is-hermes.md) and walk down the list below. Every chapter ends with prev/next links so you can read it like a book. New to a term? The **[Glossary](docs/reference/glossary.md)** defines every one.

## Table of contents

### 1. Getting started
1. [What Hermes Is — Identity, Learning Loop, and Six Capabilities](docs/getting-started/what-is-hermes.md)
2. [Glossary — LLM, Kanban, SQLite, WAL, CAS, Toolset, and More](docs/reference/glossary.md)
3. [What This Guide Covers — An Honest Map of Hermes v0.16.0](docs/getting-started/scope-and-coverage.md)

### 2. Core runtime
4. [The AIAgent Class and the run_conversation() Loop](docs/core-runtime/aiagent-and-conversation-loop.md)
5. [Iteration Budget, Toolsets, and the Tools Registry](docs/core-runtime/iteration-budget-and-toolsets.md)
6. [Sequential vs Concurrent Tool Dispatch and the Guardrail Controller](docs/core-runtime/tool-dispatch-and-guardrails.md)

### 3. Memory
7. [The Five Memory Layers](docs/memory/five-memory-layers.md)
8. [ContextCompressor and the LCM Context Engine](docs/memory/context-compressor-and-lcm.md)
9. [MemoryManager, External Memory Providers, and the Nudge-to-Persist Loop](docs/memory/memory-manager-and-external-providers.md)
10. [SessionDB — SQLite, WAL, FTS5, and Conversation Search](docs/memory/sessiondb-fts-and-search.md)

### 4. Persistence and state
11. [The Hermes Home Directory and Profile Isolation](docs/persistence/home-directory-and-profiles.md)
12. [Compression Chains, Session Splitting, and WAL Fallback](docs/persistence/compression-chains-and-wal-fallback.md)

### 5. Autonomy
13. [The Cron Scheduler — tick(), Job Kinds, and Inactivity Timeout](docs/autonomy/cron-scheduler.md)
14. [Webhook Triggers and the Kanban Dispatcher Tick](docs/autonomy/webhooks-and-dispatcher-tick.md)

### 6. Tools
15. [The Tools Registry, Approval Gate, and File-Write Safety](docs/tools/tools-registry-and-approval-gate.md)

### 7. Multi-agent
16. [In-Process Delegation with delegate_task](docs/multi-agent/in-process-delegation.md)
17. [Kanban Dispatch — Boards, dispatch_once(), CAS Claim, and Worker Context](docs/multi-agent/kanban-dispatch.md)
18. [Swarm Topologies with create_swarm()](docs/multi-agent/swarm-topologies.md)

### 8. Task lifecycle
19. [The Nine-Status Task State Machine](docs/task-lifecycle/nine-status-state-machine.md)
20. [The Task Dataclass, DAG Links, Worker Handoff, and Artifacts](docs/task-lifecycle/task-dataclass-dag-and-handoff.md)

### 9. Providers and models
21. [Config-Driven Provider Routing and the Four api_mode Values](docs/providers/config-driven-routing-and-api-modes.md)
22. [Provider Adapters — Anthropic, Bedrock, Gemini, and Codex](docs/providers/provider-adapters.md)
23. [CredentialPool — Rotation Strategies, Cooldowns, and Failover](docs/providers/credential-pool-and-failover.md)

### 10. Skills and the learning loop
24. [Skill Structure, the Three Skill Tools, and Skill Bundles](docs/skills/skill-structure-and-tools.md)
25. [The Curator and the Full Learning Loop](docs/skills/curator-and-the-learning-loop.md)

### 11. Gateway and platforms
26. [Gateway Routing, Delivery Targets, and Stream Event Vocabulary](docs/gateway/routing-delivery-and-stream-events.md)
27. [Gateway Authorization, DM Pairing, Slash Commands, and Handoff State](docs/gateway/authorization-pairing-and-slash-commands.md)

### 12. Security
28. [Security — The OS Boundary, Heuristics, and Isolation Postures](docs/security/os-boundary-and-isolation-postures.md)

### 13. Extension surfaces
29. [The Plugin System and Observer Hooks (hermes.observer.v1)](docs/extensions/plugin-system-and-observer-hooks.md)
30. [Middleware — Rewriting Requests and Wrapping Execution (hermes.middleware.v1)](docs/extensions/middleware.md)
31. [MCP Client Integration and hermes mcp serve](docs/extensions/mcp-client-and-server.md)
32. [ACP Adapter, IDE Integration, and the Plugin LLM Facade](docs/extensions/acp-adapter-and-plugin-llm.md)

### 14. Interfaces and deployment
33. [CLI, TUI, Web Dashboard, and Electron Desktop](docs/interfaces/cli-tui-and-web-dashboard.md)
34. [Terminal Backends and the Mixture-of-Agents Tool](docs/interfaces/terminal-backends-and-moa.md)

### 15. Observability
35. [Observability — Log Files, Observer Events, and Bundled Consumers](docs/observability/logs-hooks-and-plugins.md)

### 16. End-to-end scenarios
36. [Scenario 1 — From Conversation to Skill Creation](docs/scenarios/single-agent-conversation-to-skill.md)
37. [Scenario 2 — Kanban Board Workflow End to End](docs/scenarios/kanban-board-workflow.md)
38. [Scenario 3 — Multi-Profile Swarm Coordination](docs/scenarios/multi-profile-swarm.md)
39. [Scenario 4 — Cron Job with Stale-Run Fast-Forward and Platform Delivery](docs/scenarios/cron-and-webhook-delivery.md)
40. [Scenario 5 — Human-in-the-Loop Approval Workflow](docs/scenarios/human-in-the-loop-approval.md)

### 17. Design and best practices
41. [Architecture Decisions and Tradeoffs](docs/design/architecture-decisions-and-tradeoffs.md)
42. [Best Practices — Delegation, Budgets, Credentials, Memory, and Skill Authoring](docs/design/best-practices.md)

---

Start at [What Hermes Is](docs/getting-started/what-is-hermes.md) → or open the [landing map](index.md).
