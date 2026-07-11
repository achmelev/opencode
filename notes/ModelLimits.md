# How opencode uses `model.limit.context` and `model.limit.output`

A model's `limit` object has three fields — `context`, `output`, and an optional `input`
(`packages/core/src/v1/config/provider.ts:41-47`). `context` and `output` are the two you'll
normally set; they drive two independent mechanisms.

## Where the values come from

For models known to [models.dev](https://models.dev), `limit` is seeded from that catalog
(`packages/opencode/src/provider/provider.ts:1220-1224`, `fromModelsDevModel`). Your config's
`provider.<id>.models.<id>.limit` then overrides it key-by-key on top of the catalog value
(`provider.ts:1485-1489`):

```ts
limit: {
  context: model.limit?.context ?? existingModel?.limit?.context ?? 0,
  input: model.limit?.input ?? existingModel?.limit?.input,
  output: model.limit?.output ?? existingModel?.limit?.output ?? 0,
},
```

**For a fully custom provider/model that isn't in the catalog** (e.g. a self-hosted
LiteLLM model), `existingModel` is `undefined`, so any `limit` field you don't set
explicitly defaults to `0` (`context`/`output`) or stays `undefined` (`input`). As shown
below, `0`/`undefined` isn't "zero tokens" — it's treated as "unknown," which silently
disables the mechanisms that depend on it. This is why `examples/opencode.jsonc` sets both
`limit.context` and `limit.output` explicitly for `muster-litellm`'s model.

## `limit.output` — caps the actual request parameter

`ProviderTransform.maxOutputTokens` (`packages/opencode/src/provider/transform.ts:1345-1346`)
is the single source of truth for how many tokens the model is allowed to generate:

```ts
export const OUTPUT_TOKEN_MAX = 32_000 // transform.ts:18

export function maxOutputTokens(model: Provider.Model, outputTokenMax = OUTPUT_TOKEN_MAX): number {
  return Math.min(model.limit.output, outputTokenMax) || outputTokenMax
}
```

- The result is `Math.min(limit.output, cap)`, where `cap` defaults to `32_000` and can be
  lowered via the runtime flag `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX`
  (`packages/opencode/src/effect/runtime-flags.ts:52`).
- The trailing `|| outputTokenMax` is a deliberate safety net: if `limit.output` is `0` or
  otherwise falsy (unset, no catalog entry), `Math.min(0, cap)` would be `0` — which would
  cap every response at zero tokens. The `||` catches that and falls back to the full cap
  instead, so an unconfigured custom model isn't silently gagged.
- This computed value is threaded straight into the request as the real, top-level
  parameter — `session/llm/request.ts:129` (`maxOutputTokens: ProviderTransform.maxOutputTokens(...)`)
  → `session/llm.ts:320` (`streamText({ maxOutputTokens: prepared.params.maxOutputTokens, ... })`),
  and the equivalent field on the native (non-AI-SDK) runtime path
  (`session/llm/native-request.ts:161-162`).

`limit.output` also sizes Anthropic extended-thinking token budgets for the `high`/`max`
reasoning-effort variants (`provider/transform.ts:944-957`, `:1182`): e.g.
`budgetTokens: Math.min(16_000, Math.floor(model.limit.output / 2 - 1))` for `high` and
`Math.min(31_999, model.limit.output - 1)` for `max`. A model's declared output capacity
therefore also caps how large a "thinking" budget opencode will ever request for it.

## `limit.context` (and `limit.input`) — drives auto-compaction, not the request itself

Unlike `limit.output`, `limit.context` is never sent to the provider directly. It's consumed
by `packages/opencode/src/session/overflow.ts`, which decides when a session needs
compaction (summarizing/trimming history because it's approaching the model's window):

```ts
const COMPACTION_BUFFER = 20_000

export function usable(input: { cfg; model; outputTokenMax? }) {
  const context = input.model.limit.context
  if (context === 0) return 0

  const reserved =
    input.cfg.compaction?.reserved ??
    Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))
  return input.model.limit.input
    ? Math.max(0, input.model.limit.input - reserved)
    : Math.max(0, context - ProviderTransform.maxOutputTokens(input.model, input.outputTokenMax))
}

export function isOverflow(input: { cfg; tokens; model; outputTokenMax? }) {
  if (input.cfg.compaction?.auto === false) return false
  if (input.model.limit.context === 0) return false
  const count = input.tokens.total || (input.tokens.input + input.tokens.output + input.tokens.cache.read + input.tokens.cache.write)
  return count >= usable(input)
}
```

- **`usable()`** computes the token budget available for conversation history: it reserves
  a chunk for the model's reply (`min(20_000, maxOutputTokens(model))`, or `cfg.compaction.reserved`
  if configured) and subtracts it from either `limit.input` (if the provider tracks input
  tokens separately from total context — some do) or `limit.context`.
- **`isOverflow()` is the actual trigger**: it's checked in two places —
  `session/processor.ts:477-482` right after each assistant turn finishes (using that
  turn's own reported usage), and `session/prompt.ts:1161-1168` before starting the next
  turn (using the previous message's usage). Either one flags `needsCompaction`, which
  causes the session loop to run a compaction pass before continuing
  (`SessionCompaction.Service` wraps the same `overflow.ts` functions,
  `session/compaction.ts:17,131,168`).
- **`limit.context === 0` disables the whole mechanism** — treated as "unknown context
  size," not "zero." A custom model with no catalog entry and no explicit `limit.context`
  in config never auto-compacts, and its usage is reported as unbounded.

`limit.context` is also surfaced read-only to external consumers, not just used internally:

- **ACP** (Zed's Agent Client Protocol): `acp/usage.ts:113-119` (`findContextLimit`) reports
  it to the client so Zed can render its own context-usage indicator.
- **`opencode run`**: `cli/cmd/run/runtime.boot.ts:112-123` builds a
  `providerID/modelID → limit.context` lookup table across all configured providers,
  filtering out any model whose limit is missing or `<= 0` — the same "0/absent means
  unknown, exclude it" convention as `isOverflow`.

## Summary

| Field | What it caps | Consumed by | Effect of leaving it unset (`0`) |
|---|---|---|---|
| `limit.output` | The literal `maxOutputTokens` sent on every request, plus Anthropic thinking-budget sizing | `ProviderTransform.maxOutputTokens` (`transform.ts:1345`) | **Not** capped at 0 — falls back to the 32k (or flag-configured) runtime default via `\|\| outputTokenMax` |
| `limit.context` (+ optional `limit.input`) | When auto-compaction kicks in | `session/overflow.ts` (`usable`, `isOverflow`) | Auto-compaction is disabled entirely; usage is reported as unbounded to ACP/CLI consumers |

Both ultimately interact: the reserved buffer in `usable()` is derived from
`maxOutputTokens(model)`, which itself is derived from `limit.output` — so raising a
custom model's `limit.output` also shrinks the effective history budget before
auto-compaction triggers, and vice versa.
