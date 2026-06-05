---
title: "Interactive Mode: Input, Keyboard Shortcuts, and Session Commands"
description: "How user keystrokes flow through the input pipeline into actions, overlays, and message rendering in the interactive TUI."
category: coding-agent
type: tutorial
tags: [input handling, keyboard shortcuts, session commands, interactive mode, overlays, diff view, selectors, summaries, coding-agent, editor, hotkeys, user interaction, key events, matchesKey, keybindings, model selector, session selector, settings selector, tool execution, assistant message, user message, footer, compaction summary, branch summary, diff viewer, slash commands, bash mode]
keywords: [key handler, onEscape, onAction, cycleModel, cycleThinkingLevel, showModelSelector, showSessionSelector, showSettingsSelector, showTreeSelector, ToolExecutionComponent, AssistantMessageComponent, UserMessageComponent, FooterComponent, BranchSummaryMessageComponent, CompactionSummaryMessageComponent, renderDiff, toggle tool output, thinking block visibility, follow-up, steer, dequeue, bash command, external editor, clipboard paste]
sources: [S60, S61]
---

**TL;DR** — In the previous chapter we mounted the TUI app shell and started its render loop. Now we look at what happens when the user presses a key: how raw key events map to actions, how slash commands and `!`-prefixed bash commands are dispatched, and how overlay selectors (model picker, session picker, settings panel) are opened and closed. We then walk through the rendering components that put messages, tool calls, diffs, and summaries on screen.

# Interactive Mode: Input, Keyboard Shortcuts, and Session Commands

In [Interactive Mode: Startup, Wiring, and the TUI App Shell](./interactive-mode-startup-and-wiring.md) we saw how `InteractiveMode` constructs its `TUI` instance, lays out the container hierarchy, wires the `AgentSession` subscription, and calls `ui.start()`. The app shell is now running — but it only becomes useful when the user can actually type something.

This chapter is Part 2: the **input pipeline**. We will follow a keystroke from the moment it arrives in the editor, through the binding table that maps keys to actions, out to the selectors and overlays that pop up in response, and finally to the rendering components that paint the conversation on screen.

A quick prerequisite recap: the TUI layer exposes a `matchesKey(data, keyId)` helper that compares a raw terminal byte sequence against a named binding. The full binding mechanism is covered in [Terminal Abstraction and Input](../terminal-ui/terminal-abstraction-and-input.md); here we treat `matchesKey` and `KeybindingsManager` as given and focus on how `InteractiveMode` uses them.

---

## 1. The Editor as the Input Hub

After `init()` runs, the TUI's focus is set to the editor component:

```ts
// Simplified view of init() — the editor receives all keystrokes
this.ui.setFocus(this.editor);
```

`this.editor` is initially a `CustomEditor` instance (stored as `this.defaultEditor`). Extensions can swap in a custom editor component, but the default editor is always the fallback. When the user types, keystrokes flow into `this.editor.handleInput()`.

The editor handles its own text editing (moving the cursor, deleting words, autocomplete). But it also surfaces a set of **escape hatches** — callbacks and named action handlers — that `InteractiveMode` populates in `setupKeyHandlers()`. Those callbacks are how keystrokes escape the editor and drive the rest of the application.

