# ACP render templates: author's guide

This directory holds the built-in [template4d](https://github.com/worthies/template4d)
templates commander uses to turn raw ACP (`agent_client_protocol`) session
updates into Markdown chat messages. template4d is a pure-Dart engine
implementing a practical subset of Go's `text/template` syntax. One file
per agent type: `drift.yaml`, `claude.yaml`, `goose.yaml`, `opencode.yaml`,
plus `default.yaml` as the catch-all for any agent type without its own
file.

If you just want to reskin how an agent's messages look, copy one of these
files into your user override directory (Settings → Message Render
Templates) and edit away — you do not need to touch any Dart code.

## How a message becomes text

```
raw ACP JSON  →  acpNotificationKind (selects a template)  →  template4d template rendered against the RAW object  →  Markdown
```

1. `agents/` sends each session update as a raw ACP JSON object (a
   `SessionNotification`, or a permission request tagged with a synthetic
   `kind`). No formatting happens on that side.
2. commander's `acpNotificationKind` (in `lib/render/acp_message_view.dart`)
   looks at the object just long enough to pick which `kind` key selects a
   template (`update.sessionUpdate` for ordinary notifications, the
   synthetic `permission_request` tag otherwise) — **it does not transform
   the object**.
3. `AcpRenderEngine` renders the template registered under that `kind` for
   the message's agent type directly against the **unmodified raw
   notification** — `{{.sessionId}}`, `{{.update.<field>}}` for ordinary
   notifications, or `{{.toolCall}}`/`{{.options}}` for a permission
   request. A template can read and branch on *any* field the protocol
   actually sends, not just ones this module happens to expose.
4. The rendered fragments for every update in the batch are concatenated,
   in order, into the message's Markdown content.

Templates are expected to do their own branching (`{{if}}`, `{{switch}}`,
`{{range}}`) directly on ACP fields. What a template genuinely *cannot*
compute itself — icon lookups, JSON pretty-printing, human-readable token
counts, joining/extracting text from a ToolCallContent list, a unified
diff body — is exposed as a **custom function** registered in
`lib/render/acp_template_functions.dart`; see "Custom functions" below for
the full list. If a template needs a value that isn't a plain ACP field
and isn't covered by an existing function, that's a Dart change (see
"Adding a new custom function" below).

## File format

Each YAML file is a flat map: `<kind>: <template4d template string>`.
The built-in files use YAML double-quoted strings with `\n` escapes, but
YAML **block scalars** work just as well and are usually more readable for
anything longer than one line:

```yaml
# Double-quoted, single-line — what most of the built-in files use for
# short templates.
agent_message_chunk: "{{contentText .update.content}}"
current_mode_update: "\n\n[Mode: {{.update.currentModeId}}]"

# Literal block scalar (|) — real newlines, no \n escaping needed. `|-`
# strips the trailing newline, `|+` keeps it, plain `|` keeps exactly one.
tool_call: |-
  {{$content := toolCallContentJoined .update.content}}

  [Tool: **{{.update.title}}**]
  {{if $content}}

  {{$content}}
  {{else if .update.rawInput}}

  {{prettyJson .update.rawInput}}
  {{end}}

# Folded block scalar (>) — line breaks in the source fold into SPACES in
# the resulting string, not real newlines; `>-` additionally strips the
# trailing newline. Rarely what you want for a multi-branch template (see
# the note below) — mostly useful for wrapping one long single-line
# template for readability in the YAML source itself.
agent_message_chunk: >-
  {{contentText .update.content}}
```

- The key is the `kind` value that selects a template (almost always the
  ACP `sessionUpdate` discriminator verbatim, e.g. `tool_call`,
  `agent_message_chunk`, `plan`; see the full list below).
- Whichever scalar style you use, be deliberate about the exact newlines
  it produces — `|-`/`|+`/`|`/`>`/`>-` each have different trailing/folding
  behavior (this is standard YAML, not a template4d thing); a stray or
  missing blank line in your rendered output is usually a scalar-style
  mismatch, not a template bug. In particular, **`>`/`>-` fold every line
  break into a space** — don't use a folded scalar for a template that
  relies on real blank lines (e.g. the `\n\n` that separates Markdown
  paragraphs); use a literal block scalar (`|-`) instead. See the built-in
  `tool_call`/`plan`/`plan_update` templates for the literal-block-scalar
  idiom this project uses: a leading `{{$var := ...}}` assignment line
  (contributes no output), a deliberate blank line where the rendered
  output needs one, and `{{end}}` placed directly after the preceding tag
  (not on its own line) wherever a trailing newline must NOT be emitted.
- Field access on the notification itself always starts with a dot:
  `{{.update.title}}`, `{{.sessionId}}`, `{{.toolCall.title}}` (the last
  one only for `permission_request`, which has no `update` wrapper — see
  the per-kind reference below). Inside a `{{range}}` block, `.` refers to
  the current item instead.
- Templates emit their values as raw text (this is a Markdown target, not
  HTML — `**` and `` ` `` come through unescaped, there is no
  auto-escaping to worry about).
- A `kind` your file doesn't mention simply falls through to the next
  layer in the resolution chain (see below) — you only need to override
  the kinds you actually want to change.
- **Never add an `unknown:` key.** It is stripped at parse time no matter
  what you put there. `unknown` is not a real ACP kind; templating it
  would silently swallow every malformed/unrecognized notification instead
  of showing commander's raw-JSON fallback. See "Fallback behavior" below.

## Template resolution order

For a given agent type, commander resolves each `kind` independently,
most-specific first:

1. `<user override URL prefix>/<agentType>.yaml` (fetched via HTTP GET,
   if a URL prefix is configured)
2. `<user override dir>/<agentType>.yaml` (read from disk, if a directory
   is configured)
3. built-in `assets/render_templates/<agentType>.yaml`
4. `<user override URL prefix>/default.yaml`
5. `<user override dir>/default.yaml`
6. built-in `assets/render_templates/default.yaml`

So a user override that only defines `tool_call` still falls through to
the built-in file (or `default.yaml`) for every other kind — you never
need to copy the whole file just to change one line. If both a URL prefix
and a directory happen to be configured at once, the URL always wins per
kind (the settings dialog only ever lets you actively edit one at a time,
so this only matters if you've set both by hand, e.g. by editing
SharedPreferences directly).

## Loading templates from a URL

Instead of (or as well as) a local directory, templates can be served
from any HTTP(S) endpoint under a single URL prefix — useful for sharing
one template set across a team, or updating it without pushing a new app
build. Set it via Settings → Message Render Templates → "URL prefix", or
by writing the `render_template_url_prefix` key directly.

Given a prefix of `https://example.com/templates`, commander fetches:

- `https://example.com/templates/<agentType>.yaml` for each known agent
  type (`drift`, `claude`, `goose`, `opencode`)
- `https://example.com/templates/default.yaml` as the catch-all

Each response body is parsed exactly like a local file — same YAML
format, same template4d syntax, same per-kind merge/fallback rules. A
trailing slash on the prefix is tolerated (stripped automatically).

Failure handling is deliberately permissive, matching the "messages are
never silently dropped" invariant (see below): a request that times out
(8s), returns a non-200 status, or can't be reached at all is treated
exactly like a missing local file — that layer is simply absent, and
resolution falls through to the next one. A single unreachable URL never
breaks rendering; at worst you silently get the built-in templates back
for that agent type.

Live editing works the same way as with a local directory: the app polls
the configured URLs every 60s and reloads the moment a fetched body's
content actually changes (byte comparison via hash, so a CDN re-serving
byte-identical content never triggers a spurious reload).

## template4d syntax quick reference

template4d implements a practical subset of Go's `text/template` syntax.
Everything used by these built-in templates:

| Syntax | Meaning |
|---|---|
| `{{.field}}` | Interpolate a scalar value (string/number/bool → its string form). The leading `.` refers to the current notification object (or, inside `{{range}}`, the current item). |
| `{{if .flag}}...{{end}}` | Render the enclosed block only if `flag` is truthy (`true`, a non-empty string, or a non-empty list). |
| `{{if .flag}}...{{else}}...{{end}}` | Render the first block if truthy, otherwise the `else` block. |
| `{{if .a}}...{{else if .b}}...{{else}}...{{end}}` | Chained conditions, evaluated in order. |
| `{{range .list}}...{{end}}` | If `list` is a non-empty list of maps, render the block once per item, with that item's fields available via `.field` inside (not nested under `list`). |
| `{{range $i, $v := .list}}...{{end}}` | Same, but also binds `$i` (index) and `$v` (item) as variables — needed to get a 1-based display number via `{{add $i 1}}`, since there's no index field on the item itself. |
| `{{$x := pipeline}}` | Binds a template-scoped variable, e.g. `{{$content := toolCallContentJoined .update.content}}` so an expensive computation only runs once even if used in multiple branches. |
| `{{switch .tag}}{{case v1, v2}}...{{default}}...{{end}}` | **A template4d-specific extension — not in Go's own `text/template`.** First matching case wins (values compared to `.tag` with `==`, a case may list several comma-separated values); `{{switch}}` with no tag instead evaluates each case value for truthiness, like a chain of `{{if}}`/`{{else if}}`. Used for `tool_call_update`'s completed/failed/in-progress branching in `drift.yaml`/`claude.yaml`. |
| `{{funcName arg1 arg2}}`, `{{arg | funcName}}` | Call a function — either directly or piped. See "Built-in functions" and "Custom functions" below for what's available. |

There is no `{{! comment }}` Mustache-style comment; use `{{/* comment */}}`
instead, matching Go's own comment syntax.

### Built-in functions

Every template4d template has these available regardless of which app
embeds it (defined in template4d's own `lib/src/builtins.dart`):

| Function | Signature | Meaning |
|---|---|---|
| `and` | `and a b ...` | Returns the first falsy argument, or the last argument if all are truthy. |
| `or` | `or a b ...` | Returns the first truthy argument, or the last argument if all are falsy. |
| `not` | `not a` | Boolean negation of `a`'s truthiness. |
| `eq` | `eq a b ...` | `true` if `a` equals any of the remaining arguments. |
| `ne` | `ne a b` | `true` if `a` does not equal `b`. |
| `lt` / `le` / `gt` / `ge` | `lt a b` etc. | Numeric `<` / `<=` / `>` / `>=`; both arguments must be numbers. |
| `len` | `len v` | Length of a string, list, or map. |
| `index` | `index v i [j ...]` | Indexes into a list (by integer) or map (by key), chaining through nested indices. |
| `print` | `print a b ...` | Concatenates the string form of every argument. |
| `printf` | `printf fmt a ...` | `%s`/`%v`/`%d`/`%f` (all render the argument's string form) and `%%` (literal `%`) placeholders in `fmt`. |
| `upper` | `upper s` | Uppercases a string. |
| `lower` | `lower s` | Lowercases a string. |
| `add` | `add a b ...` | Sums any number of numeric arguments — mainly for a 1-based index inside `{{range}}`: `{{add $i 1}}`. |
| `sub` | `sub a b` | `a - b`. |

Example uses in this codebase: `{{if eq .update.status "completed"}}`,
`{{if not .update.rawInput}}`, `{{.update.title | upper}}`,
`{{add $i 1}}`.

### Custom functions

commander additionally registers its own function library
(`lib/render/acp_template_functions.dart`), available in every built-in
and user-override template alongside the built-ins above. These exist
because a template genuinely has no way to compute them itself — icon
lookups, JSON pretty-printing, human-readable token counts, joining a
list while mapping+filtering it, and so on:

| Function | Signature | Meaning |
|---|---|---|
| `contentText` | `contentText contentBlock` | Extracts display text from an ACP ContentBlock (`{type: text\|image\|audio\|resource\|resource_link, ...}`) — e.g. `{{contentText .update.content}}`. |
| `toolCallText` | `toolCallText toolCallContent` | Extracts display text from ONE ToolCallContent entry (wraps a ContentBlock, a diff, or a terminal reference) — for use inside `{{range .update.content}}{{toolCallText .}}{{end}}`. |
| `toolCallContentJoined` | `toolCallContentJoined list` | Joins an entire ToolCallContent list into one newline-separated string via `toolCallText`, skipping empty entries — e.g. `{{toolCallContentJoined .update.content}}`. |
| `toolIcon` | `toolIcon kind` | Maps an ACP `ToolKind` string to a display emoji (📖 read, ✏️ edit, 🗑️ delete, 📦 move, 🔍 search, ⚡ execute, 🧠 think, 🌐 fetch, 🔀 switch_mode, 🔧 anything else/missing). |
| `planIcon` | `planIcon status` | Maps a `PlanEntryStatus` string to a display icon (⬜ pending, ⏳ in_progress, ✅ completed, • anything else). |
| `diff` | `diff oldText newText` | Unified-diff-style body (` `/`-`/`+`-prefixed lines) via a line-level LCS alignment. A brand-new file (`oldText` null/empty) prefixes every line with `+`. |
| `rawOutputText` | `rawOutputText rawOutput` | Best-effort text for a `tool_call_update.rawOutput` value: a string is used verbatim, anything else is pretty-printed rather than dropped. |
| `prettyJson` | `prettyJson value` | Minimal JSON pretty-printer (one key per line) for display, e.g. `{{prettyJson .update.rawInput}}`. |
| `humanTokens` | `humanTokens n` | Compact token-count formatting: `999`, `127K`, `1.5M`, `2B`. |
| `pct` | `pct used size` | Integer percentage of `used/size`, clamped to `[0, 100]`; `0` when `size` isn't positive (no divide-by-zero guard needed in the template). |
| `metaTokens` | `metaTokens meta "input"` | A `usage_update` token count: prefers the agent's own pre-formatted `<key>Human` string in `_meta`, falling back to formatting the raw `<key>` integer. `key` is one of `input`/`output`/`cacheNew`/`cacheRead`. |
| `label` | `label title toolCallId` | `title` if non-empty, else `toolCallId` — a `tool_call_update`'s display label. |

If you add a new custom function to `acp_template_functions.dart`, list it
in this table too.

### The "prefer content over rawInput" pattern

Several kinds (`tool_call`, `tool_call_update`) carry two possible ways to
show a value with a priority order. `{{else if}}` expresses this directly
— note the `{{$content := ...}}` assignment so the (possibly expensive)
join only happens once:

```
{{$content := toolCallContentJoined .update.content}}
{{if $content}}
{{$content}}
{{else if .update.rawInput}}
{{prettyJson .update.rawInput}}
{{end}}
```

Read this as: "if there's joined content text, show it; otherwise, if
there's raw input, show that instead."

### Whitespace

Unlike Mustache, template4d does **not** silently strip any newline
following a tag on its own line — every `\n` you write is emitted exactly
as written. If you want to trim whitespace deliberately, use `{{-`/`-}}`
(trims adjacent whitespace on that side), matching Go's own convention.
Several built-in templates rely on this — e.g. `tool_call_update`'s
`{{$content := ... -}}` trims the newline that would otherwise appear
before the `{{switch}}` that follows it. See "File format" above for the
general idiom.

### Lists

`{{range .options}}...{{end}}` iterates `.options` (or any list field —
e.g. `.update.entries`, `.update.availableCommands`); inside the block, `.`
is the current item, so its fields are `.name`, `.optionId`, etc. There's
no index field on the item itself — use `{{range $i, $o := .options}}` to
bind one, then `{{add $i 1}}` for a 1-based display number:

```
{{if .options}}
Options:{{range $i, $o := .options}}
  {{add $i 1}}. {{if $o.name}}{{$o.name}}{{else}}{{$o.optionId}}{{end}}{{end}}
{{end}}
```

## The `kind`s and their raw fields

Every notification always has a `kind` that selects which template
applies (see `acpNotificationKind`), but **the object itself is passed to
the template unmodified** — the fields below are what ACP itself puts on
`.update` (or, for `permission_request`, at the top level). Nothing here
is precomputed for you; branch on these directly with `{{if}}`/
`{{switch}}`/`{{range}}`, and reach for a custom function (see the table
above) for anything a template can't express on its own.

### `agent_message_chunk`, `user_message_chunk`, `agent_thought_chunk`
Streamed text chunks (assistant reply, user echo, and reasoning/thinking
respectively — same shape, different kind).

- `.update.content` — a ContentBlock (`{type: text, text: "..."}` or
  `{type: image|audio|resource|resource_link, ...}`). Use
  `{{contentText .update.content}}` to get display text regardless of
  block type — it returns `""` for text blocks with no text, and a
  bracketed placeholder (`[Image]`, `[Link: name](uri)`, …) for non-text
  blocks.

### `tool_call`
A new tool invocation was started.

- `.update.title` — human-readable description, e.g. `"Read file"`.
- `.update.toolCallId` — the tool call's ACP id.
- `.update.status` — `pending` / `in_progress` / `completed` / `failed`.
- `.update.kind` — the raw ACP `ToolKind` string (`read`, `edit`,
  `execute`, …). Use `{{toolIcon .update.kind}}` for a display emoji.
- `.update.rawInput` — the tool's raw input parameters (a map), or absent.
  Use `{{prettyJson .update.rawInput}}` to display it; `{{if
  .update.rawInput}}` to check presence (an empty/absent map is falsy).
- `.update.content` — a list of ToolCallContent entries attached directly
  to this tool_call (e.g. a diff Claude attaches to its own tool_call
  instead of a later update). Use `{{toolCallContentJoined
  .update.content}}` to get one joined display string, or `{{range
  .update.content}}{{toolCallText .}}{{end}}` to handle each entry
  yourself.

### `tool_call_update`
A follow-up update to an existing tool call (status change and/or result).

- `.update.toolCallId`, `.update.status`, `.update.kind`,
  `.update.content` — as above.
- `.update.title` — may be absent; use `{{label .update.title
  .update.toolCallId}}` for "title if present, else the tool call id".
- `.update.rawOutput` — carries the result when there's no structured
  `content` list at all (e.g. Claude's `ToolUseSummary`). Use
  `{{rawOutputText .update.rawOutput}}` — a string renders verbatim,
  anything else is pretty-printed rather than dropped.

A ToolCallContent entry of `{"type": "diff", "path", "oldText"?,
"newText"}` — extracted via `toolCallText`/`toolCallContentJoined` — renders
as `[Diff: path]` followed by the fenced new text (or `[New file: path]`
when `oldText` is absent). If you want to build a diff display yourself
instead, the `{{diff oldText newText}}` function gives you the raw
unified-diff body (` `/`-`/`+`-prefixed lines, no `[Diff: ...]` header or
fencing) from two text fields.

### `permission_request`
Not a `SessionUpdate` at all — ACP models this as a JSON-RPC method call,
not a session notification, but agents/ tags it with a synthetic `kind` so
it flows through the same batch. This is the one kind whose raw JSON has
**no `update` wrapper** — fields sit directly on the notification object.

- `.toolCall.title` — the tool call's title, if any.
- `.toolCall.kind` — as above, use `{{toolIcon .toolCall.kind}}`.
- `.toolCall.rawInput` — as above, use `{{prettyJson .toolCall.rawInput}}`.
- `.options` — list of `{optionId, name, kind}`. `name` may be absent —
  fall back to `optionId` yourself: `{{if $o.name}}{{$o.name}}{{else}}
  {{$o.optionId}}{{end}}`. There's no precomputed index; use `{{range $i,
  $o := .options}}` + `{{add $i 1}}` for a 1-based display number.

This is the one kind where showing `options` isn't optional in spirit — a
permission request with no visible options is useless to whoever has to
answer it. Keep a `{{range .options}}` loop in any override.

### `plan`
The agent's full execution plan (replaces any previous plan).

- `.update.entries` — list of `{content, status, priority?}`. Use `{{range
  $i, $e := .update.entries}}` for a 1-based index (`{{add $i 1}}`) and
  `{{planIcon $e.status}}` for a display icon (⬜ pending, ⏳ in_progress,
  ✅ completed, • anything else).

### `plan_update` *(UNSTABLE ACP capability)*
A content update for a plan by id — can be structured entries, a markdown
blob, or a file reference.

- `.update.plan.id`, `.update.plan.type` (`items` / `markdown` / `file`).
- `.update.plan.entries` — same shape as `plan` above (no `priority`).
- `.update.plan.content` — a raw markdown blob to embed verbatim.
- `.update.plan.uri` — a file reference.

Check each with `{{if .update.plan.entries}}` / `{{if
.update.plan.content}}` / `{{if .update.plan.uri}}` — exactly one is
typically present per update.

### `plan_removed` *(UNSTABLE ACP capability)*
- `.update.id` — the removed plan's id.

### `available_commands_update`
- `.update.availableCommands` — list of `{name, description?}`. Use `{{len
  .update.availableCommands}}` for the count.

### `current_mode_update`
- `.update.currentModeId` — the new mode id string.

### `config_option_update`
- `.update.configOptions` — list of `{name, type, currentValue,
  description?}`. `currentValue` covers both select-style (an id string)
  and boolean options uniformly.

### `session_info_update`
- `.update.title`, `.update.updatedAt` — the latter an ISO-ish timestamp
  string, shown verbatim (no date formatting is done for you).

### `usage_update` *(UNSTABLE ACP capability)*
Context-window and cost tracking.

- `.update.used`, `.update.size` — raw integers (context tokens used /
  window size). Use `{{humanTokens .update.used}}` for compact display
  (`999`, `127K`, `1.5M`, …), `{{pct .update.used .update.size}}` for a
  0–100 percentage (`0` when `size` isn't positive — check `{{if
  .update.size}}` before showing it).
- `.update._meta` — an optional map that may carry `model`, plus
  `input`/`output`/`cacheNew`/`cacheRead` (raw integers) and/or their
  pre-formatted `<key>Human` string counterparts. Use `{{metaTokens
  .update._meta "input"}}` etc. rather than reading the raw/human fields
  yourself — it handles the string-preferred-over-computed fallback for
  you.

### Any other/future `sessionUpdate` kind
If the ACP protocol adds a new `sessionUpdate` discriminator this file
doesn't have a template for, `acpNotificationKind` still returns that
kind verbatim (it's only ever `"unknown"` for genuinely malformed input —
see "Fallback behavior" below). You can add a template keyed by the new
kind name immediately; the notification's raw fields are already there
for you to reference, whatever they are — no Dart change is required
purely to *add* a template for a new-but-well-formed kind, only to add a
new *custom function* if the fields need computation a template can't do
itself.

## Fallback behavior ("messages are never silently dropped")

This pipeline is built around one invariant: a message is never silently
swallowed just because a template is missing or a notification is
malformed.

- If a `kind` has no template anywhere in the resolution chain, or its
  template throws while rendering, the engine falls back to plain text
  extracted straight from the raw notification (`{{contentText
  .update.content}}`'s equivalent, but only for the three plain
  content-chunk kinds — see `AcpRenderEngine._extractableText`), and
  failing that, a raw JSON dump wrapped in a `<pre>` block. It does
  **not** render nothing.
- If an agent type has no templates at all in any layer (not even
  `default.yaml` — e.g. a corrupted asset bundle), the entire batch
  renders via `lib/render/builtin_fallback.dart`'s hardcoded formatter
  instead of a template, which is output-equivalent to `default.yaml` for
  the well-known kinds and still falls back to a raw dump for anything it
  doesn't recognize.
- A notification with no `update` map or no `sessionUpdate` discriminator
  resolves to the synthetic kind `"unknown"`. There is deliberately no
  `unknown` template anywhere — see "File format" above for why — so this
  case always falls through to the raw-JSON dump.

When customizing templates, you generally don't need to think about this;
it's what lets you write an incomplete override file without worrying
you've broken anything.

## Testing a template change

1. Edit the YAML in your user override directory (or at your override
   URL prefix), or, if working on the built-ins, edit the file here and
   rebuild commander so the new asset is bundled.
2. If you're editing a **user override** (directory or URL), just save
   it — commander polls the configured source every 60s (content hash,
   not mtime/ETag, so a save-without-changes or a CDN re-serving
   identical bytes never triggers a spurious reload) and calls
   `AcpRenderEngine.reloadTemplates` automatically the moment content
   actually changes. No restart, no trip through Settings required. See
   `AcpRenderEngine.startWatching` in `lib/render/acp_render_engine.dart`
   if you need to change the interval or watch behavior. Editing a
   **built-in** asset always needs a rebuild — the watch only ever looks
   at the user override source.
3. Trigger the relevant ACP update again (e.g. re-run a tool call) and
   check the rendered message. If you don't want to wait up to 60s, the
   Settings → Message Render Templates dialog's Save button still
   force-reloads immediately.

For quick iteration without a live agent, `commander/test/render/` has
unit tests that render a hand-built raw notification object through
`AcpRenderEngine`/`TemplateRegistry` directly — see
`acp_render_engine_test.dart` and `template_registry_test.dart` for the
pattern (build a `{'update': {'sessionUpdate': '<kind>', ...}}` map and
call `.render(...)` on it).

## Adding a new custom function (Dart-side change)

If a template needs to compute something no existing custom function
covers (see the table above) — as opposed to just reading/branching on a
plain ACP field, which needs no Dart change at all — add it to
`lib/render/acp_message_view.dart` and register it in
`lib/render/acp_template_functions.dart`:

1. Write the pure Dart function in `acp_message_view.dart` (see
   `contentText`/`toolIcon`/`prettyJson`/etc. for the existing style —
   plain functions taking/returning basic Dart values, no ACP-specific
   types).
2. Register it in `acpTemplateFunctions` (`acp_template_functions.dart`)
   under whatever name templates should call it by, adapting/validating
   argument types at that boundary (template4d passes `List<Object?>`).
3. Add unit tests for the function itself in
   `test/render/acp_message_view_test.dart`.
4. Reference the new function from the built-in template(s) that need it,
   and add it to the custom-functions table in this README.
