---
title: "The CLI Entry Point: Argument Parsing and Mode Selection"
description: "How the xzy binary bootstraps, parses every CLI flag, selects a mode (interactive/print/rpc), and hands off to the right subsystems."
category: coding-agent
type: tutorial
tags: [CLI, arg parsing, mode selection, xzy CLI, bin entry, args, help, flags, coding-agent, entry point, main, wiring, parseArgs, Args, Mode, printHelp, bootstrap]
keywords: [argument parser, CLI flags reference, mode switching, process.argv, binary entry point, xzy flags, xzy options, CLI reference]
sources: [S70, S71]
---

**TL;DR** — When you run `xzy`, a three-line bootstrap file handles the first milliseconds: it sets the process title, configures the HTTP layer, then hands raw `process.argv` to `main()`. Inside `main()` the first thing that happens is argument parsing — every flag the user typed is turned into a typed `Args` object by `parseArgs()`. This chapter walks the entry point from first byte to mode handoff, then provides a comprehensive reference table of every CLI flag.

# The CLI Entry Point: Argument Parsing and Mode Selection

## The problem: something has to go first

You type `xzy "List all .ts files"` and hit Return. The shell finds the `xzy` binary and Node.js starts. But before any agent logic can run — before a session is created, before any LLM is contacted, before a TUI is drawn — *something* has to:

1. Figure out which flags you passed.
2. Decide which mode to launch (interactive shell, non-interactive print, or JSON-RPC daemon).
3. Wire up the right managers and hand off to the chosen mode.

That "something" is a short chain of two files: `src/cli.ts` (the actual binary entry point) and `src/cli/args.ts` (the argument parser). Let's trace them in order.

## Step 1 — The binary bootstrap (`cli.ts`)

The file `src/cli.ts` is the file Node.js executes when the `xzy` binary runs. It is deliberately thin: its job is to prepare the process environment and immediately delegate to `main()`.

```ts
// src/cli.ts — the binary entry point (shown in full)
#!/usr/bin/env node

import { APP_NAME } from "./config.ts";
import { configureHttpDispatcher } from "./core/http-dispatcher.ts";
import { main } from "./main.ts";

process.title = APP_NAME;
process.env.XZY_CODING_AGENT = "true";
process.emitWarning = (() => {}) as typeof process.emitWarning;

// Configure undici's global dispatcher before provider SDKs issue requests.
// Runtime settings are applied once SettingsManager has loaded global/project settings.
configureHttpDispatcher();

main(process.argv.slice(2));
```

Let's look at each line and understand why it exists.

**`process.title = APP_NAME`** — this sets the process name visible in tools like `ps` or `top`. Without it the process would show up as `node` which is unhelpful when you are trying to identify the running agent.

**`process.env.XZY_CODING_AGENT = "true"`** — marks the Node.js process as the coding agent. Any code deeper in the call stack can check this environment variable to know it is running inside the CLI (rather than being used as a library via the SDK).

**`process.emitWarning = (() => {}) as typeof process.emitWarning`** — silences Node.js deprecation and experimental warnings. These would otherwise clutter the terminal with noise unrelated to the agent's own output.

**`configureHttpDispatcher()`** — sets up the global HTTP dispatcher (using `undici`) *before* any provider SDK has a chance to issue network requests. Provider SDKs often configure HTTP at import time; by running this setup first, the agent controls connection settings such as timeouts and proxy handling before those SDKs get a chance to.

**`main(process.argv.slice(2))`** — `process.argv` is the Node.js array of all command-line tokens; index 0 is `node`, index 1 is the script path, so `.slice(2)` strips those and gives us only the tokens the user actually typed. `main()` receives a plain `string[]` — it does not touch `process.argv` directly.

Notice what `cli.ts` does *not* do: it does not parse arguments, it does not create an `AgentSession`, and it does not pick a mode. All of that lives inside `main()`. The entry point stays tiny so that the binary can be shimmed or replaced without touching the real logic.

## Step 2 — Turning raw tokens into a typed `Args` object

<!-- GAP: src/main.ts is not an assigned source, so the mode-selection branch and AgentSession wiring inside main() cannot be traced here. The following covers argument parsing from src/cli/args.ts (S71) as the first thing main() does. -->

The first thing `main()` does with the token array is call `parseArgs()` from `src/cli/args.ts`. This function converts the flat `string[]` into a strongly-typed `Args` object that every later module can read without touching raw strings again.

### What `Args` looks like

`Args` is an interface that captures every flag the CLI understands:

