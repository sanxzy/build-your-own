---
title: "Skills: SKILL.md Structure, Loading Precedence, and Token Cost"
description: "How SKILL.md files are structured, the six-level loading order, self-gating, the token cost formula, and the 58 bundled skills."
category: extending
type: explanation
tags: [skills, SKILL.md, frontmatter, loading precedence, token cost, self-gating, metadata.openclaw.requires, bundled skills, slash commands, command-dispatch, skill workshop, ClawHub skills, skill allowlist, user-invocable, disable-model-invocation, bins, env, config, os, always, plugin skills, extraDirs, workspace skills, managed skills, personal skills]
keywords: [skill file, skill instruction, agent prompt injection, skill gating, skill collision, same-name precedence, skill cost formula, skill token budget, openclaw skill, command-dispatch tool, skill tool dispatch, AgentSkills spec, SKILL.md format]
sources: [S60, S61, S62, S112, S113, S119]
---

**TL;DR** — A skill is a Markdown instruction file that OpenClaw injects into the agent's system prompt as a compact XML block. This chapter explains how to structure a `SKILL.md` file, the exact order in which six skill sources are resolved (and which one wins a name collision), how self-gating gates a skill to the environments where it will actually work, what the token-cost formula means for your prompt budget, and what `command-dispatch: tool` does that a plain skill cannot.

# Skills: SKILL.md Structure, Loading Precedence, and Token Cost

Before we go further, let's quickly orient to how skills fit into the bigger picture.

A **skill** is one of three distinct extension primitives in OpenClaw — not to be confused with tools or plugins. A tool is a callable function the model can invoke; a plugin is a packaged runtime module that can register tools, channels, and hooks. A skill, by contrast, is a plain Markdown instruction file: it changes *how* the agent behaves without adding any new callable function. (For the full three-way comparison, see [Plugins, Skills, and Tools](./11-plugins-skills-tools.md).)

Skills are also injected into the **system prompt's Skills section**, which is the fixed part of the context the model reads before every conversation turn. (For how `buildAgentSystemPrompt` assembles the full prompt, see [System Prompt and Context](../agents/09-system-prompt.md).)

---

## The problem a skill solves

Imagine you want your agent to always use the `gh` CLI for GitHub operations and never reach for raw `curl` or the REST API directly. You could put that instruction in `AGENTS.md` — your agent's general workspace file — but then *every* conversation carries that overhead, including conversations that never touch GitHub. What you want is a conditional instruction: inject this guidance only when the `gh` binary is actually available.

That is exactly what a skill does: it packages a focused instruction block, declares when it should be active, and lets OpenClaw decide at load time whether the current environment satisfies those conditions.

---

## Anatomy of a SKILL.md file

Every skill is a directory containing exactly one `SKILL.md` file. That file has two parts: a YAML frontmatter block and a Markdown body.

