---
title: "System Prompt Construction and Skill Loading"
description: "Build the system prompt assembler that combines tool descriptions, skill definitions, context files, and behavioral guidelines into the prompt that defines the agent's personality, then implement SKILL.md discovery and loading with collision precedence."
category: coding-agent
type: tutorial
tags: [system prompt, buildSystemPrompt, context files, tools, skills, SKILL.md, frontmatter, skill loader, collision precedence, guidelines, coding-agent]
keywords: [system prompt, SKILL.md, skill loading, prompt templates, tool descriptions]
sources: [S25, S26, S31]
---

**TL;DR** — The system prompt defines who the agent is and what it can do. We'll build a template-based prompt assembler that injects tool descriptions, skill instructions, project context, and behavioral guidelines into a structured prompt. Then we'll implement SKILL.md discovery — finding and loading skill files with precedence (project > user > global) and conflict resolution.

## The system prompt template

The coding agent's system prompt is assembled from a template. Create `packages/coding-agent/src/core/system-prompt.ts`:

```ts
const CODING_AGENT_TEMPLATE = `
You are a coding agent — an AI that helps developers write, edit, and understand code.

## Your capabilities
You have access to these tools:
{{tools}}

## Skills
The following skills are available:
{{skills}}

## Project context
- Working directory: {{cwd}}
- Current date: {{date}}

## Guidelines
{{guidelines}}

## Rules
1. Always read a file before editing it.
2. Explain your changes before making them.
3. If a command fails, read the error and adjust — don't retry the same command.
4. When you're done with a task, tell the user what you changed and why.
5. Never make changes the user didn't ask for.
`;

export function buildSystemPrompt(params: {
  tools: AgentTool[];
  skills: SkillDefinition[];
  cwd: string;
  date: string;
  guidelines?: string;
}): string {
  return CODING_AGENT_TEMPLATE
    .replace("{{tools}}", formatToolSection(params.tools))
    .replace("{{skills}}", formatSkillSection(params.skills))
    .replace("{{cwd}}", params.cwd)
    .replace("{{date}}", params.date)
    .replace("{{guidelines}}", params.guidelines ?? getDefaultGuidelines());
}
```

### Tool descriptions

Tools are formatted with their name, description, and parameter schema:

```ts
function formatToolSection(tools: AgentTool[]): string {
  return tools.map(t => {
    const schema = JSON.stringify(t.parameters, null, 2);
    return `### ${t.name}\n${t.description}\n\`\`\`json\n${schema}\n\`\`\``;
  }).join("\n\n");
}
```

### Skill descriptions

Skills are formatted with their name, description, and truncated instructions:

```ts
function formatSkillSection(skills: SkillDefinition[]): string {
  if (skills.length === 0) return "No skills loaded.";
  return skills.map(s =>
    `- **${s.name}**: ${s.description}`
  ).join("\n");
}
```

Skill instructions are injected separately into the conversation when the skill is triggered, not into the system prompt (which would waste context window space).

## SKILL.md discovery

Skills are defined in Markdown files with YAML frontmatter. Create `packages/agent-core/src/harness/skills.ts`:

```ts
interface SkillDefinition {
  name: string;
  description: string;
  instructions: string;
  source: string;            // file path or "inline"
  triggers?: string[];       // keywords that activate this skill
}

function parseSkillFile(content: string, source: string): SkillDefinition | null {
  const match = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
  if (!match) return null;

  const frontmatter = parseYaml(match[1]);
  const instructions = match[2].trim();

  if (!frontmatter.name || !frontmatter.description) {
    return null; // invalid skill — missing required fields
  }

  return {
    name: frontmatter.name,
    description: frontmatter.description,
    instructions,
    source,
    triggers: frontmatter.triggers,
  };
}
```

### Discovery paths

Skills are discovered in three locations, with the first match winning:

```ts
function discoverSkills(paths: string[]): SkillDefinition[] {
  const seen = new Set<string>();
  const skills: SkillDefinition[] = [];

  for (const dir of paths) {
    const skillFiles = findSkillFiles(dir);
    for (const file of skillFiles) {
      const content = fs.readFileSync(file, "utf-8");
      const skill = parseSkillFile(content, file);
      if (skill && !seen.has(skill.name)) {
        seen.add(skill.name);
        skills.push(skill);
      }
    }
  }

  return skills;
}

function findSkillFiles(dir: string): string[] {
  if (!fs.existsSync(dir)) return [];
  const results: string[] = [];
  walkDir(dir, (file) => {
    if (file.endsWith("SKILL.md")) {
      results.push(file);
    }
  });
  return results;
}
```

### Precedence

The search order gives local project skills priority:

1. **`<cwd>/.claude/skills/`** — project-specific skills (highest priority)
2. **`~/.claude/skills/`** — user skills
3. **`<install-dir>/skills/`** — built-in/system skills (lowest priority)

If two skills have the same `name`, the first one found wins. This lets projects override built-in skills.

## What we've built

- **System prompt template** with variable substitution for tools, skills, cwd, date, and guidelines
- **SKILL.md parser** extracting YAML frontmatter and Markdown instructions
- **Skill discovery** across project, user, and system paths with precedence-based collision resolution

---

← Previous: [The Hooks System](./hooks-system.md) · Next: [Sessions, Branching, and the Session Tree](./sessions-and-branching.md) →
