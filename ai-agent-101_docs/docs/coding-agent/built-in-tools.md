---
title: "Built-In Coding Tools: Bash, Read, Write, Edit, and More"
description: "A guided walkthrough of every built-in tool the coding agent uses to touch the filesystem and shell — schemas, execution logic, and error contracts."
category: coding-agent
type: tutorial
tags: [bash tool, read tool, write tool, edit tool, edit-diff, grep, find, ls, accumulator, truncate, queue, path-utils, tool definition, coding-agent, built-in tools, file system, TypeBox, AgentTool, ToolDefinition, fuzzy match, CRLF, BOM, ripgrep, fd, output truncation, mutation queue]
keywords: [shell command execution, file read write, exact text replacement, directory listing, glob search, content search, output streaming, temp file, path resolution, line limit, byte limit]
sources: [S55, S88]
---

**TL;DR** — The built-in tools are what makes our agent a *coding* agent. Each one wraps a file-system or shell operation inside the `AgentTool` contract: a TypeBox schema the model fills in, an `execute` function that runs the operation, and a result or error the agent loop feeds back. This chapter builds every tool in turn — read, write, edit, bash, grep, find, ls — then unpacks the shared plumbing (output truncation, file-mutation serialisation, path resolution) that keeps the whole set safe and bounded.

# Built-In Coding Tools: Bash, Read, Write, Edit, and More

## The shape every tool must fit

Before we write any tool, let's recall the two contracts it must satisfy, because they govern every decision in this chapter.

**AgentTool** (from [the agent loop chapter](../agent-loop/the-agent-loop.md)) is the interface the core runtime calls. It carries: a `name` string the model uses to invoke the tool, a `parameters` schema the model must fill in, and an `execute` async function. The execute signature is:

```ts
execute(
  toolCallId: string,
  params: <your-schema-type>,
  signal?: AbortSignal,
  onUpdate?: (partial) => void,
): Promise<{ content: ContentBlock[]; details?: Details }>
```

Where `ContentBlock` is typically `{ type: "text"; text: string }` — the text the agent loop inserts into the conversation as the tool result.

**ToolDefinition** is the richer internal type defined in `src/core/extensions/types.ts`. It extends `AgentTool` with rendering hooks (`renderCall`, `renderResult`) and metadata used by the system prompt. Every built-in is authored as a `ToolDefinition` and then wrapped by `wrapToolDefinition()` to produce the `AgentTool` the runtime consumes:

```ts
// Simplified view of wrapToolDefinition (tool-definition-wrapper.ts)
export function wrapToolDefinition<TDetails>(
  definition: ToolDefinition<any, TDetails>,
): AgentTool<any, TDetails> {
  return {
    name: definition.name,
    label: definition.label,
    description: definition.description,
    parameters: definition.parameters,
    prepareArguments: definition.prepareArguments,
    execute: (toolCallId, params, signal, onUpdate) =>
      definition.execute(toolCallId, params, signal, onUpdate, /* ctx */),
  };
}
```

So the pattern for every tool below is: define the TypeBox schema → implement `execute` → call `wrapToolDefinition` → export the result.

**What is TypeBox?** TypeBox is a library that lets you build JSON-Schema-compatible schemas in TypeScript. `Type.Object({...})` creates an object schema; `Type.String({...})`, `Type.Number({...})`, `Type.Optional(...)` are its field types. The `Static<typeof schema>` utility extracts the matching TypeScript type. The model receives the schema as a JSON Schema, and its response is validated against it before `execute` is called.

**AgentSession and the tool registry** (from [AgentSession: The Core of the Coding Agent](./agent-session-core.md)) is where all tools are registered. `AgentSession` holds a map of tool names to `ToolDefinition` objects, exposes that set to the system-prompt builder (so the model knows which tools exist), and dispatches incoming tool calls to the right `execute` function.

Now let's build the tools.

---

## The read tool — reading a file with optional line ranges

The first thing the agent needs to do is look at code. Let's start with `read`.

### The schema

