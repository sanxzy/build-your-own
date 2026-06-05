---
title: "What We Are Building: Architecture and the Four Layers"
description: "An orientation to xzy — the terminal coding agent we build across this library — and the four-layer architecture that structures every chapter ahead."
category: getting-started
type: explanation
tags: [architecture, overview, four layers, llm-toolkit, agent-core, tui, coding-agent, agent-loop, terminal-ui, dependency order, monorepo, xzy, ai agent, coding agent, building from scratch]
keywords: [layer dependency graph, build order, workspace packages, llm abstraction, agent runtime, terminal rendering, interactive cli, typescript monorepo]
sources: [S1, S2, S3, S4, S92]
---

**TL;DR** — This chapter orients you to `xzy`, the terminal coding agent we build across this library, and to the four-layer architecture it is made of: `llm-toolkit` (talk to language models), `agent-core` (the agent loop), `tui` (the terminal UI), and `coding-agent` (the full CLI that composes the other three). By the end of this chapter you will have a clear mental map of what we are building, why each layer exists, and the exact order in which we build them.

# What We Are Building: Architecture and the Four Layers

Imagine typing a question into your terminal and watching an AI reason through it, call tools, read and write files, run commands, and reply — all in a structured, extensible loop you wrote yourself. That is `xzy`: a terminal coding agent built from scratch, layer by layer, in plain TypeScript.

This library is a guided build. Every chapter adds one piece to the architecture. Before we write a single line of code, we need a shared map of the whole system. This chapter gives you that map.

## What is a terminal coding agent?

A **coding agent** is a program that accepts a natural-language instruction, sends it to a language model, receives a response that may include *tool calls* (requests to run a specific function — read a file, execute a command, search a directory), executes those tools, feeds the results back to the model, and keeps going until the model decides it is done. The loop is autonomous: the model decides when to call tools and when to stop.

`xzy` runs this loop in your terminal. It is an **interactive command-line interface**: you type a prompt, it thinks, it acts, and it streams its output back to you in a scrollable, keyboard-driven UI. You can also run it in a non-interactive "print" mode for scripting, or drive it programmatically through an RPC interface.

The critical insight is that this is not one big program. It is four independent packages that compose together. Understanding why they are separate — and in what order they depend on each other — is the foundation for everything that follows.

## The four layers

Here is the complete picture, from the bottom up:

```
llm-toolkit          →   talks to language models (OpenAI, Anthropic, Google, …)
     ↓
agent-core           →   runs the agent loop: send prompt, call tools, loop
     ↓                   (depends on llm-toolkit for model calls)
tui              ─┐
                  ↓
coding-agent     →   the full CLI: composes agent-core + tui + everything else
```

A simpler ASCII dependency diagram:

```
llm-toolkit
│
└─► agent-core
│
tui ────────────────────┐
                        │
                        ▼
                  coding-agent
```

`tui` and `agent-core` are both direct dependencies of `coding-agent`, but they do **not** depend on each other — the terminal UI layer has no knowledge of the agent loop, and the agent loop has no knowledge of the terminal. They are kept separate deliberately so that either can be tested, developed, or replaced independently.

Let's walk through each layer.

### Layer 1 — `llm-toolkit`: talking to models

The first thing any coding agent needs is a way to send messages to a language model and get a response back. Different providers — Anthropic, OpenAI, Google — all have different HTTP APIs, authentication schemes, and streaming formats.

`llm-toolkit` is the bottom layer: a unified, multi-provider LLM API. Its job is to hide those provider differences behind a single interface. When `agent-core` wants to call a model, it does not care whether the underlying provider is Anthropic or OpenAI — it speaks to `llm-toolkit` and lets that layer handle the translation.

`llm-toolkit` has **no dependencies on any other layer in this project**. It only knows about language model APIs. This isolation is what makes it safe to test independently and swap out later.

### Layer 2 — `agent-core`: the agent loop

Once we can talk to a model, we need the logic that drives the conversation forward. The agent loop does this:

1. Format the current conversation (system prompt + message history) and send it to `llm-toolkit`.
2. Receive the model's response — which may be plain text, or may include *tool call requests*.
3. If the model requests a tool call, execute that tool and append the result to the message history.
4. Go back to step 1 and send the updated history to the model.
5. When the model produces a final text response with no more tool calls, the loop ends.

`agent-core` (the package that implements this loop) depends on `llm-toolkit` for model calls. It is the brain of the system, but it is not opinionated about how output is displayed — that is the job of `tui`.

### Layer 3 — `tui`: the terminal UI

