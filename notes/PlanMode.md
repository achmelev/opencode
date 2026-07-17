# Plan mode

"Plan mode" is not a separate mode flag anywhere in opencode — it's shorthand for
"the built-in `plan` agent is the one handling the current turn." Everything described
below is keyed off the active agent's name being `"plan"`, either through that agent's
own declared config (see `notes/BuiltInAgents.md`) or through a handful of places in the
harness that hardcode the literal string `"plan"`.

## How plan mode gets activated

Switching to plan mode means making `plan` the active agent for the next turn. There's
no independent toggle — these are the ways to do that:

1. **Tab / Shift+Tab in the TUI** — bound to `agent.cycle` / `agent.cycle.reverse`
   (`packages/tui/src/config/keybind.ts:130-131`). This calls `local.agent.move(±1)`
   (`packages/tui/src/app.tsx:701/735`), which steps through:
   ```ts
   const agents = createMemo(() => sync.data.agent.filter((agent) => agent.mode !== "subagent" && !agent.hidden))
   ```
   (`packages/tui/src/context/local.tsx:78`) — i.e. only agents that are not
   `subagent`-mode and not `hidden`. `build` and `plan` both qualify (`mode: "primary"`,
   not hidden); `general`/`explore` are excluded (subagent-mode); `compaction`/`title`/
   `summary` are excluded (hidden).

2. **The `/agents` dialog** (`packages/tui/src/component/dialog-agent.tsx`) — lists the
   same filtered set and lets you pick `plan` directly by name (`local.agent.set("plan")`)
   instead of cycling.

3. **CLI `--agent` flag** — `packages/opencode/src/cli/cmd/run.ts:170-173`, e.g.
   `opencode run --agent plan "..."`, for starting a headless run directly in plan mode.

4. **`default_agent` config** (`opencode.jsonc`) — sets which agent a *new* session
   starts with; set it to `"plan"` to start every session in plan mode. See
   `notes/Agents.md` for its resolution rules.

There is also a TUI-only, currently-inert auto-switch path: `packages/tui/src/routes/
session/index.tsx:319-334` listens for a completed `plan_enter` tool call and calls
`local.agent.set("plan")` automatically (mirrored by `plan_exit` → `local.agent.set
("build")`). As of this investigation there is no registered `plan_enter` tool anywhere
in the tool registry, so today only the `plan_exit` half is reachable, and only when the
experimental flag below is enabled.

## What the `plan` agent's own config does (not harness-special-cased)

This part is just the normal agent-config machinery from `notes/Agents.md` doing its
job — no special harness code is needed for any of it:

- **Permission ruleset** (`packages/opencode/src/agent/agent.ts:156-181`): denies all
  edits except to two specific plan-markdown paths (`.opencode/plans/*.md` and the
  global data-dir `plans/*.md`), denies spawning the `general` subagent via `task`, adds
  an `external_directory` allow for the global plans directory, and allows `question`
  and `plan_exit`.
- **Description**: `"Plan mode. Disallows all edit tools."` — shown in the `/agents`
  dialog and elsewhere.
- Like `build`, `plan` sets no custom `model`/`temperature`/`topP`/`prompt` — it inherits
  the session's model and the default system prompt.

## What the harness does differently — beyond that config

These are places that check `agent.name === "plan"` (or `"build"`) directly, independent
of anything declared on the agent itself:

### 1. Per-turn synthetic reminders — `packages/opencode/src/session/reminders.ts`

`SessionReminders.apply` runs on every user message before it's sent to the model and
branches on the active agent's name. Behavior depends on the `experimentalPlanMode`
runtime flag:

**Default (flag off)** — `reminders.ts:26-48`:
- If the active agent is `plan`, append a synthetic text part containing
  `prompt/plan.txt` to the user message: a blunt "zero edits, read-only phase" reminder
  that does **not** mention any plan-file path.
- If the active agent just became `build` after a prior `plan` turn (detected by
  scanning message history for the last assistant message tagged `agent: "plan"`),
  append `prompt/build-switch.txt`: "Your operational mode has changed from plan to
  build... you are permitted to make file changes..."

**Experimental (`experimentalPlanMode` on)** — `reminders.ts:51-89`:
- On the plan → build transition, computes the plan file path via `Session.plan(session,
  ctx)` (`session/session.ts:331-335` — deterministic path from worktree/session
  timestamp/slug) and, if it exists, appends `build-switch.txt` plus
  `"A plan file exists at ${plan}. You should execute on the plan defined within it"`.
- While `plan` stays active, computes the same path each turn, checks whether the file
  exists (`fsys.existsSafe`), creates the parent directory if not, and appends
  `prompt/plan-mode.txt` with its `${planInfo}` placeholder filled in with either
  *"A plan file already exists at ‹path›. You can read it and make incremental edits..."*
  or *"No plan file exists yet. You should create your plan at ‹path› using the write
  tool."* `plan-mode.txt` also carries a detailed multi-phase planning workflow (explore
  → design → review → write final plan → call `plan_exit`) that isn't part of the
  agent's base prompt.

This is the main harness-level special-casing: nothing in `plan`'s `Agent.Info` entry
says "inject a reminder" — the harness notices the agent's name on every turn and reacts,
and in the experimental path it actively computes and injects the live plan-file path
that the permission carve-out was written to allow editing.

### 2. TUI auto-switching — `packages/tui/src/routes/session/index.tsx:319-334`

The TUI subscribes to `message.part.updated` events and, purely as client-side UI state
(not server/session state), flips its local agent selector whenever a `plan_exit` (or,
if ever wired, `plan_enter`) tool call completes:
```ts
if (part.tool === "plan_exit") {
  local.agent.set("build")
} else if (part.tool === "plan_enter") {
  local.agent.set("plan")
}
```
This only affects what agent the TUI offers for the *next* message — it's a convenience
so the user doesn't have to manually Tab back to `build` after approving a plan.

### 3. Tool registration gating — `packages/opencode/src/tool/registry.ts:243`

The `plan_exit` tool (`packages/opencode/src/tool/plan.ts`, `PlanExitTool` — asks the
user "plan is complete, switch to build?" and, if yes, injects a synthetic user message
tagged `agent: "build"` to hand the session off) is only registered when
`flags.experimentalPlanMode && flags.client === "cli"`. Outside that flag+client
combination, the tool doesn't exist, so `plan` mode has no explicit "I'm done, let's
build" tool call available — the user simply has to switch back to `build` manually
(Tab, `/agents`, etc.).

## Summary

| What | Where it lives | Special-cased on `"plan"`? |
|---|---|---|
| Blocks edits except to plan-markdown paths | `agent.ts` permission ruleset | No — ordinary agent config |
| Denies `general` subagent spawning | `agent.ts` permission ruleset | No — ordinary agent config |
| Per-turn "you're read-only" / "plan file is at X" reminder | `session/reminders.ts` | **Yes** — literal `agent.name === "plan"` check |
| "Mode changed, you can edit now" reminder on plan→build | `session/reminders.ts` | **Yes** — scans history for `agent: "plan"` |
| Auto-switch TUI's agent selector on `plan_exit`/`plan_enter` | `tui/routes/session/index.tsx` | **Yes** — literal tool-name check, UI-only |
| `plan_exit` tool exists at all | `tool/registry.ts` | **Yes** — gated by `experimentalPlanMode` + CLI client |
