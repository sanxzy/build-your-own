---
title: "Loading and Running Extensions: Discovery, Isolation, and Lifecycle"
description: "How the agent discovers TypeScript extension files at runtime, isolates failures, and wires loaded extensions into a live session through ExtensionRunner."
category: extensions
type: tutorial
tags:
  - extension loader
  - jiti
  - TypeScript runtime
  - multi-path discovery
  - error isolation
  - ExtensionRunner
  - bind to session
  - event dispatch
  - lifecycle
  - reload
  - extensions
  - coding-agent
  - discoverAndLoadExtensions
  - loadExtensions
  - loadExtensionModule
  - createExtensionRuntime
  - createExtensionAPI
  - bindCore
  - bindCommandContext
  - setUIContext
  - emit
  - emitToolCall
  - emitInput
  - ExtensionFactory
  - ExtensionRuntime
  - ExtensionContext
  - test harness
  - faux provider
keywords:
  - runtime TypeScript loading
  - extension hot reload
  - extension discovery
  - virtual modules
  - jiti static
  - extension isolation
  - AgentSession extensions
  - extension error handling
  - coding agent plugin system
sources: [S49, S50, S85]
---

**TL;DR** — Extensions are plain TypeScript files that users drop into well-known directories. This chapter follows the path from file discovery through runtime loading (using jiti so no build step is needed), per-extension error isolation, and finally the `ExtensionRunner` that binds loaded extensions to a live agent session and routes every lifecycle event to the right handlers. By the end you will understand how the entire extension pipeline works and how to test it with the faux-provider harness.

# Loading and Running Extensions: Discovery, Isolation, and Lifecycle

In the [previous chapter](./extension-api-and-types.md) we looked at the `ExtensionAPI` — the object an extension author receives and uses to register handlers, tools, commands, and flags. That told us *what* extensions can do. Now we need to answer a harder question: how does the agent actually get those extension files into memory, and how does it connect them to a running session?

Three problems have to be solved in sequence:

1. **Loading** — extensions are TypeScript files. The agent must execute them without a separate compile step.
2. **Discovery** — extensions can live in several places (project-local, global, explicitly configured). We need a single pass that finds all of them without loading the same file twice.
3. **Lifecycle** — once loaded, extensions must receive every agent event (tool calls, context updates, session changes) as they happen, in a thread-safe-enough way, and a single bad extension must not crash the agent.

Let's work through each layer.

---

## The loading problem: TypeScript files without a build step

When a user writes an extension, they write a `.ts` file — raw TypeScript. The agent is a compiled binary (or a Node.js process), so it cannot `import()` a `.ts` file the way it imports its own modules. We need a tool that reads TypeScript source, transpiles it on the fly, and returns the resulting module.

The agent uses **jiti** for this. jiti is a TypeScript/ESM-compatible runtime `import` layer: you give it a path to a `.ts` file and it transpiles and executes it, returning whatever the module exports.

### `loadExtensionModule` — one file, one jiti call

The smallest building block is `loadExtensionModule`. It accepts the resolved absolute path to a `.ts` or `.js` file and returns the default export — which must be an `ExtensionFactory` (a function that takes the `ExtensionAPI` and sets up the extension):

```ts
// Simplified view of loadExtensionModule (from loader.ts)
async function loadExtensionModule(extensionPath: string) {
  const jiti = createJiti(import.meta.url, {
    moduleCache: false,
    // In compiled Bun binary: virtualModules provides bundled packages
    // In Node.js / development: alias maps package names to node_modules paths
    ...(isBunBinary
      ? { virtualModules: VIRTUAL_MODULES, tryNative: false }
      : { alias: getAliases() }),
  });

  const module = await jiti.import(extensionPath, { default: true });
  const factory = module as ExtensionFactory;
  return typeof factory !== "function" ? undefined : factory;
}
```

A few things are worth noticing here.

**`moduleCache: false`** tells jiti to re-read the file every time. That is what makes hot reload work: we call `loadExtensionModule` again after a reload request and get the new version.

**`virtualModules` vs `alias`** — this is a deployment seam. When the agent runs as a compiled Bun binary, the package files (`agent-core`, `llm-toolkit`, etc.) are bundled *into* the binary; there is no filesystem location for jiti to resolve them. `virtualModules` makes jiti resolve those import specifiers directly to the already-loaded in-memory modules. In development (Node.js), the packages live in `node_modules` and `alias` maps the specifier strings to the correct paths on disk. Either way, an extension author writes `import { ... } from "agent-core"` and it works.

