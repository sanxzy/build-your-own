---
title: "OAuth and API-Key Authentication"
description: "Walk through device-code flow, PKCE, and token refresh for Anthropic and OpenAI Codex OAuth, and learn how API keys are resolved from environment variables."
category: llm-toolkit
type: tutorial
tags: [OAuth, device code flow, PKCE, refresh token, Anthropic OAuth, OpenAI Codex, API key, auth, env var, XZY_ANTHROPIC_API_KEY, authentication, subscription, llm-toolkit, OAuthCredentials, OAuthProviderInterface, pollOAuthDeviceCodeFlow, generatePKCE, loginAnthropic, loginOpenAICodex, loginOpenAICodexDeviceCode, refreshAnthropicToken, refreshOpenAICodexToken, getOAuthApiKey, token exchange, callback server, PKCE code verifier, code challenge, authorization code flow]
keywords: [oauth flow, browser login, headless login, cli oauth, token refresh, expired token, access token, refresh token, subscription auth, claude pro, claude max, chatgpt plus, chatgpt pro, codex subscription, env var auth, api key auth, oauth provider registry, registerOAuthProvider, getOAuthProvider]
sources: [S20, S6]
---

**TL;DR** — Not every user has a raw API key; many authenticate via an OAuth subscription (Claude Pro/Max, ChatGPT Plus/Pro). This chapter walks through how the llm-toolkit library implements the OAuth authorization-code + PKCE flow for Anthropic, the device-code flow for OpenAI Codex, and silent token refresh for both. By the end you will understand the full handshake sequence, know how to initiate and refresh a session, and understand how OAuth credentials and plain API keys plug into the same provider interface.

# OAuth and API-Key Authentication

## The problem: some users don't have an API key

When you set up a new provider, the simplest credential is a static API key. The user sets an environment variable (`XZY_ANTHROPIC_API_KEY`, for example), and the provider picks it up — that path is covered in the [provider registry and per-provider env-var key resolution](./api-registry-and-extensibility.md) chapter.

But many users access Anthropic or OpenAI through a subscription plan (Claude Pro/Max, ChatGPT Plus/Pro) rather than a paid API account. Subscription accounts don't issue raw API keys to copy into a terminal. They authenticate through OAuth: the user logs in via a browser, the server issues short-lived tokens, and those tokens are refreshed automatically as they expire.

We need to support both paths through the same `OAuthProviderInterface` abstraction — and we need token refresh to be transparent to the rest of the agent.

## The shared credential type

Before we look at any flow, let's establish what we're working with. Both OAuth paths produce the same credential object:

```ts
// Simplified view — src/utils/oauth/types.ts
export type OAuthCredentials = {
  refresh: string;   // long-lived token; used to obtain a new access token
  access: string;    // short-lived token; sent as the API key on each request
  expires: number;   // Unix timestamp (ms) at which the access token expires
  [key: string]: unknown;  // providers may attach extra fields (e.g. accountId)
};
```

The `access` field is what ultimately goes on the wire as a bearer token. The `refresh` field is stored persistently; when `expires` is in the past we trade the refresh token for a new pair.

## The provider interface

Every OAuth provider in the library implements this interface:

```ts
// src/utils/oauth/types.ts
export interface OAuthProviderInterface {
  readonly id: OAuthProviderId;  // e.g. "anthropic", "openai-codex"
  readonly name: string;         // human-readable label

  /** Run the login flow, return credentials to persist */
  login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;

  /** Whether login uses a local callback server (and supports manual code paste) */
  usesCallbackServer?: boolean;

  /** Refresh expired credentials, return updated credentials */
  refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;

  /** Convert credentials to the API key string used in requests */
  getApiKey(credentials: OAuthCredentials): string;

  /** Optional: modify available models when credentials are known */
  modifyModels?(models: Model<Api>[], credentials: OAuthCredentials): Model<Api>[];
}
```

