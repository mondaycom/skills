# Block operations matrix

Per-block-type guide to which `update_doc` operation to use. Picking the wrong operation is the most common cause of silent doc corruption.

## How to use this file

1. Call `read_docs` with `include_blocks: true` to get the doc's blocks.
2. Find the target block by `id` and look up its `type` field.
3. Match the row below to choose the operation.
4. Compose the operation per the column "Example payload".

## Type naming - input vs output

monday's API uses two different naming conventions for block types:

- **Input** (when you call `update_doc` with `create_block` or `replace_block`): UPPER_SNAKE_CASE, e.g. `LARGE_TITLE`, `BULLETED_LIST`, `INFO`.
- **Output** (when you read blocks from `read_docs`): lowercase with spaces, e.g. `large title`, `bulleted list`, `notice box`.

When matching a read block to a row in this table, normalize on the lowercase output form.

## Routing matrix

| Output type | Subtypes | Allowed ops | Disallowed | Notes |
|---|---|---|---|---|
| `large title`, `medium title`, `small title`, `normal text`, `quote` | text block subtypes (LARGE_TITLE / MEDIUM_TITLE / SMALL_TITLE / NORMAL_TEXT / QUOTE) | `update_block` (content), `replace_block` (subtype change), `delete_block`, `add_comment` | Cannot change subtype with `update_block` | Use `block_content_type: "text"` in update_block |
| `code` | language-tagged code block | `update_block`, `replace_block`, `delete_block`, `add_comment` | - | Use `block_content_type: "code"` in update_block; pass `language` to update or change |
| `bulleted list`, `numbered list`, `check list` | list_item subtypes (BULLETED_LIST / NUMBERED_LIST / CHECK_LIST) | `update_block` (content), `replace_block` (variant change), `delete_block`, `add_comment` | Cannot change list variant with `update_block` | Use `block_content_type: "list_item"` in update_block; pass `checked: bool` for CHECK_LIST items; `indentation: 0..N` for nesting |
| `divider` | - | `replace_block`, `delete_block` | `update_block` (no mutable properties), `add_comment` | Replace with another divider is a no-op visually; rarely needed |
| `page break` | - | `delete_block` | `update_block`, `replace_block`, `add_comment` | Print-layout only; treat as decorative |
| `table` | row_count, column_count, column widths | `replace_block`, `delete_block` | `update_block` (cell content not addressable), `add_comment` | See "Tables" below |
| `cell` (table cell) | - | `delete_block` only via parent table replacement | `update_block`, `replace_block`, `create_block` with parent_block_id, `add_comment` | Cells are pseudo-blocks; cannot be mutated directly |
| `image` | public_url or asset_id | `replace_block` (swap source), `delete_block` | `update_block`, `add_comment` | URL / asset_id immutable after creation |
| `video` | raw_url | `replace_block` (swap source), `delete_block` | `update_block`, `add_comment` | URL immutable after creation |
| `notice box` | theme: INFO / TIPS / WARNING / GENERAL | `replace_block` (theme change), `delete_block`, `create_block` with parent_block_id (in a SEPARATE later call) | `update_block`, `add_comment` directly on the notice_box itself | Theme immutable; content lives in child blocks. See "Notice boxes" below |
| `layout` | column_count: 2-6, column_style widths | `replace_block`, `delete_block` | `update_block`, `create_block` with parent_block_id, `add_comment` | Cell content not populatable via API at all. See "Layouts" below |
| `BOARD` embed | - | `delete_block` only | `update_block`, `replace_block`, `create_block`, `add_comment` | No public API to create or modify; must be added from the monday UI |
| `WIDGET` embed | - | `delete_block` only | `update_block`, `replace_block`, `create_block`, `add_comment` | No public API to create or modify |
| `DOC` embed | - | `delete_block` only | `update_block`, `replace_block`, `create_block`, `add_comment` | No public API to create or modify |
| `GIPHY` | - | `delete_block` only | `update_block`, `replace_block`, `create_block`, `add_comment` | No public API to create or modify |

## update_block - the in-place edit path

Use for content edits on text, code, and list_item blocks where the subtype is staying the same.

