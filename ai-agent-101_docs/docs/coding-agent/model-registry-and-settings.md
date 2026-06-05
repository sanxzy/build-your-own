---
title: "Model Registry, Settings, and Resource Loading"
description: "Build the four configuration managers that give the agent its model list, user preferences, extension/skill paths, and persisted credentials at startup."
category: coding-agent
type: tutorial
tags: [ModelRegistry, models.json, extension providers, SettingsManager, global settings, project settings, merge, DefaultResourceLoader, resource discovery, auth.json, AuthStorage, coding-agent, configuration, settings precedence, context files, AGENTS.md, SYSTEM.md, API key, OAuth, runtime override]
keywords: [custom models, provider config, settings merge, file lock, reload, skill paths, prompt templates, themes, two-tier config, credential storage]
sources: [S58, S69, S67, S68]
---

**TL;DR** — When the agent starts up it needs four things: a list of every model it can talk to, the user's preferences, the locations of every extension and skill, and the user's API credentials. Each concern lives in its own small class — `ModelRegistry`, `SettingsManager`, `DefaultResourceLoader`, and `AuthStorage`. This chapter builds each one in turn, explaining why it exists before showing how it works.

# Model Registry, Settings, and Resource Loading

Before the agent runs its first turn, it has to answer four questions:

1. Which models can I use, and where are their endpoints?
2. What are the user's preferences — default model, retry behaviour, compaction settings?
3. Where do I find extensions, skills, prompt templates, and themes?
4. What credentials do I use to authenticate to each provider?

Each question is answered by one small manager class. We'll build them up in that order, connecting each one to the next as we go.

If you have not read [AgentSession](./agent-session-core.md) yet — `AgentSession` is the object that consumes all four managers at session start — that chapter gives the broader context. For credentials specifically, [OAuth + API-key auth](../llm-toolkit/oauth-and-api-key-auth.md) explains how the underlying OAuth machinery and env-var key lookup work; we'll recap the relevant parts here.

---

## Part 1 — ModelRegistry: one list from three sources

The agent needs to know which models exist before it can ask the user to pick one. The problem is that models come from three different places:

- **Built-in models** — the llm-toolkit package ships a curated list for known providers (Anthropic, OpenAI, etc.).
- **User's `models.json`** — users can define extra providers (a local Ollama instance, a private endpoint) or override built-in model properties.
- **Extension providers** — extensions can call `registerProvider()` at runtime to inject entirely new models or modify existing ones.

`ModelRegistry` merges all three into one flat list and is the single authority for "what models are available right now."

### Creating the registry

```ts
// Simplified view of ModelRegistry.create()
import { ModelRegistry } from "coding-agent/core/model-registry";
import { AuthStorage } from "coding-agent/core/auth-storage";

const auth = AuthStorage.create();                // reads ~/.xzy/auth.json
const registry = ModelRegistry.create(auth);      // reads ~/.xzy/models.json by default
```

`ModelRegistry.create(authStorage, modelsJsonPath?)` — the second argument defaults to `~/.xzy/models.json`. Calling it is synchronous; the constructor immediately calls `loadModels()`.

There is also `ModelRegistry.inMemory(authStorage)` which skips disk entirely — useful in tests.

### What `loadModels()` does

Let's trace through the private `loadModels()` call so the merge order is clear:

```ts
// Simplified view of loadModels()
private loadModels(): void {
  // Step 1 — read models.json (if it exists)
  const { models: customModels, overrides, modelOverrides, error } =
    this.modelsJsonPath
      ? this.loadCustomModels(this.modelsJsonPath)
      : emptyCustomModelsResult();

  // Step 2 — load built-in models and apply any provider/model overrides from models.json
  const builtInModels = this.loadBuiltInModels(overrides, modelOverrides);

  // Step 3 — merge: custom wins on provider+id conflicts
  let combined = this.mergeCustomModels(builtInModels, customModels);

  // Step 4 — let OAuth providers update their models (e.g., set dynamic baseUrl)
  for (const oauthProvider of this.authStorage.getOAuthProviders()) {
    const cred = this.authStorage.get(oauthProvider.id);
    if (cred?.type === "oauth" && oauthProvider.modifyModels) {
      combined = oauthProvider.modifyModels(combined, cred);
    }
  }

  this.models = combined;
}
```

The merge rule in step 3 is: if a custom model from `models.json` has the same `provider` + `id` as a built-in model, the custom definition wins entirely. New custom models (unknown `provider+id`) are appended.

