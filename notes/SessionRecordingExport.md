# Session recording and the two export paths

## Recording: every session is durably persisted locally, by default

opencode is not just streaming a conversation through — it durably records it as it
happens. As a turn streams in, `session/processor.ts` writes each message/part to a local
SQLite database in near-real-time via `Session.Service.updatePart`/`updateMessage`
(`session/session.ts:631-655`), not just once at the end. Storage:

- **Schema** (`packages/core/src/session/sql.ts`): a `session` table (title, directory,
  model, aggregate token/cost counters, timestamps), a `message` table (one row per
  message, JSON `data` column with role/agent/timestamps), and a `part` table (one row per
  message *part* — `text`, `reasoning`, `tool`, `file`, `patch`, `snapshot`, `subtask`,
  `agent`, `step-start`/`step-finish` — keyed by `message_id`, JSON `data` column holding
  the part content).
- **Location**: one global file, `~/.local/share/opencode/opencode.db` (or
  `opencode-<channel>.db` for non-stable install channels) — shared across every
  project/directory you've ever run opencode in on that machine, not per-project
  (`packages/core/src/database/database.ts:46-54`).
- **Override**: `Flag.OPENCODE_DB` can redirect the path, including to the special value
  `:memory:` for a non-durable, in-memory database that's discarded on process exit
  (`database.ts:44-46`) — not something a normal interactive user sets, but it exists (e.g.
  for tests or embedded/SDK use).

**What's never recorded**: the assembled system prompt (persona + `<env>` block +
instruction files + MCP instructions + skills listing, see `notes/SystemPrompt.md`) has no
storage slot at all — the `Part` union has no `"system"` type. It's rebuilt fresh every
provider turn in `LLMRequestPrep.prepare()` and sent straight to the model, never
persisted. The one exception: a one-off per-turn override attached via the
`sessions.prompt(...)` API's `system` field is stored as `SessionV1.User.system` on that
specific user message and does appear in exports.

## Export path 1: `opencode export` (CLI)

`packages/opencode/src/cli/cmd/export.ts:222-292`. Reads directly from the database via
`Session.Service.get(sessionID)` / `.messages({ sessionID })` — same process, no network
hop.

```bash
opencode export <sessionID>              # export a specific session
opencode export                          # interactive picker (most-recently-updated first)
opencode export <sessionID> --sanitize   # redact sensitive content
```

- Prompts/status go to **stderr**; the export itself (pretty-printed JSON, `{ info,
  messages }`) goes to **stdout only** — safe to redirect (`opencode export <id> >
  session.json`).
- `messages` is the full `{ info, parts }` array for every message, including every part
  type opencode stores — text, **reasoning** (verbatim unless `--sanitize`), tool
  input/output/state, files, patches, snapshots, subtasks.
- `--sanitize` replaces sensitive values (text, reasoning, tool I/O, file content, diffs,
  snapshots, `user.system`) with `[redacted:<kind>:<id>]` placeholders — structure and part
  types survive, content doesn't. Omit it if you specifically want reasoning tokens.

## Export path 2: `/export` (TUI slash command)

`session.export` command, `packages/tui/src/routes/session/index.tsx:943-1017`. Does
**not** touch the database directly — it reads from the TUI's client-side reactive store
(`sync.session` / `sync.data.message`, `packages/tui/src/context/sync.tsx`), which is
populated by calling opencode's HTTP/SDK API (`sdk.client.session.get`/`.messages`,
`sync.tsx:596-599`) when the session loads, and kept live via an SSE event subscription
(`sync.tsx:170`) while the session is open. So the chain is **TUI → HTTP API →
`Session.Service` → SQLite** — same ultimate source as the CLI path, just fronted by a
client cache and a network/IPC hop instead of a direct in-process call. If the TUI is
attached to a remote `opencode serve`, `/export` still works purely over the API; it never
touches the database file itself.

- Exports only **the currently open session** (not any session by ID).
- Produces a **human-readable Markdown transcript** (`formatTranscript(...)`,
  `packages/tui/src/util/transcript.ts`), not raw JSON.
- Opens an options dialog (`DialogExportOptions`) first: output filename (default
  `session-<id8>.md`), whether to include **"thinking"/reasoning content**
  (`transcript.ts:89` formats `type === "reasoning"` parts, but only if this toggle is on),
  whether to include tool-call details and assistant metadata, or skip saving and just open
  the transcript directly in `$EDITOR`.
- Writes to the current working directory, then reopens the file in `$EDITOR` for review
  before the final save.

## Comparison

| | `opencode export` (CLI) | `/export` (TUI) |
|---|---|---|
| Data source | direct DB read (`Session.Service`, same process) | client-side synced store, populated via HTTP API |
| Scope | any session, by ID or picker | only the currently open session |
| Format | raw JSON (`{ info, messages }`, every part verbatim) | formatted Markdown transcript |
| Reasoning content | included by default (redacted only with `--sanitize`) | included only if the dialog's "thinking" toggle is on |
| System prompt | never (not persisted anywhere) — except a one-off `user.system` per-turn override | never |
| Destination | stdout (redirectable) | a file in cwd, optionally reopened in `$EDITOR` |
| Intended use | tooling/programmatic, complete data | human-readable sharing/review |

Both are entirely local — reading local SQLite state via a local process or a local/attached
API — and are unrelated to the opt-in **session sharing** feature (`notes/ExternalSystems.md`),
which is the only one of these mechanisms that sends conversation content off-machine.