```ts
// read.ts — TypeBox schema
const readSchema = Type.Object({
  path: Type.String({ description: "Path to the file to read (relative or absolute)" }),
  offset: Type.Optional(Type.Number({ description: "Line number to start reading from (1-indexed)" })),
  limit:  Type.Optional(Type.Number({ description: "Maximum number of lines to read" })),
});

export type ReadToolInput = Static<typeof readSchema>;
```

Three fields: `path` (required), and optional `offset` and `limit` for windowed reads. `offset` is **1-indexed** — that is, `offset: 1` starts at the very first line.

### The execute logic

When `execute` runs, it resolves the path, checks readability, then reads:

1. **Path resolution** — `resolveReadPathAsync(path, cwd)` calls into `path-utils.ts` (covered later in this chapter) to expand `~`, handle relative paths, and try macOS filename variants.
2. **Access check** — `ops.access(absolutePath)` throws if the file does not exist or is not readable. The error propagates out and the agent loop records it as a tool error.
3. **Text content** — the file is read as a Buffer, decoded as UTF-8, split on `\n` into lines.
4. **Offset application** — `offset` is converted from 1-indexed to 0-indexed array access. If `offset` is beyond the end of the file, the tool throws: `"Offset N is beyond end of file (M lines total)"`.
5. **Limit and truncation** — if the user specified `limit`, that many lines are taken; otherwise `truncateHead()` applies the default limits. If the result is truncated, the output text gains a continuation notice: `"[Showing lines 1-2000 of 2500. Use offset=2001 to continue.]"`.

A complete end-to-end example using the exported factory:

```ts
import { createReadTool } from "coding-agent";

const read = createReadTool(process.cwd());

// Read lines 51–60 of a large file
const result = await read.execute("call-1", { path: "./src/server.ts", offset: 51, limit: 10 });
// result.content[0].text contains lines 51-60
// result.content[0].text ends with "[40 more lines in file. Use offset=61 to continue.]"
//   when the file has 100 lines total
```

The test suite confirms these contracts:

```ts
// S88 — tools.test.ts (read tool)
it("should truncate files exceeding line limit", async () => {
  // 2500-line file → only first 2000 lines returned
  expect(output).toContain("Line 2000");
  expect(output).not.toContain("Line 2001");
  expect(output).toContain("[Showing lines 1-2000 of 2500. Use offset=2001 to continue.]");
});

it("should handle offset + limit together", async () => {
  // offset=41, limit=20 → lines 41-60
  expect(output).toContain("Line 41");
  expect(output).toContain("Line 60");
  expect(output).not.toContain("Line 61");
  expect(output).toContain("[40 more lines in file. Use offset=61 to continue.]");
});
```

**Image support.** The read tool also handles image files (JPEG, PNG, GIF, WebP). It detects the MIME type from file magic bytes (not extension), resizes the image to at most 2000×2000 pixels by default, and returns both a text note (`"Read image file [image/png]"`) and an `image` content block. If the current model does not support vision input, a note saying so is appended. Files with an image extension but non-image content are read as text.

### Error contract summary

| Condition | Result |
|---|---|
| File not found or not readable | Throws (agent loop records as error) |
| `offset` beyond end of file | Throws: `"Offset N is beyond end of file (M lines total)"` |
| Content fits within limits | `details` is `undefined` |
| Content truncated | `details.truncation` is set; continuation notice appended to text |

---

## The write tool — creating or overwriting a file

Now that the agent can read, it needs to write. The write tool is intentionally simple: it replaces the entire file contents in one shot.

### The schema

```ts
// write.ts
const writeSchema = Type.Object({
  path:    Type.String({ description: "Path to the file to write (relative or absolute)" }),
  content: Type.String({ description: "Content to write to the file" }),
});
```

No optional fields — `path` and `content` are always required.

### The execute logic

