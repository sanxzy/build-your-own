---
title: Build an AI Coding Agent from Scratch
description: A dependency-ordered, beginner-friendly course that teaches you to build a complete terminal coding agent from the ground up — the LLM layer, the agent loop, the terminal UI, and the full assembled tool.
category: overview
type: explanation
tags: [ai agent, coding agent, build from scratch, llm, streaming, agent loop, tools, terminal ui, sessions, extensions, tutorial, course, index]
keywords: [how to build an ai agent, coding assistant, agent framework, llm toolkit, terminal app, from scratch]
---

**TL;DR** — This library teaches you to build **xzy**, a fully-featured terminal coding agent, one layer at a time. You start by learning how to talk to a language model, then build the loop that turns a model into an agent, then a terminal interface, and finally assemble everything into a real coding tool with tools, sessions, extensions, and multi-agent orchestration. No prior AI experience is assumed — if you can read TypeScript, you can follow along.

## What you'll build

By the end you will have built, from scratch and in dependency order:

- A **unified LLM toolkit** that streams responses from many providers behind one API.
- An **agent loop** that lets a model call tools, with steering, abort, compaction, and persistent sessions.
- A **terminal UI engine** with differential rendering, components, and input handling.
- A **complete coding agent** that reads, writes, and edits files, runs shell commands, branches its history, loads skills, and runs as an interactive app, a one-shot command, an RPC service, or an embeddable SDK.
- An **extension system** and **multi-agent orchestration** on top of it all.

## How to read this

Read the chapters **in order** — each builds on the ones before it. The four layers are taught foundation-first: the LLM toolkit has no dependencies, the agent loop builds on it, the terminal UI stands alone, and the coding agent composes all three. Every chapter is self-contained, so you can also revisit any single topic later.

## The reading path

### 1. Getting started
The map of the whole project and how to set up your workspace.

1. [What We Are Building: Architecture and the Four Layers](docs/getting-started/what-we-are-building.md) — what xzy is, the four layers, and how they depend on each other.
2. [Setting Up the Workspace and Toolchain](docs/getting-started/workspace-and-toolchain.md) — the monorepo layout, shared TypeScript config, and build/test commands.

### 2. The LLM toolkit (foundation)
Everything needed to talk to a language model behind one consistent API.

3. [Message Types and the Unified Streaming API](docs/llm-toolkit/message-types-and-streaming-api.md) — the core data model and the four streaming entry points.
4. [The EventStream Observable Backbone](docs/llm-toolkit/event-stream-and-observable-backbone.md) — the observable that every streamed response flows through.
5. [Provider Adapters: Anthropic and OpenAI](docs/llm-toolkit/provider-adapters-anthropic-and-openai.md) — translating two real wire protocols into our events.
6. [Provider Adapters: Google Gemini and the Faux Test Provider](docs/llm-toolkit/provider-adapters-google-and-faux.md) — a third real provider and a test double for offline development.
7. [The API Registry: Registering and Looking Up Providers](docs/llm-toolkit/api-registry-and-extensibility.md) — finding the right adapter at runtime and adding your own.
8. [Model Metadata, Cost Calculation, and Streaming JSON Parsing](docs/llm-toolkit/models-cost-and-streaming-json.md) — model info, cost, and parsing partial tool-call arguments.
9. [OAuth and API-Key Authentication](docs/llm-toolkit/oauth-and-api-key-auth.md) — device-code/PKCE login and key resolution.

### 3. The agent loop (the brain)
Turning a model into an agent that calls tools, holds state, and manages its own context.

