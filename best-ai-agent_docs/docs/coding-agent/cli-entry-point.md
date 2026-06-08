---
title: "The CLI Entry Point: Wiring Everything Together"
description: "Build the CLI entry point — argument parsing, config loading, model selection, mode dispatch, and the final wiring that assembles all four layers into the binary users run."
category: coding-agent
type: tutorial
tags: [CLI, argument parsing, mode selection, bin entry, args, help, flags, main, wiring, coding-agent, entry point, startup]
keywords: [CLI, argument parsing, mode dispatch, binary entry, flag reference]
sources: [S44, S54]
---

**TL;DR** — All four layers are built. The final step: wire them into a single binary users can run. We'll parse CLI arguments, load configuration, select the model, dispatch to the correct mode (interactive, print, or RPC), and handle startup errors gracefully. This chapter ties the entire build together.

## The CLI entry point

Create `packages/coding-agent/src/cli.ts`. This is the file that runs when a user types `coding-agent`:

```ts
#!/usr/bin/env node
import { parseArgs } from "./cli/args.ts";
import { loadConfig } from "./config.ts";
import { runInteractive } from "./modes/interactive/app.ts";
import { runPrintMode } from "./modes/print/print-mode.ts";
import { runRpcMode } from "./modes/rpc/rpc-mode.ts";

async function main(): Promise<void> {
  const args = parseArgs(process.argv.slice(2));

  if (args.help) {
    printHelp();
    process.exit(0);
  }

  if (args.version) {
    console.log(`coding-agent v${VERSION}`);
    process.exit(0);
  }

  try {
    // Load configuration
    const config = await loadConfig({
      cwd: args.cwd ?? process.cwd(),
      configDir: args.configDir ?? ".coding-agent",
    });

    // Resolve model
    const modelId = args.model ?? config.settings.model ?? "claude-sonnet-4-6";

    // Dispatch to mode
    if (args.rpc) {
      await runRpcMode({ cwd: config.cwd, modelId });
    } else if (args.prompt !== undefined) {
      await runPrintMode({
        cwd: config.cwd,
        modelId,
        prompt: args.prompt || await readStdin(),
        sessionId: args.session,
      });
    } else {
      await runInteractive({
        cwd: config.cwd,
        modelId,
        sessionId: args.session,
        theme: config.settings.theme,
        skillPaths: args.skillPaths ?? config.settings.skillPaths,
      });
    }
  } catch (err) {
    console.error("Error:", err.message);
    if (args.debug) console.error(err.stack);
    process.exit(1);
  }
}

main();
```

## Argument parsing

Create `packages/coding-agent/src/cli/args.ts`:

```ts
interface CliArgs {
  help: boolean;
  version: boolean;
  debug: boolean;
  prompt?: string;
  rpc: boolean;
  model?: string;
  session?: string;
  cwd?: string;
  configDir?: string;
  skillPaths?: string[];
}

function parseArgs(raw: string[]): CliArgs {
  const args: CliArgs = {
    help: false,
    version: false,
    debug: false,
    rpc: false,
  };

  let i = 0;
  while (i < raw.length) {
    const arg = raw[i];
    switch (arg) {
      case "-h": case "--help":
        args.help = true; break;
      case "-v": case "--version":
        args.version = true; break;
      case "-d": case "--debug":
        args.debug = true; break;
      case "-p": case "--prompt":
        args.prompt = raw[++i] ?? ""; break;
      case "-m": case "--model":
        args.model = raw[++i]; break;
      case "-s": case "--session":
        args.session = raw[++i]; break;
      case "--rpc":
        args.rpc = true; break;
      case "-C": case "--cwd":
        args.cwd = raw[++i]; break;
      case "--config-dir":
        args.configDir = raw[++i]; break;
      case "--skill-path":
        args.skillPaths = [...(args.skillPaths ?? []), raw[++i]]; break;
      case "--":
        // Rest are positional
        if (!args.prompt) args.prompt = raw.slice(i + 1).join(" ");
        i = raw.length;
        break;
      default:
        if (!args.prompt && !arg.startsWith("-")) {
          args.prompt = raw.slice(i).join(" ");
          i = raw.length;
        }
        break;
    }
    i++;
  }

  return args;
}
```

## Full flag reference

| Flag | Description |
|---|---|
| `-p`, `--prompt <text>` | Single prompt (print mode). Use `-` to read from stdin |
| `--rpc` | Start in RPC mode (JSONL over stdio) |
| `-m`, `--model <id>` | Model to use (e.g., `claude-sonnet-4-6`) |
| `-s`, `--session <id>` | Resume a previous session |
| `-C`, `--cwd <path>` | Working directory (default: current directory) |
| `--config-dir <path>` | Config directory (default: `.coding-agent`) |
| `--skill-path <path>` | Additional skill directory (repeatable) |
| `-d`, `--debug` | Show error stack traces |
| `-h`, `--help` | Show help |
| `-v`, `--version` | Show version |

## The package.json bin

To make the CLI available as a command, declare the binary in `packages/coding-agent/package.json`:

```json
{
  "name": "coding-agent",
  "bin": {
    "coding-agent": "./dist/cli.js"
  }
}
```

The `#!/usr/bin/env node` shebang at the top of `cli.ts` makes it directly executable. After `npm install -g`, users can run `coding-agent` from anywhere.

## What we've built — the complete system

The Coding Agent layer is complete. Let's review what we've built across all four layers:

| Layer | Chapters | Key deliverables |
|---|---|---|
| LLM Toolkit | 7 | Unified types, EventStream, 3 provider adapters, API registry, OAuth, model registry, streaming JSON parser |
| Agent Core | 5 | Agent types, agent loop, Agent class with state machine, AgentHarness with compaction and sessions, cross-provider transforms |
| Terminal UI | 4 | TUI with differential rendering, Terminal abstraction, keybindings, widget library, autocomplete, chat interface |
| Coding Agent | 10 | AgentSession, 7 built-in tools, hooks system, system prompt + skills, session tree with branching, compaction with file tracking, config + auth, interactive mode, print mode, RPC mode, CLI entry |

In the final section, we'll build the extension system — enabling users to add custom tools, hooks, and multi-agent workflows.

---

← Previous: [Print Mode and RPC Mode](./print-and-rpc-modes.md) · Next: [The Extension API: Handlers, Context, and Events](../extensions/extension-api.md) →
