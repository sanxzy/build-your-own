---
title: "The Adapter Registry"
description: Build a mutable Map-backed adapter registry that lets the orchestrator look up any SwarmAdapter by name at runtime, with support for runtime plugin registration.
category: the-agent
type: tutorial
tags: [adapter registry, register, unregister, requireAdapter, runtime plugin, list adapters, find adapter, provider registry, dynamic registration, plugin system, assertKnownAdapterType, mutable registry, ServerAdapterModule, registerAdapter, unregisterAdapter, findAdapter, listAdapters, getAdapter, SwarmAdapter, invoke, status, cancel]
keywords: [adapter lookup, runtime truth, open-ended adapter type, plugin adapter, LLM provider registry, registerApiProvider, unregisterApiProviders, ApiProvider, Map registry, dynamic adapter, adapter override, builtin fallback, source of truth]
sources: [S26, S40, S45]
---

**TL;DR** — We now have four adapters (mock, LLM, process, and HTTP), each implementing the `SwarmAdapter` contract (`invoke` / `status` / `cancel`) defined in the previous chapter. But the orchestrator has no way to select one by name at runtime. This chapter builds a mutable `Map`-backed registry with `register` / `unregister` / `get` / `list` / `find` / `require` operations, explains why the registry — not a hardcoded enum — must be the source of truth for valid adapter types, and shows the exact same pattern applied to LLM providers. By the end you will be able to register adapters at startup, add a plugin adapter without touching core code, and get a clear error when an unknown type is requested.

# The Adapter Registry

We have built four adapters. Each satisfies the `SwarmAdapter` interface defined in [The Adapter Interface](./adapter-interface.md): three methods (`invoke`, `status`, `cancel`) plus the associated result types. As a quick recap: `invoke` starts an agent execution window, `status` checks whether a run is still in progress, and `cancel` stops it. Every backend — mock, LLM, process, HTTP — exposes exactly these three methods, so the orchestrator can drive any of them identically.

What we are missing is a lookup table. The orchestrator needs to answer a question at the start of every agent run: "this agent is configured with adapter type `claude-cli` — give me that adapter." Right now there is no mechanism to do that. Let's build one.

## The problem with a hardcoded list

The obvious first move is a plain object or a `switch` statement:

```ts
// Tempting, but fragile
function getAdapter(type: string): SwarmAdapter {
  if (type === "mock") return mockAdapter;
  if (type === "llm") return llmAdapter;
  if (type === "process") return processAdapter;
  if (type === "http") return httpAdapter;
  throw new Error(`Unknown adapter: ${type}`);
}
```

This works until you want to add a new adapter — or let an external plugin contribute one — without editing the core. The moment someone ships an adapter for a new AI tool, they should not need a pull request to the orchestrator's core module. We need the registry to be *mutable at runtime*.

## The registered entry shape

The registry stores more than a bare `SwarmAdapter`. Alongside the three-method contract, a registered entry carries metadata the orchestrator uses at runtime: the type string that keys the map, optional model lists for UI dropdowns, environment-check support, and flags that tell the orchestrator how to treat this adapter.

We call this registered shape `ServerAdapterModule`. It wraps a `SwarmAdapter` implementation alongside that metadata:

```ts
// Simplified view of the registered shape (from S26)
// Each registered entry pairs a type string with a SwarmAdapter plus
// optional metadata this chapter introduces.
interface ServerAdapterModule {
  // The type string used as the registry key — e.g. "claude-cli", "mock"
  type: string;

  // The core SwarmAdapter methods (from The Adapter Interface, P4):
  //   invoke(agent, context): Promise<AdapterResult>  — start the work
  //   status(run): Promise<RunStatus>                 — check progress
  //   cancel(run): Promise<void>                      — stop the run
  invoke(agent: AdapterAgent, context: InvocationContext): Promise<AdapterResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;

  // Optional registry metadata (not from P4; introduced here on this page):
  models?: AdapterModel[];           // static model list for selection UIs
  listModels?: () => Promise<AdapterModel[]>; // dynamic model discovery
  testEnvironment?(ctx: AdapterEnvironmentTestContext): Promise<AdapterEnvironmentTestResult>;
  supportsLocalAgentJwt?: boolean;   // whether this adapter accepts an agent JWT
  supportsInstructionsBundle?: boolean; // whether to pass a managed AGENTS.md
  requiresMaterializedRuntimeSkills?: boolean; // whether skills must be written to disk first
}
```