Think of a SKILL.md as a recipe card: the frontmatter is the label on the card (what it's called, when to reach for it), and the body is the recipe itself (the actual instructions for the agent).

### Required frontmatter fields

At minimum, a `SKILL.md` needs a `name` and a `description`:

```markdown
---
name: github
description: "GitHub CLI for issues, PRs, CI/check logs, comments, reviews, releases, repos, and gh api queries."
---

# GitHub

Use `gh` for GitHub. Use `git` for local commits/branches/push/pull. Use code-reading tools for deep reviews.
...
```

The real bundled `github` skill (at `skills/github/SKILL.md`) looks exactly like this — the frontmatter names it `github` and describes it in one line, and the body gives concrete `gh` command patterns.

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Unique slug: lowercase letters, digits, and hyphens |
| `description` | Yes | One-line summary shown to the agent and in discovery; keep under 160 characters |

OpenClaw follows the [AgentSkills](https://agentskills.io) spec. The frontmatter parser supports **single-line keys only** — `metadata` must be a single-line JSON object.

### Optional frontmatter fields

| Field | Default | Description |
|---|---|---|
| `user-invocable` | `true` | Expose the skill as a `/skillname` slash command the user can type |
| `disable-model-invocation` | `false` | Keep the skill out of the agent's normal prompt; still usable via `/skill <name>` |
| `command-dispatch` | — | Set to `"tool"` to route the slash command directly to a registered tool, bypassing the model |
| `command-tool` | — | Tool name to invoke when `command-dispatch: tool` is set |
| `command-arg-mode` | `"raw"` | For tool dispatch: forwards the raw args string to the tool |
| `homepage` | — | URL shown as "Website" in the macOS Skills UI |

We'll look at `command-dispatch: tool` closely in its own section below.

### The body

The body is plain Markdown. It becomes the agent's instructions for this skill. You can use `{baseDir}` anywhere in the body to reference files inside the skill's own directory without hardcoding a path:

```markdown
Run the setup script at `{baseDir}/scripts/setup.sh`.
```

OpenClaw expands `{baseDir}` to the absolute path of the skill's directory at load time.

---

## Loading precedence: six sources, highest first

OpenClaw discovers skills from six separate sources. When the same `name` appears in more than one source, the source with the **highest** precedence wins — the lower-precedence copy is silently ignored.

| Priority | Source | Path |
|---|---|---|
| 1 — highest | Workspace skills | `<workspace>/skills/` |
| 2 | Project agent skills | `<workspace>/.agents/skills/` |
| 3 | Personal agent skills | `~/.agents/skills/` |
| 4 | Managed / local skills | `~/.openclaw/skills/` |
| 5 | Bundled skills | Shipped with the install |
| 6 — lowest | Extra directories + plugin skills | `skills.load.extraDirs` + plugin-declared skill dirs |

Think of the list as a stack of transparencies: the topmost non-blank layer wins for any given skill name.

### Same-name collision: which one wins?

If your workspace `skills/github/SKILL.md` and the bundled `github` skill both have `name: github`, the workspace copy at priority 1 wins and the bundled version is not loaded. You can use this to **override** any bundled skill by placing a same-named skill higher in the stack.

The precedence tests in `src/skills/loading/workspace-precedence.test.ts` confirm this: a workspace version of `demo-skill` with description `"Workspace version"` suppresses both the managed and bundled copies.

### Folder grouping does not affect the name

You can nest skills in subfolders for organization:

```text
<workspace>/skills/personal/hello-world/SKILL.md   → name: "hello-world"
<workspace>/skills/research/SKILL.md               → name: "research"
```

OpenClaw walks any depth under a configured skill root. The folder path is for your organization only. The skill's name, slash command, and allowlist key all come from the `name` frontmatter field (or the directory name when `name` is missing).

### Per-agent vs shared scope

In a multi-agent setup, each agent has its own workspace directory. Use the path level that matches the visibility you want:

| Scope | Path | Visible to |
|---|---|---|
| Per-agent | `<workspace>/skills/` | Only that agent |
| Project-agent | `<workspace>/.agents/skills/` | Only that workspace's agent |
| Personal | `~/.agents/skills/` | All agents on this machine |
| Shared managed | `~/.openclaw/skills/` | All agents on this machine |
| Extra dirs | `skills.load.extraDirs` | All agents on this machine |

---

## Self-gating: loading only when the environment is ready

Without gating, a skill that requires the `gh` binary would be injected into every agent's prompt — even on a machine where `gh` is not installed, where the instruction would be confusing rather than helpful.

Self-gating lets a skill declare its own prerequisites. OpenClaw evaluates these gates at load time and drops any skill that fails them. A skill with no `metadata.openclaw` block is always eligible.

The gates live in a single-line JSON object under the `metadata` key:

```markdown
---
name: github
description: "GitHub CLI for issues, PRs, CI/check logs, comments, reviews, releases, repos, and gh api queries."
metadata:
  { "openclaw": { "requires": { "bins": ["gh"] }, "install": [...] } }
---
```

This is the real `github` bundled skill: it gates on `bins: ["gh"]`, so the skill only loads on machines where `gh` is on `PATH`.

### Gating options

| Key | Type | Gate condition |
|---|---|---|
| `requires.bins` | `string[]` | **All** listed binaries must exist on `PATH` |
| `requires.anyBins` | `string[]` | **At least one** binary must exist on `PATH` |
| `requires.env` | `string[]` | Each env var must exist in the process or be provided via config |
| `requires.config` | `string[]` | Each `openclaw.json` path must be truthy |
| `os` | `"darwin" \| "linux" \| "win32"` | Platform filter |
| `always` | `boolean` | When `true`, skip all gates and always include the skill |

`always: true` overrides even failing `env` or `bins` checks — it is an unconditional include.

A more complete gating example, showing multiple conditions together:

```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
metadata:
  {
    "openclaw": {
      "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
      "primaryEnv": "GEMINI_API_KEY"
    }
  }
---
```

This skill loads only when `uv` is on `PATH`, `GEMINI_API_KEY` is set, and `browser.enabled` is truthy in `openclaw.json`. If any condition fails, the skill is silently skipped — the agent prompt never mentions it.

### Legacy metadata

Skills that define a `metadata.clawdbot` block instead of `metadata.openclaw` are still accepted by OpenClaw when `metadata.openclaw` is absent. New skills should use `metadata.openclaw`.

---

## Token cost: why it matters

Every eligible skill gets serialized into a compact XML block and injected into the Skills section of the agent's system prompt. That injection has a real cost in tokens.

Think of your context window as a notebook with a finite number of pages. Every skill you enable fills in a few more lines. At some point, you are spending notebook space on skills the model rarely consults — space that could go to recent conversation history.

The cost formula, directly from the source documentation (S60):

```text
total = 195 + Σ (97 + len(name) + len(description) + len(filepath))
```

Breaking this down:

- **Base overhead (~195 characters):** The `<available_skills>` wrapper, the preamble lines telling the model how to use skills, and surrounding whitespace. This cost only applies when at least one skill is eligible.
- **Per-skill overhead (~97 characters):** The `<skill>`, `<name>`, `<description>`, `<location>`, and closing tag characters for each skill entry.
- **Variable per skill:** The length of the actual `name`, `description`, and `filepath` strings.
- **XML escaping:** Characters like `&`, `<`, `>`, `"`, `'` in names or descriptions expand into entities (`&amp;`, `&lt;`, etc.), adding a few characters per occurrence.

At approximately four characters per token, the 97-character per-skill overhead is roughly 24 tokens per skill before field lengths are added.

**What this means for an operator:** If you have 40 eligible skills and each has a 120-character description and a 60-character filepath, the Skills section alone occupies roughly:

```
195 + 40 × (97 + ~20 + ~120 + ~60) ≈ 195 + 40 × 297 ≈ 12,075 characters ≈ ~3,000 tokens
```

That is a non-trivial slice of a 200k-token context window. Two levers control it:

1. **Keep descriptions short.** The formula is linear in description length. A 60-character description costs half what a 120-character one costs in the variable term.
2. **Reduce eligible skills.** Use agent allowlists (see below) or disable skills via `skills.entries.<name>.enabled: false` in `openclaw.json` to drop skills that a particular agent does not need.

OpenClaw also applies tiered prompt limits: if the full skill block exceeds a configured `skills.limits.maxSkillsPromptChars` budget, it falls back to a compact format (descriptions omitted), then binary-searches to fit as many skills as possible within the budget.

---

## The 58 bundled skills

OpenClaw ships 58 bundled skills covering a wide range of use cases. These load at priority 5 — below workspace, project-agent, personal, and managed skills, but above `extraDirs` and plugin skills. Most bundled skills are gated (they require specific binaries or env vars), so they only become active when the relevant tool is actually present.

A sample of what is included to give a concrete sense of the range:

| Skill name | Gate condition | Purpose |
|---|---|---|
| `github` | `gh` binary on PATH | GitHub CLI for issues, PRs, CI logs, reviews |
| `gemini` | `gemini` binary on PATH | Gemini CLI for coding and Google search |
| `slack` | (varies) | Slack workspace interactions |
| `discord` | (varies) | Discord operations |
| `notion` | (varies) | Notion workspace read/write |
| `obsidian` | macOS | Obsidian note management |
| `spotify-player` | CLI binary | Spotify playback control |
| `weather` | (varies) | Weather lookups |
| `tmux` | `tmux` binary | Terminal multiplexer automation |
| `coding-agent` | opt-in + CLI binary | Delegate coding tasks to a sub-agent |
| `python-debugpy` | `debugpy` binary | Python debugger integration |
| `peekaboo` | macOS | macOS screen capture |
| `summarize` | (none) | Text summarization workflow |
| `skill-creator` | (none) | Skill Workshop proposals |

The full list in `skills/` covers: `1password`, `apple-notes`, `apple-reminders`, `bear-notes`, `blogwatcher`, `blucli`, `camsnap`, `canvas`, `clawhub`, `coding-agent`, `diagram-maker`, `discord`, `eightctl`, `gemini`, `gh-issues`, `gifgrep`, `github`, `gog`, `goplaces`, `healthcheck`, `himalaya`, `imsg`, `mcporter`, `meme-maker`, `model-usage`, `nano-pdf`, `node-connect`, `node-inspect-debugger`, `notion`, `obsidian`, `openai-whisper`, `openai-whisper-api`, `openhue`, `oracle`, `ordercli`, `peekaboo`, `python-debugpy`, `sag`, `session-logs`, `sherpa-onnx-tts`, `skill-creator`, `slack`, `songsee`, `sonoscli`, `spike`, `spotify-player`, `summarize`, `taskflow`, `taskflow-inbox-triage`, `things-mac`, `tmux`, `trello`, `video-frames`, `voice-call`, `wacli`, `weather`, and `xurl`. That is 58 entries.

Because most are gated by binary presence, a fresh install with no extra CLIs will activate only the ungated or always-on skills — the rest remain dormant until their dependency is installed.

### The coding-agent skill is opt-in

The `coding-agent` bundled skill is disabled by default. To enable it, set `skills.entries.coding-agent.enabled: true` in `openclaw.json` and ensure one of `claude`, `codex`, `opencode`, or another supported CLI is installed and authenticated.

---

## Agent allowlists: separating location from visibility

Loading precedence controls *which copy of a skill* is used. An agent allowlist controls *which skills an agent can see* — regardless of where they loaded from.

These are two separate controls, and you can use both at once.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // all agents see these two by default
    },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults entirely — only docs-search
      { id: "locked-down", skills: [] }, // no skills at all
    ],
  },
}
```

Key rules:
- Omit `agents.defaults.skills` to leave all skills unrestricted by default.
- Omit `agents.list[].skills` to inherit `agents.defaults.skills`.
- A non-empty `agents.list[].skills` list is the **final** set — it does not merge with the defaults.
- Set `agents.list[].skills: []` to give an agent no skills.

The effective allowlist applies to prompt building, slash-command discovery, sandbox sync, and skill snapshots.

---

## User-invocable skills and slash commands

When `user-invocable: true` (the default), a skill is registered as a slash command. You can invoke it by typing `/github` or `/weather` directly in chat without waiting for the model to decide to use it.

The agent still goes through the normal model inference path — it receives the slash command as input, and the skill's instructions in its system prompt guide what it does next.

### command-dispatch: tool — bypassing the model

Here is where skills gain a capability that normal skills do not have.

Setting `command-dispatch: "tool"` on a skill causes the slash command to **bypass the model entirely** and dispatch directly to a registered tool. The model is not involved — the command routes straight to a tool call.

```markdown
---
name: my-quick-action
description: Run a quick action directly.
command-dispatch: tool
command-tool: my_registered_tool
command-arg-mode: raw
---

