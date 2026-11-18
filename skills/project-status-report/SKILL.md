---
name: project-status-report
description: |
  Generates structured project status reports from monday.com - a customizable AI work
  platform with dedicated products for work management, CRM, dev, service, and campaigns.
  Surfaces overdue items, blockers, workload distribution, and recent activity
  across one or more boards.
  Use when user asks for "project status", "board summary", "what's overdue",
  "status report", "how's the project going", "weekly update", "board overview",
  "progress update", or "team workload".
  Don't use for writing or editing a monday doc (use docs-collaborator); when the user wants the report as a doc, build it here, then write it with docs-collaborator.
license: MIT
metadata:
  author: monday.com
  version: "0.1.5"
---
# Project Status Report

Generate a structured status report from one or more monday.com boards. monday.com is a customizable AI work platform with dedicated products for work management, CRM, dev, service, and campaigns - boards are the central unit of organization. This skill combines board metadata, item data, aggregated insights, and recent activity into a single actionable summary.

## Output Behavior

Present the report in plain language with clean Markdown formatting. Don't narrate which API tools were called or show raw JSON - summarize what the data means. Highlight what needs attention (blockers, overdue items, unassigned work) rather than echoing raw numbers. Always include each board's URL so the user can navigate directly.

## Visual sections: widgets when available, Markdown otherwise

When `show-chart`, `show-battery` and `show-table` are in your tool list, use them for the visual sections. When they are not, render the same section in clean Markdown and include the board URL:

- **Status distribution**: `show-chart` with `type: "pie"` or `"bar"` and `data: [{"name": "<label>", "y": <count>}]`; otherwise a Markdown table with Status, Item Count, and % of Total
- **Project completion**: `show-battery` with one segment per status (`name`, `y`, and `color` - all three required); otherwise a text progress line (e.g., `12 of 20 items done (60%)`)
- **Overdue items**: `show-table` with `boardId` and a `filters` array scoping to overdue items, so the user can sort, filter, and click through; otherwise a Markdown table with Item, Owner, Due Date, Status, and Group (cap at 10 rows and append `(+N more)` if more match, alongside the board URL)
- **Workload by person**: `show-chart` with `type: "bar"` comparing owners; otherwise a Markdown table comparing total, in-progress, and overdue items per owner

For short summary lines (totals, counts), use compact bullet points. When a widget rendered, do not duplicate its content as a Markdown table afterwards - pick one or the other, except for the fallback below.

### show-table filter fallback (required)

When you call `show-table` with a `filters` array (for example, to scope to overdue items or to exclude Done), always print a short plain-text list of the filtered rows directly below the widget. Some clients surface a filter chip but do NOT auto-apply the filter, so Done items remain visible in the table. The plain-text list is the authoritative view; the widget is the interactive surface.

Format the fallback as a tight bulleted list of at most 10 rows, each showing item name, owner, due date, and status. If more than 10 rows match, print the first 10 and add a "(+N more)" line. Skip the fallback only when no `filters` array is sent (an unfiltered show-table renders correctly).

## Precondition

Before starting, always call `get_board_info` for every board included in the report. This retrieves column IDs, column types, status label mappings, and group structure - all required to correctly interpret item data.

## Workflow

### Step 1: Identify the target boards

If the user specifies a board by name or ID, use it directly (use `search` with `searchType: "BOARD"` and `searchTerm: "<board name>"` when given a name; it returns each board's id, title, url and workspaceId). Otherwise:

1. Call `get_user_context` to retrieve the user's favorites (boards, folders, workspaces, dashboards), most relevant boards, and account (note: `get_user_context` does not return every board in a workspace — call `workspace_info` with `workspace_id` to list a workspace's boards; `search` needs a non-empty `searchTerm` and cannot list everything)
2. If ambiguous, ask the user which board(s) to report on
3. For multi-board reports, collect all board IDs

### Step 2: Fetch board structure

For each board, call `get_board_info` with the board ID. Extract:

- **Column definitions** - IDs, types, and allowed values (especially status label mappings)
- **Groups** - group IDs and names (these represent workflow stages like "To Do", "In Progress", "Done")
- **Owners** - board owners (subscribers only when you pass `includeSubscribers: true`)
- **Board URL** - the `url` field, for linking the report

Identify which columns represent:
- **Status** - the primary workflow state (usually a `status` type column)
- **Date/Timeline** - deadlines or date ranges (columns of type `date` or `timeline`)
- **Person** - assignment (columns of type `people`)
- **Priority** - importance level (usually a `status` or `color` type column labeled "Priority")

