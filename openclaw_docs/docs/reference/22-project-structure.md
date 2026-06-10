---
title: "Project Structure: Monorepo Layout, Packages, and Extension Inventory"
description: Reference map of the OpenClaw monorepo — top-level directories, library packages with one-line responsibilities, and the extension inventory.
category: reference
type: reference
tags: [project structure, monorepo, packages, extensions, agent-core, llm-core, llm-runtime, gateway-client, gateway-protocol, acp-core, sdk, net-policy, plugin-sdk, memory-host-sdk, skills directory, apps, deploy, src, ui, extensions, tool-call-repair, speech-core, media-core, web-content-core, terminal-core, markdown-core, normalization-core, model-catalog-core, plugin boundary, package layout, pnpm workspace]
keywords: [openclaw monorepo, package map, workspace layout, plugin-sdk boundary, library packages, extension directory, bundled extensions, bundled skills, thin coverage packages]
sources: [S82, S8, S10, S2, S113, S100, S105]
---

**TL;DR** — OpenClaw is organized as a single pnpm workspace. The runtime TypeScript source lives in `src/`, the web Control UI in `ui/`, reusable library packages in `packages/`, plugins (bundled and community) in `extensions/`, and the 58 bundled skills in `skills/`. There are 21 library packages in `packages/` with distinct responsibilities; eight of them have thin source coverage in the documentation and are explicitly flagged below. After reading this chapter you will be able to navigate the repo, know which package to look at for any given concern, and understand the hard boundary a plugin author must never cross.

# Project Structure: Monorepo Layout, Packages, and Extension Inventory

The architecture of OpenClaw — the four layers of Provider → Model → Agent runtime → Channel — maps onto a specific directory layout (see [High-Level Architecture](../getting-started/02-architecture.md) for the layer model). This chapter gives you the reference map: where each layer lives on disk, what every package in `packages/` is responsible for, and what the `extensions/` and `skills/` directories contain.

> Think of the repo like a city: `src/` is the city core (roads, power, governance), `packages/` is the shared utility infrastructure (water treatment, telecoms), `extensions/` is the set of businesses and shops that run on top of that infrastructure, and `skills/` is the employee handbook distributed to every store.

We will work outward from the workspace root, then zoom in on the packages and extensions.

---

## Top-Level Directory Map

The workspace root is defined by `pnpm-workspace.yaml` (S10), which lists four workspace members: `.` (the root package), `ui`, `packages/*`, and `extensions/*`.

```
openclaw/
├── src/           ← Core TypeScript runtime (agents, gateway, channels, LLM, CLI, config…)
├── ui/            ← Web Control UI (Vite app; package name: openclaw-control-ui)
├── packages/      ← 21 shared library packages (@openclaw/*)
├── extensions/    ← Plugins — bundled and community-style (each is a workspace package)
├── skills/        ← 58 bundled SKILL.md instruction packs
├── apps/          ← Native companion clients (iOS, macOS, Android, shared; thin coverage)
├── deploy/        ← Hosting configuration files (Fly.io private config)
├── docs/          ← Source documentation (published to docs.openclaw.ai)
├── security/      ← OpenGrep static-analysis rules for security checks
├── scripts/       ← Build, packaging, install, and maintenance scripts
├── test/          ← Cross-cutting test helpers
└── config/        ← Shared configuration baseline files
```

