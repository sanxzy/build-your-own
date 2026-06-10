---
title: "Plugins, Skills, and Tools: Three Distinct Primitives"
description: "Precisely defines and contrasts tools, skills, and plugins — how each extends OpenClaw, how the manifest enables cold discovery, and how to install from ClawHub."
category: extending
type: explanation
tags: [plugins, skills, tools, plugin manifest, openclaw.plugin.json, cold discovery, ClawHub, plugin install, code plugin, bundle plugin, plugin shapes, clawhub, npm install, registerTool, SKILL.md, tool callable, skill instruction, in-process, defineToolPlugin, definePluginEntry, contracts.tools, activation, configSchema, plugin policy, plugins.slots, optional tool, factory tool]
keywords: [openclaw primitives, extend openclaw, plugin architecture, skill vs tool, plugin vs skill, openclaw.plugin.json required fields, cold manifest inspection, gateway in-process, plugin security, clawhub install, npm plugin, git plugin, plugin shapes]
sources: [S63, S69, S70, S71, S72, S100, S112, S113]
---

**TL;DR** — OpenClaw has three distinct extension primitives that beginners routinely conflate: **tools** (typed callable functions the model can invoke), **skills** (`SKILL.md` instruction packs that change how the agent behaves without adding new functions), and **plugins** (packaged runtime capabilities that can register tools, providers, channels, hooks, and more). This chapter defines all three precisely, explains the plugin manifest (`openclaw.plugin.json`) and why it enables cold discovery, and walks through installation via ClawHub.

# Plugins, Skills, and Tools: Three Distinct Primitives

When you want to extend OpenClaw, you have three building blocks available. Each one does something different. Using the wrong one for a job does not produce an error — it produces a system that does not work as you intended. So before writing a line of configuration or code, we need to understand what each primitive is for.

Let's work from the smallest and most targeted up to the largest and most general.

---

## Tools: functions the model can call

A **tool** is a typed callable function that the model can invoke during a run. When the model decides it needs to look something up, run a command, or take an action, it selects a tool by name and provides arguments. The agent loop executes that function and feeds the result back to the model.

Three properties define a tool:

| Property | What it is | Example |
|---|---|---|
| `name` | Stable lowercase identifier | `stock_quote` |
| `description` | Human-readable explanation the model uses to decide when to call this tool | `"Fetch a stock quote snapshot."` |
| `parameters` | TypeBox-validated JSON schema for the function's arguments | `{ symbol: string }` |

Here is the smallest possible tool registration, written directly inside a plugin's `register(api)` callback (we will get to what a plugin is in a moment):

```ts
// Simplified view of api.registerTool(...)
api.registerTool({
  name: "stock_quote",
  description: "Fetch a stock quote snapshot.",
  parameters: Type.Object({
    symbol: Type.String({ description: "Ticker symbol, for example OPEN." }),
  }),
  async execute(_id, params) {
    return { content: [{ type: "text", text: `Price for ${params.symbol}: ...` }] };
  },
});
```

The model never calls this function directly. The agent loop maintains an **effective tool policy** — the resolved set of tools the model is permitted to see for a given run — and only tool schemas that survive that policy filter are included in the model call. The model picks from what it sees; the loop executes it. We cover effective tool policy in depth in the next chapter (see [Tool System: Registration, Effective Policy, and Built-in Categories](./12-tool-system.md)).

**Tools change what the agent can do.** They add new actions to the agent's repertoire. They do not change how the agent reasons, speaks, or structures its replies.

---

## Skills: instructions that change how the agent behaves

A **skill** is a `SKILL.md` instruction pack — a Markdown file with a YAML frontmatter block — that OpenClaw injects into the system prompt as compact XML at context-assembly time.

Think of a skill like a laminated reference card pinned above an employee's desk. The card does not give the employee a new ability; it reminds them of a process to follow. The agent already has tools available; the skill tells it how and when to use them.

Here is a real example, abbreviated from the bundled `github` skill:

```markdown
---
name: github
description: "GitHub CLI for issues, PRs, CI/check logs, comments, reviews, releases, repos, and gh api queries."
metadata:
  openclaw:
    requires:
      bins: ["gh"]
---

# GitHub

Use `gh` for GitHub. Use `git` for local commits/branches/push/pull.

## PRs

    gh pr list --repo owner/repo --json number,title,state
    gh pr view 55 --repo owner/repo --json title,body
```

