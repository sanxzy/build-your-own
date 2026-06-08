---
title: "What We're Building: The AI Coding Agent Architecture"
description: "A tour of the four-layer architecture behind a production AI coding agent — the LLM abstraction, agent loop, terminal UI, and CLI assembly — and the dependency order we'll follow to build each layer from scratch."
category: getting-started
type: explanation
tags: [architecture, overview, four layers, llm-toolkit, agent-core, terminal-ui, coding-agent, dependency order, monorepo, AI agent, coding agent, building from scratch, system design]
keywords: [architecture, layers, dependency graph, build order, monorepo, packages]
sources: [S1, S2, S3, S5, S19, S34, S44]
---

**TL;DR** — An AI coding agent is a program that uses large language models to read, write, and edit code on your behalf, running in a terminal. We'll build one from scratch across four layers: a unified LLM API (so we can talk to any model), an agent loop (the brain that decides what to do next), a terminal UI (the interface you interact with), and the coding-agent assembly (tools, sessions, and the CLI). By the end of this guide, you'll have a working terminal AI coding agent you built yourself.

## What is an AI coding agent?

An AI coding agent is a program you run in your terminal. You give it a task in plain English — "fix the bug in the login flow" or "add a dark mode toggle to settings" — and it reads your codebase, decides what changes to make, and applies them. It can run shell commands, read and write files, search across your project, and interact with you to clarify requirements.

Under the hood, an AI coding agent is a loop:

1. **You** send a message (the task).
2. The **agent** sends your message — plus the conversation history, plus context about your project — to a large language model.
3. The **LLM** responds. Its response might be a text answer, or it might be a *tool call*: "run this bash command" or "read this file" or "edit that function."
4. If the LLM asked for a tool call, the agent **executes the tool** and feeds the result back to the LLM.
5. The LLM processes the result and either responds with more text, asks for another tool call, or signals it's done.
6. Repeat until the task is complete.

That's the agent loop. It's simple to describe, but building one that works reliably — across different LLM providers, with long conversations that exceed context windows, with a responsive terminal UI, with safe tool execution, and with an extensible plugin system — requires careful architecture.

## The four-layer architecture

Our agent is organized into four layers. Each layer depends only on the one below it, which means we can build and test each one independently before moving to the next.

```
┌────────────────────────────────────┐
│        CODING AGENT (CLI)          │  ← tools, hooks, sessions, CLI
├────────────────────────────────────┤
│        TERMINAL UI (TUI)           │  ← rendering, input, markdown
├────────────────────────────────────┤
│        AGENT CORE                  │  ← the loop, state, compaction
├────────────────────────────────────┤
│        LLM TOOLKIT                 │  ← types, streaming, providers
└────────────────────────────────────┘
```

### Layer 1: LLM Toolkit (`packages/llm-toolkit`)

The foundation. This layer answers one question: **how do we talk to any large language model through a single, unified API?**

The problem is that every LLM provider has a different API. Anthropic's Messages API uses a different request format than OpenAI's Responses API, which is different from Google's Gemini API. They all have different authentication methods (API keys, OAuth), different streaming formats (SSE vs non-SSE), different ways of representing tool calls and thinking blocks, and different model names.

The LLM Toolkit solves this by defining:

- A **unified type system** — one set of types (`Message`, `AssistantMessage`, `Tool`, `Provider`) that every provider adapter maps to and from.
- An **EventStream observable** — every provider streams events through the same Observable interface, so the rest of the system never has to know which provider is on the other end.
- **Provider adapters** — one adapter per provider (Anthropic, OpenAI, Google, and more) that translates between the provider's native format and our unified types.
- An **API registry** — a pluggable registry where providers register themselves. Adding a new provider means writing one adapter and calling `registerProvider()`.
- **Authentication** — unified handling of API keys (from environment variables) and OAuth (device-code flow with PKCE).

By the end of this layer, you'll be able to write:

```ts
const stream = streamSimple({
  provider: "anthropic",
  model: "claude-sonnet-4-6",
  messages: [{ role: "user", content: [{ type: "text", text: "Hello!" }] }],
  apiKey: process.env.ANTHROPIC_API_KEY,
})

for await (const event of stream) {
  // handle text deltas, tool calls, thinking blocks, errors
}
```

And the same code works for OpenAI, Google, or any registered provider — you just change the `provider` and `model` fields.

### Layer 2: Agent Core (`packages/agent-core`)

The brain. This layer answers: **how does the agent decide what to do next, and how does it manage its own state?**

The LLM Toolkit gives us raw streaming — text deltas and tool call blocks. The Agent Core wraps that in a decision loop:

- The **Agent class** is a state machine. It can be `idle`, `running`, or `stopping`. It manages a message queue (so you can send follow-up messages while it's thinking), dispatches events to subscribers (so the UI can react), and handles abort signals.
- The **agent loop** is the turn-by-turn engine. Each turn: build context → stream the LLM → process tool calls → feed results back → repeat. It handles sequential tool execution (one at a time) and parallel execution (all at once), queue draining (when to inject queued messages), and early termination (when to stop before the LLM says so).
- The **Agent Harness** layers production concerns on top: context compaction (what to do when the conversation outgrows the context window), session persistence (saving and loading conversations), system prompt construction, and skill loading.

By the end of this layer, you'll have a working agent loop that can hold a multi-turn conversation with tool execution, persist sessions to disk, and automatically compact when the context window fills up.

### Layer 3: Terminal UI (`packages/tui`)

The interface. This layer answers: **how does the user interact with the agent in a terminal?**

A terminal is a challenging UI environment. There's no DOM, no CSS layout, no mouse events. Everything is a grid of characters, and every update requires careful ANSI escape sequence management to avoid flickering.

The Terminal UI layer provides:

- A **differential render engine** — instead of re-rendering the whole screen, it computes the diff between the previous frame and the next one, then only writes the characters that changed. This eliminates flicker.
- A **component library** — Box (flexbox-inspired layout), Input (multi-line text with cursor, selection, history), Markdown (streaming to terminal with syntax highlighting), Loader (animated spinners), SelectList (interactive selection).
- **Terminal abstraction** — a `Terminal` interface with a real implementation (`ProcessTerminal` over stdin/stdout) and a virtual one for testing.
- **Keybinding system** — configurable key maps with chord sequences and platform-aware key parsing (including the Kitty keyboard protocol for rich key events).
- **Autocomplete** — fuzzy matching with ranked suggestions for slash commands and file paths.

This layer is self-contained — it doesn't know anything about LLMs or agents. It's a general-purpose terminal UI framework, and you can use it for any terminal application.

### Layer 4: Coding Agent (`packages/coding-agent`)

The complete application. This layer answers: **how do we wire everything together into a usable CLI tool?**

The Coding Agent composes the three layers below into a single application:

- **AgentSession** — the shared core that all modes (interactive, print, RPC) build on. It wires the Agent Harness to runtime services (file I/O, subprocess execution), configures model selection and compaction, and owns the session lifecycle.
- **Built-in tools** — Bash (subprocess with PTY, streaming output, timeout), Read (file reading with line ranges), Write (file creation/overwrite), Edit (string-precise patching with diff display), Grep/Glob/Find (code search).
- **Hooks system** — an event pipeline where extensions can intercept every significant event (PreToolUse to block dangerous commands, PostToolUse to modify results, SessionStart/Stop, UserPromptSubmit, PreCompact).
- **System prompt** — the prompt that defines the agent's personality, capabilities, and behavioral rules, assembled from templates with tool descriptions and skill definitions.
- **Three run modes** — Interactive (full TUI chat), Print (single-prompt, non-interactive for scripting), and RPC (JSONL stdio bridge for IDE integrations).
- **Extensions** — a plugin system that lets anyone add custom tools, providers, hooks, slash commands, and multi-agent workflows.

## The build order

We build bottom-up, following the dependency chain:

```
LLM Toolkit  →  Agent Core  →  Coding Agent
                  ↑
            Terminal UI
```

1. **LLM Toolkit** (chapters 3–9). We start here because everything else depends on the streaming API. We define the types, build the EventStream, implement provider adapters, wire up the registry and auth.

2. **Agent Core** (chapters 10–14). With streaming working, we build the agent loop, the Agent class, and the harness with compaction and sessions.

3. **Terminal UI** (chapters 15–18). In parallel with the agent (it has no dependency on it), we build the terminal UI framework. By the end, we have a complete chat interface.

4. **Coding Agent** (chapters 19–28). We compose the agent core and terminal UI, add built-in tools and hooks, implement the three run modes, and wire the CLI entry point.

5. **Extensions** (chapters 29–32). With the full application working, we add the extension system and build multi-agent orchestrations.

## What you'll have at the end

When you finish this guide, you'll have built, from scratch:

- A **unified LLM API** that talks to Anthropic, OpenAI, and Google through one interface
- An **agent loop** with tool execution, state management, and automatic context compaction
- A **terminal UI framework** with differential rendering, markdown, and autocomplete
- A **working CLI coding agent** you can run in your terminal to read, write, edit, and search code
- An **extension system** for adding custom tools, hooks, and multi-agent workflows

Every chapter gives you something runnable. You won't just read about the architecture — you'll build each piece, see it work, and understand why it's shaped the way it is.

---

Next: [Setting Up the Workspace and Toolchain](./workspace-and-toolchain.md) →
