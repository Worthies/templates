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
raw ACP JSON  →  flattenAcpNotification (Dart)  →  view map  →  template4d template  →  Markdown
```

1. `agents/` sends each session update as a raw ACP JSON object (a
   `SessionNotification`, or a permission request tagged with a synthetic
   `kind`). No formatting happens on that side.
2. commander's `flattenAcpNotification` (in
   `lib/render/acp_message_view.dart`) turns one such object into a flat
   **view map**: a `kind` string (which template to use) plus a handful of
   already-computed fields (text, booleans, lists) ready for the template.
3. `AcpRenderEngine` looks up the template registered under that `kind` for
   the message's agent type and renders it against the view map.
4. The rendered fragments for every update in the batch are concatenated,
   in order, into the message's Markdown content.

Every percentage, human-readable count, icon choice, or existence check a
template might want is precomputed in Dart and handed to it as a plain
field — templates only branch on fields, they don't compute derived values
themselves (aside from the small set of functions registered in
`lib/render/acp_template_functions.dart`, e.g. `{{diff .Old .New}}`). If a
template needs a value that doesn't exist yet, that's usually a Dart
change (see "Adding a new view field" below).

## File format

Each YAML file is a flat map: `<kind>: "<template4d template string>"`.

```yaml
agent_message_chunk: "{{.text}}"
tool_call: "\n\n[Tool: **{{.title}}**]\n{{if .hasInput}}\n\n{{.inputJson}}\n{{end}}"
```

- The key is the `kind` value in the view map (almost always the ACP
  `sessionUpdate` discriminator verbatim, e.g. `tool_call`,
  `agent_message_chunk`, `plan`; see the full list below).
- The value is a single-line YAML string containing a template4d template.
  Use `\n` for newlines — YAML double-quoted strings interpret backslash
  escapes, so this is the only way to control line breaks precisely.
- Field access always starts with a dot: `{{.field}}`, not `{{field}}` —
  the leading dot refers to the current view map (or, inside a
  `{{range}}` block, the current item).
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

1. `<user override dir>/<agentType>.yaml`
2. built-in `assets/render_templates/<agentType>.yaml`
3. `<user override dir>/default.yaml`
4. built-in `assets/render_templates/default.yaml`

So a user override file that only defines `tool_call` still falls through
to the built-in file (or `default.yaml`) for every other kind — you never
need to copy the whole file just to change one line.

## template4d syntax quick reference

template4d implements a practical subset of Go's `text/template` syntax.
Everything used by these built-in templates:

| Syntax | Meaning |
|---|---|
| `{{.field}}` | Interpolate a scalar value (string/number/bool → its string form). The leading `.` refers to the current view map. |
| `{{if .flag}}...{{end}}` | Render the enclosed block only if `flag` is truthy (`true`, a non-empty string, or a non-empty list). |
| `{{if .flag}}...{{else}}...{{end}}` | Render the first block if truthy, otherwise the `else` block. |
| `{{if .a}}...{{else if .b}}...{{else}}...{{end}}` | Chained conditions, evaluated in order. |
| `{{range .list}}...{{end}}` | If `list` is a non-empty list of maps, render the block once per item, with that item's fields available via `.field` inside (not nested under `list`). |
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

Example uses in this codebase: `{{if eq .Status "done"}}`,
`{{if not .hasContentText}}`, `{{.Name | upper}}`.

### Custom functions

commander additionally registers its own function library
(`lib/render/acp_template_functions.dart`), available in every built-in
and user-override template alongside the built-ins above:

| Function | Signature | Meaning |
|---|---|---|
| `diff` | `diff oldText newText` | Unified-diff-style body (` `/`-`/`+`-prefixed lines) computed via the same LCS algorithm `acp_message_view.dart` uses for the `contentText`/`diffs` view fields. Useful in a user-override template that wants to render a diff itself from raw `oldText`/`newText` fields instead of relying on the precomputed ones. |

If you add a new custom function there, list it in this table too.

### The "prefer contentText over inputJson" pattern

Several kinds (`tool_call`, `tool_call_update`) carry two possible ways to
show a value with a priority order. `{{else if}}` expresses this directly:

```
{{if .hasContentText}}
{{.contentText}}
{{else if .hasInput}}
{{.inputJson}}
{{end}}
```

Read this as: "if there's contentText, show it; otherwise, if there's
input, show that instead."

### Whitespace

Unlike Mustache, template4d does **not** silently strip any newline
following a tag on its own line — every `\n` you write is emitted exactly
as written. If you want to trim whitespace deliberately, use `{{-`/`-}}`
(trims adjacent whitespace on that side), matching Go's own convention;
the built-in templates don't currently need this since there's no
implicit stripping to compensate for.

### Lists

`{{range .options}}...{{end}}` iterates `options`, a list of maps; inside
the block, that item's own fields (`.index`, `.name`, `.optionId`, …) are
available directly via a leading dot, not nested under `options`:

```
{{if .hasOptions}}
Options:{{range .options}}
  {{.index}}. {{.name}}{{end}}
{{end}}
```

## The `kind`s and their view fields

Every view map always has a `kind` field naming which template applies.
Below is every field available per kind. Fields prefixed `has`/booleans
(`isX`) exist purely for use as section conditions.

### `agent_message_chunk`, `user_message_chunk`, `agent_thought_chunk`
Streamed text chunks (assistant reply, user echo, and reasoning/thinking
respectively — same shape, different kind).

- `text` — the chunk's text (empty string if the content block isn't text,
  e.g. an image; see below).
- `hasText` — `text.isNotEmpty`.

Non-text content blocks (`image`, `audio`, `resource`, `resource_link`)
never populate `text` with anything meaningful for images/audio — only
`resource_link` embeds a `[Link: name](uri)` string. If you need to
distinguish block types, that's a Dart change (see below); the flattener
currently only cares about text extraction here.

### `tool_call`
A new tool invocation was started.

- `title` — human-readable description, e.g. `"Read file"`.
- `toolCallId` — the tool call's ACP id.
- `status` — `pending` / `in_progress` / `completed` / `failed`.
- `kindLabel` — the raw ACP `ToolKind` string (`read`, `edit`, `execute`, …).
- `kindIcon` — an emoji for `kindLabel` (📖 read, ✏️ edit, 🗑️ delete, 📦
  move, 🔍 search, ⚡ execute, 🧠 think, 🌐 fetch, 🔀 switch_mode, 🔧
  anything else).
- `inputJson` — pretty-printed `rawInput`, `""` if absent.
- `hasInput` — `inputJson.isNotEmpty`.
- `contentText` — rendered text of any `content` entries attached directly
  to this tool_call (e.g. a diff Claude attaches to its own tool_call
  instead of a later update — see "diff rendering" below).
- `hasContentText` — `contentText.isNotEmpty`.

### `tool_call_update`
A follow-up update to an existing tool call (status change and/or result).

- `toolCallId`, `status` — as above.
- `label` — `title` if the update carries one, else `toolCallId`.
- `kindIcon` — as above, from this update's `kind` if present.
- `completed`, `failed`, `inProgress` — mutually exclusive booleans derived
  from `status` (`inProgress` is true for anything that isn't `completed`
  or `failed`, including `pending`).
- `contentText` — joined text of every `content` entry (see "diff
  rendering" below); if `content` is empty, falls back to `rawOutput`
  (string used verbatim, anything else pretty-printed) so a tool result
  delivered only via `rawOutput` (e.g. Claude's `ToolUseSummary`) is never
  dropped.
- `hasContentText` — `contentText.isNotEmpty`.

#### Diff rendering (shared by `tool_call` and `tool_call_update`)
A `content` entry of `{"type": "diff", "path", "oldText"?, "newText"}`
renders as:
```
[Diff: path]
```
newText
```
```
(or `[New file: path]` when `oldText` is absent — a fresh file, not a
modification). The full new text is fenced, not just a path placeholder —
if you're customizing this, keep in mind `contentText` already contains
the fenced block; don't re-wrap it in your own ` ``` ` in the template.