Notice what is and is not in this file. There is no executable code. There is no function registration. There is a `metadata.openclaw.requires.bins` field that gates the skill — if the `gh` binary is not present on the machine, OpenClaw does not load this skill at all. That is called **self-gating**.

When the skill does load, OpenClaw injects its content into the system prompt before each model call. The agent learns from the instructions inside the skill, not from an API function. Skill loading, precedence (workspace skills win over bundled ones), and token cost are covered in full in [Skills](./13-skills.md).

**Skills change how the agent communicates and reasons.** They add no callable functions to the model's toolbox. They shape its behavior.

---

## Plugins: packaged runtime capabilities

A **plugin** is a package that can register any combination of:

- **Tools** (via `api.registerTool(...)`) — callable functions for the model
- **Providers** (via `api.registerProvider(...)`) — model inference endpoints
- **Channels** (via `api.registerChannel(...)`) — messaging surfaces like Telegram or Discord
- **Hooks** (via `api.on(...)`) — callbacks that intercept the agent loop lifecycle
- **Skills** — instruction packs bundled with the plugin package
- **Services** (via `api.registerService(...)`) — background processes the plugin owns
- **CLI commands** (via `api.registerCli(...)`) — new subcommands under the `openclaw` CLI

If tools are verbs the model can use and skills are instructions it follows, then a plugin is the packaging system that delivers both — plus everything else.

An analogy: tools are electrical outlets (discrete points of power); skills are the operating manual (instructions about how to use things); a plugin is the building contractor who wires in the outlets, furnishes the manual, and connects the plumbing.

### Prerequisite recap

Before we go deeper, here is a brief grounding in two concepts from earlier chapters that plugins connect to:

- **Agents** (see [Agents](../agents/05-agents.md)) have a workspace directory at `~/.openclaw/agents/<agentId>/agent/`. A `skills/` directory there holds per-agent custom skills. Plugins extend what an agent's workspace can do at the runtime level.
- **The agent loop** (see [The Agent Loop](../agents/06-agent-loop.md)) has a context-assembly stage (where skills are injected) and a tool-execution stage (where registered tools are called). Plugins are the mechanism by which new tools and new behaviors enter those stages.

---

## What the manifest file does — and why it matters

Every OpenClaw plugin has a file called `openclaw.plugin.json` in its package root. This manifest is separate from the JavaScript runtime code.

Here is the minimal valid manifest from the quickstart in the source:

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Let's look at each field:

| Field | Required? | Purpose |
|---|---|---|
| `id` | Yes | The unique identifier for this plugin; also the key in `plugins.entries` config |
| `configSchema` | Yes | JSON Schema for plugin-specific configuration; use an empty strict object when the plugin has no config |
| `name` | No (recommended) | Display name shown in `openclaw plugins list` and ClawHub |
| `description` | No (recommended) | Short description for discovery surfaces |
| `contracts.tools` | No | List of tool names this plugin registers; enables cold discovery |
| `contracts.embeddingProviders` | No | Embedding providers registered by this plugin |
| `activation.onStartup` | No | When `true`, the plugin loads at Gateway startup; when `false`, it is opt-in |
| `providers` | No | Provider ids this plugin registers |
| `setup.providers[].envVars` | No | Environment variable names for provider credentials — enables setup hints without loading plugin code |

### Cold discovery: what the Gateway gains

Now we can answer one of the most important questions about this design: **why does the manifest enable cold discovery without loading plugin runtime?**

When the Gateway starts, it needs to know which plugins own which tools, which providers exist, and how to configure them — but it does not want to import and execute every plugin's JavaScript merely to find that out. Loading all plugin code eagerly would be slow and would execute untrusted code before the operator has a chance to apply policy.

The manifest provides a static, code-free description of what the plugin owns. The Gateway reads the manifest to build its internal registry. It can answer questions like "which plugin registered `my_tool`?" without executing any plugin code at all. This is called cold discovery — inspecting from cold metadata alone.

Concretely, what the Gateway gains:

1. **Tool ownership resolution** — `contracts.tools` lists all tools a plugin owns. If a tool name conflict appears, the Gateway can report it before importing a single module.
2. **Policy enforcement without execution** — `toolMetadata.<tool>.optional: true` in the manifest tells the Gateway this tool requires an explicit `tools.allow` entry before it is sent to the model. The Gateway applies this rule at policy time, not at execution time.
3. **Activation planning** — `activation.onStartup: false` keeps a plugin dormant. The Gateway can still surface it in `openclaw plugins list`, show its configuration schema, and load it lazily when a trigger condition fires (e.g., a matching provider model ref is selected), all without importing the module.
4. **Setup hints without code** — `setup.providers[].envVars` lets the setup wizard tell you which environment variables to configure before the plugin is enabled — without executing the plugin's auth logic.