### The `models.json` format

The file lives at `~/.xzy/models.json` by default. Here is the validated schema in concrete form:

```json
{
  "providers": {
    "my-local-llm": {
      "baseUrl": "http://localhost:11434/v1",
      "apiKey": "ollama",
      "api": "openai-completions",
      "models": [
        {
          "id": "llama3.2",
          "name": "Llama 3.2",
          "reasoning": false,
          "input": ["text"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 128000,
          "maxTokens": 16384
        }
      ]
    }
  }
}
```

The file supports JSON with comments (`// …`) — the registry strips them before parsing.

Here is a summary of what each provider entry can contain:

| Field | Required? | Notes |
|---|---|---|
| `baseUrl` | Required for new providers | API endpoint root URL |
| `apiKey` | Required for new providers | Literal key, `$ENV_VAR`, or command string |
| `api` | Required for new providers | `"openai-completions"`, `"openai-responses"`, or `"anthropic-messages"` |
| `models` | At least one of `models`/`modelOverrides`/`baseUrl`/`headers`/`compat` | Array of model definitions |
| `modelOverrides` | Optional | Patch specific built-in model fields without replacing the whole model |
| `headers` | Optional | Extra HTTP headers for every request to this provider |
| `compat` | Optional | Provider-level compatibility flags |

Each **model definition** inside `models` requires:

| Field | Default (if absent) |
|---|---|
| `id` | — (required) |
| `reasoning` | `false` |
| `input` | `["text"]` |
| `cost` | `{ input:0, output:0, cacheRead:0, cacheWrite:0 }` |
| `contextWindow` | `128000` |
| `maxTokens` | `16384` |

**Override-only config.** If you want to point a built-in provider at a different base URL — for example, routing Anthropic traffic through a proxy — you can add a provider entry with only `baseUrl` and no `models`:

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/anthropic"
    }
  }
}
```

The registry applies the `baseUrl` to all existing Anthropic models; it does not add any new ones.

### Deep-merging model overrides

`modelOverrides` lets you patch individual fields of a built-in model without replacing the whole thing. The `applyModelOverride()` helper merges cost and `compat` objects rather than replacing them:

```json
{
  "providers": {
    "anthropic": {
      "modelOverrides": {
        "claude-opus-4-5": {
          "maxTokens": 8192,
          "cost": { "input": 0.003 }
        }
      }
    }
  }
}
```

Here only `maxTokens` and the `input` cost are replaced; other cost fields and all `compat` flags are preserved from the built-in model.

### Querying the registry

Once loaded, three methods cover everyday needs:

```ts
// All models, whether or not auth is configured
const all = registry.getAll();

// Only models that have credentials (fast check, no token refresh)
const available = registry.getAvailable();

// Look up a specific model
const model = registry.find("anthropic", "claude-opus-4-5");
```

`getAvailable()` calls `hasConfiguredAuth(model)` for every model, which checks `AuthStorage` and any inline `apiKey` from `models.json` without touching the network.

### Registering extension providers

Extensions call `registry.registerProvider(name, config)` to inject models at runtime. The `ProviderConfigInput` type is the programmatic counterpart to the JSON provider schema:

```ts
registry.registerProvider("my-ext-provider", {
  baseUrl: "https://api.example.com/v1",
  apiKey: "$MY_EXT_API_KEY",
  api: "openai-completions",
  models: [
    {
      id: "gpt-x",
      name: "GPT X",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0.001, output: 0.002, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 64000,
      maxTokens: 4096,
    },
  ],
});
```

If the provider already exists, the new values are merged (undefined fields in the incoming config are preserved from the stored config). Calling `unregisterProvider(name)` removes it and triggers a full `refresh()`, restoring built-in models the provider had overridden.

### Resolving auth at request time

At request time, `getApiKeyAndHeaders(model)` assembles the full auth for a given model. The priority order is:

1. API key from `AuthStorage` (covering `auth.json` stored keys, OAuth tokens, and env vars)
2. Inline `apiKey` from `models.json` (resolved if it starts with `$`, executed if it is a command string)
3. Custom headers from `models.json` (per-provider and per-model)

If `authHeader: true` is set for a provider, the key is injected as `Authorization: Bearer <key>` instead of relying on the underlying API client.

### Refreshing after file changes

`registry.refresh()` reloads everything from disk: it clears cached auth, resets the llm-toolkit's dynamic provider registrations, re-runs `loadModels()`, and then re-applies all extension providers that have been registered. Call it whenever `models.json` changes on disk.

---

## Part 2 — SettingsManager: global + project config with merge

Now we have models. But which one does the user prefer? How aggressive should retries be? Should the agent auto-compact long sessions? These answers live in settings files.

The challenge: there are two settings files. The **global** file (`~/.xzy/settings.json`) holds user-wide preferences. The **project** file (`.xzy/settings.json` in the current working directory) holds project-specific overrides. A project might want a different default model without disturbing the user's global theme choice.

`SettingsManager` loads both files, merges them, and exposes typed getters and setters.

### Creating the manager

```ts
import { SettingsManager } from "coding-agent/core/settings-manager";

