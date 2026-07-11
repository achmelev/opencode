# Does `model.options` pass through to the Chat Completions API call?

Not as a blanket rule. Two separate mechanisms are involved, and only one of
them behaves like a raw passthrough — and only for one provider package.

## Where `model.options` actually goes

`packages/opencode/src/session/llm/request.ts:91` merges four sources into
one flat object per request:

```
options = merge(merge(merge(ProviderTransform.options(...), model.options), agent.options), variant)
```

That merged object is then wrapped by `ProviderTransform.providerOptions(model, options)`
(`packages/opencode/src/provider/transform.ts:1285`) into the AI SDK v5
`providerOptions` shape — `{ [providerKey]: options }` — and handed to
`generateText`/`streamText` as `providerOptions: prepared.params.options`
(`packages/opencode/src/session/llm.ts:239,316`).

## What happens to it next depends on the target AI-SDK provider package

Each AI-SDK provider package decides for itself what to do with
`providerOptions[key]`:

- **`@ai-sdk/openai-compatible`** (generic OpenAI-compatible endpoints, e.g.
  LiteLLM proxies) — the provider key is the provider's own id
  (`model.providerID.split(".")[0]`). This package's implementation is
  permissive: it spreads whatever keys land in `providerOptions[<providerID>]`
  directly into the raw JSON body sent to `.../chat/completions`. So for this
  specific provider package, arbitrary keys (e.g. `chat_template_kwargs`)
  become literal extra fields in the Chat Completions request body — matching
  Continue's `extraBodyProperties` behavior.
- **Provider packages with a typed `providerOptions` schema** — `@ai-sdk/openai`,
  `@ai-sdk/anthropic`, `@ai-sdk/google`, etc. — do **not** forward arbitrary
  unknown keys to the wire. They only read the specific option names they
  define (`reasoningEffort`, `thinking`, `promptCacheKey`, ...) and silently
  ignore everything else.

## A second, unrelated path (hardcoded providers only)

A handful of hardcoded providers (openai, anthropic, azure, bedrock,
google-vertex, ...) register a custom `getModel` loader in
`packages/opencode/src/provider/provider.ts` (e.g. lines 205, 264, 365). For
those, `{...provider.options, ...model.options}` is passed as constructor-time
settings to `sdk.languageModel(id, settings)` — a narrower surface than the
per-request `providerOptions` body-passthrough above, gated by whatever that
specific loader chooses to read.

## Bottom line

"Any key in `model.options` passes through to the Chat Completions API call"
is correct specifically for `@ai-sdk/openai-compatible` targets (e.g. a
LiteLLM gateway) — not a general rule across every provider in opencode.
