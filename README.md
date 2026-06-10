# Build Your Own

A collection of **build-it-yourself documentation libraries**. Each one teaches you to build a real piece of software from scratch, one motivated step at a time — no prior expertise assumed, nothing hidden behind a framework you can't see into.

Every library is self-contained, portable Markdown: read it straight on GitHub, or render it with MkDocs, Docusaurus, or Astro Starlight.

## Libraries

| Library | Build your own… |
|---|---|
| [xzy_agent_docs](xzy_agent_docs/README.md) | **AI coding agent.** A 34-chapter course that builds a terminal coding agent from the ground up: the LLM toolkit, the agent loop, the terminal UI, and the full coding-agent composition with tools, sessions, extensions, and multi-agent orchestration. |
| [hermes-agent_docs](hermes-agent_docs/README.md) | **Self-improving AI agent.** A 42-chapter, beginner-first guide to Hermes Agent v0.16.0: the conversation loop, five memory layers, autonomy (cron/webhooks/dispatcher), multi-agent kanban and swarm topologies, the provider/credential system, the closed learning loop, the OS security model, and the full extension surface — every claim traced to source. |
| [openclaw_docs](openclaw_docs/README.md) | **Self-hosted AI gateway.** A 27-chapter guide to OpenClaw, a multi-channel AI gateway you run yourself: the control-plane gateway and wire protocol, channels and DM pairing, the agent loop and sessions, the memory system, the plugin/skill/tool primitives, model integration and fallback, multi-agent coordination, automation, and operations — configuration, storage, security, observability, and deployment. |
| [fc_ai_agent_docs](fc_ai_agent_docs/README.md) | **AI coding agent, from scratch.** A 32-chapter, teaching-voice course that builds a production-grade terminal coding agent layer by layer: a unified LLM streaming API across Anthropic, OpenAI, and Google; the agent loop with tool execution and context compaction; a differential-rendering terminal UI; and the full CLI with interactive, print, and RPC modes, plus a plugin system and multi-agent orchestration. Complete, runnable TypeScript throughout. |

More libraries will land here over time — each in its own folder.

## What makes these different

- **You build it, step by step.** Each chapter starts from the problem it solves, then arrives at the solution — never a finished-code dump you can't follow.
- **Self-contained chapters.** Every term is introduced on first use; you can read any page without the source repository open.
- **A guided reading spine.** Chapters are ordered foundation-first, with prev/next links and an `index.md` landing map per library.
- **Brand- and version-agnostic.** Examples use generic names and minimal starting versions, so nothing is tied to a specific upstream — the concepts are yours to reuse.

## How to read a library

1. Open the library's folder and start at its `README.md` or `index.md`.
2. Follow the chapters in order — each builds on the ones before it.
3. Use the prev/next links at the bottom of each chapter to move through it like a book.

## Repository layout

```
.
├── README.md              ← you are here (the collection index)
└── <name>_docs/           ← one self-contained library per folder
    ├── README.md          ← that library's front page
    ├── index.md           ← that library's reading map
    └── docs/...           ← the chapters
```

## Adding a library

Create a new `<name>_docs/` folder as a sibling here, give it its own `README.md` and `index.md`, and add a row to the **Libraries** table above.
