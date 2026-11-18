# project-status-report: tool evidence

Verified against mondaycom/mcp @ 1261f96 (agent-toolkit 5.78.11) and the live tools/list on 2026-11-18.

"The review" below is an external check of these skills against the live tool schemas, which this verification started from.

Base path for links: `https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/`.

## `get_user_context`

- Skill says: returns favorites (boards, folders, workspaces, dashboards), most relevant boards and account; does not list every board in a workspace (SKILL.md:52, 210)
  - Code: takes no input; returns `user`, `account`, `favorites`, `relevantBoards`, `relevantDocs`, `relevantPeople`. Only favorite workspaces are returned. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context-tool.ts#L21-L75
  - Verdict: fixed. The review's "recent and favorite boards, account, and workspaces" now names the favorites and relevant boards the tool returns.
- Skill says: plan is `account.tier` (SKILL.md:189)
  - Code: `account.tier`, `active_members_count`, `is_during_trial`, `products`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context-tool.ts#L57-L65
  - Verdict: fixed (added, so the plan-tier note has a source).

## `workspace_info`

- Skill says: list a workspace's boards with `workspace_id`, up to 100; at 100 assume more (SKILL.md:52, 184, 211)
  - Code: required `workspace_id` (number). The description says it returns up to 100 of each object type. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/workspace-info-tool/workspace-info-tool.ts#L9-L25
  - Verdict: fixed (added the 100 cap).

## `search`

Served by a monday service outside mondaycom/mcp; checked against live tools/list. The toolkit schema is a subset of the live one: the live schema adds `strategy`, `dateRange`, `personIds`, `teamIds`, `statusLabels`, `tagNames` and others, and none of those are used here. The toolkit code is cited only for the fields the two schemas share.

- Skill says: find a board by name with `searchType: "BOARD"` and `searchTerm`; returns id, title, url, workspaceId; cannot list everything (SKILL.md:50, 52, 184, 212)
  - Live tools/list: `searchType` is required (enum BOARD, DOCUMENTS, FOLDERS, WORKSPACES, UPDATES, ITEMS, TIMELINE_ITEMS, DASHBOARDS). `searchTerm` has `minLength: 1` and is described as required. The description says "This tool has no 'list everything' mode" and points to `workspace_info` for browsing. BOARD search returns id, title, url, description, workspaceId and creatorId, so the skill names a subset. The toolkit has the same fields and the same no-browse rule — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/search-tool/search-tool.ts#L183-L247
  - Verdict: fixed. The review's version said to use `search` with `searchType: "BOARD"` to list all boards in a workspace (Step 1 and Multi-Board step 1). The live description rules that out, so that part was reverted: `workspace_info` lists boards, and `search` only finds a board by name.

## `get_board_info`

Served by a monday service outside mondaycom/mcp; checked against live tools/list. The live input schema (`boardId`, `includeKnowledge`, `includeSubscribers`, `filters`) and description match the toolkit at 1261f96 word for word. The live list does not describe the response fields, so the toolkit query is cited for those.

- Skill says: call it first for column IDs, types, status labels and groups (SKILL.md:44, 58-61)
  - Live tools/list: "Get comprehensive board information including metadata, structure, owners, and configuration ... Call this FIRST whenever you are not already familiar with a board structure (column IDs, column types, column revisions, status labels)".
  - Code (same schema): returns `columns { id title description type settings revision }`, `groups { id title }`. Its description says to call it first before `get_board_items_page` and `board_insights`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info.graphql.ts#L49-L62 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info-tool.ts#L111-L121
  - Verdict: matches.
- Skill says: owners always, subscribers only with `includeSubscribers: true` (SKILL.md:62)
  - Live tools/list: "Set includeSubscribers to true to also return the board's subscribers"; the parameter is described as "Omitted by default".
  - Code (same schema): `subscribers` / `team_subscribers` are `@include(if: $includeSubscribers)`, and the parameter defaults to false. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info-tool.ts#L36-L41 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info.graphql.ts#L64-L87
  - Verdict: fixed. The skill said "owners and subscribers" unconditionally.
- Skill says: board URL comes from `url` (SKILL.md:63)
  - Live tools/list: response fields are not described.
  - Code (same schema): `url` is selected. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info.graphql.ts#L12-L20
  - Verdict: fixed (added). The live `get_board_items_page` description lists no board URL in its response, so the report needs this field. See open question 4.
