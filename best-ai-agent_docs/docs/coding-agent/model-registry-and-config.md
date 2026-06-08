---
title: "Model Registry, Settings, and Configuration"
description: "Build the configuration layer — ModelRegistry with extension-provided models, SettingsManager with global and project settings merge, resource discovery, and secure auth credential storage."
category: coding-agent
type: tutorial
tags: [ModelRegistry, models.json, extension providers, SettingsManager, global settings, project settings, merge, ResourceLoader, auth storage, configuration, coding-agent]
keywords: [configuration, settings manager, model discovery, auth storage, resource loading]
sources: [S45, S55, S56]
---

**TL;DR** — The coding agent needs configuration for models, settings, and credentials. We'll build a three-part configuration layer: a **ModelRegistry** that discovers models from built-in definitions, user `models.json` files, and extension providers; a **SettingsManager** that merges global, project, and environment settings with clear precedence; and an **AuthStorage** that securely persists API keys and OAuth tokens.

## ModelRegistry

Models come from three sources. Create `packages/coding-agent/src/core/model-registry.ts`:

```ts
class ModelRegistry {
  private builtIn = new Map<string, Model>();
  private userModels = new Map<string, Model>();
  private extensionProviders = new Map<string, ModelProvider>();

  constructor() {
    this.loadBuiltIn();
    this.loadUserModels();
  }

  private loadBuiltIn(): void {
    // All models from the llm-toolkit's generated registry
    for (const model of getModels()) {
      this.builtIn.set(model.id, model);
    }
  }

  private loadUserModels(): void {
    const configDir = this.findConfigDir();
    const modelsPath = path.join(configDir, "models.json");
    if (fs.existsSync(modelsPath)) {
      const userDefs = JSON.parse(fs.readFileSync(modelsPath, "utf-8"));
      for (const def of userDefs) {
        this.userModels.set(def.id, { ...this.builtIn.get(def.id), ...def });
      }
    }
  }

  registerExtensionProvider(provider: ModelProvider): void {
    this.extensionProviders.set(provider.id, provider);
  }

  getModel(id: string): Model | undefined {
    return this.userModels.get(id)
      ?? this.builtIn.get(id)
      ?? this.findInExtensions(id);
  }

  listModels(filter?: { provider?: string; supportsImages?: boolean }): Model[] {
    let all = [...this.builtIn.values(), ...this.userModels.values()];
    for (const provider of this.extensionProviders.values()) {
      all.push(...provider.listModels());
    }
    if (filter?.provider) all = all.filter(m => m.provider === filter.provider);
    if (filter?.supportsImages) all = all.filter(m => m.supportsImages);
    return all;
  }
}
```

Precedence: user `models.json` overrides built-in, extensions add new models.

## SettingsManager

Settings come from three tiers with clear merge rules. Create `packages/coding-agent/src/config.ts`:

```ts
interface Settings {
  model?: string;
  thinkingLevel?: ModelThinkingLevel;
  theme?: "dark" | "light";
  editor?: string;
  autoCompact?: boolean;
  maxTokens?: number;
  showThinking?: boolean;
  // ... more settings
}

class SettingsManager {
  private global: Partial<Settings> = {};
  private project: Partial<Settings> = {};

  constructor(private configDir: string) {
    this.loadGlobal();
  }

  private loadGlobal(): void {
    const path = this.globalSettingsPath();
    if (fs.existsSync(path)) {
      this.global = JSON.parse(fs.readFileSync(path, "utf-8"));
    }
  }

  loadProject(cwd: string): void {
    const path = path.join(cwd, this.configDir, "settings.json");
    if (fs.existsSync(path)) {
      this.project = JSON.parse(fs.readFileSync(path, "utf-8"));
    }
  }

  get<K extends keyof Settings>(key: K): Settings[K] | undefined {
    // Environment overrides everything
    const envKey = `CODING_AGENT_${key.toUpperCase()}`;
    if (process.env[envKey]) {
      return this.parseEnv(process.env[envKey]!);
    }
    // Project overrides global
    if (this.project[key] !== undefined) return this.project[key];
    // Global is the default
    return this.global[key];
  }

  async setGlobal<K extends keyof Settings>(key: K, value: Settings[K]): Promise<void> {
    this.global[key] = value;
    await this.saveGlobal();
  }

  async setProject<K extends keyof Settings>(key: K, value: Settings[K], cwd: string): Promise<void> {
    this.project[key] = value;
    const dir = path.join(cwd, this.configDir);
    await fs.promises.mkdir(dir, { recursive: true });
    await fs.promises.writeFile(
      path.join(dir, "settings.json"),
      JSON.stringify(this.project, null, 2),
    );
  }
}
```

Precedence: **Environment variables > Project settings > Global settings > Built-in defaults**.

## AuthStorage

API keys and OAuth tokens need secure persistence:

```ts
class AuthStorage {
  constructor(private configDir: string) {}

  private get authPath(): string {
    return path.join(this.configDir, "auth.json");
  }

  async getApiKey(provider: string): Promise<string | undefined> {
    const auth = await this.load();
    return auth.apiKeys?.[provider];
  }

  async setApiKey(provider: string, key: string): Promise<void> {
    const auth = await this.load();
    auth.apiKeys ??= {};
    auth.apiKeys[provider] = key;
    await this.save(auth);
  }

  async getOAuthTokens(provider: string): Promise<OAuthTokens | undefined> {
    const auth = await this.load();
    const stored = auth.oauth?.[provider];
    if (!stored) return undefined;
    if (stored.expiresAt < Date.now()) {
      // Token expired — try refresh
      return this.refreshOAuthTokens(provider, stored);
    }
    return stored;
  }

  private async load(): Promise<AuthData> {
    try {
      const data = await fs.promises.readFile(this.authPath, "utf-8");
      return JSON.parse(data);
    } catch {
      return {};
    }
  }

  private async save(data: AuthData): Promise<void> {
    await fs.promises.mkdir(this.configDir, { recursive: true });
    await fs.promises.writeFile(
      this.authPath,
      JSON.stringify(data, null, 2),
      { mode: 0o600 }, // owner read/write only
    );
  }
}
```

The `auth.json` file is created with `0o600` permissions — only the file owner can read it. This is basic but effective for local credential storage.

## Resource discovery

The `ResourceLoader` finds config files across the standard search paths:

```ts
class ResourceLoader {
  private searchPaths: string[];

  constructor(configDir: string) {
    this.searchPaths = [
      path.join(configDir),               // global: ~/.coding-agent/
      ".",                                 // cwd
      path.join(".", configDir),           // cwd/.coding-agent/
    ];
  }

  find(name: string): string | undefined {
    for (const dir of this.searchPaths) {
      const candidate = path.join(dir, name);
      if (fs.existsSync(candidate)) return candidate;
    }
    return undefined;
  }

  findAll(name: string): string[] {
    return this.searchPaths
      .map(dir => path.join(dir, name))
      .filter(p => fs.existsSync(p));
  }
}
```

## What we've built

- **ModelRegistry** — merges built-in, user, and extension-provided models
- **SettingsManager** — three-tier merge (env > project > global) with typed getters
- **AuthStorage** — secure credential persistence with OAuth token refresh
- **ResourceLoader** — config file discovery across standard paths

---

← Previous: [Context Compaction and Branch Summarization](./compaction-and-summarization.md) · Next: [Interactive Mode: The Full Terminal Chat Experience](./interactive-mode.md) →
