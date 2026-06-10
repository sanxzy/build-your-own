---
title: "Print Mode and RPC Mode"
description: "Implement two headless alternatives to the interactive TUI: print mode for single-prompt stdout output and RPC mode for a JSONL stdin-to-stdout protocol bridge."
category: coding-agent
type: tutorial
tags: [print mode, rpc mode, -p flag, non-interactive, stdout, JSONL, RpcCommand, RpcEvent, stdin bridge, single prompt, coding-agent, headless, automation, programmatic, PrintModeOptions, runPrintMode, runRpcMode, AgentSessionRuntime, RpcResponse, RpcExtensionUIRequest, RpcSessionState, RpcSlashCommand, exit code, signal handling, SIGTERM, SIGHUP, compaction, bash execution, session stats]
keywords: [headless agent, non-interactive CLI, pipe agent, embed agent, programmatic control, JSONL protocol, newline-delimited JSON, xzy -p, xzy --mode rpc, xzy --mode json, print mode text, print mode json, agent pipe, stdin stdout bridge, agent automation, batch prompt]
sources: [S63, S64, S65, S89]
---

**TL;DR** — Not every use of the agent needs a full interactive terminal. This chapter builds two lightweight headless alternatives that reuse the same `AgentSession` core: *print mode*, which sends one prompt, streams the answer to stdout, and exits; and *RPC mode*, which keeps the process alive and bridges a JSONL command/event protocol over stdin and stdout. By the end you will understand the full shape of both modes, the JSONL message types, and how a host program drives the agent over a pipe.

# Print Mode and RPC Mode

## The problem: not every use is interactive

So far we have met the interactive TUI — the full-featured terminal interface where a human types messages and reads responses in a rich rendering loop (see [Interactive Mode: Startup and Wiring](./interactive-mode-startup-and-wiring.md) for how it boots). That works well for a developer at a keyboard.

But two common use-cases have no need for a TUI at all.

**The first** is a script that wants one answer and nothing else — like piping a question into the tool and capturing the result in a variable. The script does not care about rendering; it only wants text on stdout.

**The second** is another program (a GUI, an editor extension, a CI orchestrator) that wants to *drive* the agent over a pipe, sending commands and receiving structured events without any human in the loop.

Both use-cases need the same `AgentSession` engine under the hood — the same model calls, the same tool execution, the same session persistence. What changes is how input arrives and how output leaves. Let's implement each mode in turn.

> **Quick recap — AgentSession.** `AgentSession` is the mode-agnostic core that runs the agent loop, holds the conversation state, and emits structured events. Every mode — interactive, print, or RPC — drives the same session; they differ only in how they feed it prompts and what they do with the events it emits. See [AgentSession: The Mode-Agnostic Core](./agent-session-core.md) for the full picture.

---

## Print mode: one prompt, text out, done

### The goal

We want `xzy -p "summarise this file"` to write the assistant's final reply to stdout and exit with a success or failure code. No TUI, no interactive prompt, no lingering process.

### The entry point

Print mode is implemented by `runPrintMode`, an async function in `src/modes/print-mode.ts`:

```ts
// Simplified view of the public signature
export interface PrintModeOptions {
  /** "text" for the final response only; "json" for all session events as JSONL */
  mode: "text" | "json";
  /** First message to send (may contain @file references) */
  initialMessage?: string;
  /** Images to attach to the initial message */
  initialImages?: ImageContent[];
  /** Additional prompts to send after initialMessage */
  messages?: string[];
}

export async function runPrintMode(
  runtimeHost: AgentSessionRuntime,
  options: PrintModeOptions
): Promise<number>   // returns an exit code
```

`AgentSessionRuntime` — introduced here because print mode is the first place we meet it — is a thin wrapper around an `AgentSession` that adds lifecycle operations: `dispose()`, `newSession()`, `fork()`, and `switchSession()`. It also carries a `session` property that gives you the current live session. You can think of it as "session plus host-level operations".

### Two output sub-modes

`PrintModeOptions.mode` picks between two behaviours:

| `mode` value | What goes to stdout |
|---|---|
| `"text"` | The assistant's final text response only, followed by a newline. |
| `"json"` | Every session event as a newline-delimited JSON object (JSONL), including the session header. |

The `"json"` sub-mode is how another tool can capture a full structured trace of a single run — every streaming chunk, tool call, and status change — without the overhead of a full RPC session.