`agent-core` produces a stream of events — model text chunks, tool call starts, tool results, errors. Something needs to render those events to a terminal in a way that is readable and interactive.

`tui` is a terminal UI library with its own differential rendering system. It handles drawing boxes, scrolling content, capturing keyboard input, and updating only the parts of the terminal that changed. It knows nothing about language models or agent loops — it is a general-purpose terminal rendering layer.

Because `tui` has no dependency on `agent-core` or `llm-toolkit`, it can be developed and tested in complete isolation. A chapter in this library walks you through building it before we wire it to the agent.

### Layer 4 — `coding-agent`: the full CLI

The top layer is `coding-agent`. It composes `agent-core` and `tui` into a complete interactive CLI, and adds everything a real coding agent needs on top: tool definitions (file read/write, shell execution, etc.), session management, a system prompt, slash commands, an extension system, authentication storage, settings, and multiple run modes (interactive terminal, non-interactive print, programmatic RPC).

When you run `xzy` in your terminal, you are running `coding-agent`. It is the only layer that depends on all the others.

## The dependency order is the build order

The four layers form a directed acyclic graph (DAG). Each layer can only be built after its dependencies are built. The root `package.json` build script encodes this explicitly:

```bash
# From the root build script (S3)
cd packages/tui && npm run build
cd ../llm-toolkit && npm run build
cd ../agent-core && npm run build
cd ../coding-agent && npm run build
```

`tui` and `llm-toolkit` have no intra-project dependencies, so they build first (in either order). `agent-core` requires `llm-toolkit` to be built before it. `coding-agent` requires all three. This is not an arbitrary choice — it is the natural dependency graph, and it is exactly the order in which this library teaches.

We build in that order because you cannot understand `agent-core` without first understanding what `llm-toolkit` provides. You cannot understand `coding-agent` without knowing both the agent loop and the terminal layer. Each chapter builds on a stable foundation laid by the previous one.

## The monorepo workspace

The project lives in a TypeScript monorepo managed with npm workspaces. The `package.json` at the repo root declares the workspace members:

```json
{
  "workspaces": [
    "packages/*",
    "packages/coding-agent/examples/extensions/with-deps",
    "packages/coding-agent/examples/extensions/custom-provider-anthropic",
    "packages/coding-agent/examples/extensions/custom-provider-gitlab-duo",
    "packages/coding-agent/examples/extensions/sandbox",
    "packages/coding-agent/examples/extensions/gondolin"
  ]
}
```

The four core packages live under `packages/`:

| Directory | Package name | What it is |
|---|---|---|
| `packages/llm-toolkit/` | `llm-toolkit` | Unified LLM API layer |
| `packages/agent-core/` | `agent-core` | Agent loop and state management |
| `packages/tui/` | `tui` | Terminal UI with differential rendering |
| `packages/coding-agent/` | `coding-agent` | Full interactive coding agent CLI |

All packages share a common TypeScript configuration in `tsconfig.base.json` at the repo root. The shared dialect targets ES2022, uses Node16 module resolution, enables strict type checking, and requires **erasable-only TypeScript syntax** — meaning no `enum`, no `namespace`, no parameter properties, no constructs that require JavaScript emit beyond stripping type annotations. This keeps the code runnable directly with `node --strip-types` or tools like `tsx`, without a full compile step.

## The context-file convention

One detail you will notice as we move through the library: the project uses "context files" — markdown files (`AGENTS.md` at the repo root, and similar files in each package) that describe working conventions, code-quality rules, and tool-usage patterns for both human and AI contributors. We reference these lightly in the library because they capture the design philosophy baked into the codebase. The convention is: read the context files before making broad changes to a package.

## What the library covers

Here is how the four layers map to the library's main sections:

| Library section | Layer | What you build |
|---|---|---|
| llm-toolkit | `llm-toolkit` | Provider abstraction, streaming, model registry, OAuth |
| agent-core | `agent-core` | Agent loop, tool calling, session, compaction |
| tui | `tui` | Terminal rendering, keyboard input, differential updates |
| coding-agent | `coding-agent` | CLI, tools, extensions, session management, run modes |

Each section takes a layer from zero to working, then hands off to the next. By the time you finish, you have a complete terminal coding agent — not a toy, not a wrapper around someone else's SDK, but a system you understand from the ground up.

## Before you continue

This chapter was the map. The next chapter gets your workspace ready: installing Node.js at a recent version (Node 22 or later, declared in the `engines` field in `package.json`), creating the monorepo, and verifying the build from scratch. From there, we start building.

---

Next: [Setting Up the Workspace and Toolchain](./workspace-and-toolchain.md) →