```ts
// Simplified view of setupKeyHandlers()
private setupKeyHandlers(): void {
  // Escape key: abort streaming, exit bash mode, or trigger double-escape action
  this.defaultEditor.onEscape = () => { /* ... see below */ };

  // Named app-level actions registered on the editor
  this.defaultEditor.onAction("app.clear",              () => this.handleCtrlC());
  this.defaultEditor.onAction("app.suspend",            () => this.handleCtrlZ());
  this.defaultEditor.onAction("app.thinking.cycle",     () => this.cycleThinkingLevel());
  this.defaultEditor.onAction("app.model.cycleForward", () => this.cycleModel("forward"));
  this.defaultEditor.onAction("app.model.cycleBackward",() => this.cycleModel("backward"));
  this.defaultEditor.onAction("app.model.select",       () => this.showModelSelector());
  this.defaultEditor.onAction("app.tools.expand",       () => this.toggleToolOutputExpansion());
  this.defaultEditor.onAction("app.thinking.toggle",    () => this.toggleThinkingBlockVisibility());
  this.defaultEditor.onAction("app.editor.external",    () => this.openExternalEditor());
  this.defaultEditor.onAction("app.message.followUp",   () => this.handleFollowUp());
  this.defaultEditor.onAction("app.message.dequeue",    () => this.handleDequeue());
  this.defaultEditor.onAction("app.session.new",        () => this.handleClearCommand());
  this.defaultEditor.onAction("app.session.tree",       () => this.showTreeSelector());
  this.defaultEditor.onAction("app.session.fork",       () => this.showUserMessageSelector());
  this.defaultEditor.onAction("app.session.resume",     () => this.showSessionSelector());

  // Editor text change: detect bash mode (line starts with "!")
  this.defaultEditor.onChange = (text: string) => {
    const wasBashMode = this.isBashMode;
    this.isBashMode = text.trimStart().startsWith("!");
    if (wasBashMode !== this.isBashMode) {
      this.updateEditorBorderColor();
    }
  };

  // Clipboard image paste (triggered on Ctrl+V)
  this.defaultEditor.onPasteImage = () => {
    this.handleClipboardImagePaste();
  };
}
```

Notice that `onAction` takes an **action name** string (like `"app.model.select"`) rather than a raw key sequence. The `KeybindingsManager` resolves which physical key that name is bound to — the binding is configurable, so the application logic never hard-codes a key code.

---

## 2. The Escape Key: Context-Sensitive Behaviour

The Escape key deserves special attention because what it does depends entirely on the current application state:

```ts
this.defaultEditor.onEscape = () => {
  if (this.session.isStreaming) {
    // Abort the ongoing request and restore any queued messages to the editor
    this.restoreQueuedMessagesToEditor({ abort: true });
  } else if (this.session.isBashRunning) {
    // Cancel the running bash command
    this.session.abortBash();
  } else if (this.isBashMode) {
    // Clear the editor and leave bash mode
    this.editor.setText("");
    this.isBashMode = false;
    this.updateEditorBorderColor();
  } else if (!this.editor.getText().trim()) {
    // Double-escape with an empty editor: trigger the configured action
    const action = this.settingsManager.getDoubleEscapeAction();
    if (action !== "none") {
      const now = Date.now();
      if (now - this.lastEscapeTime < 500) {
        if (action === "tree") {
          this.showTreeSelector();
        } else {
          this.showUserMessageSelector(); // "fork"
        }
        this.lastEscapeTime = 0;
      } else {
        this.lastEscapeTime = now;
      }
    }
  }
};
```

The 500 ms threshold for the double-escape is tracked via `this.lastEscapeTime`. The configured double-escape action can be `"tree"`, `"fork"`, or `"none"` — this comes from `settingsManager.getDoubleEscapeAction()`.

---

## 3. Keyboard Shortcut Reference Table

The `/hotkeys` command builds a reference table dynamically from the resolved key names. Here is the same information as a static reference table, using the action identifiers from the source:

| Action ID | Category | Effect |
|---|---|---|
| `tui.input.submit` (Enter) | Editing | Send message to the agent |
| `tui.input.newLine` | Editing | Insert a newline in the editor |
| `tui.editor.deleteToLineEnd` | Editing | Delete from cursor to end of line |
| `tui.editor.deleteToLineStart` | Editing | Delete from cursor to start of line |
| `tui.editor.deleteWordBackward` | Editing | Delete word to the left |
| `tui.editor.deleteWordForward` | Editing | Delete word to the right |
| `tui.editor.yank` | Editing | Paste most-recently-deleted text |
| `tui.editor.yankPop` | Editing | Cycle through deleted text after paste |
| `tui.editor.undo` | Editing | Undo last edit |
| `tui.input.tab` | Editing | Path completion / accept autocomplete |
| `app.interrupt` (Ctrl+C) | App | Cancel autocomplete; abort streaming |
| `app.clear` (Ctrl+C×2) | App | Clear editor (first press); exit (second press within 500 ms) |
| `app.exit` (Ctrl+D) | App | Exit when editor is empty |
| `app.suspend` (Ctrl+Z) | App | Suspend to background (non-Windows) |
| `app.thinking.cycle` | Model | Cycle thinking level |
| `app.model.cycleForward` / `app.model.cycleBackward` | Model | Step through available models |
| `app.model.select` | Model | Open the model selector overlay |
| `app.tools.expand` | Display | Toggle tool output expansion |
| `app.thinking.toggle` | Display | Toggle thinking block visibility |
| `app.editor.external` | Editor | Open current text in `$VISUAL` / `$EDITOR` |
| `app.message.followUp` | Queue | Queue current text as a follow-up (waits until agent finishes) |
| `app.message.dequeue` | Queue | Restore all queued messages back to the editor |
| `app.clipboard.pasteImage` | Input | Paste image from clipboard |
| `app.session.new` | Session | Start a new session |
| `app.session.tree` | Session | Open the tree/branch navigator |
| `app.session.fork` | Session | Open the user-message fork selector |
| `app.session.resume` | Session | Open the session resume selector |

---

## 4. The Submit Handler: Slash Commands and Bash Mode

When the user presses Enter (or the configured submit key), `this.defaultEditor.onSubmit` fires with the current text. This is where most of the high-level dispatch happens:

```ts
// Simplified view of setupEditorSubmitHandler()
this.defaultEditor.onSubmit = async (text: string) => {
  text = text.trim();
  if (!text) return;

  // --- Slash commands ---
  if (text === "/settings")        { this.showSettingsSelector(); return; }
  if (text === "/model" || text.startsWith("/model ")) {
    const searchTerm = text.startsWith("/model ") ? text.slice(7).trim() : undefined;
    await this.handleModelCommand(searchTerm); return;
  }
  if (text === "/resume")          { this.showSessionSelector(); return; }
  if (text === "/fork")            { this.showUserMessageSelector(); return; }
  if (text === "/tree")            { this.showTreeSelector(); return; }
  if (text === "/hotkeys")         { this.handleHotkeysCommand(); return; }
  if (text === "/session")         { this.handleSessionCommand(); return; }
  if (text === "/changelog")       { this.handleChangelogCommand(); return; }
  if (text === "/compact" || text.startsWith("/compact ")) {
    const instructions = text.startsWith("/compact ") ? text.slice(9).trim() : undefined;
    await this.handleCompactCommand(instructions); return;
  }
  if (text === "/new")             { await this.handleClearCommand(); return; }
  if (text === "/reload")          { await this.handleReloadCommand(); return; }
  if (text === "/quit")            { await this.shutdown(); return; }
  // ... (export, import, share, copy, name, login/logout, debug, clone)

  // --- Bash mode ---
  if (text.startsWith("!")) {
    const isExcluded = text.startsWith("!!");  // !! = excluded from agent context
    const command = isExcluded ? text.slice(2).trim() : text.slice(1).trim();
    if (command) {
      await this.handleBashCommand(command, isExcluded);
      return;
    }
  }

  // --- Messages during compaction or streaming ---
  if (this.session.isCompacting) {
    this.queueCompactionMessage(text, "steer");
    return;
  }
  if (this.session.isStreaming) {
    await this.session.prompt(text, { streamingBehavior: "steer" });
    return;
  }

  // --- Normal message submission ---
  this.flushPendingBashComponents();
  if (this.onInputCallback) {
    this.onInputCallback(text);
  } else {
    this.pendingUserInputs.push(text);
  }
};
```

A few points worth noting:

- `!command` runs a bash command in the session's working directory. The output streams into a `BashExecutionComponent` shown in the chat.
- `!!command` does the same but marks the command as **excluded from context** — the agent will not see its output.
- When the agent is streaming (generating a response), new text is treated as a **steering message** that can redirect the current response, rather than being queued for later. The queue is exposed via `this.session.getSteeringMessages()`.
- During compaction, messages are queued locally in `this.compactionQueuedMessages` and dispatched after compaction finishes via `flushCompactionQueue()`.

---

## 5. Model Cycling and Thinking Level

Two common mid-session actions are cycling through models and adjusting the thinking level. Both update the editor's border colour so the user has a visual reminder of the current mode.

```ts
private cycleThinkingLevel(): void {
  const newLevel = this.session.cycleThinkingLevel();
  if (newLevel === undefined) {
    this.showStatus("Current model does not support thinking");
  } else {
    this.footer.invalidate();
    this.updateEditorBorderColor();
    this.showStatus(`Thinking level: ${newLevel}`);
  }
}

private async cycleModel(direction: "forward" | "backward"): Promise<void> {
  const result = await this.session.cycleModel(direction);
  if (result === undefined) {
    const msg = this.session.scopedModels.length > 0
      ? "Only one model in scope"
      : "Only one model available";
    this.showStatus(msg);
  } else {
    this.footer.invalidate();
    this.updateEditorBorderColor();
    const thinkingStr = result.model.reasoning && result.thinkingLevel !== "off"
      ? ` (thinking: ${result.thinkingLevel})`
      : "";
    this.showStatus(`Switched to ${result.model.name || result.model.id}${thinkingStr}`);
  }
}
```

`updateEditorBorderColor()` picks either the bash-mode border colour or the thinking-level border colour and applies it to the editor. The theme provides both via `theme.getBashModeBorderColor()` and `theme.getThinkingBorderColor(level)`.

---

## 6. Selectors: Replacing the Editor with an Overlay

Several actions (picking a model, choosing a session, adjusting settings) need to present a list UI. Rather than opening a separate window, the app swaps the **editor container** content with a selector component, then restores the editor when the user confirms or cancels.

This pattern is encapsulated in a private helper:

```ts
// Simplified view of showSelector()
private showSelector(
  create: (done: () => void) => { component: Component; focus: Component }
): void {
  const done = () => {
    this.editorContainer.clear();
    this.editorContainer.addChild(this.editor);
    this.ui.setFocus(this.editor);
  };
  const { component, focus } = create(done);
  this.editorContainer.clear();
  this.editorContainer.addChild(component);
  this.ui.setFocus(focus);
  this.ui.requestRender();
}
```

`done` is a callback the selector calls when it is finished. All the selector methods follow this pattern.

### 6.1 Model Selector

`showModelSelector()` opens a `ModelSelectorComponent` — a search-driven list of all available models:

```ts
private showModelSelector(initialSearchInput?: string): void {
  this.showSelector((done) => {
    const selector = new ModelSelectorComponent(
      this.ui,
      this.session.model,          // currently selected model
      this.settingsManager,
      this.session.modelRegistry,
      this.session.scopedModels,   // models in scope for this session
      async (model) => {
        await this.session.setModel(model);
        this.footer.invalidate();
        this.updateEditorBorderColor();
        done();
        this.showStatus(`Model: ${model.id}`);
      },
      () => { done(); this.ui.requestRender(); },
      initialSearchInput,
    );
    return { component: selector, focus: selector };
  });
}
```

`ModelSelectorComponent` (in `components/model-selector.ts`) holds a `searchInput` (an `Input` component) and a `listContainer`. It loads models from `modelRegistry.getAvailable()` asynchronously, and fuzzy-filters the list as the user types. A Tab key toggles between the "all" and "scoped" model lists. Arrow keys and Enter navigate and select.

The currently selected model is shown with a `✓` checkmark and sorted to the top.