Three things to note here:

- The `invoke` / `status` / `cancel` methods are the `SwarmAdapter` contract from P4. They are mandatory on every registered entry. If you implement a `SwarmAdapter` and want to register it, you add `type` plus the optional metadata fields to get a `ServerAdapterModule`.
- `models` and `listModels` are optional registry metadata — they do not belong to the `SwarmAdapter` contract and were not discussed in [The Adapter Interface](./adapter-interface.md). They exist so the server can serve model lists without knowing adapter internals.
- `testEnvironment` is a convenience hook for pre-flight checks (does the CLI exist? are credentials set?). Also not part of `SwarmAdapter` — it is a registry extension.

You can think of `ServerAdapterModule` as: *"a `SwarmAdapter` plus a name tag and optional discovery helpers."*

## Building the mutable registry

The heart of the registry is a single `Map<string, ServerAdapterModule>`. Adapter type strings are the keys; the full module objects are the values. On module load we immediately populate it with built-ins, and from then on the map is the single source of truth.

```ts
// src/adapters/registry.ts  (simplified view — production adds more fields)
import type { ServerAdapterModule } from "./types.js";
import { processAdapter } from "./process/index.js";
import { httpAdapter } from "./http/index.js";

// The live registry: type string → registered adapter module
const adaptersByType = new Map<string, ServerAdapterModule>();

// When an external adapter overrides a built-in, we stash the original here
// so we can restore it if the override is later deactivated.
const builtinFallbacks = new Map<string, ServerAdapterModule>();

function registerBuiltInAdapters(): void {
  for (const adapter of [processAdapter, httpAdapter /* … */]) {
    adaptersByType.set(adapter.type, adapter);
  }
}

registerBuiltInAdapters();
```

Three things to notice:

- `adaptersByType` is module-level. It is allocated once and shared across the whole server process. That is deliberate: the registry is a singleton because the process has one set of available adapters.
- `builtinFallbacks` stores the original built-in whenever an external adapter overrides a known type. This makes it possible to restore the built-in if the external is later removed.
- `registerBuiltInAdapters()` is called immediately as the module initialises, so the built-ins are always present from the first moment the registry is imported.

## The five core operations

Now we add the public API on top of that map. Let's go operation by operation.

### `registerAdapter` — add or override

```ts
// S26 — registerServerAdapter, adapted to our generic naming
export function registerAdapter(adapter: ServerAdapterModule): void {
  // If this type was previously a built-in and we don't already have a fallback,
  // stash the original before we overwrite it.
  if (BUILTIN_ADAPTER_TYPES.has(adapter.type) && !builtinFallbacks.has(adapter.type)) {
    const existing = adaptersByType.get(adapter.type);
    if (existing) {
      builtinFallbacks.set(adapter.type, existing);
    }
  }
  adaptersByType.set(adapter.type, adapter);
}
```

You can call `registerAdapter` at startup (for built-ins) or at any point during the server's lifetime (for plugins). When an incoming adapter has the same `type` as a known built-in, the original is saved to `builtinFallbacks` first. That lets `unregisterAdapter` cleanly restore the built-in later.

### `unregisterAdapter` — remove or restore

Unregistering is more nuanced than deleting from the map:

```ts
// S26 — unregisterServerAdapter, adapted
export function unregisterAdapter(type: string): void {
  // The generic process and HTTP adapters are structural — they cannot be removed.
  if (type === processAdapter.type || type === httpAdapter.type) return;

  // If this type overrode a built-in, restore the built-in instead of deleting.
  if (builtinFallbacks.has(type)) {
    const fallback = builtinFallbacks.get(type)!;
    adaptersByType.set(type, fallback);
    builtinFallbacks.delete(type);
    return;
  }

  // A pure built-in cannot be removed through this API.
  if (BUILTIN_ADAPTER_TYPES.has(type)) return;

  // A third-party external type is simply removed.
  adaptersByType.delete(type);
}
```

There are three cases:

| Case | Behaviour |
|---|---|
| `type` is `process` or `http` | No-op — these are the fallback adapters the registry always needs |
| `type` overrode a built-in | Restore the original built-in from `builtinFallbacks` |
| `type` is a pure third-party | Remove it from the map |

The asymmetry between "pure built-in" and "external-overriding-a-built-in" is intentional: a pure built-in (one that was never overridden) cannot be deleted — that would leave agents referencing it with no adapter to run against.