```ts
async execute(_toolCallId, { path, content }, signal?) {
  const absolutePath = resolveToCwd(path, cwd);
  const dir = dirname(absolutePath);

  return withFileMutationQueue(absolutePath, async () => {
    // Check abort before each filesystem op
    const throwIfAborted = (): void => {
      if (signal?.aborted) throw new Error("Operation aborted");
    };

    throwIfAborted();
    await ops.mkdir(dir);       // mkdir -p the parent directory
    throwIfAborted();
    await ops.writeFile(absolutePath, content);
    throwIfAborted();

    return {
      content: [{ type: "text", text: `Successfully wrote ${content.length} bytes to ${path}` }],
      details: undefined,
    };
  });
}
```

Two things to notice here. First, `ops.mkdir(dir)` uses `{ recursive: true }` under the hood, so the write tool automatically creates any missing parent directories — you never need a separate mkdir step. Second, the whole operation is wrapped in `withFileMutationQueue` (explained in the shared plumbing section below), which serialises concurrent writes to the same file.

On success the result text is `"Successfully wrote N bytes to <path>"`. On failure (the file system throws, or the signal is aborted), the error propagates and the agent loop records it as a tool error. The tool's `promptGuidelines` field tells the model: *"Use write only for new files or complete rewrites."*

---

## The edit tool — precise in-place edits

Write is fine for whole-file creation, but the agent often needs to change one function inside a large file. Rewriting the whole file is risky (the model might forget other parts) and expensive. That's what `edit` solves.

### The schema

```ts
// edit.ts
const replaceEditSchema = Type.Object({
  oldText: Type.String({ description:
    "Exact text for one targeted replacement. It must be unique in the original file ..."
  }),
  newText: Type.String({ description: "Replacement text for this targeted edit." }),
}, { additionalProperties: false });

const editSchema = Type.Object({
  path:  Type.String({ description: "Path to the file to edit (relative or absolute)" }),
  edits: Type.Array(replaceEditSchema, { description:
    "One or more targeted replacements. Each edit is matched against the original file, not incrementally. ..."
  }),
}, { additionalProperties: false });
```

The key design: `edits` is an **array**, so the model can target multiple disjoint regions of a file in one tool call. Each `edits[i].oldText` is matched against the *original* file, not against the state after previous edits.

### The execute logic

The execute path, simplified:

1. **Validate** — `edits` must contain at least one entry; throws `"edits must contain at least one replacement"` otherwise.
2. **Access check** — the file must exist and be readable and writable. On failure: `"Could not edit file: <path>. Error code: ENOENT."` (or `EACCES` for permission errors).
3. **Read** — the file is read as a Buffer, decoded as UTF-8.
4. **BOM stripping** — if the file starts with a UTF-8 BOM (`﻿`), it is stripped before matching. The BOM is re-added to the output after editing.
5. **Line-ending normalisation** — the file's dominant line ending (`\r\n` or `\n`) is detected and stored. The content is normalised to `\n` for matching, then restored to the original ending after edits are applied.
6. **Apply edits** — `applyEditsToNormalizedContent()` matches each `oldText` and replaces it with `newText`.
7. **Write back** — the edited content is written atomically via `withFileMutationQueue`.
8. **Diff generation** — `generateDiffString()` and `generateUnifiedPatch()` produce the human-readable diff and the standard unified patch, both returned in `details`.

On success: `"Successfully replaced N block(s) in <path>."` in content, plus `details.diff`, `details.patch`, and `details.firstChangedLine`.

### Fuzzy matching

The edit tool tries an **exact** match first. If that fails, it retries with a *fuzzy-normalised* version of both the content and `oldText`. Fuzzy normalisation:

- Strips trailing whitespace from each line
- Normalises smart quotes (`'`, `'`, `"`, `"`) to ASCII equivalents (`'`, `"`)
- Normalises Unicode dashes (en-dash, em-dash, etc.) to ASCII hyphen (`-`)
- Normalises special Unicode spaces (non-breaking space, etc.) to regular space (`' '`)
- Applies Unicode NFKC decomposition

This means an LLM that copies code into `oldText` and accidentally picks up smart quotes or trailing spaces will still match, rather than failing with a confusing "not found" error.

### Error contract for edit

