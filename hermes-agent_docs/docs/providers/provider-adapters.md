---
title: "Provider Adapters — Anthropic, Bedrock, Gemini, and Codex"
description: "How Hermes's four provider-specific adapters translate internal requests into each provider's exact wire protocol, covering auth variants, transport quirks, and replay safety."
category: providers
type: explanation
tags:
  - Anthropic adapter
  - three auth variants
  - API key
  - OAuth setup-token
  - Claude Code credential
  - cache_control
  - ANTHROPIC_API_KEY
  - ANTHROPIC_TOKEN
  - CLAUDE_CODE_OAUTH_TOKEN
  - Bedrock adapter
  - boto3
  - Converse API
  - cross-region inference
  - inference profiles
  - us.anthropic.claude
  - global.anthropic.claude
  - Gemini native adapter
  - OpenAI compat bypass
  - GeminiNativeClient
  - generativelanguage.googleapis.com
  - Codex Responses adapter
  - issuer stamping
  - encrypted_content replay
  - invalid_encrypted_content
  - api_mode
  - anthropic_messages
  - bedrock_converse
  - chat_completions
  - codex_responses
  - ProviderProfile
keywords:
  - Anthropic Messages API
  - bearer auth
  - x-api-key header
  - AWS credential chain
  - IAM roles
  - SSO profile
  - GOOGLE_API_KEY
  - GEMINI_API_KEY
  - xAI Responses
  - GitHub Models Responses
  - cross-issuer guard
  - token refresh
  - AnthropicBedrock
sources: [S28, S29, S30, S31, S32]
---

**TL;DR** — Each `api_mode` speaks a different wire protocol, and some providers need handling that a generic OpenAI-compatible path cannot give. This chapter walks through Hermes's four provider-specific adapters — Anthropic (three auth variants), Bedrock (boto3 + Converse API), Gemini native (bypasses the OpenAI-compat layer), and Codex Responses (issuer-stamped encrypted reasoning) — explaining why each exists and how it works.

# Provider Adapters — Anthropic, Bedrock, Gemini, and Codex

## The problem each adapter solves

In the previous chapter ([Config-Driven Provider Routing and the Four api_mode Values](./config-driven-routing-and-api-modes.md)), we saw how Hermes chooses an `api_mode` for each provider — for example, `anthropic_messages` for native Anthropic, `bedrock_converse` for AWS Bedrock, `chat_completions` for Gemini, and `codex_responses` for OpenAI Codex.

But choosing a wire protocol is only half the work. Once Hermes knows *which* protocol to use, something must translate Hermes's internal request format — which is modelled on the OpenAI Chat Completions shape — into the exact payload that provider expects. That translator is the **adapter**.

Think of an adapter as a shim: it sits between the agent loop and the network, converting Hermes's standard message format going out, and normalising the provider's response format coming back. Each adapter is isolated in its own module so all provider-specific quirks stay in one place.

```mermaid
flowchart LR
    AgentLoop["Agent loop\n(OpenAI-shaped messages)"] --> Router["api_mode\nrouter"]
    Router -->|anthropic_messages| AnthAdapter["anthropic_adapter.py"]
    Router -->|bedrock_converse| BedrockAdapter["bedrock_adapter.py"]
    Router -->|chat_completions + native Gemini| GeminiAdapter["gemini_native_adapter.py"]
    Router -->|codex_responses| CodexAdapter["codex_responses_adapter.py"]
    AnthAdapter -->|Messages API| Anthropic["api.anthropic.com"]
    BedrockAdapter -->|Converse API / boto3| Bedrock["AWS Bedrock"]
    GeminiAdapter -->|generateContent REST| Gemini["generativelanguage.googleapis.com"]
    CodexAdapter -->|Responses API| Codex["chatgpt.com/backend-api/codex"]
```

Let's build up an understanding of each one, starting with the most complex.

---

## The Anthropic adapter — three auth variants

### Why auth is complicated here

Most providers have one authentication method: you get an API key, you put it in a header, done. Anthropic has three distinct credential types that each require a different HTTP authentication scheme:

| Variant | Token shape | HTTP mechanism |
|---------|------------|----------------|
| Regular API key | `sk-ant-api*` | `x-api-key` header |
| OAuth setup-token | `sk-ant-oat*`, `eyJ*`, `cc-*` | `Authorization: Bearer` + extra beta headers |
| Claude Code credential | OAuth token from `~/.claude/.credentials.json` or macOS Keychain | `Authorization: Bearer` + Claude Code identity headers |

If you send an OAuth token via `x-api-key`, or a regular API key via `Bearer`, the Anthropic API will reject the request. The adapter must auto-detect which variant it has and configure the underlying HTTP client accordingly.

### How the Anthropic adapter detects and routes each variant

The entry point is `build_anthropic_client()` in `agent/anthropic_adapter.py`. Let's walk through what it does with each variant.

**Variant 1 — Regular API key (`ANTHROPIC_API_KEY`)**

A regular API key starts with `sk-ant-api`. The adapter recognises this prefix and sets `api_key=` on the Anthropic SDK client, which sends `x-api-key: <key>` on every request. Common beta headers (for features like interleaved thinking and fine-grained tool streaming) are included as client-level defaults:

```python
# Simplified view of build_anthropic_client() — regular API key path
kwargs["api_key"] = api_key
if common_betas:
    kwargs["default_headers"] = {"anthropic-beta": ",".join(common_betas)}
return anthropic_sdk.Anthropic(**kwargs)
```

Set this up in your environment:

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-..."
```

Or in `~/.hermes/.env`:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-...
```

**Variant 2 — OAuth setup-token**

An OAuth setup-token looks like `sk-ant-oat*`, or it may be a JWT (`eyJ*`) or a Claude Code OAuth access token (`cc-*`). The adapter's `_is_oauth_token()` function checks these prefixes.

For OAuth tokens, the adapter sends `Authorization: Bearer <token>` and adds two extra beta headers that Anthropic's infrastructure uses to route OAuth traffic correctly: `claude-code-20250219` and `oauth-2025-04-20`. It also sets a `user-agent` and `x-app: cli` header with the Claude Code version — without these, Anthropic's OAuth infrastructure intermittently returns HTTP 500:

```python
# Simplified view — OAuth setup-token path
all_betas = common_betas + ["claude-code-20250219", "oauth-2025-04-20"]
kwargs["auth_token"] = api_key
kwargs["default_headers"] = {
    "anthropic-beta": ",".join(all_betas),
    "user-agent": f"claude-cli/{claude_code_version} (external, cli)",
    "x-app": "cli",
}
```

The `ANTHROPIC_TOKEN` environment variable is the primary slot for OAuth tokens in Hermes. `CLAUDE_CODE_OAUTH_TOKEN` is also accepted and works identically:

```bash
export ANTHROPIC_TOKEN="sk-ant-oat01-..."
```

**Variant 3 — Claude Code credential (auto-detected)**

This is the smoothest variant. If you have Claude Code (`claude` CLI) installed and have logged in, Hermes finds and uses your existing session. The adapter checks two locations in priority order:

1. The macOS Keychain entry `"Claude Code-credentials"` (available from Claude Code ≥ 2.1.114)
2. `~/.claude/.credentials.json`

Both files store a `claudeAiOauth` JSON object with an `accessToken`, `refreshToken`, and `expiresAt`. The adapter checks whether the token is still valid (with a 60-second safety buffer), and if it has expired, uses the `refreshToken` to get a fresh one automatically.

`read_claude_code_credentials()` reads both sources, and `_resolve_claude_code_token_from_credentials()` handles the refresh logic:

```python
# Simplified view — Claude Code credential resolution
creds = read_claude_code_credentials()
if creds and is_claude_code_token_valid(creds):
    return creds["accessToken"]
if creds:
    # Token expired — try to refresh
    refreshed = _refresh_oauth_token(creds)
    if refreshed:
        return refreshed
```

The full token resolution priority is:

1. `ANTHROPIC_TOKEN` env var (OAuth/setup token saved by Hermes)
2. `CLAUDE_CODE_OAUTH_TOKEN` env var
3. Claude Code credential files (`~/.claude/.credentials.json` or macOS Keychain)
4. `ANTHROPIC_API_KEY` env var (regular API key)

### cache_control on system prompts and tools

The Anthropic Messages API supports **prompt caching** — you can mark specific content blocks with `cache_control: {type: ephemeral}` to tell Anthropic to cache that content server-side. This reduces latency and cost for long system prompts and tool schemas that repeat across many turns.

The adapter preserves `cache_control` markers during message conversion. In `convert_messages_to_anthropic()` (around line 2088), when a system message has content blocks that carry `cache_control`, the adapter passes them as a list of blocks rather than a flat string — keeping the markers intact for Anthropic to process:

```python
# Simplified view of convert_messages_to_anthropic — cache_control handling
if role == "system":
    if isinstance(content, list):
        has_cache = any(
            p.get("cache_control") for p in content if isinstance(p, dict)
        )
        if has_cache:
            system = [p for p in content if isinstance(p, dict)]  # preserve blocks
        else:
            system = "\n".join(p["text"] for p in content if p.get("type") == "text")
```

Similarly, `convert_tools_to_anthropic()` forwards any `cache_control` dict from a tool definition to the resulting Anthropic tool object, so you can cache the entire tool schema by placing `cache_control` on the last tool in the array.

### The `anthropic` provider profile

The `AnthropicProfile` in `plugins/model-providers/anthropic/__init__.py` registers the provider with `api_mode="anthropic_messages"` and three recognised env vars:

```python
anthropic = AnthropicProfile(
    name="anthropic",
    aliases=("claude", "claude-oauth", "claude-code"),
    api_mode="anthropic_messages",
    env_vars=("ANTHROPIC_API_KEY", "ANTHROPIC_TOKEN", "CLAUDE_CODE_OAUTH_TOKEN"),
    base_url="https://api.anthropic.com",
    auth_type="api_key",
    default_aux_model="claude-haiku-4-5-20251001",
)
```

The `aliases` mean you can address this provider as `anthropic`, `claude`, `claude-oauth`, or `claude-code` in your config.

### Worked example — configuring all three auth variants

**Option A: Regular API key**

```yaml
# ~/.hermes/config.yaml
model:
  provider: anthropic
  name: claude-opus-4-7
```

```bash
# ~/.hermes/.env
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
```

**Option B: OAuth setup-token**

```bash
# Run the interactive OAuth login
hermes auth login
```

This stores a token in `ANTHROPIC_TOKEN`. Alternatively, provide it directly:

```bash
# ~/.hermes/.env
ANTHROPIC_TOKEN=sk-ant-oat01-your-setup-token-here
```

**Option C: Claude Code credential (zero-config)**

If `claude` CLI is installed and you have run `claude login`, Hermes picks up your credential automatically — no env var needed. You can verify which token source is active:

```bash
hermes doctor
```

### Edge cases

**Expired OAuth setup-token**

A setup-token that cannot be refreshed (because no `refreshToken` is present) causes `_resolve_claude_code_token_from_credentials()` to return `None`. Hermes will fall through to the next priority source (e.g. `ANTHROPIC_API_KEY`). If no source resolves, the agent halts at startup with a configuration error. The adapter logs a debug message: `"Token refresh failed — re-run 'claude setup-token' to reauthenticate"`.

You can recover by running:

```bash
claude setup-token
```

This writes a fresh token to `~/.claude/.credentials.json`, which Hermes will find on the next startup.

---

## The Bedrock adapter — boto3 and the Converse API

### Why Bedrock needs its own adapter

AWS Bedrock does not expose a REST endpoint that accepts the OpenAI message format. Instead, it provides:

- A **native Converse API** — a unified interface for all Bedrock models (Claude, Nova, Llama, Mistral, and others) that uses its own message structure.
- An **AWS credential chain** via `boto3` — authentication uses IAM roles, SSO profiles, environment variables, or instance metadata rather than a single API key.

