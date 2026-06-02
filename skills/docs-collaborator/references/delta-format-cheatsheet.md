# Delta format cheatsheet

`delta_format` is the array of insert operations that defines a text, code, or list_item block's content. It is required by `update_block`, `create_block` (for content blocks), and `replace_block` (for content blocks).

## The three rules

1. **Always end with `{insert: {text: "\n"}}`.** No exceptions. The terminator is mandatory; the API rejects deltas without it.
2. **Each operation is one insert.** No update / delete / retain ops. To change content, send a fresh delta_format and the API replaces the entire block content.
3. **Three insert types:**
   - `{insert: {text: "..."}}` - plain or styled text.
   - `{insert: {mention: {id, type}}}` - inline mention of a user, doc, or board.
   - `{insert: {column_value: {item_id, column_id}}}` - live embed of a board column's current value.

## Text inserts and attributes

```json
{ "insert": { "text": "Hello world" } }
```

Add formatting via `attributes`:

```json
{ "insert": { "text": "bold text" }, "attributes": { "bold": true } }
```

Supported attributes (all optional, all apply to text inserts only):

| Attribute | Type | Notes |
|---|---|---|
| `bold` | bool | - |
| `italic` | bool | - |
| `underline` | bool | - |
| `strike` | bool | strikethrough |
| `code` | bool | inline code style (not a code block) |
| `link` | string | URL; the text is the link label |
| `color` | string | text color (CSS-style or named) |
| `background` | string | background highlight color |
| `comments` | array of update IDs | anchors comments to this text insert. Preserve when editing blocks that have existing comments - clean rewrites detach comments. |

Combine attributes:

```json
{
  "insert": { "text": "important link" },
  "attributes": { "bold": true, "link": "https://example.com" }
}
```

## Mentions

Tag a user, doc, or board inline. Mention inserts ignore `attributes` (do not bold, italicize, or color them).

```json
{ "insert": { "mention": { "id": 38288909, "type": "USER" } } }
```

Three mention types:

| `type` | What `id` means | How to resolve |
|---|---|---|
| `USER` | numeric user ID | `list_users_and_teams` with `name: "<display name>"` |
| `DOC` | doc's `object_id` | `read_docs` to find the target doc, use the `object_id` field |
| `BOARD` | numeric board ID | `search` with `searchType: "BOARD"`, or the user provides it |

Always pass `name` (or `userIds`) to `list_users_and_teams`. The unfiltered call returns the entire account user list and breaks token limits on real tenants.

Example - "Hey @daniel, the @Q3-Plan doc is ready":

```json
[
  { "insert": { "text": "Hey " } },
  { "insert": { "mention": { "id": 48202303, "type": "USER" } } },
  { "insert": { "text": ", the " } },
  { "insert": { "mention": { "id": 18234567890, "type": "DOC" } } },
  { "insert": { "text": " doc is ready." } },
  { "insert": { "text": "\n" } }
]
```

## Live column-value embeds - the differentiator

Embed the current value of a board column on a specific item. Renders live every time the doc is opened. Enables dashboards-as-paragraphs.

```json
{ "insert": { "column_value": { "item_id": 11890636850, "column_id": "color_mm2x25ey" } } }
```

`item_id` accepts string or number. `column_id` is a string (e.g. `"status"`, `"date4"`, `"color_mm2x25ey"`).

Resolution rules:

- Validate `item_id` exists via `get_board_items_page` or `get_full_board_data`.
- Validate `column_id` matches a real column on the item's board via `get_board_info`.
- Wrong column IDs render as empty space - silent failure.
- The viewer's permissions on the source board apply. Private boards mean private embeds.

Example - status report paragraph with live status:

```json
[
  { "insert": { "text": "Q3 plan is currently " } },
  { "insert": { "column_value": { "item_id": 11890636850, "column_id": "color_mm2x25ey" } } },
  { "insert": { "text": ", owned by " } },
  { "insert": { "column_value": { "item_id": 11890636850, "column_id": "multiple_person_mm2x4gzh" } } },
  { "insert": { "text": ", due " } },
  { "insert": { "column_value": { "item_id": 11890636850, "column_id": "date_mm2x10qd" } } },
  { "insert": { "text": "." } },
  { "insert": { "text": "\n" } }
]
```

When the doc is opened, the embeds render as the live values. When the item's status changes, the doc reflects it without an edit.

## Internal storage note

When you read a doc back, column_value embeds appear in storage as `macro` blots:

```json
{
  "insert": {
    "macro": {
      "type": "COLUMN_VALUE",
      "macroId": "...",
      "macroData": { "itemId": 11890636850, "columnId": "color_mm2x25ey" }
    }
  }
}
```

This is the on-disk representation. Use the input form (`{insert: {column_value: ...}}`) when writing - the API normalizes it. When reading, expect the macro form in the blocks array. The `blocks_as_markdown` projection does NOT render column_value embeds (they appear as empty space in the markdown view); inspect the blocks array directly to find them.

## Code blocks

Code blocks use `block_content_type: "code"` and accept a `language` field (top-level on the content object, not inside the delta).

