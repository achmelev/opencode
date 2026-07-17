# `SessionReminders` — injecting dynamic, per-turn context

`packages/opencode/src/session/reminders.ts` is the harness's mechanism for splicing
per-turn, dynamically-computed text into the conversation right before it's sent to the
model — context that needs to reflect the *current* state (which agent is active, whether
a file exists, etc.) and so can't just live in the static system prompt. Despite the
generic name, it currently does exactly one thing: the plan/build-mode reminders covered
in `notes/PlanMode.md`. There's no broader "reminder registry" or plugin system behind
it — `SessionReminders.apply` is a single hardcoded function.

## Where it's called from

`packages/opencode/src/session/prompt.ts:1180`, inside `runLoop` — the core `while
(true)` loop that drives a session's provider turns (`prompt.ts:1081-1184`, roughly what
`CONTEXT.md` calls a Session Drain running provider turns to completion). Simplified:

```ts
while (true) {
  let msgs = yield* MessageV2.filterCompactedEffect(sessionID)   // reload durable history fresh
  // ... resolve lastUser / lastAssistant / tasks, handle subtask/compaction branches ...
  const agent = yield* agents.get(lastUser.agent)
  msgs = yield* SessionReminders.apply({ messages: msgs, agent, session })
  // ... build this turn's assistant message, then call llm.stream(...) ...
}
```

Two placement details matter:

- `msgs` is **reloaded from durable storage at the top of every loop iteration**
  (`MessageV2.filterCompactedEffect`), so `SessionReminders.apply` runs fresh on *every
  provider turn* — including every step of a multi-step tool-calling stretch within one
  user prompt, not just once per user message.
- It runs immediately before the assistant message is constructed and `llm.stream(...)`
  is invoked, so anything it appends is guaranteed to be part of that turn's model
  context.

## How it works mechanically

`SessionReminders.apply(input: { messages, agent, session })` uses two separate
`findLast` lookups, which play different roles:

- `userMessage = input.messages.findLast((msg) => msg.info.role === "user")`
  (`reminders.ts:23`) — the **target**. A message (`SessionV1.WithParts`) is `info`
  (role/agent/id/... metadata) plus a `parts` array of discrete content objects (text
  parts, tool parts, file parts, ...) — the actual content isn't a single string.
  "Attaching" a reminder means **pushing a brand-new, separate `TextPart` object onto
  `userMessage.parts`**, e.g. `userMessage.parts.push({ id: PartID.ascending(),
  messageID, sessionID, type: "text", text: ..., synthetic: true })`. This does not
  touch or rewrite the user's own original text part(s) in any way — it's an additional
  array element sitting alongside them, flagged `synthetic: true` to mark it as
  harness-injected rather than something the user actually typed. When the message is
  serialized into the LLM request, all of a message's parts get concatenated in order,
  so the model still sees the reminder as extra text within that turn — but structurally
  it's a sibling part, not a mutation of the user's own part.
- `assistantMessage = input.messages.findLast((msg) => msg.info.role === "assistant")`
  (`reminders.ts:51`, experimental path only) — **read-only**, never appended to. It's
  used purely to inspect `assistantMessage?.info.agent` for two decisions: detecting a
  plan→build transition (`reminders.ts:52`, previous assistant turn was `plan`, current
  agent isn't) and the dedup guard (`reminders.ts:70`, skip re-injecting the "still in
  plan mode" reminder if the previous assistant turn was already tagged `plan`).

## Two attachment strategies, depending on a flag

The current implementation branches on the `experimentalPlanMode` runtime flag
(`RuntimeFlags.Service`), and the two branches attach their synthetic parts differently:

### Default path (flag off) — `reminders.ts:26-48`

Pushes directly onto the in-memory array: `userMessage.parts.push({...})`. It is
**never persisted** via `sessions.updatePart` — since `msgs` is reloaded from the DB
fresh every loop iteration anyway, redoing this cheap, unconditional push every time is
simpler than persisting it. It only affects what the model sees for that one request,
not durable session history. No dedup check — it just fires whenever the active agent's
name is `"plan"` (or, for the build-switch reminder, whenever the active agent is
`"build"` and the history shows a prior `plan` turn).

### Experimental path (flag on) — `reminders.ts:51-89`

Calls `yield* sessions.updatePart(...)`, which **durably persists** the synthetic part
onto that user message in the database, then also pushes the returned part into the
in-memory array for the current request. Because this one is persisted, it needs (and
has) a dedup guard: before injecting the "still in plan mode" reminder, it checks whether
the *previous* assistant message is already tagged `agent: "plan"` and skips re-injecting
if so — otherwise a multi-step plan-mode stretch would accumulate duplicate reminder
parts in durable history on every provider turn.

This path is also the one that computes live state to interpolate into the reminder —
e.g. calling `Session.plan(session, ctx)` and `fsys.existsSafe(plan)` to tell the model
the actual plan-file path and whether it already exists (see `notes/PlanMode.md` for the
full content difference between `prompt/plan.txt` and `prompt/plan-mode.txt`).

## Summary

| | Default path | Experimental path (`experimentalPlanMode`) |
|---|---|---|
| Trigger | `agent.name === "plan"` / `"build"` | same, plus history scan for prior `plan` turn |
| Attachment | in-memory `parts.push`, not persisted | `sessions.updatePart`, durably persisted |
| Dedup | none — reinjected every provider turn | yes — skipped if last assistant already tagged `agent: "plan"` |
| Content | static (`prompt/plan.txt`, `prompt/build-switch.txt`) | includes live-computed plan-file path/state (`prompt/plan-mode.txt`) |
| Current scope | plan mode only | plan mode only |