| Condition | Error message |
|---|---|
| `edits` array is empty | `"edits must contain at least one replacement"` |
| File not found | `"Could not edit file: <path>. Error code: ENOENT."` |
| File not writable | `"Could not edit file: <path>. Error code: EACCES."` |
| `oldText` not found in file | `"Could not find the exact text in <path>. ..."` |
| `oldText` found more than once | `"Found N occurrences of the text in <path>. ..."` |
| Two edits target overlapping regions | `"edits[i] and edits[j] overlap in <path>. ..."` |
| All edits succeed but produce identical content | `"No changes made to <path>. The replacement produced identical content."` |

The tests confirm that when a multi-edit call has one failing entry, the file is **not** partially modified — all edits are validated before any writes happen:

```ts
// S88 — tools.test.ts
it("should not partially apply edits when one edit fails", async () => {
  const originalContent = "alpha\nbeta\ngamma\n";
  writeFileSync(testFile, originalContent);

  await expect(
    editTool.execute("call", {
      path: testFile,
      edits: [
        { oldText: "alpha\n", newText: "ALPHA\n" },   // valid
        { oldText: "missing\n", newText: "MISSING\n" }, // not found
      ],
    }),
  ).rejects.toThrow(/Could not find/);

  // File is unchanged
  expect(readFileSync(testFile, "utf-8")).toBe(originalContent);
});
```

### edit-diff.ts — the shared diff engine

The file `edit-diff.ts` contains the diff computation that both `edit.ts` (for execution) and the TUI's live preview (for rendering a diff before the model commits) share:

- `applyEditsToNormalizedContent(normalizedContent, edits, path)` — matches and applies edits, returning `{ baseContent, newContent }`.
- `generateDiffString(oldContent, newContent)` — returns a line-by-line human-readable diff string with line numbers, plus `firstChangedLine`.
- `generateUnifiedPatch(path, oldContent, newContent)` — generates a standard unified diff patch string (using the `diff` library).
- `computeEditsDiff(path, edits, cwd)` — reads the file and computes the diff *without* applying it, used by the TUI to preview changes.

---

## The bash tool — running a shell command

Sometimes the agent needs to do something no file tool can do: compile code, run tests, install a dependency. That's `bash`.

### The schema

```ts
// bash.ts
const bashSchema = Type.Object({
  command: Type.String({ description: "Bash command to execute" }),
  timeout: Type.Optional(Type.Number({ description: "Timeout in seconds (optional, no default timeout)" })),
});
```

`command` is the shell command string. `timeout` is optional — there is no default timeout; if not supplied the command runs until it exits or the `AbortSignal` fires.

### The execute logic

The bash tool execution has several moving parts. Let's walk through them.

**Command prefix.** If `BashToolOptions.commandPrefix` is set, it is prepended to every command with a newline: `${commandPrefix}\n${command}`. This lets an extension inject shell setup (e.g. `source ~/.nvm/nvm.sh`) before each command without the model knowing about it.

**Spawn hook.** `BashToolOptions.spawnHook` is a function `(context: BashSpawnContext) => BashSpawnContext` that can rewrite the command, working directory, or environment variables just before the process is spawned. Extensions use this to redirect commands to a remote host.

**Streaming output.** The child process's stdout and stderr are both piped to an `OutputAccumulator` (described below). As data arrives, `scheduleOutputUpdate()` throttles calls to `onUpdate` so the TUI is not flooded — updates are coalesced to at most one per `BASH_UPDATE_THROTTLE_MS` milliseconds (100 ms).

**Timeout and abort.** If `timeout` is provided, a `setTimeout` fires after `timeout * 1000` ms and kills the process tree. If the `AbortSignal` fires, `killProcessTree(child.pid)` is called. Both throw errors whose messages become the tool result.

**Exit code check.** If the process exits with a non-zero, non-null code, the tool throws an error: `"<output text>\n\nCommand exited with code N"`. Exit code 0 means success.

**Output format on truncation.** When output is truncated, the result text includes a footer such as:
```
[Showing lines 2001-4000 of 4000. Full output: /tmp/xzy-bash-abc123.log]
```
The full output is saved to a temp file, and the path appears in both the text (for the model) and `details.fullOutputPath` (for the TUI to offer a "view full output" action).