### Walking through the implementation

**Step 1 — signal handling.** Before sending any prompt, print mode registers handlers for `SIGTERM` and (on non-Windows) `SIGHUP`. If either signal arrives, the mode calls `killTrackedDetachedChildren()` to clean up any spawned child processes, then `disposeRuntime()`, then exits with code `143` for `SIGTERM` or `129` for `SIGHUP`. Signal cleanup is deregistered in the `finally` block so no handlers leak between runs.

**Step 2 — bind extensions.** The mode calls `session.bindExtensions(...)` to wire up the active extensions. Extensions receive a `mode` value of `"json"` when JSON output is active or `"print"` when text output is active. This lets extensions adapt their behaviour — an extension that emits status widgets, for example, does nothing in print mode.

**Step 3 — emit the JSON header (JSON sub-mode only).** When `mode === "json"`, before sending any prompt the session header is fetched via `session.sessionManager.getHeader()` and written to stdout:

```ts
if (mode === "json") {
  const header = session.sessionManager.getHeader();
  if (header) {
    writeRawStdout(`${JSON.stringify(header)}\n`);
  }
}
```

Every subsequent event is also serialised as `${JSON.stringify(event)}\n`, framed with a single newline. This is the same framing rule used by RPC mode — **split on `\n` only**, never `\r\n`.

**Step 4 — subscribe to session events (JSON sub-mode only).** Still before sending the prompt, an event subscriber is registered:

```ts
unsubscribe = session.subscribe((event) => {
  if (mode === "json") {
    writeRawStdout(`${JSON.stringify(event)}\n`);
  }
});
```

In `"text"` mode no subscriber is needed; the final message is read from session state after the prompt completes.

**Step 5 — send the prompts.** The initial message (if any) is sent first, then any additional messages from `options.messages`:

```ts
if (initialMessage) {
  await session.prompt(initialMessage, { images: initialImages });
}
for (const message of messages) {
  await session.prompt(message);
}
```

Each `session.prompt()` call awaits the full agent turn — the model call, any tool executions, and the final response — before resolving.

**Step 6 — extract and print the response (text sub-mode only).** After all prompts have resolved, the `"text"` path reads the last message from session state:

```ts
const state = session.state;
const lastMessage = state.messages[state.messages.length - 1];

if (lastMessage?.role === "assistant") {
  const assistantMsg = lastMessage as AssistantMessage;
  if (assistantMsg.stopReason === "error" || assistantMsg.stopReason === "aborted") {
    console.error(assistantMsg.errorMessage || `Request ${assistantMsg.stopReason}`);
    exitCode = 1;
  } else {
    for (const content of assistantMsg.content) {
      if (content.type === "text") {
        writeRawStdout(`${content.text}\n`);
      }
    }
  }
}
```

Notice the `stopReason` check: if the agent stopped due to an error or was aborted, the error message is written to `stderr` and the exit code becomes `1`. A clean response exits with `0`.

**Step 7 — cleanup.** The `finally` block always fires: signal handlers are removed, `disposeRuntime()` is called, and `flushRawStdout()` drains any buffered output before the process returns.

### A complete print-mode run

Here is what a single-prompt text run looks like end-to-end:

```bash
# Ask a question; the answer appears on stdout; the process exits when done
xzy -p "What is the current working directory?"

# Capture the output in a shell variable
result=$(xzy -p "List the files in src/ as a comma-separated string")
echo "Agent replied: $result"
```

And a JSON-mode run that captures every event:

```bash
# JSON sub-mode: every event on stdout as a JSONL line
xzy --mode json "Summarise README.md" > events.jsonl
```

The file `events.jsonl` will contain one JSON object per line: the session header first, then every streaming chunk, tool call, and final message event.

---

## RPC mode: a JSONL pipe protocol

### The problem print mode does not solve

Print mode is great for scripted one-shots. But a GUI application or an editor extension needs something richer: it wants to send a sequence of commands over the lifetime of the agent, receive structured events as they arrive, and control session-level operations like forking or model switching — without human interaction.

We need a long-lived process that reads commands and writes events, both as newline-delimited JSON. That is RPC mode.

### How the protocol works

The protocol has three communication directions:

| Direction | Description |
|---|---|
| stdin → process | `RpcCommand` — a JSON object with a `type` field, sent one per line |
| process → stdout | `RpcResponse` — a synchronous reply to a command, one per line |
| process → stdout | Session events — `AgentSessionEvent` objects emitted as they happen |
| process → stdout | `RpcExtensionUIRequest` — when an extension needs user input |
| stdin → process | `RpcExtensionUIResponse` — the host's reply to a UI request |

**Framing rule.** Every object in both directions is serialised as `JSON.stringify(obj) + "\n"`. The reader splits on `"\n"` exclusively — never `"\r\n"`. A line that is not valid JSON produces an error response and processing continues.

**Correlation.** Every `RpcCommand` carries an optional `id` field. The matching `RpcResponse` echoes the same `id`. This lets a host correlate responses to commands when commands are sent concurrently.

### Starting the mode

RPC mode is implemented by `runRpcMode` in `src/modes/rpc/rpc-mode.ts`:

```ts
// Simplified signature
export async function runRpcMode(
  runtimeHost: AgentSessionRuntime
): Promise<never>   // keeps the process alive until stdin closes or a signal arrives
```

Notice the return type `Promise<never>`. Unlike `runPrintMode`, this function never resolves normally — it runs an infinite loop driven by stdin:

```ts
// Simplified view of the keepalive at the end of runRpcMode
return new Promise(() => {});
```

The process stays alive until stdin closes (the host closed the pipe) or a signal is received.

### The `RpcCommand` type: every command at a glance

All commands sent on stdin are typed as `RpcCommand`. Let's look at the full set, grouped by category. Every variant may carry an optional `id` string for correlation.

**Prompting commands**

| `type` | Fields | Effect |
|---|---|---|
| `"prompt"` | `message`, optional `images`, optional `streamingBehavior: "steer" \| "followUp"` | Send a user message; events stream back asynchronously |
| `"steer"` | `message`, optional `images` | Send a steering message mid-response |
| `"follow_up"` | `message`, optional `images` | Send a follow-up message |
| `"abort"` | — | Abort the current running prompt |
| `"new_session"` | optional `parentSession` | Replace the current session with a fresh one |

**State commands**

| `type` | Fields | Effect |
|---|---|---|
| `"get_state"` | — | Returns an `RpcSessionState` snapshot |

**Model commands**

| `type` | Fields | Effect |
|---|---|---|
| `"set_model"` | `provider`, `modelId` | Switch to a specific model |
| `"cycle_model"` | — | Advance to the next available model |
| `"get_available_models"` | — | Returns `{ models: Model[] }` |

**Thinking commands**

| `type` | Fields | Effect |
|---|---|---|
| `"set_thinking_level"` | `level: ThinkingLevel` | Set the reasoning depth |
| `"cycle_thinking_level"` | — | Advance to the next thinking level |

**Queue-mode commands**

| `type` | Fields | Effect |
|---|---|---|
| `"set_steering_mode"` | `mode: "all" \| "one-at-a-time"` | Control how steering messages queue |
| `"set_follow_up_mode"` | `mode: "all" \| "one-at-a-time"` | Control how follow-up messages queue |

**Compaction commands**

| `type` | Fields | Effect |
|---|---|---|
| `"compact"` | optional `customInstructions` | Trigger context compaction; returns `CompactionResult` |
| `"set_auto_compaction"` | `enabled: boolean` | Enable or disable automatic compaction |

**Retry commands**

| `type` | Fields | Effect |
|---|---|---|
| `"set_auto_retry"` | `enabled: boolean` | Enable or disable automatic retry on failure |
| `"abort_retry"` | — | Cancel a pending retry |

**Bash commands**

| `type` | Fields | Effect |
|---|---|---|
| `"bash"` | `command`, optional `excludeFromContext: boolean` | Execute a shell command; returns `BashResult` |
| `"abort_bash"` | — | Cancel the running bash command |

**Session-management commands**

| `type` | Fields | Effect |
|---|---|---|
| `"get_session_stats"` | — | Returns `SessionStats` |
| `"export_html"` | optional `outputPath` | Export the session to an HTML file |
| `"switch_session"` | `sessionPath` | Switch to a different session file |
| `"fork"` | `entryId` | Fork the conversation from a specific entry |
| `"clone"` | — | Clone the conversation at the current leaf |
| `"get_fork_messages"` | — | Returns user messages available as fork points |
| `"get_last_assistant_text"` | — | Returns the last assistant text or null |
| `"set_session_name"` | `name` | Rename the current session |
| `"get_messages"` | — | Returns all messages in the conversation |
| `"get_commands"` | — | Returns all commands registered by extensions, prompt templates, and skills |

