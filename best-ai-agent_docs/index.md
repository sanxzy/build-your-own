---
title: "Build the Best AI Coding Agent from Scratch"
description: "A step-by-step, dependency-ordered guide that takes you from zero to a fully-featured terminal AI coding agent — every layer built by you."
---

# Build the Best AI Coding Agent from Scratch

A comprehensive, step-by-step guide that takes you from zero to a fully-featured terminal AI coding agent. You'll build every layer yourself — the unified LLM streaming API, the agent loop with tool execution, context compaction, a terminal UI engine with differential rendering, and the complete CLI — with complete, runnable code at every milestone.

This guide is designed for students and junior developers. Each chapter is self-contained, introduces every concept before using it, and follows a motivated walkthrough shape: problem → solution → next problem. You don't need prior AI/ML experience — just TypeScript and curiosity.

## What you'll build

By the end of this guide, you'll have built a working terminal AI coding agent that can:

- **Talk to any LLM** (Anthropic, OpenAI, Google) through a single unified API
- **Read, write, and edit code** with file-system tools
- **Execute shell commands** and interpret the results
- **Manage long conversations** with automatic context compaction
- **Persist sessions** with branching and forking
- **Run in three modes** — interactive terminal chat, single-prompt scripting, and IDE-integrated RPC
- **Accept extensions** for custom tools, hooks, and multi-agent workflows

## Reading order

Start at Chapter 1 and follow the sequence. Each chapter builds on the previous ones.

### Getting Started
1. [What We're Building: The AI Coding Agent Architecture](./docs/getting-started/what-we-are-building.md) — The four-layer architecture and why each layer exists
2. [Setting Up the Workspace and Toolchain](./docs/getting-started/workspace-and-toolchain.md) — Monorepo, TypeScript, npm workspaces

### LLM Toolkit — The Foundation
3. [Message Types and the Core Streaming API](./docs/llm-toolkit/message-types-and-core-api.md) — The unified type vocabulary
4. [The EventStream: Observable Backbone for Streaming](./docs/llm-toolkit/eventstream-observable-backbone.md) — Async-iterable event queue
5. [Provider Adapter: Anthropic Messages API](./docs/llm-toolkit/provider-adapter-anthropic.md) — SSE parsing, cache-control, thinking
6. [Provider Adapter: OpenAI Responses API](./docs/llm-toolkit/provider-adapter-openai.md) — Reasoning, tool streaming, compatibility
7. [Provider Adapter: Google Gemini](./docs/llm-toolkit/provider-adapter-google.md) — Non-SSE format, schema conversion, grounding
8. [The API Registry and Authentication](./docs/llm-toolkit/api-registry-and-auth.md) — Pluggable providers, OAuth, API keys
9. [Model Registry, Costs, and Streaming JSON](./docs/llm-toolkit/models-and-streaming-json.md) — Pricing, partial JSON parsing

### Agent Core — The Brain
10. [Agent Types, Context, and the Stream Function](./docs/agent-core/agent-types-and-context.md) — The agent's type vocabulary
11. [The Agent Loop: Turn-by-Turn Reasoning and Tool Execution](./docs/agent-core/the-agent-loop.md) — The core turn engine
12. [The Agent Class: State Machine and Lifecycle](./docs/agent-core/the-agent-class.md) — State, subscribers, abort
13. [The Agent Harness: Compaction, Sessions, and Skills](./docs/agent-core/harness-compaction-sessions.md) — Production concerns
14. [Cross-Provider Message Transforms](./docs/agent-core/cross-provider-message-transforms.md) — Multi-model conversations

### Terminal UI — The Interface
15. [The TUI Class and Differential Render Engine](./docs/terminal-ui/tui-class-and-render-engine.md) — Flicker-free rendering
16. [Terminal Abstraction, Input, and Keybindings](./docs/terminal-ui/terminal-abstraction-and-input.md) — Raw I/O and key parsing
17. [Built-In Components: Widgets and Layout](./docs/terminal-ui/components-and-layout.md) — Box, Input, Markdown, Loader
18. [Autocomplete and Assembling a Chat Interface](./docs/terminal-ui/autocomplete-and-chat-interface.md) — Fuzzy matching, full chat UI

### Coding Agent — The Complete Application
19. [AgentSession: The Core of the Coding Agent](./docs/coding-agent/agent-session-core.md) — Runtime services, session lifecycle
20. [Built-In Coding Tools: Bash, Read, Write, Edit, and Search](./docs/coding-agent/built-in-tools.md) — The seven essential tools
21. [The Hooks System: Intercepting Every Agent Event](./docs/coding-agent/hooks-system.md) — Event pipeline for extensions
22. [System Prompt Construction and Skill Loading](./docs/coding-agent/system-prompt-and-skills.md) — Templates and SKILL.md
23. [Sessions, Branching, and the Session Tree](./docs/coding-agent/sessions-and-branching.md) — JSONL persistence, forks
24. [Context Compaction and Branch Summarization](./docs/coding-agent/compaction-and-summarization.md) — Smart cut points, summaries
25. [Model Registry, Settings, and Configuration](./docs/coding-agent/model-registry-and-config.md) — Config layer
26. [Interactive Mode: The Full Terminal Chat Experience](./docs/coding-agent/interactive-mode.md) — Wiring it all together
27. [Print Mode and RPC Mode: Headless Operation](./docs/coding-agent/print-and-rpc-modes.md) — Scripting and IDE integration
28. [The CLI Entry Point: Wiring Everything Together](./docs/coding-agent/cli-entry-point.md) — The binary users run

### Extensions — Going Further
29. [The Extension API: Handlers, Context, and Events](./docs/extensions/extension-api.md) — Plugin interface
30. [Loading and Running Extensions: Discovery, Isolation, and Lifecycle](./docs/extensions/extension-loader.md) — Runtime execution
31. [Building Real Extensions: Tools, State, Commands, and Hooks](./docs/extensions/building-real-extensions.md) — Three real examples
32. [Multi-Agent Orchestration: Fan-Out with Subagents](./docs/extensions/multi-agent-orchestration.md) — Planner, scouts, workers, reviewer

## How to use this guide

- **Read in order.** Each chapter builds on the previous ones. The reading order is dependency-ordered — you won't encounter a concept before it's introduced.
- **Type the code.** Every chapter includes complete, runnable TypeScript. Don't copy-paste — typing builds understanding.
- **Run after each section.** After completing the LLM Toolkit section, you can stream from real LLMs. After Agent Core, you have a working loop. After Terminal UI, you have a chat interface. Each section gives you something tangible.
- **Experiment.** Change things. Add a provider. Build a custom tool. The architecture is designed for extension.
