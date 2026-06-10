---
title: "Interactive Mode: The Full Terminal Chat Experience"
description: "Assemble the interactive mode — wiring AgentSession to the TUI, loading extensions and themes, registering slash commands, binding keyboard shortcuts, and managing the complete event loop."
category: coding-agent
type: tutorial
tags: [interactive mode, TUI app, startup, AgentSession, extensions, application shell, initialization, event loop, slash commands, theme, diff view, keyboard shortcuts, session management, coding-agent]
keywords: [interactive mode, TUI integration, event loop, slash commands, theme, session UI]
sources: [S44, S51]
---

**TL;DR** — The interactive mode is the main way users interact with the coding agent. It wires `AgentSession` to the TUI we built in the previous section, registers slash commands (`/help`, `/model`, `/session`), binds keyboard shortcuts (`Ctrl+C` to abort, `Ctrl+L` to clear), loads themes for color customization, and runs the event loop that connects user input to agent responses.

## The application shell

Create `packages/coding-agent/src/modes/interactive/app.ts`:

```ts
class InteractiveApp {
  private tui: Tui;
  private session: AgentSession;
  private chat: ChatInterface;
  private commandRegistry = new Map<string, SlashCommand>();
  private theme: Theme;

  async start(config: InteractiveConfig): Promise<void> {
    // 1. Initialize session
    this.session = new AgentSession();
    await this.session.start({
      sessionId: config.sessionId,
      cwd: config.cwd,
      modelId: config.modelId,
      skillPaths: config.skillPaths,
    });

    // 2. Load theme
    this.theme = await loadTheme(config.theme ?? "dark");

    // 3. Set up TUI
    const terminal = new ProcessTerminal();
    this.tui = new Tui(terminal);
    this.chat = new ChatInterface(this.theme);

    // 4. Register slash commands
    this.registerCommands();

    // 5. Wire agent events to UI
    this.wireAgentToUI();

    // 6. Wire UI input to agent
    this.wireUIToAgent();

    // 7. Start the event loop
    this.tui.mount(this.chat);
    this.tui.start();
  }
}
```

## Wiring agent events to UI

The agent emits streaming events; the UI renders them:

```ts
private wireAgentToUI(): void {
  this.session.subscribe((event) => {
    switch (event.type) {
      case "turn_start":
        this.chat.showLoader("Thinking...");
        break;

      case "text_start":
        this.chat.markdown.startBlock(event.contentIndex);
        break;

      case "text_delta":
        this.chat.markdown.appendToBlock(event.contentIndex, event.delta);
        break;

      case "text_end":
        this.chat.markdown.finishBlock(event.contentIndex);
        break;

      case "tool_execution_start":
        this.chat.showToolStatus(event.toolCall.name, "running");
        break;

      case "tool_execution_end":
        this.chat.showToolStatus(event.toolCall.name, event.isError ? "error" : "done");
        break;

      case "assistant_message":
        this.chat.appendMessage(event.message);
        break;

      case "turn_end":
        this.chat.hideLoader();
        this.tui.render();
        break;

      case "error":
        this.chat.showError(event.error);
        break;
    }
  });
}
```

## Wiring UI input to agent

User keystrokes go through the chat interface, which emits structured actions:

```ts
private wireUIToAgent(): void {
  this.chat.onSubmit((message) => {
    if (message.startsWith("/")) {
      this.handleSlashCommand(message);
    } else {
      this.session.prompt(message);
    }
  });

  this.chat.onAbort(() => {
    this.session.abort();
  });
}
```

## Slash commands

Commands are registered with a name, description, and handler:

```ts
interface SlashCommand {
  name: string;
  description: string;
  handler: (args: string, context: CommandContext) => Promise<string>;
}

private registerCommands(): void {
  this.command("/help", "Show available commands", async () => {
    return Array.from(this.commandRegistry.entries())
      .map(([name, cmd]) => `/${name} — ${cmd.description}`)
      .join("\n");
  });

  this.command("/model", "Change the model", async (args) => {
    const models = this.session.listModels();
    const match = fuzzyFind(args.trim(), models);
    if (!match) return `No model matching "${args}"`;
    await this.session.setModel(match.id);
    return `Switched to ${match.name}`;
  });

  this.command("/session", "Manage sessions", async (args) => {
    const [sub, ...rest] = args.trim().split(/\s+/);
    switch (sub) {
      case "list": return this.listSessions();
      case "switch": return this.switchSession(rest[0]);
      case "fork": return this.forkSession(rest[0]);
      case "new": return this.newSession();
      default: return "Usage: /session [list|switch <id>|fork <id>|new]";
    }
  });

  this.command("/theme", "Change the theme", async (args) => {
    this.theme = await loadTheme(args.trim() || "dark");
    this.tui.render();
    return `Theme: ${this.theme.name}`;
  });
}
```

## Keyboard shortcuts

The TUI keybinding system is configured with agent-specific shortcuts:

```ts
private configureKeybindings(): void {
  this.tui.keybindings.add([
    { keys: "ctrl+c", action: "abort", description: "Stop the current agent turn" },
    { keys: "ctrl+l", action: "clear", description: "Clear the screen" },
    { keys: "ctrl+r", action: "history", description: "Search input history" },
    { keys: "ctrl+o", action: "session_overview", description: "Show session tree" },
    { keys: "ctrl+d", action: "exit", description: "Exit (on empty input)" },
  ]);
}
```

## Theme system

Themes define colors for every UI element:

```ts
interface Theme {
  name: string;
  colors: {
    background: string;
    foreground: string;
    accent: string;
    error: string;
    success: string;
    muted: string;
    userMessage: string;
    assistantMessage: string;
    toolCall: string;
    codeBlock: string;
    diffAdded: string;
    diffRemoved: string;
  };
}
```

Themes are JSON files in the theme directory. The active theme is applied globally — all components reference `theme.colors.*` when choosing ANSI styles.

## The event loop

The interactive app runs until the user exits:

```ts
async run(): Promise<void> {
  this.tui.start();

  return new Promise((resolve) => {
    this.tui.onExit(() => {
      this.session.save();
      this.tui.stop();
      resolve();
    });
  });
}
```

## What we've built

- **Interactive app** wiring `AgentSession` to the TUI
- **Agent-to-UI event bridge** — streaming text, tool status, errors
- **UI-to-agent input bridge** — user messages, slash commands, abort
- **Slash commands** — `/help`, `/model`, `/session`, `/theme`
- **Keyboard shortcuts** — abort, clear, history, session overview
- **Theme system** — configurable ANSI color palette

---

← Previous: [Model Registry, Settings, and Configuration](./model-registry-and-config.md) · Next: [Print Mode and RPC Mode: Headless Operation](./print-and-rpc-modes.md) →
