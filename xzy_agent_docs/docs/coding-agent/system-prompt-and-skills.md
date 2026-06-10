---
title: "System Prompt Construction and Skill Loading"
description: "How buildSystemPrompt assembles tools, guidelines, context files, and skills into the model's opening instructions, and how SKILL.md files are discovered and loaded."
category: coding-agent
type: tutorial
tags: [buildSystemPrompt, system prompt, context files, tools, skills, SKILL.md, frontmatter, date, cwd, guidelines, skill loader, collision precedence, coding-agent, formatSkillsForPrompt, loadSkills, loadSkillsFromDir, SkillFrontmatter, Skill, BuildSystemPromptOptions]
keywords: [system prompt assembly, skill discovery, skill precedence, AGENTS.md, context injection, tool list, agent instructions, prompt construction, skill frontmatter validation]
sources: [S56, S57, S46]
---

**TL;DR** — Every time the agent starts a turn, it sends the model a system prompt that describes the available tools, behavioural guidelines, project context files, and any loaded skills. This chapter walks through `buildSystemPrompt` — the function that assembles that prompt — and `loadSkills` — the function that discovers `SKILL.md` files from multiple locations and deduplicates them by name. By the end you will understand exactly what the model is told about itself, and how you can extend those instructions with your own skill files.

# System Prompt Construction and Skill Loading

## The problem: the model starts each session knowing nothing

A language model has no inherent knowledge of which tools it can call, what directory it is working in, or what conventions your project follows. All of that must be delivered in text — specifically, in the *system prompt* sent at the very start of every conversation.

Our agent therefore needs a function that assembles a complete, self-describing opening message: here are your tools, here are the rules you should follow, here is any project-specific guidance, here are the skills you can load on demand, and — critically — here is the current date and working directory so the model can reason accurately about the filesystem.

That function is `buildSystemPrompt`, defined in `src/core/system-prompt.ts`. We will walk through it section by section.

## The shape of `buildSystemPrompt`

`buildSystemPrompt` accepts a single options object typed as `BuildSystemPromptOptions` and returns a plain string — the assembled prompt. Here is the full interface, drawn directly from the source:

```ts
// Simplified view of BuildSystemPromptOptions — all fields from source
export interface BuildSystemPromptOptions {
  /** Custom system prompt (replaces default). */
  customPrompt?: string;
  /** Tools to include in prompt. Default: [read, bash, edit, write] */
  selectedTools?: string[];
  /** Optional one-line tool snippets keyed by tool name. */
  toolSnippets?: Record<string, string>;
  /** Additional guideline bullets appended to the default system prompt guidelines. */
  promptGuidelines?: string[];
  /** Text to append to system prompt. */
  appendSystemPrompt?: string;
  /** Working directory. */
  cwd: string;
  /** Pre-loaded context files. */
  contextFiles?: Array<{ path: string; content: string }>;
  /** Pre-loaded skills. */
  skills?: Skill[];
}
```

Every field except `cwd` is optional. The caller — `AgentSession`, which we covered in [AgentSession and the session core](./agent-session-core.md) — provides these values when it initialises the agent. Let's trace what happens to each one.

## Step 1: date and working directory

The first thing `buildSystemPrompt` does, regardless of all the other options, is compute two environmental facts the model will always need:

```ts
const now = new Date();
const year = now.getFullYear();
const month = String(now.getMonth() + 1).padStart(2, "0");
const day = String(now.getDate()).padStart(2, "0");
const date = `${year}-${month}-${day}`;

const promptCwd = resolvedCwd.replace(/\\/g, "/");
```

The date is formatted as `YYYY-MM-DD`. The working directory path has backslashes normalised to forward slashes so the model sees a consistent POSIX-style path regardless of the host operating system.

These two values are appended **last** in the final prompt, after every other section:

```
Current date: 2025-06-05
Current working directory: /home/user/my-project
```

Placing them last is a deliberate choice: the model reads them as the freshest, most grounded facts — the contextual anchor for everything it is about to do.

## Step 2: the custom prompt path

