# Bug Report: Infinite Hang When Using Custom OpenAI-Compatible API (ofox) After `/login`

## Summary

When a user configures a custom OpenAI-compatible provider (e.g., **ofox** at `https://api.ofox.ai/v1`) and attempts to use a model, MiMoCode hangs indefinitely without producing any error. The root cause is a missing default `headerTimeout` for providers that are not the built-in `openai` provider.

## Environment

- **MiMoCode version**: 0.1.13
- **Provider**: Custom OpenAI-compatible (e.g., ofox `https://api.ofox.ai/v1`)
- **Config file**: `~/.config/mimocode/mimocode.jsonc`
- **OS**: Windows

## How to Reproduce

1. Run `mimo` (or open the TUI) and execute `/login`.
2. Select "ofox" or "Custom provider" from the provider list.
3. Enter the base URL (`https://api.ofox.ai/v1`), API key, and model ID.
4. Select a model (e.g., `openai/gpt-5.6-sol`).
5. Send a message to the model.
6. **Observe**: The TUI hangs indefinitely at the "thinking" stage with no error, no timeout, and no response.

## Root Cause Analysis

### The Timeout Mechanism in `provider.ts`

MiMoCode wraps every HTTP fetch call to provider APIs with a custom timeout layer in `resolveSDK()` (in `packages/opencode/src/provider/provider.ts`). This wrapper has three independent timeout mechanisms:

| Timeout | Default | Controlled by | What it protects |
|---|---|---|---|
| **Header timeout** | `DEFAULT_OPENAI_HEADER_TIMEOUT` (5 min / 300,000 ms) | `options["headerTimeout"]` | Server not sending response headers |
| **Chunk timeout** | `DEFAULT_CHUNK_TIMEOUT` (8 min / 480,000 ms) | `options["chunkTimeout"]` | Server not sending response body chunks (SSE only) |
| **Request timeout** | Not set by default | `options["timeout"]` | Overall request duration |

The relevant code (before the fix):

```typescript
// provider.ts, line 1680 (BEFORE)
const headerTimeout = options["headerTimeout"]   // ← undefined for custom providers!
const chunkTimeout =
  typeof userChunkTimeout === "number"
    ? userChunkTimeout
    : DEFAULT_CHUNK_TIMEOUT

// Inside the fetch wrapper:
const headerTimeoutMs = headerTimeout === false ? undefined : headerTimeout
const headerTimeoutCtl = typeof headerTimeoutMs === "number"
  ? timeoutController(headerTimeoutMs)
  : undefined   // ← undefined! No header timeout created.
```

### Why Custom Providers Don't Get a Header Timeout

The `custom()` function in `provider.ts` returns loaders for **specific** provider IDs (e.g., `openai`, `xiaomi`, `xai`, `anthropic`, `azure`, `opencode`, etc.). Each loader can set `options` that get merged into the provider's configuration. Only the `openai` loader sets `headerTimeout`:

```typescript
// provider.ts, the ONLY loader that sets headerTimeout
openai: () =>
  Effect.succeed({
    autoload: false,
    async getModel(sdk: any, modelID: string, _options?: Record<string, any>) {
      return sdk.responses(modelID)
    },
    options: { headerTimeout: DEFAULT_OPENAI_HEADER_TIMEOUT },  // ← only here!
  }),
```

A custom provider like "ofox" (or any user-added OpenAI-compatible provider) is **not** in this list. It falls through to the default `resolveSDK` path:
- `options["headerTimeout"]` is `undefined` (never set by any loader)
- `headerTimeoutMs` becomes `undefined`
- `headerTimeoutCtl` (the `AbortController` that would abort the fetch after 5 minutes) is **never created**
- The only active timeout is `chunkTimeout` (8 min), but `wrapSSE()` only fires **after response headers are received** — so it never triggers when the server is completely unresponsive

### The Complete Hang Path

```
getLanguage(model)
  → resolveSDK(model, ...)
      → fetchFn(input, opts)  ← awaiting response headers...
          (no headerTimeout controller exists)
          (chunkTimeout only applies via wrapSSE AFTER headers arrive)
          → HANGS FOREVER if server never responds
```

When the ofox endpoint doesn't respond (due to network issues, proxy misconfiguration, firewall, or simply slow upstream), the `await fetchFn(...)` promise never resolves. No timeout fires because:
1. `headerTimeoutCtl` is `undefined` — no header timeout
2. `requestTimeoutCtl` is `undefined` — no request timeout (only set when `options["timeout"]` is a number)
3. `chunkAbortCtl` exists (8 min), but `wrapSSE()` is only called **after** `res` is received, so it never gets a chance to fire

## The Fix

**File**: `packages/opencode/src/provider/provider.ts`, line 1680

