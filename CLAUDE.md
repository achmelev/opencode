# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

OpenCode is an open source AI coding agent, distributed as a CLI/TUI, a headless HTTP server, a shared web UI, and an Electron desktop app. It's a Bun/TypeScript monorepo built on Effect (v4 beta), Drizzle, Hono, and SolidJS.

The repo root also has `AGENTS.md`, which is the canonical style guide (branch/commit conventions, code style, Effect conventions). Read it — this file complements it with commands and architecture, and does not repeat its contents. Several packages have their own `AGENTS.md` with package-specific rules (`packages/opencode/AGENTS.md`, `packages/opencode/test/AGENTS.md`, `packages/llm/AGENTS.md`, `packages/schema/AGENTS.md`, `packages/codemode/AGENTS.md`, `packages/effect-drizzle-sqlite/AGENTS.md`, `packages/app/AGENTS.md`, `packages/desktop/AGENTS.md`, `packages/stats/AGENTS.md`) — check for one in the package you're editing before making changes there.

- Default branch: `dev` (not `main`; a local `main` ref may not exist — diff against `dev`/`origin/dev`).
- Package manager: `bun@1.3.14` (see `packageManager` in root `package.json`); a pre-push hook enforces the Bun version matches.

## Commands

Run from the repo root unless noted.

```bash
bun install              # install deps (workspaces)
bun dev                  # run the OpenCode TUI against packages/opencode itself
bun dev <directory>      # run the TUI against another directory/repo
bun dev serve            # run the headless API server (port 4096)
bun dev web              # run server + open the web interface
bun dev:desktop          # run the Electron desktop app (packages/desktop)
bun dev:web              # run the shared web app dev server (packages/app)
bun dev:console          # run packages/console/app
bun dev:stats            # run the stats site (packages/stats/app), via sst shell
bun dev:storybook        # run Storybook (packages/storybook)

bun lint                 # oxlint across the repo
bun typecheck            # bun turbo typecheck (runs tsgo --noEmit in every package)
```

### Tests

**Tests cannot be run from the repo root** — a guard (`do-not-run-tests-from-root`) blocks it. Always `cd` into (or `--cwd`) the specific package:

```bash
cd packages/opencode && bun test --timeout 30000 --only-failures
cd packages/opencode && bun test path/to/some.test.ts   # single file
cd packages/opencode && bun test -t "test name"         # by name

cd packages/core && bun test --only-failures
cd packages/llm && bun test --timeout 30000 --only-failures
cd packages/tui && bun test --timeout 30000 --only-failures
cd packages/sdk-next && bun test --timeout 5000
cd packages/client && bun test --timeout 5000
cd packages/codemode && bun test
cd packages/app && bun run test:unit      # bun test --preload ./happydom.ts ./src
cd packages/app && bun run test:e2e       # Playwright
cd packages/ui && bun test src --only-failures
cd packages/session-ui && bun test src --only-failures
```

Per-package typecheck: always run `bun typecheck` from inside the package directory (it wraps `tsgo --noEmit`), never invoke `tsc`/`tsgo` directly.

### Generated code / SDKs

- After changing the public Protocol or Server `HttpApi`, regenerate from `packages/opencode`: `./script/generate.ts` (regenerates `packages/sdk` OpenAPI + the legacy JS SDK). Never hand-edit `src/generated` or `src/generated-effect` under `packages/client`.
- To regenerate the legacy JS SDK directly: `./packages/sdk/js/script/build.ts`.
- `packages/client`'s own `bun run generate` (from `packages/client`) regenerates its `src/generated`/`src/generated-effect`; `bun run check:generated` verifies they're up to date.

### Building a standalone binary

```bash
./packages/opencode/script/build.ts --single
./packages/opencode/dist/opencode-<platform>/bin/opencode   # e.g. darwin-arm64, linux-x64
```

## Architecture

### Package dependency direction

```
Schema  ->  Protocol  ->  Server
Schema, Protocol  ->  Client (never Core/Server)
Client, Core, Server  ->  sdk-next
```