### The `RpcResponse` shape

Every response has the same envelope:

```ts
// Success — with optional data
{ id?: string; type: "response"; command: "<command-type>"; success: true; data?: ... }

// Failure — any command can fail
{ id?: string; type: "response"; command: "<command-type>"; success: false; error: string }
```

The `command` field mirrors the `type` of the command that triggered it. The `data` field carries command-specific payload when `success` is true (absent for commands that only acknowledge). The `error` field carries a plain-text message when `success` is false.

### `RpcSessionState`: the state snapshot

When you send `get_state`, you receive an `RpcSessionState`:

```ts
interface RpcSessionState {
  model?: Model<any>;          // currently selected model
  thinkingLevel: ThinkingLevel;
  isStreaming: boolean;
  isCompacting: boolean;
  steeringMode: "all" | "one-at-a-time";
  followUpMode: "all" | "one-at-a-time";
  sessionFile?: string;        // path to the persisted session file
  sessionId: string;
  sessionName?: string;
  autoCompactionEnabled: boolean;
  messageCount: number;
  pendingMessageCount: number;
}
```

This is a flat, serialisable snapshot — no live references. Use it to initialise host-side state after connecting, or to reconcile state after a session switch.

### Walking through the implementation

**Step 1 — take over stdout.** The very first thing `runRpcMode` does is call `takeOverStdout()`. This redirects all stdout writes through `writeRawStdout`, preventing anything else (console.log, third-party libraries) from interleaving unstructured text with the JSONL stream.

**Step 2 — bind extensions with an RPC UI context.** Like print mode, RPC mode calls `session.bindExtensions(...)`. The `mode` is `"rpc"`, and the `uiContext` is an `ExtensionUIContext` implementation that replaces TUI widgets with RPC protocol messages:

- Dialog methods (`select`, `confirm`, `input`, `editor`) emit an `RpcExtensionUIRequest` on stdout and await a matching `extension_ui_response` from stdin.
- Fire-and-forget methods (`notify`, `setStatus`, `setTitle`, `setWidget`, `setEditorText`) emit an `RpcExtensionUIRequest` with no response expected.
- TUI-only methods (`setWorkingMessage`, `setWorkingVisible`, `addAutocompleteProvider`, etc.) are no-ops in RPC mode — they require TUI access that does not exist here.

**Step 3 — subscribe to session events.** After binding, all session events are forwarded to stdout:

```ts
unsubscribe = session.subscribe((event) => {
  output(event);   // serializes as JSON + "\n"
});
```

A backpressure subscriber is also registered on the agent. When stdout's write buffer fills up, the agent loop waits before continuing — this prevents unbounded memory growth if the host is reading slowly:

```ts
unsubscribeBackpressure = session.agent.subscribe(async () => {
  await waitForRawStdoutBackpressure();
});
```

**Step 4 — attach the JSONL line reader.** The stdin reader is attached via `attachJsonlLineReader`:

```ts
detachInput = (() => {
  const detachJsonl = attachJsonlLineReader(process.stdin, (line) => {
    void handleInputLine(line);
  });
  return () => {
    detachJsonl();
    process.stdin.off("end", onInputEnd);
  };
})();
```

`attachJsonlLineReader` splits incoming bytes on `"\n"` and calls the callback for each complete line. The `detachInput` function stored here is called during shutdown to stop reading.

**Step 5 — handle each line.** `handleInputLine` parses the JSON, dispatches to `handleCommand`, emits the response, and then calls `checkShutdownRequested()`:

```ts
const handleInputLine = async (line: string) => {
  let parsed: unknown;
  try {
    parsed = JSON.parse(line);
  } catch (parseError) {
    output(error(undefined, "parse", `Failed to parse command: ...`));
    await waitForRawStdoutBackpressure();
    return;
  }

  // Extension UI responses are not commands — route them to pending requests
  if (parsed?.type === "extension_ui_response") {
    const response = parsed as RpcExtensionUIResponse;
    const pending = pendingExtensionRequests.get(response.id);
    if (pending) {
      pendingExtensionRequests.delete(response.id);
      pending.resolve(response);
    }
    return;
  }

  const command = parsed as RpcCommand;
  const response = await handleCommand(command);
  if (response) {
    output(response);
    await waitForRawStdoutBackpressure();
  }
  await checkShutdownRequested();
};
```

