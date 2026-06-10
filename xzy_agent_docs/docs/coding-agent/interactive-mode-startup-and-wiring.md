---
title: "Interactive Mode: Startup, Wiring, and the TUI App Shell"
description: How InteractiveMode initialises AgentSession, mounts the TUI, loads themes and extensions, registers slash commands, and enters the event loop (Part 1 of 2).
category: coding-agent
type: tutorial
tags: [interactive mode, TUI app, startup, wiring, AgentSession, extensions, coding-agent, application shell, initialization, event loop, slash commands, theme, InteractiveMode, rebindCurrentSession, subscribeToAgent, BUILTIN_SLASH_COMMANDS, initTheme, onThemeChange, ThemeJson, Theme, container layout]
keywords: [terminal UI startup, agent session wiring, TUI component tree, theme hot reload, fs watch, slash command registry, autocomplete provider, interactive agent loop, getUserInput, run method, init method]
sources: [S60, S62, S59]
---

**TL;DR** — Interactive mode is the conductor that wires every layer we have built — `AgentSession`, the TUI engine, the theme system, extensions, and slash commands — into a running terminal application. This chapter (Part 1 of 2) walks through the startup sequence: constructing the component tree, loading and hot-reloading themes, registering slash commands for autocomplete, and entering the event loop. Part 2 ([Interactive Mode: Input, Keyboard Shortcuts, and Session Commands](./interactive-mode-input-and-shortcuts.md)) covers keyboard handling and slash-command dispatch.

# Interactive Mode: Startup, Wiring, and the TUI App Shell

At this point in the series we have built all the ingredients separately:

- An **`AgentSession`** — the shared object that drives the underlying agent loop, owns settings, the model registry, extensions, and the resource loader (see [AgentSession: the shared core](./agent-session-core.md)).
- A **TUI render engine** — the `TUI` class that controls the terminal, composites `Component` children, and redraws on demand (see [The TUI Class and Render Engine](../terminal-ui/the-tui-class-and-render-engine.md)).
- A **system prompt and skill loader** — assembled at startup so the agent knows what tools and context are available (see [System Prompt and Skills](./system-prompt-and-skills.md)).
- A **model registry and settings loader** — which resolved the active model and all user preferences before we reach this point (see [Model Registry, Settings, and Resource Loading](./model-registry-and-settings.md)).

Now we need someone to take all of those and assemble a working program. That is exactly what `InteractiveMode` does.

---

## The problem: who wires it all together?

Every piece exists independently. `AgentSession` does not know what a terminal is; `TUI` does not know what an agent is. We need a coordinator that:

1. Takes an already-constructed `AgentSession` and mounts a TUI on top of it.
2. Loads the visual theme and watches the theme file for live-edit changes.
3. Registers all slash commands so the autocomplete dropdown is populated.
4. Subscribes to the session's event stream so every agent message, tool result, and status change gets rendered.
5. Runs the interactive prompt loop: ask the user for a message, feed it to the session, repeat.

`InteractiveMode` is that coordinator. Let's build it up one concern at a time.

---

## Step 1 — Construction: assembling the component tree

The constructor sets up the UI skeleton before any IO happens. Let's look at what gets created:

```ts
// Simplified view of InteractiveMode's constructor
constructor(runtimeHost: AgentSessionRuntime, options: InteractiveModeOptions = {}) {
  this.runtimeHost = runtimeHost;          // wraps AgentSession + lifecycle
  this.options = options;

  // Create the TUI instance connected to the real process terminal
  this.ui = new TUI(new ProcessTerminal(), this.settingsManager.getShowHardwareCursor());
  this.ui.setClearOnShrink(this.settingsManager.getClearOnShrink());

  // Build the slot containers that will hold rendered content
  this.headerContainer   = new Container();   // logo / keybinding hints
  this.chatContainer     = new Container();   // conversation history
  this.pendingMessagesContainer = new Container(); // messages queued while streaming
  this.statusContainer   = new Container();   // "Working…" spinner
  this.widgetContainerAbove = new Container(); // extension widgets above editor
  this.widgetContainerBelow = new Container(); // extension widgets below editor
  this.editorContainer   = new Container();   // the active editor component

  // Create the keybindings manager, then the default editor
  this.keybindings = KeybindingsManager.create();
  setKeybindings(this.keybindings);
  this.defaultEditor = new CustomEditor(
    this.ui,
    getEditorTheme(),
    this.keybindings,
    { paddingX: editorPaddingX, autocompleteMaxVisible }
  );
  this.editor = this.defaultEditor;
  this.editorContainer.addChild(this.editor as Component);

  // Footer (session name, git branch, context stats)
  this.footerDataProvider = new FooterDataProvider(this.sessionManager.getCwd());
  this.footer = new FooterComponent(this.session, this.footerDataProvider);

  // Load registered themes from resource loader, then apply the user's chosen theme
  setRegisteredThemes(this.session.resourceLoader.getThemes().themes);
  initTheme(this.settingsManager.getTheme(), true);
}
```