```json
{
  "operation_type": "update_block",
  "block_id": "<id>",
  "content": {
    "block_content_type": "code",
    "language": "python",
    "delta_format": [
      { "insert": { "text": "def hello():\n    print(\"hi\")" } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

Code block content does not parse mention or column_value inserts as embeds - they render as literal text. Keep code blocks plain.

## List items

List items use `block_content_type: "list_item"` with optional `indentation` (0 to N) and, for CHECK_LIST items, `checked` (bool).

```json
{
  "operation_type": "update_block",
  "block_id": "<id>",
  "content": {
    "block_content_type": "list_item",
    "indentation": 1,
    "delta_format": [
      { "insert": { "text": "Nested bullet content" } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

To toggle a checkbox:

```json
{
  "operation_type": "update_block",
  "block_id": "<id>",
  "content": {
    "block_content_type": "list_item",
    "checked": true,
    "delta_format": [
      { "insert": { "text": "Done item" } },
      { "insert": { "text": "\n" } }
    ]
  }
}
```

## Common patterns

### Plain paragraph

```json
[
  { "insert": { "text": "Plain text content." } },
  { "insert": { "text": "\n" } }
]
```

### Bold word in a sentence

```json
[
  { "insert": { "text": "This is " } },
  { "insert": { "text": "important" }, "attributes": { "bold": true } },
  { "insert": { "text": " context." } },
  { "insert": { "text": "\n" } }
]
```

### Inline link

```json
[
  { "insert": { "text": "See the " } },
  { "insert": { "text": "design doc" }, "attributes": { "link": "https://monday.monday.com/docs/123456" } },
  { "insert": { "text": " for details." } },
  { "insert": { "text": "\n" } }
]
```

### Mention + live status in one line

```json
[
  { "insert": { "mention": { "id": 38288909, "type": "USER" } } },
  { "insert": { "text": " is owning, current status: " } },
  { "insert": { "column_value": { "item_id": 11890636850, "column_id": "color_mm2x25ey" } } },
  { "insert": { "text": "\n" } }
]
```

### Highlighted phrase with strike-through

```json
[
  { "insert": { "text": "Old plan: " } },
  { "insert": { "text": "ship Q2" }, "attributes": { "strike": true } },
  { "insert": { "text": ". New plan: ship Q3." } },
  { "insert": { "text": "\n" } }
]
```

### Color-coded callout phrase

```json
[
  { "insert": { "text": "Status: " } },
  { "insert": { "text": "BLOCKED" }, "attributes": { "bold": true, "color": "#d83a52", "background": "#ffe0e6" } },
  { "insert": { "text": "\n" } }
]
```

## Passive-mode slot markers - the scaffold contract

When the skill creates a doc in **passive scaffold mode** (the default, before user names specific items / users), reserve visible slots for embeds and mentions using italic placeholder markers in the markdown body. Slots telegraph platform shape without inserting broken embeds.

| Slot type | Marker format | Example use in a delta |
|---|---|---|
| Live column-value embed | `_[embed: <column> of <item-ref>]_` | `_[embed: status of <top priority item>]_` |
| User mention | `_[mention: <user-ref>]_` | `_[mention: <owner>]_` |
| Doc mention | `_[mention-doc: <doc-name>]_` | `_[mention-doc: <related RFC>]_` |
| Board mention | `_[mention-board: <board-name>]_` | `_[mention-board: <project tracker>]_` |

**Rules:**

- Wrap the entire marker in italic via the surrounding markdown (`_..._`) or via `attributes: {italic: true}` on the text insert.
- Use angle brackets `< >` around the user-supplied reference name, keep `[embed:` / `[mention:` / `[mention-doc:` / `[mention-board:` as the literal prefix.
- One marker per slot. Multiple slots per block are fine (e.g. `_[mention: <owner>] - _[embed: status of <item>]_`).

**Worked example - passive scaffold for a weekly update header line:**

```json
[
  { "insert": { "text": "Overall status: " }, "attributes": { "italic": true, "bold": true } },
  { "insert": { "text": "[embed: status of <milestone item>]" }, "attributes": { "italic": true } },
  { "insert": { "text": "\n" } }
]
```

**Worked example - passive scaffold for a partner-contact line (mention slot):**

```json
[
  { "insert": { "text": "Owner: " }, "attributes": { "bold": true } },
  { "insert": { "text": "[mention: <owner>]" }, "attributes": { "italic": true } },
  { "insert": { "text": "\n" } }
]
```

**Active mode replaces slot markers with real inserts.** Once the user names the underlying item / user, the skill targets the slot block via `update_block` and replaces the delta with the actual mention / column_value insert. The slot marker format is grep-able, so a downstream agent reading the doc can detect unfilled slots and prompt the user to complete them.

## Anti-patterns - things that look reasonable but fail

- **Forgetting the terminator.** Every delta_format MUST end with `{insert: {text: "\n"}}`. Even short ones.
- **Mention with attributes.** `attributes` on a mention insert is ignored. Use formatting on adjacent text inserts instead.
- **column_value in a code block.** Code blocks render the column_value insert as literal text, not as a live embed. Use a regular text block for embeds.
- **Mention id as a string when the API expects a number** - the schema accepts string OR number, but be consistent. User IDs from `list_users_and_teams` come back as numeric strings; cast to number before sending if mixing.
- **Multiple terminators.** Only one `{insert: {text: "\n"}}` at the end. Embedded `\n` characters inside other inserts are fine for line breaks within the block, but the terminator must be the last entry.
- **Trying to delete a mention by clearing the text around it.** `update_block` replaces the whole delta. To remove a mention, omit it from the new delta_format and re-send the rest.