### `permission_request`
Not a `SessionUpdate` at all — ACP models this as a JSON-RPC method call,
not a session notification, but agents/ tags it with a synthetic `kind` so
it flows through the same batch. This is the one kind whose raw JSON has
no `update` wrapper.

- `title`, `hasTitle` — the tool call's title, if any.
- `kindIcon` — as above, from the tool call's `kind`.
- `inputJson`, `hasInput` — pretty-printed `rawInput`.
- `options` — list of `{index (1-based), optionId, name, kind}`. `name`
  falls back to `optionId` when the option has no display name.
- `hasOptions` — `options.isNotEmpty`.

This is the one kind where showing `options` isn't optional in spirit — a
permission request with no visible options is useless to whoever has to
answer it. Keep a `{{range .options}}` loop in any override.

### `plan`
The agent's full execution plan (replaces any previous plan).

- `entries` — list of `{index (1-based), content, status, statusIcon,
  priority, hasPriority}`.
- `statusIcon` — ⬜ pending, ⏳ in_progress, ✅ completed, • anything else.
- `hasEntries` — `entries.isNotEmpty`.

### `plan_update` *(UNSTABLE ACP capability)*
A content update for a plan by id — can be structured entries, a markdown
blob, or a file reference; exactly one of `hasEntries` / `hasMarkdown` /
`hasUri` will be true (or none, for an empty update).

- `planId`, `hasPlanId`.
- `planType` — `items` / `markdown` / `file`.
- `entries`, `hasEntries` — same shape as `plan` above (no `priority` field
  here).
- `markdown`, `hasMarkdown` — a raw markdown blob to embed verbatim.
- `uri`, `hasUri` — a file reference.