**Before**:
```typescript
const headerTimeout = options["headerTimeout"]
```

**After**:
```typescript
const headerTimeout = options["headerTimeout"] ?? DEFAULT_OPENAI_HEADER_TIMEOUT
```

This applies the `DEFAULT_OPENAI_HEADER_TIMEOUT` (5 minutes) as a fallback for **all** providers, not just the built-in `openai` provider. The `??` (nullish coalescing) operator correctly:
- Defaults to 5 minutes when `headerTimeout` is `undefined` (not set) — **the fix**
- Preserves explicit numeric values (e.g., `60_000`) — no change
- Preserves `false` (explicit disable) because `??` does not treat `false` as nullish
- Preserves `null` as a default to 5 minutes

### Behavior Matrix After Fix

| Provider config `headerTimeout` | Before fix | After fix |
|---|---|---|
| Not set (custom/ofox) | No timeout → **infinite hang** | 5 min default → **times out with error** |
| `false` (explicitly disabled) | No timeout | No timeout (unchanged) |
| `60000` (custom value) | 1 min timeout | 1 min timeout (unchanged) |
| `300000` (openai provider) | 5 min timeout | 5 min timeout (unchanged) |

## Protocol: Responses API vs Chat Completions API

### Which Protocol Does MiMoCode Use?

MiMoCode uses **different API endpoints** depending on the provider:

| Provider type | SDK call | Wire protocol | Endpoint |
|---|---|---|---|
| Built-in `openai` | `sdk.responses()` | **Responses API** | `/v1/responses` |
| Built-in `xai` | `sdk.responses()` | **Responses API** | `/v1/responses` |
| Built-in `azure` | `sdk.responses()` | **Responses API** | `/v1/responses` |
| Built-in `github-copilot` | `sdk.responses()` or `sdk.chat()` | Responses / Chat | `/v1/responses` or `/v1/chat/completions` |
| **Custom OpenAI-compatible** (ofox, OpenRouter, etc.) | `sdk.languageModel()` | **Chat Completions API** | `/v1/chat/completions` |

### Why This Matters

The ofox API supports **both** endpoints at `https://api.ofox.ai/v1`:
- `/v1/chat/completions` — OpenAI-compatible Chat Completions
- `/v1/responses` — OpenAI Responses API (newer, with better prompt caching)

MiMoCode routes custom OpenAI-compatible providers through `sdk.languageModel()`, which uses **Chat Completions** (`/v1/chat/completions`). The Responses API (`/v1/responses`) offers better cache hit rates because it has native prompt caching support, whereas Chat Completions does not support server-side prompt caching through the same mechanism.

**For ofox specifically**, since it supports both protocols, using the Responses API would provide better cache efficiency. However, changing this for all custom providers would require verifying that each custom endpoint supports the Responses API (not all do).

## Network and Proxy Considerations

The user noted that network issues may contribute to the hang and suggested using system proxy.

### How Proxy Works in MiMoCode

- MiMoCode uses Bun's native `fetch` for all HTTP requests to provider APIs.
- Bun's `fetch` respects standard proxy environment variables:
  - `HTTPS_PROXY` / `https_proxy` — for HTTPS requests
  - `HTTP_PROXY` / `http_proxy` — for HTTP requests
  - `NO_PROXY` / `no_proxy` — to bypass proxy for specific hosts

### Windows-Specific Note

On Windows, the system proxy is often configured via **WinHTTP** (registry), not as environment variables. Bun's `fetch` may not automatically pick up WinHTTP proxy settings. If the ofox endpoint (`api.ofox.ai`) is only reachable through a corporate proxy, the user should:

```powershell
# Set proxy explicitly
$env:HTTPS_PROXY = "http://proxy.example.com:8080"
$env:HTTP_PROXY = "http://proxy.example.com:8080"
```

After setting these and restarting MiMoCode, the `headerTimeout` fix ensures that even if the proxy is misconfigured, the request will time out after 5 minutes instead of hanging forever.

## Workaround (Before Fix is Released)

Users experiencing this issue can add `headerTimeout` to their provider config as a temporary workaround:

```jsonc
{
  "provider": {
    "ofox": {
      "name": "OfoxAI",
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "https://api.ofox.ai/v1",
        "apiKey": "{env:OFOX_API_KEY}",
        "headerTimeout": 300000,
        "chunkTimeout": 480000
      },
      "models": {
        "openai/gpt-5.6-sol": { "name": "GPT-5.6" }
      }
    }
  }
}
```

## Files Changed

1. `packages/opencode/src/provider/provider.ts` — Added default `headerTimeout` fallback
2. `packages/opencode/test/provider/provider-chunk-timeout.test.ts` — Added test for default header timeout behavior