```ts
// S88 — tools.test.ts (bash)
it("should not count a trailing newline as an extra truncated bash output line", async () => {
  // 4000 lines → last 2000 shown (tail truncation)
  expect(result.details?.truncation?.totalLines).toBe(4000);
  expect(result.details?.truncation?.outputLines).toBe(2000);
  expect(output).toContain("line-2001");
  expect(output).toContain("line-4000");
  expect(output).toMatch(/\[Showing lines 2001-4000 of 4000\. Full output: /);
});
```

Notice the direction: bash uses **tail truncation** (keep the *last* N lines), while the read tool uses **head truncation** (keep the *first* N lines). For bash, the end of the output — where errors and final results appear — is what matters.

### BashOperations — pluggable execution backend

`BashToolOptions.operations` is an optional `BashOperations` object:

```ts
export interface BashOperations {
  exec: (
    command: string,
    cwd: string,
    options: {
      onData: (data: Buffer) => void;
      signal?: AbortSignal;
      timeout?: number;
      env?: NodeJS.ProcessEnv;
    },
  ) => Promise<{ exitCode: number | null }>;
}
```

The default implementation (`createLocalBashOperations()`) spawns a child process using the local shell. By supplying a custom `BashOperations`, an extension can redirect commands to an SSH remote, a container, or a fake execution backend in tests.

### Error contract for bash

| Condition | Error message |
|---|---|
| Working directory does not exist | `"Working directory does not exist: <cwd>\nCannot execute bash commands."` |
| Command exits non-zero | `"<output>\n\nCommand exited with code N"` |
| Timeout | `"<output>\n\nCommand timed out after N seconds"` |
| AbortSignal fires | `"<output>\n\nCommand aborted"` |

---

## The grep tool — searching file contents

Now that the agent can modify files, it needs to search them. `grep` finds lines matching a regex (or literal string) pattern, respects `.gitignore`, and returns results with file path and line number.

### The schema

```ts
// grep.ts
const grepSchema = Type.Object({
  pattern:     Type.String({ description: "Search pattern (regex or literal string)" }),
  path:        Type.Optional(Type.String({ description: "Directory or file to search (default: current directory)" })),
  glob:        Type.Optional(Type.String({ description: "Filter files by glob pattern, e.g. '*.ts' or '**/*.spec.ts'" })),
  ignoreCase:  Type.Optional(Type.Boolean({ description: "Case-insensitive search (default: false)" })),
  literal:     Type.Optional(Type.Boolean({ description: "Treat pattern as literal string instead of regex (default: false)" })),
  context:     Type.Optional(Type.Number({ description: "Number of lines to show before and after each match (default: 0)" })),
  limit:       Type.Optional(Type.Number({ description: "Maximum number of matches to return (default: 100)" })),
});
```

All fields except `pattern` are optional.

### The execute logic

Under the hood, `grep` spawns **ripgrep** (`rg`) with `--json --line-number --color=never --hidden`. The `--hidden` flag includes dotfiles; `.gitignore` is respected by ripgrep's default behaviour. The pattern is always passed after `--` to prevent injection of ripgrep flags (confirmed by the test `"should treat flag-like patterns as search text"`).

Options map to ripgrep flags:

| Option | ripgrep flag |
|---|---|
| `ignoreCase: true` | `--ignore-case` |
| `literal: true` | `--fixed-strings` |
| `glob: "*.ts"` | `--glob *.ts` |

Context lines (`context: N`) cause the grep tool to re-read the file and emit `N` lines before/after each match. Context lines use `-` as separator (`file:2: match line`), non-match context uses `-` instead of `:` (`file-1- before`).

Long match lines are truncated to `GREP_MAX_LINE_LENGTH` characters (500 chars) to keep output compact. The `details.linesTruncated` flag signals this to the TUI.

**Match limit.** The default is 100 matches (`DEFAULT_LIMIT`). When the limit is reached, ripgrep is killed and the result includes a notice: `"100 matches limit reached. Use limit=200 for more, or refine pattern"`.