// Standard: reads ~/.xzy/settings.json and <cwd>/.xzy/settings.json
const settings = SettingsManager.create(process.cwd());

// Testing: no file I/O
const settings = SettingsManager.inMemory({ defaultProvider: "anthropic" });
```

`SettingsManager.create(cwd, agentDir?)` — `agentDir` defaults to `~/.xzy/`. The constructor reads both files immediately (synchronously via `FileSettingsStorage`).

### The merge rule

The `deepMergeSettings(base, overrides)` function defines how global and project settings combine:

```ts
// Simplified view of deepMergeSettings
function deepMergeSettings(base: Settings, overrides: Settings): Settings {
  const result = { ...base };

  for (const key of Object.keys(overrides)) {
    const overrideValue = overrides[key];
    const baseValue = base[key];

    if (overrideValue === undefined) continue;

    // Nested objects (compaction, retry, terminal, images) → merge recursively
    if (isPlainObject(overrideValue) && isPlainObject(baseValue)) {
      result[key] = { ...baseValue, ...overrideValue };
    } else {
      // Primitives and arrays → override wins outright
      result[key] = overrideValue;
    }
  }

  return result;
}
```

The key distinction: **nested objects merge**; **primitives and arrays replace**. Let's make that concrete with an example.

**Global `~/.xzy/settings.json`:**
```json
{
  "defaultProvider": "anthropic",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  }
}
```

**Project `.xzy/settings.json`:**
```json
{
  "defaultModel": "claude-opus-4-5",
  "compaction": {
    "keepRecentTokens": 30000
  }
}
```

**Merged result (what the agent sees):**
```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-opus-4-5",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 30000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  }
}
```

The project only specified `compaction.keepRecentTokens`, so `enabled` and `reserveTokens` from the global file survived. `defaultModel` came from the project; `defaultProvider` came from the global file.

Here is a summary of the merge behaviour by value type:

| Value type | Merge behaviour | Example |
|---|---|---|
| Nested plain object | Keys merged; project wins on conflicts | `compaction`, `retry`, `terminal`, `images` |
| Primitive (`string`, `boolean`, `number`) | Project wins entirely | `defaultProvider`, `theme`, `quietStartup` |
| Array | Project wins entirely (no concat) | `extensions`, `skills`, `packages` |

### Settings file paths

| Scope | Default location |
|---|---|
| Global | `~/.xzy/settings.json` |
| Project | `<cwd>/.xzy/settings.json` |

Both files are created on first write. `FileSettingsStorage` creates parent directories as needed and uses `proper-lockfile` to prevent race conditions when multiple agent instances write concurrently.

### Key settings fields

The `Settings` interface has many fields. Here are the ones most directly relevant to the agent's core behaviour:

| Field | Default | Notes |
|---|---|---|
| `defaultProvider` | `undefined` | Provider selected at startup |
| `defaultModel` | `undefined` | Model selected at startup |
| `defaultThinkingLevel` | `undefined` | One of `"off"`, `"minimal"`, `"low"`, `"medium"`, `"high"`, `"xhigh"` |
| `compaction.enabled` | `true` | Auto-compaction of long sessions |
| `compaction.reserveTokens` | `16384` | Tokens kept for the compaction LLM response |
| `compaction.keepRecentTokens` | `20000` | Tokens of recent messages to keep verbatim |
| `retry.enabled` | `true` | Whether to retry on transient errors |
| `retry.maxRetries` | `3` | Max retry attempts |
| `retry.baseDelayMs` | `2000` | Base delay (exponential backoff: 2 s, 4 s, 8 s) |
| `transport` | `"auto"` | `"sse"`, `"websocket"`, or `"auto"` |
| `extensions` | `[]` | Local extension file paths or directories |
| `skills` | `[]` | Local skill file paths or directories |
| `prompts` | `[]` | Local prompt template paths or directories |
| `themes` | `[]` | Local theme paths or directories |
| `packages` | `[]` | npm/git package sources to load resources from |
| `enableSkillCommands` | `true` | Register skills as `/skill:name` slash commands |

### Reading and writing settings

Getters return the merged value with defaults applied:

```ts
settings.getDefaultProvider();       // string | undefined
settings.getCompactionEnabled();     // boolean (default: true)
settings.getRetrySettings();         // { enabled, maxRetries, baseDelayMs }
```

Setters always write to the **global** scope. They update the in-memory state immediately and enqueue an async write:

```ts
settings.setDefaultModelAndProvider("anthropic", "claude-opus-4-5");
settings.setCompactionEnabled(false);
await settings.flush(); // wait for all queued writes to complete
```

For project-scoped writes, use the `setProject*` variants:

```ts
settings.setProjectExtensionPaths(["./my-extension.ts"]);
settings.setProjectSkillPaths(["./my-skills/"]);
```

### Runtime overrides

`applyOverrides(overrides)` layers additional settings on top of the merged global+project settings without touching either file. This is how CLI flags — like `--offline` — temporarily adjust behaviour:

```ts
settings.applyOverrides({ transport: "sse" });
```

These overrides survive only for the current process lifetime.

### Settings migrations

When `SettingsManager` loads a settings file, it runs `migrateSettings()` to translate old field names to new ones silently. Current migrations:

| Old field | New field |
|---|---|
| `queueMode` | `steeringMode` |
| `websockets: boolean` | `transport: "websocket" \| "sse"` |
| `skills` (object) | `skills` (array) + `enableSkillCommands` |
| `retry.maxDelayMs` | `retry.provider.maxRetryDelayMs` |

Migrations run at read time and are never written back unless the user writes a new value.

---

## Part 3 — DefaultResourceLoader: discovering extensions, skills, and prompts

We know our models and preferences. Now we need the resources the agent will use: extensions (tools and commands), skills (reusable prompt procedures), prompt templates, themes, and context files (`AGENTS.md`, `CLAUDE.md`).

`DefaultResourceLoader` implements the `ResourceLoader` interface and is responsible for discovering and loading all of these from their standard search paths. Its `reload()` method is async and must be called after construction before any resources are accessible.

### Construction

```ts
import { DefaultResourceLoader } from "coding-agent/core/resource-loader";

