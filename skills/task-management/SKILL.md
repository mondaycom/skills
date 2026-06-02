---
name: task-management
description: |
  Creates, updates, assigns, and manages items across monday.com boards. monday.com
  is a customizable AI work platform with dedicated products for work management,
  CRM, dev, service, and campaigns. Handles correct column value formatting for all column types.
  Use when user asks to "create a task", "add an item", "update status", "assign to",
  "change due date", "move item", "mark as done", "create a subitem", "add a comment",
  or "update the priority".
license: MIT
metadata:
  author: monday.com
  version: "0.1.1"
---

# Task Management

Create, update, assign, and manage items on monday.com boards. In monday.com, every row on a board is an "item" - it could represent a task, a deal, a bug, a content piece, or any unit of work. This skill teaches the correct column value formatting required by monday.com's API - the most common source of errors when working with the platform programmatically.

## Critical Precondition

**Before creating or updating any item, ALWAYS call `get_board_info` first** to retrieve the board's column definitions. You need:
- Column IDs (e.g., `status`, `date4`, `person`)
- Column types (e.g., `status`, `date`, `people`)
- Status label mappings (e.g., which index maps to "Done")
- Allowed values for dropdown columns

Without this step, column value writes will fail silently or set incorrect values.

## Workflow: Create an Item

### Step 1: Find the target board

If the user specifies a board name:
1. Call `search` with the board name to find it
2. Confirm the match with the user if multiple results

If the user doesn't specify:
1. Call `get_user_context` to list their boards
2. Ask which board to use

### Step 2: Get board structure

Call `get_board_info` with the board ID. Note:
- All column IDs and their types
- Group IDs and names (to know where to place the item)
- Status label options (the exact label strings like "Working on it", "Done")

### Step 3: Get column type details (if needed)

If you need to understand the exact format for a column type, call `get_column_type_info` with the column type. This returns the JSON schema and validation rules.

### Step 4: Create the item

Call `create_item` with:
- `boardId`: the target board ID
- `groupId`: the group to place the item in (from Step 2)
- `itemName`: the item name
- `columnValues`: a JSON string with column values (see formatting guide below)

### Step 5: Confirm and follow up

After creation:
- Report what was done in plain language - don't narrate the API calls
- Include the item name, board name, and the direct URL returned by the MCP tool
- Suggest 2-3 logical next steps, for example:
  - "Want me to assign someone to this?"
  - "Should I set a due date?"
  - "Want me to break this into subitems?"
  - "Should I add a comment with more context?"

## Workflow: Update an Item

### Step 1: Locate the item

If the user provides an item name but not an ID:
1. Call `get_board_info` to understand the board structure
2. Call `get_board_items_page` with a search query to find the item
3. **Cross-check returned names against the user's target name.** `get_board_items_page` with `searchTerm` behaves as a relevance sort, not a strict name filter - when no true match exists, it still returns items, ranked by partial overlap. Before writing, scan the returned list for an exact (case-insensitive) or near-exact name match. If none of the returned items name-match the target, treat the search as a no-match result and route to step 4, even though the response was non-empty.
4. If multiple results match, confirm the match with the user before writing
5. If no results name-match, do not silently fail and do not write to the closest-ranked but wrong-name item. Ask the user to confirm the name, suggest the closest matches you saw, or offer to list the board's items

### Step 2: Get board structure

Call `get_board_info` if not already done. You must know the column IDs and types before writing values.

### Step 3: Update column values

Call `change_item_column_values` with:
- `boardId`: the board ID
- `itemId`: the item ID
- `columnValues`: a JSON string with only the columns being changed

### Step 4: Confirm

Report what was changed in plain language: item name, which fields were updated, and the new values. Always include the direct URL. Don't describe which API tools were called.

## When to use MCP UI widgets

Default to plain text confirmations. Reach for `show-assign` when the user asks to assign work and needs to pick from a long list of teammates - the widget renders a person picker that the user can interact with. For single-person assignments where the user has already named the assignee, just look up the user ID via `list_users_and_teams` and write directly without the widget.

Do not use `show-table` to confirm a single create or update - plain text plus the item URL is cleaner.

## Column Value Formatting Guide

The `columnValues` parameter for `create_item` and `change_item_column_values` is a JSON string. Each column type requires a specific format. **Getting this wrong is the #1 cause of failures.**

For the complete formatting reference with all column types, see **`references/column-value-formats.md`**.

Quick reference for the most common types:

**Status** - Use `{"label": "Done"}` with the exact label string from `get_board_info`:
```json
{"status_column_id": {"label": "Done"}}
```

**Date** - Use `{"date": "YYYY-MM-DD"}`:
```json
{"date_column_id": {"date": "2025-06-15"}}
```

**People** - Use `{"personsAndTeams": [{"id": USER_ID, "kind": "person"}]}`:
```json
{"person_column_id": {"personsAndTeams": [{"id": 12345678, "kind": "person"}]}}
```

**Text** - Plain string value:
```json
{"text_column_id": "Some text value"}
```

**Numbers** - Plain string of the number:
```json
{"numbers_column_id": "42"}
```

**Email** - Use `{"email": "...", "text": "..."}`:
```json
{"email_column_id": {"email": "user@example.com", "text": "Contact"}}
```

**Phone** - Use `{"phone": "...", "countryShortName": "..."}`:
```json
{"phone_column_id": {"phone": "+15551234567", "countryShortName": "US"}}
```

