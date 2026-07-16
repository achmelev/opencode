# The opencode agent system

An **agent** is a named bundle of behavior config — model, system prompt, sampling
params, tool permissions, and a `mode` — that a session can run as. Multiple agents can
be configured for different purposes (a full-access coding agent, a read-only planning
agent, a specialized research subagent, ...), and both users and opencode itself define
them.

## Where agents are defined

- **Runtime shape**: `Agent.Info` (`packages/opencode/src/agent/agent.ts:35-56`) — the
  resolved, in-memory representation every agent (built-in or user-defined) is normalized
  into.
- **Config shape**: `ConfigAgentV1.Info` (`packages/core/src/v1/config/agent.ts:12-41`) —
  what a user writes under `agent.<name>` in `opencode.jsonc`.
- **Built-in agents** are constructed directly in code (`agent.ts:140-265`) and then
  user config entries are merged/overlaid on top, keyed by name (`agent.ts:267-294`).
  A user config entry can extend a built-in agent of the same name, or define a brand
  new one (which starts from `mode: "all"` and inherited default permissions).
- Custom agents can also be added via `.opencode/agent/*.md` files, or generated with
  `opencode agent create` (`packages/opencode/src/cli/cmd/agent.ts`), which asks the LLM
  itself to draft a `description`/system prompt for a new agent from a natural-language
  request (`Agent.Service.generate`, `agent.ts:368-436`).

## Per-agent configuration fields

From `ConfigAgentV1.Info` (`packages/core/src/v1/config/agent.ts:12-40`):

| Field | Purpose |
|---|---|
| `model` | `provider/model` ID this agent uses |
| `variant` | default model **variant** (reasoning-effort style variant), applies only when the agent uses its own configured model |
| `temperature`, `top_p` | sampling parameters for this agent's requests |
| `prompt` | system prompt override (replaces/extends the assembled system prompt — see `notes/SystemPrompt.md`) |
| `permission` | per-tool ruleset: `allow` / `ask` / `deny`, keyed by tool name and glob pattern (`PermissionV1.Ruleset`) |
| `tools` | *(deprecated)* simple boolean per-tool enable/disable; normalized into `permission` at load time (`agent.ts:69-76` in the config package) — `write`/`edit`/`patch: false` collapses to `permission.edit: "deny"` |
| `mode` | `"primary"`, `"subagent"`, or `"all"` — see below |
| `description` | human-readable "when to use this agent" text; shown in the `@` autocomplete and is what the `task` tool/LLM reads to pick a `subagent_type` |
| `hidden` | hide this agent from the `@` autocomplete (only meaningful for `mode: "subagent"`) — it remains callable by exact name via `task` |
| `disable` | remove this agent entirely (works even for built-ins — deletes the key from the resolved agent map, `agent.ts:268-271`) |
| `color` | hex code (`#RRGGBB`) or theme color name (`primary`/`secondary`/`accent`/`success`/`warning`/`error`/`info`) — purely cosmetic, used by the TUI to tint the agent's UI chrome |
| `steps` | max agentic iterations (tool-call rounds) before the agent is forced into a text-only response |
| `maxSteps` | *(deprecated)* alias for `steps` |
| `options` | provider-specific passthrough options merged into this agent's requests (same mechanism as `notes/ModelOptions.md`, but scoped to the agent rather than the model) |

Any additional/unrecognized top-level key is folded into `options` as a passthrough
(`normalize()`, `packages/core/src/v1/config/agent.ts:62-81`) — the config schema is a
`Schema.StructWithRest`, so arbitrary extra keys are tolerated rather than rejected.

So beyond model/prompt/temperature/permissions/mode, an agent can also carry: `variant`,
`description` (functionally important, not just cosmetic — see below), `hidden`,
`disable`, `color`, `steps`/`maxSteps`, and free-form `options`.

## `mode` — the three roles an agent can play

Defined in `packages/opencode/src/agent/agent.ts:38` / config `agent.ts:26`, defaulting
to `"all"` when a user config entry omits it (`agent.ts:276`):

- **`primary`** — a main conversational agent the user talks to directly. Selectable as
  the active agent for a session, and cyclable mid-session via the `agent.cycle`
  keybind (`packages/tui/src/component/prompt/index.tsx:165`, default Tab). Built-ins:
  `build` (default, full tool permissions) and `plan` (edit tools denied, meant for
  drafting a plan before switching to `build`).
- **`subagent`** — only invocable through the `task` tool (or an `@name` mention that
  resolves to it), never selectable as a session's main agent. Built-ins: `general`
  (parallel multi-step research) and `explore` (fast read-only codebase search).
- **`all`** — can serve either role; this is the default for a user-defined agent that
  doesn't set `mode` explicitly.

