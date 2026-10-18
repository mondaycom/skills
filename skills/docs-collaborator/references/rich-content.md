# Rich content in monday docs

Detail for the rich-content section of `SKILL.md`.

## Tables

The `table` block type is supported but constrained:

- **Pre-filled tables**: only via `add_markdown_content` or the initial `create_doc` markdown body. Markdown table syntax populates cells in one shot.
- **Empty tables via `create_block`**: possible, but cells cannot be populated via the API after creation. The user has to fill cells from the monday UI.
- **Restructuring (changing row / column count or widths)**: `replace_block` - the new table can be pre-filled if you replace it via `add_markdown_content` instead, by deleting the old table and appending a new markdown table after the deletion target.

When the user asks to "edit cell B2", explain the constraint and offer the closest workaround: rewrite the whole table via markdown.

## Notice boxes

A notice_box is a styled callout container. Creating one with content takes two `update_doc` calls:

1. **Call 1:** `create_block` with `block_type: "notice_box"` and `theme: "INFO" | "TIPS" | "WARNING" | "GENERAL"`. Capture the returned block ID.
2. **Call 2:** `create_block` with `parent_block_id: "<notice_box_id>"` and the child block content (typically text).

Theme cannot be changed after creation - use `replace_block` to swap themes, but child content does not transfer; recreate the children inside the new notice_box.

## Layouts

The `layout` block type creates a multi-column layout (2-6 columns). **Cell content cannot be populated via the API at all.** No markdown shortcut, no `parent_block_id` support. Layouts created via the API are empty shells the user fills from the monday UI.

If the user asks for "a two-column section with a summary on the left and metrics on the right", call this out: build the layout shell via `create_block`, then ask the user to fill the cells in the UI, or fall back to a different shape (heading + sub-headings, or a single-column with a separator).

## Mentions

`{insert: {mention: {id, type}}}` tags a user, doc, or board inline. Three types:

- `USER` (most common). `id` is the numeric user ID from `list_users_and_teams`.
- `DOC`. `id` is the doc's `object_id`.
- `BOARD`. `id` is the board ID.

Resolution rules:

- The user gives a name ("tag Daniel"): call `list_users_and_teams` with `name: "Daniel"` to get the ID. Confirm the match if multiple results.
- The user gives an email: search by the local-part as a name (e.g. `daniel.h` for `daniel.h@example.com`); fall back to asking for the display name.
- **Never call `list_users_and_teams` without a `name` or `userIds` parameter.** The unfiltered call returns the entire account user list (hundreds of KB on real tenants) and breaks token limits.

Mention blots ignore `attributes`. Do not try to bold or color a mention.

## Live column-value embeds - the differentiator

`{insert: {column_value: {item_id, column_id}}}` embeds the current value of a column on a specific board item, rendered live every time the doc is opened. This is the capability ecosystem builders should reach for - it turns a static doc into a living dashboard fragment.

Use cases worth scaffolding:

- **Weekly update:** embed the status, owner, and due date of the top 5 priorities.
- **RFC:** embed the linked tracking item's status so reviewers see whether the work has shipped.
- **Runbook:** embed the on-call rotation column and the current incident's status.
- **Project tracker summary:** at the top of a project doc, embed key milestones' due dates and statuses.

Resolution rules:

- Validate `item_id` exists via `get_board_items_page` or `get_full_board_data` before writing.
- Validate `column_id` matches a real column on the item's board via `get_board_info`. Wrong column IDs produce silent rendering gaps - the embed shows as empty space.
- **Always use the board-specific status column ID** (e.g. `color_mm2x25ey`), never a generic literal like `"status"`. Cross-skill rule.
- Be careful with private boards: embedded column values respect board permissions, so a viewer without access to the source board sees nothing.

## Comments

`add_comment` operates at three scopes:

- **Doc-level**: omit `block_id`. The comment is attached to the doc as a whole.
- **Block-level**: pass `block_id` (or an array of block IDs to highlight several at once). Only works on text, code, list_item, title, and quote blocks - not on divider, table, layout, notice_box, image, video, or giphy blocks.
- **Text-selection**: pass a single `block_id` plus `selection_from` (0-indexed start) and `selection_length`. Only valid on text / code / list_item.

Threading: pass `parent_update_id` (the comment's update ID, returned by `add_comment` or `read_docs` with `include_comments: true`) to reply.

Comment body uses HTML, not markdown. To tag a user in a comment, do not put `@` in the body - use `mentions_list` with a JSON array: `[{"id":"38288909","type":"User"}]`. Valid types: `User`, `Team`, `Board`, `Project`.

Deleting a comment via `delete_update` does NOT auto-clean the anchored reference on the host block's text insert (`attributes.comments` still lists the dead update_id). For full cleanup, separately update the block's delta to drop the `comments` attribute.