`buildSystemPrompt` has two distinct branches. If `customPrompt` is provided, the function uses it as the base text and skips the default preamble entirely. This allows SDK callers or configuration files to replace the built-in instructions with something of their own design.

Even on the custom path, the function still appends:

1. An `appendSystemPrompt` section, if provided.
2. Project context files (in the same `<project_context>` XML structure described below).
3. Skills (if the `read` tool is available — explained in step 4).
4. The date and working directory.

If `customPrompt` is omitted — which is the typical case — execution falls through to the **default prompt** path. That is what we will focus on next.

## Step 3: the default prompt — tools and guidelines

The default prompt opens with a fixed preamble that introduces the agent to the model. Here is the structure of that preamble (genericised per the brand rules of this library):

```
You are an expert coding assistant operating inside xzy, a coding agent harness.
You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: <one-line description>
- bash: <one-line description>
- edit: <one-line description>
- write: <one-line description>

In addition to the tools above, you may have access to other custom tools
depending on the project.

Guidelines:
- <guideline 1>
- <guideline 2>
...
```

Two things are worth noting here.

### Which tools appear in the list

A tool appears in the "Available tools" section only when the caller provides a one-line snippet for it via `toolSnippets`. If no snippet is provided, the tool is not listed — even if it is enabled.

The set of enabled tools defaults to `["read", "bash", "edit", "write"]` when `selectedTools` is not specified. You can restrict this set (e.g. pass `["read"]` for a read-only agent) or expand it with additional built-in tools such as `grep`, `find`, and `ls`. See [Built-In Coding Tools](./built-in-tools.md) for the full list of available tool names.

```ts
const tools = selectedTools || ["read", "bash", "edit", "write"];
const visibleTools = tools.filter((name) => !!toolSnippets?.[name]);
const toolsList =
  visibleTools.length > 0
    ? visibleTools.map((name) => `- ${name}: ${toolSnippets![name]}`).join("\n")
    : "(none)";
```

If `toolSnippets` is empty or undefined, the list reads `(none)`. The model can still use tools — the list is informational, not a capability gate.

### How guidelines are assembled

Guidelines are accumulated into an ordered, deduplicated list. The function first checks which tools are present and adds a conditional guideline:

```ts
const hasBash = tools.includes("bash");
const hasGrep = tools.includes("grep");
const hasFind = tools.includes("find");
const hasLs   = tools.includes("ls");

if (hasBash && !hasGrep && !hasFind && !hasLs) {
  addGuideline("Use bash for file operations like ls, rg, find");
}
```

This means: if `bash` is available but the dedicated file-exploration tools (`grep`, `find`, `ls`) are not, the model is told to use `bash` for those tasks instead. Once the conditional guidelines are placed, any caller-supplied `promptGuidelines` are appended (deduplicated). Finally, two guidelines are always present:

- `"Be concise in your responses"`
- `"Show file paths clearly when working with files"`

The deduplication is done with a `Set<string>` so that a caller-supplied guideline that happens to match a built-in one does not appear twice.

## Step 4: folding in project context files

After the tool list and guidelines, `buildSystemPrompt` appends the pre-loaded context files. Context files are typically project-level instruction documents — for example, an `AGENTS.md` or `CLAUDE.md` file placed at the root of a repository.

The README (S46) describes the discovery pattern: the agent loads files named `AGENTS.md` (or `CLAUDE.md`) from the global config directory and by walking parent directories up from the current working directory, concatenating all matches. By the time `buildSystemPrompt` is called, that loading has already happened and the results are passed in via `contextFiles`.

Inside `buildSystemPrompt`, each file is wrapped in XML tags:

```ts
if (contextFiles.length > 0) {
  prompt += "\n\n<project_context>\n\n";
  prompt += "Project-specific instructions and guidelines:\n\n";
  for (const { path: filePath, content } of contextFiles) {
    prompt += `<project_instructions path="${filePath}">\n${content}\n</project_instructions>\n\n`;
  }
  prompt += "</project_context>\n";
}
```

