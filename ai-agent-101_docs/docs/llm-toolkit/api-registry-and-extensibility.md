---
title: "The API Registry: Registering and Looking Up Providers"
description: How llm-toolkit's provider registry works — registering adapters by API id, lazy loading built-ins, resolving env-var API keys, and plugging in a custom provider.
category: llm-toolkit
type: tutorial
tags: [api registry, registerApiProvider, getApiProvider, extensibility, lazy registration, built-in providers, custom provider, llm-toolkit, provider registry, unregisterApiProviders, clearApiProviders, getApiProviders, ApiProvider, Api, env-var key resolution, findEnvKeys, getEnvApiKey]
keywords: [register provider, lookup provider, provider map, lazy import, dynamic import, api key environment variable, custom adapter, plug in provider, streaming registry]
sources: [S9, S15, S18]
---

**TL;DR** — `llm-toolkit` routes every streaming call through a central `Map`-backed registry. Providers register themselves under a string API id; the streaming front door looks them up at call time. Built-in providers are registered lazily — their modules are not imported until first use. By the end of this chapter you will understand how the registry works, why built-ins use lazy loading, how each provider resolves its API key from the environment, and how to add your own custom provider without touching `llm-toolkit`'s source.

# The API Registry: Registering and Looking Up Providers

## The problem: we have adapters, but no way to find them

In the previous two chapters we built the provider adapters — the translation layers that turn a generic streaming request into the wire format each LLM API expects, and turn vendor-specific server-sent events back into a uniform event stream:

- The Anthropic and OpenAI adapters ([provider adapters: Anthropic and OpenAI](./provider-adapters-anthropic-and-openai.md)) each expose a `stream` and `streamSimple` function for their respective API families.
- The Google Gemini and faux adapters ([provider adapters: Google and faux](./provider-adapters-google-and-faux.md)) follow the same shape.

Now we have a practical problem: when the streaming front door receives a call for, say, `"google-generative-ai"`, how does it find the right adapter? And when someone builds on top of `llm-toolkit` and wants to add a provider we've never heard of, how do they wire that in without forking the library?

The answer is the **API registry** — a central lookup table that maps an API id string to its adapter. Let's build an understanding of it from the ground up.

## Step 1 — The registry data structure

The registry lives in `api-registry.ts`. At its core it is a `Map` keyed by a string we call an **API id** — a short identifier like `"anthropic-messages"` or `"openai-completions"` that names the wire protocol a particular adapter speaks. The value stored for each key is a wrapper object that holds the adapter's two streaming functions.

```ts
// Simplified view of api-registry.ts — the raw storage
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();
```

`RegisteredApiProvider` is an internal type that bundles the adapter and an optional `sourceId` (a string callers can use to group and remove providers together — more on that shortly):

```ts
// Internal shape — not exported
type RegisteredApiProvider = {
  provider: ApiProviderInternal;  // the normalised stream + streamSimple functions
  sourceId?: string;              // optional group tag for bulk removal
};
```

You won't use `RegisteredApiProvider` directly — it is an implementation detail. What you interact with is the public-facing `ApiProvider` interface:

```ts
export interface ApiProvider<
  TApi extends Api = Api,
  TOptions extends StreamOptions = StreamOptions
> {
  api: TApi;             // the API id string, e.g. "anthropic-messages"
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}
```

`TApi` is a generic parameter constrained to `Api` — the union of all known API id strings (defined in `types.ts`). For built-in providers `TApi` is one of the named members of that union; for a custom provider you can pass any `string` that is assignable to `Api`.

## Step 2 — Registering a provider

To add an adapter to the registry, call `registerApiProvider`:

```ts
export function registerApiProvider<
  TApi extends Api,
  TOptions extends StreamOptions
>(
  provider: ApiProvider<TApi, TOptions>,
  sourceId?: string,
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
```

There are two things worth noticing here.

**API id as key.** `provider.api` is the key in the map. If you call `registerApiProvider` twice with the same `api` value, the second registration replaces the first. That is intentional — it lets you override a built-in with a custom implementation.

**The `wrapStream` safety net.** Rather than storing the raw `stream` function, `registerApiProvider` wraps it using `wrapStream` (and the matching `wrapStreamSimple`). The wrapper does one thing: it checks that the `model.api` field at call time matches the API id the adapter was registered for, and throws a descriptive error if they do not match:

```ts
// Simplified view of wrapStream
function wrapStream(api, stream) {
  return (model, context, options) => {
    if (model.api !== api) {
      throw new Error(`Mismatched api: ${model.api} expected ${api}`);
    }
    return stream(model, context, options);
  };
}
```

This guard means a routing mistake (asking the Anthropic adapter to handle a Google model) surfaces immediately with a clear error rather than producing a confusing API response.

## Step 3 — Looking up a provider

There are three lookup functions, each for a different use-case:

```ts
// Look up a single provider by API id.
// Returns undefined if no provider is registered for that id.
export function getApiProvider(api: Api): ApiProviderInternal | undefined {
  return apiProviderRegistry.get(api)?.provider;
}

// List every registered provider (useful for debugging/introspection).
export function getApiProviders(): ApiProviderInternal[] {
  return Array.from(apiProviderRegistry.values(), (entry) => entry.provider);
}
```

The streaming front door (covered in the earlier chapter on the `stream` function) calls `getApiProvider(model.api)` to retrieve the right adapter at call time. If the result is `undefined` — meaning no provider was registered for the requested API id — the caller receives `undefined` and must handle it (typically by throwing a user-facing error about an unconfigured provider).

## Step 4 — The problem with eager imports

We could stop here: each provider module imports its adapter and calls `registerApiProvider` at the top level of `api-registry.ts`. But that creates an immediate startup cost problem.

`llm-toolkit` ships adapters for nine different APIs: Anthropic, OpenAI Completions, OpenAI Responses, OpenAI Codex Responses, Azure OpenAI Responses, Google Generative AI, Google Vertex AI, Mistral, and Amazon Bedrock. If we eagerly import all nine adapter modules the moment the library loads, we pay the parse and initialisation cost of every SDK dependency up front — even if the user only ever calls Anthropic models. For a CLI tool that starts fresh on every invocation, that startup cost is felt on every run.

So we use **lazy loading**: register each built-in provider with a wrapper that imports the real adapter module only when the first streaming call arrives for that API id.

## Step 5 — Lazy registration in `register-builtins.ts`

The lazy loading machinery lives in `register-builtins.ts`. Let's walk through how it works for the Anthropic provider — the pattern is identical for every built-in.

First, a module-level variable holds the in-flight import promise:

```ts
let anthropicProviderModulePromise:
  | Promise<LazyProviderModule<"anthropic-messages", AnthropicOptions, SimpleStreamOptions>>
  | undefined;
```

`LazyProviderModule` is a thin interface that describes what we need from the adapter module — just `stream` and `streamSimple`:

```ts
interface LazyProviderModule<TApi, TOptions, TSimpleOptions> {
  stream: (model: Model<TApi>, context: Context, options?: TOptions) => AsyncIterable<AssistantMessageEvent>;
  streamSimple: (model: Model<TApi>, context: Context, options?: TSimpleOptions) => AsyncIterable<AssistantMessageEvent>;
}
```

A loader function triggers the dynamic import the first time it is called, then caches the resulting promise so all subsequent callers share the same module load:

```ts
function loadAnthropicProviderModule(): Promise<
  LazyProviderModule<"anthropic-messages", AnthropicOptions, SimpleStreamOptions>
> {
  anthropicProviderModulePromise ||= import("./anthropic.ts").then((module) => {
    return {
      stream: module.streamAnthropic,
      streamSimple: module.streamSimpleAnthropic,
    };
  });
  return anthropicProviderModulePromise;
}
```

Notice the `||=` assignment: if `anthropicProviderModulePromise` is already set (from a previous call), we re-use it. The import happens at most once per process lifetime.

Now we need a `stream` function that we can hand to `registerApiProvider` today, even though the real module won't be loaded until the first call. `createLazyStream` builds this wrapper:

```ts
// Simplified view of createLazyStream
function createLazyStream(loadModule) {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();  // an immediately-returnable event stream

    loadModule()
      .then((module) => {
        const inner = module.stream(model, context, options);
        forwardStream(outer, inner);  // pipe events from the real adapter into outer
      })
      .catch((error) => {
        const message = createLazyLoadErrorMessage(model, error);
        outer.push({ type: "error", reason: "error", error: message });
        outer.end(message);
      });

    return outer;  // returned synchronously — events appear as the real adapter produces them
  };
}
```

The key insight: `AssistantMessageEventStream` (the push-based event stream covered in earlier chapters) can be returned immediately and then filled in asynchronously. The caller starts iterating the outer stream; events flow in once the module loads and the real adapter starts emitting. If the module fails to load, an error event is pushed and the stream closes cleanly.

