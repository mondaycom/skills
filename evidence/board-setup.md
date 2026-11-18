# board-setup: tool evidence

Verified against mondaycom/mcp @ 1261f96 (agent-toolkit 5.78.11) and the live tools/list on 2026-11-18.

"The review" below is an external check of these skills against the live tool schemas, which this verification started from.

Base: `T = https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools`, `G = https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/monday-graphql`.

## `search`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: name-collision preflight with `searchType: "BOARD"`, `searchTerm`, scoped with `workspaceIds: [<target workspace id>]` (SKILL.md:276); same check after a failed `create_board` (SKILL.md:64)
- Code: `searchTerm` is required, `min(1)`; BOARD search forwards `workspaceIds` and `boardIds` to the boards query — T/search-tool/search-tool.ts#L189-L205, T/search-tool/search-tool.ts#L283-L289, T/search-tool/search-tool.ts#L329-L335. Live tools/list: `searchType` enum includes `BOARD`; description says "BOARD search ... Optionally scope it with workspaceIds and/or boardIds".
- Verdict: fixed (the review's `searchType`/`searchTerm` kept; "scoped to the target workspace if supported" replaced with the real `workspaceIds` parameter)

## `create_board`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: `boardName`, `boardKind` `"public"`/`"private"`, `workspaceId`, `folderId`, `creationPrompt` (SKILL.md:45-49); returns a board URL (SKILL.md:173)
- Code: toolkit schema has `boardName`, `boardKind` (default public), `boardDescription`, `workspaceId`, `boardOwnerIds`, `useMlsTemplate`, `useDatasetTemplate` — no `folderId` or `creationPrompt`; returns `board_id`, `board_name`, `board_url` — T/create-board-tool.ts#L11-L19, T/create-board-tool.ts#L39-L54. Live tools/list: required `boardName`, `boardKind` (enum `private`/`public`/`share`), and has `folderId` (integer) and `creationPrompt` ("Omit to create an empty board").
- Verdict: matches the live schema (live wins). Code and live schema disagree on `folderId`/`creationPrompt`; see Open questions.

## `create_group`

- Skill says: `positionRelativeMethod: "before_at"`, `relativeTo: "topics"`; `groupColor` accepts only its enum hex values (SKILL.md:74, SKILL.md:87)
- Code: `groupColor` is `z.enum(GROUP_COLORS)` (19 hex values); `positionRelativeMethod` is `PositionRelative` (`after_at`/`before_at`) — T/create-group/create-group-tool.ts#L12-L26, T/create-group/create-group.consts.ts#L6-L26. Live tools/list matches.
- Verdict: matches

## `create_column`

- Skill says: `boardId`, `columnTitle`, `columnType`, `columnSettings` as a JSON string (SKILL.md:108-112); malformed `columnSettings` returns an error (SKILL.md:277)
- Code: schema `boardId` (number), `columnType`, `columnTitle`, `columnDescription`, `columnSettings` (string); `columnSettings` goes through `JSON.parse` (throws on bad JSON) and is sent as `defaults` of `create_column` — T/create-column-tool.ts#L9-L22, T/create-column-tool.ts#L52-L62, G/queries.graphql.ts#L116-L135. Live tools/list: required `boardId`, `columnType`, `columnTitle`.
- Verdict: matches (the review's `title` -> `columnTitle` fix kept)
- Skill says: label entry needs `label`, `color`, `index`; `is_done` optional; color tokens from StatusColumnColors (SKILL.md:127-139, references/column-types-guide.md:17-27)
- Code: `CreateStatusLabelInput { label!, color: StatusColumnColors!, index!, description, is_done }`; every token the skill lists is in `StatusColumnColors` — G/schema.graphql#L12300-L12310, G/schema.graphql#L13152-L13193
- Verdict: matches
- Skill says: column type `board_relation (Connect boards)` (references/column-types-guide.md:203)
- Code: `columnType` enum has `board_relation`, no `connect_boards` (live tools/list `create_column.columnType.enum`)
- Verdict: matches (the review's fix kept). Every other type in references/column-types-guide.md is in the enum.

## `update_column`

- Skill says: send `boardId`, `columnId`, `columnType` (existing type), `revision` and every label to keep with its `id`; new labels without `id`; use the returned `revision` next (SKILL.md:158-163, SKILL.md:287)
- Code: required `boardId`, `columnId`, `columnType` ("Must match the existing column type"), `revision`; returns `column_id`, `column_title`, `revision` — T/update-column-tool.ts#L15-L48, T/update-column-tool.ts#L80-L109. Live tools/list: required `boardId`, `columnId`, `columnType`, `revision`.
- Verdict: matches (the review's `columnType` fix kept; the duplicate parameter list in the intro sentence was dropped to stay under the word limit)
- Skill says: to recolor, send the label's existing `id` with the new `color`, then read the column back with `get_board_info`; if unchanged, use the label editor (SKILL.md:165)
- Code: the tool description lists "rename, recolor, or reorder" as supported edits (T/update-column-tool.ts#L67). `columnSettings` is `JSON.parse`d and passed unchanged as `settings: JSON` to the generic `update_column` mutation — no field is stripped or rewritten (T/update-column-tool.ts#L83-L100, G/queries.graphql.ts#L145-L171). The API input type `UpdateStatusLabelInput` takes `color: StatusColumnColors!` together with `id: Int` (G/schema.graphql#L13264-L13276), so a color change on an existing id is accepted by the schema. Whether the backend persists it is not visible in this repo.
- Verdict: fixed (reworded as a supported operation with a read-back check; the previous "not supported" claim is contradicted by the tool description and schema)
- Skill says: `update_column` rejects a `settings` wrapper (SKILL.md:152)
- Code: tool description: columnSettings is the flat payload, "not wrapped again as {"settings": {"labels": [...]}}" — T/update-column-tool.ts#L66
- Verdict: matches

## `get_board_info`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: `filters.columns.ids` returns the column's `revision`, `type`, `settings.labels` (SKILL.md:160, SKILL.md:288); `top_group` and group order are checked after `creationPrompt` (SKILL.md:55)
- Code: columns select `id title description type settings revision`, filtered by `columnIds`; board selects `groups { id title }` and `top_group { id }` — T/get-board-info/get-board-info.graphql.ts#L48-L62, T/get-board-info/get-board-info.graphql.ts#L95-L98. Live tools/list has `filters.columns.ids`.
- Verdict: matches

## `get_column_type_info`

- Skill says: shows the `columnSettings` schema (SKILL.md:152, SKILL.md:283); its guidelines list `any_of` with `["EXACT", "<date>"]` for item filters (SKILL.md:278)
- Code: `fetchMode` `schema` (default) / `guidelines` — T/get-column-type-info/get-column-type-info-tool.ts#L14-L40; date guidelines list `any_of` with `["EXACT", "2025-01-01"]`, `within_the_next`, `within_the_last`, `between` — T/utils/column-filter-aggregation-guidelines.ts#L49-L59
- Verdict: matches

## `create_items`

- Skill says: up to 20 items per call (SKILL.md:169)
- Code: `MAX_ITEMS_PER_CALL = 20` — T/create-items-tool/constants.ts#L1. Live tools/list `items.maxItems: 20`.
- Verdict: matches

## `get_board_items_page`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: call with `includeColumns: true` to check starter items (SKILL.md:53)
- Code: `includeColumns` boolean, default false — T/get-board-items-page-tool/get-board-items-page-tool.ts#L90
- Verdict: matches

## `update_items`

- Skill says: fill blank starter values in one call (SKILL.md:58)
- Code: up to 40 updates per call — T/update-items-tool/constants.ts#L1, T/update-items-tool/update-items-tool.ts#L55
- Verdict: matches

## `create_view_table` / `update_view_table`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: saved-view date filters (SKILL.md:278); `create_view_table` for saved views (SKILL.md:290)
- Code: both tools exist with a `filter.rules[]` of `{column_id, compare_value, operator}` — T/create-view-table-tool/create-view-table-tool.ts#L65-L75, T/update-view-table-tool/update-view-table-tool.ts#L65-L66, T/update-view-table-tool/update-view-table-tool.ts#L104. Live tools/list includes `update_view_table` ("Update an existing table-type board view").
- Verdict: fixed (The review removed `update_view_table` as unavailable; it is in the live list, so the mention is restored)
- The date values that views accept (`between`, `"TODAY"`, `["DAYS", 7]`, `EXACT` rejected) are server behavior the toolkit does not validate (`compare_value` is `z.any()`); not verifiable from code.

## `create_automation` / `list_automations` / `manage_automations`

- Skill says: one automation per `create_automation` call, verify with `list_automations`, deactivate with `manage_automations`; some automations cannot be changed through the API (SKILL.md:49, SKILL.md:59, SKILL.md:238)
- Code: `create_automation` takes `userPrompt`, `boardId` (string), live description "Use one trigger" — T/automations-tools/create-automation/create-automation-tool.ts#L11-L22; `list_automations` returns `workflows` and read-only `legacyAutomations` — T/automations-tools/list-automations/list-automations-tool.ts#L28-L60; `manage_automations` `action` enum `activate`/`deactivate`/`delete`, `workflowId` — T/automations-tools/manage-automations/manage-automations-tool.ts#L21-L31
- Verdict: matches

## `all_monday_api`

- Skill says: `delete_group`, `update_group` (`group_attribute: title` / `relative_position_before` / `relative_position_after`, needs a selection set), `change_column_title`; on rejection, inspect the schema with `get_graphql_schema` / `get_type_details` (SKILL.md:76-83, SKILL.md:121)
- Code: the tool validates the document against the schema and throws `GRAPHQL_VALIDATION_FAILED` before sending — T/all-monday-api-tool.ts#L20-L21, T/all-monday-api-tool.ts#L145-L147; `delete_group(board_id, group_id)`, `update_group(board_id, group_attribute: GroupAttributes!, group_id, new_value)`, `GroupAttributes` includes `title`, `relative_position_before`, `relative_position_after`; `change_column_title(board_id, column_id, title)` — G/schema.graphql#L537-L539, G/schema.graphql#L670-L674, G/schema.graphql#L11008-L11019, G/schema.graphql#L385-L388. `get_graphql_schema` and `get_type_details` are in the live list.
- Verdict: matches

## Next-step suggestions (SKILL.md:197-200)

- Skill says: "invite team members", "create a dashboard"
- Code: `add_users_to_board(board_id, kind, user_ids)` is in the API schema (G/schema.graphql#L349-L352, via `all_monday_api`); `create_dashboard` is in the live list (T/dashboard-tools/create-dashboard-tool.ts#L14-L30)
- Verdict: fixed (the review's replacements reverted; both actions are available)

## Open questions

- `create_board` `creationPrompt` and `folderId` are in the live tools/list but not in the toolkit's `create_board` at 1261f96 (T/create-board-tool.ts#L11-L19). The skill follows the live schema; the creationPrompt behavior it describes (group order, unrequested automations, blank starter values) cannot be checked against public code.
- Recoloring: the toolkit and API schema accept a new `color` on an existing label id, but whether the backend applies it cannot be seen in this repo. The skill keeps a read-back check. A live test would settle it.
- `update_column`'s description points to `get_board_schema` as the preferred source of the revision, and `create_column`'s to `configure_ai_column`; neither tool is in the live list. The skill uses `get_board_info`, which is.
