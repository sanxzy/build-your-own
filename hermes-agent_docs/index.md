---
title: Hermes Agent — Build Your Own Self-Improving AI Agent
description: A dependency-ordered, beginner-first learning guide to every real subsystem of Hermes Agent v0.16.0 — from the conversation loop to multi-agent kanban and the closed learning loop.
category: index
type: explanation
tags: [hermes agent, nous research, ai agent, learning loop, documentation library, reading guide, index, landing page]
keywords: [hermes v0.16.0, self-improving agent, build your own agent, table of contents]
---

**TL;DR** — This is a complete, beginner-first guide to **Hermes Agent v0.16.0**, the self-improving AI agent built by Nous Research. It teaches every real subsystem — what it is, why it was designed that way, and how to operate it — in a dependency-correct reading order, with every claim traced to the Hermes source and a diagram at every multi-step mechanism. Read it top to bottom like a book, or jump to a topic and follow the links back to its prerequisites.

# Hermes Agent — Build Your Own Self-Improving AI Agent

Hermes is the self-improving AI agent built by **Nous Research**. Its defining capability is a **closed learning loop**: it creates skills from experience, improves them during use, nudges itself to persist knowledge, searches its own past conversations, and builds a deepening model of who you are across sessions. It runs anywhere — a $5 VPS, a GPU cluster, idle-cheap serverless — is provider-agnostic across roughly thirty model providers, and is research-ready for batch trajectory generation.

This library is four documents in one — a learning guide, a technical handbook, an architecture reference, and an operational manual — applied at the right depth for each topic. Every chapter is self-contained: you can read any page without the source repository open, and prerequisites are recapped and linked rather than assumed.

## How to read this guide

The chapters are ordered so each one builds only on concepts established before it. If you are new to Hermes, read in order. If you are looking something up, start with the **[Glossary](./docs/reference/glossary.md)** and follow the links. New to the term "AI agent" itself? Start at chapter 1.

## The reading path

### Getting started
1. [What Hermes Is — Identity, Learning Loop, and Six Capabilities](./docs/getting-started/what-is-hermes.md)
2. [Glossary — LLM, Kanban, SQLite, WAL, CAS, Toolset, and More](./docs/reference/glossary.md)
3. [What This Guide Covers — An Honest Map of Hermes v0.16.0](./docs/getting-started/scope-and-coverage.md)

### Core runtime
4. [The AIAgent Class and the run_conversation() Loop](./docs/core-runtime/aiagent-and-conversation-loop.md)
5. [Iteration Budget, Toolsets, and the Tools Registry](./docs/core-runtime/iteration-budget-and-toolsets.md)
6. [Sequential vs Concurrent Tool Dispatch and the Guardrail Controller](./docs/core-runtime/tool-dispatch-and-guardrails.md)

### Memory
7. [The Five Memory Layers — In-Context, Compressor, Manager, SessionDB, Skills](./docs/memory/five-memory-layers.md)
8. [ContextCompressor and the LCM Context Engine](./docs/memory/context-compressor-and-lcm.md)
9. [MemoryManager, External Memory Providers, and the Nudge-to-Persist Loop](./docs/memory/memory-manager-and-external-providers.md)
10. [SessionDB — SQLite, WAL, FTS5, and Conversation Search](./docs/memory/sessiondb-fts-and-search.md)

### Persistence and state
11. [The Hermes Home Directory and Profile Isolation](./docs/persistence/home-directory-and-profiles.md)
12. [Compression Chains, Session Splitting, and WAL Fallback](./docs/persistence/compression-chains-and-wal-fallback.md)

### Autonomy
13. [The Cron Scheduler — tick(), Job Kinds, and Inactivity Timeout](./docs/autonomy/cron-scheduler.md)
14. [Webhook Triggers and the Kanban Dispatcher Tick](./docs/autonomy/webhooks-and-dispatcher-tick.md)