### `plan_removed` *(UNSTABLE ACP capability)*
- `planId`, `hasPlanId`.

### `available_commands_update`
- `commandCount` — total count.
- `commands` — list of `{name, description, hasDescription}`.
- `hasCommands`.

### `current_mode_update`
- `currentModeId` — the new mode id string.

### `config_option_update`
- `configOptions` — list of `{name, type, value, hasValue, description,
  hasDescription}`. Covers both select-style and boolean options
  uniformly (`value` is always the stringified current value).
- `hasConfigOptions`.

### `session_info_update`
- `title`, `hasTitle`.
- `updatedAt`, `hasUpdatedAt` — ISO-ish timestamp string, shown verbatim
  (no date formatting is done for you).

### `usage_update` *(UNSTABLE ACP capability)*
Context-window and cost tracking.

- `used`, `size` — raw integers (context tokens used / window size).
- `usedHuman`, `sizeHuman` — compact form (`999`, `127K`, `1.5M`, …).
- `hasSize` — `size > 0`.
- `pct` — 0–100 integer, only meaningful when `hasPct` is true.
- `hasPct` — `size > 0` (i.e. a percentage could be computed).
- `nearLimit` — `pct >= 90`, handy for a warning color/icon.
- `model` — model name string, if the agent supplied one in `_meta`.
- `inputHuman`, `outputHuman`, `cacheNewHuman`, `cacheReadHuman` — compact
  token counts, preferring the agent's own pre-formatted `_meta` strings
  when present, falling back to formatting the numeric `_meta` fields
  otherwise.

### Any other/future `sessionUpdate` kind
If the ACP protocol adds a new `sessionUpdate` discriminator the flattener
doesn't know about yet, the view map is just `{kind: "<the new kind>"}` —
no extra fields. You can still add a template keyed by that new kind name;
it just won't have anything to interpolate until the Dart flattener is
taught to extract fields for it (see below).

## Fallback behavior ("messages are never silently dropped")

This pipeline is built around one invariant: a message is never silently
swallowed just because a template is missing or a notification is
malformed.

- If a `kind` has no template anywhere in the resolution chain, or its
  template throws while rendering, the engine falls back to that
  notification's own `text` field if it has one, and failing that, a raw
  JSON dump wrapped in a `<pre>` block. It does **not** render nothing.
- If an agent type has no templates at all in any layer (not even
  `default.yaml` — e.g. a corrupted asset bundle), the entire batch
  renders via `lib/render/builtin_fallback.dart`'s hardcoded formatter
  instead of a template, which is output-equivalent to `default.yaml` for
  the well-known kinds and still falls back to a raw dump for anything it
  doesn't recognize.
- Malformed/unparseable notification JSON flattens to `{kind: "unknown",
  raw: <original>}`. There is deliberately no `unknown` template anywhere
  — see "File format" above for why.

When customizing templates, you generally don't need to think about this;
it's what lets you write an incomplete override file without worrying
you've broken anything.

## Testing a template change

1. Edit the YAML in your user override directory (or, if working on the
   built-ins, edit the file here and rebuild commander so the new asset is
   bundled).
2. If you're editing a **user override** file, just save it — commander
   polls the configured override directory every 60s (content hash per
   file, not mtime, so a save-without-changes never triggers a spurious
   reload) and calls `AcpRenderEngine.reloadTemplates` automatically the
   moment a `.yaml`/`.yml` file's content actually changes. No restart, no
   trip through Settings required. See `AcpRenderEngine.startWatching` in
   `lib/render/acp_render_engine.dart` if you need to change the interval
   or watch behavior. Editing a **built-in** asset always needs a rebuild
   — the watch only ever looks at the user override directory.
3. Trigger the relevant ACP update again (e.g. re-run a tool call) and
   check the rendered message. If you don't want to wait up to 60s, the
   Settings → Message Render Templates dialog's Save button still
   force-reloads immediately.

For quick iteration without a live agent, `commander/test/render/` has
unit tests that render a hand-built view map or raw notification JSON
through `AcpRenderEngine`/`flattenAcpNotification` directly — see
`acp_render_engine_test.dart` and `acp_message_view_test.dart` for the
pattern.

## Adding a new view field (Dart-side change)

If a template needs a value that isn't listed above, it has to be added to
the corresponding `_xxxView` function in `lib/render/acp_message_view.dart`
first, or exposed as a custom function in
`lib/render/acp_template_functions.dart` if it's a general-purpose
computation rather than a per-kind field. Steps for a new view field:

1. Find (or add) the `_xxxView(Map update)` function for the `kind`.
2. Add the new key(s) to the returned map, computing whatever the template
   will need to just interpolate/branch on directly (a formatted string, a
   boolean flag, a list of maps for a section).
3. Add/update the corresponding case in `flattenAcpNotification`'s test
   suite (`test/render/acp_message_view_test.dart`) covering the new
   field's presence/absence.
4. Reference the new field from the template(s) that need it.