```json
{
  "operation_type": "update_block",
  "block_id": "<id from read_docs>",
  "content": {
    "block_content_type": "text",
    "delta_format": [
      { "insert": { "text": "New paragraph content. " } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

Variants:

- `block_content_type: "code"` - pass `language` to change the highlight (`"python"`, `"javascript"`, etc).
- `block_content_type: "list_item"` - pass `indentation` (0 to N) for nesting; pass `checked` (bool) for CHECK_LIST items.

`update_block` cannot change the block subtype. To go from a paragraph to an H2, use `replace_block`.

## replace_block - delete + create in one slot

Use when:

- The block has no mutable properties (`divider`, `page break`).
- The block is structurally immutable (table, image source, video source, notice_box theme, layout).
- The user wants to change a text block's subtype (paragraph -> H1) or list variant (bulleted -> numbered).

```json
{
  "operation_type": "replace_block",
  "block_id": "<id of block to remove>",
  "block": {
    "block_type": "text",
    "text_block_type": "MEDIUM_TITLE",
    "delta_format": [
      { "insert": { "text": "New H2 content" } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

Note: `replace_block` is delete + create. Comments anchored to the old block are lost. Children of the old block (notice_box children, table cells) are NOT moved to the new block; they become orphaned blocks.

## create_block - insert anywhere

Use to add a new block at a precise position.

- `after_block_id`: the block this new block follows. Omit to append at the end of the doc.
- `parent_block_id`: only works for nesting inside a `notice_box`. Cannot be used for table cells or layout cells.

```json
{
  "operation_type": "create_block",
  "after_block_id": "<id of preceding block>",
  "block": {
    "block_type": "notice_box",
    "theme": "WARNING"
  }
}
```

Then in a SEPARATE call:

```json
{
  "operation_type": "create_block",
  "parent_block_id": "<notice_box id from previous call>",
  "block": {
    "block_type": "text",
    "text_block_type": "NORMAL_TEXT",
    "delta_format": [
      { "insert": { "text": "Watch out for X." } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

## delete_block - the only path for embed types

Works for every block type. The only available operation for `BOARD`, `WIDGET`, `DOC` embed, and `GIPHY` blocks.

```json
{
  "operation_type": "delete_block",
  "block_id": "<id>"
}
```

Deleting a parent block (notice_box, table, layout) does NOT delete its children. Delete children first if you want a clean removal.

## Tables

Tables are pre-populated only via markdown:

- **Initial creation:** include the table in the markdown body of `create_doc`.
- **After creation:** `add_markdown_content` with a markdown table appends a new pre-filled table.
- **Editing existing cells:** not supported by the API. `replace_block` swaps the entire table; the new one can be populated only by also using `add_markdown_content` (delete the old table, then append a new markdown table).

If the user asks to "edit cell B2 of my table", explain the constraint and offer:

- Rewrite the entire table via markdown (best for small tables).
- Direct the user to edit in the monday UI (best for large tables).

## Notice boxes

A notice_box must be created in two `update_doc` calls when it has children. The first call creates the container; the second references its ID via `parent_block_id`.

```
Call 1:  create_block notice_box { theme: "INFO" }
         -> returns block ID X

Call 2:  create_block { parent_block_id: X, block: { block_type: "text", ... } }
```

A notice_box created in the same call as its child cannot be referenced - the API rejects forward references to IDs created within the same operations array.

To change a notice_box's theme:

- `replace_block` with the new theme. Children are NOT carried over - they become orphaned.
- Recreate children after the replace, in a separate later call (the new notice_box ID is only available after the first call returns).

## Layouts

Layouts are multi-column shells. The API can create them empty (2-6 columns, with column widths), but **cannot populate cell content**. The `parent_block_id` field rejects layout block IDs.

When the user asks for a layout with content:

- Create the layout shell via `create_block`.
- Tell the user the cells must be filled from the monday UI.
- Or, fall back to a different shape: a heading + sub-headings (visually similar in many cases), or a single-column with separators.

There is no markdown shortcut for layout cell content.

## Block subtypes - input enums

Text block subtypes (`text_block_type`):

- `LARGE_TITLE` - rendered as H1
- `MEDIUM_TITLE` - rendered as H2
- `SMALL_TITLE` - rendered as H3
- `NORMAL_TEXT` - paragraph
- `QUOTE` - blockquote

List block subtypes (`list_block_type`):

- `BULLETED_LIST` - default
- `NUMBERED_LIST` - auto-numbered
- `CHECK_LIST` - supports `checked: true | false` per item

Notice box themes (`theme`):

- `INFO` - blue, informational
- `TIPS` - green, helpful hint
- `WARNING` - red / orange, watch-out
- `GENERAL` - neutral

## Quick reference - common edits

| User intent | Operation | Notes |
|---|---|---|
| Append a section to the end | `add_markdown_content` (markdown body) | No block IDs needed |
| Edit one paragraph in place | `update_block` (text) | Re-read first to get block_id |
| Promote a paragraph to H2 | `replace_block` (text, MEDIUM_TITLE) | Subtype change requires replace |
| Switch a bulleted list to numbered | `replace_block` per item (list_item, NUMBERED_LIST) | One replace per item |
| Tag a user inline | `update_block` with mention insert in delta_format | See delta-format-cheatsheet.md |
| Embed a live status from a board | `update_block` with column_value insert in delta_format | See delta-format-cheatsheet.md |
| Add a callout box with content | Two `create_block` calls (notice_box, then child) | Cannot inline in one call |
| Change a callout box theme | `replace_block` (notice_box) + recreate children | Theme immutable; children orphaned by replace |
| Swap an image source | `replace_block` (image) | URL / asset_id immutable |
| Add a Slack-style threaded comment | `add_comment` with `parent_update_id` | parent_update_id from include_comments |
| Highlight a specific phrase with a comment | `add_comment` with `block_id` + `selection_from` + `selection_length` | Text-only blocks |
| Remove an embedded board | `delete_block` (BOARD type) | No way to create or update embeds |
| Rename the doc | `set_name` | One operation, separate from block ops |