You can verify cold vs. runtime state yourself:

```bash
# Cold manifest and registry check (no plugin code loaded):
openclaw plugins inspect my-plugin

# Full runtime check (requires the live Gateway to import the plugin):
openclaw plugins inspect my-plugin --runtime --json
```

The difference between these two commands is the difference between cold discovery (manifest) and runtime registration (code).

---

## Two plugin styles: code plugins and bundle-style plugins

The source identifies two plugin formats:

| Format | How it loads | Use when |
|---|---|---|
| Native code plugin | `openclaw.plugin.json` + a runtime JavaScript module loaded in-process | You are adding tools, hooks, providers, channels, or services that require executing code |
| Compatible bundle | A compatible plugin layout (skills, MCP servers, config) mapped into the OpenClaw plugin inventory | You are packaging skills, commands, or configuration that does not need runtime code execution |

**What is the practical difference?**

A **code plugin** has a `register(api)` function — a JavaScript callback that runs inside the Gateway process when the plugin is activated. This is where you call `api.registerTool(...)`, `api.registerProvider(...)`, `api.on(...)`, and so on. Every bundled OpenClaw extension in the `extensions/` directory is a code plugin.

A **bundle-style plugin** packages content that OpenClaw can consume without executing a `register` callback. The primary example is a skill bundle: a directory with one or more `SKILL.md` files. The plugin manifest describes the content; OpenClaw loads the skills from disk. No JavaScript module is required.

When should you write each? If you need to call an external API, intercept the agent loop, register a model provider, or add new callable tools — write a code plugin. If you are packaging curated instructions, MCP server configuration, or a set of prompts — a bundle is lighter and simpler.

### A minimal code plugin

Here is the full skeleton of a code plugin that registers one tool:

```ts
// index.ts — a complete minimal code plugin
import { Type } from "typebox";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Adds a custom tool to OpenClaw",
  register(api) {
    api.registerTool({
      name: "my_tool",
      description: "Echo one input value",
      parameters: Type.Object({ input: Type.String() }),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: `Got: ${params.input}` }],
        };
      },
    });
  },
});
```

For simple tool-only plugins with a fixed list of tools, OpenClaw provides a shorthand called `defineToolPlugin` that generates the manifest metadata automatically:

```ts
// defineToolPlugin — the shorthand for tool-only plugins
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quote snapshots.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "Quote API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      description: "Fetch a stock quote snapshot.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol, e.g. OPEN." }),
      }),
      async execute({ symbol }, config) {
        return { symbol: symbol.toUpperCase(), configured: Boolean(config.apiKey) };
      },
    }),
  ],
});
```

Run `openclaw plugins build --entry ./dist/index.js` after building to generate (or update) the matching `openclaw.plugin.json` from the static metadata exposed by `defineToolPlugin`. The build step writes `contracts.tools` so cold discovery works.

---

## Plugin shapes: what can a plugin register?

We have seen that `api.registerTool(...)` adds a tool. Let's look at the full surface the plugin API exposes, grouped by area:

### Capabilities

| Method | What it registers |
|---|---|
| `api.registerTool(tool, opts?)` | Agent tool (required, or `{ optional: true }` for opt-in tools) |
| `api.registerProvider(...)` | Text inference provider (LLM) |
| `api.registerChannel(...)` | Messaging channel |
| `api.registerEmbeddingProvider(...)` | Vector embedding provider |
| `api.registerSpeechProvider(...)` | Text-to-speech / STT provider |
| `api.registerImageGenerationProvider(...)` | Image generation |
| `api.registerWebFetchProvider(...)` | Web fetch / scrape provider |
| `api.registerWebSearchProvider(...)` | Web search |

### Infrastructure

| Method | What it registers |
|---|---|
| `api.on(hookName, handler, opts?)` | Typed lifecycle hook (e.g. `before_tool_call`) |
| `api.registerService(service)` | Background service |
| `api.registerHttpRoute(params)` | Custom Gateway HTTP endpoint |
| `api.registerCli(registrar, opts?)` | CLI subcommand |
| `api.registerCommand(def)` | Custom command that bypasses the model |