### 6.2 Session Selector

`showSessionSelector()` opens a `SessionSelectorComponent`, which lists saved sessions so the user can resume one:

```ts
private showSessionSelector(): void {
  this.showSelector((done) => {
    const selector = new SessionSelectorComponent(
      (onProgress) => SessionManager.list(
        this.sessionManager.getCwd(),
        this.sessionManager.getSessionDir(),
        onProgress,
      ),
      (onProgress) => this.sessionManager.usesDefaultSessionDir()
        ? SessionManager.listAll(onProgress)
        : SessionManager.listAll(this.sessionManager.getSessionDir(), onProgress),
      async (sessionPath) => {
        done();
        await this.handleResumeSession(sessionPath);
      },
      () => { done(); this.ui.requestRender(); },
      () => { void this.shutdown(); },
      () => this.ui.requestRender(),
      {
        renameSession: async (sessionFilePath, nextName) => {
          const next = (nextName ?? "").trim();
          if (!next) return;
          const mgr = SessionManager.open(sessionFilePath);
          mgr.appendSessionInfo(next);
        },
        showRenameHint: true,
        keybindings: this.keybindings,
      },
      this.sessionManager.getSessionFile(),
    );
    return { component: selector, focus: selector };
  });
}
```

The `SessionSelectorComponent` shows sessions in either flat (recent/relevance) or threaded (parent→child) layout, with a search input for filtering. Inside it, a `SessionList` handles navigation and a `SessionSelectorHeader` renders scope/sort/filter state. Sessions can be deleted (with a confirmation step) and renamed.

Tab toggles scope between "Current Folder" and "All". The `app.session.toggleSort`, `app.session.toggleNamedFilter`, and `app.session.delete` bindings are handled inside the `SessionList` component.

### 6.3 Settings Selector

`showSettingsSelector()` opens a `SettingsSelectorComponent` that lets the user adjust runtime behaviour without leaving the session:

```ts
private showSettingsSelector(): void {
  this.showSelector((done) => {
    const selector = new SettingsSelectorComponent(
      {
        autoCompact: this.session.autoCompactionEnabled,
        showImages: this.settingsManager.getShowImages(),
        thinkingLevel: this.session.thinkingLevel,
        currentTheme: this.settingsManager.getTheme() || "dark",
        hideThinkingBlock: this.hideThinkingBlock,
        doubleEscapeAction: this.settingsManager.getDoubleEscapeAction(),
        // ... more settings
      },
      {
        onAutoCompactChange: (enabled) => {
          this.session.setAutoCompactionEnabled(enabled);
          this.footer.setAutoCompactEnabled(enabled);
        },
        onThemeChange: (themeName) => {
          setTheme(themeName, true);
          this.settingsManager.setTheme(themeName);
          this.ui.invalidate();
        },
        // ... more callbacks
        onCancel: () => { done(); this.ui.requestRender(); },
      },
    );
    return { component: selector, focus: selector.getSettingsList() };
  });
}
```

The `SettingsConfig` type (in `components/settings-selector.ts`) covers around two dozen settings including `autoCompact`, `showImages`, `transport`, `httpIdleTimeoutMs`, `thinkingLevel`, `currentTheme`, `hideThinkingBlock`, `doubleEscapeAction`, `quietStartup`, `clearOnShrink`, and `showTerminalProgress`.

### 6.4 Tree and Fork Selectors

`showTreeSelector()` opens a `TreeSelectorComponent` that shows the full session branch tree. Selecting a node calls `session.navigateTree(entryId, { summarize, customInstructions })`. If the user asks for a summary, a `Loader` is shown in the status area while the branch is summarised; pressing Escape aborts the summarisation.

`showUserMessageSelector()` opens a `UserMessageSelectorComponent` that lists all user messages in the current session as fork points. Selecting one calls `runtimeHost.fork(entryId)`, which creates a new branch at that message.

---

## 7. Extension Shortcuts