If the module's default export is not a function, `loadExtensionModule` returns `undefined`. The caller treats that as a load failure.

### `ExtensionRuntime` — the shared action bus

Before we can call the factory, we need an `ExtensionRuntime`. Think of `ExtensionRuntime` as the internal side of the `ExtensionAPI`: it holds the actual function references that the API methods delegate to. Every extension loaded in one batch shares a single runtime object.

```ts
// Simplified view of createExtensionRuntime (from loader.ts)
export function createExtensionRuntime(): ExtensionRuntime {
  const notInitialized = () => {
    throw new Error(
      "Extension runtime not initialized. Action methods cannot be called during extension loading."
    );
  };
  const state: { staleMessage?: string } = {};

  return {
    // Action stubs — replaced by bindCore() once the session exists
    sendMessage: notInitialized,
    sendUserMessage: notInitialized,
    appendEntry: notInitialized,
    setSessionName: notInitialized,
    getSessionName: notInitialized,
    setLabel: notInitialized,
    getActiveTools: notInitialized,
    getAllTools: notInitialized,
    setActiveTools: notInitialized,
    refreshTools: () => {},   // safe no-op during load
    getCommands: notInitialized,
    setModel: () => Promise.reject(new Error("Extension runtime not initialized")),
    getThinkingLevel: notInitialized,
    setThinkingLevel: notInitialized,

    // Staleness tracking
    assertActive: () => {
      if (state.staleMessage) throw new Error(state.staleMessage);
    },
    invalidate: (message) => { state.staleMessage ??= message ?? "This ctx is stale..."; },

    // Provider registration — queued pre-bind, direct post-bind
    flagValues: new Map(),
    pendingProviderRegistrations: [],
    registerProvider: (name, config, extensionPath) => {
      runtime.pendingProviderRegistrations.push({ name, config, extensionPath });
    },
    unregisterProvider: (name) => {
      runtime.pendingProviderRegistrations =
        runtime.pendingProviderRegistrations.filter((r) => r.name !== name);
    },
  };
}
```

The key insight: action methods like `sendMessage` start as throwing stubs. An extension author cannot call `ctx.sendMessage()` *during* the factory call (i.e., at registration time) — only inside event handlers and tool callbacks that fire later. This gives us a clean separation between "setup" and "run".

Provider registrations made during setup are queued in `pendingProviderRegistrations`. They get flushed when the runtime is bound to a live session (in `bindCore`, covered later).

### `createExtensionAPI` — connecting the API to the runtime and extension object

Each loaded extension also gets its own `ExtensionAPI` instance. The API's registration methods (`on`, `registerTool`, `registerCommand`, …) write into the extension's own data structures. The action methods (`sendMessage`, `exec`, …) delegate to the shared runtime. The `events` field is the extension's private `EventBus`.

```ts
// Simplified view of createExtensionAPI (from loader.ts)
function createExtensionAPI(
  extension: Extension,
  runtime: ExtensionRuntime,
  cwd: string,
  eventBus: EventBus,
): ExtensionAPI {
  return {
    // Registration — writes into `extension`
    on(event, handler) {
      runtime.assertActive();
      const list = extension.handlers.get(event) ?? [];
      list.push(handler);
      extension.handlers.set(event, list);
    },
    registerTool(tool) {
      runtime.assertActive();
      extension.tools.set(tool.name, { definition: tool, sourceInfo: extension.sourceInfo });
      runtime.refreshTools();
    },
    // ... registerCommand, registerShortcut, registerFlag, registerMessageRenderer ...

    // Action — delegates to shared runtime
    sendMessage(message, options) {
      runtime.assertActive();
      runtime.sendMessage(message, options);
    },
    // ... other actions ...

    events: eventBus,
  } as ExtensionAPI;
}
```

### `loadExtension` — one file, full error isolation

Now we can put the pieces together. `loadExtension` loads one file and wraps everything in a try/catch:

```ts
// Simplified view of loadExtension (from loader.ts)
async function loadExtension(
  extensionPath: string,
  cwd: string,
  eventBus: EventBus,
  runtime: ExtensionRuntime,
): Promise<{ extension: Extension | null; error: string | null }> {
  const resolvedPath = resolvePath(extensionPath, cwd, { normalizeUnicodeSpaces: true });

  try {
    const factory = await loadExtensionModule(resolvedPath);
    if (!factory) {
      return {
        extension: null,
        error: `Extension does not export a valid factory function: ${extensionPath}`,
      };
    }

    const extension = createExtension(extensionPath, resolvedPath);
    const api = createExtensionAPI(extension, runtime, cwd, eventBus);
    await factory(api);  // run the extension's setup code

    return { extension, error: null };
  } catch (err) {
    const message = err instanceof Error ? err.message : String(err);
    return { extension: null, error: `Failed to load extension: ${message}` };
  }
}
```

If `factory(api)` throws — a syntax error, a missing import, a runtime exception in the setup code — the error is captured and returned as a string. The calling code records the error and moves on. This is the **per-extension error isolation** guarantee: one broken extension does not prevent the others from loading.

---

## Discovery: finding extensions in the right places

The agent looks for extensions in three locations, in this order:

| Priority | Location | Description |
|----------|----------|-------------|
| 1 | `<project>/.xzy/extensions/` | Project-local extensions; committed alongside the code they extend |
| 2 | `~/.xzy/extensions/` | Global extensions; active in every project |
| 3 | Explicitly configured paths | Paths from the settings file |

Duplicates (the same resolved file path appearing in more than one location) are silently deduplicated — each file is loaded at most once.

### `discoverExtensionsInDir` — what counts as an extension

Within any of the three directories above, the discovery logic applies a small set of rules (no deep recursion):

```
extensions/
  my-tool.ts          ← direct file: loaded directly
  permission-gate.ts  ← direct file: loaded directly
  subagent/
    index.ts          ← subdirectory with index: loaded
  custom-provider/
    package.json      ← subdirectory with package.json + "xzy.extensions" field: loads declared paths
    index.ts
```

Concretely, for each entry in the directory:

- If it is a `.ts` or `.js` file — load it directly.
- If it is a subdirectory — look for `package.json` with an `xzy.extensions` array first; if found, load the declared paths. Otherwise fall back to `index.ts` or `index.js`.
- No recursion beyond one level. Complex multi-file extensions use `package.json` to declare their entry points.

The function `resolveExtensionEntries` handles the subdirectory logic:

```ts
// Simplified view of resolveExtensionEntries (from loader.ts)
function resolveExtensionEntries(dir: string): string[] | null {
  const packageJsonPath = path.join(dir, "package.json");
  if (fs.existsSync(packageJsonPath)) {
    const manifest = readPackageManifest(packageJsonPath); // reads the "xzy" field
    if (manifest?.extensions?.length) {
      const entries: string[] = [];
      for (const extPath of manifest.extensions) {
        const resolved = path.resolve(dir, extPath);
        if (fs.existsSync(resolved)) entries.push(resolved);
      }
      if (entries.length > 0) return entries;
    }
  }

  const indexTs = path.join(dir, "index.ts");
  const indexJs = path.join(dir, "index.js");
  if (fs.existsSync(indexTs)) return [indexTs];
  if (fs.existsSync(indexJs)) return [indexJs];
  return null;
}
```

### `loadExtensions` — from a list of paths to a result

Once all paths are collected, `loadExtensions` iterates them, loading each one with the shared runtime and accumulating both successes and failures:

```ts
// Simplified view of loadExtensions (from loader.ts)
export async function loadExtensions(
  paths: string[],
  cwd: string,
  eventBus?: EventBus,
): Promise<LoadExtensionsResult> {
  const extensions: Extension[] = [];
  const errors: Array<{ path: string; error: string }> = [];
  const resolvedCwd = resolvePath(cwd);
  const resolvedEventBus = eventBus ?? createEventBus();
  const runtime = createExtensionRuntime();  // one shared runtime for all

  for (const extPath of paths) {
    const { extension, error } = await loadExtension(
      extPath, resolvedCwd, resolvedEventBus, runtime,
    );

    if (error) {
      errors.push({ path: extPath, error });
      continue;  // keep going — one failure does not stop the rest
    }

    if (extension) extensions.push(extension);
  }

  return { extensions, errors, runtime };
}
```

The returned `LoadExtensionsResult` carries three things: the successfully loaded `Extension[]` objects, the `errors` array (one entry per failed file), and the shared `runtime` (needed by `ExtensionRunner.bindCore` later).

### `discoverAndLoadExtensions` — the full pipeline

In normal operation the agent calls `discoverAndLoadExtensions`, which combines the three-location scan with deduplication and then calls `loadExtensions`:

```ts
// Simplified view of discoverAndLoadExtensions (from loader.ts)
export async function discoverAndLoadExtensions(
  configuredPaths: string[],
  cwd: string,
  agentDir: string = getAgentDir(),  // defaults to ~/.xzy/
  eventBus?: EventBus,
): Promise<LoadExtensionsResult> {
  const resolvedCwd = resolvePath(cwd);
  const resolvedAgentDir = resolvePath(agentDir);
  const allPaths: string[] = [];
  const seen = new Set<string>();

  const addPaths = (paths: string[]) => {
    for (const p of paths) {
      const resolved = path.resolve(p);
      if (!seen.has(resolved)) { seen.add(resolved); allPaths.push(p); }
    }
  };

  // 1. Project-local: <cwd>/.xzy/extensions/
  addPaths(discoverExtensionsInDir(path.join(resolvedCwd, ".xzy", "extensions")));

  // 2. Global: ~/.xzy/extensions/
  addPaths(discoverExtensionsInDir(path.join(resolvedAgentDir, "extensions")));

  // 3. Explicitly configured paths
  for (const p of configuredPaths) {
    const resolved = resolvePath(p, resolvedCwd, { normalizeUnicodeSpaces: true });
    if (fs.existsSync(resolved) && fs.statSync(resolved).isDirectory()) {
      const entries = resolveExtensionEntries(resolved);
      if (entries) { addPaths(entries); continue; }
      addPaths(discoverExtensionsInDir(resolved));
      continue;
    }
    addPaths([resolved]);
  }

  return loadExtensions(allPaths, resolvedCwd, eventBus);
}
```

After this call, we have a list of `Extension` objects and a shared `ExtensionRuntime`. The extensions know what events to handle, what tools to expose, what commands to register. But their action methods still throw — they are not yet connected to a live session.

---

## The ExtensionRunner: binding to a session and dispatching events

`ExtensionRunner` is the bridge between the loaded extension data and the live agent session. You construct it with the loaded extensions and runtime, then call several bind methods to wire in the session's capabilities. After that, every agent event goes through the runner, which fans it out to the registered handlers across all extensions.

### Construction

```ts
// Simplified view of ExtensionRunner constructor (from runner.ts)
const runner = new ExtensionRunner(
  extensions,   // Extension[] from loadExtensions
  runtime,      // ExtensionRuntime from loadExtensions
  cwd,          // current working directory
  sessionManager,
  modelRegistry,
);
```

At this point the runner holds the extensions and the inert runtime. No action methods work yet.

### `bindCore` — connecting action methods to the live session

`bindCore` is the critical step that makes action methods functional. The caller passes in `ExtensionActions` (functions from the live `AgentSession`) and `ExtensionContextActions` (functions that read session state):

```ts
// Simplified view of bindCore (from runner.ts)
runner.bindCore(
  {
    sendMessage,       // from AgentSession
    sendUserMessage,
    appendEntry,
    setSessionName,
    getSessionName,
    setLabel,
    getActiveTools,
    getAllTools,
    setActiveTools,
    refreshTools,
    getCommands,
    setModel,
    getThinkingLevel,
    setThinkingLevel,
  },
  {
    getModel,          // () => current Model
    isIdle,
    getSignal,
    abort,
    hasPendingMessages,
    shutdown,
    getContextUsage,
    compact,
    getSystemPrompt,
    getSystemPromptOptions,
  },
);
```

Inside `bindCore`, every stub in the shared runtime is replaced with the real function. From this point on, any extension calling `ctx.sendMessage(...)` in a handler reaches directly into the live session.

`bindCore` also flushes `pendingProviderRegistrations` — providers that were registered during extension setup are now registered with the real model registry.

### `bindCommandContext` — adding session-navigation actions

Commands (slash-command handlers) need a richer context: the ability to wait for the agent to be idle, create a new session, fork a branch, navigate the session tree, switch sessions, or trigger a reload. These are set up separately with `bindCommandContext`:

```ts
// Simplified view of bindCommandContext (from runner.ts)
runner.bindCommandContext({
  waitForIdle,
  newSession,
  fork,
  navigateTree,
  switchSession,
  reload,
});
```

If you are setting up a headless / non-interactive mode, you can call `bindCommandContext()` with no argument; it installs safe no-ops for all navigation actions.

### `setUIContext` — connecting the terminal UI