- **`packages/schema`**: browser-safe wire/storage contracts (Effect `Schema.Struct` definitions), no service layers or side effects. Distinguishes unversioned *current* contracts (`Session`, `Permission`, ...) from explicitly `V1`-suffixed legacy contracts kept for compatibility/migration.
- **`packages/protocol`**: composes Schema values into HTTP paths, payloads, envelopes, errors, cursors, and streams (owns endpoint/group definitions under `groups/`, `middleware/`).
- **`packages/server`**: hosts Protocol's groups as the concrete `HttpApi`, owns protocol/domain adaptation. Neither Schema nor Protocol may transitively depend on databases, Drizzle, Session execution, providers, watchers, native modules, or WASM.
- **`packages/core`**: domain services — accounts, config, credentials, filesystem, git, models/catalog, plugins, integrations, database (Drizzle schema lives in `packages/core/src/**/*.sql.ts`; migrations applied by core).
- **`packages/opencode`**: the main business-logic package — CLI (`src/cli`), the TUI (`src/cli/cmd/tui`, SolidJS + [opentui](https://github.com/sst/opentui)), the HTTP server wiring (`src/server`), session runtime (`src/session`), tools, permissions, plugins, LSP, MCP, providers, sync, storage, etc. This is where most day-to-day feature work happens.
- **`packages/llm`**: standalone, session-agnostic Effect Schema-first LLM client. Provider-neutral `Message`/`Model`/`LLMRequest`/`LLMEvent` model plus a `Route` abstraction (`Protocol` + `Endpoint` + `Auth` + `Framing`) that lets many providers reuse one protocol implementation (e.g. DeepSeek/TogetherAI/Cerebras/Fireworks all reuse `OpenAIChat.protocol`). `packages/opencode/src/session/llm.ts` and its `src/session/llm/*` adapters are the integration point that decides AI-SDK vs. native route runtime — session concerns (auth, permissions, plugins, telemetry) stay out of `packages/llm`.
- **`packages/client`** (`@opencode-ai/client`): generated Promise and Effect network clients derived from Server's `HttpApi` via a runtime-neutral **SDK Contract IR**. Root export is zero-Effect (`fetch`-based Promise API); `/effect` export depends on Effect + Schema + Protocol only.
- **`packages/sdk-next`**: will become `@opencode-ai/sdk`. Composes Client + Core + Server into "Embedded OpenCode" — an in-process host that executes Server's `HttpRouter` in memory (no network listener) while sharing the same routing/middleware/codecs/handlers as networked mode.
- **`packages/app`**: shared web UI (SolidJS), consumed by both the web app and the Electron desktop shell.
- **`packages/desktop`**: Electron wrapper around `packages/app`. Renderer only talks to the main process via `window.api` (`src/preload`); IPC handlers register in `src/main/ipc.ts`.
- **`packages/tui`**, **`packages/ui`**, **`packages/session-ui`**: terminal UI and shared UI component packages.
- **`packages/plugin`**: source for `@opencode-ai/plugin`.
- **`packages/codemode`**: sandboxed, schema-described tool execution engine. Deliberately unaware of any hosting session/channel model; hosts own authorization and persistence.
- **`packages/effect-drizzle-sqlite`**: vendored, intentionally generic Drizzle+Effect+SQLite adapter (no opencode-specific tables/paths belong here).
- **`packages/stats`**, **`packages/console`**, **`packages/slack`**: separate products/surfaces (stats site, console app, Slack integration) built on the same core.

### Session runtime (V2)

`packages/opencode/src/session` implements the durable session/agent execution model described in detail in `CONTEXT.md` at the repo root — read it before making non-trivial changes to session execution, context assembly, or streaming. Key ideas:

- **System Context** is assembled from independently-registered **Context Sources** (via a Location-scoped **System Context Registry**) into a **Baseline System Context** at the start of a **Context Epoch**; later source changes are admitted lazily at a **Safe Provider-Turn Boundary** as durable **Mid-Conversation System Messages**, never pushed asynchronously.
- Prompts are durably **admitted** into a session inbox and **promoted** into visible `Session History` at safe boundaries; `SessionV2.prompt(...)` does durable admission + an advisory `SessionExecution.wake(sessionID)`, decoupled from actual model execution.
- `SessionExecution` is process-global and Session-ID based; a **Session Drain** is process-local, non-durable coordination (not a durable entity) that promotes input and runs provider turns until no continuation remains. Durable crash recovery must reason from prompts/history/tool state, not from an invented drain identity.
- Exactly one explicit `llm.stream(request)` call happens per provider turn; orchestration does not go through the legacy `SessionPrompt.loop(...)` or an in-memory tool loop.
- Compaction starts a fresh Context Epoch (new baseline, new snapshot); prior Mid-Conversation System Messages stay in durable audit history but leave projected model history.
- Oversized tool output is bounded before being written to session history, with the full text spilled to a temporary **Managed Tool Output File**; the bounded result — not the file — is the durable record.

### Effect conventions (see `packages/opencode/AGENTS.md` for the full list)

- Services are defined with `Context.Service`, run via `makeRuntime` (`src/effect/run-service.ts`); per-directory/per-project state uses `InstanceState` (`src/effect/instance-state.ts`, keyed by directory, one copy per open project).
- Modules use flat top-level exports plus a self-reexport at the bottom (`export * as Foo from "./foo"`), not `export namespace`. Multi-sibling directories (e.g. `src/session/`, `src/config/`) avoid barrel `index.ts` files — import the specific sibling module directly.
- `EffectBridge` is the sanctioned way to re-enter Effect services from native/external callbacks (watchers, node-pty, plugin callbacks) with instance/workspace context.

### Testing conventions

See `packages/opencode/test/AGENTS.md` for the full guide. Highlights:
- `tmpdir()` / `tmpdirScoped()` (`test/fixture/fixture.ts`) create auto-cleaned temp dirs (`opencode-test-*`), optionally git-initialized and/or config-seeded.
- `testEffect(...)` (`test/lib/effect.ts`) provides the Effect test runtime; use `it.effect` for tests needing `TestClock`/`TestConsole`, `it.live` for real time/fs/git/process behavior, `it.instance` for live tests needing a scoped temp instance.
- Never synchronize concurrent test fibers with `Effect.sleep`/`setTimeout` races — wait on a real readiness signal instead (`pollWithTimeout`, `awaitWithTimeout`, `llm.wait(n)`, `SessionStatus.Service.get`, `BackgroundJob.wait`, bus subscriptions, `Deferred`). Fixed sleeps are only OK for testing debounce/throttle itself, real mtime-resolution waits, or intentional latency simulation.
- `packages/llm` uses cassette-based recorded provider tests (`recordedTests(...)`); replay is the default, `RECORD=true` re-records against real providers behind required API-key env vars. Don't blanket re-record a whole file — delete/target the one cassette you're refreshing.

## Documentation map

- `AGENTS.md` (root) — style guide: branch names, commit/PR title conventions, general code style, Effect rules, module shape.
- `CONTEXT.md` (root) — glossary and design constraints for the Session V2 runtime and client/SDK contract architecture; treat term definitions here as authoritative vocabulary.
- `CONTRIBUTING.md` — contribution policy (issue-first PRs, PR title/scope conventions, debugger setup, vouch/denounce system).
- `specs/` — design specs and in-progress proposals (e.g. `specs/effect/migration.md` for the Effect migration pattern reference); treat these as design intent, not necessarily current shipped behavior.
