---
name: docs-collaborator
description: |
  Creates, reads, edits, and comments on monday.com docs - the rich-content
  surface inside monday for meeting notes, RFCs, runbooks, weekly updates,
  postmortems, and any work-adjacent writing. Goes beyond markdown shuffling
  by embedding live board column values inline (so the doc reflects real-time
  status), tagging users / docs / boards via mentions, scaffolding common doc
  shapes, and routing each edit through the correct update_doc operation.
  Use when user asks to "create a doc", "make a monday doc", "draft meeting
  notes", "write an RFC", "scaffold a runbook", "summarize a meeting in a
  doc", "add a section to my doc", "edit my doc", "update the doc",
  "comment on a doc", "tag someone in a doc", "embed a status in a doc",
  "show live board data in a doc", "rename a doc", or "show me my docs".
license: MIT
metadata:
  author: monday.com
  version: "0.2.0"
---

# Docs Collaborator

Create, read, edit, and comment on monday.com docs. **The skill is a capability surface, not a template engine.** Every doc is a stream of typed blocks (headings, paragraphs, lists, tables, code, dividers, notice boxes, layouts, embedded boards / widgets / docs / images / videos / GIFs). The named shapes listed below are starting points - compose and extend freely. The differentiators (live column-value embeds, inline mentions, notice boxes, layouts) apply to any shape, named or invented.

The platform-native value over a raw markdown editor lives in three things most builders miss: inline mentions of users / docs / boards, **live column-value embeds** that reflect the current state of a board item, and a closed-loop edit pattern that prevents corrupting the doc's block graph.

## Core constraint - read this first

**Never edit a block you have not read.** monday's update_doc API is block-ID-based, and block IDs are only knowable through `read_docs` with `include_blocks: true`. Skipping the read leads to operations referencing IDs that do not exist (silent failure) or - worse - reusing an ID from another doc. The closed-loop pattern is non-negotiable:

1. `read_docs` with `include_blocks: true` to get block IDs, types, and positions.
2. Identify the target block(s) by type and position or by content.
3. `update_doc` with the operation that matches the target block's type (see the routing rule below).

## Block operation routing - the non-obvious part

Each block type has a different set of allowed mutations. Picking the wrong operation is the #1 cause of "the call succeeded but the doc looks wrong" bugs.

| Target block type | Use this operation | Why |
|---|---|---|
| text (any subtype: NORMAL_TEXT, LARGE_TITLE, MEDIUM_TITLE, SMALL_TITLE, QUOTE) | `update_block` for content edits; `replace_block` to change the subtype (e.g. promote a paragraph to H2) | Subtype is set on creation; `update_block` cannot change it |
| code | `update_block` | Content and language editable in place |
| list_item (BULLETED_LIST, NUMBERED_LIST, CHECK_LIST) | `update_block` for content edits; `replace_block` to switch between bulleted / numbered / check | List variant is set on creation |
| divider | `replace_block` only | No mutable properties |
| table | `replace_block` only | Row / column count and column widths immutable; table cell content not addressable via API after creation - see "Tables" in `references/rich-content.md` |
| image | `replace_block` to swap source | URL / asset_id immutable after creation |
| video | `replace_block` to swap source | raw_url immutable after creation |
| notice_box | `replace_block` to change theme | theme immutable; content lives in child blocks (see "Notice boxes" in `references/rich-content.md`) |
| layout | `replace_block` only | Cell content cannot be populated via API at all - see "Layouts" in `references/rich-content.md` |
| BOARD embed, WIDGET embed, DOC embed, GIPHY | `delete_block` only | No public API to create or modify these block types |

**Rendering quirk - list_item + mention via `update_block`.** When adding an inline mention to a list_item that previously contained only text, `update_block` may save the mention blot correctly (data confirms) but some clients can fail to render the pill (text-prefix-before-mention case). Workaround: use `replace_block` with a fresh list_item containing the same delta. Same pattern in a `normal_text` block renders correctly; the quirk appears specific to list_item updates.

Full per-block-type detail with worked examples lives in `references/block-operations-matrix.md`. Read it before authoring any non-trivial structural edit.

## Workflow: Create a doc

### Step 1: Decide the location

monday docs live in one of two places:

- **Workspace** (most common). Requires `workspace_id`. Optional `doc_kind`: `public` (default), `private` (only the creator), `share` (specific people). Optional `folder_id` to nest under a folder.
- **Item-attached**. The doc lives in a doc-type column on a specific board item. Requires `item_id`. Optional `column_id` if the user picked a specific doc column; otherwise the tool auto-creates one.

**Cell-limit preflight for item-attached docs.** Doc-type cells hold one doc per cell - hard API limit. Before calling `create_doc location=item`, preflight the target column via `get_board_items_page` with `columnIds: [<doc_col_id>]` to check whether the cell is already populated. If populated, do NOT attempt the create - it will fail with `CellLimitExceededException`. Fall back to **creating a workspace-level doc and adding a bi-directional DOC mention cross-link** between the new doc and the existing attached doc. Never auto-detach the existing doc.

If the user does not name a workspace, fall back to the user's primary workspace via `get_user_context`. Ask once if it's still ambiguous; do not guess across multiple workspaces.

### Step 2: Plan-tier guard before creating a 4th doc

**Free and Basic plans cap at 3 docs per account.** Before calling `create_doc` on a Free / Basic account, check whether this would be the 4th doc. There is no direct doc-count tool, so the practical guard is: if the user has hit this limit before or asks "can I create more than 3 docs?", warn them - upgrade to Standard+ removes the cap. Treat plan-tier as an open question rather than blocking the create call; the platform itself will reject if the cap is hit.

### Step 3: Compose the body - shapes are examples, not boundaries

`create_doc` takes a `markdown` parameter that is parsed into blocks on import. This is the most efficient way to scaffold a doc in one call. Reach for one of these common shapes as a starting point when the user names a use case, then extend freely:

- **Meeting notes:** title, date, attendees (with `@mentions`), agenda bullets, decisions, action items table, follow-ups.
- **Weekly update:** highlights, key metrics, risks, next-week plan.
- **RFC:** problem, proposed solution, alternatives considered, open questions, decision log.
- **Runbook:** trigger, prerequisites, numbered procedure, rollback steps, escalation path.
- **Postmortem:** incident summary, timeline, root cause, what went well, what did not, action items.

When the user invents a shape ("partner enablement page", "AI readiness snapshot", "Q3 onboarding context"), compose with judgment - sections, tables, notice boxes, embeds - rather than forcing a named template. The differentiators apply to any shape.

**Passive vs active mode.** By default, scaffold with visible slot markers (`_[embed: ...]_`, `_[mention: ...]_`) where embeds and mentions belong. Switch to active mode - wiring real embeds and mentions - when the user refers to items, boards or people by name; resolve names to IDs yourself (if a name is ambiguous, ask once with the top 3 candidates) and state what you picked. Read `references/create-modes.md` before wiring anything: it has the marker formats, escalation triggers, name-to-ID resolution chain and the communication contract.

Markdown notes that matter:

- Markdown headings (`#`, `##`, `###`) map to LARGE_TITLE / MEDIUM_TITLE / SMALL_TITLE blocks.
- Markdown tables work and the cells will be populated with the values you put in. **This is the only way to get pre-filled table content.** Cell-level API population after creation is not supported.
- Markdown images and links work. For asset-based images (uploaded files in monday), use `update_doc` with `create_block` and `block_type: "image"` plus `asset_id` after creation - markdown cannot reference assets.
- Markdown does not have notice_box, layout, embed, divider-with-spacing, mention, or live column-value syntax. Add those via `update_doc` after the doc is created.

### Step 4: Call `create_doc`

```
create_doc({
  doc_name: "<title>",
  location: "workspace",
  workspace_id: <id>,
  doc_kind: "public" | "private" | "share",  // optional
  folder_id: <id>,                            // optional
  markdown: "<full markdown body>"
})
```

For item-attached (after preflight):

```
create_doc({
  doc_name: "<title>",
  location: "item",
  item_id: <id>,
  column_id: "<doc_column_id>",  // optional
  markdown: "<full markdown body>"
})
```

Capture the returned `doc_id`, `object_id`, and `doc_url`. The user wants the URL to share or open; the `doc_id` is the primary key for every subsequent edit.

### Step 5: Layer in the rich-content extras