| Directory | What lives here | Notes |
|-----------|----------------|-------|
| `src/` | Core TS runtime | Agents, Gateway, channels, LLM/provider registry, CLI, cron, hooks, sessions, config — see below |
| `ui/` | Web Control UI | Vite + React; connects to the Gateway over WebSocket |
| `packages/` | 21 library packages | `@openclaw/*` npm scoped; consumed by `src/` and `extensions/` |
| `extensions/` | Plugins | Both bundled official plugins and third-party-style plugins; each has `openclaw.plugin.json` |
| `skills/` | 58 bundled SKILL.md packs | Injected into the agent system prompt as compact XML instruction blocks |
| `apps/` | Native clients | iOS, macOS, Android, shared SDK — Swift/Kotlin/cross-platform; thin source coverage |
| `deploy/` | Hosting configs | Fly.io private config (`fly.private.toml`); root also has `Dockerfile`, `docker-compose.yml`, `fly.toml`, `render.yaml` |
| `docs/` | Source documentation | Mintlify-hosted; synced to `openclaw/docs` publish repo |
| `security/` | OpenGrep rules | Static-analysis security checks compiled from `security/opengrep/` |

---

## The `src/` Core Runtime

`src/` is the monorepo root's own TypeScript source — the Gateway process itself, not a library. It is not a separately publishable package; it is the product's runtime entry point.

Key subdirectories (from S82):

| `src/` subdirectory | Role |
|--------------------|------|
| `src/agents/` | Agent loop, session management, tool definitions, hooks, runtime facade over `@openclaw/agent-core` |
| `src/agents/embedded-agent-runner/` | Built-in attempt loop, provider stream adapters, compaction, model selection, session wiring |
| `src/agents/sessions/` | Session persistence, extension loading, resource discovery, skills, prompts, themes, TUI-backed tool renderers |
| `src/agents/runtime/` | OpenClaw facade that wraps `@openclaw/agent-core` for the local runtime context |
| `src/agents/agent-tools*.ts` | OpenClaw-owned tool definitions, schemas, tool policy, before/after hook adapters, host edit support |
| `src/agents/agent-hooks/` | Built-in runtime hooks: compaction safeguards, context pruning |
| `src/gateway/` | Gateway process: WebSocket RPC server, HTTP routes, auth, node pairing |
| `src/channels/` | Channel implementations (Telegram, Discord, Slack, WhatsApp, Signal, iMessage…) |
| `src/llm/` | Model/provider registry, transport helpers, provider-specific stream implementations |
| `src/config/` | Configuration loading, Zod validation, hot-reload orchestration |
| `src/cron/` | Cron job scheduler runtime |
| `src/hooks/` | Hook dispatch runtime |
| `src/sessions/` | Session routing and lifecycle |
| `src/plugin-sdk/` | The internal implementation that backs the public `openclaw/plugin-sdk/*` barrels |

The `src/agents/templates/` subdirectory holds the default workspace-file templates (`AGENTS.md`, `SOUL.md`, etc.) that are copied when a new agent is created.

---

## The `packages/` Library Packages

`packages/` contains **21 library packages**, all scoped `@openclaw/*`. These are the shared building blocks consumed by the core runtime (`src/`), the web UI (`ui/`), and extensions. Below we give each package a one-line responsibility; packages marked **[thin coverage]** have limited internal documentation in the source — their existence and API surface are confirmed, but their internals are not fully documented here.

> Think of `packages/` as a set of tightly-focused Lego bricks. Each brick does one job (e.g., "parse LLM response types" or "enforce network policy"). The core runtime (`src/`) and plugins assemble them without those bricks knowing about each other.

### Core Agent and Model Packages

| Package | `@openclaw/` name | One-line responsibility |
|---------|------------------|------------------------|
| `agent-core` | `agent-core` | Reusable agent loop, harness types, message types, compaction helpers, prompt templates, and tool/session contracts |
| `llm-core` | `llm-core` | Shared TypeScript contracts for the LLM layer: `KnownApi` wire formats, `ThinkingLevel`, `CacheRetention`, `Transport`, streaming types, and provider type definitions |
| `llm-runtime` | `llm-runtime` | Runtime API registry (`api-registry.ts`) that maps wire-format IDs to streaming adapter functions; provider plugins register into this registry |

**`llm-core` vs `llm-runtime` — what is the difference?**

This is the question that trips up most readers navigating the codebase. Here is the line:

- `llm-core` is **types-only**. It defines the contracts — the TypeScript interfaces and type aliases (`KnownApi`, `Model`, `Context`, `StreamOptions`, `ThinkingLevel`) — but contains no runtime code that executes API calls.
- `llm-runtime` is **the registry that makes those types concrete**. It provides `ApiProvider` and `apiProviderRegistry`, which map an API wire-format ID (e.g. `"anthropic-messages"`) to the actual streaming adapter function that runs at call time. Provider plugins call into `llm-runtime` to register themselves; the agent loop calls into `llm-runtime` to look up a stream function at runtime.

If you are writing a provider plugin, you import types from `llm-core` and register your implementation into the registry from `llm-runtime`. If you are reading code that does model type-checking, you are in `llm-core`. If you are reading code that dispatches actual model calls, you are in `llm-runtime`.

### Gateway and Protocol Packages

| Package | `@openclaw/` name | One-line responsibility |
|---------|------------------|------------------------|
| `gateway-protocol` | `gateway-protocol` | Wire-protocol TypeBox schemas for WebSocket frames (`connect`, `req`, `res`, `event`), method/event payloads, error-detail codes, protocol version constants, and client-info contracts |
| `gateway-client` | `gateway-client` | WebSocket client implementation for connecting to a Gateway: connect handshake, device-auth, frame send/receive, protocol versioning, and client-mode negotiation |
| `acp-core` | `acp-core` | ACP (Agent Communication Protocol) session contracts: in-memory session store (`createInMemorySessionStore`), session-interaction modes, session lineage metadata, and text normalization helpers |
| `sdk` | `sdk` | High-level OpenClaw SDK for external clients: `OpenClaw` client class, namespaced APIs (`AgentsNamespace`, `SessionsNamespace`, `RunsNamespace`, `ModelsNamespace`, etc.), `EventHub`, and stable SDK types |

**`gateway-client` vs `sdk` — two layers of the same connection**

- `gateway-client` is the low-level WebSocket transport layer. It handles connect frames, device auth, timeouts, and the raw frame loop. A plugin or native app that wants full control over the connection uses this.
- `sdk` is the high-level developer API. It wraps `gateway-client` inside organized namespaces so you can write `openclaw.agents.run(...)` instead of manually crafting RPC frames. Most external integrations use the SDK, not the raw gateway client.

### Plugin and Extension Surface Packages

| Package | `@openclaw/` name | One-line responsibility |
|---------|------------------|------------------------|
| `plugin-sdk` | `plugin-sdk` | Public plugin-facing API surface: `plugin-entry.ts`, `plugin-runtime.ts`, provider auth/stream/tools contracts, exec-approvals runtime, security runtime, config runtime, and all other `openclaw/plugin-sdk/*` barrel entrypoints |
| `memory-host-sdk` | `memory-host-sdk` | Host-side memory engine contracts used by memory plugins: `MemoryEngine`, query, storage, embeddings, QMD interfaces, and runtime config/path helpers |
| `plugin-package-contract` | `plugin-package-contract` | Validation contracts for external code plugin `package.json`: compatibility metadata extraction, validation issue types, and required field paths (`openclaw.compat.pluginApi`, `openclaw.build.openclawVersion`) |

### Network, Model, and Utility Packages

| Package | `@openclaw/` name | One-line responsibility |
|---------|------------------|------------------------|
| `net-policy` | `net-policy` | Network policy enforcement: IP address parsing, SSRF-blocked ranges (loopback, private, link-local, broadcast, multicast, carrier-grade NAT, and reserved IPv4/IPv6 ranges), URL userinfo redaction |
| `model-catalog-core` | `model-catalog-core` | Model catalog data contracts: `ModelCatalogApi`, `ModelCatalogThinkingFormat`, `ModelCatalogCompatConfig`; provider/model ID normalization and configured model ref helpers |

### Thin-Coverage Packages

The following eight packages exist and have confirmed source structure, but their internal implementations are not deeply documented here. Each package's name and inferred role are stated; deeper internals should be verified from the source code directly.

