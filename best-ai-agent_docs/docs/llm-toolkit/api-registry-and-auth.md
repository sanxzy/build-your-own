---
title: "The API Registry and Authentication"
description: "Build the pluggable API registry that resolves provider adapters at runtime, environment-based API key resolution, and the OAuth device-code flow with PKCE for Anthropic and OpenAI Codex."
category: llm-toolkit
type: tutorial
tags: [API registry, registerProvider, lookupProvider, lazy registration, built-in providers, custom provider, API keys, env vars, OAuth, device code flow, PKCE, authentication, llm-toolkit]
keywords: [provider registry, registerApiProvider, getApiProvider, lazy loading, env API keys, OAuth PKCE, device authorization]
sources: [S7, S13, S14, S15]
---

**TL;DR** — We need a way to register provider adapters and look them up at runtime, so `streamSimple()` can resolve `"anthropic-messages"` to our `streamAnthropic` function. We'll build the API registry with lazy registration (providers are only loaded when first used), environment-variable API key resolution, and OAuth authentication with the device-code flow and PKCE for providers that require it.

## The API registry

The API registry is a key-value store mapping API identifiers (like `"anthropic-messages"`) to provider adapter objects. Create `packages/llm-toolkit/src/api-registry.ts`:

```ts
import type { Api, StreamFunction } from "./types.ts";

interface ApiProvider {
  streamSimple: StreamFunction;
  completeSimple: (model: any, context: any, options?: any) => Promise<any>;
}

const registry = new Map<Api, ApiProvider | (() => Promise<ApiProvider>)>();

export function registerApiProvider(
  api: Api,
  provider: ApiProvider | (() => Promise<ApiProvider>),
): void {
  if (registry.has(api)) {
    throw new Error(`API provider already registered for api: ${api}`);
  }
  registry.set(api, provider);
}

export function getApiProvider(api: Api): ApiProvider | undefined {
  const entry = registry.get(api);
  if (!entry) return undefined;
  if (typeof entry === "function") {
    // Lazy provider — resolve and cache
    const resolved = (entry as (() => Promise<ApiProvider>))();
    registry.set(api, resolved as any);
    return undefined; // Caller should handle async resolution
  }
  return entry;
}
```

The registry supports two registration modes:

1. **Eager** — the adapter object is registered directly. Used for built-in providers loaded at startup.
2. **Lazy** — a factory function is registered. The real adapter loads only when first requested. Used for optional or heavy providers.

### Built-in provider registration

Each adapter file calls `registerApiProvider()` at import time. The registration file `packages/llm-toolkit/src/providers/register-builtins.ts` imports all built-in adapters:

```ts
// This file is imported by stream.ts to ensure all built-in providers are registered
import "./anthropic.ts";
import "./openai-responses.ts";
import "./openai-completions.ts";
import "./google.ts";
import "./google-vertex.ts";
import "./mistral.ts";
import "./amazon-bedrock.ts";
import "./azure-openai-responses.ts";
import "./openai-codex-responses.ts";
```

The `stream.ts` file imports this at the top:

```ts
import "./providers/register-builtins.ts";
```

By the time `streamSimple()` is called, all built-in adapters are registered. Custom adapters can be registered at any time by calling `registerApiProvider()` directly.

## Environment-based API key resolution

Most providers authenticate with API keys stored in environment variables. We standardize the naming convention in `packages/llm-toolkit/src/env-api-keys.ts`:

```ts
const PROVIDER_API_KEY_ENV: Record<string, string> = {
  "anthropic":        "ANTHROPIC_API_KEY",
  "openai":           "OPENAI_API_KEY",
  "google":           "GEMINI_API_KEY",
  "google-vertex":    "GOOGLE_API_KEY",
  "mistral":          "MISTRAL_API_KEY",
  "deepseek":         "DEEPSEEK_API_KEY",
  "xai":              "XAI_API_KEY",
  "groq":             "GROQ_API_KEY",
  "openrouter":       "OPENROUTER_API_KEY",
  "github-copilot":   "GITHUB_TOKEN",
  "cerebras":         "CEREBRAS_API_KEY",
  "amazon-bedrock":   "AWS_ACCESS_KEY_ID",  // Uses AWS credential chain
  // ... more providers
};

export function getEnvApiKey(provider: string): string | undefined {
  const envVar = PROVIDER_API_KEY_ENV[provider];
  if (!envVar) return undefined;

  if (typeof process !== "undefined" && process.env) {
    return process.env[envVar];
  }
  return undefined;
}
```

