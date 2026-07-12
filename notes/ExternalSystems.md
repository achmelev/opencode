# External systems opencode talks to, besides the LLM provider

Found by grepping every hardcoded external hostname across `packages/opencode/src` and
`packages/core/src`, then tracing each one to its call site. They split into "automatic,
on by default" and "opt-in."

## Automatic / on by default

**1. models.dev — the model/provider catalog.** `models.dev` (overridable via
`OPENCODE_MODELS_URL`, `packages/core/src/models-dev.ts:138`) is fetched to populate
pricing, context/output limits, and capability metadata for the ~75+ built-in providers —
this is where the catalog defaults described in `notes/ModelLimits.md` come from.

**2. Auto-update check.** Every time the interactive TUI starts (`cli/tui/worker.ts:61`
calls `upgrade()`), opencode checks for a newer version — and **for patch releases, can
silently self-install the update** without asking (`cli/upgrade.ts:8-42`). It queries
whichever install method you used: `api.github.com/repos/anomalyco/opencode/releases/latest`,
`formulae.brew.sh` (Homebrew), `community.chocolatey.org`, or the Scoop bucket on
`raw.githubusercontent.com` (`installation/index.ts:219-258`). Controlled by config
`autoupdate` (`true`/`false`/`"notify"`) or `OPENCODE_DISABLE_AUTOUPDATE`.

**3. npm registry.** Every time config loads for a directory, opencode forks a background
`npm install` of `@opencode-ai/plugin` into that directory (`config/config.ts:438-457`,
`"background dependency install failed"` on failure) — this hits the npm registry
automatically, not just when you explicitly configure a custom-provider `npm:` package.
Any custom provider's `npm` field (like `@ai-sdk/openai-compatible` in
`examples/opencode.jsonc`) triggers the same install path on demand.

## Opt-in (only if you use the feature)

**4. Session sharing** (`share/share-next.ts`) — uploads session/message content to
`opncd.ai` (or your own `enterprise.url` config) to generate a shareable link. This is the
one that actually **sends conversation content off-machine**, not just metadata.

Nothing is shared automatically by default — `SessionShare.create` (`share/session.ts:39-46`)
only triggers an upload on session creation if `conf.share === "auto"` or the
`flags.autoShare` runtime flag is set (or the deprecated `autoshare: true`, which
`config.ts` normalizes into `share: "auto"`). Neither is true out of the box, so normal use
never uploads a session on its own.

That's distinct from whether *manual* sharing (explicitly running the share command/API on
a session) is available: `SessionShare.share` (`share/session.ts:26-32`) only refuses when
`conf.share === "disabled"`. Since the default `share` value is unset — neither `"disabled"`
nor `"auto"` — the manual-share capability is available out of the box; it just isn't used
unless you explicitly invoke it. Lock it down with `share: "disabled"`, or disable the
feature entirely with `OPENCODE_DISABLE_SHARE=true`.

**5. OpenCode Console** (`console.opencode.ai`) — account/org login and billing for
OpenCode Zen/Go (the hosted model marketplace), and remote config distribution for
enterprise orgs (`cli/cmd/account.ts:18`, `core/plugin/provider/opencode.ts:16`). Only
contacted if you `/connect` to Zen/Go or log into a Console account.

**6. GitHub API** (`api.github.com`, `api.opencode.ai/get_github_app_installation`) — used
by the `opencode github` command/plugin for PR/issue triage and installing opencode's
GitHub App (`cli/cmd/github.handler.ts:323`). Only hit when you use GitHub-integration
commands.

**7. MCP servers** — entirely user-configured (`mcp` config block); opencode just speaks
the protocol to whatever host/port you point it at. Not opencode's own infrastructure, but
a real external-communication channel you control.

**8. `WebFetch`/`WebSearch` tools** — model-directed, not opencode-directed: the agent can
fetch an arbitrary URL or run a web search if the tool is permitted. Destination is chosen
at runtime by the model/user, unlike everything else in this list which hits a fixed host.

## Telemetry, precisely

**OpenTelemetry export is off by default and goes nowhere by default.**
`packages/core/src/observability/otlp.ts:7,50-56` gates everything on the standard
`OTEL_EXPORTER_OTLP_ENDPOINT` env var — if it's unset, `loggers()` returns `[]` and the
tracing layer is `Layer.empty`. There's no opencode-owned telemetry backend it phones home
to; enabling `experimental.openTelemetry: true` only activates instrumentation, and the
destination is whatever OTLP collector *you* point the standard `OTEL_EXPORTER_OTLP_*` env
vars at.

**Sentry** does exist in the codebase (`@sentry/solid`, `@sentry/vite-plugin` in the root
`package.json` catalog), but only in `packages/app/src/{app,entry,pages/error}.tsx` and
`packages/desktop/src/renderer/index.tsx` — i.e. only if you're running the web UI or the
Electron desktop app. It's not present anywhere in `packages/opencode` (the
CLI/TUI/headless-server package), so a pure CLI/TUI/server user never triggers it.

## Not really "external" but worth noting

**mDNS** (`server/mdns.ts`) — announces a running `opencode serve` on the local network via
Bonjour (`opencode.local`) so other devices on your LAN can discover it. Local network
only, not internet.

## Summary

| System | Default state | What it sends |
|---|---|---|
| models.dev | automatic | none (read-only catalog fetch) |
| Update check / self-upgrade | automatic | version string; may download+install a new binary |
| npm registry | automatic (per project dir) | package name/version requests |
| Session sharing (`opncd.ai`) | capability available by default; upload only on explicit manual share or `share: "auto"` | session + message content |
| OpenCode Console | opt-in (`/connect`, login) | auth tokens, account/org data |
| GitHub API | opt-in (github commands) | repo/owner info, app installation lookups |
| MCP servers | opt-in (fully user-configured) | whatever the configured server exchanges |
| WebFetch / WebSearch tools | opt-in, model-directed | arbitrary URL/query chosen at runtime |
| OpenTelemetry | off, user-controlled destination | traces/logs, only if `OTEL_EXPORTER_OTLP_ENDPOINT` is set |
| Sentry | only in web/desktop GUI, never in CLI/TUI/server | client-side error reports |
| mDNS | automatic when `opencode serve` runs | LAN-only service announcement |
