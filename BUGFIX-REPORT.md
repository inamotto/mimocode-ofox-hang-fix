# Bug Report: Infinite Hang When Using Custom OpenAI-Compatible API (ofox) After `/login`

## Summary

When a user configures a custom OpenAI-compatible provider (e.g., **ofox** at `https://api.ofox.ai/v1`) and attempts to use a model, MiMoCode hangs indefinitely without producing any error. After investigation, we identified **two independent root causes** and one **secondary issue**:

1. **Primary root cause**: Clash Verge TUN mode intercepting DNS for `ofox.ai`, causing TLS handshake to hang indefinitely
2. **Secondary root cause**: Missing default `headerTimeout` for custom providers in MiMoCode's `provider.ts`
3. **Model ID issue**: ofox requires date-suffixed model IDs (e.g., `deepseek/deepseek-v4-flash-0731`), bare IDs return 404

## Environment

- **MiMoCode version**: 0.1.13
- **Provider**: Custom OpenAI-compatible (e.g., ofox `https://api.ofox.ai/v1`)
- **Config file**: `~/.config/mimocode/mimocode.jsonc`
- **OS**: Windows 11
- **Network**: Clash Verge with TUN mode enabled

## Root Cause Analysis

### Root Cause 1: Clash Verge TUN Mode DNS Interception (Primary)

**The primary cause of the infinite hang was NOT MiMoCode code — it was the network proxy configuration.**

Clash Verge's TUN mode intercepted DNS requests for `ofox.ai` and returned fake IPs (198.18.0.x range). This caused:
- TCP connection to succeed (fake IP was reachable)
- TLS handshake to hang indefinitely (couldn't complete with the fake endpoint)
- No timeout fired because the TCP connection was technically "established"

**Verification**: Direct `curl` to `https://api.ofox.ai/v1/models` hung indefinitely before the fix, returned results in ~2 seconds after.

**Fix**: Added a domain-suffix rule in Clash Verge for `ofox.ai` to route through a working proxy node:
- Rule type: "匹配域名后缀" (domain suffix match)
- Domain: `ofox.ai`
- This covers both `ofox.ai` and `api.ofox.ai` with a single rule

### Root Cause 2: Missing Default `headerTimeout` (Secondary)

Even after fixing the network issue, MiMoCode's code has a secondary issue: custom OpenAI-compatible providers don't get a default `headerTimeout`, so if the server doesn't respond for any reason (network issues, slow upstream, etc.), the request hangs indefinitely.

**The Timeout Mechanism in `provider.ts`**

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

**Why Custom Providers Don't Get a Header Timeout**

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

**The Complete Hang Path (when network is broken)**

```
getLanguage(model)
  → resolveSDK(model, ...)
      → fetchFn(input, opts)  ← awaiting response headers...
          (no headerTimeout controller exists)
          (chunkTimeout only applies via wrapSSE AFTER headers arrive)
          → HANGS FOREVER if server never responds
```

When the ofox endpoint doesn't respond (due to Clash Verge TUN mode intercepting DNS), the `await fetchFn(...)` promise never resolves. No timeout fires because:
1. `headerTimeoutCtl` is `undefined` — no header timeout
2. `requestTimeoutCtl` is `undefined` — no request timeout (only set when `options["timeout"]` is a number)
3. `chunkAbortCtl` exists (8 min), but `wrapSSE()` is only called **after** `res` is received, so it never gets a chance to fire

### Issue 3: Ofox Model IDs Require Date Suffix

Ofox's API requires model IDs to include a date suffix. Bare IDs without a date suffix return 404:

| Model ID (WRONG) | Model ID (CORRECT) | Status |
|---|---|---|
| `deepseek/deepseek-v4-flash` | `deepseek/deepseek-v4-flash-0731` | ❌ 404 → ✅ Works |
| `deepseek/deepseek-v4-pro` | `deepseek/deepseek-v4-pro-0813` | ❌ 404 → ✅ Works |
| `deepseek/deepseek-v4-flash-vision-exp` | `deepseek/deepseek-v4-flash-vision-exp` | ✅ Works (no date suffix needed) |

Full list of available DeepSeek models on ofox (confirmed via `GET /v1/models`):
- `deepseek/deepseek-v4-flash-0731` — Latest Flash
- `deepseek/deepseek-v4-flash-0423` — Older Flash
- `deepseek/deepseek-v4-flash-vision-exp` — Vision model
- `deepseek/deepseek-v4-pro-0813` — Latest Pro
- `deepseek/deepseek-v4-pro-0423` — Older Pro
- `deepseek/deepseek-v3.2` — V3.2

## The Fix

### Fix 1: Network (Clash Verge) — User Configuration

Add a domain-suffix rule in Clash Verge for `ofox.ai`:
1. Open Clash Verge → Rules → Add Rule
2. Rule type: "匹配域名后缀" (domain suffix match)
3. Domain: `ofox.ai`
4. Select a working proxy node

This covers both `ofox.ai` and `api.ofox.ai` with a single rule.

### Fix 2: Code — Default `headerTimeout` for All Providers

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

**Behavior Matrix After Fix**

| Provider config `headerTimeout` | Before fix | After fix |
|---|---|---|
| Not set (custom/ofox) | No timeout → **infinite hang** | 5 min default → **times out with error** |
| `false` (explicitly disabled) | No timeout | No timeout (unchanged) |
| `60000` (custom value) | 1 min timeout | 1 min timeout (unchanged) |
| `300000` (openai provider) | 5 min timeout | 5 min timeout (unchanged) |

### Fix 3: Config — Use Correct Model IDs

Update `~/.config/mimocode/mimocode.jsonc` to use date-suffixed model IDs:

```jsonc
{
  "provider": {
    "ofox": {
      "name": "OfoxAI",
      "npm": "@ai-sdk/openai-compatible",
      "only_configured_models": true,
      "env": ["OFOX_API_KEY"],
      "models": {
        "deepseek/deepseek-v4-flash-0731": {
          "name": "DeepSeek V4 Flash 0731"
        },
        "deepseek/deepseek-v4-pro-0813": {
          "name": "DeepSeek V4 Pro 0813"
        },
        "deepseek/deepseek-v4-flash-vision-exp": {
          "name": "DeepSeek V4 Flash Vision"
        }
      },
      "options": {
        "baseURL": "https://api.ofox.ai/v1",
        "apiKey": "{env:OFOX_API_KEY}",
        "headerTimeout": 300000,
        "chunkTimeout": 480000
      }
    }
  }
}
```

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

## Additional Notes

### Ofox is NOT Geo-Blocked in China

Contrary to initial assumptions, ofox.ai is NOT blocked in mainland China:
- 88.52% of visitors are from China (HypeStat)
- API infrastructure on Alibaba Cloud (Hong Kong, Japan)
- Website on Cloudflare CDN
- Domain registered 2025-12-12, registrant country SG

Connection issues were caused by local proxy configuration (Clash Verge TUN mode), not geo-restriction.

## Files Changed

1. `packages/opencode/src/provider/provider.ts` — Added default `headerTimeout` fallback
2. `packages/opencode/test/provider/provider-chunk-timeout.test.ts` — Added test for default header timeout behavior
3. `~/.config/mimocode/mimocode.jsonc` — Added ofox provider with correct model IDs and timeout config
