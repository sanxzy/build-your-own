# OpenClaw Documentation Library

A guided, build-it-yourself library for **OpenClaw** — the self-hosted, multi-channel AI gateway you run on your own machine. It takes one agent runtime and connects it to the messaging surfaces people already use (Telegram, Discord, Slack, WhatsApp, Signal, iMessage), behind a single control-plane gateway you own end to end.

This library is meant to be **read in order**. Each chapter is self-contained — every term is introduced where it first appears — but the chapters are arranged as a dependency-ordered spine: foundations first, then the layers that build on them. If you read straight through, no chapter ever leans on an idea you have not met yet. If you are here to look something up, jump to the chapter you need; the [Glossary](docs/reference/27-glossary.md) and the per-chapter recaps will catch you up.

Every technical claim in this library traces to OpenClaw's own source. Where the source is silent on a detail, the text says so rather than guessing.

## How to read this library

The reading order is **P1 → P27**. The chapters are grouped below into nine parts that follow the build: you stand up the gateway, attach channels, understand how an agent thinks and remembers, learn to extend it, then operate and harden it for real use.

### Part 1 — Orientation

1. [Introduction: What OpenClaw Is and Why It Exists](docs/getting-started/01-introduction.md) — the problem OpenClaw solves and a map of the territory.
2. [High-Level Architecture: Four Layers and the Gateway Control Plane](docs/getting-started/02-architecture.md) — the four layers (provider, model, agent runtime, channel) and how the gateway ties them together.

### Part 2 — The Gateway and Channels

3. [The Gateway: Port 18789, Wire Protocol, and Node Pairing](docs/gateway/03-gateway.md) — the single control-plane port, the connect/req/res/event wire protocol, and how nodes pair.
4. [Channels: Message Surfaces, Session Grammar, and DM Pairing](docs/channels/04-channels.md) — how each messaging surface maps onto sessions, and how direct-message pairing gates access.

### Part 3 — Agents and the Run Model

5. [Agents: Workspace, Bootstrap Files, and Harness Types](docs/agents/05-agents.md) — what an agent is: an isolated runtime with its own workspace and bootstrap files.
6. [The Agent Loop: Six Stages from Intake to Persistence](docs/agents/06-agent-loop.md) — the six stages a single turn passes through, from message intake to persisted transcript.
7. [Sessions: Routing, Lifecycle, dmScope, and JSONL Persistence](docs/agents/07-sessions.md) — how messages route to a session, the session lifecycle, and the JSONL transcript.
8. [Run Queue and Concurrency: Session Lanes, Queue Modes, and maxConcurrent](docs/agents/08-run-queue.md) — how concurrent work is serialized per session and bounded across the gateway.
9. [System Prompt and Context: Assembly, Bootstrap Injection, and Compaction](docs/agents/09-system-prompt.md) — how the prompt is assembled each turn, how bootstrap files are injected, and how long contexts are compacted.

### Part 4 — Memory

10. [Memory System: File Memory, memory-core, memory-lancedb, and memory-wiki](docs/memory/10-memory-system.md) — how an agent remembers across sessions, and the memory plugins that back it.

### Part 5 — Extending OpenClaw

11. [Plugins, Skills, and Tools: Three Distinct Primitives](docs/extending/11-plugins-skills-tools.md) — the three extension primitives and how they differ.
12. [Tool System: Registration, Effective Policy, and Built-in Categories](docs/extending/12-tool-system.md) — how tools register, how the effective allow/deny policy resolves, and the built-in categories.
13. [Skills: SKILL.md Structure, Loading Precedence, and Token Cost](docs/extending/13-skills.md) — how skills are structured, discovered, and loaded, and what they cost in context.
14. [Agent Loop Hooks: Inventory, Priority, and before_tool_call in Depth](docs/extending/14-hooks.md) — the hook points the loop exposes, how priority orders them, and `before_tool_call` in depth.

### Part 6 — Models

15. [AI Model Integration: Provider Refs, Fallback Chains, and ThinkingLevel](docs/models/15-model-integration.md) — how providers and models are referenced, how fallback chains work, and how thinking level is set.

### Part 7 — Coordination and Automation

16. [Multi-Agent Coordination: Bindings, Specificity Rules, and Subagent Calls](docs/coordination/16-multi-agent.md) — how messages route to agents by binding specificity, and how agents call subagents.
17. [Automation and Scheduling: Cron, Heartbeat, and Dreaming](docs/automation/17-automation.md) — scheduled runs, the heartbeat, and overnight memory consolidation.

### Part 8 — Operations

18. [Configuration System: openclaw.json, Zod Validation, and Hot Reload](docs/operations/18-configuration.md) — the configuration file, how it is validated, and how it reloads.
19. [Storage and Persistence: SQLite, JSONL Sessions, and the Workspace](docs/operations/19-storage.md) — what lives in SQLite (runtime state), what lives in JSONL (transcripts), and what lives in the workspace.
20. [Security and Governance: Pairing, Auth Modes, Sandbox, and Network Policy](docs/operations/20-security.md) — pairing, authentication modes, the sandbox, and network exposure policy.
21. [Monitoring and Observability: Logs, Debug Flags, OTel, Prometheus, Health Endpoints](docs/operations/21-observability.md) — every observation surface, from a log file up to OpenTelemetry export.

### Part 9 — Reference

22. [Project Structure: Monorepo Layout, Packages, and Extension Inventory](docs/reference/22-project-structure.md) — how the monorepo is laid out and what each package and extension does.
23. [Deployment and Lifecycle: Install, Daemon Setup, Docker, and Hosted Options](docs/reference/23-deployment.md) — installing, running as a daemon, Docker, and hosted deployment options.
24. [End-to-End Walkthroughs: DM Conversation, Cron Run, and Subagent Coordination](docs/reference/24-walkthroughs.md) — three complete traces that connect every earlier chapter.
25. [Design Decisions and Tradeoffs: SQLite, Exclusive Memory Slot, Loopback, In-Process Plugins](docs/reference/25-design-decisions.md) — the reasoning behind the load-bearing choices, and what each one trades away.
26. [Best Practices: Configuration, Security, Tool Policy, Sessions, Observability](docs/reference/26-best-practices.md) — field-tested operating guidance, organized by area.
27. [Glossary: OpenClaw Vocabulary Reference](docs/reference/27-glossary.md) — an alphabetized reference of OpenClaw vocabulary.

## A note on scope

This library documents OpenClaw's behavior as described in its own source. A few areas are deliberately out of depth because the source material does not cover them in enough detail to document faithfully: the ACP wire-format internals; provider streaming wire formats below the hook level; the thin support packages (`tool-call-repair`, `speech-core`, `media-core`, `web-content-core`, `terminal-core`, `markdown-core`, `normalization-core`, `model-catalog-core`); the iOS and macOS Swift clients under `apps/`; and backup/recovery procedures beyond the documented `openclaw backup` command. Where a chapter reaches one of these edges, it says so rather than inventing the missing detail.

Start at [Introduction](docs/getting-started/01-introduction.md) →