In interactive mode, the runner also receives a `ExtensionUIContext` — the object that lets extensions show confirmation dialogs, set status text, add autocomplete providers, and so on. In non-interactive (print) mode, a `noOpUIContext` is installed automatically:

```ts
runner.setUIContext(uiContext, "interactive");
// or, implicitly, noOpUIContext is already the default
```

The runner's `hasUI()` method returns `true` only when a real UI context has been provided.

---

## Event dispatch: how the runner fans out events

Once bound, the runner sits in the event path of the agent session. When the session fires an event — a tool call, a completed message, a context update — it calls the corresponding method on the runner.

### The generic `emit` path

Most events go through `emit<TEvent>`. The runner walks every loaded extension, finds all handlers registered for that event type, and calls each one with the event and a fresh `ExtensionContext`. Errors from individual handlers are caught and forwarded to error listeners — they do not propagate back to the agent:

```ts
// Simplified view of emit (from runner.ts)
async emit<TEvent extends RunnerEmitEvent>(event: TEvent): Promise<RunnerEmitResult<TEvent>> {
  const ctx = this.createContext();
  let result: SessionBeforeEventResult | undefined;

  for (const ext of this.extensions) {
    const handlers = ext.handlers.get(event.type);
    if (!handlers || handlers.length === 0) continue;

    for (const handler of handlers) {
      try {
        const handlerResult = await handler(event, ctx);

        // For "session_before_*" events: if the handler cancels, short-circuit immediately
        if (this.isSessionBeforeEvent(event) && handlerResult) {
          result = handlerResult as SessionBeforeEventResult;
          if (result.cancel) return result as RunnerEmitResult<TEvent>;
        }
      } catch (err) {
        this.emitError({ extensionPath: ext.path, event: event.type, error: ... });
      }
    }
  }

  return result as RunnerEmitResult<TEvent>;
}
```

The `session_before_*` events (`session_before_switch`, `session_before_fork`, `session_before_compact`, `session_before_tree`) are special: any handler can return `{ cancel: true }` to abort the operation, and the runner stops calling further handlers immediately.

### Specialised emit methods

Some events have dedicated emit methods because their result shapes are unusual:

| Method | What it does |
|--------|-------------|
| `emitToolCall(event)` | Fans out to `tool_call` handlers; any handler can block the call by returning `{ block: true }` |
| `emitToolResult(event)` | Fans out to `tool_result` handlers; handlers can replace `content`, `details`, or `isError` |
| `emitMessageEnd(event)` | Each handler may return a modified `AgentMessage`; handlers chain — each sees the output of the previous |
| `emitContext(messages)` | Each handler may return a new messages array; chains similarly; starts from a deep-clone |
| `emitBeforeProviderRequest(payload)` | Chains the raw provider request payload through `before_provider_request` handlers |
| `emitBeforeAgentStart(...)` | Fans out `before_agent_start`; collects prepend-messages + systemPrompt overrides from all handlers |
| `emitUserBash(event)` | First handler to return a result short-circuits; used to intercept user shell commands |
| `emitInput(text, images, source, ...)` | Chains `input` handlers; first `{ action: "handled" }` short-circuits |
| `emitResourcesDiscover(cwd, reason)` | Collects `skillPaths`, `promptPaths`, `themePaths` from all handlers |

You might notice the "transform chain" pattern used in `emitMessageEnd`, `emitContext`, `emitBeforeProviderRequest`, and `emitInput`: rather than passing the original value to every handler independently, each handler receives the *output* of the previous one. This lets extensions cooperate — one extension adds context window metadata to a message, and a downstream extension sees the enriched version.

### `createContext` — the ExtensionContext visible to handlers

Every call to `emit*` creates a fresh `ExtensionContext` via `createContext()`. The context uses lazy getter properties so that values like `model`, `cwd`, and `ui` are read at handler invocation time, not at emit time. This means a handler always sees the current session state, even if `bindCore` was called in between.

```ts
// Simplified view of createContext (from runner.ts)
createContext(): ExtensionContext {
  const runner = this;
  return {
    get ui() { runner.assertActive(); return runner.uiContext; },
    get mode() { runner.assertActive(); return runner.mode; },
    get cwd() { runner.assertActive(); return runner.cwd; },
    get sessionManager() { runner.assertActive(); return runner.sessionManager; },
    get model() { runner.assertActive(); return runner.getModel(); },
    isIdle: () => { runner.assertActive(); return runner.isIdleFn(); },
    abort: () => { runner.assertActive(); runner.abortFn(); },
    getContextUsage: () => { runner.assertActive(); return runner.getContextUsageFn(); },
    compact: (options) => { runner.assertActive(); runner.compactFn(options); },
    getSystemPrompt: () => { runner.assertActive(); return runner.getSystemPromptFn(); },
    // ...
  };
}
```