- Skill says: each status label has an `is_done` flag (SKILL.md:88)
  - Code: column `settings` is passed through raw from the API. The toolkit neither parses nor documents `is_done`.
  - Verdict: open question (API behaviour, not toolkit code).

## `get_board_items_page`

Served by a monday service outside mondaycom/mcp; checked against live tools/list. The live schema differs from the toolkit at 1261f96 in two places: `limit` maximum is 2000 (500 in code), and the live schema adds `includeIsLeaf` and `hierarchyScope`. The toolkit code is cited for nothing here.

- Skill says: `boardId`, `includeColumns: true` (SKILL.md:73-76)
  - Live tools/list: `boardId` is a required integer. `includeColumns` defaults to false. `includeGroup` defaults to true, so the Group column is available.
  - Verdict: matches.
- Skill says: paged, `limit` default 25 up to 2000; follow `pagination.nextCursor` while `pagination.has_more` (SKILL.md:78, 215)
  - Live tools/list: `limit` is an integer with minimum 1, maximum 2000 and default 25. The description says "pagination (has_more, nextCursor, count). Use nextCursor from the response as cursor to get the next page while has_more is true."
  - Verdict: fixed (added). Without it the report counted only the first 25 items.
- Skill says: `get_board_items_page` replaces the retired `get_full_board_data` (review)
  - Code / live: `get_full_board_data` is absent from the live tools/list.
  - Verdict: matches.

## `get_updates`