Because neither the message format nor the auth model resembles OpenAI Chat Completions, the Bedrock adapter (`agent/bedrock_adapter.py`) handles both concerns: it converts messages and tools to the Converse schema, and it lets boto3 manage credentials entirely.

### How boto3 credentials work

`boto3` — AWS's Python SDK — implements a **credential chain**. When you call `boto3.client("bedrock-runtime", ...)`, boto3 automatically checks sources in priority order:

1. `AWS_BEARER_TOKEN_BEDROCK` — a Bedrock-specific bearer token
2. `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` — explicit IAM key pair
3. `AWS_PROFILE` — a named profile from `~/.aws/config` (covers SSO, assume-role, and other flows)
4. `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` — ECS task role
5. `AWS_WEB_IDENTITY_TOKEN_FILE` — EKS IRSA
6. EC2 instance metadata (IMDS) — the implicit case for cloud-deployed agents

You do not set an API key anywhere in Hermes for Bedrock. You configure AWS credentials once (via the AWS CLI or environment), and boto3 picks them up.

```bash
# Local development: configure a profile
aws configure --profile hermes-bedrock

# Or use environment variables
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
```

The Bedrock provider profile in `plugins/model-providers/bedrock/__init__.py` reflects this:

```python
bedrock = BedrockProfile(
    name="bedrock",
    aliases=("aws", "aws-bedrock", "amazon-bedrock", "amazon"),
    api_mode="bedrock_converse",
    env_vars=(),          # AWS SDK credentials — not managed via Hermes env vars
    base_url="https://bedrock-runtime.us-east-1.amazonaws.com",
    auth_type="aws_sdk",
)
```

Note that `env_vars` is empty — Hermes doesn't manage Bedrock credentials, boto3 does.

### Cross-region inference profiles

Now we have a problem: AWS Bedrock quotas are per-region. If `us-east-1` is at capacity, your request fails outright. **Cross-region inference profiles** solve this by routing requests to whichever AWS region has capacity for the requested model.

An inference profile is an Bedrock resource with an ID like `us.anthropic.claude-sonnet-4-6-20250514-v1:0` or `global.anthropic.claude-opus-4-7`. The prefix (`us.`, `global.`, `eu.`, `ap.`, `jp.`) tells Bedrock the routing scope. You pass the profile ID as the model ID — Bedrock handles the region selection internally.

The adapter's `is_anthropic_bedrock_model()` recognises both bare model IDs (`anthropic.claude-*`) and regional profile IDs by checking the prefix:

```python
# Simplified view of is_anthropic_bedrock_model()
def is_anthropic_bedrock_model(model_id: str) -> bool:
    model_lower = model_id.lower()
    for prefix in ("us.", "global.", "eu.", "ap.", "jp."):
        if model_lower.startswith(prefix):
            model_lower = model_lower[len(prefix):]
            break
    return model_lower.startswith("anthropic.claude")
```

The adapter also discovers inference profiles dynamically via the Bedrock control plane (`list_inference_profiles()`), caching results for one hour per region to avoid repeated API calls. Global cross-region profiles are sorted to the top of the list.

### The Converse API: message format conversion

The Converse API uses its own message schema — not OpenAI's. The adapter converts between them with two functions:

- `convert_messages_to_converse()` — turns OpenAI-format messages into Converse format, extracting system prompts, converting tool calls to `toolUse` blocks, and merging consecutive same-role messages (which Converse requires).
- `normalize_converse_response()` — turns a Converse response back into an OpenAI-compatible `SimpleNamespace` that the agent loop expects.

A key Converse requirement: messages must strictly alternate between `user` and `assistant` roles. The adapter handles this by merging consecutive messages with the same role and, if needed, inserting placeholder messages to satisfy the rule.

### Sequence: a Bedrock Converse call