### Tools
15. [The Tools Registry, Approval Gate, and File-Write Safety](./docs/tools/tools-registry-and-approval-gate.md)

### Multi-agent
16. [In-Process Delegation with delegate_task](./docs/multi-agent/in-process-delegation.md)
17. [Kanban Dispatch — Boards, dispatch_once(), CAS Claim, and Worker Context](./docs/multi-agent/kanban-dispatch.md)
18. [Swarm Topologies with create_swarm()](./docs/multi-agent/swarm-topologies.md)

### Task lifecycle
19. [The Nine-Status Task State Machine](./docs/task-lifecycle/nine-status-state-machine.md)
20. [The Task Dataclass, DAG Links, Worker Handoff, and Artifacts](./docs/task-lifecycle/task-dataclass-dag-and-handoff.md)

### Providers and models
21. [Config-Driven Provider Routing and the Four api_mode Values](./docs/providers/config-driven-routing-and-api-modes.md)
22. [Provider Adapters — Anthropic, Bedrock, Gemini, and Codex](./docs/providers/provider-adapters.md)
23. [CredentialPool — Rotation Strategies, Cooldowns, and Failover](./docs/providers/credential-pool-and-failover.md)

### Skills and the learning loop
24. [Skill Structure, the Three Skill Tools, and Skill Bundles](./docs/skills/skill-structure-and-tools.md)
25. [The Curator and the Full Learning Loop](./docs/skills/curator-and-the-learning-loop.md)

### Gateway and platforms
26. [Gateway Routing, Delivery Targets, and Stream Event Vocabulary](./docs/gateway/routing-delivery-and-stream-events.md)
27. [Gateway Authorization, DM Pairing, Slash Commands, and Handoff State](./docs/gateway/authorization-pairing-and-slash-commands.md)

### Security
28. [Security — The OS Boundary, Heuristics, and Isolation Postures](./docs/security/os-boundary-and-isolation-postures.md)

### Extension surfaces
29. [The Plugin System and Observer Hooks (hermes.observer.v1)](./docs/extensions/plugin-system-and-observer-hooks.md)
30. [Middleware — Rewriting Requests and Wrapping Execution (hermes.middleware.v1)](./docs/extensions/middleware.md)
31. [MCP Client Integration and hermes mcp serve](./docs/extensions/mcp-client-and-server.md)
32. [ACP Adapter, IDE Integration, and the Plugin LLM Facade](./docs/extensions/acp-adapter-and-plugin-llm.md)

### Interfaces and deployment
33. [CLI, TUI, Web Dashboard, and Electron Desktop](./docs/interfaces/cli-tui-and-web-dashboard.md)
34. [Terminal Backends and the Mixture-of-Agents Tool](./docs/interfaces/terminal-backends-and-moa.md)

### Observability
35. [Observability — Log Files, Observer Events, and Bundled Consumers](./docs/observability/logs-hooks-and-plugins.md)

### End-to-end scenarios
36. [Scenario 1 — From Conversation to Skill Creation](./docs/scenarios/single-agent-conversation-to-skill.md)
37. [Scenario 2 — Kanban Board Workflow End to End](./docs/scenarios/kanban-board-workflow.md)
38. [Scenario 3 — Multi-Profile Swarm Coordination](./docs/scenarios/multi-profile-swarm.md)
39. [Scenario 4 — Cron Job with Stale-Run Fast-Forward and Platform Delivery](./docs/scenarios/cron-and-webhook-delivery.md)
40. [Scenario 5 — Human-in-the-Loop Approval Workflow](./docs/scenarios/human-in-the-loop-approval.md)

### Design and best practices
41. [Architecture Decisions and Tradeoffs](./docs/design/architecture-decisions-and-tradeoffs.md)
42. [Best Practices — Delegation, Budgets, Credentials, Memory, and Skill Authoring](./docs/design/best-practices.md)

---

Start at [What Hermes Is](./docs/getting-started/what-is-hermes.md) →