`createLazySimpleStream` is the same pattern for `streamSimple`.

Putting it together, the exported lazy wrappers look like this:

```ts
export const streamAnthropic = createLazyStream(loadAnthropicProviderModule);
export const streamSimpleAnthropic = createLazySimpleStream(loadAnthropicProviderModule);

// ... and one pair for each of the other eight built-in providers
```

Finally, `registerBuiltInApiProviders` calls `registerApiProvider` once for every built-in, passing the lazy wrappers rather than the real adapter functions:

```ts
export function registerBuiltInApiProviders(): void {
  registerApiProvider({
    api: "anthropic-messages",
    stream: streamAnthropic,
    streamSimple: streamSimpleAnthropic,
  });

  registerApiProvider({
    api: "openai-completions",
    stream: streamOpenAICompletions,
    streamSimple: streamSimpleOpenAICompletions,
  });

  registerApiProvider({
    api: "mistral-conversations",
    stream: streamMistral,
    streamSimple: streamSimpleMistral,
  });

  registerApiProvider({
    api: "openai-responses",
    stream: streamOpenAIResponses,
    streamSimple: streamSimpleOpenAIResponses,
  });

  registerApiProvider({
    api: "azure-openai-responses",
    stream: streamAzureOpenAIResponses,
    streamSimple: streamSimpleAzureOpenAIResponses,
  });

  registerApiProvider({
    api: "openai-codex-responses",
    stream: streamOpenAICodexResponses,
    streamSimple: streamSimpleOpenAICodexResponses,
  });

  registerApiProvider({
    api: "google-generative-ai",
    stream: streamGoogle,
    streamSimple: streamSimpleGoogle,
  });

  registerApiProvider({
    api: "google-vertex",
    stream: streamGoogleVertex,
    streamSimple: streamSimpleGoogleVertex,
  });

  registerApiProvider({
    api: "bedrock-converse-stream",
    stream: streamBedrockLazy,
    streamSimple: streamSimpleBedrockLazy,
  });
}
```

The file ends with a single top-level call:

```ts
registerBuiltInApiProviders();
```

This runs the moment `register-builtins.ts` is imported — so all nine providers are entered into the registry with their lazy wrappers, but none of the heavy adapter modules are loaded yet.

### Summary: all nine built-in API ids

| API id | Provider |
|---|---|
| `anthropic-messages` | Anthropic (Messages API) |
| `openai-completions` | OpenAI Chat Completions |
| `openai-responses` | OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses |
| `azure-openai-responses` | Azure OpenAI Responses |
| `mistral-conversations` | Mistral AI |
| `google-generative-ai` | Google Generative AI (Gemini) |
| `google-vertex` | Google Cloud Vertex AI |
| `bedrock-converse-stream` | Amazon Bedrock |

### The Bedrock special case

You might notice that Bedrock has an extra mechanism: `setBedrockProviderModule`. Because the Bedrock SDK is a Node.js-only package, the loader uses `importNodeOnlyProvider` instead of a bare `import()`. Additionally, the file exposes `setBedrockProviderModule` so a caller can inject a pre-built module directly — for example, in test environments where dynamic import of native Node modules is inconvenient:

```ts
export function setBedrockProviderModule(module: BedrockProviderModule): void {
  bedrockProviderModuleOverride = {
    stream: module.streamBedrock,
    streamSimple: module.streamSimpleBedrock,
  };
}
```

When `bedrockProviderModuleOverride` is set, `loadBedrockProviderModule` returns `Promise.resolve(bedrockProviderModuleOverride)` instead of triggering a dynamic import.

### Resetting the registry

Two functions handle cleanup:

```ts
// Remove all providers registered under a given sourceId
export function unregisterApiProviders(sourceId: string): void { ... }

// Wipe every registered provider
export function clearApiProviders(): void { ... }
```

And `register-builtins.ts` exposes a convenience function that wipes and re-registers everything:

```ts
export function resetApiProviders(): void {
  clearApiProviders();
  registerBuiltInApiProviders();
}
```

This is useful in tests that need to start from a clean slate.

## Step 6 — How providers find their API keys

Knowing which adapter to call is only half the picture. The adapter also needs an API key. Rather than each adapter hard-coding how to find its key, `llm-toolkit` centralises key resolution in `env-api-keys.ts`.

### `findEnvKeys` and `getEnvApiKey`

The two exported functions a provider or caller uses are:

```ts
// Returns the names of the env vars that are currently set for this provider.
// Returns undefined if the provider is unknown OR no matching var is set.
export function findEnvKeys(provider: string): string[] | undefined

// Returns the actual key value from the first set env var.
// Returns undefined if no key is available.
export function getEnvApiKey(provider: string): string | undefined
```

Both take a **provider string** — a short name like `"anthropic"`, `"openai"`, or `"google"` — not the API id string from the registry. This is a separate identifier used purely for key resolution.

### The env-var mapping table

Internally, `getApiKeyEnvVars` contains a lookup table that maps provider names to their environment variable names. Here is the full mapping from the source:

| Provider string | Environment variable(s) |
|---|---|
| `anthropic` | `ANTHROPIC_OAUTH_TOKEN` (checked first), `ANTHROPIC_API_KEY` |
| `openai` | `OPENAI_API_KEY` |
| `azure-openai-responses` | `AZURE_OPENAI_API_KEY` |
| `google` | `GEMINI_API_KEY` |
| `google-vertex` | `GOOGLE_CLOUD_API_KEY` (key) or ADC credentials (see below) |
| `mistral` | `MISTRAL_API_KEY` |
| `deepseek` | `DEEPSEEK_API_KEY` |
| `groq` | `GROQ_API_KEY` |
| `cerebras` | `CEREBRAS_API_KEY` |
| `xai` | `XAI_API_KEY` |
| `openrouter` | `OPENROUTER_API_KEY` |
| `nvidia` | `NVIDIA_API_KEY` |
| `fireworks` | `FIREWORKS_API_KEY` |
| `together` | `TOGETHER_API_KEY` |
| `huggingface` | `HF_TOKEN` |
| `cloudflare-workers-ai` | `CLOUDFLARE_API_KEY` |
| `cloudflare-ai-gateway` | `CLOUDFLARE_API_KEY` |
| `github-copilot` | `COPILOT_GITHUB_TOKEN` |
| `minimax` | `MINIMAX_API_KEY` |
| `minimax-cn` | `MINIMAX_CN_API_KEY` |
| `moonshotai` | `MOONSHOT_API_KEY` |
| `moonshotai-cn` | `MOONSHOT_API_KEY` |
| `kimi-coding` | `KIMI_API_KEY` |
| `opencode` | `OPENCODE_API_KEY` |
| `opencode-go` | `OPENCODE_API_KEY` |
| `xiaomi` | `XIAOMI_API_KEY` |
| `xiaomi-token-plan-cn` | `XIAOMI_TOKEN_PLAN_CN_API_KEY` |
| `xiaomi-token-plan-ams` | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` |
| `xiaomi-token-plan-sgp` | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` |
| `vercel-ai-gateway` | `AI_GATEWAY_API_KEY` |
| `zai` | `ZAI_API_KEY` |
| `zai-coding-cn` | `ZAI_CODING_CN_API_KEY` |
| `ant-ling` | `ANT_LING_API_KEY` |

A provider name not in this table causes both `findEnvKeys` and `getEnvApiKey` to return `undefined`.

### Anthropic: OAuth token takes precedence

Notice that Anthropic has two entries: `ANTHROPIC_OAUTH_TOKEN` is checked first, with `ANTHROPIC_API_KEY` as the fallback. This ordering is deliberate — when both are set, the OAuth token wins.

### Google Vertex: ADC as an alternative to an API key

The comment in the source is explicit: `findEnvKeys` and `getEnvApiKey` only report **API key** variables. They intentionally exclude ambient credential sources like AWS profiles or Google Application Default Credentials.

However, `getEnvApiKey` makes one special exception for Vertex AI: if `GOOGLE_CLOUD_API_KEY` is not set, it checks whether all three ADC conditions are met — a credentials file exists, `GOOGLE_CLOUD_PROJECT` (or `GCLOUD_PROJECT`) is set, and `GOOGLE_CLOUD_LOCATION` is set. If all three are true, `getEnvApiKey` returns the sentinel string `"<authenticated>"` to signal that auth is available without exposing an actual key value.

### Amazon Bedrock: multiple credential sources

Similarly, Bedrock does not use a single API key. `getEnvApiKey` checks for any of several standard AWS credential mechanisms:

| Mechanism | Env vars checked |
|---|---|
| Named profile | `AWS_PROFILE` |
| IAM key pair | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Bedrock bearer token | `AWS_BEARER_TOKEN_BEDROCK` |
| ECS task role (relative) | `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` |
| ECS task role (full URI) | `AWS_CONTAINER_CREDENTIALS_FULL_URI` |
| IRSA (Kubernetes) | `AWS_WEB_IDENTITY_TOKEN_FILE` |

If any of these are present, `getEnvApiKey` returns `"<authenticated>"`.

### The Bun sandbox workaround

One subtlety worth knowing: on Bun, compiled binaries running inside certain Linux sandbox environments have an empty `process.env`. The module contains a fallback, `getProcEnv`, that reads `/proc/self/environ` directly in that situation. Both `findEnvKeys` and `getEnvApiKey` transparently call through to `getProcEnv` when the standard `process.env` lookup returns nothing.

## Step 7 — Plugging in a custom provider

Now that we understand the registry, let's put it all together and add a provider that `llm-toolkit` does not know about. Suppose you have a private inference endpoint that speaks the OpenAI Chat Completions wire format.

There are two things we need to do:

1. Create an `ApiProvider` object that wraps your adapter's `stream` and `streamSimple` functions.
2. Call `registerApiProvider` with a custom API id string.

```ts
import { registerApiProvider } from "llm-toolkit";

// Your adapter — stream and streamSimple follow the same signature
// as the built-in adapters (see the Anthropic/OpenAI chapters).
import { streamMyProvider, streamSimpleMyProvider } from "./my-provider.ts";

registerApiProvider({
  api: "my-private-api",          // any string — becomes the lookup key
  stream: streamMyProvider,
  streamSimple: streamSimpleMyProvider,
});
```

When the streaming front door calls `getApiProvider("my-private-api")` it will find your adapter and dispatch to it. No changes to `llm-toolkit`'s source are required.

### Overriding a built-in provider

Because `registerApiProvider` replaces any existing entry for the same API id, you can also override a built-in. For instance, to swap in a custom Anthropic adapter that adds bespoke retry logic:

```ts
import { registerApiProvider } from "llm-toolkit";
import { streamWithRetry, streamSimpleWithRetry } from "./anthropic-with-retry.ts";

// Same api id as the built-in — replaces it in the registry.
registerApiProvider({
  api: "anthropic-messages",
  stream: streamWithRetry,
  streamSimple: streamSimpleWithRetry,
});
```

### Grouping custom providers with `sourceId`

If you register multiple custom providers from a single module or plugin, pass the same `sourceId` string to each call. You can then remove all of them in one operation:

```ts
registerApiProvider(
  { api: "my-api-v1", stream: streamV1, streamSimple: streamSimpleV1 },
  "my-plugin"
);
registerApiProvider(
  { api: "my-api-v2", stream: streamV2, streamSimple: streamSimpleV2 },
  "my-plugin"
);

// Later, remove only your plugin's providers:
import { unregisterApiProviders } from "llm-toolkit";
unregisterApiProviders("my-plugin");
```

### To add key resolution for your provider

If you want `getEnvApiKey` and `findEnvKeys` to understand your provider's API key, you currently need to contribute a mapping entry to `env-api-keys.ts` in the `llm-toolkit` source — there is no public extension point for the key-resolution table itself. In the meantime, resolve the key in your adapter or in the call-site that constructs the model descriptor before dispatching to the registry.

<!-- GAP: there is no exported function to register a custom provider-name → env-var mapping; source silent on a public extension API for env-api-keys.ts -->

## The full picture

Let's trace a request from start to finish so the moving parts are clear:

1. A caller invokes `stream(model, context, options)` on the streaming front door, where `model.api = "google-generative-ai"`.
2. The front door calls `getApiProvider("google-generative-ai")`.
3. The registry returns the lazy wrapper registered by `registerBuiltInApiProviders`.
4. The lazy wrapper creates an `AssistantMessageEventStream` and triggers `loadGoogleProviderModule()`.
5. The first time this runs, `import("./google.ts")` fires and loads the Google adapter module. On subsequent calls the cached promise resolves immediately.
6. Once the module is available, `forwardStream` pipes events from the real Google adapter into the outer stream.
7. The caller iterates the outer stream and sees events as they arrive — with no knowledge of whether the adapter was loaded lazily or eagerly.

---

← Previous: [Provider Adapters: Google Gemini and the Faux Test Provider](./provider-adapters-google-and-faux.md) · Next: [Model Metadata, Cost Calculation, and Streaming JSON Parsing](./models-cost-and-streaming-json.md) →