const loader = new DefaultResourceLoader({
  cwd: process.cwd(),
  agentDir: "~/.xzy",          // optional, defaults to getAgentDir()
  settingsManager: settings,   // optional, creates one internally if absent
});

await loader.reload();         // discover and load everything from disk
```

The constructor accepts many optional overrides — we'll cover the most important ones below.

### What `reload()` does

`reload()` is the central loading pass. It runs these steps in order:

1. **Reload settings** — calls `settingsManager.reload()` so any settings changes on disk are picked up.
2. **Resolve packages** — the `DefaultPackageManager` resolves npm/git package sources listed in `settings.packages` and extracts their extension, skill, prompt, and theme paths.
3. **Load extensions** — collects paths from CLI args, packages, and `settings.extensions`; loads each extension file via `loadExtensions()`. Detects and reports tool/command/flag name conflicts between extensions.
4. **Load inline extension factories** — if `extensionFactories` were passed as constructor options, loads them into the same extension runtime.
5. **Load skills** — collects paths from CLI args, packages, and `settings.skills`; loads skill files; dedupes by name (first wins on conflict).
6. **Load prompt templates** — same pattern; dedupes by name.
7. **Load themes** — same pattern; dedupes by name.
8. **Load context files** — walks from `cwd` up to the filesystem root collecting `AGENTS.md` / `CLAUDE.md` files (see below).
9. **Resolve system prompt** — looks for `SYSTEM.md` in `.xzy/` (project) then `~/.xzy/` (global).

### Resource search paths

For each resource type, paths come from three sources that are merged (deduplicated by canonical path):

| Source | Example |
|---|---|
| CLI args (`additionalExtensionPaths`, etc.) | `--extension ./my-ext.ts` |
| npm/git packages (`settings.packages`) | `["my-xzy-extension-pkg"]` |
| Settings paths (`settings.extensions`, etc.) | `["~/.xzy/extensions/"]` |

Each source type follows this directory layout under `~/.xzy/` (global) and `.xzy/` (project):

| Resource | Global directory | Project directory |
|---|---|---|
| Extensions | `~/.xzy/extensions/` | `.xzy/extensions/` |
| Skills | `~/.xzy/skills/` | `.xzy/skills/` |
| Prompt templates | `~/.xzy/prompts/` | `.xzy/prompts/` |
| Themes | `~/.xzy/themes/` | `.xzy/themes/` |

### Context file discovery

Context files are the `AGENTS.md` / `CLAUDE.md` files that inject persistent instructions into the agent's system prompt. The discovery algorithm:

1. Look for an `AGENTS.md` or `CLAUDE.md` in the global agent directory (`~/.xzy/`). If found, add it first.
2. Walk up from `cwd` to the filesystem root, collecting context files from each directory.
3. Sort ancestor files from root to `cwd` (so the closest file wins at append time).

The candidates checked in each directory are: `AGENTS.md`, `AGENTS.MD`, `CLAUDE.md`, `CLAUDE.MD`.

### System prompt discovery

The `discoverSystemPromptFile()` helper looks for a `SYSTEM.md` file:

1. `.xzy/SYSTEM.md` (project-level)
2. `~/.xzy/SYSTEM.md` (global)

The first one found wins. If neither exists, the system prompt is `undefined` and the agent uses its built-in default. There is also `APPEND_SYSTEM.md` at the same locations, which appends content to the system prompt rather than replacing it.

### Accessing loaded resources

After `reload()`, the loader exposes:

```ts
const { extensions, errors } = loader.getExtensions();
const { skills, diagnostics } = loader.getSkills();
const { prompts, diagnostics } = loader.getPrompts();
const { themes, diagnostics } = loader.getThemes();
const { agentsFiles } = loader.getAgentsFiles();
const systemPrompt = loader.getSystemPrompt();
```

`diagnostics` is an array of `ResourceDiagnostic` objects — each has a `type` (`"error"`, `"warning"`, or `"collision"`), a `message`, and a `path`. Errors indicate files that failed to load; collisions report name conflicts between resources.

### Dynamic resource injection

Extensions can register additional skill, prompt, or theme paths after the initial load via `extendResources()`:

```ts
loader.extendResources({
  skillPaths: [{ path: "/path/to/more/skills", metadata: { source: "extension", scope: "temporary", origin: "top-level" } }],
  promptPaths: [],
  themePaths: [],
});
```

This is how extensions provide their own skills and templates without knowing the full search path up front.

### Suppressing categories

Pass `no*` flags to the constructor to skip entire categories:

```ts
const loader = new DefaultResourceLoader({
  cwd,
  agentDir,
  noExtensions: true,      // skip all extension loading
  noSkills: true,
  noPromptTemplates: true,
  noContextFiles: true,    // skip AGENTS.md / CLAUDE.md discovery
});
```

This is useful in non-interactive modes (print, RPC) where extensions and context files are not needed.

---

## Part 4 — AuthStorage: persisting and resolving credentials

The agent needs API keys (or OAuth tokens) for every provider it talks to. We saw how `ModelRegistry` calls `AuthStorage` when resolving request auth — let's look at what `AuthStorage` actually does.

`AuthStorage` covers two credential types:

- **API key** (`{ type: "api_key", key: string }`) — a raw key string, possibly an env-var reference.
- **OAuth token** (`{ type: "oauth", …OAuthCredentials }`) — a full OAuth credential set including access token and expiry.

For a full explanation of how OAuth login/refresh work in the llm-toolkit, see [OAuth + API-key auth](../llm-toolkit/oauth-and-api-key-auth.md). Here we focus on how `AuthStorage` persists and retrieves these credentials.

### The storage file

By default `AuthStorage` reads and writes `~/.xzy/auth.json`. The file is created with permissions `0600` (owner-readable only) and the parent directory with `0700`. File locking (`proper-lockfile`) prevents corruption when multiple agent instances try to update tokens simultaneously.

```ts
import { AuthStorage } from "coding-agent/core/auth-storage";