10. [Agent Context, Events, and Types](docs/agent-loop/agent-context-and-types.md) — the type vocabulary the loop operates on.
11. [The Agent Loop: Turn-by-Turn LLM and Tool Execution](docs/agent-loop/the-agent-loop.md) — the heart of the whole library.
12. [The Agent Class: State Machine, Steering, and Lifecycle](docs/agent-loop/the-agent-class.md) — a stateful, subscribable wrapper over the loop.
13. [The Agent Harness: Compaction, Session Storage, and Skills](docs/agent-loop/harness-session-and-compaction.md) — reusable building blocks for long-running assistants.
14. [Cross-Provider Message Transforms and Handoff](docs/agent-loop/cross-provider-message-transforms.md) — switching providers mid-conversation safely.

### 4. The terminal UI (the face)
A flicker-free terminal interface engine and its components.

15. [The TUI Class and Differential Render Engine](docs/terminal-ui/the-tui-class-and-render-engine.md) — updating only what changed on screen.
16. [Terminal Abstraction, Input Handling, and Keybindings](docs/terminal-ui/terminal-abstraction-and-input.md) — a testable terminal and rich key input.
17. [Built-In Components: Widgets, Layout, and ANSI Utilities](docs/terminal-ui/built-in-components-and-layout.md) — the widget toolbox and width-correct text math.
18. [Autocomplete and Building a Complete Chat Interface](docs/terminal-ui/autocomplete-and-a-complete-chat-interface.md) — assembling a working terminal chat app.

### 5. The coding agent (the composition)
Wiring the three layers into a real coding tool with tools, sessions, and run modes.

19. [AgentSession: The Core of the Coding Agent](docs/coding-agent/agent-session-core.md) — the shared hub every run mode is built on.
20. [Built-In Coding Tools: Bash, Read, Write, Edit, and More](docs/coding-agent/built-in-tools.md) — the tools that make it a *coding* agent.
21. [System Prompt Construction and Skill Loading](docs/coding-agent/system-prompt-and-skills.md) — what the agent tells the model, and pluggable skills.
22. [Sessions, Branching, and the Session Tree](docs/coding-agent/sessions-and-branching.md) — durable, branchable conversation history.
23. [Coding-Agent Compaction and Branch Summarization](docs/coding-agent/compaction-and-branch-summarization.md) — keeping long, file-aware sessions within the context window.
24. [Model Registry, Settings, and Resource Loading](docs/coding-agent/model-registry-and-settings.md) — the configuration backbone loaded at startup.
25. [Interactive Mode: Startup, Wiring, and the TUI App Shell](docs/coding-agent/interactive-mode-startup-and-wiring.md) — assembling the full interactive app (part 1).
26. [Interactive Mode: Input, Keyboard Shortcuts, and Session Commands](docs/coding-agent/interactive-mode-input-and-shortcuts.md) — driving the app with the keyboard (part 2).
27. [Print Mode and RPC Mode](docs/coding-agent/print-and-rpc-modes.md) — headless one-shot and machine-readable modes.
28. [The CLI Entry Point: Argument Parsing and Mode Selection](docs/coding-agent/cli-entry-point.md) — where the program starts; full CLI reference.
29. [The SDK: createAgentSession and Programmatic Use](docs/coding-agent/sdk-and-programmatic-use.md) — embedding the agent in your own app.
30. [Package Management and Config Migrations](docs/coding-agent/package-manager-and-migrations.md) — sharing extensions as packages and evolving config.

### 6. Extensions and beyond (extensibility)
Adding capabilities without forking, and composing multiple agents.

31. [The Extension API: Handlers, Context, and Events](docs/extensions/extension-api-and-types.md) — the full extensibility contract and event model.
32. [Loading and Running Extensions: Discovery, Isolation, and Lifecycle](docs/extensions/extension-loader-and-runner.md) — loading TypeScript extensions at runtime and wiring them in.
33. [Building Real Extensions: Tools, State, Commands, and Hooks](docs/extensions/building-real-extensions.md) — building progressively richer real extensions.
34. [Multi-Agent Orchestration](docs/extensions/multi-agent-orchestration.md) — a planner that fans work out to scout, worker, and reviewer agents.

---

Start with [What We Are Building](docs/getting-started/what-we-are-building.md) and work straight down the list.