### Staleness and invalidation

When a session is replaced (new session, fork, switch, reload), the runner can be marked stale by calling `runner.invalidate(message)`. After that, `assertActive()` throws with a descriptive message. This prevents extensions from inadvertently using a captured context object that refers to a session that no longer exists.

### Shortcut conflict resolution

Extensions can register keyboard shortcuts. Before the shortcuts are used, the runner resolves conflicts via `getShortcuts(resolvedKeybindings)`. It has two tiers of conflict:

- **Reserved keybindings** (e.g. `app.interrupt`, `app.exit`, `tui.input.submit`) — extension shortcuts that collide with these are silently dropped, and a warning is emitted.
- **Non-reserved keybindings** — if an extension shortcut collides with a built-in but non-reserved binding, a warning is emitted and the extension's shortcut takes precedence.
- **Extension–extension conflicts** — last registration wins; a warning is emitted.

Warnings are collected in `shortcutDiagnostics` (accessible via `getShortcutDiagnostics()`).

---

## Testing extensions with the faux-provider harness

You do not need a real LLM to test an extension. The test harness (`test/suite/harness.ts`) wires up a complete stack — a faux provider that responds from a script, a real `AgentSession`, and optional extension factories — all in a temporary directory that is cleaned up after the test.

A quick recap of the faux provider concept (covered fully in [the provider chapter](../llm-toolkit/provider-adapters-google-and-faux.md)): a faux provider is a controllable stub that accepts a list of pre-scripted response steps and replays them in order, one per agent turn. It lets tests control exactly what the model "says" without any network call.

### `createHarness` — the entry point

```ts
// From test/suite/harness.ts
const harness = await createHarness({
  // Pre-scripted model responses (optional — defaults to empty list)
  // models: [{ id: "faux-model", responses: [...] }],

  // Inject extension factories directly
  extensionFactories: [myExtensionFactory],

  // Override the system prompt
  systemPrompt: "You are a test assistant.",
});
```

`createHarness` accepts a `HarnessOptions` object:

| Option | Type | Description |
|--------|------|-------------|
| `models` | `FauxModelDefinition[]` | Pre-scripted model definitions for the faux provider |
| `settings` | `Partial<Settings>` | Override agent settings |
| `systemPrompt` | `string` | System prompt (defaults to `"You are a test assistant."`) |
| `tools` | `AgentTool[]` | Override the built-in tool set |
| `initialActiveToolNames` | `string[]` | Active tools at session start |
| `allowedToolNames` | `string[]` | Tool allowlist |
| `excludedToolNames` | `string[]` | Tool denylist |
| `resourceLoader` | `ResourceLoader` | Custom resource loader |
| `extensionFactories` | `Array<ExtensionFactory \| CreateTestExtensionsResultInput>` | Extension factories to load into the session |
| `withConfiguredAuth` | `boolean` | Whether to pre-configure auth (defaults to `true`) |

The function returns a `Harness` object:

```ts
export interface Harness {
  session: AgentSession;           // the live session under test
  sessionManager: SessionManager;
  settingsManager: SettingsManager;
  authStorage: AuthStorage;
  faux: FauxProviderRegistration;  // the faux provider — use to set scripted responses
  models: [Model<string>, ...Model<string>[]];
  getModel: (modelId?: string) => Model<string> | undefined;
  setResponses: (responses: FauxResponseStep[]) => void;
  appendResponses: (responses: FauxResponseStep[]) => void;
  getPendingResponseCount: () => number;
  events: AgentSessionEvent[];     // all events emitted by the session so far
  eventsOfType<T>(type: T): Extract<AgentSessionEvent, { type: T }>[];
  tempDir: string;
  cleanup: () => void;             // call this in afterEach
}
```

### A complete extension test

Here is a minimal but complete test showing how to verify that a `tool_call` handler fires and blocks a tool call:

```ts
// A complete, runnable test using the harness
import { describe, it, expect, afterEach } from "vitest";
import { createHarness, type Harness } from "./harness.ts";
import type { ExtensionAPI } from "../../src/core/extensions/types.ts";

describe("permission-gate extension", () => {
  let harness: Harness;

  afterEach(() => harness?.cleanup());

  it("blocks a tool call and returns a custom message", async () => {
    // 1. Define the extension factory inline
    const factory = async (api: ExtensionAPI) => {
      api.on("tool_call", async (event, ctx) => {
        if (event.name === "bash" && event.input.command?.startsWith("rm")) {
          return { block: true, content: "Blocked: destructive command" };
        }
      });
    };

    // 2. Create the harness with that factory
    harness = await createHarness({ extensionFactories: [factory] });

    // 3. Script the faux provider to attempt a "rm" tool call
    harness.setResponses([
      { type: "tool_call", name: "bash", input: { command: "rm -rf /tmp/test" } },
    ]);

    // 4. Send a user message to trigger the agent loop
    await harness.session.sendUserMessage("clean up");

    // 5. Inspect session events or state
    const toolCallEvents = harness.eventsOfType("tool_call");
    expect(toolCallEvents.length).toBeGreaterThan(0);
  });
});
```

Notice that there is no network, no file system extension discovery, and no compiled binary. `extensionFactories` accepts the factory function directly — it bypasses `discoverAndLoadExtensions` entirely and hands the factory straight to `loadExtensionFromFactory`:

```ts
// From loader.ts — for testing and SDK embedding
export async function loadExtensionFromFactory(
  factory: ExtensionFactory,
  cwd: string,
  eventBus: EventBus,
  runtime: ExtensionRuntime,
  extensionPath = "<inline>",
): Promise<Extension> {
  const extension = createExtension(extensionPath, extensionPath);
  const resolvedCwd = resolvePath(cwd);
  const api = createExtensionAPI(extension, runtime, resolvedCwd, eventBus);
  await factory(api);
  return extension;
}
```

This is how the harness loads extension factories without touching the filesystem at all.

### Inspecting events

The `Harness.events` array records every `AgentSessionEvent` in order. The `eventsOfType` helper narrows the type:

```ts
// Inspect specific event types
const toolResults = harness.eventsOfType("tool_result");
const messageEndEvents = harness.eventsOfType("message_end");

// Or read messages directly from the session
const userTexts = getUserTexts(harness);       // all user-role message texts
const assistantTexts = getAssistantTexts(harness); // all assistant-role texts
```

`getUserTexts` and `getAssistantTexts` (exported from `harness.ts`) are convenience helpers that walk `harness.session.messages` and extract text content.

---

## Putting it all together: the lifecycle in one view

Let's trace the full path from user setup to event dispatch:

1. **Agent starts up.** It calls `discoverAndLoadExtensions(configuredPaths, cwd)`.
2. **Loader scans** `.xzy/extensions/`, `~/.xzy/extensions/`, and any explicitly configured paths, deduplicates, and calls `loadExtensions(allPaths, cwd)`.
3. **Per-extension:** `loadExtension` → jiti loads the `.ts` file → factory is called with `createExtensionAPI(extension, runtime, ...)` → handlers, tools, commands, flags are registered into the `Extension` object.
4. **Result returned:** `{ extensions, errors, runtime }`. The agent logs any errors.
5. **`ExtensionRunner` constructed** with the extensions and runtime.
6. **`bindCore` called** with the live session's action and context function objects. Runtime stubs become real. Provider registrations are flushed.
7. **`bindCommandContext` called** with session-navigation functions.
8. **`setUIContext` called** (interactive mode only).
9. **Agent loop runs.** On each event (tool call, message end, context update, …) the agent session calls the matching `runner.emit*` method.
10. **Runner fans out** to all handlers across all extensions, catches errors, chains results where appropriate.
11. **On reload:** the runner is invalidated, `discoverAndLoadExtensions` runs again, a new runner is constructed and bound.

---

## See also

- [The Extension API: Handlers, Context, and Events](./extension-api-and-types.md) — the `ExtensionAPI` type, event catalogue, and handler signatures that extensions use.
- [Building Real Extensions: Tools, State, Commands, and Hooks](./building-real-extensions.md) — end-to-end examples of real extensions built against this loader and runner.

---

← Previous: [The Extension API: Handlers, Context, and Events](./extension-api-and-types.md) · Next: [Building Real Extensions: Tools, State, Commands, and Hooks](./building-real-extensions.md) →