**Dropdown** - Use `{"labels": ["Option 1", "Option 2"]}`:
```json
{"dropdown_column_id": {"labels": ["High Priority"]}}
```

**Timeline** - Use `{"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"}`:
```json
{"timeline_column_id": {"from": "2025-06-01", "to": "2025-06-30"}}
```

**Checkbox** - Use `{"checked": "true"}` or `{"checked": "false"}` (string, not boolean):
```json
{"checkbox_column_id": {"checked": "true"}}
```

### Multiple columns at once

Combine multiple column updates in a single JSON string:
```json
{
  "status": {"label": "Working on it"},
  "date4": {"date": "2025-06-15"},
  "person": {"personsAndTeams": [{"id": 12345678, "kind": "person"}]},
  "text0": "Implementation notes here"
}
```

## Creating Subitems

To create a subitem, call `create_item` with the `parentItemId` parameter set to the parent item's ID. The subitem will be created on the parent item's subitem board.

Note: Subitem boards have their own column structure. Call `get_board_info` on the subitem board to get its column definitions before setting column values.

## Adding Comments

To add a comment or update to an item, call `create_update` with:
- `itemId`: the item ID
- `body`: the comment text (supports basic HTML: `<b>`, `<i>`, `<a>`, `<ul>`, `<li>`)

## Common Patterns

**Mark as done:** Get the board's "done" status label from `get_board_info`, then call `change_item_column_values` with `{"status_column_id": {"label": "Done"}}`. After confirming the status change, check whether the board has a matching terminal group (Done, Closed, Completed, Resolved, Fixed, Won, Lost, Archived) and the item is not already in it. If so, offer it as a next step: "Want me to also move it to the Done group?" Do not auto-move - some boards intentionally keep items in their original group regardless of status, so always ask first.

**Reassign:** Get the new person's user ID from `list_users_and_teams`, then call `change_item_column_values` with the people column format.

**Batch updates:** When updating multiple items, call `change_item_column_values` for each item. There is no batch endpoint.

**Move to group:** The MCP toolkit has no direct "move item" tool (`move_object` is for boards/folders/overviews, not items). Use `all_monday_api` with the `move_item_to_group` mutation: `mutation { move_item_to_group (item_id: <ID>, group_id: "<GROUP_ID>") { id } }`. Do not use `create_item` with `duplicateFromItemId` for moves - that creates a copy in the new group and leaves the original in place.

## Smart Defaults

When the user doesn't specify all values:
- **Status**: Default to the first non-done label (usually "Not Started" or "Working on it") - check the board's label order via `get_board_info`. State the default in your response.
- **Group**: Choose a group in this order. (1) If any group's name matches a backlog-pattern (case-insensitive: `To Do`, `Todo`, `Backlog`, `New`, `Open`, `Not Started`, `Inbox`, `Lead`, `Triage`), use that. (2) Otherwise, use the top non-terminal group. A terminal group is one whose name matches a done-pattern (case-insensitive: `Done`, `Closed`, `Completed`, `Complete`, `Verified`, `Resolved`, `Fixed`, `Won`, `Lost`, `Archived`, `Cancelled`, `Canceled`). (3) If every group is terminal, ask the user which group to use rather than guess. Some monday boards return "Done" as the first group from the API, so do not blindly use `groups[0]`. Always state which group you used so the user can override.
- **Person**: If the user is known and the board has a people column, offer to assign to them. Do not auto-assign without asking.

## Bulk Operations

When creating or updating 5+ items in one request:
- Process all items, then present a count summary and a link to the board
- Don't list every individual item - say "Created 12 items on the Marketing board" with the board URL
- If any items fail, report the failure count and reasons separately

For bulk **destructive** operations (3+ updates that could be hard to reverse), confirm with the user before executing.

## Error Prevention and Remediation

1. **Always call `get_board_info` first** - never guess column IDs or status labels
2. **Use exact status label strings** - "Working on it" not "working on it" (case-sensitive)
3. **Check column type before writing** - a `status` column and a `color` column use different formats
4. **Stringify the columnValues** - the parameter expects a JSON string, not a JSON object
5. **Validate person IDs** - use `list_users_and_teams` to get valid user IDs before assigning. Always pass the `name` parameter (or `userIds` if known) to scope the query. Calling `list_users_and_teams` with no parameters returns the entire account user list and will exceed token limits on large tenants. If the user provides an email, search by the local-part as a name first (e.g. "alex.po" for `alex.po@example.com`); if no match, ask the user to provide the person's display name

When an MCP call fails, do not pass the raw error through. Diagnose and propose a fix:

- "Status label 'Done!' not found on this board. The board uses 'Done' (no exclamation). Should I use that?"
- "Person ID 12345 is not in your account. Want me to list available users so you can pick the right one?"
- "Permission denied on this board - it is private and you are not subscribed. Want me to ask the owner to add you, or pick a different board?"

## Tools Used

- `get_user_context` - find user's boards
- `search` - find boards by name
- `get_board_info` - board structure, column IDs, status labels (REQUIRED before writes)
- `get_column_type_info` - detailed column type schema and validation rules
- `get_board_items_page` - find items by name or filter
- `create_item` - create new items or subitems
- `change_item_column_values` - update column values on existing items
- `create_update` - add comments to items
- `all_monday_api` - for the `move_item_to_group` mutation when moving items between groups
- `list_users_and_teams` - look up user IDs for people columns
- `show-assign` (selectively) - person-picker widget when user is choosing from many teammates

## Additional Resources

- **`references/column-value-formats.md`** - Complete column value formatting reference for all monday.com column types
