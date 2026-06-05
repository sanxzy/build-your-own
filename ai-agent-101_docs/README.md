# Build an AI Coding Agent from Scratch

A complete, beginner-friendly course that teaches you to build **xzy** — a fully-featured terminal coding agent — from the ground up, one layer at a time. No prior AI experience required. If you can read TypeScript, you can follow along.

This is a **documentation library**: 34 self-contained, step-by-step chapters arranged as a single guided reading path, from "how do I talk to a language model?" all the way to a real coding agent with tools, sessions, extensions, and multi-agent orchestration.

> Everything here is written to be followed **without any source repository open**. Each chapter introduces every term it uses, shows complete examples, and links back to the concepts it builds on.

---

## Who this is for

- Developers who want to understand how an AI coding agent actually works — not just use one.
- Anyone (students included) who learns best by **building it themselves**, in small motivated steps.
- Engineers evaluating how to embed an agent into their own product.

You need: basic TypeScript, Node.js 22 or later, and curiosity. That's it.

## What you'll build

By the end you will have built, in strict dependency order:

- A **unified LLM toolkit** that streams responses from many providers behind one API.
- An **agent loop** that lets a model call tools, with steering, abort, compaction, and persistent sessions.
- A **terminal UI engine** with flicker-free differential rendering, components, and input handling.
- A **complete coding agent** that reads, writes, and edits files, runs shell commands, branches its history, loads skills, and runs as an interactive app, a one-shot command, an RPC service, or an embeddable SDK.
- An **extension system** and a **multi-agent orchestration** layer on top of it all.

## How to read it

Read the chapters **in order** — each builds on the ones before it. The four layers are taught foundation-first, because that is how they depend on one another:

```
coding-agent      (the full tool — composes everything below)
├── llm-toolkit   (talk to models)        ← no dependencies, taught first
├── agent-core    (the agent loop)        ← builds on llm-toolkit
└── tui           (terminal UI)           ← stands alone
```

Start at **[the landing page](index.md)** for the full linked map, or jump straight into [Chapter 1](docs/getting-started/what-we-are-building.md) and walk down the list below. Every chapter ends with prev/next links so you can read it like a book.

## Table of contents

### 1. Getting started
1. [What We Are Building: Architecture and the Four Layers](docs/getting-started/what-we-are-building.md)
2. [Setting Up the Workspace and Toolchain](docs/getting-started/workspace-and-toolchain.md)

### 2. The LLM toolkit — the foundation
3. [Message Types and the Unified Streaming API](docs/llm-toolkit/message-types-and-streaming-api.md)
4. [The EventStream Observable Backbone](docs/llm-toolkit/event-stream-and-observable-backbone.md)
5. [Provider Adapters: Anthropic and OpenAI](docs/llm-toolkit/provider-adapters-anthropic-and-openai.md)
6. [Provider Adapters: Google Gemini and the Faux Test Provider](docs/llm-toolkit/provider-adapters-google-and-faux.md)
7. [The API Registry: Registering and Looking Up Providers](docs/llm-toolkit/api-registry-and-extensibility.md)
8. [Model Metadata, Cost Calculation, and Streaming JSON Parsing](docs/llm-toolkit/models-cost-and-streaming-json.md)
9. [OAuth and API-Key Authentication](docs/llm-toolkit/oauth-and-api-key-auth.md)

### 3. The agent loop — the brain
10. [Agent Context, Events, and Types](docs/agent-loop/agent-context-and-types.md)
11. [The Agent Loop: Turn-by-Turn LLM and Tool Execution](docs/agent-loop/the-agent-loop.md)
12. [The Agent Class: State Machine, Steering, and Lifecycle](docs/agent-loop/the-agent-class.md)
13. [The Agent Harness: Compaction, Session Storage, and Skills](docs/agent-loop/harness-session-and-compaction.md)
14. [Cross-Provider Message Transforms and Handoff](docs/agent-loop/cross-provider-message-transforms.md)