Three methods define a provider's contract:
- `login` — runs the initial flow and returns credentials you store.
- `refreshToken` — given stored credentials, issues fresh ones.
- `getApiKey` — converts stored credentials to the string the HTTP client sends as the bearer token.

For both Anthropic and OpenAI Codex, `getApiKey` returns `credentials.access` — the short-lived access token is the bearer token.

## Concepts: PKCE and why it exists

The Anthropic flow and the OpenAI Codex browser flow both use **PKCE** (Proof Key for Code Exchange). Let's understand why before looking at the implementation.

In a normal authorization-code flow, an attacker who intercepts the authorization code can exchange it for tokens. PKCE closes this gap: before the flow starts, the client generates a random **code verifier**, hashes it to produce a **code challenge**, and sends the challenge to the authorization server up front. When the client later exchanges the authorization code for tokens, it must send the original verifier. The server checks that `SHA-256(verifier) == challenge` — proving the exchanger is the same party that started the flow.

The library generates PKCE pairs using the Web Crypto API (works in both Node.js 20+ and browsers):

```ts
// src/utils/oauth/pkce.ts
export async function generatePKCE(): Promise<{ verifier: string; challenge: string }> {
  // 32 random bytes → base64url-encoded verifier
  const verifierBytes = new Uint8Array(32);
  crypto.getRandomValues(verifierBytes);
  const verifier = base64urlEncode(verifierBytes);

  // SHA-256 hash of verifier → base64url-encoded challenge
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  const challenge = base64urlEncode(new Uint8Array(hashBuffer));

  return { verifier, challenge };
}
```

The `challenge` is sent in the authorization URL. The `verifier` is sent only when exchanging the authorization code for tokens.

## The Anthropic OAuth flow (authorization code + PKCE)

Anthropic's OAuth flow is a standard authorization-code flow with PKCE. Let's walk through it step by step.

### Step 1 — generate PKCE and build the authorization URL

`loginAnthropic` starts by generating a PKCE pair:

```ts
// src/utils/oauth/anthropic.ts (simplified)
const AUTHORIZE_URL = "https://claude.ai/oauth/authorize";
const TOKEN_URL    = "https://platform.claude.com/v1/oauth/token";
const CALLBACK_PORT = 53692;
const REDIRECT_URI  = `http://localhost:${CALLBACK_PORT}/callback`;
const SCOPES = "org:create_api_key user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload";

export async function loginAnthropic(options: { ... }): Promise<OAuthCredentials> {
  const { verifier, challenge } = await generatePKCE();
  // ...
}
```

Notice the scope list — it requests inference permission (`user:inference`), file uploads, MCP server access, and session management. These are the permissions the agent needs to operate.

### Step 2 — start the local callback server

The library starts an HTTP server on `localhost:53692` that will receive the authorization code when the browser completes the flow:

```ts
// Simplified from startCallbackServer() in anthropic.ts
//
// The server listens on CALLBACK_HOST (env XZY_OAUTH_CALLBACK_HOST, defaulting to 127.0.0.1)
// and resolves a promise as soon as /callback?code=...&state=... arrives.
const server = await startCallbackServer(verifier);
```

The expected `state` value passed to `startCallbackServer` is the PKCE `verifier` itself — the server will reject any callback where the state doesn't match.

### Step 3 — send the user to the browser

With the callback server running, the flow calls `options.onAuth(...)` to hand off the URL to the caller (usually the CLI or TUI layer, which opens a browser or displays the link):

```ts
const authParams = new URLSearchParams({
  response_type: "code",
  client_id: CLIENT_ID,
  redirect_uri: REDIRECT_URI,
  scope: SCOPES,
  code_challenge: challenge,           // SHA-256(verifier)
  code_challenge_method: "S256",
  state: verifier,                     // echoed back in the redirect
});