```ts
// src/cli/args.ts — the Args interface (simplified view showing all fields)
export type Mode = "text" | "json" | "rpc";

export interface Args {
  // Model / provider selection
  provider?: string;
  model?: string;
  apiKey?: string;
  thinking?: ThinkingLevel;
  models?: string[];

  // System prompt
  systemPrompt?: string;
  appendSystemPrompt?: string[];

  // Mode
  mode?: Mode;
  print?: boolean;
  export?: string;

  // Session management
  continue?: boolean;
  resume?: boolean;
  name?: string;
  noSession?: boolean;
  session?: string;
  sessionId?: string;
  fork?: string;
  sessionDir?: string;

  // Tool control
  tools?: string[];
  excludeTools?: string[];
  noTools?: boolean;
  noBuiltinTools?: boolean;

  // Extension / skill / resource control
  extensions?: string[];
  noExtensions?: boolean;
  skills?: string[];
  noSkills?: boolean;
  promptTemplates?: string[];
  noPromptTemplates?: boolean;
  themes?: string[];
  noThemes?: boolean;
  noContextFiles?: boolean;

  // Misc
  help?: boolean;
  version?: boolean;
  listModels?: string | true;
  offline?: boolean;
  verbose?: boolean;

  // Positional inputs
  messages: string[];     // bare words the user typed
  fileArgs: string[];     // @filename arguments (@ prefix stripped)

  // Extension hooks
  unknownFlags: Map<string, boolean | string>;
  diagnostics: Array<{ type: "warning" | "error"; message: string }>;
}
```

Three fields deserve special attention:

- **`messages`** — bare words that are not flags (e.g. `"List all .ts files"`) accumulate here. The mode that runs will use these as the initial prompt.
- **`fileArgs`** — arguments that start with `@` (e.g. `@prompt.md`) accumulate here with the `@` prefix stripped. The caller reads the referenced files and attaches their contents to the message.
- **`unknownFlags`** — flags that `parseArgs()` does not recognise are stored here rather than rejected. This is how extensions can register their own flags (e.g. `--plan` from a plan-mode extension) without modifying the core parser.
- **`diagnostics`** — parse errors and warnings (e.g. `"--name requires a value"`) are collected here as structured objects rather than thrown immediately. The caller can display them cleanly.

### How `parseArgs()` works

`parseArgs()` is a hand-written sequential scanner — no third-party parser library. It walks the `args` array from left to right with an index variable `i`, consuming one or two tokens per iteration:

```ts
// src/cli/args.ts — the parseArgs loop skeleton (simplified)
export function parseArgs(args: string[]): Args {
  const result: Args = {
    messages: [],
    fileArgs: [],
    unknownFlags: new Map(),
    diagnostics: [],
  };

  for (let i = 0; i < args.length; i++) {
    const arg = args[i];

    if (arg === "--help" || arg === "-h") {
      result.help = true;

    } else if (arg === "--mode" && i + 1 < args.length) {
      const mode = args[++i];          // consume the next token
      if (mode === "text" || mode === "json" || mode === "rpc") {
        result.mode = mode;
      }
      // invalid mode values are silently ignored

    } else if (arg === "--models" && i + 1 < args.length) {
      result.models = args[++i].split(",").map((s) => s.trim());

    } else if (arg.startsWith("@")) {
      result.fileArgs.push(arg.slice(1));   // strip the @ prefix

    } else if (arg.startsWith("--")) {
      // unknown flag — store for extensions
      const eqIndex = arg.indexOf("=");
      if (eqIndex !== -1) {
        result.unknownFlags.set(arg.slice(2, eqIndex), arg.slice(eqIndex + 1));
      } else {
        // peek at the next token to decide if it is the flag's value
        const flagName = arg.slice(2);
        const next = args[i + 1];
        if (next !== undefined && !next.startsWith("-") && !next.startsWith("@")) {
          result.unknownFlags.set(flagName, next);
          i++;
        } else {
          result.unknownFlags.set(flagName, true);
        }
      }

    } else if (!arg.startsWith("-")) {
      result.messages.push(arg);   // bare word → initial message
    }
    // ...other cases elided for brevity
  }

  return result;
}
```

A few design choices are worth noticing:

- **`++i` inside the condition** — when a flag expects a value (`--model <id>`), the parser increments `i` in place so the next iteration skips the consumed value. This is the standard two-token idiom for hand-rolled parsers.
- **`--mode` validates its value** — only `"text"`, `"json"`, and `"rpc"` are accepted. An unrecognised mode string is silently ignored (the `mode` field stays `undefined`).
- **`--thinking` validates and diagnostics** — the valid thinking levels are `off | minimal | low | medium | high | xhigh`. An invalid level pushes a `"warning"` diagnostic rather than crashing.
- **`--print` is special** — after recording `print: true`, the parser peeks at the *next* token. If it does not start with `@` or `-` (or starts with `---`), it is consumed as the initial message. This lets `xzy -p "my prompt"` work naturally.
- **Unknown short flags** (e.g. `-x`) push an `"error"` diagnostic; unknown long flags (`--anything`) go into `unknownFlags` for extensions to handle.

### Mode: the three values of `Mode`

After parsing, the `mode` and `print` fields together tell `main()` which subsystem to launch:

| Resulting mode | When |
|---|---|
| **Interactive** (default) | `mode` is `undefined` and `print` is `false`/`undefined` |
| **Print / text** | `--print` / `-p` flag, or `--mode text` |
| **JSON print** | `--mode json` |
| **RPC** | `--mode rpc` |

Interactive mode is covered in detail in [Interactive Mode Startup and Wiring](./interactive-mode-startup-and-wiring.md). Print mode and RPC mode are covered in [Print Mode and RPC Mode](./print-and-rpc-modes.md).

## Step 3 — From `Args` to mode handoff

<!-- GAP: src/main.ts (the function that reads the Args object, builds AgentSession + resources, and calls the selected mode) is not in the assigned sources (S70, S71). The mode-selection branch and AgentSession wiring cannot be grounded here. -->

Once `parseArgs()` returns, `main()` reads the resulting `Args` object and does the wiring: it constructs the session managers and resources, then calls one of the three mode launchers. That logic lives in `src/main.ts`, which is outside the scope of this chapter.

What we do know from S71 is the shape of the information that flows into `main()`:

- **`args.help === true`** → print help text via `printHelp()` and exit.
- **`args.version === true`** → print the version string and exit.
- **`args.listModels`** → list available models (with optional fuzzy search pattern) and exit.
- **`args.mode === "rpc"`** → launch RPC mode (see [Print Mode and RPC Mode](./print-and-rpc-modes.md)).
- **`args.print === true` or `args.mode === "text"/"json"`** → launch print mode.
- Otherwise → launch interactive mode (see [Interactive Mode Startup and Wiring](./interactive-mode-startup-and-wiring.md)).

### The help text pipeline

`printHelp()` in `src/cli/args.ts` generates the full usage text using `chalk` for bold formatting. It accepts an optional `extensionFlags` array so that any extension that registered custom flags has them printed in a separate "Extension CLI Flags" section below the built-in options.

```ts
// src/cli/args.ts — printHelp signature
export function printHelp(extensionFlags?: ExtensionFlag[]): void { ... }
```

If `extensionFlags` is non-empty, the help output gains a block like:

```
Extension CLI Flags:
  --plan <value>                Registered by /path/to/plan-mode.ts
```

The caller is responsible for collecting `extensionFlags` from the loaded extensions and passing them to `printHelp()`.

---

## CLI Flags Reference

The rest of this chapter is a reference lookup: every flag `xzy` accepts, grouped by concern, with its type, default, and description drawn directly from `src/cli/args.ts`.

### Model and provider options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--provider <name>` | — | string | `google` | Provider name (e.g. `openai`, `anthropic`, `google`). |
| `--model <pattern>` | — | string | provider default | Model pattern or ID. Supports `provider/id` syntax and optional `:<thinking>` suffix (e.g. `sonnet:high`). |
| `--api-key <key>` | — | string | from env var | API key. Falls back to the provider-specific environment variable if omitted. |
| `--thinking <level>` | — | enum | — | Extended thinking level. Valid values: `off`, `minimal`, `low`, `medium`, `high`, `xhigh`. |
| `--models <patterns>` | — | string (CSV) | — | Comma-separated model patterns for in-session model cycling (Ctrl+P). Supports globs (`anthropic/*`, `*sonnet*`) and fuzzy matching. Also supports `model:thinking` shorthand. |
| `--list-models [search]` | — | string \| flag | — | List available models and exit. If a search pattern is given, filters the list. |

### System prompt options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--system-prompt <text>` | — | string | built-in coding assistant prompt | Replace the entire system prompt. |
| `--append-system-prompt <text>` | — | string (repeatable) | — | Append text (or file contents if the value is a path) to the system prompt. Can be used multiple times. |

