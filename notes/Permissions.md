# How opencode's permission system works

Permissions gate whether a tool call runs automatically, prompts the user first, or is
blocked outright. This note covers the rule shape, where rules can be configured, how
they're merged/evaluated, the built-in defaults, subagent permission derivation, and
whether an "ask" approval is ever remembered.

## Rule shape

A permission rule is `{ permission: string, pattern: string, action: "allow" | "ask" |
"deny" }` (`PermissionV1.Rule`). `permission` is a tool/capability name (see the known
keys below), `pattern` is a glob matched against the tool's argument (a file path for
`read`/`edit`, a command prefix for `bash`, an agent name for `task`, etc., `"*"` meaning
"any"), and `action` is the resulting decision. A `PermissionV1.Ruleset` is just an
ordered array of these rules.

## Known permission keys

From the config schema (`packages/core/src/v1/config/permission.ts:17-36`): `read`,
`edit`, `glob`, `grep`, `list`, `bash`, `task`, `external_directory`, `todowrite`,
`question`, `webfetch`, `websearch`, `lsp`, `doom_loop`, `skill`. This is not a closed
set — the schema is a `Schema.StructWithRest`, so any other tool name (including custom
MCP tool names) is accepted too. Each key's value is either a single `Action`
(`"ask"`/`"allow"`/`"deny"`, applied as pattern `"*"`) or an object mapping glob patterns
to actions, e.g.:

```jsonc
"permission": {
  "bash": "ask",                          // shorthand: same as { "*": "ask" }
  "edit": { "*": "deny", "src/**": "allow" },
  "external_directory": { "~/.ssh/*": "deny" }
}
```

`external_directory` patterns support `~/`, `~`, and `$HOME` expansion
(`packages/opencode/src/permission/index.ts:178-184`).

## Where permissions are configured

Two config layers, both accepting the same rule shape:

1. **Global**: the top-level `permission` key in `opencode.jsonc`
   (`cfg.permission` → `ConfigPermissionV1.Info`). Loaded once as
   `user = Permission.fromConfig(cfg.permission ?? {})` (`agent.ts:138`) and folded into
   every agent's ruleset.
2. **Per-agent**: `agent.<name>.permission` in config, merged on top of that specific
   agent's ruleset (`item.permission = Permission.merge(item.permission,
   Permission.fromConfig(value.permission ?? {}))`, `agent.ts:293`). See
   `notes/Agents.md` for the full per-agent field reference and
   `notes/BuiltInAgents.md` for each built-in agent's resulting ruleset.

There's also a third, non-config layer: a **session's own `permission` field**
(`SessionV1.Info.permission`), which is durable (DB-persisted) but not set from
`opencode.jsonc` directly — it's populated by things like CLI `--tools` overrides or,
for `task`-spawned subagent sessions, by `deriveSubagentSessionPermission` (see below).

## Merge order and evaluation

Construction order for an agent's base ruleset (`agent.ts:119-138` + per-built-in
overrides):

```
opencode's hardcoded baseline (`defaults`)
  → built-in agent-specific overrides (e.g. plan's edit denies)
  → global `permission` config (`user`)
  → agent.<name>.permission config (applied afterward, agent.ts:293)
```

At call time (`session/tools.ts:81-89`), the ruleset actually evaluated is
`Permission.merge(agent.permission, session.permission ?? [])` — the session's own
ruleset (if any) is appended after the agent's.

`Permission.merge(...)` (`permission/index.ts:200-202`) is literally just flattening the
arrays in argument order — there's no field-by-field diffing. All the precedence logic
lives in `evaluate()` (`permission/index.ts:28-38`):

```ts
export function evaluate(permission, pattern, ...rulesets) {
  return rulesets.flat().findLast((rule) =>
    Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern)
  ) ?? { action: "ask", permission, pattern: "*" }
}
```

- It scans the flattened, concatenated ruleset **from the end**, matching on wildcard
  patterns for both the permission name and the argument pattern — so **the last rule
  added that matches wins**, i.e. later/more-specific layers override earlier/more-general
  ones.
- If nothing matches at all, the implicit default is **`"ask"`** — not allow, not deny.

## Tool visibility vs. call-time denial

A full wildcard deny (`pattern: "*"`, `action: "deny"`) for a permission doesn't just
block calls — it removes the tool from what's offered to the model at all.
`Permission.disabled()` / `Permission.visibleTools()` (`permission/index.ts:204-219`)
filter the tool list handed to the LLM: any tool whose *last matching* `"*"` rule is
`"deny"` is excluded from `registry.tools(...)` entirely (`session/tools.ts:92-97`), so
the model never even sees it as callable. `edit`/`write`/`apply_patch` are treated as
one group under the `edit` permission for this check, and the three MCP resource-listing
tools are treated as `read`.

## Subagent permission derivation

When the `task` tool spawns a subagent session (`packages/opencode/src/tool/task.ts`),
that child session gets its own `permission` computed by
`deriveSubagentSessionPermission` (`packages/opencode/src/agent/subagent-permissions.ts`):

- It inherits the **parent session's `deny` rules and `external_directory` rules** (not
  its allows — a restrictive parent constrains its children, but a subagent's own
  ruleset otherwise governs its capabilities independently of the parent's allows).
- It adds a default `todowrite: deny` and `task: deny` (no further subagent spawning)
  **unless** the subagent's own `permission` already explicitly allows those.

## The `ask` flow — does opencode remember approvals?

When evaluation yields `"ask"`, the tool calls `ctx.ask({ permission, patterns, always,
metadata })`. The UI (TUI/web app) presents exactly three reply options
(`schema/v1/permission.ts`, TUI `routes/session/permission.tsx`, app
`context/permission.tsx`): **`once`**, **`always`**, **`reject`**.

- **`once`** — resolves just that single pending call. Nothing is recorded; the same
  call pattern will ask again next time.
- **`reject`** — fails the call (optionally with feedback), and also rejects any other
  pending requests in the same session. Nothing recorded.
- **`always`** — for each pattern in the tool's `always` list (e.g. `["*"]` for
  `task`/`write`/`edit`, or a computed bash command-prefix glob for `bash`), pushes a new
  `{ permission, pattern, action: "allow" }` rule into an **in-memory `approved` array**
  (`permission/index.ts:145-151`), then immediately re-evaluates and resolves any other
  still-pending requests that now match.

**Scope of `approved`**: it lives in `InstanceState`, keyed only by **project
directory** (`permission/index.ts:46-65`) — not by session ID. That means an "always"
approval is shared across *every* session running against that project in the current
opencode process, not just the session that triggered the prompt. It is held purely
in memory and is **never written back to `opencode.jsonc`** or any other file — a fresh
`opencode` process (or a restarted server) starts with an empty `approved` list and asks
again from scratch.

So: opencode supports "remember my decision" — but only as **process-lifetime, in-memory
memory shared across sessions for the current project**, not as a durable,
cross-restart, config-persisted decision. There is no UI option or code path that writes
an approved rule permanently into config.

## Summary table

| Layer | Configured via | Scope | Persisted? |
|---|---|---|---|
| Baseline `defaults` | hardcoded in `agent.ts` | all agents | n/a (code) |
| Global | `opencode.jsonc` → `permission` | all agents | yes (file) |
| Per-agent | `opencode.jsonc` → `agent.<name>.permission` | one agent | yes (file) |
| Session | DB `session.permission` (CLI `--tools`, subagent derivation) | one session (+ its subagent children, via inheritance of denies) | yes (DB) |
| "Always" approval | user's `ask` reply | whole project directory, current process only | **no** — in-memory only, lost on restart |