(Body is not injected into the model prompt when disable-model-invocation is also true)
```

When the user types `/my-quick-action some arguments here`, OpenClaw:

1. Intercepts the slash command before sending it to the model.
2. Calls the registered tool `my_registered_tool` directly.
3. Passes `{ command: "some arguments here", commandName: "my-quick-action", skillName: "my-quick-action" }` to the tool.

The tool executes and returns a result without the model being consulted at any step.

**Why this matters:** A normal skill injection tells the model *how* to use a tool. The model still decides when to call it. `command-dispatch: tool` removes that decision layer — the user's `/command` is a direct, deterministic tool invocation. This is the right choice for quick actions where model latency is unwanted and the outcome is always the same given the same command (a status check, a toggle, a fixed script invocation).

The tool dispatch path enforces the same full effective tool policy pipeline as the normal model-call path — the shortcut goes around the model, not around security controls.

---

## Config reference: skills.* in openclaw.json

You can toggle and configure any skill, bundled or installed, under `skills.entries` in `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    allowBundled: ["github", "weather"], // restrict which bundled skills are eligible
    load: {
      extraDirs: ["~/Projects/my-skills"],
      watch: true,
      watchDebounceMs: 250,
    },
    entries: {
      "github": { enabled: true },
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "key-here" },
      },
      "sag": { enabled: false }, // explicitly disable a bundled skill
    },
  },
}
```

| Key | Description |
|---|---|
| `skills.allowBundled` | Allowlist for bundled skills only; managed and workspace skills are unaffected |
| `skills.load.extraDirs` | Additional skill directories at the lowest precedence |
| `skills.load.watch` | Watch skill folders for changes (default `true`) |
| `skills.entries.<key>.enabled` | `false` disables the skill even if bundled or installed |
| `skills.entries.<key>.apiKey` | Convenience field for skills declaring `metadata.openclaw.primaryEnv` |
| `skills.entries.<key>.env` | Env vars injected for the agent run (host only, not sandbox) |
| `skills.entries.<key>.config` | Optional bag for custom per-skill configuration |

> **Sandbox note:** `skills.entries.<key>.env` and `apiKey` inject into the **host** process for that agent turn only — they do not reach inside a Docker sandbox. To pass secrets into sandboxed runs, use `agents.defaults.sandbox.docker.env` instead.

---

## Snapshots and when changes take effect

OpenClaw snapshots the eligible skill list when a session starts and reuses that snapshot for all subsequent turns. Changes to skills or config take effect on the **next new session**.

Two mid-session refresh paths exist:

- The skills watcher detects a `SKILL.md` file change.
- A new eligible remote node connects.

When a refresh fires, the new snapshot is picked up on the next agent turn in that session.

---

## Skill Workshop: letting the agent propose skills

If you want to let the agent draft a skill proposal for your review before it goes live, use the Skill Workshop:

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

By default, `skills.workshop.approvalPolicy` is `"pending"`, meaning agent-initiated proposals need operator approval before applying.

---

## Installing from ClawHub

[ClawHub](https://clawhub.ai) is the public skill registry. Install skills into your workspace or globally:

```bash
openclaw skills install <slug>                     # into workspace skills/
openclaw skills install <slug> --global            # into ~/.openclaw/skills
openclaw skills install git:owner/repo@ref         # from a Git repository
openclaw skills update --all                       # update all workspace skills
openclaw skills verify <slug>                      # check trust envelope
```

Treat third-party skills as untrusted content: read them before enabling, and prefer sandboxed runs for tools that accept untrusted input.

---

## Security note on skill roots

Workspace, project-agent, and `extraDirs` skill discovery only accepts skill roots whose resolved real path stays inside the configured root, unless `skills.load.allowSymlinkTargets` explicitly trusts a target root. This prevents a symlinked skill directory from escaping to an arbitrary location on the filesystem.

---

← Previous: [Tool System: Registration, Effective Policy, and Built-in Categories](./12-tool-system.md) · Next: [Agent Loop Hooks: Inventory, Priority, and before_tool_call in Depth](./14-hooks.md) →
