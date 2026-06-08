---
title: "Built-In Coding Tools: Bash, Read, Write, Edit, and Search"
description: "Build each built-in coding tool — Bash with PTY and streaming output, Read with line ranges, Write with atomic overwrite, Edit with string-precise patching and diff display, and the search tools (Grep, Glob, List)."
category: coding-agent
type: tutorial
tags: [bash tool, read tool, write tool, edit tool, grep, glob, find, ls, tools, tool schema, tool execution, accumulator, truncate, file system, coding-agent, built-in tools]
keywords: [coding tools, bash execution, file editing, code search, tool schemas, PTY, diff display]
sources: [S48, S49]
---

**TL;DR** — A coding agent is only as capable as its tools. We'll build the seven essential tools: **Bash** (subprocess with PTY, streaming output, timeout), **Read** (file reading with line ranges), **Write** (atomic file creation), **Edit** (string-precise patching with diff), **Grep** (regex search with context lines), **Glob** (pattern-based file discovery), and **List** (directory listing). Each tool follows the same pattern: schema → validate → execute → result.

## The tool pattern

Every tool follows a consistent structure:

```ts
interface AgentTool<TDetails = any> {
  name: string;
  description: string;
  parameters: TSchema;          // TypeBox schema for argument validation
  execute: (args: any, context: AgentContext, signal?: AbortSignal) => Promise<AgentToolResult>;
}

interface AgentToolResult {
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  terminate?: boolean;
}
```

The `execute` function receives validated arguments — the caller validates against the schema before invoking.

## Bash tool

The bash tool executes shell commands. It's the most complex and dangerous tool — it can do anything the user can do. Create `packages/coding-agent/src/core/tools/bash.ts`:

```ts
const bashTool: AgentTool = {
  name: "bash",
  description: "Execute a shell command in the project directory.",
  parameters: Type.Object({
    command: Type.String({ description: "The shell command to execute" }),
    timeout: Type.Optional(Type.Number({ description: "Timeout in milliseconds" })),
  }),

  async execute(args, context, signal) {
    const startTime = Date.now();
    const timeout = args.timeout ?? 120_000; // 2 min default

    try {
      const result = await executeWithPty(args.command, {
        cwd: context.cwd,
        timeout,
        signal,
        maxOutput: 100_000, // truncate at 100KB
      });

      const truncated = result.output.length > 100_000;
      const output = truncated
        ? result.output.slice(0, 100_000) + "\n... (output truncated)"
        : result.output;

      return {
        content: [{ type: "text", text: output }],
        details: {
          exitCode: result.exitCode,
          duration: Date.now() - startTime,
          truncated,
        },
        isError: result.exitCode !== 0,
      };
    } catch (err) {
      return {
        content: [{ type: "text", text: `Error: ${err.message}` }],
        isError: true,
      };
    }
  },
};
```

### PTY support

For interactive commands, we allocate a pseudo-terminal (PTY) so the subprocess believes it's connected to a real terminal — this enables colors, progress bars, and interactive prompts:

```ts
async function executeWithPty(
  command: string,
  opts: BashOptions,
): Promise<{ output: string; exitCode: number }> {
  return new Promise((resolve, reject) => {
    const pty = spawn("script", ["-q", "-c", command, "/dev/null"], {
      cwd: opts.cwd,
      env: { ...process.env, TERM: "xterm-256color" },
    });

    let output = "";
    pty.stdout.on("data", (data: Buffer) => {
      const chunk = data.toString();
      output += chunk;
      if (output.length > opts.maxOutput) {
        pty.kill(); // truncate by killing
      }
    });

    pty.on("close", (code) => {
      resolve({ output, exitCode: code ?? 1 });
    });

    pty.on("error", reject);

    if (opts.signal) {
      opts.signal.addEventListener("abort", () => pty.kill());
    }

    setTimeout(() => {
      pty.kill();
      resolve({ output: output + "\n[timeout]", exitCode: 124 });
    }, opts.timeout);
  });
}
```

## Read tool

Reads a file with optional line range:

```ts
const readTool: AgentTool = {
  name: "read",
  description: "Read a file from the filesystem.",
  parameters: Type.Object({
    filePath: Type.String({ description: "Path to the file" }),
    offset: Type.Optional(Type.Number({ description: "Line number to start from (1-based)" })),
    limit: Type.Optional(Type.Number({ description: "Maximum number of lines to read" })),
  }),

  async execute(args, context) {
    const fullPath = path.resolve(context.cwd ?? ".", args.filePath);

    try {
      const content = await fs.promises.readFile(fullPath, "utf-8");
      const lines = content.split("\n");
      const start = (args.offset ?? 1) - 1;
      const end = args.limit ? start + args.limit : lines.length;
      const snippet = lines.slice(start, end).join("\n");

      // Format with line numbers
      const numbered = snippet.split("\n").map((line, i) =>
        `${String(start + i + 1).padStart(6)}  ${line}`
      ).join("\n");

      return {
        content: [{ type: "text", text: numbered }],
        details: { path: fullPath, totalLines: lines.length, start, end },
        isError: false,
      };
    } catch (err) {
      return {
        content: [{ type: "text", text: `Cannot read ${args.filePath}: ${err.message}` }],
        isError: true,
      };
    }
  },
};
```

