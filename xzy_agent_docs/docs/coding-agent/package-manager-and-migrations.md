---
title: "Package Management and Config Migrations"
description: "How to install, remove, and update extension packages from npm or git, how resources are auto-discovered, and how the agent safely upgrades its config schema across versions."
category: coding-agent
type: tutorial
tags: [package manager, npm, git packages, install, remove, update, manifest, auto-discovery, migrations, config schema, coding-agent, packages, version migration, telemetry, offline mode, XZY_OFFLINE, XZY_TELEMETRY, PackageManager, DefaultPackageManager, runMigrations, ProgressEvent, ResolvedPaths, PackageManifest]
keywords: [xzy install, xzy remove, xzy update, extension packages, npm package install, git clone package, package manifest key, conventional directories, settings migration, auth migration, keybindings migration, prompts migration, telemetry opt-out, anonymous version check]
sources: [S73, S91, S90]
---

**TL;DR** — Once users build extensions and skills, they want to share them. This chapter walks through the package manager that installs extension bundles from npm or a git repository, how packages advertise their resources via a manifest key or conventional directory layout, and how to remove or update those packages. We then look at the config-migration system that upgrades old settings files automatically on startup, and finish with the telemetry pattern and its opt-out mechanism.

# Package Management and Config Migrations

## The problem: sharing extensions across machines and teams

So far in this library we have written extensions, skills, prompts, and themes directly into the local config directories. That works well for your own machine, but it falls apart the moment you want to share a curated bundle with a teammate, install someone else's toolkit, or deploy a reproducible agent environment.

We need a way to package up a collection of resources, publish it, and let others install it with a single command. That is what the package manager provides.

A quick note on context: the settings manager (covered in [Model Registry and Settings](./model-registry-and-settings.md)) persists the list of installed packages in `settings.json`, and the resource loader reads those paths into the agent at startup. Packages drop resources into the same search paths that the resource loader already understands — installing a package is just a way to populate those paths from a remote source.

---

## Step 1 — Understanding the `PackageManager` interface

Let's start with what the package manager promises. In `src/core/package-manager.ts`, the `PackageManager` interface defines every operation callers can request:

```ts
// Simplified view of PackageManager — real names preserved
export interface PackageManager {
  resolve(onMissing?: (source: string) => Promise<MissingSourceAction>): Promise<ResolvedPaths>;
  install(source: string, options?: { local?: boolean }): Promise<void>;
  installAndPersist(source: string, options?: { local?: boolean }): Promise<void>;
  remove(source: string, options?: { local?: boolean }): Promise<void>;
  removeAndPersist(source: string, options?: { local?: boolean }): Promise<boolean>;
  update(source?: string): Promise<void>;
  listConfiguredPackages(): ConfiguredPackage[];
  resolveExtensionSources(
    sources: string[],
    options?: { local?: boolean; temporary?: boolean },
  ): Promise<ResolvedPaths>;
  addSourceToSettings(source: string, options?: { local?: boolean }): boolean;
  removeSourceFromSettings(source: string, options?: { local?: boolean }): boolean;
  setProgressCallback(callback: ProgressCallback | undefined): void;
  getInstalledPath(source: string, scope: "user" | "project"): string | undefined;
}
```

A few things to notice right away:

- **`install` vs `installAndPersist`** — `install` downloads the package but does not record it in settings. `installAndPersist` does both: it calls `install` then immediately calls `addSourceToSettings`. This is the pair you typically want for a user-facing command.
- **`remove` vs `removeAndPersist`** — mirrors the install pair.
- **`local` option** — when `true`, the package is scoped to the current project (written to `.xzy/settings.json`); when `false` (the default), it is user-scoped (written to `~/.xzy/settings.json`).
- **`resolve`** — compiles every configured source into a single `ResolvedPaths` result the rest of the agent consumes. This is what runs at agent startup.

The concrete implementation is `DefaultPackageManager`, which is constructed with `cwd`, `agentDir`, and a `SettingsManager`:

```ts
interface PackageManagerOptions {
  cwd: string;
  agentDir: string;
  settingsManager: SettingsManager;
}
```

---

## Step 2 — Source types: npm, git, and local

Now we have a problem to solve: a "source string" can refer to three entirely different things. The package manager handles them uniformly through a private `parseSource` method that returns one of three parsed shapes:

```ts
type NpmSource = {
  type: "npm";
  spec: string;   // e.g. "my-extension@1.2.0"
  name: string;   // e.g. "my-extension"
  pinned: boolean; // true when a version is specified
};

type LocalSource = {
  type: "local";
  path: string;
};

// GitSource comes from the git URL parser — has host, path, repo, ref
type ParsedSource = NpmSource | GitSource | LocalSource;
```

The `parseSource` logic works as follows:

1. If the string starts with `"npm:"`, strip the prefix and parse the spec.
2. If the string looks like a local path (starts with `/`, `./`, `../`, or `~`), treat it as `LocalSource`.
3. Otherwise, try to parse it as a git URL. If that succeeds, use `GitSource`.
4. If nothing matched, fall back to `LocalSource`.

This means users can write sources in any of these forms:

```bash
# npm package (latest)
xzy install npm:my-extension-pkg

# npm package pinned to a specific version
xzy install npm:my-extension-pkg@1.2.0

# git repository (clones and checks out default branch)
xzy install github.com/someone/cool-extension

# git repository at a specific ref
xzy install github.com/someone/cool-extension@v2.0.0

# local directory (useful during development)
xzy install ./my-local-extension
```

The `local` option on the install command determines *scope* (user vs project), not *source type*.

---

## Step 3 — Installing from npm

When the source is an npm package, the package manager invokes npm (or your configured alternative) through the private `installNpm` method. Let's trace what happens:

1. It determines the *install root* — a managed directory where all npm packages live. For user scope this is `~/.xzy/npm/`; for project scope it is `.xzy/npm/` relative to the project root.
2. It calls `ensureNpmProject`, which creates that directory if it does not exist, writes a minimal `package.json` into it (so npm has a valid project to install into), and adds a `.gitignore` to keep the directory out of version control.
3. It then calls npm (or the configured package manager) with `install <spec> --prefix <installRoot>` plus peer-dependency flags appropriate to the detected package manager.

The install args differ by package manager to suppress peer-dependency resolution. Here are the actual flags used:

| Package manager | Extra flags |
|---|---|
| npm (default) | `--legacy-peer-deps` |
| bun | `--omit=peer` |
| pnpm | `--config.auto-install-peers=false --config.strict-peer-dependencies=false --config.strict-dep-builds=false` |

Why suppress peer deps? Extension packages are loaded inside the agent process, which already provides the agent's own core libraries. If npm tried to install its own copy of those libraries as peers, it could produce version conflicts that block updates. The `--legacy-peer-deps` family of flags prevents that.

A `ProgressEvent` is emitted at each stage so the UI can show progress:

```ts
export interface ProgressEvent {
  type: "start" | "progress" | "complete" | "error";
  action: "install" | "remove" | "update" | "clone" | "pull";
  source: string;
  message?: string;
}

export type ProgressCallback = (event: ProgressEvent) => void;
```

Callers register a callback with `setProgressCallback`; the package manager fires it at the start and end of every operation (or on error).

---

## Step 4 — Installing from git

Git packages follow a different path. The `installGit` method:

1. Computes a deterministic target directory inside `~/.xzy/git/` (or `.xzy/git/` for project scope) using the repository host and path.
2. If the directory already exists, it updates to the correct ref rather than re-cloning.
3. If it does not exist, it creates the parent directories, runs `git clone <repo> <targetDir>`, and if a `ref` was specified, checks it out.
4. After cloning, if the repository has a `package.json`, it runs `npm install` (with `--omit=dev`) to install the package's own dependencies.

Updating a git package is slightly more nuanced. For packages *without* a pinned ref, the updater fetches the upstream branch and resets the working tree with `git reset --hard` followed by `git clean -fdx` (to restore a pristine state). For packages *with* a pinned ref, it fetches only that ref and resets to it — reconciling the local clone when the configured ref changes.

---

## Step 5 — How a package advertises its resources

Once a package is installed, the package manager needs to know which files inside it are extensions, which are skills, which are prompts, and which are themes. There are two discovery mechanisms, tried in order:

### The `xzy` manifest key

If the package's `package.json` contains a `"xzy"` key, that key is treated as the authoritative manifest:

```json
{
  "name": "my-extension-bundle",
  "version": "1.0.0",
  "xzy": {
    "extensions": ["src/my-tool.ts", "src/another-tool.ts"],
    "skills":     ["skills/"],
    "prompts":    ["prompts/review.md"],
    "themes":     []
  }
}
```

The manifest entries are paths or glob patterns, resolved relative to the package root. An empty array for a resource type explicitly disables all resources of that type.

The `PackageManifest` interface in the source captures this shape exactly:

```ts
interface PackageManifest {
  extensions?: string[];
  skills?: string[];
  prompts?: string[];
  themes?: string[];
}
```

### Conventional directories (auto-discovery fallback)

If there is no `xzy` key, the package manager looks for conventional subdirectories in the package root:

| Subdirectory | Resource type | File pattern |
|---|---|---|
| `extensions/` | extensions | `.ts` or `.js` files |
| `skills/` | skills | `SKILL.md` files |
| `prompts/` | prompts | `.md` files |
| `themes/` | themes | `.json` files |

If any of those directories exist, their contents are collected automatically. If none exist, the package contributes no resources.

For extensions specifically, there is a smarter resolution step:

1. Check if the directory itself has a `package.json` with an `extensions` manifest key or an `index.ts`/`index.js` file — if so, use that entry point.
2. Otherwise, scan subdirectories: for each subdirectory, check for its own `package.json`/`index.ts`/`index.js` and use those.
3. Fall back to collecting all `.ts`/`.js` files directly in the directory.

For skills, the discoverer looks for `SKILL.md` files — the standard skill entry-point marker.

---

## Step 6 — Resolving all resources at startup

When the agent starts, it calls `resolve()` on the package manager. This method:

1. Reads the package list from both user-scoped and project-scoped settings.
2. Deduplicates: if the same package appears in both, the project-scoped entry wins.
3. For each package: if it is already installed, collects its resources; if it is missing, optionally calls the `onMissing` callback to decide whether to install it now, skip it, or throw an error.
4. Collects locally-configured resource entries from settings (the `extensions`, `skills`, `prompts`, `themes` arrays that users can set directly, independent of packages).
5. Auto-discovers resources from conventional directories in both the user config dir (`~/.xzy/`) and the project config dir (`.xzy/`).
6. Returns a `ResolvedPaths` object with four arrays, one per resource type.

Each entry in a `ResolvedPaths` array is a `ResolvedResource`:

```ts
export interface ResolvedResource {
  path: string;
  enabled: boolean;
  metadata: PathMetadata;
}

export interface PathMetadata {
  source: string;
  scope: SourceScope;         // "user" | "project" | "temporary"
  origin: "package" | "top-level";
  baseDir?: string;
}
```

The `enabled` flag matters: it lets the settings layer disable individual resources within an installed package without uninstalling the package. Resources with `enabled: false` are loaded but not activated.

### Precedence when two sources provide the same file path

The resource accumulator uses a first-wins strategy keyed on canonical file path. The precedence ordering (highest to lowest) is:

| Rank | Description |
|---|---|
| 0 | Project scope + settings entry (`origin: "top-level"`, `source: "local"`, `scope: "project"`) |
| 1 | Project scope + auto-discovered (`origin: "top-level"`, `source: "auto"`, `scope: "project"`) |
| 2 | User scope + settings entry (`origin: "top-level"`, `source: "local"`, `scope: "user"`) |
| 3 | User scope + auto-discovered (`origin: "top-level"`, `source: "auto"`, `scope: "user"`) |
| 4 | Package resource (`origin: "package"`) |

Project-scoped entries always beat user-scoped entries; explicitly-configured entries beat auto-discovered ones; both beat resources that came from a package.

---

## Step 7 — Updating packages

The `update(source?)` method updates one named package or, when called with no arguments, all configured packages. It skips packages that are pinned to a specific npm version (because a pinned version is intentionally fixed). Git packages with a pinned ref are included — updating reconciles an existing clone when the configured ref has changed.

Updates run with concurrency limits to avoid overwhelming the network:

```ts
const NETWORK_TIMEOUT_MS = 10000;
const UPDATE_CHECK_CONCURRENCY = 4;
const GIT_UPDATE_CONCURRENCY = 4;
```