```mermaid
sequenceDiagram
    participant AgentLoop as Agent loop
    participant Adapter as bedrock_adapter.py
    participant Boto3 as boto3 client
    participant Bedrock as AWS Bedrock

    AgentLoop->>Adapter: call_converse_stream(region, model, messages, tools)
    Adapter->>Adapter: convert_messages_to_converse(messages)
    Adapter->>Adapter: convert_tools_to_converse(tools)
    Adapter->>Boto3: converse_stream(modelId, messages, toolConfig, inferenceConfig)
    Boto3->>Bedrock: HTTPS POST (SigV4 signed by boto3)
    Bedrock-->>Boto3: event stream
    Boto3-->>Adapter: stream iterator
    Adapter->>Adapter: stream_converse_with_callbacks()
    Adapter-->>AgentLoop: OpenAI-compatible SimpleNamespace
```

### Worked example — using Bedrock with a cross-region profile

```yaml
# ~/.hermes/config.yaml
model:
  provider: bedrock
  name: us.anthropic.claude-sonnet-4-6-20250514-v1:0
```

```bash
# Set your region (or configure ~/.aws/config)
export AWS_REGION=us-east-1
```

To use the global profile (widest capacity):

```yaml
model:
  name: global.anthropic.claude-opus-4-7
```

### Edge cases

**Stale boto3 connection**

boto3 caches HTTPS connections inside the client object. If the connection is dropped (NAT timeout, VPN flap, TCP RST), the next call raises a `botocore.exceptions.ConnectionClosedError` or `urllib3.exceptions.ProtocolError`. The adapter detects these with `is_stale_connection_error()` and calls `invalidate_runtime_client(region)` to evict the cached client for that region, so the next request allocates a fresh connection. You do not need to restart Hermes.

**Bedrock cross-region failover**

If you are using a regional profile (`us.anthropic.*`) and the `us-east-1` region returns a throttling error (`ThrottlingException`), Bedrock may automatically reroute the request within the `us.*` geography. With a `global.*` profile, the routing scope is worldwide. From your perspective, the call takes slightly longer; no special handling is needed in your config.

---

## The Gemini native adapter — bypassing the OpenAI-compat layer

### Why a native adapter is needed

Google provides an OpenAI-compatible endpoint at `https://generativelanguage.googleapis.com/v1beta/openai`. You might expect Hermes to use this and be done with it — it would let Google's Gemini sit alongside OpenAI and other providers with zero special code.

The problem is that this compat layer has been brittle for Hermes's multi-turn agentic tool loop. The source file `agent/gemini_native_adapter.py` documents the specific issues: auth churn, tool-call replay quirks, and thought-signature requirements that the compat layer handles inconsistently.

The native Gemini API at `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` is the **canonical path** and avoids these issues entirely. The Gemini native adapter talks directly to this endpoint.

### How the native adapter works

The adapter (`GeminiNativeClient`) is a facade that looks like an OpenAI SDK client to the rest of Hermes. The agent loop calls `client.chat.completions.create(...)` just as it would for any OpenAI-compatible provider. Internally, `GeminiNativeClient` intercepts this call and routes it through `build_gemini_request()` to translate the payload to Gemini's `generateContent` schema, then calls the native REST API directly via `httpx`.

The Gemini profile registers with `api_mode="chat_completions"` — the same mode used by all OpenAI-compatible providers. But when the agent loop sees that the base URL is a native Gemini endpoint (detected by `is_native_gemini_base_url()`), it substitutes the `GeminiNativeClient` for the standard OpenAI SDK client. The adapter's design means the agent loop never needs to know the difference.

```python
# From plugins/model-providers/gemini/__init__.py
gemini = GeminiProfile(
    name="gemini",
    aliases=("google", "google-gemini", "google-ai-studio"),
    api_mode="chat_completions",   # same mode as OpenAI providers ...
    env_vars=("GOOGLE_API_KEY", "GEMINI_API_KEY"),
    base_url="https://generativelanguage.googleapis.com/v1beta",  # ... but native endpoint
)
```

The native endpoint detection:

```python
# Simplified view of is_native_gemini_base_url()
def is_native_gemini_base_url(base_url: str) -> bool:
    normalized = str(base_url or "").strip().rstrip("/").lower()
    if "generativelanguage.googleapis.com" not in normalized:
        return False
    return not normalized.endswith("/openai")  # /openai suffix = compat layer
```