You might wonder why `extension_ui_response` is handled before the command dispatcher. The answer is that UI responses are not commands — they are replies to questions the server asked. Routing them through the pending-requests map resolves the waiting dialog promise without going through `handleCommand` at all.

**Step 6 — shutdown.** The `shutdown` function is called when stdin closes (`process.stdin.on("end", onInputEnd)`) or a signal arrives. It disposes the runtime, detaches stdin, flushes stdout, and calls `process.exit()`. It is guarded by a `shuttingDown` flag so repeated signals do not double-dispose.

### The `prompt` command in detail: async acknowledgement

Most commands are synchronous — you send, you wait, you get a response, done. `prompt` is the notable exception.

When the host sends a `prompt` command, the agent does not immediately produce a full reply. Instead, the session begins streaming. The acknowledgement (`type: "response", command: "prompt", success: true`) is emitted only after *preflight* succeeds — that is, after the prompt is accepted into the queue. The actual assistant reply arrives later, as a stream of session events:

```ts
case "prompt": {
  let preflightSucceeded = false;
  void session
    .prompt(command.message, {
      images: command.images,
      streamingBehavior: command.streamingBehavior,
      source: "rpc",
      preflightResult: (didSucceed) => {
        if (didSucceed) {
          preflightSucceeded = true;
          output(success(id, "prompt"));   // ← emitted early, before events
        }
      },
    })
    .catch((e) => {
      if (!preflightSucceeded) {
        output(error(id, "prompt", e.message));
      }
    });
  return undefined;  // ← no response from handleCommand; response already emitted via callback
}
```

This design means the host can send additional commands (like `abort`) while the agent is still running. The stream of session events tells the host when the turn is done.

### A complete RPC session transcript

Here is a generic example of a host program driving the agent over a pipe. Each line is one JSON object; comments explain the direction.

```jsonl
// → stdin: send a prompt
{"id": "req-1", "type": "prompt", "message": "What is 2 + 2?"}

// ← stdout: preflight acknowledgement
{"id": "req-1", "type": "response", "command": "prompt", "success": true}

// ← stdout: session events stream (simplified)
{"type": "message_start", "message": {"role": "assistant", ...}}
{"type": "content_block_delta", "delta": {"type": "text_delta", "text": "4"}}
{"type": "message_end", "message": {"role": "assistant", "content": [{"type": "text", "text": "4"}], ...}}

// → stdin: query session state
{"id": "req-2", "type": "get_state"}

// ← stdout: state snapshot
{"id": "req-2", "type": "response", "command": "get_state", "success": true,
 "data": {"isStreaming": false, "messageCount": 2, ...}}

// → stdin: run a bash command
{"id": "req-3", "type": "bash", "command": "echo hello"}

// ← stdout: bash result
{"id": "req-3", "type": "response", "command": "bash", "success": true,
 "data": {"output": "hello\n", "exitCode": 0, "cancelled": false}}

// stdin closes → process shuts down cleanly
```

### Verifying the contract with tests (S89)

The test suite in `test/rpc.test.ts` exercises the full protocol using an `RpcClient` helper that starts the CLI process and wraps commands as async calls. The suite runs only when API credentials are present (it makes real model calls). A few properties verified by the tests:

- After `start()`, `get_state` returns a model with `provider: "anthropic"` and `isStreaming: false`.
- `promptAndWait("...")` collects all events until `message_end` and returns them; the result contains at least one user message event and one assistant message event.
- After `promptAndWait`, `messageCount` is greater than zero; after `newSession`, it resets to zero.
- `bash("echo hello")` returns `{ output: "hello\n", exitCode: 0, cancelled: false }`.
- A bash command's output is present in the session context (as a `bashExecution`-role message), so a follow-up prompt can reference it.
- `compact()` returns a `CompactionResult` with a non-empty `summary` and `tokensBefore > 0`; the compaction entry appears in the persisted session `.jsonl` file.
- `exportHtml()` returns a path to an existing `.html` file.
- `setSessionName("my-test-session")` persists a `session_info` entry in the `.jsonl` session file with the supplied name.

These tests document the exact observable contract a host program can rely on.

### Extension UI requests and responses