// Standard: reads ~/.xzy/auth.json
const auth = AuthStorage.create();

// Custom path
const auth = AuthStorage.create("/path/to/custom/auth.json");

// In-memory (tests or ephemeral sessions)
const auth = AuthStorage.inMemory({ anthropic: { type: "api_key", key: "sk-…" } });
```

### Reading and writing credentials

```ts
// Store an API key
auth.set("anthropic", { type: "api_key", key: "sk-ant-…" });

// Retrieve a credential (synchronous, no refresh)
const cred = auth.get("anthropic"); // AuthCredential | undefined

// Remove
auth.remove("anthropic");

// List all providers that have stored credentials
const providers = auth.list(); // string[]
```

These operations update the in-memory data and immediately persist to `auth.json` under a lock.

### API key resolution priority

`getApiKey(provider)` is the async method that resolves a working API key. It checks sources in this order:

| Priority | Source | Notes |
|---|---|---|
| 1 | Runtime override (`setRuntimeApiKey`) | Set via CLI `--api-key` flag; not persisted |
| 2 | Stored API key (`auth.json`) | `type: "api_key"` credential |
| 3 | OAuth token (`auth.json`) | `type: "oauth"` — auto-refreshed with locking |
| 4 | Environment variable | Looked up via the llm-toolkit's `getEnvApiKey()` |
| 5 | Fallback resolver | Custom resolver set by `ModelRegistry` for `models.json` providers |

```ts
const key = await auth.getApiKey("anthropic");
// → "sk-ant-…" if any of the above sources has it, or undefined
```

### Runtime override

When the user passes `--api-key sk-…` on the command line, the CLI calls:

```ts
auth.setRuntimeApiKey("anthropic", "sk-ant-…");
```

This puts the key at the top of the priority list without writing to disk. The override is removed when the process exits. `removeRuntimeApiKey(provider)` removes it explicitly.

### OAuth auto-refresh

When a stored OAuth token has passed its expiry (`Date.now() >= cred.expires`), `getApiKey()` triggers an async refresh under a file lock. This prevents the classic "thundering herd" problem where multiple concurrent agent instances all try to refresh the same token at the same time:

1. The first instance acquires the lock and refreshes the token.
2. Other instances wait, then re-read the file.
3. If the file now has a fresh token, they use it — no extra network call.

If the refresh fails, `AuthStorage` reloads the file from disk to check whether another instance already succeeded before giving up.

### Checking auth without fetching a key

`hasAuth(provider)` is a synchronous fast-path that returns `true` if *any* source has a credential — without triggering an OAuth refresh:

```ts
if (auth.hasAuth("anthropic")) {
  // safe to offer this provider in the model list
}
```

`getAuthStatus(provider)` returns a structured object that indicates which source the credential comes from, without exposing the credential value itself:

```ts
const status = auth.getAuthStatus("anthropic");
// { configured: true, source: "stored" }
// { configured: true, source: "environment", label: "ANTHROPIC_API_KEY" }
// { configured: false }
```

### Logging in and logging out

For OAuth providers registered via extensions or the built-in provider list:

```ts
// Login: opens the OAuth flow (browser redirect, device code, etc.)
await auth.login("anthropic", {
  onUrl: (url) => console.log(`Open: ${url}`),
  onCode: (code) => console.log(`Code: ${code}`),
});