Extensions can register keyboard shortcuts via `extensionRunner.getShortcuts()`. After each session rebind, `setupExtensionShortcuts()` reads those shortcuts and installs a handler on the default editor:

```ts
// Simplified view of setupExtensionShortcuts()
this.defaultEditor.onExtensionShortcut = (data: string) => {
  for (const [shortcutStr, shortcut] of shortcuts) {
    if (matchesKey(data, shortcutStr as KeyId)) {
      // Run handler async, do not block input
      Promise.resolve(shortcut.handler(createContext())).catch((err) => {
        this.showError(`Shortcut handler error: ${err instanceof Error ? err.message : String(err)}`);
      });
      return true;
    }
  }
  return false;
};
```

`matchesKey` (imported from the `tui` package) compares the raw terminal byte sequence `data` against the binding string. If it matches, the extension's handler runs asynchronously so it never blocks the input loop. Any error from the handler is surfaced as a visible error in the chat.

Extension shortcuts are also displayed in the `/hotkeys` output under an **Extensions** section.

---

## 8. The Follow-Up Queue

When the agent is streaming, the user cannot submit a new "turn" — but they can **queue messages**. The `app.message.followUp` action collects the current editor text and calls `session.prompt(text, { streamingBehavior: "followUp" })`, which holds the message until the agent finishes:

```ts
private async handleFollowUp(): Promise<void> {
  const text = (this.editor.getExpandedText?.() ?? this.editor.getText()).trim();
  if (!text) return;

  if (this.session.isStreaming) {
    this.editor.addToHistory?.(text);
    this.editor.setText("");
    await this.session.prompt(text, { streamingBehavior: "followUp" });
    this.updatePendingMessagesDisplay();
    this.ui.requestRender();
  } else if (this.editor.onSubmit) {
    // Not streaming: acts like a regular submit
    this.editor.setText("");
    this.editor.onSubmit(text);
  }
}
```

Queued messages (both steering and follow-up) are shown in `this.pendingMessagesContainer` as truncated, dim text lines. The `app.message.dequeue` action calls `handleDequeue()`, which restores all queued messages back into the editor as a single combined text block.

---

## 9. Message Rendering Components

Now we turn from input to output. When `handleEvent()` receives events from the agent subscription, it creates components and appends them to `this.chatContainer`. Here is the component for each message role.

### 9.1 User Messages: `UserMessageComponent`

`UserMessageComponent` renders the text the user typed. It wraps the text in a `Box` with a themed background (`userMessageBg`) and passes it to a `Markdown` renderer with the `userMessageText` foreground colour:

```ts
// Simplified view of UserMessageComponent constructor
export class UserMessageComponent extends Container {
  constructor(text: string, markdownTheme: MarkdownTheme = getMarkdownTheme()) {
    super();
    this.contentBox = new Box(1, 1, (content) => theme.bg("userMessageBg", content));
    this.contentBox.addChild(
      new Markdown(text, 0, 0, markdownTheme, {
        color: (content) => theme.fg("userMessageText", content),
      }, { preserveOrderedListMarkers: true }),
    );
    this.addChild(this.contentBox);
  }
}
```

The component also wraps its rendered lines in OSC 133 shell integration zones (`\x1b]133;A\x07` … `\x1b]133;C\x07`) so terminals that support shell integration can highlight the prompt boundary.

### 9.2 Assistant Messages: `AssistantMessageComponent`

`AssistantMessageComponent` handles the more complex assistant output: a mix of text blocks, thinking blocks, and tool call placeholders.

