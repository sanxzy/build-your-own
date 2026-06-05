# Documentation Libraries

A collection of self-contained, publish-ready documentation libraries. Each subdirectory is one complete library — a guided, dependency-ordered set of teaching chapters in portable Markdown that you can read on GitHub or render with MkDocs, Docusaurus, or Astro Starlight.

## Libraries

| Library | What it covers |
|---|---|
| [ai-agent-101_docs](ai-agent-101_docs/README.md) | **Build an AI Coding Agent from Scratch** — a 34-chapter course that builds a terminal coding agent from the ground up: the LLM toolkit, the agent loop, the terminal UI, and the full coding-agent composition with tools, sessions, extensions, and multi-agent orchestration. |

## Conventions

Every library in this repository follows the same rules:

- **Portable Markdown** — plain GitHub-Flavored Markdown; no framework-specific components, so it renders anywhere.
- **Self-contained chapters** — each page introduces the terms it uses and can be read without any source repository open.
- **Guided reading spine** — chapters are ordered foundation-first, with prev/next links and an `index.md` landing map per library.
- **Brand- and version-agnostic** — examples use generic names and minimal starting versions; nothing depends on a specific upstream.

## Adding a library

Each library lives in its own `*_docs/` directory with its own `README.md` and `index.md`. Add a new one as a sibling folder here and link it from the table above.