Mentions, live column-value embeds, notice boxes, layouts, and asset images all need a follow-up `update_doc` call. In active mode, wire them now using the resolved IDs from Step 3. In passive mode, leave the slot markers in place - the user fills slots later by naming items. See "Common patterns" in `references/delta-format-cheatsheet.md`.

### Step 6: Confirm and suggest next steps

Plain-language confirmation: name, workspace, doc URL. State the mode used (passive scaffold or active). Then 2-3 follow-ups appropriate to the shape:

- "Want me to tag the attendees with @mentions?"
- "Want me to embed live status from the project board?"
- "Should I add a notice box at the top with the meeting outcome?"
- "Want me to share the doc URL in a Slack channel?"

## Workflow: Read a doc

### Step 1: Resolve the doc identifier

Three ways the user might point at a doc:

- **doc_id** (e.g., `41302579`) - the API's primary key, returned by `create_doc` and `read_docs`.
- **object_id** (e.g., `18411484379`) - the numeric segment of the doc URL: `https://monday.monday.com/docs/{object_id}`.
- **doc name or fuzzy reference** ("the Q3 retro doc") - resolve via `read_docs` with `type: "workspace_ids"` to list docs in a workspace, then match by name.

`read_docs` is forgiving: if you call it with `type: "ids"` and the value is actually an `object_id`, it auto-retries as `object_ids`. Lean on this rather than guessing.

### Step 2: Decide what to read

Two modes:

- **content** (default) - returns the doc's blocks and / or markdown projection.
  - `include_blocks: true` is **mandatory** if any edit will follow. Do not omit it.
  - `include_comments: true` to fetch all comments and replies. Use `comments_limit` to cap (default 50).
- **version_history** - returns the doc's edit history (restoring points). Pass the `object_id`, not the `id`. Set `version_history_limit` for "last N changes" requests; set `include_diff: true` to see what changed (slower).

### Step 3: Render with resolved mentions and embeds, not placeholders

The API's `blocks_as_markdown` projection renders mentions as `@{USER-XXX}` placeholders and live column-value embeds as empty space. **This is not the right rendering for a "show me what's in this doc" prompt** - the user sees pills and live chips when they open the doc; the textual rendering should mirror that.

For "show me / what's in / read this doc" prompts: resolve every mention to a display name, resolve embeds to their current values when cheap, and add an enumeration table of inline non-text elements with their IDs. Read `references/read-rendering.md` for the exact rules.

Do not use `show-table` for doc content - the doc itself is the UI. Render in chat with markdown.

## Workflow: Edit a doc

### Step 1: Read first

`read_docs` with `include_blocks: true`. This is the closed-loop precondition. Capture every block's `id`, `type`, and position.

### Step 2: Pick the right operation

Reach for the simplest one that does the job:

- **Add new content at the end:** `add_markdown_content` (no block IDs needed; supports text, lists, tables, code).
- **Add asset-based images:** `create_block` with `block_type: "image"` and `asset_id` (markdown does not support asset images).
- **Edit existing text / code / list_item content in place:** `update_block` with the matching `block_content_type` (`text`, `code`, or `list_item`).
- **Change a block's subtype** (paragraph -> H2, bulleted -> numbered, INFO notice -> WARNING notice): `replace_block`. Subtypes are set on creation only.
- **Insert a block at a precise position:** `create_block` with `after_block_id` set. There is no `before_block_id` operation - to insert at the very top of a doc, the only path is to insert after the doc's first block.
- **Delete:** `delete_block`. The only path for BOARD / WIDGET / DOC embed / GIPHY blocks.
- **Comment:** `add_comment` - HTML body; tag people via `mentions_list`, not `@` in the body (see "Comments" in `references/rich-content.md`).
- **Rename the doc:** `set_name`.

The full type-to-operation routing lives in `references/block-operations-matrix.md`.

### Step 3: Compose the delta_format (for text / code / list_item)

`update_block` and `create_block` for content blocks take a `delta_format` array of insert ops. Three rules govern every delta:

1. The last op MUST be `{insert: {text: "\n"}}`. Always. No exceptions.
2. Plain text uses `{insert: {text: "..."}}` with optional `attributes` (bold, italic, underline, strike, code, link, color, background).
3. Two non-text inserts unlock the differentiators:
   - Mention: `{insert: {mention: {id: <numeric-id>, type: "USER" | "DOC" | "BOARD"}}}`. Resolve user IDs via `list_users_and_teams` (always pass `name` to scope the call).
   - Live column-value embed: `{insert: {column_value: {item_id: <id>, column_id: "<column_id>"}}}`. Renders the current value of that column on that item, live, every time the doc is opened.

Worked examples and the full attribute table live in `references/delta-format-cheatsheet.md`.

### Step 4: Batch operations carefully

`update_doc` accepts up to 25 operations in one call, executed in order, stopping at the first failure. Four gotchas:

- **Notice box nesting requires two calls.** A notice_box created in the same call cannot be referenced by `parent_block_id` in a later operation in the same call. Create the notice_box, capture the returned ID, then make a second `update_doc` call to nest children.
- **Position dependencies.** If operation B references the position of a block created by operation A, prefer two calls. Failures partway through the array leave the doc in a partial state.
- **Same-block conflict detection.** When multiple operations in the same call (or the same user turn) target the same `block_id`, surface the conflict to the user before executing - "last write wins" is the observed behavior. Example: `update_block` adds a mention to paragraph X, then a later `update_block` replaces paragraph X entirely. The second wins; the mention is lost.
- **Comment anchors live in the delta.** When a block has anchored comments, the read response shows `attributes.comments: [<update_id>, ...]` on the relevant text insert. `update_block` replaces the entire delta_format, so a clean rewrite detaches the comment from the block (the comment itself survives on the backing item but loses its block link). To preserve anchored comments through an edit, include the `comments` attribute on the matching text insert in the new delta.

### Step 5: Verify and confirm

After a non-trivial edit, re-read with `include_blocks: true` to confirm the blocks landed as intended. Plain-language confirmation: what changed, the doc URL. Do not narrate the API calls.

## Rich content: tables, notice boxes, layouts, mentions, live embeds, comments

Live column-value embeds are the differentiator: they keep a doc in sync with real board data. Before adding any of these elements, read `references/rich-content.md`. It covers table constraints, notice-box nesting, layouts, mention types, live column-value embeds and comments.

## Smart defaults

When the user does not specify all values:

- **Workspace**: the user's primary workspace via `get_user_context`. State the default. Ask once if multiple workspaces are plausible.
- **doc_kind**: `public` (the platform default).
- **Folder**: none (root of the workspace).
- **Doc shape**: if the user names a use case ("meeting notes", "RFC", "weekly update"), scaffold the matching shape as a starting point and extend per the user's specific asks. If the user invents a shape, compose with judgment - no template needed.
- **Mode**: passive scaffold (slot markers) by default. Escalate to active mode (real embeds / mentions / cross-links) only when name-based intent signals are present.
- **Embeds and mentions in active mode**: state the resolved choices and reasoning before wiring. Offer override.

State every default in the response so the user can override.

## Validate preconditions before writing

- **Doc exists:** `read_docs` succeeded for the target ID before any write.
- **Block IDs are real:** every `block_id` in the operations array came from the most recent `read_docs` call.
- **User IDs for mentions:** resolved via `list_users_and_teams` (always with a `name` or `userIds` filter).
- **Item IDs and column IDs for embeds:** `get_board_info` and `get_board_items_page` confirm both before the write. Status column IDs are always board-specific (e.g. `color_mm2x25ey`), never the generic literal `"status"`.
- **Workspace exists:** `list_workspaces` if the user named one but did not provide an ID.
- **Cell capacity for item-attached docs:** preflight the target doc column via `get_board_items_page` before `create_doc location=item` (see Step 1).
- **Empty-data fallback for active mode:** when the user's intent references a column with no values set (e.g. "top priorities" but the Priority column is entirely null), do NOT silently pick. Detect the empty state, disclose to the user ("no Priority values set on any items"), offer a fallback heuristic with explicit reasoning ("picking by Status + Due Date proximity instead"), and offer override.

## Duplicate detection before writing

**Before any `create_block`, `add_markdown_content`, `add_comment`, or content-replacing `update_block`, compare the proposed content against existing blocks in the doc.** Default to skip + report on match; surface options to the user; never silently duplicate.

Match heuristics by block type:

| Block type | Match heuristic |
|---|---|
| text / heading / quote / code | exact delta text + same subtype |
| list_item | exact text + same list type + adjacency (multi-bullet sections matched as a group) |
| notice_box | same theme + same child text |
| table | same dimensions + same cell content |
| mention | same user_id at same block position |
| column_value embed | same item_id + column_id at same block position |
| comment | same body text + same anchor (excluded by default - see below) |

Skill behavior on match found:

1. **Default: skip + report.** "X already exists at <position>. Skipping to avoid duplicate."
2. **Surface options** (do not silently pick):
   - **(a) skip** - keep existing, do nothing
   - **(b) duplicate** - proceed as requested, accept redundancy
   - **(c) replace** - delete existing, create fresh
   - **(d) move** - offered only when the new request has explicit positional intent ("at the top / at the bottom / after X") that conflicts with the existing match's position

Exclusions where duplication is normal and detection should NOT trigger:

- **Comments are excluded** except for a short stutter window. Same comment body submitted to the same scope (doc or block) within 60 seconds is almost certainly a double-submit / network retry; beyond that, treat as legitimate (legitimate re-review, sentiment echo).
- Adjacent same-style blocks in a list - intended structure, not duplication.
- Repeating section names in different parents (e.g. two "Action Items" H3s under different H2s).

Apply this duplicate-detection rule to every write path in the skill.

## Surface errors with remediation

When a call fails, tell the user what went wrong and how to fix it. The catalog of common failures and their remediation (malformed or stale block IDs, schema mismatches, nesting in one call, cell limits, blank embeds, plan limits, unsupported block types, orphan comment anchors) is in `references/errors.md`.

## Be conservative on destructive operations

Require explicit confirmation for:

- Deleting 3+ blocks in one call.
- Deleting a block that is the only content in its parent (deletes the parent's only content; the parent stays as an empty shell).
- Deleting a notice_box, table, or layout - children are NOT auto-deleted; they become orphaned. Either delete children first or accept the orphan state.
- Bulk replace_block operations (each replace is a delete + create; non-reversible).
- Renaming a doc that has live external references (Slack messages, other docs, item updates).

Single block edits, single block creates, and single comments need no extra confirmation - the user already asked.

## When to use MCP UI widgets

The doc is the UI. Do not render `show-table` or `show-chart` to display doc content. Plain-text confirmations and markdown summaries are the right output.

## Plain language over raw JSON

Never tell the user "I called `read_docs` then `update_doc` with an `update_block` operation." Tell them what changed in human terms: "Updated the second paragraph of your runbook with the new escalation contact." Always include the doc URL.

## Tools used

- `get_user_context` - resolve the user's primary workspace
- `list_workspaces` - find a workspace by name when not provided
- `create_doc` - create a doc in a workspace or attached to an item; markdown body imported as blocks on creation
- `read_docs` - fetch one or more docs (content mode) or version history; `include_blocks: true` is mandatory before any edit; `include_comments: true` for comment threads
- `update_doc` - sequential operations: `set_name`, `add_markdown_content`, `update_block`, `create_block`, `delete_block`, `replace_block`, `add_comment`
- `list_users_and_teams` - resolve user IDs for mentions and `mentions_list` (always pass `name` or `userIds` to scope)
- `get_board_info` - validate column IDs before writing live column-value embeds; preflight board structure for active-mode embed wiring
- `get_board_items_page` - validate item IDs before writing live column-value embeds; preflight doc-column cells for item-attached create; resolve item names to IDs in active mode
- `search` - fuzzy-resolve a doc by name or a board by name in active mode

## Additional resources

- **`references/block-operations-matrix.md`** - per-block-type allowed operations, gotchas, and example payloads.
- **`references/create-modes.md`** - passive-scaffold slot markers, active-mode triggers, name-to-ID resolution and the communication contract.
- **`references/rich-content.md`** - tables, notice boxes, layouts, mentions, live column-value embeds and comments.
- **`references/read-rendering.md`** - how to render mentions, embeds and inline elements when reading a doc.
- **`references/errors.md`** - common failures and how to remediate them.
- **`references/delta-format-cheatsheet.md`** - delta_format insert ops, attributes, mentions, column_value embeds, the terminator rule, common patterns, and the passive-mode slot marker format.