### `findAdapter` and `getAdapter` — soft and hard lookup

We want two flavours of lookup: one that returns `null` for unknown types (useful for validation), and one that falls back gracefully:

```ts
// S26 — findServerAdapter
export function findAdapter(type: string): ServerAdapterModule | null {
  return adaptersByType.get(type) ?? null;
}

// S26 — getServerAdapter (fallback to processAdapter for unknowns)
export function getAdapter(type: string): ServerAdapterModule {
  return adaptersByType.get(type) ?? processAdapter;
}
```

`findAdapter` returns `null` when the type is not in the registry — useful in route handlers that want to respond with a 404 rather than throwing. `getAdapter` always returns *something*: it falls back to `processAdapter` for unknown types, which is the most generic adapter (it launches any shell command). You would use `getAdapter` when the code has already validated the type and just needs the module.

### `requireAdapter` — fail loudly on unknown types

Sometimes you want the call to throw if the type is not registered — for example, in a code path that should only be reached after validation has already confirmed the type exists:

```ts
// S26 — requireServerAdapter, adapted
export function requireAdapter(type: string): ServerAdapterModule {
  const adapter = findAdapter(type);
  if (!adapter) {
    throw new Error(`Unknown adapter type: ${type}`);
  }
  return adapter;
}
```

This is a narrow wrapper over `findAdapter`. It throws with a clear message rather than returning `null`. Call `requireAdapter` in execution paths; call `findAdapter` in validation paths.

### `listAdapters` — enumerate all registered adapters

```ts
// S26 — listServerAdapters
export function listAdapters(): ServerAdapterModule[] {
  return Array.from(adaptersByType.values());
}
```

`listAdapters` returns every currently-registered adapter. This feeds API endpoints (so a UI can populate a "choose your adapter" dropdown) and health-check routes.

## Why the registry is the source of truth for valid types

Here is a question you might have: if we already have a TypeScript type union listing the built-in adapter types, why not validate against that union?

The answer is that a type union is frozen at compile time. As soon as someone registers a plugin adapter at runtime — an adapter whose type string was not in the original union — the union-based check incorrectly rejects it.

The solution, from S40, is to make input schemas accept *any non-empty string* for adapter type and let the registry perform the actual check at request time. There is a server-side function, `assertKnownAdapterType`, that awaits the external-adapters loading promise (plugins are loaded asynchronously at startup) and then confirms the type is present in the live map:

```ts
// S40 / S26 — assertKnownAdapterType pattern (simplified view)
async function assertKnownAdapterType(type: string): Promise<void> {
  // Wait for any async plugin adapters to finish registering.
  await waitForExternalAdapters();
  const adapter = findAdapter(type);
  if (!adapter) {
    throw new Error(`Adapter type "${type}" is not registered`);
  }
}
```

<!-- GAP: the exact file path and full body of assertKnownAdapterType in the source (it lives in server/src/routes/agents.ts per S40, not in the registry module itself); S26/S40 describe it but the routes file was not assigned — needed to show the complete function. -->

The `waitForExternalAdapters()` call is important. Plugin adapters are loaded asynchronously at startup (they may require disk I/O to discover installed packages). Without awaiting that promise, a request that arrives during the brief load window would see an incomplete registry and incorrectly reject a valid external type.

This is the architecture S40 describes as "server as the real source of truth": the shared input schema stays open-ended (any non-empty string passes validation), and the server registry decides what is *actually* acceptable at runtime.

## Registering adapters at startup

With the registry in hand, here is what the startup sequence looks like. We register built-ins first (synchronously), then load any external plugin adapters (asynchronously):

```ts
// src/adapters/registry.ts — startup registration (simplified)
import { mockAdapter }    from "./mock/index.js";
import { llmAdapter }     from "./llm/index.js";
import { processAdapter } from "./process/index.js";
import { httpAdapter }    from "./http/index.js";

// Built-ins registered synchronously at module load time.
for (const adapter of [mockAdapter, llmAdapter, processAdapter, httpAdapter]) {
  adaptersByType.set(adapter.type, adapter);
}

// External plugins loaded asynchronously.
// `buildExternalAdapters()` discovers installed plugin packages and calls
// each package's factory function to produce a ServerAdapterModule.
const externalAdaptersReady: Promise<void> = (async () => {
  const externals = await buildExternalAdapters();
  for (const ext of externals) {
    registerAdapter(ext);
  }
})();

export function waitForExternalAdapters(): Promise<void> {
  return externalAdaptersReady;
}
```