The `path` attribute tells the model exactly which file each block of instructions came from. This matters when the model needs to report that it is following a specific project convention — it can cite the source file. An example of the rendered output for a project with one context file might look like:

```
<project_context>

Project-specific instructions and guidelines:

<project_instructions path="/home/user/my-project/AGENTS.md">
Always run tests before committing.
Use conventional commit messages.
</project_instructions>

</project_context>
```

## Step 5: appending skills

After context files come skills. Skills — introduced fully in the next section — are on-demand capability packages: each one is a Markdown file with a name, a description, and a body of instructions. When the model encounters a task that matches a skill's description, it reads the skill file for detailed instructions.

The skills section is only added when **two conditions** are both true:
1. The `read` tool is available (skills are loaded by the model calling `read` on the skill file path).
2. At least one skill is present in the `skills` array.

```ts
const hasRead = tools.includes("read");

if (hasRead && skills.length > 0) {
  prompt += formatSkillsForPrompt(skills);
}
```

`formatSkillsForPrompt` is imported from `skills.ts` and produces an XML block. We will look at it in detail in the skills section below.

## The assembled prompt — a concrete example

Putting all the pieces together, here is a simplified example of what the model receives at the start of a session. Assume one tool, one guideline, one context file, and one skill:

```
You are an expert coding assistant operating inside xzy, a coding agent harness.
You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: Read any file from the filesystem

In addition to the tools above, you may have access to other custom tools
depending on the project.

Guidelines:
- Be concise in your responses
- Show file paths clearly when working with files

<project_context>

Project-specific instructions and guidelines:

<project_instructions path="/home/user/my-project/AGENTS.md">
Always run tests before committing.
</project_instructions>

</project_context>

The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory
(parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.

<available_skills>
  <skill>
    <name>deploy</name>
    <description>Use this skill when the user asks to deploy the application.</description>
    <location>/home/user/.xzy/skills/deploy/SKILL.md</location>
  </skill>
</available_skills>

Current date: 2025-06-05
Current working directory: /home/user/my-project
```

Notice the flow: the model learns its role and tools first, then behavioural rules, then project conventions, then available skills, then environmental facts. Each section builds on the last.

---

## Skills: the on-demand capability standard

Now we have a problem. The agent's core tools (`read`, `bash`, `edit`, `write`) are general-purpose. But some tasks benefit from richer, domain-specific instructions that would be wasteful to include in every prompt — they are only needed occasionally. We need a way to define those instructions once, advertise their existence to the model, and let the model pull them in only when relevant.

That is what skills solve. A skill is a Markdown file with YAML frontmatter. The model sees only the name and description in the system prompt. When it decides a skill is relevant, it uses `read` to load the file and follows the instructions it finds there.

### The SKILL.md format

A skill is defined by a file named `SKILL.md` (for named skills in their own directory) or any `.md` file directly in a skills root. The file must have YAML frontmatter with at least a `description` field.

Here is a complete, working example:

```markdown
---
name: run-tests
description: >
  Use this skill when the user asks to run, execute, or check tests
  for the current project.
---

# Run Tests

## Before you start

Read the project's `package.json` to find the test script. Also check
for a `jest.config.ts` or `vitest.config.ts` at the project root.

## Steps

1. Run `bash` with the test command (e.g. `npm test` or `npx vitest`).
2. If tests fail, report the failing test names and their error messages.
3. Do not attempt to fix failing tests unless the user explicitly asks.
```

The frontmatter fields the loader reads are typed as `SkillFrontmatter`:

```ts
export interface SkillFrontmatter {
  name?: string;
  description?: string;
  "disable-model-invocation"?: boolean;
  [key: string]: unknown;
}
```

| Field | Required | Default | Constraint |
|---|---|---|---|
| `name` | No | Parent directory name | Lowercase a-z, 0-9, hyphens only; max 64 chars; no leading/trailing/consecutive hyphens |
| `description` | **Yes** | — | Non-empty; max 1024 chars |
| `disable-model-invocation` | No | `false` | When `true`, skill is hidden from the model's `<available_skills>` list |