| Package | `@openclaw/` name | Inferred role | Coverage note |
|---------|------------------|--------------|---------------|
| `tool-call-repair` | `tool-call-repair` | Repairs malformed tool-call payloads from model output (`grammar.ts`, `stream-normalizer.ts`, `promote.ts`) | **[thin source coverage; internals not documented here]** |
| `speech-core` | `speech-core` | Text-to-speech runtime contracts (`tts.ts`) | **[thin source coverage; internals not documented here]** |
| `media-core` | `media-core` | Media handling utilities: base64, MIME constants, content-length, file naming, inbound path policy | **[thin source coverage; internals not documented here]** |
| `web-content-core` | `web-content-core` | Shared provider runtime helpers for web-content fetching/processing | **[thin source coverage; internals not documented here]** |
| `terminal-core` | `terminal-core` | Terminal rendering utilities: ANSI processing, display-string helpers, link formatting, health-style display | **[thin source coverage; internals not documented here]** |
| `markdown-core` | `markdown-core` | Markdown processing: text chunking, fences, code-spans, frontmatter parsing, IR-level block transformations | **[thin source coverage; internals not documented here]** |
| `normalization-core` | `normalization-core` | Data normalization helpers: string normalization/coercion, record coercion, number coercion | **[thin source coverage; internals not documented here]** |
| `model-catalog-core` | `model-catalog-core` | (listed above under Network/Utility) | See table above |
| `media-generation-core` | `media-generation-core` | Model-ref and catalog contracts for image/video generation providers | **[thin source coverage; internals not documented here]** |
| `media-understanding-common` | `media-understanding-common` | Shared contracts for multimodal understanding: format, provider IDs, output extraction, OpenAI-compatible video | **[thin source coverage; internals not documented here]** |

> **Verified package count:** The `packages/` directory contains **21 folders**: `acp-core`, `agent-core`, `gateway-client`, `gateway-protocol`, `llm-core`, `llm-runtime`, `markdown-core`, `media-core`, `media-generation-core`, `media-understanding-common`, `memory-host-sdk`, `model-catalog-core`, `net-policy`, `normalization-core`, `plugin-package-contract`, `plugin-sdk`, `sdk`, `speech-core`, `terminal-core`, `tool-call-repair`, `web-content-core`. All 21 are confirmed by direct directory listing.

---

## The Plugin Boundary: What a Plugin Author Imports (and Must Not)

Now that we have the package map, we can state the boundary rule precisely. This is documented in S82 (agent-runtime-architecture.md) and S2 (VISION.md), and the extensions boundary document:

**A plugin author imports from `openclaw/plugin-sdk/*` barrels only.** The `packages/plugin-sdk/` package re-exports from the internal `src/plugin-sdk/` implementation; this is the stable contract surface.

```ts
// Correct — use the public plugin-sdk barrels
import type { PluginEntry } from "openclaw/plugin-sdk"
import { pluginEntry } from "openclaw/plugin-sdk/core"
import { registerProvider } from "openclaw/plugin-sdk/provider-entry"
```

**A plugin author must never import from:**

- `src/**` — the core runtime internals
- `src/plugin-sdk-internal/**` — internal helpers not exported through the SDK
- Another extension's `src/**` — extension internals are private to that extension
- Deep paths inside `packages/*/src/` — use the public barrel (`openclaw/plugin-sdk/*`), not the package's internal source

> The analogy: `packages/plugin-sdk/` is the electrical outlet in the wall. Your appliance (plugin) plugs into the outlet. You do not wire your appliance directly into the breaker box (`src/`).

This boundary matters for two reasons stated in S2 and S82: (1) core stays plugin-agnostic so it can evolve without breaking published plugins, and (2) plugins run in-process with the Gateway, so an accidental import of a core internal can trigger side effects or access state it was never meant to touch.

---

## `extensions/` vs `packages/`