A few things to notice here:

- **`AgentSessionRuntime`** (`runtimeHost`) is the outer host that wraps `AgentSession` and manages its lifecycle — session creation, forking, and disposal. `InteractiveMode` talks to the session through convenience getters (`this.session`, `this.agent`, `this.sessionManager`, `this.settingsManager`) that delegate into `runtimeHost`.
- **`Container`** is a layout primitive from the `tui` package — it holds an ordered list of child `Component` objects that the TUI engine composites top-to-bottom during render.
- **`initTheme(name, enableWatcher: true)`** is called here in the constructor so colour is available even before `init()` runs. We will come back to what `initTheme` does in the theming section.

### `InteractiveModeOptions`

The constructor accepts an options bag that carries per-invocation context:

```ts
export interface InteractiveModeOptions {
  /** Providers that were migrated to auth.json (shows warning) */
  migratedProviders?: string[];
  /** Warning message if session model couldn't be restored */
  modelFallbackMessage?: string;
  /** Initial message to send on startup (can include @file content) */
  initialMessage?: string;
  /** Images to attach to the initial message */
  initialImages?: ImageContent[];
  /** Additional messages to send after the initial message */
  initialMessages?: string[];
  /** Force verbose startup (overrides quietStartup setting) */
  verbose?: boolean;
}
```

These are passed in by the CLI entry point before handing off to `InteractiveMode`.

---

## Step 2 — `init()`: mounting the TUI and wiring extensions

After construction the caller invokes `init()`. This is where the containers get attached to the TUI, extensions are bound, and the render loop starts.

```ts
async init(): Promise<void> {
  if (this.isInitialized) return;

  this.registerSignalHandlers();       // SIGTERM, SIGHUP, uncaughtException

  // Determine which changelog entries are new since last run
  this.changelogMarkdown = this.getChangelogForDisplay();

  // Download fd and rg if not already present (needed for autocomplete and grep)
  const [fdPath] = await Promise.all([ensureTool("fd"), ensureTool("rg")]);
  this.fdPath = fdPath;

  // ── Build the component tree ──────────────────────────────────────────────
  this.ui.addChild(this.headerContainer);       // header: logo + hints

  // Build the header (logo + compact/expanded keybinding hints)
  if (this.options.verbose || !this.settingsManager.getQuietStartup()) {
    const logo = /* ... */ APP_NAME + ` v${this.version}`;
    // ... keybinding hints ...
    this.builtInHeader = new ExpandableText(/* collapsed */, /* expanded */, ...);
    this.headerContainer.addChild(new Spacer(1));
    this.headerContainer.addChild(this.builtInHeader);
    this.headerContainer.addChild(new Spacer(1));
  }

  this.ui.addChild(this.chatContainer);
  this.ui.addChild(this.pendingMessagesContainer);
  this.ui.addChild(this.statusContainer);
  this.renderWidgets();                          // init with default spacer
  this.ui.addChild(this.widgetContainerAbove);
  this.ui.addChild(this.editorContainer);
  this.ui.addChild(this.widgetContainerBelow);
  this.ui.addChild(this.footer);
  this.ui.setFocus(this.editor);

  this.setupKeyHandlers();
  this.setupEditorSubmitHandler();

  // ── Start the UI first so extensions can show dialogs during session_start ─
  this.ui.start();
  this.isInitialized = true;

  // ── Bind extensions and set up autocomplete ───────────────────────────────
  await this.rebindCurrentSession();

  // ── Render any messages already in the session (resumed session case) ─────
  this.renderInitialMessages();

  // ── Register theme and git-branch watchers ────────────────────────────────
  onThemeChange(() => {
    this.ui.invalidate();
    this.updateEditorBorderColor();
    this.ui.requestRender();
  });
  this.footerDataProvider.onBranchChange(() => {
    this.ui.requestRender();
  });

  await this.updateAvailableProviderCount();
}
```