```ts
// Simplified view of updateContent()
updateContent(message: AssistantMessage): void {
  this.lastMessage = message;
  this.contentContainer.clear();

  for (const content of message.content) {
    if (content.type === "text" && content.text.trim()) {
      this.contentContainer.addChild(
        new Markdown(content.text.trim(), 1, 0, this.markdownTheme)
      );
    } else if (content.type === "thinking" && content.thinking.trim()) {
      if (this.hideThinkingBlock) {
        // Show a static label instead of the full trace
        this.contentContainer.addChild(
          new Text(theme.italic(theme.fg("thinkingText", this.hiddenThinkingLabel)), 1, 0)
        );
      } else {
        // Show thinking trace in italic, thinkingText colour
        this.contentContainer.addChild(
          new Markdown(content.thinking.trim(), 1, 0, this.markdownTheme, {
            color: (t) => theme.fg("thinkingText", t),
            italic: true,
          })
        );
      }
    }
  }

  // If stopped with an error or abort, show an error line
  const hasToolCalls = message.content.some((c) => c.type === "toolCall");
  if (!hasToolCalls && message.stopReason === "aborted") {
    this.contentContainer.addChild(new Text(theme.fg("error", "Operation aborted"), 1, 0));
  }
}
```

The `hideThinkingBlock` flag comes from `settingsManager.getHideThinkingBlock()` and can be toggled live via the `app.thinking.toggle` action. When hidden, the `hiddenThinkingLabel` (default: `"Thinking..."`) is shown as a placeholder instead.

`setHideThinkingBlock(hide)` and `setHiddenThinkingLabel(label)` are both public methods that `InteractiveMode` calls when the setting or label changes; they trigger `updateContent(this.lastMessage)` so the displayed text updates immediately.

### 9.3 Tool Calls: `ToolExecutionComponent`

Each tool call in an assistant message gets its own `ToolExecutionComponent`. These are created as streaming progresses and updated when results arrive:

```ts
// From handleEvent, "message_update" case (simplified)
if (content.type === "toolCall") {
  if (!this.pendingTools.has(content.id)) {
    const component = new ToolExecutionComponent(
      content.name,          // tool name, e.g. "read_file"
      content.id,            // unique tool call id
      content.arguments,     // args JSON (may be partial while streaming)
      {
        showImages: this.settingsManager.getShowImages(),
        imageWidthCells: this.settingsManager.getImageWidthCells(),
      },
      this.getRegisteredToolDefinition(content.name), // custom renderer, if any
      this.ui,
      this.sessionManager.getCwd(),
    );
    component.setExpanded(this.toolOutputExpanded);
    this.chatContainer.addChild(component);
    this.pendingTools.set(content.id, component);
  } else {
    this.pendingTools.get(content.id)!.updateArgs(content.arguments);
  }
}
```

`ToolExecutionComponent` supports three rendering modes:

- **Custom renderer** — if the tool has a `renderCall` / `renderResult` function registered (via the extension tool definition API), those functions produce the display components.
- **Default renderer** — the component wraps the call renderer in a `Box` with a themed background (`toolPendingBg`, `toolSuccessBg`, or `toolErrorBg` depending on state).
- **Fallback** — if no renderer is defined at all, a plain `Text` component shows the tool name and prettified JSON arguments.

Calling `setExpanded(true)` (triggered by `app.tools.expand`) makes tool result renderers show their full content rather than a summary.

Images in tool results are rendered via `Image` components. On terminals that support the Kitty graphics protocol, non-PNG images are converted to PNG asynchronously before display.

### 9.4 The Diff Renderer

File-editing tools produce diffs. The `renderDiff(diffText)` function in `components/diff.ts` colourises a unified diff string for terminal display:

- **Context lines** are rendered in `toolDiffContext` colour (typically dim).
- **Removed lines** (`-`) are rendered in `toolDiffRemoved` colour (typically red).
- **Added lines** (`+`) are rendered in `toolDiffAdded` colour (typically green).
- When exactly one line is removed and one added (a single-line modification), **intra-line diff** is computed using word-level diffing (`Diff.diffWords`) and changed tokens are highlighted with inverse video.