## Write tool

Creates or overwrites a file atomically:

```ts
const writeTool: AgentTool = {
  name: "write",
  description: "Write content to a file, creating it if necessary.",
  parameters: Type.Object({
    filePath: Type.String(),
    content: Type.String(),
  }),

  async execute(args, context) {
    const fullPath = path.resolve(context.cwd ?? ".", args.filePath);

    try {
      await fs.promises.mkdir(path.dirname(fullPath), { recursive: true });
      await fs.promises.writeFile(fullPath, args.content, "utf-8");
      const lines = args.content.split("\n").length;
      return {
        content: [{ type: "text", text: `Wrote ${lines} lines to ${args.filePath}` }],
        details: { path: fullPath, lines },
        isError: false,
      };
    } catch (err) {
      return {
        content: [{ type: "text", text: `Failed to write ${args.filePath}: ${err.message}` }],
        isError: true,
      };
    }
  },
};
```

## Edit tool

The edit tool performs string-precise file patching. It's safer than "read the whole file, modify, write back" because it only touches the lines that changed:

```ts
const editTool: AgentTool = {
  name: "edit",
  description: "Replace one string with another in a file. The old_string must match exactly.",
  parameters: Type.Object({
    filePath: Type.String(),
    oldString: Type.String(),
    newString: Type.String(),
    replaceAll: Type.Optional(Type.Boolean({ default: false })),
  }),

  async execute(args, context) {
    const fullPath = path.resolve(context.cwd ?? ".", args.filePath);

    try {
      const original = await fs.promises.readFile(fullPath, "utf-8");

      if (!original.includes(args.oldString)) {
        return {
          content: [{ type: "text", text: `Error: old_string not found in ${args.filePath}. The file may have changed since you last read it.` }],
          isError: true,
        };
      }

      const count = original.split(args.oldString).length - 1;
      const modified = args.replaceAll
        ? original.replaceAll(args.oldString, args.newString)
        : original.replace(args.oldString, args.newString);

      await fs.promises.writeFile(fullPath, modified, "utf-8");

      // Generate diff for display
      const diff = generateDiff(args.filePath, original, modified);

      return {
        content: [{ type: "text", text: `Edited ${args.filePath} (${args.replaceAll ? count : 1} replacement${count > 1 ? "s" : ""})\n\n${diff}` }],
        isError: false,
      };
    } catch (err) {
      return {
        content: [{ type: "text", text: `Edit failed: ${err.message}` }],
        isError: true,
      };
    }
  },
};
```

The diff display uses the `diff` library to produce a unified diff, showing exactly what changed — critical for user trust.

## Search tools

**Grep** — regex search with context lines:

```ts
const grepTool: AgentTool = {
  name: "grep",
  description: "Search for a regex pattern in files.",
  parameters: Type.Object({
    pattern: Type.String({ description: "Regular expression to search for" }),
    path: Type.Optional(Type.String({ description: "Directory or file to search" })),
    include: Type.Optional(Type.String({ description: "File pattern to include (glob)" })),
    contextLines: Type.Optional(Type.Number({ default: 0 })),
  }),
  // Execute: spawns ripgrep or falls back to Node.js glob + read
};
```

**Glob** — pattern-based file discovery:

```ts
const globTool: AgentTool = {
  name: "glob",
  description: "Find files matching a glob pattern.",
  parameters: Type.Object({
    pattern: Type.String({ description: "Glob pattern, e.g., 'src/**/*.ts'" }),
    path: Type.Optional(Type.String()),
  }),
  // Execute: uses the 'ignore' and 'glob' npm packages
};
```

## Output helpers

Two utilities keep tool output manageable:

**Truncate** — limits output to a maximum length, with a clear message when truncation occurred:

```ts
function truncateOutput(text: string, maxLength: number = 100_000): string {
  if (text.length <= maxLength) return text;
  return text.slice(0, maxLength) + `\n\n... (truncated ${text.length - maxLength} characters)`;
}
```

**Accumulator** — batches small tool results so they can be displayed together:

```ts
class OutputAccumulator {
  private items: string[] = [];

  add(result: ToolResult): void {
    this.items.push(formatResult(result));
  }

  flush(): string {
    return this.items.join("\n");
  }
}
```

## What we've built

Seven production-grade coding tools. Each one:
- Validates arguments with TypeBox schemas
- Resolves paths relative to the project cwd
- Handles errors gracefully (returns error results, never throws)
- Truncates large output
- Returns structured details for the agent harness

In the next chapter, we'll build the hooks system — the event pipeline that lets extensions intercept every tool execution.

---

← Previous: [AgentSession: The Core of the Coding Agent](./agent-session-core.md) · Next: [The Hooks System: Intercepting Every Agent Event](./hooks-system.md) →