The ordering matters:

1. **`ui.start()`** comes *before* `rebindCurrentSession()`, because extensions may want to show a TUI dialog (a selector, a confirmation prompt) during their `session_start` handler. The TUI must already be running for that to work.
2. **`rebindCurrentSession()`** applies settings, binds extensions, calls `subscribeToAgent()` to connect the event stream, then shows the loaded-resources summary in the chat container.
3. **`renderInitialMessages()`** replays any messages already stored in the session — this handles the case where the user resumed a saved session that already has conversation history.
4. **`onThemeChange()`** registers a callback that fires whenever the theme file changes on disk; the callback invalidates the TUI so the new colours take effect immediately.

### The UI component order

The TUI renders its children top-to-bottom. After `init()` completes, the stack looks like this:

```
┌─────────────────────────────────┐
│  headerContainer                │  ← logo + keybinding hints
├─────────────────────────────────┤
│  chatContainer                  │  ← conversation messages + tool output
├─────────────────────────────────┤
│  pendingMessagesContainer       │  ← queued messages while streaming
├─────────────────────────────────┤
│  statusContainer                │  ← "Working…" spinner
├─────────────────────────────────┤
│  widgetContainerAbove           │  ← extension widgets above editor
├─────────────────────────────────┤
│  editorContainer                │  ← the active editor component
├─────────────────────────────────┤
│  widgetContainerBelow           │  ← extension widgets below editor
├─────────────────────────────────┤
│  footer                         │  ← session name, git branch, model, context
└─────────────────────────────────┘
```

---

## Step 3 — `rebindCurrentSession()`: wiring session, extensions, and autocomplete

We need a dedicated method to re-wire everything because the session can change at runtime (the user runs `/new`, `/resume`, or an extension calls `fork`). `rebindCurrentSession()` is the single place that re-applies all session-dependent wiring:

```ts
private async rebindCurrentSession(): Promise<void> {
  this.unsubscribe?.();         // detach any previous event subscription
  this.unsubscribe = undefined;

  this.applyRuntimeSettings(); // sync settings → UI (cursor, padding, timeouts)
  await this.bindCurrentSessionExtensions(); // run extension session_start + build autocomplete
  this.subscribeToAgent();     // re-attach event listener for new session
  await this.updateAvailableProviderCount();
  this.updateEditorBorderColor();
  this.updateTerminalTitle();
}
```

Let's look at what each call does.

### `applyRuntimeSettings()`

Pushes the current settings values into all the live UI objects:

```ts
private applyRuntimeSettings(): void {
  configureHttpDispatcher(this.settingsManager.getHttpIdleTimeoutMs());
  this.footer.setSession(this.session);
  this.footer.setAutoCompactEnabled(this.session.autoCompactionEnabled);
  this.footerDataProvider.setCwd(this.sessionManager.getCwd());
  this.hideThinkingBlock = this.settingsManager.getHideThinkingBlock();
  this.ui.setShowHardwareCursor(this.settingsManager.getShowHardwareCursor());
  this.ui.setClearOnShrink(this.settingsManager.getClearOnShrink());
  // ... editor padding, autocomplete max items ...
}
```

### `bindCurrentSessionExtensions()`

This method calls `this.session.bindExtensions(...)`, handing the session a UI context (`ExtensionUIContext`) that extensions can use to show selectors, inputs, custom editors, and widgets. After the extensions initialise, it rebuilds the autocomplete provider and shows the loaded-resources summary:

```ts
private async bindCurrentSessionExtensions(): Promise<void> {
  const uiContext = this.createExtensionUIContext(); // gives extensions access to TUI dialogs
  await this.session.bindExtensions({
    uiContext,
    mode: "tui",
    abortHandler: () => { /* ... */ },
    commandContextActions: { /* newSession, fork, navigateTree, ... */ },
    shutdownHandler: () => { this.shutdownRequested = true; /* ... */ },
    onError: (error) => { this.showExtensionError(/* ... */); },
  });

  setRegisteredThemes(this.session.resourceLoader.getThemes().themes);
  this.setupAutocompleteProvider();   // build autocomplete from slash commands + extensions + skills
  const extensionRunner = this.session.extensionRunner;
  this.setupExtensionShortcuts(extensionRunner);
  this.showLoadedResources({ force: false, showDiagnosticsWhenQuiet: true });
  this.showStartupNoticesIfNeeded();
}
```

`createExtensionUIContext()` returns an object that implements `ExtensionUIContext`. It exposes methods like `select`, `confirm`, `input`, `notify`, `setStatus`, `setWidget`, `setFooter`, `setHeader`, `setTheme`, and more — all of which delegate to private methods on `InteractiveMode`. Extensions call these methods and the TUI responds; they never get a reference to `InteractiveMode` itself.

### `subscribeToAgent()`

The session emits events — `agent_start`, `message_start`, `message_update`, `message_end`, `tool_execution_start`, `tool_execution_end`, `agent_end`, `compaction_start`, etc. `subscribeToAgent()` wires up the handler:

```ts
private subscribeToAgent(): void {
  this.unsubscribe = this.session.subscribe(async (event) => {
    await this.handleEvent(event);
  });
}
```

`handleEvent` is a large `switch` statement (covered in Part 2) that translates each event into a UI mutation — adding a message component, updating a streaming component, showing or hiding a loading spinner, and calling `this.ui.requestRender()` to schedule a redraw.

---

## Step 4 — Theming: `Theme`, `ThemeJson`, and hot reload

Before we get to the event loop, let's understand the theme system that the constructor already initialised.

### The `ThemeJson` shape

Every theme is a JSON file validated against a schema. The schema defines three groups of colour tokens:

| Group | Tokens (excerpt) | Used for |
|---|---|---|
| Core UI | `accent`, `border`, `borderAccent`, `borderMuted`, `success`, `error`, `warning`, `muted`, `dim`, `text`, `thinkingText` | General UI chrome |
| Backgrounds & text | `selectedBg`, `userMessageBg`, `toolPendingBg`, `toolSuccessBg`, `toolErrorBg`, `toolTitle`, `toolOutput` | Message / tool bubbles |
| Markdown | `mdHeading`, `mdLink`, `mdCode`, `mdCodeBlock`, `mdQuote`, `mdHr`, `mdListBullet` | Rendered markdown |
| Tool diffs | `toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext` | File-edit diffs |
| Syntax highlight | `syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation` | Code blocks |
| Thinking borders | `thinkingOff`, `thinkingMinimal`, `thinkingLow`, `thinkingMedium`, `thinkingHigh`, `thinkingXhigh` | Editor border colour by thinking level |
| Bash mode | `bashMode` | Editor border when `!` prefix active |

Each colour value is either a hex string (`"#ff0000"`), a 256-colour index (integer), an empty string (meaning "use the terminal's default foreground/background"), or a reference to a variable defined in the optional `vars` section of the theme file. Variable references are resolved recursively — if `vars.primary = "#5e81ac"` then a colour field can say `"primary"` and the loader will substitute the hex value.

```ts
// Simplified schema excerpt (from theme.ts)
const ThemeJsonSchema = Type.Object({
  name: Type.String(),
  vars: Type.Optional(Type.Record(Type.String(), ColorValueSchema)), // variable aliases
  colors: Type.Object({
    accent: ColorValueSchema,
    border: ColorValueSchema,
    // ... ~50 colour tokens ...
  }),
});
```

### The `Theme` class