```ts
// From renderDiff() — intra-line diff path
if (removedLines.length === 1 && addedLines.length === 1) {
  const { removedLine, addedLine } = renderIntraLineDiff(
    replaceTabs(removed.content),
    replaceTabs(added.content),
  );
  result.push(theme.fg("toolDiffRemoved", `-${removed.lineNum} ${removedLine}`));
  result.push(theme.fg("toolDiffAdded",   `+${added.lineNum} ${addedLine}`));
}
```

`renderIntraLineDiff` calls `Diff.diffWords` on the old and new content, applies `theme.inverse()` to changed tokens, and returns styled strings for both the removed and added lines.

### 9.5 Branch and Compaction Summaries

Two special message types appear when the session tree is navigated or the context is compacted.

**`BranchSummaryMessageComponent`** (role `"branchSummary"`) shows a collapsed `[branch]` label by default:

```ts
// Collapsed state (default)
"Branch summary (" + keyText("app.tools.expand") + " to expand)"

// Expanded state
"**Branch Summary**\n\n" + message.summary  // rendered as Markdown
```

**`CompactionSummaryMessageComponent`** (role `"compactionSummary"`) works the same way, collapsed to a label and expanded to the full summary Markdown. The collapsed label also shows how many tokens were compacted:

```ts
// Collapsed state
`Compacted from ${tokenStr} tokens (` + keyText("app.tools.expand") + " to expand)"

// Expanded state
`**Compacted from ${tokenStr} tokens**\n\n` + message.summary
```

Both components have a `setExpanded(boolean)` method, so the `app.tools.expand` action can toggle all expandable components at once.

---

## 10. The Footer: Persistent Status Bar

`FooterComponent` renders a two-line (sometimes three-line) status bar at the bottom of the TUI. It reads its data from two sources: the `AgentSession` (for token counts, model, thinking level, context usage) and a `ReadonlyFooterDataProvider` (for the git branch, extension statuses, and working-directory path).

```
// Example footer output (two lines)
~/projects/myapp (main) • my-session
↑24.3k ↓8.1k $0.142  87.3%/200k (auto)       claude-3-7-sonnet • medium
```

The left side of the stats line shows cumulative token counts (`↑` input, `↓` output, `R` cache-read, `W` cache-write) and total cost. The right side shows the current model ID and thinking level.

The context percentage is colour-coded: plain text below 70 %, `warning` colour above 70 %, and `error` colour above 90 %. When auto-compaction is enabled, `(auto)` is appended to the context display.

Extension statuses (set via `ui.setStatus(key, text)`) appear as a third line, sorted alphabetically by key.

The `invalidate()` method is a no-op — git branch invalidation is handled by a background watcher in `FooterDataProvider` that calls `ui.requestRender()` when the branch changes.

---

## 11. Putting It Together: A Full Keystroke Walk-Through

Let's trace the path of the `app.model.select` action (typically bound to a chord like Alt+M):

1. User presses the bound key combination.
2. The editor's `handleInput()` receives the raw terminal bytes and calls `this.actionHandlers.get("app.model.select")()`.
3. That handler calls `this.showModelSelector()`.
4. `showModelSelector()` calls `this.showSelector(create)`.
5. `showSelector` clears `this.editorContainer`, adds the `ModelSelectorComponent`, and sets TUI focus to it.
6. The user types a search term (e.g. "sonnet"). The selector fuzzy-filters the model list and updates its `listContainer`.
7. The user presses Enter. `ModelSelectorComponent.handleInput` matches `"tui.select.confirm"` and calls the `onSelect` callback.
8. The callback calls `session.setModel(model)`, invalidates the footer, updates the editor border colour, calls `done()` (which restores the editor), and emits a status message.
9. `ui.requestRender()` is called; the footer repaints with the new model name.

Every selector in the application follows this same cycle: clear editor container → mount selector → user navigates → callback fires → `done()` restores editor.

---

← Previous: [Interactive Mode: Startup, Wiring, and the TUI App Shell](./interactive-mode-startup-and-wiring.md) · Next: [Print Mode and RPC Mode](./print-and-rpc-modes.md) →