The `withEnvApiKey()` function in `stream.ts` uses this to fill in the `apiKey` field when the caller doesn't provide one explicitly.

## OAuth authentication

Some providers (Anthropic, OpenAI Codex) support OAuth in addition to API keys. OAuth is better for distributed applications because users authenticate through their browser — the application never sees their credentials.

Our OAuth implementation handles the **device authorization flow with PKCE** (Proof Key for Code Exchange). The flow:

1. **Request a device code.** The application sends a request to the provider's device authorization endpoint.
2. **User opens a browser.** The application shows a URL (e.g., `https://anthropic.com/activate`) and a user code.
3. **User enters the code.** They paste the code into the browser page and approve the authorization.
4. **Poll for completion.** The application polls the token endpoint until the user completes authorization.
5. **Exchange for tokens.** Once authorized, the device code is exchanged for access and refresh tokens.

Create `packages/llm-toolkit/src/oauth.ts`:

```ts
export interface OAuthProvider {
  id: string;
  name: string;
  deviceCodeUrl: string;
  tokenUrl: string;
  scopes: string[];
}

export async function startDeviceCodeFlow(
  provider: OAuthProvider,
): Promise<OAuthDeviceCodeInfo> {
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = await computeCodeChallenge(codeVerifier);

  const response = await fetch(provider.deviceCodeUrl, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: provider.clientId,
      scope: provider.scopes.join(" "),
      code_challenge: codeChallenge,
      code_challenge_method: "S256",
    }),
  });

  const data = await response.json();
  return {
    deviceCode: data.device_code,
    userCode: data.user_code,
    verificationUri: data.verification_uri,
    verificationUriComplete: data.verification_uri_complete,
    expiresIn: data.expires_in,
    interval: data.interval ?? 5,
    codeVerifier,
  };
}

export async function pollForTokens(
  provider: OAuthProvider,
  deviceCode: string,
  codeVerifier: string,
  signal?: AbortSignal,
): Promise<OAuthTokens> {
  while (true) {
    if (signal?.aborted) throw new Error("OAuth flow aborted");

    const response = await fetch(provider.tokenUrl, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        grant_type: "urn:ietf:params:oauth:grant-type:device_code",
        device_code: deviceCode,
        client_id: provider.clientId,
        code_verifier: codeVerifier,
      }),
    });

    const data = await response.json();

    if (data.error === "authorization_pending") {
      // User hasn't completed yet — wait and poll again
      await sleep((data.interval ?? 5) * 1000);
      continue;
    }

    if (data.error) {
      throw new Error(`OAuth error: ${data.error} — ${data.error_description}`);
    }

    return {
      accessToken: data.access_token,
      refreshToken: data.refresh_token,
      expiresAt: Date.now() + (data.expires_in ?? 3600) * 1000,
    };
  }
}
```

PKCE adds a security layer: even if someone intercepts the device code, they can't exchange it for tokens without the `code_verifier` (which only the initiating application knows).

### Provider-specific OAuth

Each OAuth provider has its own endpoints and client configuration. The Anthropic OAuth module (`utils/oauth/anthropic.ts`) pre-configures the Anthropic-specific URLs and scopes. OpenAI Codex has its own. The generic OAuth module handles the common flow; provider-specific modules only need to define endpoints.

## What we've built

The LLM Toolkit is now complete:

- **API registry** maps API identifiers to adapter instances, with lazy loading
- **Built-in registration** ensures all first-party adapters are available at startup
- **Environment key resolution** standardizes API key discovery across 20+ providers
- **OAuth** supports the device-code flow with PKCE for browser-based authentication

In the final LLM Toolkit chapter, we'll add the model registry and the streaming JSON parser for handling partial tool call arguments.

---

← Previous: [Provider Adapter: Google Gemini](./provider-adapter-google.md) · Next: [Model Registry, Costs, and Streaming JSON](./models-and-streaming-json.md) →
