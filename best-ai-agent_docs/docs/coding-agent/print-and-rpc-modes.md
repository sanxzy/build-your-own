---
title: "Print Mode and RPC Mode: Headless Operation"
description: "Build two headless modes — Print mode for single-prompt scripting and RPC mode for IDE integrations over JSONL stdio — as lightweight alternatives to the full interactive TUI."
category: coding-agent
type: tutorial
tags: [print mode, RPC mode, -p flag, non-interactive, stdout, JSONL, stdin bridge, single prompt, coding-agent, headless, automation, programmatic, IDE integration]
keywords: [print mode, RPC mode, headless, non-interactive, JSONL protocol, IDE integration, scripting]
sources: [S52, S53]
---

**TL;DR** — Not every interaction needs a full terminal UI. We'll build two headless modes: **Print mode** (`-p "fix the bug"`) sends a single prompt, streams the response to stdout, and exits — perfect for shell scripts and CI pipelines. **RPC mode** uses JSONL over stdin/stdout, enabling IDE integrations and programmatic control through a structured request/response protocol.

## Print mode

Print mode is the simplest interaction. One prompt in, one response out. Create `packages/coding-agent/src/modes/print/print-mode.ts`:

```ts
async function runPrintMode(config: PrintConfig): Promise<void> {
  const session = new AgentSession();
  await session.start({
    cwd: config.cwd,
    modelId: config.modelId,
  });

  // Send the prompt
  session.prompt(config.prompt);

  // Stream the response to stdout
  session.subscribe((event) => {
    if (event.type === "text_delta") {
      process.stdout.write(event.delta);
    }
  });

  // Wait for completion
  await session.waitForIdle();

  // Output final newline and exit
  process.stdout.write("\n");
  process.exit(0);
}
```

Usage from the CLI:

```bash
# Single prompt
coding-agent -p "What does git status do?"

# Piped input
echo "Explain this code:" | cat - src/auth.ts | coding-agent -p -

# In a script
coding-agent -p "Add error handling to all API routes" --cwd /project
```

The `-` argument tells print mode to read the prompt from stdin. This enables shell piping:

```bash
git diff HEAD~1 | coding-agent -p "Review this diff for bugs: $(cat)"
```

## RPC mode

RPC mode is a persistent process that communicates over JSONL (JSON Lines) on stdin/stdout. Each line is a complete JSON object — a request or a response. Create `packages/coding-agent/src/modes/rpc/rpc-mode.ts`:

### Protocol

**Request** (sent on stdin):
```json
{"id":"1","method":"prompt","params":{"message":"Fix the bug"}}
{"id":"2","method":"abort","params":{}}
{"id":"3","method":"getState","params":{}}
```

**Response** (sent on stdout):
```json
{"id":"1","type":"event","event":{"type":"text_delta","delta":"I'll"}}
{"id":"1","type":"event","event":{"type":"text_delta","delta":" look"}}
{"id":"1","type":"result","result":{"stopReason":"stop","cost":0.0042}}
{"id":"2","type":"result","result":{"aborted":true}}
{"id":"3","type":"result","result":{"state":"idle","messageCount":15}}
```

### Implementation

```ts
class RpcServer {
  private session: AgentSession;
  private pending = new Map<string, (response: RpcResponse) => void>();
  private requestId = 0;

  constructor() {
    process.stdin.on("data", this.handleStdin.bind(this));
  }

  private handleStdin(data: Buffer): void {
    const lines = data.toString().trim().split("\n");
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const request: RpcRequest = JSON.parse(line);
        this.handleRequest(request);
      } catch (err) {
        this.send({ id: null, type: "error", error: `Parse error: ${err.message}` });
      }
    }
  }

  private async handleRequest(req: RpcRequest): Promise<void> {
    switch (req.method) {
      case "prompt": {
        const sub = this.session.subscribe((event) => {
          this.send({ id: req.id, type: "event", event });
        });

        this.session.prompt(req.params.message);
        await this.session.waitForIdle();
        sub(); // unsubscribe

        const messages = this.session.getMessages();
        const lastMsg = messages[messages.length - 1];
        this.send({
          id: req.id,
          type: "result",
          result: {
            stopReason: lastMsg.stopReason,
            messageCount: messages.length,
            cost: accumulateCost(messages),
          },
        });
        break;
      }

      case "abort":
        this.session.abort();
        this.send({ id: req.id, type: "result", result: { aborted: true } });
        break;

      case "getState":
        this.send({
          id: req.id,
          type: "result",
          result: {
            state: this.session.getState(),
            model: this.session.getModel(),
            messageCount: this.session.getMessages().length,
          },
        });
        break;

      case "setModel":
        await this.session.setModel(req.params.modelId);
        this.send({ id: req.id, type: "result", result: { model: this.session.getModel() } });
        break;
    }
  }

  private send(response: RpcResponse): void {
    process.stdout.write(JSON.stringify(response) + "\n");
  }
}
```

### IDE integration

RPC mode enables IDE plugins. An IDE extension spawns `coding-agent --rpc`, sends JSONL requests over stdin, and receives streaming events and results on stdout. The protocol is simple enough to implement in any language that can spawn a subprocess and read/write lines.

## What we've built

- **Print mode** — single-prompt, stdout streaming, scriptable
- **RPC mode** — JSONL protocol, persistent process, IDE-integrable
- Both modes share `AgentSession` — the same tools, models, and session management as the interactive mode

---

← Previous: [Interactive Mode](./interactive-mode.md) · Next: [The CLI Entry Point: Wiring Everything Together](./cli-entry-point.md) →