### Exclusive slots

| Method | What it registers |
|---|---|
| `api.registerContextEngine(id, factory)` | Context engine (one active at a time) |
| `api.registerMemoryCapability(capability)` | Memory implementation (one active at a time, the memory slot) |

The memory slot is exclusive — only one memory plugin can be active at a time — because two plugins writing to the same memory store simultaneously would produce conflicting updates. See [Memory System](../memory/10-memory-system.md) for how that slot is configured.

---

## Required vs. optional tools

When a plugin registers a tool with `api.registerTool(tool)` (no options), that tool is **required**: it is available to the model whenever the plugin is enabled and the effective tool policy allows it.

When a plugin registers with `api.registerTool(tool, { optional: true })`, the tool requires an **explicit opt-in** before it appears in the model's context:

```json5
{
  tools: { allow: ["stock_quote"] }   // or ["stock-quotes"] to allow all tools from the plugin
}
```

Use optional tools for side-effectful capabilities, unusual system binaries, or features that should not be exposed by default. The manifest must also declare the optional flag so cold discovery works without loading the runtime:

```json
{
  "toolMetadata": {
    "stock_quote": { "optional": true }
  }
}
```

---

## Factory tools

A **factory tool** is a tool whose existence depends on the runtime context. Instead of returning a static tool definition, the factory function receives the current context and can return `null` to suppress the tool for this particular run:

```ts
tool({
  name: "local_workflow",
  description: "Run a local workflow outside sandboxed sessions.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      // This run is sandboxed — suppress the tool entirely
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

A concrete example of when a factory tool returns nothing: when a session is running in sandbox mode (non-`main` sessions with `agents.defaults.sandbox.mode: "non-main"`), the `toolContext.sandboxed` flag is `true`. A tool factory that checks this flag can opt out entirely, so the model never even sees the tool schema for that run. The tool is not blocked by policy — it does not exist for that run at all.

Factories are still for fixed tool names. If the plugin needs to compute tool names dynamically, use `definePluginEntry` directly.

---

## Installing plugins from ClawHub

ClawHub (`clawhub.ai`) is the primary discovery surface for community plugins. Use it to search for plugins and read their documentation before installing.

To install, use the `openclaw plugins install` command with a source prefix:

```bash
# From ClawHub (preferred for community plugins)
openclaw plugins install clawhub:<package>

# From ClawHub using an @openclaw/* package name
openclaw plugins install clawhub:@openclaw/<name>

# From npm directly
openclaw plugins install npm:<package>

# From a git ref
openclaw plugins install git:github.com/<owner>/<repo>@<ref>

# From a local development checkout
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

Source prefix summary:

| Prefix | Use when |
|---|---|
| `clawhub:` | Installing from ClawHub — preferred for community plugins |
| `npm:` | You need a specific npm registry dist-tag or version workflow |
| `git:` | You need a specific branch, tag, or commit |
| `./path` | Local development and testing |

Bare package specs (without a prefix) use special compatibility behavior: if the name matches a bundled plugin id, OpenClaw uses the bundled copy; if it matches an official external plugin id, OpenClaw uses the official package catalog; otherwise, it falls back to npm during the launch cutover. Use explicit prefixes when you need deterministic source selection.

### After installation: restart required

Installing a plugin installs the JavaScript package on disk. The running Gateway has not imported it yet. A plugin takes effect only after the Gateway restarts or reloads:

```bash
openclaw gateway restart
```

When a managed Gateway is already running with config reload enabled, OpenClaw detects the changed plugin install record and restarts automatically. On VPS or container installs, restart the Gateway yourself.

After restart, verify the plugin loaded:

```bash
# Check runtime registration (proves the Gateway actually imported the plugin):
openclaw plugins inspect <plugin-id> --runtime --json
```

### Plugin policy