The adapters we have built in the preceding chapters map to these registrations:

| Chapter | Adapter type | Recap |
|---|---|---|
| [The Mock Adapter](./mock-adapter.md) | `mock` | Deterministic fake; `invoke` returns predictable output for tests; `status` always returns `"succeeded"` |
| [The LLM Adapter](./llm-adapter.md) | `llm` | `invoke` calls a real LLM provider over HTTP; streams token events |
| [Process and HTTP Adapters](./process-and-http-adapters.md) | `process` | `invoke` spawns a local subprocess (e.g. `claude`, `codex`) |
| [Process and HTTP Adapters](./process-and-http-adapters.md) | `http` | `invoke` delegates to an external HTTP endpoint |

Once these are registered, the orchestrator can look up any of them by name.

## Looking up an adapter from an agent's config

Let's put the whole registry to work. When the orchestrator picks up an agent run request, it reads the agent's `adapterType` field and retrieves the module. Because every `ServerAdapterModule` implements the `SwarmAdapter` contract, we can call `invoke` on whatever comes back:

```ts
// src/orchestrator/run-agent.ts (simplified)
import { requireAdapter, waitForExternalAdapters } from "../adapters/registry.js";

export async function startAgentRun(agentConfig: AgentConfig): Promise<void> {
  // Ensure all external adapters have had a chance to register before we look up.
  await waitForExternalAdapters();

  // requireAdapter throws with a clear message if the type is unknown.
  const adapter = requireAdapter(agentConfig.adapterType);

  // Every registered adapter satisfies SwarmAdapter, so we call invoke()
  // regardless of which backend is behind this type string.
  const result = await adapter.invoke(
    { id: agentConfig.id, name: agentConfig.name, adapterConfig: agentConfig.adapterConfig },
    { runId: newRunId(), runtime: agentConfig.runtime, config: {}, context: {} },
  );

  return result;
}
```

The code is clean because the registry hides the `Map` behind a typed function. The orchestrator never needs to know which module file backs `claude-cli` — it holds a string and the registry does the rest.

## The same pattern for LLM providers

The adapter registry solves the runtime-selection problem for *agents*. But there is a parallel problem one layer down: the LLM adapter (from [The LLM Adapter](./llm-adapter.md)) needs to select a *provider* (Anthropic, OpenAI, Google, etc.) at runtime based on a model's `api` field. The solution is identical in structure.

S45 shows an `apiProviderRegistry` that is a `Map<string, RegisteredApiProvider>`:

```ts
// src/llm/api-registry.ts — provider registry (from S45, slightly simplified)
import type { Api, Model, Context, StreamOptions } from "./types.js";

export interface ApiProvider<TApi extends Api = Api> {
  api: TApi;                  // the provider key, e.g. "anthropic" | "openai"
  stream: StreamFunction<TApi, StreamOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}

// The live map: provider key → wrapped provider
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

export function registerApiProvider<TApi extends Api>(
  provider: ApiProvider<TApi>,
  sourceId?: string,           // optional tag for grouped unregistration
): void {
  apiProviderRegistry.set(provider.api, {
    provider: {
      api: provider.api,
      stream: wrapStream(provider.api, provider.stream),
      streamSimple: wrapStreamSimple(provider.api, provider.streamSimple),
    },
    sourceId,
  });
}

export function getApiProvider(api: Api): ApiProviderInternal | undefined {
  return apiProviderRegistry.get(api)?.provider;
}

export function getApiProviders(): ApiProviderInternal[] {
  return Array.from(apiProviderRegistry.values(), (e) => e.provider);
}

export function unregisterApiProviders(sourceId: string): void {
  // Remove every provider registered under this sourceId.
  for (const [api, entry] of apiProviderRegistry.entries()) {
    if (entry.sourceId === sourceId) {
      apiProviderRegistry.delete(api);
    }
  }
}

export function clearApiProviders(): void {
  apiProviderRegistry.clear();
}
```

The `wrapStream` and `wrapStreamSimple` helpers (not shown in full here) add a guard: before delegating to the provider's streaming function, they verify that the model's `api` field matches the provider's declared `api`. If it does not, they throw a `Mismatched api` error immediately, catching misconfiguration before any network call is made.

Notice two deliberate differences from the adapter registry:

| Feature | Adapter registry | Provider registry |
|---|---|---|
| Keyed by | `adapter.type` (string) | `provider.api` (string) |
| Unregister strategy | By type, one-at-a-time; restore built-in if applicable | By `sourceId`, removing all providers from one source at once |
| Fallback on miss | Falls back to `processAdapter` | Returns `undefined` |

The `sourceId` on providers is especially useful for plugins that contribute multiple providers from a single package. When the plugin is unloaded, `unregisterApiProviders(pluginId)` removes all of them in one call. The adapter registry does not need this because each adapter has exactly one type, so one-at-a-time unregistration is sufficient.

### Registering a provider at startup

```ts
// src/llm/providers/anthropic.ts (illustrative — wiring the Anthropic provider)
import { registerApiProvider } from "../api-registry.js";
import type { AnthropicApi } from "../types.js";

registerApiProvider({
  api: "anthropic" as AnthropicApi,
  stream: streamFromAnthropic,
  streamSimple: streamSimpleFromAnthropic,
});
```

Each provider package calls `registerApiProvider` once when its module loads. The LLM adapter then calls `getApiProvider(model.api)` to retrieve the right streaming function for the model it has been asked to invoke.

## How the two registries work together

Let's trace a complete path from agent config to LLM call to see both registries in play:

```
AgentConfig { adapterType: "llm", modelConfig: { api: "anthropic", model: "claude-opus-4" } }
         │
         ▼
  adapterRegistry.requireAdapter("llm")
         │ returns the LLM adapter (a ServerAdapterModule with invoke/status/cancel)
         ▼
  llmAdapter.invoke(agent, context)
         │
         ▼
  apiProviderRegistry.getApiProvider("anthropic")
         │ returns the Anthropic streaming function
         ▼
  stream(model, context, options) → AssistantMessageEventStream
```

The adapter registry answers "which execution strategy?"; the provider registry answers "which LLM service?". Each registry is a `Map`, each uses the same register/find/require shape, and neither needs to know the other exists.

## Try it yourself

With the registry in place, here are three exercises that test different parts of the design.

**Exercise 1 — Register a second provider profile.**

Write a function that registers an OpenAI provider alongside the Anthropic one:

```ts
import { registerApiProvider } from "../llm/api-registry.js";

registerApiProvider({
  api: "openai",
  stream: streamFromOpenAI,
  streamSimple: streamSimpleFromOpenAI,
});
```

Then call `getApiProviders()` and confirm the returned array has two entries. Try calling `getApiProvider("openai")` and `getApiProvider("anthropic")` and verify each returns the correct module.

**Exercise 2 — Write a tiny plugin adapter and register it at runtime.**

A plugin adapter is any object satisfying `ServerAdapterModule`. Create a minimal one and register it after startup:

```ts
import { registerAdapter, findAdapter } from "../adapters/registry.js";
import type { ServerAdapterModule } from "../adapters/types.js";

const myPluginAdapter: ServerAdapterModule = {
  type: "my-plugin",
  // SwarmAdapter methods — the core contract from The Adapter Interface:
  invoke: async (agent, context) => {
    // Your custom execution logic here.
    return { exitCode: 0, signal: null, timedOut: false };
  },
  status: async (run) => "succeeded",
  cancel: async (run) => { /* stop the run */ },
  // Optional metadata:
  models: [],
  supportsLocalAgentJwt: false,
  supportsInstructionsBundle: false,
  requiresMaterializedRuntimeSkills: false,
};

registerAdapter(myPluginAdapter);

// Confirm it is now findable:
console.assert(findAdapter("my-plugin") === myPluginAdapter);
```

Then unregister it and confirm `findAdapter("my-plugin")` returns `null`.

**Exercise 3 — Make `requireAdapter` produce a helpful error.**

Call `requireAdapter` with a type string that has never been registered:

```ts
import { requireAdapter } from "../adapters/registry.js";

try {
  requireAdapter("does-not-exist");
} catch (err) {
  console.log(err.message); // Should print: Unknown adapter type: does-not-exist
}
```

Notice the message includes the unknown type string. Extend the error to also list the currently-registered types by calling `listAdapters().map(a => a.type)` — this gives a user debugging a misconfigured agent a clear indication of what types *are* available.

---

← Previous: [Process and HTTP Adapters](./process-and-http-adapters.md) · Next: [A Run: Sessions, Usage, and Cost](./a-run.md) →
