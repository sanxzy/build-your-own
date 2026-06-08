---
title: "Loading and Running Extensions: Discovery, Isolation, and Lifecycle"
description: "Build the extension loader — TypeScript runtime execution, multi-path discovery, error isolation, and the ExtensionRunner lifecycle."
category: extensions
type: tutorial
tags: [extension loader, TypeScript runtime, multi-path discovery, error isolation, ExtensionRunner, bind to session, event dispatch, lifecycle, hot reload, extensions]
keywords: [extension loader, runtime execution, path discovery, error isolation, hot reload]
sources: [S50]
---

**TL;DR** — Extensions are TypeScript files, but they need to be loaded at runtime (not compiled into the agent). We'll build a loader that discovers extensions across multiple paths, executes them in a TypeScript runtime, isolates errors so one broken extension doesn't crash everything, and supports hot reload during development.

## Extension discovery

Extensions are discovered in three locations:

```ts
interface ExtensionSource {
  name: string;
  path: string;
  type: "directory" | "file" | "inline";
}

function discoverExtensions(config: Config): ExtensionSource[] {
  const sources: ExtensionSource[] = [];

  // 1. Global extensions (~/.coding-agent/extensions/)
  const globalDir = path.join(config.configDir, "extensions");
  if (fs.existsSync(globalDir)) {
    for (const entry of fs.readdirSync(globalDir)) {
      sources.push({
        name: entry.replace(/\.(ts|js)$/, ""),
        path: path.join(globalDir, entry),
        type: fs.statSync(path.join(globalDir, entry)).isDirectory() ? "directory" : "file",
      });
    }
  }

  // 2. Project extensions (.coding-agent/extensions/)
  const projectDir = path.join(config.cwd, config.configDir, "extensions");
  if (fs.existsSync(projectDir)) {
    for (const entry of fs.readdirSync(projectDir)) {
      sources.push({
        name: entry.replace(/\.(ts|js)$/, ""),
        path: path.join(projectDir, entry),
        type: fs.statSync(path.join(projectDir, entry)).isDirectory() ? "directory" : "file",
      });
    }
  }

  // 3. CLI-specified extensions (--extension path/to/ext.ts)
  for (const extPath of config.extensionPaths ?? []) {
    sources.push({
      name: path.basename(extPath, path.extname(extPath)),
      path: extPath,
      type: "file",
    });
  }

  return sources;
}
```

Precedence when multiple extensions have the same name: project > global. The first one found wins.

## TypeScript runtime execution

Extensions are TypeScript files executed at runtime using `jiti` — a just-in-time TypeScript compiler:

```ts
import { createJiti } from "jiti";

const jiti = createJiti(import.meta.url, {
  interopDefault: true,
  extensions: [".ts", ".tsx", ".mts", ".cts"],
});

async function loadExtension(source: ExtensionSource): Promise<Extension> {
  const entryFile = source.type === "directory"
    ? path.join(source.path, "index.ts")
    : source.path;

  const module = await jiti.import(entryFile);

  if (!module.activate || typeof module.activate !== "function") {
    throw new Error(`Extension "${source.name}" must export an activate function`);
  }

  return {
    name: source.name,
    path: source.path,
    module,
  };
}
```

`jiti` compiles TypeScript to JavaScript on-the-fly, with full support for ESM, TypeScript features, and node_modules resolution. This means extension authors write standard TypeScript and don't need a build step.

## Error isolation

A broken extension must never crash the agent:

```ts
class ExtensionRunner {
  private extensions = new Map<string, LoadedExtension>();

  async loadAll(sources: ExtensionSource[], apiFactory: (name: string) => ExtensionAPI): Promise<void> {
    const results = await Promise.allSettled(
      sources.map(async (source) => {
        const api = apiFactory(source.name);
        const ext = await loadExtension(source);
        await ext.module.activate(api);
        return { source, api };
      }),
    );

    for (const result of results) {
      if (result.status === "fulfilled") {
        this.extensions.set(result.value.source.name, {
          extension: result.value.api,
          module: result.value.source.module,
        });
      } else {
        console.error(`Extension "${result.reason.source?.name ?? "unknown"}" failed to load:`, result.reason);
        // Continue loading other extensions — never fail the whole agent
      }
    }
  }

  async reload(name: string, apiFactory: (name: string) => ExtensionAPI): Promise<void> {
    const existing = this.extensions.get(name);
    if (!existing) throw new Error(`Extension "${name}" not loaded`);

    // Deactivate old
    if (existing.module.deactivate) {
      await existing.module.deactivate();
    }

    // Clear require cache for fresh load
    const jiti = createJiti(import.meta.url);
    jiti.import.cache.clear();

    // Reactivate
    const api = apiFactory(name);
    const source = existing.source;
    const ext = await loadExtension(source);
    await ext.module.activate(api);

    this.extensions.set(name, { api, source, module: ext.module });
  }
}
```

`Promise.allSettled` is the key — every extension loads in parallel, and failures in one don't affect others. The agent starts even if some extensions fail.

## Lifecycle binding

Extensions are bound to a session lifecycle:

```ts
class ExtensionRunner {
  async bindToSession(session: AgentSession): Promise<void> {
    for (const [name, ext] of this.extensions) {
      // Wire extension hooks into the session's hook system
      for (const [hookType, handler] of ext.api.getRegisteredHooks()) {
        session.hooks.register(hookType, handler);
      }

      // Wire extension tools into the agent
      for (const tool of ext.api.getRegisteredTools()) {
        session.registerTool(tool);
      }

      // Wire extension providers into the model registry
      for (const provider of ext.api.getRegisteredProviders()) {
        session.modelRegistry.registerExtensionProvider(provider);
      }
    }

    // Emit SessionStart for all extensions
    await session.hooks.run("SessionStart", {
      sessionId: session.id,
      cwd: session.cwd,
      modelId: session.modelId,
    });
  }

  async unbindFromSession(session: AgentSession): Promise<void> {
    await session.hooks.run("SessionEnd", {
      sessionId: session.id,
      messageCount: session.getMessages().length,
    });

    // Deactivate all extensions
    for (const [name, ext] of this.extensions) {
      if (ext.module.deactivate) {
        await ext.module.deactivate();
      }
    }
  }
}
```

## Hot reload

During development, extensions can be reloaded without restarting the agent:

```ts
// In interactive mode, bound to a key:
this.chat.bindCommand("/reload-extensions", async () => {
  await extensionRunner.unbindFromSession(this.session);
  const sources = discoverExtensions(this.config);
  await extensionRunner.loadAll(sources, createApi);
  await extensionRunner.bindToSession(this.session);
  return "Extensions reloaded.";
});
```

## What we've built

- **Multi-path discovery** — global, project, and CLI-specified extensions
- **Runtime TypeScript execution** via `jiti` — no build step for extension authors
- **Error isolation** — one broken extension never crashes the agent
- **Session binding** — extensions wired to hooks, tools, and providers at session start
- **Hot reload** — reload extensions during development without restarting

---

← Previous: [The Extension API](./extension-api.md) · Next: [Building Real Extensions: Tools, State, Commands, and Hooks](./building-real-extensions.md) →