A reader navigating the repo for the first time sometimes wonders: both directories have `@openclaw/*`-scoped packages — what is the difference?

| Dimension | `packages/` | `extensions/` |
|-----------|-------------|--------------|
| What it is | Shared library packages, no `openclaw.plugin.json` | Plugin packages, each with `openclaw.plugin.json` |
| Who consumes it | `src/`, `ui/`, and extensions import these libraries | The Gateway loads these as plugins at runtime |
| Plugin contract | None — these are pure libraries | Each declares `id`, `activation`, `configSchema`, and optionally `contracts.tools`, `providers`, `channels` |
| Discovery | Compiled into the core dist | Discovered by the plugin loader via `openclaw.plugin.json`; some bundled into dist, others installed externally |
| Examples | `@openclaw/agent-core`, `@openclaw/llm-core`, `@openclaw/net-policy` | `memory-core`, `diagnostics-otel`, `diagnostics-prometheus`, `discord`, `telegram`, `memory-lancedb` |

The extensions directory contains both **bundled official plugins** (shipped inside the OpenClaw dist, such as memory and diagnostics plugins) and **community-style plugins** (installed separately, e.g. `memory-lancedb`). Bundled plugins are excluded from the dist package via the root `package.json` `!dist/extensions/<name>/**` patterns (S8), confirming they ship as source inside the monorepo but some are excluded from the published npm bundle and loaded via the installed plugin path instead.

---

## Bundled Extension Inventory

The `extensions/` directory holds a large number of plugins (confirmed by directory listing from S8 and the dist exclude list). Below is a categorized view of the plugins with confirmed `openclaw.plugin.json` presence or clear naming:

### Memory Plugins (exclusive slot — only one active at a time)

| Plugin ID | Location | Tools registered | Notes |
|-----------|----------|-----------------|-------|
| `memory-core` | `extensions/memory-core/` | `memory_get`, `memory_search` | Bundled default; includes dreaming subagent on cron |
| `memory-lancedb` | `extensions/memory-lancedb/` | `memory_store`, `memory_recall`, `memory_forget` | External; LanceDB vector store |
| `memory-wiki` | `extensions/memory-wiki/` | `wiki_search`, `wiki_get` | Bundled; compiled knowledge vault |
| `active-memory` | `extensions/active-memory/` | — | Runs blocking pre-reply memory subagent |

### Diagnostics Plugins

| Plugin ID | Location | Purpose |
|-----------|----------|---------|
| `diagnostics-otel` | `extensions/diagnostics-otel/` | OpenTelemetry metrics and trace exporter |
| `diagnostics-prometheus` | `extensions/diagnostics-prometheus/` | Prometheus metrics exporter |

### Channel Plugins (confirmed from dist excludes / docs)

Discord, Telegram, WhatsApp, Slack, Signal, Feishu, Google Chat, Microsoft Teams, Matrix, LINE, Synology Chat, iMessage (via `sms`), Tlon, NOSTR, Twitch, QQ Bot (`qqbot`), Zalo, and more.

### Provider Plugins (confirmed from dist excludes / extensions listing)

Anthropic, OpenAI (including Codex), Google (including Vertex), Amazon Bedrock, Azure OpenAI, Ollama, OpenRouter, Perplexity, DeepSeek, Mistral (via openrouter), Fireworks, Together, Groq, xAI, Qwen, Qianfan, DeepInfra, Cerebras, Chutes, Venice, Arcee, Voyage, vLLM, LLaMA.cpp, SGLang, Vercel AI Gateway, Cloudflare AI Gateway, GitHub Copilot, OpenCode, and others.

### Tool and Capability Plugins (confirmed from directory listing)