options.onAuth({
  url: `${AUTHORIZE_URL}?${authParams.toString()}`,
  instructions:
    "Complete login in your browser. If the browser is on another machine, paste the final redirect URL here.",
});
```

### Step 4 — wait for the authorization code

There are two paths to receive the code:

**Automatic path (normal desktop use):** The browser redirects to `http://localhost:53692/callback?code=...&state=...`. The local server verifies the state, captures the code, and resolves the promise.

**Manual path (headless / remote):** If `onManualCodeInput` is provided, the caller can also accept a pasted redirect URL or bare code from the user. `parseAuthorizationInput` handles all three formats: a full URL, `code#state`, or a bare code string. Whichever arrives first (browser callback or manual paste) wins.

### Step 5 — exchange the code for tokens

Once we have the authorization code, we POST to `TOKEN_URL` with the code and the original PKCE verifier:

```ts
// Simplified from exchangeAuthorizationCode() in anthropic.ts
const response = await postJson(TOKEN_URL, {
  grant_type:    "authorization_code",
  client_id:     CLIENT_ID,
  code,
  state,
  redirect_uri:  redirectUri,
  code_verifier: verifier,   // the server checks SHA-256(this) == challenge
});

// Response shape:
// { access_token, refresh_token, expires_in }

return {
  refresh:  tokenData.refresh_token,
  access:   tokenData.access_token,
  // Subtract 5 minutes as a safety buffer so we refresh before the server rejects the token
  expires:  Date.now() + tokenData.expires_in * 1000 - 5 * 60 * 1000,
};
```

The 5-minute safety buffer (`- 5 * 60 * 1000`) ensures tokens are refreshed slightly before they actually expire, avoiding edge cases where a token expires mid-request.

### The complete Anthropic handshake at a glance

```
CLI/TUI                  Local server (:53692)     Anthropic servers
   |                           |                         |
   |-- generatePKCE() -------->|                         |
   |<- { verifier, challenge } |                         |
   |                           |                         |
   |-- startCallbackServer() ->|                         |
   |-- onAuth(url) ----------->|  (show URL to user)     |
   |                           |                         |
   |                     [user opens browser]            |
   |                           |<-- GET /authorize ------>|
   |                           |<-- redirect to :53692 ---|
   |                           |--- /callback?code=... -->|
   |<- { code, state } --------|                         |
   |                           |                         |
   |-- POST /oauth/token -------------------------------->|
   |   { grant_type, code, code_verifier, ... }          |
   |<-- { access_token, refresh_token, expires_in } -----|
   |                           |                         |
```

## Refreshing an expired Anthropic token

Access tokens expire. When they do, the agent calls `refreshAnthropicToken` to get a new pair without making the user log in again:

```ts
// src/utils/oauth/anthropic.ts
export async function refreshAnthropicToken(refreshToken: string): Promise<OAuthCredentials> {
  const responseBody = await postJson(TOKEN_URL, {
    grant_type:    "refresh_token",
    client_id:     CLIENT_ID,
    refresh_token: refreshToken,
  });

  const data = JSON.parse(responseBody) as {
    access_token: string;
    refresh_token: string;
    expires_in: number;
    scope?: string;
  };

  return {
    refresh:  data.refresh_token,
    access:   data.access_token,
    expires:  Date.now() + data.expires_in * 1000 - 5 * 60 * 1000,
  };
}
```

Notice that the server returns a new `refresh_token` too — the caller is responsible for persisting the updated credentials so the next refresh cycle uses the fresh refresh token.

The `anthropicOAuthProvider` object wires this into the interface:

```ts
export const anthropicOAuthProvider: OAuthProviderInterface = {
  id: "anthropic",
  name: "Anthropic (Claude Pro/Max)",
  usesCallbackServer: true,

  async login(callbacks) { return loginAnthropic({ ... }); },

  async refreshToken(credentials) {
    return refreshAnthropicToken(credentials.refresh);
  },

  getApiKey(credentials) {
    return credentials.access;  // access token IS the bearer token
  },
};
```

## The OpenAI Codex flow — two login methods