### Step 3: Retrieve items

Call `get_board_items_page` with:
- `boardId`: the target board
- `includeColumns`: `true`
- Use filtering/sorting as needed to focus on active items

Results are paged (`limit` defaults to 25, up to 2000). While `pagination.has_more` is true, call again with `cursor` set to `pagination.nextCursor` until you have every item in scope.

Parse each item's column values using the column definitions from Step 2.

### Step 4: Identify overdue items

An item is **overdue** when ALL of these are true:
1. It has a date or timeline column with a value in the past (before today, using the user's timezone context)
2. Its status column is NOT set to a "done" label

**Determining "done" labels.** Each label in the board's status column has an `is_done` flag. In practice this flag is often unset on user-created or default boards (observed: false on every label including "Done" on real test boards). Treat `is_done: true` as the primary signal when set; otherwise fall back to label-name matching against the conventional set: `Done`, `Completed`, `Closed`, `Resolved`, `Verified`, `Fixed`, `Won`, `Approved`. If neither signal applies, ask the user which label means "complete" on this board.

Also flag items due within 24 hours as "due soon" to give the user advance warning.

Collect overdue items with: item name, owner, due date, current status, and group name.

### Step 5: Identify blockers

An item is **blocked** when ANY of these are true:
1. Its status is "Stuck", "Blocked", or a similar label (check the board's status labels)
2. It has recent updates/comments flagging a blocker

For items with "Stuck" or blocked status, call `get_updates` once with `objectType: "Item"` and `objectIds: ["<stuck_item_id_1>", ...]` (string IDs, up to 25 per call; batch beyond 25) to fetch the latest comments explaining why they are stuck.

**Fallback when no updates exist.** If `get_updates` returns `count: 0` for a stuck item, surface context from the item's text columns instead - in priority order: any column titled `Notes`, `Description`, or `Steps to Reproduce`, then any other `text` or `long_text` column with non-empty content. Many teams record blockers in a Notes column rather than as updates. Do not present a blocked item as having "no detail" if a text column has the explanation.

### Step 6: Get aggregated insights

Call `board_insights` with explicit `groupBy` and `aggregations`. Without `aggregations` the tool returns only an error message, and without `groupBy` you get a single total instead of a breakdown. Every column in `groupBy` must also appear in `aggregations` without a `function`. Check `get_column_type_info` with `fetchMode: "guidelines"` for the column type's aggregation and filter rules.

Example call for status distribution (`boardId` is a number):
```json
{
  "boardId": <board_id>,
  "groupBy": ["<status_col_id>"],
  "aggregations": [
    {"columnId": "<status_col_id>"},
    {"function": "COUNT_ITEMS"}
  ]
}
```

Each row in `data` holds the raw value under `<status_col_id>`, the human-readable label under `LABEL_<status_col_id>_0`, and the count under `COUNT_ITEMS_0`. Run analogous `groupBy` queries for group distribution or person workload when needed. This provides the high-level numbers without re-processing all items.

**Use the exact status column ID from Step 2.** Pass the column ID exactly as `get_board_info` returns it (for example, `status` on older boards or `color_mk...` on newer ones) — never guess `"status"` without checking `get_board_info` first. Passing a nonexistent column ID surfaces as a 403 INTERNAL_SERVER_ERROR from `board_insights`. If a 403 fires here, re-read `get_board_info`, locate the column whose `type` is `status`, and retry with that exact ID. The same rule applies to Priority / Severity / Workflow status columns on dev / CRM / service boards.

### Step 7: Get recent activity

Call `get_board_activity` with an explicit time range. The MCP tool's own default is 30 days, but this skill's default is 7 days - so you MUST pass `fromDate` (ISO 8601, and optionally `toDate`) to enforce the 7-day window unless the user asks for a different period. The tool reaches back at most 90 days and clamps older dates. Each entry has `created_at`, `event`, `entity` and `user_id`. This surfaces:
- Items created, updated, or moved
- Status changes
- Assignment changes

### Step 8: Format the report

Present the report in this structure. Where a section has a widget and the widget is in your tool list (see Visual sections), call it in place of the Markdown block:

---

**Project Status: [Board Name]**
*Report generated: [today's date] | Period: [time range]*
*Board URL: [board URL]*

**Summary**
- Total items: [count]
- Completed: [count] ([percentage]%)
- In progress: [count]
- Overdue: [count]
- Blocked: [count]

**Project Completion:** `[done]` / `[total]` items complete (`[percentage]%`)

**Status Distribution**
| Status | Items | % of Total |
|--------|------:|-----------:|
| [Label] | [count] | [pct]% |

**Overdue Items**
| Item | Owner | Due Date | Status | Group |
|------|-------|----------|--------|-------|
| [Item name] | [Owner] | [YYYY-MM-DD] | [Status] | [Group] |

*(Show up to 10 overdue rows; if more exist, append `(+N more)` and point to the board URL.)*

**Blockers**

- **[Item name]** (Owner: @name) - [latest comment or reason]
- **[Item name]** (Owner: @name) - [latest comment or reason]

**Recent Activity (Last 7 Days)**
- [count] items completed
- [count] items created
- [count] status changes
- Notable: [highlight any significant moves or updates]

---

End with 2-3 logical next steps:
- "Want me to follow up with the owners of overdue items?"
- "Should I draft a status update to share with stakeholders?"
- "Want me to dig into why [specific item] is blocked?"

## Multi-Board Reports

When the user asks for a report across multiple boards or a workspace:

1. Call `workspace_info` (with `workspace_id`) to list the boards in scope; it returns up to 100 boards, so if you get 100, assume there are more. Use `search` (`searchType: "BOARD"`) only to find boards by name, and `get_user_context` only for favorite/relevant boards
2. Run Steps 2-7 for each board
3. Present a combined summary at the top, followed by per-board sections
4. Highlight cross-board patterns (e.g., "3 of 5 boards have overdue items")

**Plan-tier note:** Multi-board aggregation works best on Standard and above. On Free / Basic, monday's cross-board surfaces are more limited - the report still works but may feel single-board. `get_user_context` returns the plan as `account.tier`.

## Output Customization

Adjust the report based on user requests:
- **"Just the blockers"** - Skip to Step 5, show only blockers section
- **"Overdue items only"** - Skip to Step 4, show only the overdue section (`show-table` when available, otherwise the Markdown table)
- **"Last 30 days"** - Adjust the time range in Step 7
- **"For my team"** - Filter items by person assignment in Step 3
- **"As a doc"** - Widgets cannot go into a doc, so build every section as plain text and Markdown tables (no `show-chart`, `show-battery` or `show-table`), then hand the result to the `docs-collaborator` skill to write

## Error Prevention and Remediation

- If `get_board_info` fails (e.g., board not found, no permission), report which board failed and suggest an alternative: "I cannot access the Q3 Marketing board. Want me to use a board you have access to, or should I ask the owner to share it?"
- If `board_insights` returns empty, fall back to manual aggregation from `get_board_items_page` rather than reporting empty distributions. Items with no status set surface as an empty or null label in `board_insights`; render these as "No status" in the chart or table, not as a blank segment or row.
- If the board has no status column, note this and report based on group distribution instead.
- **Empty board.** If `get_board_items_page` returns 0 items, do not run Steps 4-7 or render any widgets. Reply: "<board name> has no items yet." Then suggest creating items (links to the `task-management` skill) or reviewing the board's setup.
- **Multi-board partial failure.** If one board in a multi-board request is inaccessible (private, deleted, or 403), do not abort the whole report. Run the full flow for accessible boards, then add a brief "Could not access: <board name(s)>" note at the top of the combined summary with a suggested next step ("ask the owner to share", "remove from request", "pick a different board").

## Tools Used

- `get_user_context` - identify the user's favorite/relevant boards, favorite workspaces, and plan
- `workspace_info` - list a workspace's boards by `workspace_id` (up to 100)
- `search` - find boards by name (`searchType: "BOARD"`, `searchTerm`)
- `get_board_info` - board structure, column definitions, status labels
- `get_column_type_info` - grouping/filter guidelines (`fetchMode: "guidelines"`)
- `get_board_items_page` - item data with column values (paged with `cursor`)
- `board_insights` - aggregated counts and distributions (`groupBy`, `aggregations`)
- `get_board_activity` - recent changes and timeline
- `get_updates` - comments on blocked/flagged items (`objectType: "Item"`, `objectIds` up to 25)
- `show-chart` - status / workload distribution widget (when in the tool list)
- `show-battery` - completion widget (when in the tool list)
- `show-table` - overdue items widget (when in the tool list)