### 4. The terminal UI — the face
15. [The TUI Class and Differential Render Engine](docs/terminal-ui/the-tui-class-and-render-engine.md)
16. [Terminal Abstraction, Input Handling, and Keybindings](docs/terminal-ui/terminal-abstraction-and-input.md)
17. [Built-In Components: Widgets, Layout, and ANSI Utilities](docs/terminal-ui/built-in-components-and-layout.md)
18. [Autocomplete and Building a Complete Chat Interface](docs/terminal-ui/autocomplete-and-a-complete-chat-interface.md)

### 5. The coding agent — the composition
19. [AgentSession: The Core of the Coding Agent](docs/coding-agent/agent-session-core.md)
20. [Built-In Coding Tools: Bash, Read, Write, Edit, and More](docs/coding-agent/built-in-tools.md)
21. [System Prompt Construction and Skill Loading](docs/coding-agent/system-prompt-and-skills.md)
22. [Sessions, Branching, and the Session Tree](docs/coding-agent/sessions-and-branching.md)
23. [Coding-Agent Compaction and Branch Summarization](docs/coding-agent/compaction-and-branch-summarization.md)
24. [Model Registry, Settings, and Resource Loading](docs/coding-agent/model-registry-and-settings.md)
25. [Interactive Mode: Startup, Wiring, and the TUI App Shell](docs/coding-agent/interactive-mode-startup-and-wiring.md)
26. [Interactive Mode: Input, Keyboard Shortcuts, and Session Commands](docs/coding-agent/interactive-mode-input-and-shortcuts.md)
27. [Print Mode and RPC Mode](docs/coding-agent/print-and-rpc-modes.md)
28. [The CLI Entry Point: Argument Parsing and Mode Selection](docs/coding-agent/cli-entry-point.md)
29. [The SDK: createAgentSession and Programmatic Use](docs/coding-agent/sdk-and-programmatic-use.md)
30. [Package Management and Config Migrations](docs/coding-agent/package-manager-and-migrations.md)

### 6. Extensions and beyond
31. [The Extension API: Handlers, Context, and Events](docs/extensions/extension-api-and-types.md)
32. [Loading and Running Extensions: Discovery, Isolation, and Lifecycle](docs/extensions/extension-loader-and-runner.md)
33. [Building Real Extensions: Tools, State, Commands, and Hooks](docs/extensions/building-real-extensions.md)
34. [Multi-Agent Orchestration](docs/extensions/multi-agent-orchestration.md)

## Repository structure

```
.
├── README.md          ← you are here
├── index.md           ← the rendered-site landing page (same reading map)
└── docs/
    ├── getting-started/   (2 chapters)
    ├── llm-toolkit/       (7 chapters)
    ├── agent-loop/        (5 chapters)
    ├── terminal-ui/       (4 chapters)
    ├── coding-agent/      (12 chapters)
    └── extensions/        (4 chapters)
```

## Format and conventions

- **Plain Markdown.** Every page is portable GitHub-Flavored Markdown — it renders as-is on GitHub, or in Obsidian, MkDocs, Docusaurus, or Astro Starlight with no changes.
- **Self-contained chapters.** Symbols are introduced on first use; prerequisites get a short recap plus a link rather than assuming you read the sibling page.
- **Motivated walkthroughs.** Each step starts from the problem it solves, then arrives at the solution — not a flat "here is the finished code" dump.
- **Brand-agnostic.** The agent is referred to generically as `xzy`, and the four packages as `llm-toolkit`, `agent-core`, `tui`, and `coding-agent`. No product brand names appear anywhere.
- **Version-agnostic.** Example `package.json` files use minimal starting versions (e.g. `^0.1.0`) — pick whatever versions you like; nothing here depends on a specific one.

## How to use this library

- **Read it on GitHub** — every link is relative, so it browses naturally.
- **Render it as a site** — point MkDocs, Docusaurus, or Astro Starlight at the `docs/` folder and use `index.md` as the home page.
- **Read it locally** — open `index.md` (or this README) in any Markdown viewer and follow the links.

Begin with **[What We Are Building](docs/getting-started/what-we-are-building.md)** and work straight down the list. Happy building.
