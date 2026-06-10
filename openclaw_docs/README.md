# OpenClaw Documentation Library

A complete, build-it-yourself guide to **OpenClaw** — a self-hosted, multi-channel AI gateway. One agent runtime, connected to the messaging surfaces people already use (Telegram, Discord, Slack, WhatsApp, Signal, iMessage), behind a single control-plane gateway you run and own.

- **New here?** Start with the guided reading map in **[index.md](index.md)** — 27 chapters arranged as a dependency-ordered spine, foundations first.
- **Looking something up?** Use the table of contents below or the **[Glossary](docs/reference/27-glossary.md)**.

Every technical claim traces to OpenClaw's own source; where the source is silent, the text says so.

## Table of contents

### Getting started
- [01 · Introduction: What OpenClaw Is and Why It Exists](docs/getting-started/01-introduction.md)
- [02 · High-Level Architecture: Four Layers and the Gateway Control Plane](docs/getting-started/02-architecture.md)

### Gateway
- [03 · The Gateway: Port 18789, Wire Protocol, and Node Pairing](docs/gateway/03-gateway.md)

### Channels
- [04 · Channels: Message Surfaces, Session Grammar, and DM Pairing](docs/channels/04-channels.md)

### Agents
- [05 · Agents: Workspace, Bootstrap Files, and Harness Types](docs/agents/05-agents.md)
- [06 · The Agent Loop: Six Stages from Intake to Persistence](docs/agents/06-agent-loop.md)
- [07 · Sessions: Routing, Lifecycle, dmScope, and JSONL Persistence](docs/agents/07-sessions.md)
- [08 · Run Queue and Concurrency: Session Lanes, Queue Modes, and maxConcurrent](docs/agents/08-run-queue.md)
- [09 · System Prompt and Context: Assembly, Bootstrap Injection, and Compaction](docs/agents/09-system-prompt.md)

### Memory
- [10 · Memory System: File Memory, memory-core, memory-lancedb, and memory-wiki](docs/memory/10-memory-system.md)

### Extending OpenClaw
- [11 · Plugins, Skills, and Tools: Three Distinct Primitives](docs/extending/11-plugins-skills-tools.md)
- [12 · Tool System: Registration, Effective Policy, and Built-in Categories](docs/extending/12-tool-system.md)
- [13 · Skills: SKILL.md Structure, Loading Precedence, and Token Cost](docs/extending/13-skills.md)
- [14 · Agent Loop Hooks: Inventory, Priority, and before_tool_call in Depth](docs/extending/14-hooks.md)

### Models
- [15 · AI Model Integration: Provider Refs, Fallback Chains, and ThinkingLevel](docs/models/15-model-integration.md)

### Coordination
- [16 · Multi-Agent Coordination: Bindings, Specificity Rules, and Subagent Calls](docs/coordination/16-multi-agent.md)

### Automation
- [17 · Automation and Scheduling: Cron, Heartbeat, and Dreaming](docs/automation/17-automation.md)

### Operations
- [18 · Configuration System: openclaw.json, Zod Validation, and Hot Reload](docs/operations/18-configuration.md)
- [19 · Storage and Persistence: SQLite, JSONL Sessions, and the Workspace](docs/operations/19-storage.md)
- [20 · Security and Governance: Pairing, Auth Modes, Sandbox, and Network Policy](docs/operations/20-security.md)
- [21 · Monitoring and Observability: Logs, Debug Flags, OTel, Prometheus, Health Endpoints](docs/operations/21-observability.md)

### Reference
- [22 · Project Structure: Monorepo Layout, Packages, and Extension Inventory](docs/reference/22-project-structure.md)
- [23 · Deployment and Lifecycle: Install, Daemon Setup, Docker, and Hosted Options](docs/reference/23-deployment.md)
- [24 · End-to-End Walkthroughs: DM Conversation, Cron Run, and Subagent Coordination](docs/reference/24-walkthroughs.md)
- [25 · Design Decisions and Tradeoffs: SQLite, Exclusive Memory Slot, Loopback, In-Process Plugins](docs/reference/25-design-decisions.md)
- [26 · Best Practices: Configuration, Security, Tool Policy, Sessions, Observability](docs/reference/26-best-practices.md)
- [27 · Glossary: OpenClaw Vocabulary Reference](docs/reference/27-glossary.md)

---

A top-level [GLOSSARY.md](GLOSSARY.md) mirrors chapter 27 for quick lookup.