**Name fallback.** If `name` is omitted from the frontmatter, the loader uses the name of the directory that contains the `SKILL.md` file. For a flat `.md` file directly in the skills root, the file's own stem (without `.md`) would serve as the identifier via the directory name heuristic.

**`disable-model-invocation`.** When this is `true`, the skill is loaded by `loadSkills` but filtered out by `formatSkillsForPrompt`. It will not appear in `<available_skills>`. This is useful for skills that are only invoked explicitly by the user via a command (e.g. `/skill:name`), not auto-selected by the model.

**Missing description is fatal.** If `description` is absent or blank, `loadSkillFromFile` returns `null` for the skill — it will not be added to the loaded set. Name validation failures produce warnings but do not block loading.

### What the model sees: `formatSkillsForPrompt`

`formatSkillsForPrompt` takes the array of loaded `Skill` objects and produces the `<available_skills>` XML block inserted into the prompt. Skills with `disableModelInvocation: true` are filtered out before formatting:

```ts
export function formatSkillsForPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter((s) => !s.disableModelInvocation);

  if (visibleSkills.length === 0) {
    return "";
  }

  const lines = [
    "\n\nThe following skills provide specialized instructions for specific tasks.",
    "Use the read tool to load a skill's file when the task matches its description.",
    "When a skill file references a relative path, resolve it against the skill directory"
    + " (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.",
    "",
    "<available_skills>",
  ];

  for (const skill of visibleSkills) {
    lines.push("  <skill>");
    lines.push(`    <name>${escapeXml(skill.name)}</name>`);
    lines.push(`    <description>${escapeXml(skill.description)}</description>`);
    lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
    lines.push("  </skill>");
  }

  lines.push("</available_skills>");

  return lines.join("\n");
}
```

The `<location>` tag gives the model the absolute path to the skill file, which it passes to the `read` tool. The instruction to resolve relative paths *against the skill directory* is important: a skill body can reference other files using relative paths (e.g. a template or a config), and the model needs to know the base for those resolutions.

---

## Discovering skills: where `loadSkills` looks

Skills can live in several places. The `loadSkills` function — also in `skills.ts` — collects them all, deduplicates by name, and returns a flat `Skill[]` that is then passed to `buildSystemPrompt`.

### The search roots

`loadSkills` accepts this options object:

```ts
export interface LoadSkillsOptions {
  /** Working directory for project-local skills. */
  cwd: string;
  /** Agent config directory for global skills. */
  agentDir: string;
  /** Explicit skill paths (files or directories) */
  skillPaths: string[];
  /** Include default skills directories. */
  includeDefaults: boolean;
}
```

When `includeDefaults` is `true`, it scans two standard directories in this order:

1. **Global skills** — `<agentDir>/skills/` — typically `~/.xzy/skills/`
2. **Project skills** — `<cwd>/.xzy/skills/`

Any additional paths in `skillPaths` are scanned after the defaults (or instead of them, if `includeDefaults` is `false`). A `skillPath` entry can be a single `.md` file or a directory.

### Discovery rules inside a directory

`loadSkillsFromDir` applies these rules when scanning a directory:

1. **`SKILL.md` terminates recursion.** If a directory contains a file named `SKILL.md`, the loader treats that directory as a single skill, loads the `SKILL.md`, and does **not** recurse into sub-directories of that directory.
2. **Flat `.md` files at the root.** If the directory does not contain a `SKILL.md`, any `.md` files directly in that directory are each loaded as individual skills.
3. **Recurse into sub-directories.** Sub-directories that do not contain a `SKILL.md` themselves are recursed into, following the same rules.
4. **Skip hidden entries.** Any entry whose name starts with `.` is ignored.
5. **Skip `node_modules`.** The `node_modules` directory is never scanned.
6. **Respect ignore files.** The loader reads `.gitignore`, `.ignore`, and `.fdignore` files and applies their patterns, so build outputs or large vendored directories are not scanned.

