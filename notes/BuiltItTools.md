# Built-in tools

All built-in tools live in `packages/opencode/src/tool/`. Each is defined via `Tool.define(id,
...)` (`tool.ts`) with an `Effect.Schema.Struct` parameter schema and a `description` string
(usually loaded from a co-located `.txt` file via a bundled text import, same mechanism as the
system-prompt `.txt` files covered in `notes/SystemPrompt.md`). The registry
(`packages/opencode/src/tool/registry.ts:224-247`) assembles the final built-in list; custom
tools from plugins/`.opencode/tool(s)` directories are appended separately and are out of scope
here.

## How the schema actually reaches the model

Unless a tool supplies its own `jsonSchema` explicitly, the parameter schema sent to the LLM is
derived automatically from the tool's Effect `Schema.Struct` via `ToolJsonSchema.fromSchema`
(`tool/json-schema.ts:8-22`): a JSON Schema draft 2020-12 document, with local `$ref`/`$defs`
inlined, `additionalProperties: true` stripped, `anyOf` unions simplified where possible (e.g. an
optional field's `T | null` collapses back to `T`), and bare `integer` types widened to
`[MIN_SAFE_INTEGER, MAX_SAFE_INTEGER]`. Every `Schema.optional(...)` field becomes a non-required
JSON Schema property; every plain field is required. Per-field `.annotate({ description: "..." })`
calls become the JSON Schema `description` on that property — these are quoted verbatim below.

## Which tools are actually available

Not every listed tool is present for every session — `registry.ts:226-244` filters:

| Tool | Condition |
|---|---|
| `question` | only if `flags.client` is `app`/`cli`/`desktop`, or `OPENCODE_ENABLE_QUESTION_TOOL` |
| `edit` / `write` | used for most models |
| `apply_patch` | used instead of `edit`/`write` when the model id contains `gpt-` (excluding `gpt-4` and `oss` variants) — see `registry.ts:292-295` |
| `execute` (code mode) | only if `OPENCODE_EXPERIMENTAL_CODE_MODE`, and only if MCP tools are actually available to describe |
| `lsp` | only if `OPENCODE_EXPERIMENTAL_LSP_TOOL` |
| `plan_exit` (`plan`) | only if `OPENCODE_EXPERIMENTAL_PLAN_MODE` **and** `flags.client === "cli"` |
| `search` (websearch) | only if `providerID === "opencode"`, or `OPENCODE_ENABLE_EXA`/`OPENCODE_ENABLE_PARALLEL` (`webSearchEnabled`, `registry.ts:58-60`) |

All other tools (`invalid`, `shell`, `read`, `glob`, `grep`, `todo`, `skill`) are always present.
`invalid` is a synthetic fallback the model never intentionally calls — it's the tool that gets
substituted when a tool-call fails to repair (see `experimental_repairToolCall` in `session/llm.ts`).

---

## `read`

**Parameters** (`tool/read.ts:28-36`):

| Name | Type | Required | Description |
|---|---|---|---|
| `filePath` | string | yes | The absolute path to the file or directory to read |
| `offset` | non-negative int | no | The line number to start reading from (1-indexed) |
| `limit` | non-negative int | no | The maximum number of lines to read (defaults to 2000) |

**Description** (`tool/read.txt`):
> Read a file or directory from the local filesystem. If the path does not exist, an error is returned.
>
> Usage:
> - The filePath parameter should be an absolute path.
> - By default, this tool returns up to 2000 lines from the start of the file.
> - The offset parameter is the line number to start from (1-indexed).
> - To read later sections, call this tool again with a larger offset.
> - Use the grep tool to find specific content in large files or files with long lines.
> - If you are unsure of the correct file path, use the glob tool to look up filenames by glob pattern.
> - Contents are returned with each line prefixed by its line number as `<line>: <content>`. For example, if a file has contents "foo\n", you will receive "1: foo\n". For directories, entries are returned one per line (without line numbers) with a trailing `/` for subdirectories.
> - Any line longer than 2000 characters is truncated.
> - Call this tool in parallel when you know there are multiple files you want to read.
> - Avoid tiny repeated slices (30 line chunks). If you need more context, read a larger window.
> - This tool can read image files and PDFs and return them as file attachments.

---

## `write`

**Parameters** (`tool/write.ts:20-25`):

| Name | Type | Required | Description |
|---|---|---|---|
| `content` | string | yes | The content to write to the file |
| `filePath` | string | yes | The absolute path to the file to write (must be absolute, not relative) |

**Description** (`tool/write.txt`):
> Writes a file to the local filesystem.
>
> Usage:
> - This tool will overwrite the existing file if there is one at the provided path.
> - If this is an existing file, you MUST use the Read tool first to read the file's contents. This tool will fail if you did not read the file first.
> - ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
> - NEVER proactively create documentation files (*.md) or README files. Only create documentation files if explicitly requested by the User.
> - Only use emojis if the user explicitly requests it. Avoid writing emojis to files unless asked.

---

## `edit`

**Parameters** (`tool/edit.ts:47-56`):

| Name | Type | Required | Description |
|---|---|---|---|
| `filePath` | string | yes | The absolute path to the file to modify |
| `oldString` | string | yes | The text to replace |
| `newString` | string | yes | The text to replace it with (must be different from oldString) |
| `replaceAll` | boolean | no | Replace all occurrences of oldString (default false) |

**Description** (`tool/edit.txt`):
> Performs exact string replacements in files.
>
> Usage:
> - You must use your `Read` tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file.
> - When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix. The line number prefix format is: line number + colon + space (e.g., `1: `). Everything after that space is the actual file content to match. Never include any part of the line number prefix in the oldString or newString.
> - ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
> - Only use emojis if the user explicitly requests it. Avoid adding emojis to files unless asked.
> - The edit will FAIL if `oldString` is not found in the file with an error "oldString not found in content".
> - The edit will FAIL if `oldString` is found multiple times in the file with an error "Found multiple matches for oldString. Provide more surrounding lines in oldString to identify the correct match." Either provide a larger string with more surrounding context to make it unique or use `replaceAll` to change every instance of `oldString`.
> - Use `replaceAll` for replacing and renaming strings across the file. This parameter is useful if you want to rename a variable for instance.

(Under the hood, matching cascades through nine fuzzy-replacer strategies — exact, line-trimmed, block-anchor, whitespace-normalized, indentation-flexible, escape-normalized, trimmed-boundary, context-aware, multi-occurrence — before failing; not exposed to the model, purely an implementation detail of `replace()`.)

---

## `apply_patch`

Used instead of `edit`/`write` for GPT-family models (see availability table above).

**Parameters** (`tool/apply_patch.ts:18-20`):

| Name | Type | Required | Description |
|---|---|---|---|
| `patchText` | string | yes | The full patch text that describes all changes to be made |

**Description** (`tool/apply_patch.txt`):
> Use the `apply_patch` tool to edit files. Your patch language is a stripped‑down, file‑oriented diff format designed to be easy to parse and safe to apply. You can think of it as a high‑level envelope:
>
> ```
> *** Begin Patch
> [ one or more file sections ]
> *** End Patch
> ```
>
> Within that envelope, you get a sequence of file operations. You MUST include a header to specify the action you are taking. Each operation starts with one of three headers:
>
> - `*** Add File: <path>` - create a new file. Every following line is a + line (the initial contents).
> - `*** Delete File: <path>` - remove an existing file. Nothing follows.
> - `*** Update File: <path>` - patch an existing file in place (optionally with a rename).
>
> Example patch:
> ```
> *** Begin Patch
> *** Add File: hello.txt
> +Hello world
> *** Update File: src/app.py
> *** Move to: src/main.py
> @@ def greet():
> -print("Hi")
> +print("Hello, world!")
> *** Delete File: obsolete.txt
> *** End Patch
> ```
>
> It is important to remember:
> - You must include a header with your intended action (Add/Delete/Update)
> - You must prefix new lines with `+` even when creating a new file

---

## `glob`

**Parameters** (`tool/glob.ts:10-15`):

| Name | Type | Required | Description |
|---|---|---|---|
| `pattern` | string | yes | The glob pattern to match files against |
| `path` | string | no | The directory to search in. If not specified, the current working directory will be used. IMPORTANT: Omit this field to use the default directory. DO NOT enter "undefined" or "null" - simply omit it for the default behavior. Must be a valid directory path if provided. |

**Description** (`tool/glob.txt`):
> - Fast file pattern matching tool that works with any codebase size
> - Supports glob patterns like "**/*.js" or "src/**/*.ts"
> - Returns matching file paths
> - Use this tool when you need to find files by name patterns
> - When you are doing an open-ended search that may require multiple rounds of globbing and grepping, use the Task tool instead
> - You have the capability to call multiple tools in a single response. It is always better to speculatively perform multiple searches as a batch that are potentially useful.

---

## `grep`

**Parameters** (`tool/grep.ts:10-18`):

| Name | Type | Required | Description |
|---|---|---|---|
| `pattern` | string | yes | The regex pattern to search for in file contents |
| `path` | string | no | The directory to search in. Defaults to the current working directory. |
| `include` | string | no | File pattern to include in the search (e.g. "*.js", "*.{ts,tsx}") |

**Description** (`tool/grep.txt`):
> - Fast content search tool that works with any codebase size
> - Searches file contents using regular expressions
> - Supports full regex syntax (eg. "log.*Error", "function\s+\w+", etc.)
> - Filter files by pattern with the include parameter (eg. "*.js", "*.{ts,tsx}")
> - Returns file paths and line numbers with matching lines
> - Use this tool when you need to find files containing specific patterns
> - If you need to identify/count the number of matches within files, use the Bash tool with `rg` (ripgrep) directly. Do NOT use `grep`.
> - When you are doing an open-ended search that may require multiple rounds of globbing and grepping, use the Task tool instead

---

## `bash` (shell)

Tool id comes from `ShellID.ToolID` (typically `"bash"`). Both the description *and* the
parameter schema are dynamically rendered per-session by `ShellPrompt.render(...)`
(`tool/shell/prompt.ts`), based on the detected/configured shell (bash, PowerShell 7+, Windows
PowerShell 5.1, or cmd.exe) and the platform.

**Parameters** (`tool/shell/prompt.ts:15-23`, same shape regardless of shell):

| Name | Type | Required | Description |
|---|---|---|---|
| `command` | string | yes | The command to execute |
| `timeout` | positive int | no | Optional timeout in milliseconds |
| `workdir` | string | no | The working directory to run the command in. Defaults to the current directory. Use this instead of 'cd' commands. |

**Description** — templated from `tool/shell/shell.txt`, with an intro line, an OS/shell line, a
workdir-usage note, a shell-specific command section (quoting rules, tool-preference guidance —
e.g. "use Glob not find", "use Grep not grep/rg", "use Read not cat/head/tail" — and chaining
guidance that differs for bash `&&`, PowerShell `; if ($?) { }`, and cmd `&`), plus a fixed Git/GitHub
section:
> # Git and GitHub
> - Only commit, amend, push, or create PRs when explicitly requested.
> - Before committing, inspect `git status`, `git diff`, and `git log --oneline -10`; stage only intended files and never commit secrets.
> - Write a concise commit message that matches the repo style.
> - Do not update git config, skip hooks, use interactive `-i`, force-push, or create empty commits unless explicitly requested.
> - If a commit fails or hooks reject it, fix the issue and create a new commit; do not amend the failed commit.
> - Before creating a PR, inspect status, diff, remote tracking, recent commits, and the diff from the base branch.
> - Review all commits included in the PR, not just the latest commit.
> - Use `gh` for GitHub tasks, including PRs, issues, checks, and releases; return the PR URL when done.

The `${tmp}` placeholder resolves to opencode's temp directory, which the description calls out
as pre-approved for external-directory access.

---

## `task`

**Parameters** (`tool/task.ts:43-62`):

| Name | Type | Required | Description |
|---|---|---|---|
| `description` | string | yes | A short (3-5 words) description of the task |
| `prompt` | string | yes | The task for the agent to perform |
| `subagent_type` | string | yes | The type of specialized agent to use for this task |
| `task_id` | string | no | This should only be set if you mean to resume a previous task (you can pass a prior task_id and the task will continue the same subagent session as before instead of creating a fresh one) |
| `command` | string | no | The command that triggered this task |
| `background` | boolean | no, **only when `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`** | Run the agent in the background. You will be notified when it completes. DO NOT sleep, poll, or proactively check on its progress |

**Description** is `task.txt` plus, at the very end, a dynamically-generated list of available
agent types and their descriptions (`ToolRegistry.describeTask`, `registry.ts:260-273`) — filtered
to non-primary agents the caller's permission ruleset doesn't deny for `task`. `task.txt`:
> Launch a new agent to handle complex, multistep tasks autonomously.
>
> When using the Task tool, you must specify a subagent_type parameter to select which agent type to use.
>
> When NOT to use the Task tool:
> - If you want to read a specific file path, use the Read or Glob tool instead of the Task tool, to find the match more quickly
> - If you are searching for a specific class definition like "class Foo", use the Grep tool instead, to find the match more quickly
> - If you are searching for code within a specific file or set of 2-3 files, use the Read tool instead of the Task tool, to find the match more quickly
> - If no available agent is a good fit for the task, use other tools directly
>
> Usage notes:
> 1. Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses
> 2. Once you have delegated work to an agent, do not duplicate that work yourself. Continue with non-overlapping tasks, or wait for the result. For background tasks, you will be notified automatically when the result is ready.
> 3. When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result. The output includes a task_id you can reuse later to continue the same subagent session.
> 4. Each agent invocation starts with a fresh context unless you provide task_id to resume the same subagent session (which continues with its previous messages and tool outputs). When starting fresh, your prompt should contain a highly detailed task description for the agent to perform autonomously and you should specify exactly what information the agent should return back to you in its final and only message to you.
> 5. The agent's outputs should generally be trusted
> 6. Clearly tell the agent whether you expect it to write code or just to do research (search, file reads, web fetches, etc.), since it is not aware of the user's intent. Tell it how to verify its work if possible (e.g., relevant test commands).
> 7. If the agent description mentions that it should be used proactively, then you should try your best to use it without the user having to ask for it first. Use your judgement.

When background subagents are disabled, `task` also overrides `jsonSchema` explicitly (dropping
`background` from the wire schema entirely, `task.ts:341`) rather than relying on the
optional-field default.

---

## `fetch` (webfetch)

**Parameters** (`tool/webfetch.ts:13-22`):

| Name | Type | Required | Description |
|---|---|---|---|
| `url` | string | yes | The URL to fetch content from |
| `format` | `"text"` \| `"markdown"` \| `"html"` | no, defaults to `"markdown"` | The format to return the content in (text, markdown, or html). Defaults to markdown. |
| `timeout` | number | no | Optional timeout in seconds (max 120) |

**Description** (`tool/webfetch.txt`):
> - Fetches content from a specified URL
> - Takes a URL and optional format as input
> - Fetches the URL content, converts to requested format (markdown by default)
> - Returns the content in the specified format
> - Use this tool when you need to retrieve and analyze web content
>
> Usage notes:
> - IMPORTANT: if another tool is present that offers better web fetching capabilities, is more targeted to the task, or has fewer restrictions, prefer using that tool instead of this one.
> - The URL must be a fully-formed valid URL
> - HTTP URLs will be automatically upgraded to HTTPS
> - Format options: "markdown" (default), "text", or "html"
> - This tool is read-only and does not modify any files
> - Results may be summarized if the content is very large

---

## `search` (websearch)

**Parameters** (`tool/websearch.ts:10-25`):

| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | yes | Websearch query |
| `numResults` | number | no | Number of search results to return (default: 8) |
| `livecrawl` | `"fallback"` \| `"preferred"` | no | Live crawl mode - 'fallback': use live crawling as backup if cached content unavailable, 'preferred': prioritize live crawling (default: 'fallback') |
| `type` | `"auto"` \| `"fast"` \| `"deep"` | no | Search type - 'auto': balanced search (default), 'fast': quick results, 'deep': comprehensive search |
| `contextMaxCharacters` | number | no | Maximum characters for context string optimized for LLMs (default: 10000) |

Routes to either Exa or Parallel as the underlying search provider (`selectWebSearchProvider`,
deterministic per-session hash unless a flag/env forces one). **Description** (`tool/websearch.txt`,
`{{year}}` substituted with the real current year at call time):
> - Search the web using the session's web search provider - performs real-time web searches and can scrape content from specific URLs
> - Provides up-to-date information for current events and recent data
> - Supports configurable result counts and returns the content from the most relevant websites
> - Use this tool for accessing information beyond knowledge cutoff
> - Searches are performed automatically within a single API call
>
> Usage notes:
> - Supports live crawling modes when available: 'fallback' (backup if cached unavailable) or 'preferred' (prioritize live crawling)
> - Search types when available: 'auto' (balanced), 'fast' (quick results), 'deep' (comprehensive search)
> - Configurable context length for optimal LLM integration
> - Domain filtering and advanced search options available
>
> The current year is {{year}}. You MUST use this year when searching for recent information or current events
> - Example: If the current year is 2026 and the user asks for "latest AI news", search for "AI news 2026", NOT "AI news 2025"

---

## `todo` (todowrite)

**Parameters** (`tool/todo.ts:6-8`, item shape from `packages/schema/src/session-todo.ts:7-15`):

| Name | Type | Required | Description |
|---|---|---|---|
| `todos` | array of `Todo` | yes | The updated todo list |

Each `Todo` item:

| Field | Type | Description |
|---|---|---|
| `content` | string | Brief description of the task |
| `status` | string | Current status of the task: pending, in_progress, completed, cancelled |
| `priority` | string | Priority level of the task: high, medium, low |

**Description** (`tool/todowrite.txt`) — the longest of the built-in tool prompts, covering when
to use/skip it, valid states, rules (exactly one `in_progress` at a time, update in real time,
preserve user-provided commands verbatim), and worked examples of when to use it vs. skip it. See
`packages/opencode/src/tool/todowrite.txt` for the full text.

---

## `skill`

**Parameters** (`tool/skill.ts:8-10`):

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | The name of the skill from available_skills |

**Description** (`tool/skill.txt`):
> Load a specialized skill when the task at hand matches one of the skills listed in the system prompt.
>
> Use this tool to inject the skill's instructions and resources into current conversation. The output may contain detailed workflow guidance as well as references to scripts, files, etc in the same directory as the skill.
>
> The skill name must match one of the skills listed in your system prompt.

(The actual catalog of available skill names/descriptions is injected separately into the system
prompt — see the "skills" section of `notes/SystemPrompt.md` — not into this tool's own schema.)

---

## `question`

Conditionally available — see availability table.

**Parameters** (`tool/question.ts:6-8`, item shape — `Question.Prompt` — from
`packages/schema/src/v1/question.ts:15-31`):

| Name | Type | Required | Description |
|---|---|---|---|
| `questions` | array of `QuestionPrompt` | yes | Questions to ask |

Each `QuestionPrompt` item:

| Field | Type | Description |
|---|---|---|
| `question` | string | Complete question |
| `header` | string | Very short label (max 30 chars) |
| `options` | array of `{ label, description }` | Available choices — `label`: "Display text (1-5 words, concise)", `description`: "Explanation of choice" |
| `multiple` | boolean (optional) | Allow selecting multiple choices |

Note: the model-facing schema is `Prompt` (question/header/options/multiple only) — the `custom`
flag (whether a free-text "type your own answer" option is added) exists on the richer `Info`
type but is applied server-side as a default, not model-controlled per call.

**Description** (`tool/question.txt`):
> Use this tool when you need to ask the user questions during execution. This allows you to:
> 1. Gather user preferences or requirements
> 2. Clarify ambiguous instructions
> 3. Get decisions on implementation choices as you work
> 4. Offer choices to the user about what direction to take.
>
> Usage notes:
> - When `custom` is enabled (default), a "Type your own answer" option is added automatically; don't include "Other" or catch-all options
> - Answers are returned as arrays of labels; set `multiple: true` to allow selecting more than one
> - If you recommend a specific option, make that the first option in the list and add "(Recommended)" at the end of the label

---

## `lsp`

Experimental — only present behind `OPENCODE_EXPERIMENTAL_LSP_TOOL`.

**Parameters** (`tool/lsp.ts:11-35`):

| Name | Type | Required | Description |
|---|---|---|---|
| `operation` | one of `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls` | yes | The LSP operation to perform |
| `filePath` | string | yes | The absolute or relative path to the file |
| `line` | int ≥ 1 | yes | The line number (1-based, as shown in editors) |
| `character` | int ≥ 1 | yes | The character offset (1-based, as shown in editors) |
| `query` | string | no | Search query for workspaceSymbol. Empty string requests all symbols. |

**Description** (`tool/lsp.txt`):
> Interact with Language Server Protocol (LSP) servers to get code intelligence features.
>
> Supported operations:
> - goToDefinition: Find where a symbol is defined
> - findReferences: Find all references to a symbol
> - hover: Get hover information (documentation, type info) for a symbol
> - documentSymbol: Get all symbols (functions, classes, variables) in a document
> - workspaceSymbol: List project-wide symbols matching a query string
> - goToImplementation: Find implementations of an interface or abstract method
> - prepareCallHierarchy: Get call hierarchy item at a position (functions/methods)
> - incomingCalls: Find all functions/methods that call the function at a position
> - outgoingCalls: Find all functions/methods called by the function at a position
>
> All operations require: filePath, line (1-based), character (1-based). workspaceSymbol also accepts `query` (empty string requests all symbols); for workspaceSymbol, filePath is not sent in the LSP workspace/symbol request — it is used by opencode to select and start the matching LSP server. LSP servers must be configured for the file type, or an error is returned.

(Note: `line`/`character` are validated required even for `workspaceSymbol`, though the
description says only `query` is needed there — a minor schema/description mismatch, not
something the model can omit.)

---

## `plan_exit`

Experimental — only present behind `OPENCODE_EXPERIMENTAL_PLAN_MODE` and only for the CLI client.

**Parameters**: none (`Schema.Struct({})`, `tool/plan.ts:13`) — takes no arguments at all.

**Description** (`tool/plan-exit.txt`):
> Use this tool when you have completed the planning phase and are ready to exit plan agent.
>
> This tool will ask the user if they want to switch to build agent to start implementing the plan.
>
> Call this tool:
> - After you have written a complete plan to the plan file
> - After you have clarified any questions with the user
> - When you are confident the plan is ready for implementation
>
> Do NOT call this tool:
> - Before you have created or finalized the plan
> - If you still have unanswered questions about the implementation
> - If the user has indicated they want to continue planning

Calling it asks the user a yes/no confirmation (via the `question` mechanism internally, not the
`question` tool) and, on "Yes," synthesizes a new user-role message that switches the session to
the `build` agent.

---

## `execute` (code mode)

Experimental — only present behind `OPENCODE_EXPERIMENTAL_CODE_MODE`, and only when there are
MCP tools available to expose to it.

**Parameters** (`tool/code-mode.ts:16-20`):

| Name | Type | Required | Description |
|---|---|---|---|
| `code` | string | yes | Script body executed by the confined interpreter. |

**Base description**: `"Run a confined orchestration script with access to connected MCP tools."`
— but the final description sent to the model also has a dynamically generated catalog of
callable MCP tools appended (`ToolRegistry.describeCodeMode`, `registry.ts:275-284`, built via
`@opencode-ai/codemode`'s `describeCatalog`), so the effective description varies per session
based on which MCP servers/tools are actually connected and permitted.

---

## `invalid`

Not a real capability — a fallback the model should never call intentionally.

**Parameters** (`tool/invalid.ts:4-7`): `tool` (string, required), `error` (string, required).

**Description**: `"Do not use"`. Its `execute` just echoes back the recorded error message; it
exists as the target `experimental_repairToolCall` (`session/llm.ts`) redirects to when a
tool-call's name/arguments can't be repaired into a valid call against a real tool.