### Message format translation

The main translation challenge is that Gemini uses a completely different message structure:

- Roles are `user` and `model` (not `assistant`).
- System prompts are a top-level `systemInstruction` field, not a message with `role: system`.
- Tool calls are `functionCall` parts; tool results are `functionResponse` parts.
- The `thoughtSignature` field on thinking-enabled models must round-trip correctly for multi-turn coherence.

`_build_gemini_contents()` handles all of this, extracting system messages, translating roles, and building the `contents` array. `build_gemini_request()` then assembles the full payload including `generationConfig` (temperature, `maxOutputTokens`, thinking config) and `tools`.

One subtle detail: if you omit `maxOutputTokens` on the native API, Gemini applies a low internal default and stops output early with `finishReason: MAX_TOKENS` — truncating tool calls mid-stream and causing retry loops. The adapter always sets `maxOutputTokens`, defaulting to `65535` (the ceiling for all current Gemini text models) when the caller passes `None`.

### Worked example — setting up the Gemini native provider

```bash
# Get an API key from https://aistudio.google.com/apikey
# Set it in your environment
export GOOGLE_API_KEY="AIza..."
```

```yaml
# ~/.hermes/config.yaml
model:
  provider: gemini
  name: gemini-2.5-flash
```

The adapter sends requests directly to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent` — no OpenAI-compat intermediary.

### Edge case — free-tier quota exhaustion

The Gemini free tier has a low daily request cap (for example, 250 requests per day for `gemini-2.5-flash`). Hermes makes 3–10 API calls per user turn, so the free tier is exhausted in a handful of messages. When this happens, the adapter raises a `GeminiAPIError` with `code="gemini_rate_limited"` and appends a message explaining that you need to enable billing on your Google Cloud project. The resolution is to enable billing at `https://aistudio.google.com/apikey` and regenerate your API key.

---

## The Codex Responses adapter — issuer stamping and encrypted reasoning

### The problem: reasoning blobs are provider-sealed

The OpenAI Responses API (used by OpenAI Codex, xAI Grok, GitHub Models, and other compatible endpoints) has a feature called **encrypted reasoning**: after a model produces a reasoning step, the API returns an `encrypted_content` blob. This blob must be sent back on subsequent turns so the model can continue its chain of thought across turns — without it, the model loses its reasoning state.

Here is the problem: the `encrypted_content` blob is **sealed to the endpoint that minted it**. If you start a conversation with Codex and then switch to xAI, and replay Codex's encrypted reasoning blob against xAI's endpoint, xAI will return HTTP 400 `invalid_encrypted_content`. The blob cannot be decrypted by a different endpoint, even one that implements the same protocol.

This is the core issue that **issuer stamping** solves.

### How issuer stamping works

Every time the Codex Responses adapter (`agent/codex_responses_adapter.py`) normalises a response, it records which endpoint minted the `encrypted_content` blob. The `_classify_responses_issuer()` function assigns a stable identifier based on the endpoint type:

```python
# Simplified view of _classify_responses_issuer()
def _classify_responses_issuer(
    *,
    is_xai_responses: bool = False,
    is_github_responses: bool = False,
    is_codex_backend: bool = False,
    base_url: str | None = None,
) -> str:
    if is_xai_responses:
        return "xai_responses"
    if is_github_responses:
        return "github_responses"
    if is_codex_backend:
        return "codex_backend"
    if base_url:
        return f"other:{base_url}"
    return "other"
```

When `_normalize_codex_response()` processes a response, it stamps each reasoning item with this issuer kind under the key `_issuer_kind`:

```python
# Simplified view — stamping during normalisation
raw_item = {"type": "reasoning", "encrypted_content": encrypted}
if issuer_kind:
    raw_item["_issuer_kind"] = issuer_kind  # e.g. "codex_backend"
```

### How the cross-issuer guard works at replay time