**Output truncation.** Even after the match limit, the total output is passed through `truncateHead` with the byte limit (50 KB). If bytes are the bottleneck, `details.truncation` is set.

The result when no matches are found: `{ content: [{ type: "text", text: "No matches found" }], details: undefined }`.

---

## The find tool — locating files by glob pattern

`grep` searches file *contents*. `find` searches file *names* by glob pattern. It uses **fd** (`fd-find`) internally and also respects `.gitignore`.

### The schema

```ts
// find.ts
const findSchema = Type.Object({
  pattern: Type.String({ description: "Glob pattern to match files, e.g. '*.ts', '**/*.json', or 'src/**/*.spec.ts'" }),
  path:    Type.Optional(Type.String({ description: "Directory to search in (default: current directory)" })),
  limit:   Type.Optional(Type.Number({ description: "Maximum number of results (default: 1000)" })),
});
```

### The execute logic

The default implementation spawns `fd` with:

```
fd --glob --color=never --hidden --no-require-git --max-results <limit> -- <pattern> <searchPath>
```

`--no-require-git` makes fd apply hierarchical `.gitignore` semantics whether or not the directory is a git repository. `--hidden` includes dotfiles.

**Path-containing patterns.** When `pattern` contains `/`, `--full-path` is added and the pattern is prefixed with `**/` if it doesn't already start with `/` or `**/`. This makes `src/**/*.spec.ts` match correctly in full-path mode.

Results are returned as paths relative to the search root, using POSIX separators. If the result count hits `limit`, a notice is appended: `"1000 results limit reached. Use limit=2000 for more, or refine pattern"`.

The tool also applies byte truncation (`truncateHead` with `maxLines: Number.MAX_SAFE_INTEGER`) to cap the total output size.

**Custom `FindOperations`.** Extensions can supply `operations.glob(pattern, cwd, { ignore, limit })` to replace the `fd` invocation — useful when searching a remote filesystem.

---

## The ls tool — listing a directory

`find` works for glob searches. But the agent often needs a flat list of what's in a directory — particularly when exploring an unfamiliar project. That's `ls`.

### The schema

```ts
// ls.ts
const lsSchema = Type.Object({
  path:  Type.Optional(Type.String({ description: "Directory to list (default: current directory)" })),
  limit: Type.Optional(Type.Number({ description: "Maximum number of entries to return (default: 500)" })),
});
```

### The execute logic

The execute path:

1. Resolves the path with `resolveToCwd`.
2. Checks existence with `ops.exists`. If the path does not exist, throws `"Path not found: <path>"`.
3. Checks `ops.stat` — if not a directory, throws `"Not a directory: <path>"`.
4. Reads directory entries with `ops.readdir`, sorts them **alphabetically, case-insensitive**.
5. For each entry, `ops.stat` is called to determine if it is a directory, appending `/` to directory names. Entries that cannot be stated are skipped.
6. Applies the entry limit (default 500). If the limit is hit: `"500 entries limit reached. Use limit=1000 for more"`.
7. Applies byte truncation (`truncateHead` with `maxLines: Number.MAX_SAFE_INTEGER`).

`ls` includes dotfiles (files and directories beginning with `.`). The test confirms:

```ts
// S88 — tools.test.ts
it("should list dotfiles and directories", async () => {
  // creates .hidden-file and .hidden-dir/
  expect(output).toContain(".hidden-file");
  expect(output).toContain(".hidden-dir/");
});
```

On an empty directory, the result is `"(empty directory)"`.

---

## Quick reference: all seven built-in tools

| Tool | Name | Key schema fields | Default limits | Engine |
|---|---|---|---|---|
| Read | `read` | `path`, `offset?`, `limit?` | 2000 lines / 50 KB (head) | Node.js fs |
| Write | `write` | `path`, `content` | — | Node.js fs |
| Edit | `edit` | `path`, `edits[]` | — | Node.js fs |
| Bash | `bash` | `command`, `timeout?` | 2000 lines / 50 KB (tail) | Shell child process |
| Grep | `grep` | `pattern`, `path?`, `glob?`, … | 100 matches / 50 KB | ripgrep (`rg`) |
| Find | `find` | `pattern`, `path?`, `limit?` | 1000 results / 50 KB | fd (`fd-find`) |
| Ls | `ls` | `path?`, `limit?` | 500 entries / 50 KB | Node.js fs |