- Skill says: one call with `objectType: "Item"` and `objectIds` (string IDs, up to 25; batch beyond) (SKILL.md:100, 218)
  - Code: `objectType` is required (enum `Item`, `Board`). `objectIds` is `z.array(z.string()).min(1).max(25)`. More than 25 IDs is rejected. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-updates-tool/get-updates-tool.ts#L13-L31 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-updates-tool/get-updates-tool.ts#L133-L140
  - Verdict: matches (the review's fix kept; "batch beyond 25" added).
- Skill says: an item with no updates comes back with `count: 0` (SKILL.md:102)
  - Code: each item returns `{ item_id, url, updates, count }`, so an item without updates has `updates: []` and `count: 0` rather than an empty response. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-updates-tool/get-updates-tool.ts#L154-L178
  - Verdict: fixed ("returns empty" changed to `count: 0`).

## `board_insights`

- Skill says: `aggregations` is needed (without it, only an error message); `groupBy` gives the breakdown; every `groupBy` column also goes in `aggregations` without a `function` (SKILL.md:106)
  - Code: with no `aggregations` it returns `Input must contain the "aggregations" field.`. Every aggregation needs a `columnId` except `COUNT_ITEMS`. The `groupBy` description requires its columns in `aggregations` without a function. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.ts#L16-L42 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.ts#L87-L94
  - Verdict: fixed. The review's "does not return useful breakdowns" now states what the tool does.
- Skill says: worked example `boardId` (number), `groupBy: ["<status_col_id>"]`, `aggregations: [{columnId}, {function: "COUNT_ITEMS"}]` (SKILL.md:108-118)
  - Code: `boardId: z.number()`. `groupBy` is an array of column-id strings. `COUNT_ITEMS` is in the allowed function enum (live tools/list too). — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.ts#L17-L42 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights.consts.ts#L6-L19
  - Verdict: fixed. The review quoted `"boardId": "<board_id>"` as a string. The rest of the example matches.
- Skill says: rows hold the raw value under `<status_col_id>`, the label under `LABEL_<status_col_id>_0`, the count under `COUNT_ITEMS_0` (SKILL.md:120)
  - Code: a `LABEL` aggregation is added automatically for each `groupBy` column. Function aliases are `${function}_${columnId}_${index}`, or `${function}_${index}` without a column. Plain columns are aliased by their id. The tests expect `LABEL_status_0` and `COUNT_ITEMS_0`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-utils.ts#L94-L146 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.test.ts#L1218-L1236
  - Verdict: fixed. The review's `LABEL_<status_col_id>` follows the tool description (L76), but the actual key has an `_0` suffix.
- Skill says: an unknown column ID surfaces as a 403 INTERNAL_SERVER_ERROR (SKILL.md:122)
  - Code: the toolkit does not map errors for this tool. Whatever the API returns passes through, and nothing in the toolkit produces a 403.
  - Verdict: open question (observed API behaviour; kept as worded).
- Skill says: empty result falls back to manual aggregation; no-status rows have an empty or null label (SKILL.md:203)
  - Code: no rows returns `No board insights found for the given query.`. An entry with no value is set to `null`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.ts#L113-L131
  - Verdict: fixed ("empty-string" changed to "empty or null").
- Skill says (previous): AI Portfolio Risk insights in `board_insights` are Enterprise-only
  - Code: the tool only filters, groups and aggregates columns. It has no risk feature and no plan gating. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/board-insights/board-insights-tool.ts#L71-L78
  - Verdict: matches the review's removal.

## `get_column_type_info`

- Skill says: `fetchMode: "guidelines"` for aggregation and filter rules (SKILL.md:106, 214)
  - Code: `columnType` is required. `fetchMode` is `schema` (default) or `guidelines`, and `guidelines` returns `guidelines.filter` and `guidelines.aggregation`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-column-type-info/get-column-type-info-tool.ts#L13-L24
  - Verdict: matches.

## `get_board_activity`

- Skill says: tool default 30 days, skill default 7, so pass `fromDate` (ISO 8601); max 90 days, clamped; entries have `created_at`, `event`, `entity`, `user_id` (SKILL.md:126)
  - Code: `fromDate` defaults to 30 days ago, values older than 90 days are clamped, `toDate` defaults to now. Each entry is `{created_at, event, entity, user_id}` (plus `data` only with `includeData`). — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-activity/get-board-activity-tool.ts#L19-L33 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-activity/get-board-activity-tool.ts#L62-L119
  - Verdict: matches (format, 90-day clamp and entry fields added).

## `show-chart`, `show-battery`, `show-table`

- Skill says: use the widgets when they are in the tool list, otherwise the same section in Markdown with the board URL; no widgets in a doc (SKILL.md:25-34, 133, 195, 198, 205, 219-221)
  - Live tools/list: all three are present with MCP Apps `_meta.ui.resourceUri`. They are not in mondaycom/mcp.
  - Verdict: fixed. The review's version removed the widgets for every client. The conditional restores them for clients that render them, and Markdown remains the fallback.
- Skill says: `show-chart` with `type: "pie"|"bar"` and `data: [{name, y}]` (SKILL.md:29, 32)
  - Live tools/list: required `data` (`name`, `y`, optional `color`, minItems 1) and `type` enum `pie`, `bar`.
  - Verdict: matches.
- Skill says: `show-battery` with one segment per status, `name`, `y`, `color` all required (SKILL.md:30)
  - Live tools/list: `data[]` items require `name`, `y` (min 0), `color`.
  - Verdict: matches. The previous text described it as a single-percentage indicator, but the schema takes segments.
- Skill says: `show-table` with `boardId` and a `filters` array (SKILL.md:31, 38-40)
  - Live tools/list: required `boardId` (string). It also takes optional `filters` (`columnId`, `compareValue`, `operator`, `compareAttribute`) and `filtersOperator`. It has no parameter for choosing columns, so the previous "columns: Item, Owner, ..." was not expressible and is not restored.
  - Verdict: matches.
- Skill says: filter fallback, because some clients show a filter chip but do not apply it (SKILL.md:38-40)
  - Live tools/list: the schema accepts `filters`, but neither the schema nor the description says anything about client-side application.
  - Verdict: open question (client rendering behaviour; rule kept as before).

## Open questions

1. `board_insights` 403 INTERNAL_SERVER_ERROR for a nonexistent column ID (SKILL.md:122) is API behaviour that the toolkit code cannot confirm.
2. The `is_done` flag on status labels and "often unset" (SKILL.md:88) come from raw column `settings`, so the toolkit code cannot confirm them. `board_insights` also offers an `IS_DONE` function, which might be a more reliable done signal; this is untested.
3. The `show-table` filter-chip-not-applied premise (SKILL.md:38) is client behaviour and is not in the live schema. The live `show-table` description requires a `boardContextToken` from `get_board_info` when filtering, but neither the `show-table` input schema nor `get_board_info` in 1261f96 has such a field.
4. `get_board_info`, `get_board_items_page` and `search` are served outside mondaycom/mcp, and live tools/list does not describe their response fields. Response-field claims (`url` on `get_board_info`, and the absence of a board URL from `get_board_items_page`) rest on the live descriptions plus the toolkit query. They have not been confirmed against a live response.
