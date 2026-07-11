# Configuring HTTP headers on a provider

Headers are handled by a dedicated code path, separate from the generic
`model.options` → AI SDK `providerOptions` body-passthrough (see
`notes/ModelOptions.md`). There are exactly two supported places to set them.

## 1. `provider.options.headers` — applies to every model under the provider

Read directly in `resolveSDK` (`packages/opencode/src/provider/provider.ts:1659,1704-1708`).
It's part of `options = {...provider.options}`, which is passed straight into
the AI-SDK factory call, e.g. `createOpenAICompatible({ name, baseURL, apiKey,
headers, ... })`. This becomes real default HTTP headers on the underlying
fetch client for every model under that provider — not JSON body content.

```jsonc
"provider": {
  "muster-litellm": {
    "npm": "@ai-sdk/openai-compatible",
    "options": {
      "baseURL": "{env:OPENAI_API_BASE_ENV}",
      "apiKey": "{env:OPENAI_API_KEY_ENV}",
      "headers": {
        "User-Agent": "continue-dev",
        "x-litellm-customer-id": "Muster GmbH",
      },
    },
  },
},
```

## 2. `model.headers` — per-model, layered on top of provider headers

A dedicated *sibling* field on the model object (`headers: Record<string,
string>` in the V1 config schema) — **not** nested under `model.options`. It's
applied twice:

- At SDK-client construction time: `provider.ts:1704-1708` merges it on top of
  `provider.options.headers` (and it's part of the client cache key, so a
  model with different headers gets its own client instance).
- At per-request time: `packages/opencode/src/session/llm/request.ts:187-203`
  builds the actual headers sent with each `generateText`/`streamText` call as
  `{...opencode's own tracing headers, ...model.headers, ...pluginHeaders}`,
  where `pluginHeaders` comes from the `chat.headers` plugin hook
  (`request.ts:134-146`) and wins on conflicts since it's spread last.

```jsonc
"provider": {
  "muster-litellm": {
    "npm": "@ai-sdk/openai-compatible",
    "options": { "baseURL": "...", "apiKey": "..." },
    "models": {
      "Qwen/Qwen3.6-35B-A3B-FP8": {
        "headers": {
          "x-litellm-tags": "continue,ide,dev",
        },
      },
    },
  },
},
```

The native (non-AI-SDK) route runtime follows the same shape: provider
headers via `providerHeaders(provider.options.headers)` in
`native-runtime.ts:101`, merged with per-call headers built from
`{...model.headers, ...headers}` in `native-request.ts:159`.

## The gotcha: `model.options.headers` is not the same thing

Nesting `headers` *inside* `model.options` instead of using the sibling
`model.headers` field bypasses all of the above — `model.options` only
reaches the generic per-request `providerOptions` merge described in
`notes/ModelOptions.md`. For an `@ai-sdk/openai-compatible` provider, a
`headers` key placed there gets spread as
`providerOptions["<providerID>"].headers` and ends up as a literal
`"headers"` field inside the outgoing JSON **body**, not as actual HTTP
request headers. Use the sibling `model.headers` field instead.

## Summary

| Location | Scope | Mechanism |
|---|---|---|
| `provider.options.headers` | every model under the provider | AI-SDK client factory constructor option |
| `model.headers` (sibling field) | one model, layered over provider headers | client construction + per-request `headers`, plus `chat.headers` plugin hook |
| `model.options.headers` | — | **wrong place** — ends up as a JSON body field for openai-compatible providers, not an HTTP header |