| Plugin | Purpose |
|--------|---------|
| `browser` | Browser automation |
| `brave` | Brave Search integration |
| `exa` | Exa web search |
| `firecrawl` | Web crawling |
| `diffs` | Diff rendering |
| `codex` | OpenAI Codex harness |
| `codex-supervisor` | Codex supervisor harness |
| `voice-call` | Voice call capability |
| `file-transfer` | File transfer |
| `document-extract` | Document extraction |
| `comfy` | Image generation via ComfyUI |
| `canvas` | Canvas drawing |
| `pixverse` | Video generation |
| `phone-control` | Phone control |
| `azure-speech`, `deepgram`, `elevenlabs`, `senseaudio` | Speech/TTS providers |
| `device-pair`, `bonjour` | Device pairing / local discovery |
| `workboard` | Workboard integration |
| `webhooks` | Webhook delivery |
| `admin-http-rpc` | Admin HTTP RPC surface |

---

## `skills/` — Bundled Skill Packs

`skills/` contains **58 bundled SKILL.md files** (confirmed count from S113 directory listing). Each file is a skill pack — a YAML-frontmatter + Markdown-body instruction pack that the system loads and injects into the agent's context as compact XML. Plugins and workspace directories can add more skill packs on top of these.

A representative sample of bundled skill topics: coding agent workflows, GitHub integration, Discord bot operations, Slack workflows, weather lookup, Spotify playback control, health checks, diagram creation, task management (Things, Trello, Taskflow), voice calls, PDF operations, camera capture, Bear Notes, Apple Reminders, Apple Notes, Gemini, Eightctl, and more.

Skills are not plugins — they carry no runtime code. Think of them as the standard-operating-procedure manuals handed to the agent on its first day: they change what the agent knows how to do, not what tools it can call.

---

## `apps/` — Native Companion Clients

`apps/` contains native client applications: `ios/`, `macos/`, `android/`, `macos-mlx-tts/`, and `shared/`. These connect to the Gateway over its WebSocket API. The iOS/macOS clients are written in Swift; the Android client uses the Android platform. Source coverage of these clients in the documentation is thin — internal Swift/Kotlin details are not documented in this library.

---

## `deploy/`, `docs/`, `security/`

- **`deploy/`** — Hosting configuration. The root holds `Dockerfile` (multi-stage Node build), `docker-compose.yml`, `fly.toml` (Fly.io), `render.yaml` (Render.com), and the private `deploy/fly.private.toml`. See [Deployment and Lifecycle](./23-deployment.md) for how these are used.
- **`docs/`** — The Mintlify-hosted documentation source that feeds `docs.openclaw.ai`.
- **`security/opengrep/`** — Static-analysis rules (`precise.yml` and a `rules/` directory) used to enforce security invariants in the codebase.

---

## Putting It Together: A Quick Navigation Guide

When you need to find something, use this decision tree:

```
What are you looking for?
│
├── Agent loop logic, tool execution, session wiring
│   └── src/agents/  +  packages/agent-core/
│
├── Model/provider types (contracts)
│   └── packages/llm-core/
│
├── Model/provider runtime registry (actual stream dispatch)
│   └── packages/llm-runtime/
│
├── Gateway protocol (frame schemas, wire contracts)
│   └── packages/gateway-protocol/
│
├── Connect to a Gateway from external code (SDK)
│   └── packages/sdk/  (high-level)
│   └── packages/gateway-client/  (low-level)
│
├── Write a plugin
│   └── Import ONLY from openclaw/plugin-sdk/*
│   └── Never import from src/** or packages/*/src/**
│
├── Network SSRF policy
│   └── packages/net-policy/
│
├── Memory engine contracts (for a memory plugin)
│   └── packages/memory-host-sdk/
│
├── Bundled plugins (memory, channels, providers)
│   └── extensions/<plugin-id>/
│
└── Bundled skill packs
    └── skills/<skill-name>/SKILL.md
```

---

← Previous: [Monitoring and Observability: Logs, Debug Flags, OTel, Prometheus, and Health Endpoints](../operations/21-observability.md) · Next: [Deployment and Lifecycle: Install, Daemon Setup, Docker, and Hosted Options](./23-deployment.md) →