There are also three **hidden internal `primary` agents** — `compaction`, `title`,
`summary` (`agent.ts:219-264`) — used for opencode's own internal LLM calls (session
compaction, auto-titling, summarization). They're marked `hidden: true`, have all tools
denied, and are not meant to be selected or invoked by users or the main model at all.

## Selecting and invoking agents

- **Default primary agent**: the `default_agent` config key
  (`packages/core/src/v1/config/config.ts`) names the primary agent used when a session
  doesn't specify one. Resolution (`Agent.Service.defaultInfo`, `agent.ts:328-340`):
  the named agent must exist, must not be `mode: "subagent"`, and must not be `hidden`;
  otherwise it throws. If `default_agent` is unset, it falls back to the first
  non-subagent, non-hidden agent in definition order (typically `build`).
- **Switching the primary agent mid-session**: the `agent.cycle` TUI keybind (Tab by
  default) cycles through the configured primary agents
  (`packages/tui/src/component/prompt/index.tsx`). The CLI/API can also set a session's
  `agent` field directly.
- **Invoking a subagent**: the `task` tool's `subagent_type` parameter names the agent
  to spawn (`packages/opencode/src/tool/task.ts:46,116-119`). The calling model chooses
  `subagent_type` based on each agent's `description` — this is why `description` is
  functionally load-bearing, not just documentation. `task` spawns a **child session**
  (`sessions.create(...)`, `task.ts:142-158`) whose permissions are derived from both
  the parent session's deny rules and the subagent's own ruleset
  (`deriveSubagentSessionPermission`, `packages/opencode/src/agent/subagent-permissions.ts`):
  the child inherits the parent's `deny`/`external_directory` rules, and gets `todowrite`
  and `task` (i.e., no further subagent spawning) denied by default unless the subagent's
  own `permission` explicitly allows them.
- A subagent can also run **in the background** (`background: true` on the `task` call,
  gated by `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`), in which case the parent
  continues immediately and is notified asynchronously when the child session finishes
  (`task.ts:242-296`).

## Permissions in detail

`permission` is a ruleset of `{ permission, pattern, action }` entries (`allow`/`ask`/
`deny`), keyed by tool name (or `external_directory` for filesystem-outside-worktree
access) with glob patterns for scoping. Agent-level rulesets are layered:
`defaults` (opencode's baseline: broad allow, `.env` reads ask, plan/question tools
deny) → built-in agent overrides → user's global `permission` config → the specific
agent's own `permission`/`tools` config (`agent.ts:119-138, 267-293`). `Permission.merge`
composes these in order, so more specific layers win.

## Customizing built-in agents via config

A user config entry under `agent.<name>` doesn't have to define a new agent — if
`<name>` matches a built-in (`build`, `plan`, `general`, `explore`, `compaction`,
`title`, `summary`), it customizes that built-in in place instead. For each field,
opencode looks up the existing agent by name and, if found, overrides only the fields
present in your config, falling back to the built-in's original value otherwise
(`agent.ts:272-293`):

```ts
let item = agents[key]
if (!item) item = agents[key] = { name: key, mode: "all", permission: Permission.merge(defaults, user), options: {}, native: false }
if (value.model) item.model = Provider.parseModel(value.model)
item.variant = value.variant ?? item.variant
item.prompt = value.prompt ?? item.prompt
item.description = value.description ?? item.description
item.temperature = value.temperature ?? item.temperature
item.topP = value.top_p ?? item.topP
item.mode = value.mode ?? item.mode
...
```

- Every field except `permission` and `options` is a simple `value.field ?? item.field`
  replace-if-set — you only need to declare the fields you want to change; everything
  else (including the built-in's native behavior/description) is preserved.
- **`permission` is merged, not replaced**: `Permission.merge(item.permission,
  Permission.fromConfig(value.permission ?? {}))` layers your rules on top of the
  built-in's existing ruleset rather than discarding it.
- **`options` is deep-merged** (`mergeDeep`), same reasoning.
- **`disable: true` is the one exception that doesn't merge** — it deletes the agent
  entirely from the resolved map (`agent.ts:268-271`), built-in or not.
- Example: `{"agent": {"build": {"temperature": 0.1}}}` keeps `build`'s full default
  permissions and native behavior, but every request it makes now uses
  `temperature: 0.1`.

## Summary table — built-in agents

| Name | Mode | Hidden | Purpose |
|---|---|---|---|
| `build` | primary | no | Default agent; full permissions per config |
| `plan` | primary | no | Planning without edit access; edit tools denied except under `.opencode/plans/*.md` |
| `general` | subagent | no | Parallel multi-step research/work delegated via `task` |
| `explore` | subagent | no | Fast read-only codebase search delegated via `task` |
| `compaction` | primary | yes | Internal: summarizes/trims history at context overflow |
| `title` | primary | yes | Internal: generates session titles |
| `summary` | primary | yes | Internal: generates session summaries |