OpenAI Codex OAuth (`openaiCodexOAuthProvider`, id `"openai-codex"`, name `"ChatGPT Plus/Pro (Codex Subscription)"`) offers two login methods. The provider's `login` method asks the user to choose:

```ts
// src/utils/oauth/openai-codex.ts
export const OPENAI_CODEX_BROWSER_LOGIN_METHOD     = "browser";
export const OPENAI_CODEX_DEVICE_CODE_LOGIN_METHOD = "device_code";

// Inside openaiCodexOAuthProvider.login():
const loginMethod = await callbacks.onSelect({
  message: "Select OpenAI Codex login method:",
  options: [
    { id: OPENAI_CODEX_BROWSER_LOGIN_METHOD,     label: "Browser login (default)" },
    { id: OPENAI_CODEX_DEVICE_CODE_LOGIN_METHOD, label: "Device code login (headless)" },
  ],
});
```

### Method A: browser login (same PKCE pattern as Anthropic)

`loginOpenAICodex` follows the same authorization-code + PKCE pattern as Anthropic, but with different endpoints and a slightly different callback flow:

```ts
const AUTH_BASE_URL  = "https://auth.openai.com";
const AUTHORIZE_URL  = `${AUTH_BASE_URL}/oauth/authorize`;
const TOKEN_URL      = `${AUTH_BASE_URL}/oauth/token`;
const REDIRECT_URI   = "http://localhost:1455/auth/callback";
const SCOPE          = "openid profile email offline_access";
```

The authorization URL is built with extra parameters specific to the Codex flow:

```ts
url.searchParams.set("id_token_add_organizations", "true");
url.searchParams.set("codex_cli_simplified_flow",  "true");
url.searchParams.set("originator", options.originator ?? "xzy");
```

The local callback server listens on port `1455` (rather than Anthropic's `53692`). The state value is a random 16-byte hex string (generated with `node:crypto`'s `randomBytes`) rather than the PKCE verifier.

After exchanging the authorization code, OpenAI's token response includes a JWT-encoded `access_token`. The library decodes the JWT to extract the `chatgpt_account_id` claim at path `https://api.openai.com/auth`:

```ts
// Simplified from credentialsFromToken() in openai-codex.ts
function getAccountId(accessToken: string): string | null {
  const payload = decodeJwt(accessToken);
  const auth    = payload?.["https://api.openai.com/auth"];
  return auth?.chatgpt_account_id ?? null;
}
```

The resulting `OAuthCredentials` includes `accountId` as an extra field alongside `access`, `refresh`, and `expires`.

### Method B: device-code flow (headless)

The device-code flow is designed for headless environments — servers, CI, or machines without a browser. The user enters a short code on a separate device (their phone or another computer). Let's trace through how it works.

#### Concepts: what a device code is

In a device-code flow (RFC 8628), the client requests a **user code** — a short, human-typeable string like `ABCD-1234`. The authorization server gives the client a URL to display to the user. The user visits that URL on *any* device, enters the code, and approves the request. Meanwhile the client **polls** the authorization server at regular intervals until it receives a success response.

This is useful when the device running the agent can't open a browser (headless server, SSH session, etc.).

#### Step 1 — request a device auth session

`startOpenAICodexDeviceAuth` POSTs to the device user-code endpoint:

```ts
// src/utils/oauth/openai-codex.ts (simplified)
const DEVICE_USER_CODE_URL = `${AUTH_BASE_URL}/api/accounts/deviceauth/usercode`;
const DEVICE_VERIFICATION_URI = `${AUTH_BASE_URL}/codex/device`;

async function startOpenAICodexDeviceAuth(signal?: AbortSignal): Promise<DeviceAuthInfo> {
  const response = await fetch(DEVICE_USER_CODE_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ client_id: CLIENT_ID }),
    signal,
  });
  // Returns: { device_auth_id, user_code, interval }
  return {
    deviceAuthId:    json.device_auth_id,
    userCode:        json.user_code,
    intervalSeconds: json.interval,
  };
}
```

#### Step 2 — display the code to the user