`Theme` holds two maps of pre-computed ANSI escape sequences — one for foreground colours (`ThemeColor`) and one for backgrounds (`ThemeBg`). Its main method is `fg(color, text)`, which wraps `text` in the ANSI codes for the named colour and resets the foreground after:

```ts
fg(color: ThemeColor, text: string): string {
  const ansi = this.fgColors.get(color);
  if (!ansi) throw new Error(`Unknown theme color: ${color}`);
  return `${ansi}${text}\x1b[39m`; // \x1b[39m resets foreground to default
}
```

The `theme` export is a proxy that reads from a `globalThis` symbol at call time — not a captured reference. This means any module (including ones loaded by jiti or tsx) always sees the current theme even when the global is replaced by hot reload:

```ts
export const theme: Theme = new Proxy({} as Theme, {
  get(_target, prop) {
    const t = (globalThis as Record<symbol, Theme>)[THEME_KEY];
    if (!t) throw new Error("Theme not initialized. Call initTheme() first.");
    return (t as unknown as Record<string | symbol, unknown>)[prop];
  },
});
```

### `initTheme()` and two built-in themes

```ts
export function initTheme(themeName?: string, enableWatcher: boolean = false): void {
  const name = themeName ?? getDefaultTheme(); // detects dark/light from COLORFGBG env var
  currentThemeName = name;
  try {
    setGlobalTheme(loadTheme(name));
    if (enableWatcher) {
      startThemeWatcher();
    }
  } catch (_error) {
    // Fall back to dark theme silently if theme is invalid
    currentThemeName = "dark";
    setGlobalTheme(loadTheme("dark"));
  }
}
```

The two built-in themes are `dark.json` and `light.json`, loaded from the bundled themes directory. `getDefaultTheme()` inspects the `COLORFGBG` environment variable (which some terminals set to indicate background colour) to pick between them automatically. If that variable is absent, `dark` is the fallback.

Custom themes are `.json` files placed in `~/.xzy/themes/` (the custom themes directory). They follow the same `ThemeJson` schema and can reference the same variable system.

### Hot reload via `fs.watch`

When a custom theme is active, `startThemeWatcher()` opens an `fs.watch` on the custom themes directory. When any file change event fires for the current theme's `.json` file, it schedules a reload with a 100 ms debounce:

```ts
function startThemeWatcher(): void {
  // Only watches custom (non-built-in) themes
  if (!currentThemeName || currentThemeName === "dark" || currentThemeName === "light") {
    return;
  }

  const customThemesDir = getCustomThemesDir();
  const watchedFileName = `${currentThemeName}.json`;
  const themeFile = path.join(customThemesDir, watchedFileName);

  const scheduleReload = () => {
    if (themeReloadTimer) clearTimeout(themeReloadTimer);
    themeReloadTimer = setTimeout(() => {
      themeReloadTimer = undefined;
      if (!fs.existsSync(themeFile)) return; // file temporarily gone — keep last good theme
      try {
        const reloadedTheme = loadThemeFromPath(themeFile);
        registeredThemes.set(currentThemeName, reloadedTheme);
        setGlobalTheme(reloadedTheme);
        if (onThemeChangeCallback) onThemeChangeCallback(); // triggers ui.requestRender()
      } catch (_error) {
        // Ignore — file may be in an intermediate state while being edited
      }
    }, 100);
  };

  themeWatcher = watchWithErrorHandler(customThemesDir, (_eventType, filename) => {
    if (filename !== watchedFileName) return;
    scheduleReload();
  }, /* errorHandler */);
}
```

The 100 ms debounce means rapid saves (e.g. from an editor on every keypress) do not spam reloads. The `onThemeChangeCallback` registered in `init()` calls `this.ui.requestRender()` so the new colours paint immediately.

---

## Step 5 — Slash commands: the built-in registry

Before the user can type anything, `setupAutocompleteProvider()` needs to know which slash commands exist. The source of truth for built-in commands is `BUILTIN_SLASH_COMMANDS` in `slash-commands.ts`:

```ts
export interface BuiltinSlashCommand {
  name: string;
  description: string;
}

export const BUILTIN_SLASH_COMMANDS: ReadonlyArray<BuiltinSlashCommand> = [
  { name: "settings",      description: "Open settings menu" },
  { name: "model",         description: "Select model (opens selector UI)" },
  { name: "scoped-models", description: "Enable/disable models for Ctrl+P cycling" },
  { name: "export",        description: "Export session (HTML default, or specify path: .html/.jsonl)" },
  { name: "import",        description: "Import and resume a session from a JSONL file" },
  { name: "share",         description: "Share session as a secret GitHub gist" },
  { name: "copy",          description: "Copy last agent message to clipboard" },
  { name: "name",          description: "Set session display name" },
  { name: "session",       description: "Show session info and stats" },
  { name: "changelog",     description: "Show changelog entries" },
  { name: "hotkeys",       description: "Show all keyboard shortcuts" },
  { name: "fork",          description: "Create a new fork from a previous user message" },
  { name: "clone",         description: "Duplicate the current session at the current position" },
  { name: "tree",          description: "Navigate session tree (switch branches)" },
  { name: "login",         description: "Configure provider authentication" },
  { name: "logout",        description: "Remove provider authentication" },
  { name: "new",           description: "Start a new session" },
  { name: "compact",       description: "Manually compact the session context" },
  { name: "resume",        description: "Resume a different session" },
  { name: "reload",        description: "Reload keybindings, extensions, skills, prompts, and themes" },
  { name: "quit",          description: `Quit ${APP_NAME}` },
];
```

This array is the registration list that `setupAutocompleteProvider()` converts into `SlashCommand` objects for the autocomplete dropdown.

### `setupAutocompleteProvider()` — merging four sources

`createBaseAutocompleteProvider()` builds a `CombinedAutocompleteProvider` from four sources:

1. **Built-in slash commands** — from `BUILTIN_SLASH_COMMANDS` above. The `/model` command gets a special `getArgumentCompletions` function that fuzzy-filters available models by `provider/id`.
2. **Prompt templates** — prompt files loaded by the resource loader (e.g. `~/.xzy/prompts/*.md`) become additional slash commands, prefixed with a scope tag in their descriptions.
3. **Extension commands** — commands registered by loaded extensions, filtered to exclude any that conflict with built-in names (conflicts are reported as diagnostics, not silently dropped — the extension gets an alternate `invocationName`).
4. **Skill commands** — when the `enableSkillCommands` setting is on, each loaded skill becomes a `/skill:<name>` command.

```ts
private createBaseAutocompleteProvider(): AutocompleteProvider {
  // 1. Built-ins
  const slashCommands: SlashCommand[] = BUILTIN_SLASH_COMMANDS.map((command) => ({
    name: command.name,
    description: command.description,
  }));
  // Add argument completions for /model
  const modelCommand = slashCommands.find((c) => c.name === "model");
  if (modelCommand) {
    modelCommand.getArgumentCompletions = (prefix) => {
      const models = this.session.scopedModels.length > 0
        ? this.session.scopedModels.map((s) => s.model)
        : this.session.modelRegistry.getAvailable();
      const items = models.map((m) => ({ id: m.id, provider: m.provider, label: `${m.provider}/${m.id}` }));
      const filtered = fuzzyFilter(items, prefix, (item) => `${item.id} ${item.provider}`);
      return filtered.length === 0 ? null : filtered.map((item) => ({
        value: item.label, label: item.id, description: item.provider,
      }));
    };
  }

  // 2. Prompt templates (from resource loader)
  const templateCommands: SlashCommand[] = this.session.promptTemplates.map((cmd) => ({
    name: cmd.name,
    description: this.prefixAutocompleteDescription(cmd.description, cmd.sourceInfo),
    ...(cmd.argumentHint && { argumentHint: cmd.argumentHint }),
  }));

  // 3. Extension commands (excluding names that collide with built-ins)
  const builtinNames = new Set(slashCommands.map((c) => c.name));
  const extensionCommands: SlashCommand[] = this.session.extensionRunner
    .getRegisteredCommands()
    .filter((cmd) => !builtinNames.has(cmd.name))
    .map((cmd) => ({
      name: cmd.invocationName,
      description: this.prefixAutocompleteDescription(cmd.description, cmd.sourceInfo),
      getArgumentCompletions: cmd.getArgumentCompletions,
    }));

  // 4. Skill commands (if enabled in settings)
  this.skillCommands.clear();
  const skillCommandList: SlashCommand[] = [];
  if (this.settingsManager.getEnableSkillCommands()) {
    for (const skill of this.session.resourceLoader.getSkills().skills) {
      const commandName = `skill:${skill.name}`;
      this.skillCommands.set(commandName, skill.filePath);
      skillCommandList.push({
        name: commandName,
        description: this.prefixAutocompleteDescription(skill.description, skill.sourceInfo),
      });
    }
  }

  return new CombinedAutocompleteProvider(
    [...slashCommands, ...templateCommands, ...extensionCommands, ...skillCommandList],
    this.sessionManager.getCwd(),
    this.fdPath,  // fd binary path for file completions
  );
}
```

Extension-added autocomplete wrappers (from `addAutocompleteProvider` in the UI context) are then layered on top in `setupAutocompleteProvider()`:

```ts
private setupAutocompleteProvider(): void {
  let provider = this.createBaseAutocompleteProvider();
  for (const wrapProvider of this.autocompleteProviderWrappers) {
    provider = wrapProvider(provider);  // each extension wrapper decorates the previous provider
  }
  this.autocompleteProvider = provider;
  this.defaultEditor.setAutocompleteProvider(provider);
  if (this.editor !== this.defaultEditor) {
    this.editor.setAutocompleteProvider?.(provider); // also update a custom extension editor
  }
}
```

---

## Step 6 — The event loop: `run()` and `getUserInput()`

With the TUI mounted, extensions bound, and autocomplete wired, the caller invokes `run()`. This is the main entry point and the one that never returns until the user exits.

```ts
async run(): Promise<void> {
  await this.init();

  // Start background checks (version, package updates, tmux keyboard setup)
  checkForNewPiVersion(this.version).then((newRelease) => {
    if (newRelease) this.showNewVersionNotification(newRelease);
  });
  this.checkForPackageUpdates().then((updates) => {
    if (updates.length > 0) this.showPackageUpdateNotification(updates);
  });
  this.checkTmuxKeyboardSetup().then((warning) => {
    if (warning) this.showWarning(warning);
  });

  // Show startup warnings (migrated credentials, models.json errors, etc.)
  const { migratedProviders, modelFallbackMessage, initialMessage, initialImages, initialMessages } = this.options;
  if (migratedProviders && migratedProviders.length > 0) {
    this.showWarning(`Migrated credentials to auth.json: ${migratedProviders.join(", ")}`);
  }
  if (modelFallbackMessage) this.showWarning(modelFallbackMessage);

  // Send any initial messages passed in by the CLI (e.g. `xzy "do this"`)
  if (initialMessage) {
    try {
      await this.session.prompt(initialMessage, { images: initialImages });
    } catch (error: unknown) {
      this.showError(error instanceof Error ? error.message : "Unknown error occurred");
    }
  }
  if (initialMessages) {
    for (const message of initialMessages) {
      try {
        await this.session.prompt(message);
      } catch (error: unknown) {
        this.showError(error instanceof Error ? error.message : "Unknown error occurred");
      }
    }
  }

  // ── Main interactive loop ─────────────────────────────────────────────────
  while (true) {
    const userInput = await this.getUserInput();
    try {
      await this.session.prompt(userInput);
    } catch (error: unknown) {
      this.showError(error instanceof Error ? error.message : "Unknown error occurred");
    }
  }
}
```

The loop is deliberately simple: wait for user input, send it to the session, repeat. All the complexity of rendering, tool execution, streaming, and compaction lives in the event handler that `subscribeToAgent()` attached — not in the loop itself.

### `getUserInput()` — the bridge between the TUI and the loop

How does `run()`'s `await this.getUserInput()` get unblocked? The editor's submit handler and `getUserInput()` share a callback slot:

```ts
async getUserInput(): Promise<string> {
  // First drain any messages that were queued before we were ready
  const queuedInput = this.pendingUserInputs.shift();
  if (queuedInput !== undefined) {
    return queuedInput;
  }

  // Otherwise wait for the next editor submit
  return new Promise((resolve) => {
    this.onInputCallback = (text: string) => {
      this.onInputCallback = undefined;
      resolve(text);
    };
  });
}
```

And in `setupEditorSubmitHandler()`, when the user submits a normal (non-command) message:

```ts
// (in the submit handler, after slash-command and bash checks)
if (this.onInputCallback) {
  this.onInputCallback(text);          // resolves the getUserInput() promise
} else {
  this.pendingUserInputs.push(text);   // queued for the next getUserInput() call
}
this.editor.addToHistory?.(text);
```

So the flow is:

```
user presses Enter
  → editor.onSubmit fires
    → submit handler checks: is it a slash command? a bash command? streaming?
    → if normal: calls this.onInputCallback(text)
      → getUserInput() promise resolves
        → run()'s while-loop sends it to session.prompt()
          → agent runs, events flow through handleEvent()
            → components appear in chatContainer
              → ui.requestRender() repaints the terminal
```

### A complete startup sequence — summary diagram

```
CLI entry point
  │
  ├─ construct AgentSessionRuntime  (session, model registry, settings, resource loader)
  │
  └─ construct InteractiveMode(runtimeHost, options)
       │
       ├─ new TUI(ProcessTerminal)
       ├─ new Container × 7  (header, chat, pending, status, widgetAbove, editor, widgetBelow)
       ├─ new CustomEditor
       ├─ new FooterComponent
       ├─ setRegisteredThemes(...)   ← themes from resource loader
       └─ initTheme(name, true)     ← apply theme + start fs watcher
            │
            └─ run()
                 │
                 ├─ init()
                 │    ├─ registerSignalHandlers()
                 │    ├─ ensureTool("fd"), ensureTool("rg")
                 │    ├─ ui.addChild(headerContainer, chatContainer, ...)
                 │    ├─ ui.start()                ← render loop begins
                 │    ├─ rebindCurrentSession()
                 │    │    ├─ applyRuntimeSettings()
                 │    │    ├─ bindCurrentSessionExtensions()
                 │    │    │    ├─ session.bindExtensions(uiContext)
                 │    │    │    ├─ setRegisteredThemes(...)
                 │    │    │    └─ setupAutocompleteProvider()
                 │    │    ├─ subscribeToAgent()   ← event stream connected
                 │    │    └─ updateTerminalTitle()
                 │    ├─ renderInitialMessages()   ← replay saved session
                 │    └─ onThemeChange(callback)   ← hot-reload hook
                 │
                 ├─ [async background: version check, package updates, tmux check]
                 │
                 ├─ [show startup warnings from options]
                 │
                 ├─ [send initialMessage / initialMessages if provided]
                 │
                 └─ while (true)
                      ├─ getUserInput()            ← awaits editor submit
                      └─ session.prompt(userInput) ← agent runs; events → handleEvent()
```

---

## What we've covered

We walked through every step of interactive mode's startup:

- **Construction**: the TUI and all slot containers are created, the theme is applied.
- **`init()`**: the component tree is assembled, the UI starts, extensions are bound, the event subscription goes live, initial messages are replayed.
- **`rebindCurrentSession()`**: the single method that re-applies all session-dependent wiring when the session changes.
- **Theming**: `ThemeJson` colour tokens, the `Theme` proxy singleton, dark/light auto-detection, and 100 ms debounced hot reload via `fs.watch`.
- **Slash commands**: the `BUILTIN_SLASH_COMMANDS` registry, how it merges with prompt templates, extension commands, and skill commands into a single `CombinedAutocompleteProvider`.
- **The event loop**: the `run()` method, the `getUserInput()` / `onInputCallback` bridge, and how user input flows from the editor to the agent and back to the UI.

Part 2 picks up from here and digs into how submitted text is dispatched — slash-command routing, bash mode, streaming behaviour, keyboard shortcuts, and session commands.

---

← Previous: [Model Registry, Settings, and Resource Loading](./model-registry-and-settings.md) · Next: [Interactive Mode: Input, Keyboard Shortcuts, and Session Commands](./interactive-mode-input-and-shortcuts.md) →