// Logout: removes the stored credential
auth.logout("anthropic");
```

---

## Putting it together

At agent startup, the four managers are created in dependency order — `AuthStorage` first because `ModelRegistry` needs it, `SettingsManager` next because `DefaultResourceLoader` needs it, then both `ModelRegistry` and `DefaultResourceLoader`:

```ts
// Typical startup sequence (simplified from agent-session.ts)
import { AuthStorage } from "coding-agent/core/auth-storage";
import { ModelRegistry } from "coding-agent/core/model-registry";
import { SettingsManager } from "coding-agent/core/settings-manager";
import { DefaultResourceLoader } from "coding-agent/core/resource-loader";

const auth = AuthStorage.create();
const registry = ModelRegistry.create(auth);       // reads ~/.xzy/models.json
const settings = SettingsManager.create(cwd);      // reads global + project settings.json
const loader = new DefaultResourceLoader({ cwd, settingsManager: settings });
await loader.reload();                             // discovers all resources from disk

// Now hand these to AgentSession
```

As we build in [AgentSession](./agent-session-core.md), the session takes all four as constructor arguments and delegates every "what model / what preference / what skill" question back to them. The managers are long-lived — they stay alive for the entire session and are refreshed when settings change on disk.

---

← Previous: [Coding-Agent Compaction and Branch Summarization](./compaction-and-branch-summarization.md) · Next: [Interactive Mode: Startup, Wiring, and the TUI App Shell](./interactive-mode-startup-and-wiring.md) →