The operator controls which plugins can load via `plugins.allow`, `plugins.deny`, and per-entry `enabled` flags:

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],          // Exclusive allowlist — only listed ids can load
    deny: ["untrusted-plugin"],     // Deny wins over allow and per-entry config
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } }
    }
  }
}
```

Key rules:
- `plugins.deny` wins over `allow` and per-plugin `enabled: true`.
- When `plugins.allow` is set, only ids in that list can load, even if `tools.allow` includes `"*"`.
- `plugins.entries.<id>.enabled: false` disables one plugin without removing its config.
- `openclaw plugins install` adds the installed id to an existing `plugins.allow` list automatically so the explicit install can load after restart.

Run `openclaw doctor --fix` when config validation reports stale plugin ids, allowlist/tool mismatches, or legacy plugin paths.

---

## Plugins run in-process with the Gateway

There is an architectural fact about plugins that has security implications you should understand: **plugins run in-process with the Gateway**. They are not isolated subprocesses or sandboxes. When a plugin's `register(api)` callback runs, it executes in the same Node.js process as the Gateway itself.

This means:
- A plugin with a bug or malicious intent has access to the same memory, file system, and network connections as the Gateway process.
- The install policy (`security.installPolicy`) lets the operator run a trusted local command before a plugin install proceeds, providing a gate before any code is executed.
- ClawHub plugins are community-maintained packages. Treat plugin installs with the same scrutiny you would give a dependency in your own project.

The design rationale for in-process execution (and its tradeoffs compared to isolated subprocess execution) is documented in [Design Decisions and Tradeoffs](../reference/25-design-decisions.md). The full security model — including how sandbox mode limits what tools can do for non-`main` sessions — is covered in [Security and Governance](../operations/20-security.md).

---

## The three primitives, compared

We have now defined all three. Let's put them side by side:

| Primitive | What it is | How it works | When to use it |
|---|---|---|---|
| **Tool** | A typed callable function | The model selects it by name; the agent loop executes it; the result is fed back to the model | When you want the agent to be able to take a new action (call an API, run a command, read a file) |
| **Skill** | A `SKILL.md` instruction pack | Injected into the system prompt as XML at context-assembly time | When you want the agent to behave differently — follow a process, use a specific style, prefer certain tools |
| **Plugin** | A packaged runtime capability | Loaded in-process at Gateway startup; registers tools, providers, channels, hooks, skills, and services via the Plugin SDK API | When you are packaging a new capability that requires code, configuration, or multiple registrations |

**The key question:** if you want to change how the agent *communicates*, use a skill. If you want to change what the agent *can do*, register a tool (via a plugin). If you are packaging multiple capabilities together — or building something that requires running code — write a plugin.

A concrete contrast: the `github` skill tells the agent *how* to use the `gh` CLI that is already on the machine. If `gh` were not a system binary but an API the agent needed to call through OpenClaw, you would register it as a tool inside a plugin.

---

## What happens when a plugin cannot load

When a plugin appears in the installed registry but fails to load, OpenClaw does not block Gateway startup. Instead, it skips that plugin and reports a diagnostic. Common causes:

| Symptom | Likely cause | Fix |
|---|---|---|
| Plugin in `plugins list` but runtime hooks do not run | Gateway was not restarted after install | `openclaw gateway restart` |
| `plugin present but blocked` in diagnostics | Filesystem ownership mismatch (plugin files owned by a different Unix user) | Fix ownership, then `openclaw plugins registry --refresh` |
| Dependency import fails at runtime | Plugin was installed but its npm dependencies are not available | `openclaw plugins update <id>` or reinstall |
| `plugin entry not found: ./dist/index.js` | Package was published without compiled JavaScript | Update or reinstall after the publisher ships compiled output |

When a stale channel plugin config references a plugin that is no longer installed, the Gateway skips that channel rather than blocking all other channels. Run `openclaw doctor --fix` to remove stale entries.

---

## Summary

We have defined all three extension primitives:

- **Tools** are typed functions the model calls. They change what the agent can do. They live inside plugins.
- **Skills** are `SKILL.md` instruction packs injected into the system prompt. They change how the agent behaves. They do not add callable functions.
- **Plugins** are the packaging system. They run in-process with the Gateway, register any combination of tools, providers, channels, hooks, and services, and are described to the Gateway via the `openclaw.plugin.json` manifest — which enables cold discovery without loading plugin code.

The chapters immediately following go deeper on each:

- **Tool system** — registration, optional tools, factory tools, effective policy, and built-in categories: [Tool System](./12-tool-system.md)
- **Skills** — `SKILL.md` structure, loading precedence, self-gating, token cost, and bundled skills: [Skills](./13-skills.md)
- **Hooks** — intercepting the agent loop lifecycle with plugins: [Agent Loop Hooks](./14-hooks.md)

---

← Previous: [Memory System: File Memory, memory-core, memory-lancedb, and memory-wiki](../memory/10-memory-system.md) · Next: [Tool System: Registration, Effective Policy, and Built-in Categories](./12-tool-system.md) →
