# `doom_loop` and `external_directory` — permission keys that aren't tools

`notes/Permissions.md` lists `doom_loop` and `external_directory` among the known
permission keys (`packages/core/src/v1/config/permission.ts:17-36`), but neither appears
in `notes/BuiltInTools.md`. That's because they aren't callable tools at all — the model
never invokes them by name, and they have no parameter schema sent over the wire. They're
permission-only checks that real tools (`read`, `write`, `edit`, `bash`, `glob`, `grep`,
...) trigger internally via `ctx.ask(...)`, using these names as the `permission` field
instead of the calling tool's own name.

## `external_directory`

Checked whenever a tool is about to touch a path outside the current project worktree —
`assertExternalDirectoryEffect` (`packages/opencode/src/tool/external-directory.ts:15-45`):

```ts
export const assertExternalDirectoryEffect = Effect.fn(...)(function* (ctx, target, options) {
  if (!target) return false
  if (options?.bypass) return false
  if (containsPath(full, ins)) return false   // inside worktree: no check needed

  const dir = kind === "directory" ? full : path.dirname(full)
  const glob = path.join(dir, "*")

  yield* ctx.ask({
    permission: "external_directory",
    patterns: [glob],
    always: [glob],
    metadata: { filepath: full, parentDir: dir },
  })
  return true
})
```

- Tools that touch the filesystem or run external processes (`read`, `write`, `edit`,
  `glob`, `grep`, `bash`/shell) call this helper with their target path before acting.
  If the path is inside the worktree, it's a no-op (`containsPath` short-circuits). If
  it's outside, the check asks under the `external_directory` permission with a
  directory-glob pattern (e.g. `~/.ssh/*` or `/tmp/foo/*`) — **not** under the calling
  tool's own permission name.
- This is exactly the mechanism behind the whitelisted directories seen in
  `notes/Permissions.md`/`notes/BuiltInAgents.md`: opencode's baseline `defaults`
  ruleset pre-allows `external_directory` for the temp dir, skill dirs, and reference
  dirs (`agent.ts:108-117`), and `explore`'s `readonlyExternalDirectory` override does
  the same read-only.
- Configurable directly in `opencode.jsonc`: `"permission": { "external_directory": {
  "/some/path/*": "deny" } }`.

## `doom_loop`

A repeated-tool-call guard, checked after every tool result in
`packages/opencode/src/session/processor.ts:340-380`:

```ts
const recentParts = parts.slice(-DOOM_LOOP_THRESHOLD)
if (
  recentParts.length !== DOOM_LOOP_THRESHOLD ||
  !recentParts.every((part) =>
    part.type === "tool" && part.tool === value.name &&
    part.state.status !== "pending" &&
    JSON.stringify(part.state.input) === JSON.stringify(input)
  )
) return

yield* permission.ask({
  permission: "doom_loop",
  patterns: [value.name],
  always: [value.name],
  metadata: { tool: value.name, input },
  ruleset: agent.permission,
})
```

- If the model calls the **same tool with byte-identical input** `DOOM_LOOP_THRESHOLD`
  times in a row, opencode pauses and asks for confirmation before letting it repeat
  again — a safety net against the model getting stuck retrying an identical failing
  call in a loop.
- opencode's baseline `defaults` ruleset sets `doom_loop: "ask"` (`agent.ts:121`), so
  this is on by default for every agent unless overridden.
- Configurable in `opencode.jsonc`: `"permission": { "doom_loop": "allow" }` to disable
  the guard entirely (never ask, just let it loop), or `"deny"` to hard-block instead of
  prompting.

## Why they're absent from `notes/BuiltInTools.md`

`notes/BuiltInTools.md` catalogs the tool registry (`packages/opencode/src/tool/
registry.ts`) — things the model can name in a tool call, each with a JSON Schema
derived from an `Effect.Schema.Struct`. `doom_loop` and `external_directory` are
permission *categories* only: they have a name/pattern/action shape like any other
permission rule and can be configured the same way, but they're never registered as
tools, never get a JSON Schema, and the model never calls them directly — they're
side-effects of calling other tools.