A typical skills directory layout:

```
~/.xzy/skills/
├── deploy/
│   └── SKILL.md          ← named skill "deploy" (or name from frontmatter)
├── run-tests/
│   └── SKILL.md          ← named skill "run-tests"
└── quick-note.md         ← flat skill, name from frontmatter or "quick-note"
```

### Collision precedence

When two skills from different roots share the same name, the one loaded **first** wins. Because global skills are loaded before project skills, a name collision means the **global skill takes precedence**.

```
Load order:
  1. ~/.xzy/skills/      ← global (loaded first — wins on collision)
  2. .xzy/skills/        ← project (loaded second — loses on collision)
  3. skillPaths entries  ← explicit (loaded last)
```

The collision is recorded as a diagnostic of type `"collision"` in the returned `diagnostics` array. This diagnostic carries enough information for a caller to surface a warning to the user:

```ts
collisionDiagnostics.push({
  type: "collision",
  message: `name "${skill.name}" collision`,
  path: skill.filePath,          // the losing file
  collision: {
    resourceType: "skill",
    name: skill.name,
    winnerPath: existing.filePath, // the winning file
    loserPath: skill.filePath,
  },
});
```

The table below summarises precedence:

| Source | Load order | On name collision |
|---|---|---|
| `~/.xzy/skills/` (global) | 1st | Wins |
| `.xzy/skills/` (project) | 2nd | Loses |
| Explicit `skillPaths` | 3rd | Loses |

**Symlink deduplication.** If two different skill paths resolve to the same physical file (via symlinks), the second is silently skipped. This prevents the same skill body from being registered twice under the same or different names.

### A complete `loadSkills` usage example

Here is a simplified illustration of how `loadSkills` is called and its result used:

```ts
// Simplified view of how AgentSession calls loadSkills
import { loadSkills } from "./skills.ts";
import { buildSystemPrompt } from "./system-prompt.ts";

const { skills, diagnostics } = loadSkills({
  cwd: "/home/user/my-project",
  agentDir: "/home/user/.xzy",
  skillPaths: [],      // no explicit extra paths
  includeDefaults: true,
});

// Any diagnostics (warnings, collisions) can be surfaced to the user here
for (const diag of diagnostics) {
  if (diag.type === "collision") {
    console.warn(`Skill collision: "${diag.collision?.name}" — using ${diag.collision?.winnerPath}`);
  }
}

const prompt = buildSystemPrompt({
  cwd: "/home/user/my-project",
  selectedTools: ["read", "bash", "edit", "write"],
  toolSnippets: {
    read:  "Read any file from the filesystem",
    bash:  "Run a shell command",
    edit:  "Apply a targeted edit to a file",
    write: "Write or overwrite a file",
  },
  skills,
});
```

`skills` is the flat, deduplicated array — ready to be handed straight to `buildSystemPrompt`. The prompt it produces will include the `<available_skills>` block listing every visible skill by name, description, and file path.

---

## Putting it all together: what the model knows

By the time the first turn begins, the model has received:

| Prompt section | Source |
|---|---|
| Role preamble + tool list | Default text in `buildSystemPrompt`, shaped by `toolSnippets` |
| Behavioural guidelines | Built-in rules + caller-supplied `promptGuidelines` |
| Project context files | `contextFiles` array (pre-loaded `AGENTS.md` / `CLAUDE.md` files) |
| Available skills | `formatSkillsForPrompt(skills)` — name, description, file path per skill |
| Current date | Computed by `buildSystemPrompt` at call time |
| Current working directory | `cwd`, normalised to forward slashes |

Nothing is hidden from the model; nothing is invented. The system prompt is assembled deterministically from the options the caller provides. If you want to understand why the model behaved a certain way, you can reconstruct the prompt by tracing `buildSystemPrompt` with the same inputs.

The next chapter covers how the session tree evolves over time — branching, resuming, and switching between conversation paths.

---

← Previous: [Built-In Coding Tools: Bash, Read, Write, Edit, and More](./built-in-tools.md) · Next: [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) →