For npm packages, the updater first checks the latest published version with `npm view <name> version --json` and compares it to the installed version. It only installs if the versions differ — avoiding unnecessary reinstalls.

The entire update operation is skipped when offline mode is enabled (see [Step 9](#step-9--offline-mode-and-telemetry)).

---

## Step 8 — Config migrations

Now we move to a related but separate concern. As the agent evolves, its config schema changes: files get renamed, directories move, API key storage migrates from `settings.json` to `auth.json`, and env-var syntax tightens. If we just changed the schema and left users with old config files, the agent would silently ignore their settings or fail to start.

The solution is `runMigrations(cwd)` in `src/migrations.ts`, called once at startup. It runs a fixed set of one-time migrations in order and returns a summary:

```ts
export function runMigrations(cwd: string): {
  migratedAuthProviders: string[];
  deprecationWarnings: string[];
}
```

Here is the full sequence of migrations that run:

### 8a — Merging auth into `auth.json`

Previously, OAuth credentials lived in `oauth.json` and API keys lived inside `settings.json` under an `apiKeys` key. Both are now merged into a single `auth.json` file with a uniform shape.

The `migrateAuthToAuthJson()` function:

1. Skips the migration if `auth.json` already exists.
2. Reads `oauth.json` and records each entry as `{ type: "oauth", ... }` in a new `migrated` map; then renames the old file to `oauth.json.migrated`.
3. Reads `settings.json` and migrates any `apiKeys` entries into the same map as `{ type: "api_key", key }`, then removes the `apiKeys` property from `settings.json`.
4. If anything was collected, writes `auth.json` with permissions `0o600` (readable only by the owning user).

The function returns an array of provider names that were migrated — useful for showing the user what changed.

### 8b — Updating env-var syntax in config values

Older versions allowed raw environment variable names like `MY_API_KEY` as config values; the newer format requires an explicit `$` prefix: `$MY_API_KEY`. The `migrateExplicitEnvVarConfigValues` function walks both `auth.json` and `models.json`, detects any values matching the legacy pattern (via `isLegacyEnvVarNameConfigValue`), rewrites them with the `$` prefix, and prints a yellow warning listing each change:

```
Warning: Migrated API key/header environment references to explicit $ENV_VAR syntax.
  Plain strings will be treated as literals.
  - auth.json["openai"].key: MY_KEY -> $MY_KEY
```

### 8c — Moving misplaced session files

A bug in a prior release caused session files to be saved directly in `~/.xzy/` instead of `~/.xzy/sessions/<encoded-cwd>/`. The `migrateSessionsFromAgentRoot()` function finds any `.jsonl` files at the root of the agent directory, reads the first line of each to extract the `cwd` from the session header, computes the correct target directory using the same path-encoding logic as the session manager, and moves the file.

### 8d — Moving managed binaries

The `fd` and `rg` binaries (used for file search within the agent) used to live in a `tools/` subdirectory. They now live in `bin/`. The `migrateToolsToBin()` function moves them if found.

### 8e — Migrating keybindings config

The keybindings config format changed between versions. `migrateKeybindingsConfigFile()` reads `keybindings.json`, calls `migrateKeybindingsConfig` (from the keybindings module), and rewrites the file if any bindings needed updating.

### 8f — Renaming `commands/` to `prompts/`

The directory for user-defined prompts was renamed from `commands/` to `prompts/`. The `migrateCommandsToPrompts` helper renames the directory at both the user config level and the project config level, printing a confirmation when it does.

After this step, the function also checks for any surviving `hooks/` or `tools/` directories that contain user files. These directories were renamed in earlier versions, so if they still have content the user needs to move, the migration collects a deprecation warning.

### 8g — Deprecation warnings

If any deprecated directories were found, `runMigrations` returns them in `deprecationWarnings`. The caller can then display them with `showDeprecationWarnings(warnings)`, which prints each warning in yellow and waits for a keypress before continuing:

```ts
export async function showDeprecationWarnings(warnings: string[]): Promise<void> {
  // prints each warning, then prompts "Press any key to continue..."
}
```

This gives the user a chance to see the problem before the agent starts, without blocking it from starting altogether.

---

## Step 9 — Offline mode and telemetry

### Offline mode

Any operation that touches the network — `install`, `update`, fetching npm version info, pulling a git repo — first checks whether offline mode is enabled:

```ts
function isOfflineModeEnabled(): boolean {
  const value = process.env.XZY_OFFLINE;
  // (source uses the branded name; we genericise to XZY_OFFLINE)
  if (!value) return false;
  return value === "1" || value.toLowerCase() === "true" || value.toLowerCase() === "yes";
}
```

When `XZY_OFFLINE` is set to `1`, `true`, or `yes`, all network operations become no-ops. Update checks return empty results. Install of a missing package is skipped. The agent continues with whatever is already on disk.

This is useful in CI environments or air-gapped machines where the network is unavailable.

### Install telemetry

The agent can optionally send an anonymous signal when packages are installed or updated — a version-check ping to a configured endpoint. The telemetry decision is made by `isInstallTelemetryEnabled` in `src/core/telemetry.ts`:

```ts
export function isInstallTelemetryEnabled(
  settingsManager: SettingsManager,
  telemetryEnv: string | undefined = process.env.XZY_TELEMETRY,
  // (source uses the branded env var name; we genericise to XZY_TELEMETRY)
): boolean {
  return telemetryEnv !== undefined
    ? isTruthyEnvFlag(telemetryEnv)
    : settingsManager.getEnableInstallTelemetry();
}
```

The logic is: if the `XZY_TELEMETRY` environment variable is set, it takes precedence over whatever is in settings. If it is not set, the settings value is used.

To opt out:

```bash
# Disable telemetry for this session
XZY_TELEMETRY=0 xzy "your prompt"

# Disable telemetry permanently (add to your shell profile)
export XZY_TELEMETRY=0
```

To opt in explicitly (overriding a settings value of false):

```bash
XZY_TELEMETRY=1 xzy "your prompt"
```

Accepted truthy values for both `XZY_OFFLINE` and `XZY_TELEMETRY` are `"1"`, `"true"`, and `"yes"` (case-insensitive). Any other value (including `"0"`, `"false"`, `"no"`) disables the feature.

<!-- GAP: The actual telemetry endpoint URL and payload shape are not described in S90 (source is silent on what data is sent and where). The chapter therefore describes only the opt-out pattern. -->

---

## Putting it together: a package from scratch to install

Let's walk through the full lifecycle of building and distributing a simple extension package.

**1. Create the package directory and write your extension:**

```
my-xzy-package/
  package.json
  extensions/
    hello.ts
  skills/
    SKILL.md
```

**2. Add the `xzy` manifest key to `package.json`:**

```json
{
  "name": "my-xzy-package",
  "version": "1.0.0",
  "xzy": {
    "extensions": ["extensions/hello.ts"],
    "skills":     ["skills/"]
  }
}
```

Without the `xzy` key, the package manager falls back to conventional directory discovery and would find the same files automatically. The `xzy` key gives you explicit control.

**3. Publish to npm (or push to a git host).**

**4. A user installs it:**

```bash
# Install from npm and persist to user settings
xzy install npm:my-xzy-package

# Install from a git repository
xzy install github.com/you/my-xzy-package

# Install scoped to the current project
xzy install npm:my-xzy-package --local
```

**5. The agent resolves it at startup.** The package manager finds `my-xzy-package` in settings, verifies the install path, reads the `xzy` manifest, and adds `extensions/hello.ts` and the skills to `ResolvedPaths`. Those paths are passed to the extension loader and skill loader respectively.

**6. To update:**

```bash
xzy update npm:my-xzy-package   # update one package
xzy update                      # update all non-pinned packages
```

**7. To remove:**

```bash
xzy remove npm:my-xzy-package
```

---

## Reference: `listConfiguredPackages`

The `listConfiguredPackages()` method returns every package currently in settings, from both scopes:

```ts
export interface ConfiguredPackage {
  source: string;
  scope: "user" | "project";
  filtered: boolean;       // true when the settings entry is an object with filter patterns
  installedPath?: string;  // undefined if not yet installed on disk
}
```

This is what a `xzy packages list` command would display.

---

← Previous: [The SDK: createAgentSession and Programmatic Use](./sdk-and-programmatic-use.md) · Next: [The Extension API: Handlers, Context, and Events](../extensions/extension-api-and-types.md) →
