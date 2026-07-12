# System messages: assembly, count, and position

Everything covered on how opencode builds and sends the system message(s) to the LLM.
For the full breakdown of *what content* goes into it (persona prompt, env block,
instruction files, MCP instructions, skills, config/plugin extension points), see
`notes/SystemPrompt.md` — this note focuses on how many system messages actually get sent
and where they sit in the request.

## Assembly, in brief

`session/llm/request.ts:56-67` (`LLMRequestPrep.prepare`) joins everything into **one**
string per turn:

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ].filter((x) => x).join("\n"),
]
```

- The agent's persona prompt (either a configured `agent.prompt`, or a model-family-sniffed
  built-in like `default.txt`/`anthropic.txt`).
- `input.system[]` — env info, discovered `AGENTS.md`/`CLAUDE.md` instructions, MCP server
  instructions, skills listing — assembled once per turn in `session/prompt.ts:1257-1271`.
- `input.user.system` — an optional one-off string attached to a single prompt via the
  `sessions.prompt(...)` API's `system` field, persisted on that turn's
  `SessionV1.User.system`.

This is never persisted anywhere (no `"system"` part type exists in the database) — it's
rebuilt fresh on every provider turn. See `notes/SessionRecordingExport.md` for how that
interacts with session exports (short answer: neither export path includes it, except the
`user.system` per-turn override, which *is* persisted and does appear).

## Is it always exactly one message?

Usually, yes — but the array can grow via a plugin hook.

`request.ts:69-78`:

```ts
const header = system[0]
yield* input.plugin.trigger("experimental.chat.system.transform", { sessionID, model }, { system })
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

- **No plugin involvement** (the default, out-of-the-box behavior): `system` stays a
  one-element array → exactly **one** system message. No first-party opencode plugin
  currently uses this hook to append entries — it exists purely as a third-party/user
  plugin extension point.
- **A plugin appends exactly one extra entry, leaves `system[0]` untouched**: `length ===
  2`, the `> 2` check is false, nothing collapses → **two** system messages.
- **A plugin appends two or more extra entries, leaves `system[0]` untouched**: the collapse
  condition fires — everything after index 0 is rejoined into a single second string →
  normalized back down to **two**.
- **A plugin rewrites `system[0]` itself** (not just appends): `system[0] === header` is
  false, so the collapse is skipped regardless of length → if that plugin also appended
  several entries, they all survive uncollapsed → **more than two** is possible.

So the practical ceiling in normal use is two, and the collapsing logic exists specifically
to enforce that — unless a plugin also edits the header text itself, which disables the
safety net. This ceiling shows up elsewhere too: the prompt-cache pass (`applyCaching`,
`provider/transform.ts:323-372`) only tags the **first two** system messages with a
cache-control breakpoint (`msgs.filter(msg => msg.role === "system").slice(0, 2)`).

## Is it always first in the messages array?

Yes, in the standard path — with two exceptions where system content bypasses the
`messages` array entirely.

`request.ts:99-115`:

```ts
const messages =
  isOpenaiOauth || input.isWorkflow
    ? input.messages
    : [
        ...system.map((x): ModelMessage => ({ role: "system", content: x })),
        ...input.messages,
      ]
```

- **Normal case**: every surviving string in `system[]` is mapped to its own `{ role:
  "system", content: ... }` entry and unconditionally prepended before `input.messages`
  (actual conversation history). Since persisted conversation history never itself contains
  a `role: "system"` entry (system content is never persisted, as above), the prepended
  system message(s) are the *only* system-role entries in the request — never buried
  mid-conversation.
- **Exception 1 — `isOpenaiOauth`** (ChatGPT-subscription OAuth auth against OpenAI):
  `messages = input.messages`, nothing prepended. `system.join("\n")` instead becomes
  `options.instructions` (`request.ts:99`) — a separate top-level Responses API field. Same
  content, different wire channel.
- **Exception 2 — `input.isWorkflow`** (GitLab's DWS workflow models): same
  `messages = input.messages`. Instead, `workflowModel.systemPrompt =
  prepared.system.join("\n")` (`session/llm.ts:126`) is set as a property directly on the
  wrapped language model rather than inserted into the message list.

Both exceptions exist because those integrations have their own dedicated "system
instructions" concept in their wire protocol; opencode routes the same assembled text
there instead of synthesizing a `role: "system"` chat message.