The canonical set names are:

```ts
// index.ts
export type ToolName = "read" | "bash" | "edit" | "write" | "grep" | "find" | "ls";
export const allToolNames: Set<ToolName> = new Set(["read", "bash", "edit", "write", "grep", "find", "ls"]);
```

There are also convenience groupings: `createCodingToolDefinitions` returns `[read, bash, edit, write]` (the mutation set), and `createReadOnlyToolDefinitions` returns `[read, grep, find, ls]`.

---

## Shared plumbing

All seven tools share three utility modules that handle cross-cutting concerns. Let's look at each.

### truncate.ts — capping tool output

Every tool that produces potentially large output passes it through one of two truncation functions from `truncate.ts`. They share the same two limits:

```ts
export const DEFAULT_MAX_LINES = 2000;
export const DEFAULT_MAX_BYTES = 50 * 1024; // 50 KB
```

Whichever limit is hit first applies. Neither function ever returns a partial line (with one narrow exception noted below).

**`truncateHead(content, options?)`** — keeps the *first* N lines/bytes. Used by: read, grep, find, ls. Rationale: for file reads, you want the beginning; the model can request more with `offset`.

**`truncateTail(content, options?)`** — keeps the *last* N lines/bytes. Used by: bash. Rationale: for shell output, the error or final result is at the end.

Both return a `TruncationResult`:

```ts
export interface TruncationResult {
  content: string;        // the truncated text
  truncated: boolean;
  truncatedBy: "lines" | "bytes" | null;
  totalLines: number;     // original line count
  totalBytes: number;     // original byte count
  outputLines: number;    // lines in truncated output
  outputBytes: number;    // bytes in truncated output
  lastLinePartial: boolean;       // tail edge case only
  firstLineExceedsLimit: boolean; // head: first line alone > maxBytes
  maxLines: number;
  maxBytes: number;
}
```

When `firstLineExceedsLimit` is true (read tool only), the output text is:
```
[Line N is X, exceeds 50.0KB limit. Use bash: sed -n 'Np' <path> | head -c 51200]
```
This directs the model to an alternative approach rather than leaving it stuck.

`truncateLine(line, maxChars?)` is a third helper used only by the grep tool to cap individual match lines at `GREP_MAX_LINE_LENGTH` (500 characters), appending `... [truncated]` when the line is cut.

### output-accumulator.ts — streaming output with bounded memory

The bash tool produces output as a stream of `Buffer` chunks that may arrive faster than the model or TUI can consume. `OutputAccumulator` solves three problems at once:

1. **Bounded memory**: it keeps only a rolling tail of `maxBytes * 2` decoded bytes in memory. Older content is discarded from the in-memory buffer.
2. **UTF-8 safety**: it uses a streaming `TextDecoder` so multi-byte characters split across chunk boundaries are decoded correctly.
3. **Temp file spill**: when total output exceeds the display limits, a temp file is opened and all raw chunks are written there. The path is returned in `OutputSnapshot.fullOutputPath`.

The API:

```ts
class OutputAccumulator {
  constructor(options?: { maxLines?, maxBytes?, tempFilePrefix? });

  append(data: Buffer): void;   // call from the onData handler
  finish(): void;                // call when the process exits
  snapshot(options?: { persistIfTruncated?: boolean }): OutputSnapshot;
  async closeTempFile(): Promise<void>;
  getLastLineBytes(): number;    // size of the current incomplete line
}
```

`snapshot()` calls `truncateTail` on the in-memory tail to produce the display content. If `persistIfTruncated` is `true`, the temp file is created even if it hadn't been spilled yet — this ensures the full output survives even for timeout/abort errors.