When the next turn is being built, `_chat_messages_to_responses_input()` converts the conversation history into Responses API input items. For each reasoning item in a previous assistant message, it checks the `_issuer_kind` stamp against the current endpoint:

```python
# Simplified view — cross-issuer guard
item_issuer = ri.get("_issuer_kind")
if (
    current_issuer_kind is not None
    and item_issuer is not None
    and item_issuer != current_issuer_kind
):
    # Drop this reasoning item — the current endpoint can't decrypt it
    logger.warning(
        "Dropping reasoning item minted by %s while calling %s — "
        "encrypted_content is sealed to its issuer.",
        item_issuer, current_issuer_kind,
    )
    continue
```

The `_issuer_kind` key is a Hermes-internal field. It is stripped from outgoing payloads before they reach the API (alongside the item's `id` field, which is also stripped for `store=False` sessions):

```python
replay_item = {
    k: v for k, v in ri.items()
    if k not in ("id", "_issuer_kind")
}
```

### The session-wide kill switch

The per-item cross-issuer guard is the fine-grained defence. There is also a session-wide kill switch: `replay_encrypted_reasoning=False`. When an endpoint returns HTTP 400 `invalid_encrypted_content` and the retry loop calls `AIAgent._disable_codex_reasoning_replay`, this flag is set to `False`, which causes the converter to skip all reasoning replay for the rest of the session.

The two guards compose:
- `replay_encrypted_reasoning=False` → drop all reasoning replay (session-level kill switch)
- `current_issuer_kind` mismatch → drop only foreign-issuer items (per-item filter, runs when replay is still enabled)

### The Codex provider profile

```python
# From plugins/model-providers/openai-codex/__init__.py
openai_codex = ProviderProfile(
    name="openai-codex",
    aliases=("codex", "openai_codex"),
    api_mode="codex_responses",
    env_vars=(),  # OAuth external — no API key
    base_url="https://chatgpt.com/backend-api/codex",
    auth_type="oauth_external",
)
```

This provider uses `api_mode="codex_responses"` and OAuth-based auth — `env_vars` is empty because the credential is managed externally.

### Worked example — understanding an issuer-stamp mismatch

Suppose you run a session with Codex, which generates reasoning items stamped `"codex_backend"`. You then switch your model to xAI Grok (which uses `api_mode="codex_responses"` with issuer `"xai_responses"`). On the next turn:

1. Hermes sees the conversation history contains reasoning items with `_issuer_kind="codex_backend"`.
2. `current_issuer_kind` is `"xai_responses"`.
3. The cross-issuer guard logs a warning and drops the Codex reasoning items.
4. The request is sent to xAI without the foreign encrypted blobs.
5. xAI responds successfully. New reasoning items are stamped `"xai_responses"` and will replay correctly on subsequent xAI turns.

You do not need to do anything — the guard handles this automatically. The warning in the logs tells you that a provider switch happened mid-session.

---

## Adapter comparison table

| Adapter | api_mode | Auth mechanism | Notable transport quirk |
|---------|----------|---------------|-------------------------|
| `anthropic_adapter.py` | `anthropic_messages` | x-api-key (API key) or Bearer (OAuth/Claude Code) | Three auth variants auto-detected by token prefix; `cache_control` forwarded for prompt caching |
| `bedrock_adapter.py` | `bedrock_converse` | boto3 credential chain (IAM / SSO / IMDS) | Converse API message format; cross-region inference profiles for capacity routing |
| `gemini_native_adapter.py` | `chat_completions` | `x-goog-api-key` header | Bypasses the `/openai` compat layer; always sets `maxOutputTokens=65535` to avoid truncation |
| `codex_responses_adapter.py` | `codex_responses` | OAuth external | Issuer-stamps `encrypted_content` blobs; cross-issuer guard drops foreign reasoning items at replay |

---

← Previous: [Config-Driven Provider Routing and the Four api_mode Values](./config-driven-routing-and-api-modes.md) · Next: [CredentialPool — Rotation Strategies, Cooldowns, and Failover](./credential-pool-and-failover.md) →