The `loginOpenAICodexDeviceCode` function calls back to the UI layer to show the code:

```ts
// src/utils/oauth/openai-codex.ts
export async function loginOpenAICodexDeviceCode(options: {
  onDeviceCode: (info: OAuthDeviceCodeInfo) => void;
  signal?: AbortSignal;
}): Promise<OAuthCredentials> {
  const device = await startOpenAICodexDeviceAuth(options.signal);

  options.onDeviceCode({
    userCode:         device.userCode,
    verificationUri:  DEVICE_VERIFICATION_URI,  // "https://auth.openai.com/codex/device"
    intervalSeconds:  device.intervalSeconds,
    expiresInSeconds: DEVICE_CODE_TIMEOUT_SECONDS,  // 15 * 60 = 900 seconds
  });

  const code = await pollOpenAICodexDeviceAuth(device, options.signal);
  return exchangeAuthorizationCodeForCredentials(
    code.authorizationCode,
    code.codeVerifier,
    DEVICE_REDIRECT_URI,
    options.signal,
  );
}
```

`DEVICE_CODE_TIMEOUT_SECONDS` is 900 seconds (15 minutes). If the user doesn't approve within that window, the flow times out.

#### Step 3 — poll for authorization

`pollOpenAICodexDeviceAuth` uses the shared `pollOAuthDeviceCodeFlow` helper (from `device-code.ts`):

```ts
async function pollOpenAICodexDeviceAuth(
  device: DeviceAuthInfo,
  signal?: AbortSignal,
): Promise<DeviceTokenSuccess> {
  return pollOAuthDeviceCodeFlow<DeviceTokenSuccess>({
    intervalSeconds:  device.intervalSeconds,
    expiresInSeconds: DEVICE_CODE_TIMEOUT_SECONDS,
    signal,
    poll: async () => {
      const response = await fetch(DEVICE_TOKEN_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          device_auth_id: device.deviceAuthId,
          user_code:      device.userCode,
        }),
        signal,
      });

      if (response.ok) {
        // User approved — server returns authorization_code + code_verifier
        const json = await response.json();
        return { status: "complete", value: { authorizationCode: json.authorization_code, codeVerifier: json.code_verifier } };
      }
      if (response.status === 403 || response.status === 404) {
        return { status: "pending" };  // still waiting
      }
      // Parse errorCode from response body ...
      if (errorCode === "deviceauth_authorization_pending") return { status: "pending" };
      if (errorCode === "slow_down")                         return { status: "slow_down" };
      return { status: "failed", message: `...` };
    },
  });
}
```

Notice that a successful poll response from OpenAI's device endpoint includes both an `authorization_code` and a `code_verifier` — the server generates the PKCE pair on behalf of the device flow, then the client exchanges that code for final tokens in the same way as the browser flow.

### The device-code handshake at a glance

```
CLI/TUI                      OpenAI servers             User's other device
   |                              |                            |
   |-- POST /deviceauth/usercode ->|                            |
   |<- { device_auth_id,          |                            |
   |     user_code, interval } ---|                            |
   |                              |                            |
   |-- display user_code -------->|                    [user visits URL,
   |   "Go to ...codex/device"    |                     enters code]
   |                              |<--- user approval -------->|
   |                              |                            |
   |-- poll /deviceauth/token --->|                            |
   |   (every `interval` seconds) |                            |
   |<- { authorization_code,      |                            |
   |     code_verifier } ---------|                            |
   |                              |                            |
   |-- POST /oauth/token -------->|                            |
   |   { grant_type: auth_code,   |                            |
   |     code, code_verifier, ... }|                           |
   |<- { access_token,            |                            |
   |     refresh_token, ... } ----|                            |
```

## The polling loop — RFC 8628 compliance

The `pollOAuthDeviceCodeFlow` helper (from `device-code.ts`) handles the polling loop for any device-code flow. Let's look at it in detail because it has important behaviour around rate limiting:

```ts
// src/utils/oauth/device-code.ts
// RFC 8628 section 3.2: if interval is omitted, use 5 seconds.
const DEFAULT_POLL_INTERVAL_SECONDS = 5;
// RFC 8628 section 3.5: slow_down means add 5 seconds to all future polls.
const SLOW_DOWN_INTERVAL_INCREMENT_MS = 5000;
const MINIMUM_INTERVAL_MS = 1000;

export async function pollOAuthDeviceCodeFlow<T>(
  options: OAuthDeviceCodePollOptions<T>,
): Promise<T> {
  const deadline = typeof options.expiresInSeconds === "number"
    ? Date.now() + options.expiresInSeconds * 1000
    : Number.POSITIVE_INFINITY;

  let intervalMs = Math.max(
    MINIMUM_INTERVAL_MS,
    Math.floor((options.intervalSeconds ?? DEFAULT_POLL_INTERVAL_SECONDS) * 1000),
  );

  let slowDownResponses = 0;
  while (Date.now() < deadline) {
    if (options.signal?.aborted) throw new Error("Login cancelled");

    const result = await options.poll();
    if (result.status === "complete") return result.value;
    if (result.status === "failed")   throw new Error(result.message);
    if (result.status === "slow_down") {
      slowDownResponses += 1;
      // Permanently increase the interval by 5 seconds for this and all future polls
      intervalMs = Math.max(MINIMUM_INTERVAL_MS, intervalMs + SLOW_DOWN_INTERVAL_INCREMENT_MS);
    }

    await abortableSleep(Math.min(intervalMs, deadline - Date.now()), options.signal, "Login cancelled");
  }

  throw new Error(slowDownResponses > 0
    ? "Device flow timed out after one or more slow_down responses. ..."
    : "Device flow timed out");
}
```

Key behaviours to note:

| Situation | Behaviour |
|---|---|
| `status: "pending"` | Sleep for `intervalMs`, then poll again |
| `status: "slow_down"` | Permanently add 5 s to `intervalMs` (RFC 8628 §3.5), then sleep |
| `status: "complete"` | Return the result immediately |
| `status: "failed"` | Throw with the error message |
| Deadline exceeded | Throw a timeout error |
| `AbortSignal` fired | Throw "Login cancelled" (via `abortableSleep`) |

The `abortableSleep` function wires the `AbortSignal` to the sleep timer so that cancelling the flow (e.g. the user presses Ctrl-C in the terminal) wakes the sleep immediately rather than waiting out the full interval.

A note on the slow-down timeout message: if the flow times out after one or more `slow_down` responses, the error hints at clock drift in WSL or VM environments as a possible cause — a useful diagnostic for headless deployments.

## Refreshing OpenAI Codex tokens

The refresh path for OpenAI Codex follows the same pattern as Anthropic's, but uses the OpenAI auth endpoint:

```ts
// src/utils/oauth/openai-codex.ts
async function refreshAccessToken(refreshToken: string): Promise<OAuthToken> {
  const response = await fetch(TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type:    "refresh_token",
      refresh_token: refreshToken,
      client_id:     CLIENT_ID,
    }),
  });
  return readTokenResponse(response, "refresh");
}

export async function refreshOpenAICodexToken(refreshToken: string): Promise<OAuthCredentials> {
  return credentialsFromToken(await refreshAccessToken(refreshToken));
}
```

`credentialsFromToken` also re-extracts the `accountId` from the new JWT access token, so the updated credentials remain complete.

## Automatic token refresh at call time

The library provides a high-level helper `getOAuthApiKey` that resolves the API key for a given provider, refreshing automatically if the token has expired:

```ts
// src/utils/oauth/index.ts
export async function getOAuthApiKey(
  providerId: OAuthProviderId,
  credentials: Record<string, OAuthCredentials>,
): Promise<{ newCredentials: OAuthCredentials; apiKey: string } | null> {
  const provider = getOAuthProvider(providerId);
  if (!provider) throw new Error(`Unknown OAuth provider: ${providerId}`);

  let creds = credentials[providerId];
  if (!creds) return null;  // no credentials stored → caller falls back to API key

  // Refresh if expired
  if (Date.now() >= creds.expires) {
    creds = await provider.refreshToken(creds);
    // caller must persist newCredentials for the next call
  }

  const apiKey = provider.getApiKey(creds);
  return { newCredentials: creds, apiKey };
}
```

This is the single integration point that the rest of the agent uses. The caller:
1. Passes in its persisted `credentials` map (provider id → `OAuthCredentials`).
2. If the result is `null`, no OAuth credentials exist for that provider — fall back to a static API key from the environment.
3. If the result is non-null, use `apiKey` as the bearer token and persist `newCredentials` (which may be refreshed).

## The OAuth provider registry

Three providers are registered out of the box:

```ts
// src/utils/oauth/index.ts
const BUILT_IN_OAUTH_PROVIDERS: OAuthProviderInterface[] = [
  anthropicOAuthProvider,   // id: "anthropic"
  githubCopilotOAuthProvider, // id: "github-copilot"  (not covered in this chapter)
  openaiCodexOAuthProvider,  // id: "openai-codex"
];
```

You can look up, register, or replace providers at runtime:

```ts
// Look up a provider by id
getOAuthProvider("anthropic")      // → anthropicOAuthProvider | undefined

// Register a custom provider (adds or replaces)
registerOAuthProvider(myProvider)

// Unregister a custom provider (restores built-in if it was one)
unregisterOAuthProvider("my-custom")

// Reset to built-ins
resetOAuthProviders()

// Enumerate all registered providers
getOAuthProviders()  // → OAuthProviderInterface[]
```

Calling `unregisterOAuthProvider` with a built-in id does not remove the built-in — it restores it. Calling it with a custom id removes it completely.

## Comparing the two authentication paths

Here is how OAuth credentials and plain API keys differ from the agent's perspective:

| Property | Static API key | OAuth credential |
|---|---|---|
| Source | Environment variable (e.g. `XZY_ANTHROPIC_API_KEY`) | Stored `OAuthCredentials` object |
| Lifetime | Indefinite (until revoked) | Short-lived; expires at `credentials.expires` |
| Refresh | Not needed | Automatic via `refreshToken()` |
| User action required | Copy-paste once into env | Browser/device login once; then automatic |
| `getApiKey()` returns | The env var value directly | `credentials.access` (the JWT/bearer token) |
| Suitable for | API-plan accounts | Subscription accounts (Pro/Max, Plus/Pro) |

In practice, the provider resolution order works as follows: if `getOAuthApiKey` returns a non-null result, that API key takes precedence. If it returns null (no credentials stored), the provider falls back to looking up the static API key from the environment — the path described in the [provider registry chapter](./api-registry-and-extensibility.md).

## Callback server security

Both the Anthropic and OpenAI Codex browser flows use a local HTTP server to receive the authorization code. A few security details worth noting:

- The server listens on `127.0.0.1` (or the value of the `XZY_OAUTH_CALLBACK_HOST` environment variable). It is not accessible from the network by default.
- Every callback is validated against the expected `state` value before the code is accepted. A mismatched state triggers a `400` response with an error page.
- The server is closed immediately after the code is received (or after the flow is cancelled), in the `finally` block of the login function.
- If the port is already in use when the OpenAI Codex server tries to bind, the server error resolves the wait-for-code promise with `null`, so the flow falls through to the manual prompt rather than hanging.

<!-- GAP: The source does not document what happens when Anthropic's port 53692 is already in use (whether it errors or falls back to manual input) — the error handler for the Anthropic server calls reject() but there is no fallback shown in the flow path; source silent -->

---

← Previous: [Model Metadata, Cost Calculation, and Streaming JSON Parsing](./models-cost-and-streaming-json.md) · Next: [Agent Context, Events, and Types](../agent-loop/agent-context-and-types.md) →
