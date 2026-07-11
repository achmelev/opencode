# How opencode assembles the system prompt

Assembly happens in two stages, once per assistant turn:

1. **`packages/opencode/src/session/prompt.ts`** (the turn orchestrator) gathers several
   *content pieces* concurrently and hands them down as a plain `system: string[]` array.
2. **`packages/opencode/src/session/llm/request.ts`** (`LLMRequestPrep.prepare`) combines
   that array with the agent's base persona prompt and a couple of other inputs into the
   final text, gives plugins one chance to rewrite it, and turns it into the actual
   request sent to the model.

## 1. Content pieces gathered per turn

`prompt.ts:1257-1271` runs these concurrently via `Effect.all` for every assistant step:

```ts
const [skills, env, instructions, mcpInstructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),
  sys.environment(model),
  instruction.system().pipe(Effect.orDie),
  sys.mcp(agent, session.permission),
  MessageV2.toModelMessagesEffect(msgs, model),
])
const system = [
  ...env,
  ...instructions,
  ...(mcpInstructions ? [mcpInstructions] : []),
  ...(skills ? [skills] : []),
]
if (format.type === "json_schema") system.push(STRUCTURED_OUTPUT_SYSTEM_PROMPT)
```

- **`sys.environment(model)`** — `packages/opencode/src/session/system.ts:60-96`. Produces:
  - A one-line model identity statement (`You are powered by the model named <id>...`).
  - An `<env>` block: working directory, workspace/worktree root, whether it's a git repo,
    `process.platform`, and today's date.
  - If the project config defines named `references`/`reference` directories that have a
    `description`, an `<available_references>` block listing each `<name>`/`<path>`/`<description>`.

- **`instruction.system()`** — `packages/opencode/src/session/instruction.ts:155-169`, backed by
  `systemPaths()` (`instruction.ts:110-153`). Reads project/global "memory" files and any
  configured extra instructions, each wrapped as `Instructions from: <path>\n<content>`:
  - **Global**: the first of `~/.config/opencode/AGENTS.md`, then (unless
    `flags.disableClaudeCodePrompt`, controlled by the `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT`-style
    config flag) `~/.claude/CLAUDE.md`, that exists.
  - **Project**: walking up from the current directory to the worktree root, the first
    match among `AGENTS.md`, `CLAUDE.md` (unless disabled), and the deprecated `CONTEXT.md`
    wins — only one project-level file is loaded, not one per ancestor directory. Skipped
    entirely when `OPENCODE_DISABLE_PROJECT_CONFIG` is set.
  - **Config-declared extras**: every entry in the config's `instructions` array — glob
    patterns/absolute paths resolved relative to the project (or global config dir if
    project config is disabled), plus `https://`/`http://` URLs fetched directly
    (5s timeout, best-effort).
  - A related but distinct mechanism: `Instruction.resolve()` (`instruction.ts:179-221`)
    retroactively attaches nearby `AGENTS.md`/`CLAUDE.md` files the first time the model
    reads a file under a previously-unvisited directory. That's injected as a synthetic
    message part on the read-tool result, not into this `system` block, and is tracked
    per-message so the same file isn't attached twice (`Instruction.clear` resets the claim
    set when a turn finishes).

- **`sys.mcp(agent, permission)`** — `system.ts:112-128`. For every configured MCP server
  that exposes at least one tool the current agent/session permission set doesn't fully
  deny, wraps that server's self-declared `instructions` text into:
  ```
  <mcp_instructions>
    <server name="...">
      ...
    </server>
  </mcp_instructions>
  ```

- **`sys.skills(agent)`** — `system.ts:98-110`. Unless the `skill` tool is permission-denied
  for the agent, emits a short blurb plus a verbose listing (`Skill.fmt(list, { verbose: true
  })`) of every skill available to that agent, instructing the model to use the `skill` tool
  to load one when its description matches the task.

- **`STRUCTURED_OUTPUT_SYSTEM_PROMPT`** — `prompt.ts:82`. Appended only when the caller's
  request set `format.type === "json_schema"`; tells the model it must call the
  `StructuredOutput` tool instead of replying in plain text.

## 2. Final assembly — `LLMRequestPrep.prepare` (`session/llm/request.ts:56-99`)

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ].filter((x) => x).join("\n"),
]

const header = system[0]
yield* input.plugin.trigger("experimental.chat.system.transform", { sessionID, model }, { system })
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

Three inputs are concatenated with `\n` (falsy entries dropped) into one string:

1. **Persona prompt** — either the active agent's own `prompt` field, or, if unset, a
   built-in prompt chosen by `SystemPrompt.provider(model)` (`session/system.ts:27-42`)
   sniffing `model.api.id`: Meta/"muse-spark" → `meta.txt`; `gpt-4`/`o1`/`o3` →
   `beast.txt`; `gpt`+`codex` → `codex.txt`; other `gpt` → `gpt.txt`; `gemini-` →
   `gemini.txt`; `claude` → `anthropic.txt`; `trinity` → `trinity.txt`; `kimi` →
   `kimi.txt`; otherwise `default.txt` (the base "you are opencode, an interactive CLI
   tool..." persona with tone/style/verbosity rules). The built-in `build`, `plan`, and
   `general` agents don't set `prompt`, so they get whichever of these matches the
   selected model; the built-in `explore`, `compaction`, `title`, and `summary` agents
   (`packages/opencode/src/agent/agent.ts:196-264`) each hardcode their own dedicated
   `prompt` file instead, and a user's `agent.<name>.prompt` config value overrides
   whatever that agent would otherwise use (`agent.ts:283`).
2. **The `input.system` array** from stage 1 above (env, instructions, mcp, skills,
   structured-output note).
3. **`input.user.system`** — an optional one-off string attached to a single prompt turn
   via the `sessions.prompt(...)` API's `system` field (`PromptInput.system`,
   `session/prompt.ts:1510`), persisted on that turn's `SessionV1.User.system`
   (`packages/schema/src/v1/session.ts:352`). Lets a caller inject an extra instruction
   for just one turn without touching agent/instruction-file config.

The joined string becomes `system[0]`. The **`experimental.chat.system.transform`** plugin
hook then fires with `{ sessionID, model }` and a mutable `{ system }` box — a plugin may
edit `system[0]` in place or push more entries onto the array. If more than one entry
survives, everything after index 0 is re-joined into a single index 1, so at most two
`system` strings ever leave `prepare()`.

## 3. Turning `system` into an actual request

- **Normal case** (`request.ts:99-115`): each string in `system` becomes a
  `{ role: "system", content: ... }` `ModelMessage` prepended to the conversation history;
  the combined array is `prepared.messages`. opencode never uses the AI SDK's `system:`
  shorthand parameter — always explicit system-role messages.
- **OpenAI OAuth exception** (ChatGPT-subscription auth) and internal GitLab "workflow"
  models: `system.join("\n")` instead becomes `options.instructions` (the Responses API's
  separate instructions channel, `request.ts:99`) or `workflowModel.systemPrompt`
  (`session/llm.ts:126`), and no system-role messages are prepended to `messages`.
- **Dispatch** (`session/llm.ts:271-340`): `prepared.messages` (system included) is passed
  to `streamText({ ..., messages: prepared.messages, model: wrapLanguageModel({ model:
  language, middleware: [...] }) })`.
- **Final per-provider pass**: a `transformParams` middleware on the wrapped model
  (`llm.ts:325-339`) calls `ProviderTransform.message(...)` (`provider/transform.ts:430+`)
  right before the request is serialized. Among other normalization, this is where
  **prompt-cache breakpoints** get attached: `applyCaching` (`transform.ts:323-372`) tags
  up to the first 2 system messages and the last 2 non-system messages with a
  provider-specific cache marker (`cacheControl` for Anthropic/OpenRouter/Alibaba,
  `cachePoint` for Bedrock, `cache_control` for `@ai-sdk/openai-compatible`,
  `copilot_cache_control` for GitHub Copilot) — so the system block is typically what
  anchors the cache prefix for providers that support prompt caching.

## Related but distinct: plan-mode / build-switch reminders

`packages/opencode/src/session/reminders.ts` injects plan-mode and build/plan-switch
guidance (`prompt/plan.txt`, `prompt/plan-mode.txt`, `prompt/build-switch.txt`) — but these
are appended as **synthetic text parts on the latest user message**, not into the `system`
array. They're conversation history, not the system block, even though they read like
system-level instructions.

## Config/extension points that shape the system prompt

| What | Where |
|---|---|
| Replace an agent's whole persona block | `agent.<name>.prompt` config |
| Add extra "memory" files or URLs | `instructions` config array |
| Add the `<available_references>` block | `references` (or deprecated `reference`) config |
| Add per-server `<mcp_instructions>` | each MCP server's own declared `instructions` |
| Programmatic rewrite of the final text | `experimental.chat.system.transform` plugin hook |
| One-off per-turn addition | `system` field on a `sessions.prompt(...)` call |
| Disable project `AGENTS.md`/`CLAUDE.md` discovery | `OPENCODE_DISABLE_PROJECT_CONFIG` |
| Disable `~/.claude/CLAUDE.md` global fallback | `flags.disableClaudeCodePrompt` (`session/instruction.ts:62,66`) |