### Mode and output options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--mode <mode>` | — | `text \| json \| rpc` | `text` | Output mode for non-interactive use. |
| `--print` | `-p` | flag | — | Non-interactive: process the prompt and exit. Optionally takes the prompt as the next argument. |
| `--export <file>` | — | string | — | Export a session file to HTML and exit. |

### Session management options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--continue` | `-c` | flag | — | Continue the most recent session. |
| `--resume` | `-r` | flag | — | Show a session picker to resume an existing session. |
| `--session <path\|id>` | — | string | — | Use a specific session file path or partial UUID. |
| `--session-id <id>` | — | string | — | Use an exact project session ID, creating it if it does not exist. |
| `--fork <path\|id>` | — | string | — | Fork an existing session file or partial UUID into a new session. |
| `--session-dir <dir>` | — | string | from `XZY_SESSION_DIR` | Directory for session storage and lookup. |
| `--no-session` | — | flag | — | Ephemeral run — do not persist any session file. |
| `--name <name>` | `-n` | string | — | Set a display name for this session. |

### Tool control options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--tools <tools>` | `-t` | string (CSV) | — | Allowlist of tool names to enable. Applies to built-in, extension, and custom tools. |
| `--exclude-tools <tools>` | `-xt` | string (CSV) | — | Denylist of tool names to disable. Applies to built-in, extension, and custom tools. |
| `--no-tools` | `-nt` | flag | — | Disable all tools (built-in and extension). |
| `--no-builtin-tools` | `-nbt` | flag | — | Disable built-in tools, but keep extension/custom tools enabled. |

### Extension and resource options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--extension <path>` | `-e` | string (repeatable) | — | Load an extension file. Can be used multiple times. |
| `--no-extensions` | `-ne` | flag | — | Disable automatic extension discovery. Explicit `-e` paths still load. |
| `--skill <path>` | — | string (repeatable) | — | Load a skill file or directory. Can be used multiple times. |
| `--no-skills` | `-ns` | flag | — | Disable automatic skill discovery and loading. |
| `--prompt-template <path>` | — | string (repeatable) | — | Load a prompt template file or directory. Can be used multiple times. |
| `--no-prompt-templates` | `-np` | flag | — | Disable automatic prompt template discovery and loading. |
| `--theme <path>` | — | string (repeatable) | — | Load a theme file or directory. Can be used multiple times. |
| `--no-themes` | — | flag | — | Disable automatic theme discovery and loading. |
| `--no-context-files` | `-nc` | flag | — | Disable `AGENTS.md` and `CLAUDE.md` discovery and loading. |

### Diagnostic and startup options

| Flag | Short | Type | Default | Description |
|---|---|---|---|---|
| `--verbose` | — | flag | — | Force verbose startup output (overrides the `quietStartup` setting). |
| `--offline` | — | flag | — | Disable startup network operations. Equivalent to `XZY_OFFLINE=1`. |
| `--help` | `-h` | flag | — | Show the help text and exit. |
| `--version` | `-v` | flag | — | Show the version number and exit. |

### Built-in tool names

These are the names you use with `--tools` and `--exclude-tools`:

| Name | Description | On by default |
|---|---|---|
| `read` | Read file contents | Yes |
| `bash` | Execute bash commands | Yes |
| `edit` | Edit files with find/replace | Yes |
| `write` | Write files (creates or overwrites) | Yes |
| `grep` | Search file contents | No |
| `find` | Find files by glob pattern | No |
| `ls` | List directory contents | No |

`grep`, `find`, and `ls` are read-only tools — they can examine the filesystem but cannot change it. They are off by default to keep the default tool surface minimal.

### Environment variables

These environment variables are read at startup and may replace or supplement CLI flags:

| Variable | Description |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic Claude API key |
| `ANTHROPIC_OAUTH_TOKEN` | Anthropic OAuth token (alternative to API key) |
| `ANT_LING_API_KEY` | Ant Ling API key |
| `OPENAI_API_KEY` | OpenAI GPT API key |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI API key |
| `AZURE_OPENAI_BASE_URL` | Azure OpenAI/Cognitive Services base URL |
| `AZURE_OPENAI_RESOURCE_NAME` | Azure OpenAI resource name (alternative to base URL) |
| `AZURE_OPENAI_API_VERSION` | Azure OpenAI API version (default: `v1`) |
| `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` | Azure OpenAI model=deployment map (comma-separated) |
| `DEEPSEEK_API_KEY` | DeepSeek API key |
| `NVIDIA_API_KEY` | NVIDIA NIM API key |
| `GEMINI_API_KEY` | Google Gemini API key |
| `GROQ_API_KEY` | Groq API key |
| `CEREBRAS_API_KEY` | Cerebras API key |
| `XAI_API_KEY` | xAI Grok API key |
| `FIREWORKS_API_KEY` | Fireworks API key |
| `TOGETHER_API_KEY` | Together AI API key |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `AI_GATEWAY_API_KEY` | Vercel AI Gateway API key |
| `ZAI_API_KEY` | ZAI API key |
| `ZAI_CODING_CN_API_KEY` | ZAI Coding Plan API key (China region) |
| `MISTRAL_API_KEY` | Mistral API key |
| `MINIMAX_API_KEY` | MiniMax API key |
| `MOONSHOT_API_KEY` | Moonshot AI API key |
| `OPENCODE_API_KEY` | OpenCode Zen/OpenCode Go API key |
| `KIMI_API_KEY` | Kimi For Coding API key |
| `CLOUDFLARE_API_KEY` | Cloudflare API token (Workers AI and AI Gateway) |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare account ID (required for both Workers AI and AI Gateway) |
| `CLOUDFLARE_GATEWAY_ID` | Cloudflare AI Gateway slug (required for AI Gateway) |
| `XIAOMI_API_KEY` | Xiaomi MiMo API key (`api.xiaomimimo.com` billing) |
| `XIAOMI_TOKEN_PLAN_CN_API_KEY` | Xiaomi MiMo Token Plan API key (China region) |
| `XIAOMI_TOKEN_PLAN_AMS_API_KEY` | Xiaomi MiMo Token Plan API key (Amsterdam region) |
| `XIAOMI_TOKEN_PLAN_SGP_API_KEY` | Xiaomi MiMo Token Plan API key (Singapore region) |
| `AWS_PROFILE` | AWS profile for Amazon Bedrock |
| `AWS_ACCESS_KEY_ID` | AWS access key for Amazon Bedrock |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key for Amazon Bedrock |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock API key (bearer token) |
| `AWS_REGION` | AWS region for Amazon Bedrock (e.g. `us-east-1`) |
| `XZY_AGENT_DIR` | Config directory (default: `~/.xzy/agent`) |
| `XZY_SESSION_DIR` | Session storage directory (overridden by `--session-dir`) |
| `XZY_PACKAGE_DIR` | Override package directory (for Nix/Guix store paths) |
| `XZY_OFFLINE` | Disable startup network operations when set to `1`, `true`, or `yes` |
| `XZY_TELEMETRY` | Override install telemetry when set to `1/true/yes` or `0/false/no` |

### Positional arguments and `@file` references

```bash
# Bare words become messages (the initial prompt)
xzy "List all .ts files in src/"

# Multiple bare words become multiple messages (submitted in sequence)
xzy "Read package.json" "What dependencies do we have?"

# @filename arguments attach file contents to the initial message
xzy @prompt.md @image.png "What colour is the sky?"
```

The `@` prefix can reference any file path. The contents are read and attached to the initial message before it is sent to the model.

### Extension flags

Extensions can register additional flags by declaring an `ExtensionFlag` object. The flag appears in `--help` output under "Extension CLI Flags" and its value arrives via `args.unknownFlags` at the extension's handler. For example, a plan-mode extension might register `--plan`, making `xzy --plan my-plan.md` valid.

## Putting it together: a few common invocations

```bash
# Interactive mode — the default when no --print or --mode is given
xzy

# Interactive with an opening prompt
xzy "Show me all TODO comments in src/"

# Non-interactive (print) mode: process and exit
xzy -p "Summarise the changes in git diff --staged"

# Non-interactive with JSON output for scripting
xzy --mode json -p "List all exported functions in src/api.ts"

# Continue the previous session in interactive mode
xzy --continue

# Fork an existing session and give it a new name
xzy --fork abc123 --name "Refactor: auth module"

# Use a specific model with extended thinking
xzy --model sonnet:high "Walk me through this algorithm"

# Restrict tools to read-only operations
xzy --tools read,grep,find,ls -p "Review the security of src/auth.ts"

# Disable all built-in tools but keep extension tools active
xzy --no-builtin-tools

# Run offline (no startup network calls)
xzy --offline
```

These examples draw directly from the help text in `src/cli/args.ts`. The `--tools read,grep,find,ls` pattern is a practical read-only mode: the model can inspect the codebase but cannot modify or execute anything.

---

← Previous: [Print Mode and RPC Mode](./print-and-rpc-modes.md) · Next: [The SDK: createAgentSession and Programmatic Use](./sdk-and-programmatic-use.md) →