The bash tool calls `append` inside the `onData` handler, `finish` after the process exits, then `snapshot({ persistIfTruncated: true })` to get the final content.

### file-mutation-queue.ts — serialising concurrent writes

Both `write` and `edit` call `withFileMutationQueue(absolutePath, fn)`. Here's the problem it solves: the agent loop may invoke two tool calls concurrently (e.g. write one file while editing another). If both target the same file, the second write must not start before the first finishes.

```ts
// file-mutation-queue.ts (simplified view)
const fileMutationQueues = new Map<string, Promise<void>>();

export async function withFileMutationQueue<T>(
  filePath: string,
  fn: () => Promise<T>,
): Promise<T> {
  // Resolve symlinks to get the canonical path as the queue key.
  const key = await getMutationQueueKey(filePath);

  // Chain onto the existing queue for this file (or an empty Promise if none).
  const currentQueue = fileMutationQueues.get(key) ?? Promise.resolve();
  // ... build the chained promise, set it as the new queue tail ...
  await currentQueue;  // wait for any pending mutation to finish
  try {
    return await fn(); // run our mutation
  } finally {
    releaseNext();     // allow the next waiter to proceed
  }
}
```

Key properties:

- Operations on **different files** run in parallel — only same-file operations are serialised.
- The queue key is the **real path** (symlinks resolved via `realpath`), so two paths that resolve to the same file are correctly serialised.
- If the file does not exist yet (e.g. a new write), `resolve()` falls back to the normalised path.
- The queue entry is cleaned up after the last waiter exits, so there is no memory accumulation for files that are no longer in use.

### path-utils.ts — path resolution and macOS quirks

Every tool that takes a `path` argument routes it through `resolveToCwd(filePath, cwd)` before any filesystem operation. This function calls through to the project's `resolvePath` utility with `{ normalizeUnicodeSpaces: true, stripAtPrefix: true }`. The `stripAtPrefix` option strips a leading `@` that some model outputs accidentally prepend. `normalizeUnicodeSpaces` normalises Unicode space characters in paths.

The read tool uses a richer variant, `resolveReadPathAsync(filePath, cwd)`, which applies the same resolution and then tries four macOS-specific fallbacks in order:

1. **AM/PM variant** — replaces the space before `AM`/`PM` in screenshot names with a narrow no-break space (` `), matching the way macOS stores these filenames.
2. **NFD variant** — converts the path to NFD (decomposed Unicode), matching macOS's internal filename encoding.
3. **Curly quote variant** — replaces ASCII apostrophes with the macOS right single quotation mark (`’`), matching French screenshot names like `"Capture d'écran"`.
4. **NFD + curly quote** — the two variants combined.

If any variant resolves to a readable file, that path is used. Otherwise the original resolved path is returned (and the subsequent `access` call will throw).

---

## Putting it together: instantiating the full tool set

The `index.ts` of the tools module exports factory functions for every combination you might need:

```ts
import { createAllTools, createCodingTools, createReadOnlyTools } from "coding-agent";

// All seven tools for a full session
const allTools = createAllTools(process.cwd());

// Only the mutation tools (read + bash + edit + write)
const codingTools = createCodingTools(process.cwd());

// Only the read-only tools (read + grep + find + ls) — safe for sandboxed contexts
const readOnlyTools = createReadOnlyTools(process.cwd());
```

Each factory accepts an optional `ToolsOptions` object so you can customise individual tools:

```ts
const tools = createAllTools(process.cwd(), {
  bash: {
    commandPrefix: "source ~/.nvm/nvm.sh && nvm use 20",
    shellPath: "/usr/local/bin/bash",
  },
  read: {
    autoResizeImages: false,
  },
});
```

The `AgentSession` (see [AgentSession: The Core of the Coding Agent](./agent-session-core.md)) receives these tools at construction time and registers them into its internal `ToolDefinition` map, making them available to the model and to the system-prompt builder.

---

← Previous: [AgentSession: The Core of the Coding Agent](./agent-session-core.md) · Next: [System Prompt Construction and Skill Loading](./system-prompt-and-skills.md) →
