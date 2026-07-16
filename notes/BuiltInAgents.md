# Built-in agents — default parameter values

opencode ships seven native agents, all constructed directly in code
(`packages/opencode/src/agent/agent.ts:140-265`, all `native: true`) *before* any user
`agent.<name>` config is merged on top (see the merge rules in `notes/Agents.md`). This
file lists each built-in's as-shipped defaults for every `Agent.Info` field
(`agent.ts:35-56`): `model`, `variant`, `temperature`, `topP`, `prompt`, `permission`,
`mode`, `description`, `hidden`, `color`, `steps`, `options`. (`disable` isn't a runtime
field — it only exists in config, as an instruction to delete an agent.)

## Shared baseline permission ruleset (`defaults`)

Every built-in's `permission` is `Permission.merge(defaults, <agent-specific overrides>,
user)`, where `defaults` (`agent.ts:119-136`) is:

```
"*": "allow"
doom_loop: "ask"
external_directory: { "*": "ask", <skill/tmp/reference dirs>: "allow" }
question: "deny"
plan_enter: "deny"
plan_exit: "deny"
read: { "*": "allow", "*.env": "ask", "*.env.*": "ask", "*.env.example": "allow" }
```

`user` is the user's global `permission` config (`config.permission`), layered on top of
everything. Below, each agent's permission is described as "`defaults` + these
overrides" rather than repeating the full ruleset.

## `build`

| Field | Default |
|---|---|
| `model` | *unset* — falls back to the global default model (`provider.defaultModel()`) |
| `variant` | *unset* |
| `temperature` | *unset* → per-model heuristic default (`ProviderTransform.temperature`, e.g. `0.55` for Qwen, `undefined`/provider-default for Claude) |
| `top_p` | *unset* → per-model heuristic default (`ProviderTransform.topP`) |
| `prompt` | *unset* — uses the fully assembled system prompt with no override |
| `mode` | `"primary"` |
| `hidden` | *unset* (`false`) |
| `color` | *unset* |
| `steps` | *unset* (no cap) |
| `options` | `{}` |
| `description` | `"The default agent. Executes tools based on configured permissions."` |
| `permission` | `defaults` + `question: "allow"`, `plan_enter: "allow"` |

## `plan`

| Field | Default |
|---|---|
| `model` | *unset* — falls back to the global default model |
| `variant` | *unset* |
| `temperature` | *unset* → per-model heuristic default |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | *unset* |
| `mode` | `"primary"` |
| `hidden` | *unset* (`false`) |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | `"Plan mode. Disallows all edit tools."` |
| `permission` | `defaults` + `question: "allow"`, `plan_exit: "allow"`, `task: { general: "deny" }`, `external_directory: { <Global data>/plans/*: "allow" }`, `edit: { "*": "deny", .opencode/plans/*.md: "allow", <data-dir-relative>/plans/*.md: "allow" }` |

## `general`

| Field | Default |
|---|---|
| `model` | *unset* — inherits the parent turn's model when spawned via `task` (`task.ts:167-170`) |
| `variant` | *unset* — inherits the parent turn's variant |
| `temperature` | *unset* → per-model heuristic default |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | *unset* |
| `mode` | `"subagent"` |
| `hidden` | *unset* (`false`) — appears in `@` autocomplete |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | `"General-purpose agent for researching complex questions and executing multi-step tasks. Use this agent to execute multiple units of work in parallel."` |
| `permission` | `defaults` + `todowrite: "deny"` |

## `explore`

| Field | Default |
|---|---|
| `model` | *unset* — inherits the parent turn's model when spawned via `task` |
| `variant` | *unset* — inherits the parent turn's variant |
| `temperature` | *unset* → per-model heuristic default |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | `PROMPT_EXPLORE` (`agent/prompt/explore.txt`) — the only built-in that ships its own system prompt |
| `mode` | `"subagent"` |
| `hidden` | *unset* (`false`) |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | `"Fast agent specialized for exploring codebases. Use this when you need to quickly find files by patterns (eg. \"src/components/**/*.tsx\"), search code for keywords (eg. \"API endpoints\"), or answer questions about the codebase (eg. \"how do API endpoints work?\"). When calling this agent, specify the desired thoroughness level: \"quick\" for basic searches, \"medium\" for moderate exploration, or \"very thorough\" for comprehensive analysis across multiple locations and naming conventions."` |
| `permission` | `defaults` + `"*": "deny"`, then explicit `allow` for `grep`, `glob`, `list`, `bash`, `webfetch`, `websearch`, `read`; `external_directory` allowed read-only for whitelisted dirs (skills/tmp/references), `ask` elsewhere |

## `compaction` *(hidden, internal)*

| Field | Default |
|---|---|
| `model` | *unset* — falls back to the current session's model (`compaction.ts:329-331`) |
| `variant` | *unset* |
| `temperature` | *unset* → per-model heuristic default |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | `PROMPT_COMPACTION` (`agent/prompt/compaction.txt`) |
| `mode` | `"primary"` |
| `hidden` | `true` |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | *unset* |
| `permission` | `defaults` + `"*": "deny"` (all tools denied) |

## `title` *(hidden, internal)*

| Field | Default |
|---|---|
| `model` | *unset* — falls back to the provider's small model if defined, else the current session's model (`prompt.ts:216-221`, `provider.getSmallModel`) |
| `variant` | *unset* |
| `temperature` | `0.5` — the only built-in with an explicit fixed temperature |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | `PROMPT_TITLE` (`agent/prompt/title.txt`) |
| `mode` | `"primary"` |
| `hidden` | `true` |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | *unset* |
| `permission` | `defaults` + `"*": "deny"` |

## `summary` *(hidden, internal)*

| Field | Default |
|---|---|
| `model` | *unset* — falls back to the current session's model |
| `variant` | *unset* |
| `temperature` | *unset* → per-model heuristic default |
| `top_p` | *unset* → per-model heuristic default |
| `prompt` | `PROMPT_SUMMARY` (`agent/prompt/summary.txt`) |
| `mode` | `"primary"` |
| `hidden` | `true` |
| `color` | *unset* |
| `steps` | *unset* |
| `options` | `{}` |
| `description` | *unset* |
| `permission` | `defaults` + `"*": "deny"` |

## Notes on "unset" fields

- **`model`/`variant` unset** doesn't mean "no model" — it means the agent has no
  *opinion* of its own, so the caller's context decides: the global default model for a
  freshly started primary session, the parent turn's model/variant for a `task`-spawned
  subagent, or a dedicated small/current-session model for the internal title/
  compaction/summary agents.
- **`temperature`/`top_p` unset** doesn't mean "provider default" in the SDK sense
  either — it falls through to `ProviderTransform.temperature`/`topP`
  (`packages/opencode/src/provider/transform.ts:481-507`), which hardcodes per-model-ID
  heuristics (e.g. Qwen models get `temperature: 0.55`, `top_p: 1`; Claude models get
  neither set at all, i.e. the true provider default applies).
- **`options: {}` for every built-in** — none of them ship provider-specific passthrough
  options out of the box; this is purely a user-config extension point.