When an extension (a plugin loaded into the session) needs user input — for example, to ask whether to proceed with a destructive action — it calls into the `ExtensionUIContext`. In RPC mode, that context emits an `RpcExtensionUIRequest` on stdout instead of showing a TUI dialog:

```ts
// Variants emitted as RpcExtensionUIRequest on stdout
type RpcExtensionUIRequest =
  | { type: "extension_ui_request"; id: string; method: "select"; title: string; options: string[]; timeout?: number }
  | { type: "extension_ui_request"; id: string; method: "confirm"; title: string; message: string; timeout?: number }
  | { type: "extension_ui_request"; id: string; method: "input"; title: string; placeholder?: string; timeout?: number }
  | { type: "extension_ui_request"; id: string; method: "editor"; title: string; prefill?: string }
  | { type: "extension_ui_request"; id: string; method: "notify"; message: string; notifyType?: "info" | "warning" | "error" }
  | { type: "extension_ui_request"; id: string; method: "setStatus"; statusKey: string; statusText: string | undefined }
  | { type: "extension_ui_request"; id: string; method: "setWidget"; widgetKey: string; widgetLines: string[] | undefined; widgetPlacement?: "aboveEditor" | "belowEditor" }
  | { type: "extension_ui_request"; id: string; method: "setTitle"; title: string }
  | { type: "extension_ui_request"; id: string; method: "set_editor_text"; text: string }
```

The host responds on stdin with `RpcExtensionUIResponse`:

```ts
type RpcExtensionUIResponse =
  | { type: "extension_ui_response"; id: string; value: string }      // for select/input/editor
  | { type: "extension_ui_response"; id: string; confirmed: boolean } // for confirm
  | { type: "extension_ui_response"; id: string; cancelled: true }    // user dismissed
```

The `id` in the response must match the `id` from the request. If a timeout or abort signal fires before the host responds, the dialog resolves to its default value (`undefined` for selection/input, `false` for confirm) without blocking the extension.

A few UI methods — `setWorkingMessage`, `setWorkingVisible`, `setWorkingIndicator`, `setHiddenThinkingLabel`, `addAutocompleteProvider`, `setFooter`, `setHeader`, `getToolsExpanded`, `setToolsExpanded` — are unsupported in RPC mode (they require TUI access) and are no-ops.

---

## Comparing the two headless modes

| Concern | Print mode | RPC mode |
|---|---|---|
| Use case | Scripted single-shot prompts | Long-lived programmatic control |
| Input | Prompts from CLI args | `RpcCommand` objects over stdin |
| Output (`"text"`) | Assistant's final text to stdout | N/A |
| Output (`"json"`) | All events as JSONL to stdout | All events as JSONL to stdout (always) |
| Lifetime | Exits after all prompts complete | Runs until stdin closes or signal received |
| Session operations | `newSession`, `fork`, `switchSession` via CLI flags | All session operations via commands |
| Extension UI | Mode `"print"` or `"json"` — no dialogs | Full dialog protocol over stdin/stdout |
| Process exit | Returns exit code `0` or `1` | `process.exit()` on shutdown |
| Signal handling | `SIGTERM` → exit 143; `SIGHUP` → exit 129 | `SIGTERM` → exit 143; `SIGHUP` → exit 129 |

Both modes:
- Drive the same `AgentSession` — same agent loop, same tool execution, same session persistence.
- Call `session.bindExtensions(...)` with a mode string before sending the first prompt.
- Subscribe to session events and forward them to stdout.
- Flush stdout before exiting via `flushRawStdout()`.

---

## Launching the modes

The CLI selects between modes based on flags. Print mode is activated with the `-p` flag (for text output) or `--mode json` (for JSON event stream). RPC mode is activated with `--mode rpc`. The argument-parsing and mode-dispatch logic lives in the CLI entry point — that is the subject of the next chapter.

```bash
# Print mode — text output
xzy -p "What files are in src/?"

# Print mode — JSON event stream
xzy --mode json "What files are in src/?"

# RPC mode — JSONL protocol over stdin/stdout
xzy --mode rpc
```

See [The CLI Entry Point: Argument Parsing and Mode Selection](./cli-entry-point.md) for how these flags are parsed and the modes are selected.

---

← Previous: [Interactive Mode: Input, Keyboard Shortcuts, and Session Commands](./interactive-mode-input-and-shortcuts.md) · Next: [The CLI Entry Point: Argument Parsing and Mode Selection](./cli-entry-point.md) →
