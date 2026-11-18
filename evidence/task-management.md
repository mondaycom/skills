# task-management: tool evidence

Verified against mondaycom/mcp @ 1261f96 (agent-toolkit 5.78.11) and the live tools/list on 2026-11-18.

"The review" below is an external check of these skills against the live tool schemas, which this verification started from.

Base: `https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/` (abbreviated `src/` below; `pat/` = `src/core/tools/platform-api-tools/`).

## `get_user_context`

- Skill says: returns recent and favorite boards; full listings come from `workspace_info` (SKILL.md:39, SKILL.md:247).
- Code: no input; returns `user`, `account` (tier, member count, trial, products), `favorites` (boards, folders, workspaces, dashboards), `relevantBoards`/`relevantDocs`/`relevantPeople` capped at 10 each. No account name or slug. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context-tool.ts#L39-L75 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context.graphql.dev.ts#L3-L47
- Verdict: fixed. Tools Used line said "account, and workspaces"; now "favorite workspaces, and plan tier". The "which account am I connected to" check (SKILL.md:243) moved to `list_users_and_teams` `getMe`, because this tool returns no account identity.

## `list_workspaces` / `workspace_info`

- Skill says: get a workspace ID from `list_workspaces`, then `workspace_info` lists up to 100 boards (SKILL.md:39, SKILL.md:248).
- Code: `workspace_info` takes `workspace_id` (number, required) and queries boards/docs/folders with `limit: 100`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/workspace-info-tool/workspace-info-tool.ts#L9-L25 and https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/monday-graphql/queries.graphql.ts#L512-L524 ; `list_workspaces` — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/list-workspace-tool/list-workspace-tool.ts#L26-L37
- Verdict: fixed. Added the `list_workspaces` source of the ID and the 100 cap.

## `search`

- Skill says: `searchType: "BOARD"` + `searchTerm` (SKILL.md:35, SKILL.md:249). `search` cannot list everything (SKILL.md:39).
- Code: served by a monday service outside mondaycom/mcp; checked against live tools/list. The live schema has `required: ["searchType"]` (enum BOARD, DOCUMENTS, FOLDERS, WORKSPACES, UPDATES, ITEMS, TIMELINE_ITEMS, DASHBOARDS), and `searchTerm` has `minLength: 1`. The live description says: "It is required and must be non-empty. This tool has no 'list everything' mode: ... use workspace_info". The toolkit's `search-tool.ts` is not cited: its schema lacks the live `strategy`, `personIds` and `statusLabels` parameters, and its BOARD result fields differ.
- Verdict: fixed. The review's parameters are correct and kept. The review's "for a full list of all boards ... or `search` with `searchType: "BOARD"`" was wrong: search needs a term. That clause is removed.

## `get_board_info`

- Skill says: call first to get column IDs/types, groups and status labels (SKILL.md:22-28, SKILL.md:44-47). Subitem columns come back as `subItemColumns`, null on multi-level boards (SKILL.md:185, SKILL.md:223).
- Code: served by a monday service outside mondaycom/mcp; checked against live tools/list. The live schema is `boardId` (required), `includeKnowledge`, `includeSubscribers` and `filters`. The live description says: "On multi-level boards, subitems share the same columns as parent items and subItemColumns will be null", and it lists status labels and column IDs/types as what to call it for. The toolkit code is cited because its schema and description match the live ones. It returns `columns` (id, title, type, settings, revision), `groups` (id, title), and `subItemColumns` read from the subtasks column's subitems board, skipped on multi-level boards. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info.graphql.ts#L49-L62 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/helpers.ts#L22-L36 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info-tool.ts#L322-L334
- Verdict: fixed. The skill said to call `get_board_info` "on the subitem board". The parent board's response already carries `subItemColumns`.

## `get_column_type_info`

- Skill says: returns a column type's settings schema or filter guidelines, not the value write format (SKILL.md:51, SKILL.md:251, references/column-value-formats.md:192).
- Code: `columnType` enum + `fetchMode` (`schema` default = "JSON settings schema only", `guidelines` = filter/aggregation rules). The description says to call it before `create_column` / `update_column` and before building filters. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-column-type-info/get-column-type-info-tool.ts#L13-L44
- Verdict: fixed. SKILL.md Step 3 told agents to use it for value formats. The review's new note in the reference sent agents to it to "verify the live JSON schema" for writing `rating`/`country`/... values. Both are corrected: it returns column settings, not value shapes.

## `get_board_items_page`

- Skill says: `searchTerm` is a relevance sort, not a strict name filter; cross-check names (SKILL.md:78-81). `includeColumns: true` lists linked items for board relations (references/column-value-formats.md:160).
- Code: served by a monday service outside mondaycom/mcp; checked against live tools/list. Live: `boardId` required, `includeColumns` defaults to `false`, and `searchTerm` is "for vague or approximate wording". "A searchTerm result is capped at 20 items and cannot be paged". The toolkit's `get-board-items-page-tool.ts` is not cited: its schema lacks the live `includeIsLeaf` and `hierarchyScope` parameters.
- Verdict: matches as far as the live description goes. `includeColumns: true` was added to the reference, because the live default is false. The live description neither confirms nor denies the relevance-sort and linked-items claims (open questions).

## `create_item`

- Skill says: `boardId`, `groupId`, `name` (1-255), `columnValues` required, `"{}"` when empty (SKILL.md:55-59, SKILL.md:253). `parentItemId` creates a subitem (SKILL.md:183). With `duplicateFromItemId`, `groupId` is honored (except for a subitem source), the new `name` is applied, and it cannot be combined with `parentItemId` (SKILL.md:204).
- Code: schema `name` min 1 / max 255, `columnValues: z.string()` required, `additionalProperties: false` live. Duplicate passes `itemName` and `groupId` to `duplicate_item` (API 2027-04). When the API appends " (copy)", the tool renames the item and returns `item_name: input.name`. Combining `duplicateFromItemId` with `parentItemId` throws `INVALID_ARGUMENTS_COMBINATION`. The subitem path ignores `boardId`. The response has `item_id`, `item_name`, `item_url`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-item-tool/create-item-tool.ts#L17-L46 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-item-tool/create-item-tool.ts#L80-L164 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-item-tool/duplicate-item.graphql.ts#L3-L19
- Verdict: The review's `name` / `columnValues` fixes match and are kept. The duplicate paragraph is fixed: the old claims that the copy "ignores `groupId`" and "the response reports '<original> (copy)'" are both contradicted by the code.

## `create_items`

- Skill says: up to 20 per call; each item picks its own group/parent/template; per-entry success or failure; retry only failed entries (SKILL.md:200, SKILL.md:254).
- Code: `MAX_ITEMS_PER_CALL = 20`; per-item `groupId`/`parentItemId`/`duplicateFromItemId`/`createLabelsIfMissing`; returns `summary`, `results`, `errors`; `isError` only when all fail; rate-limit skips are coded `RATE_LIMIT_SKIPPED`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-items-tool/create-items-tool.ts#L18-L66 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-items-tool/create-items-tool.ts#L150-L168 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-items-tool/constants.ts#L1-L4
- Verdict: matches.

## `change_item_column_values`

- Skill says: `boardId`, `itemId`, `columnValues` as a JSON string (SKILL.md:89-92). `createLabelsIfMissing` is available here too (references/column-value-formats.md:22). The response returns `linked_item_ids` for board relations (references/column-value-formats.md:160).
- Code: schema + `createLabelsIfMissing`; rejects non-object JSON with `INVALID_COLUMN_VALUES_JSON`; unwraps double-encoded strings. The response maps board_relation/dependency columns to `linked_item_ids`, or to a count above 1,000. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/change-item-column-values-tool.ts#L15-L35 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/change-item-column-values-tool.ts#L85-L145 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/monday-graphql/queries.graphql.ts#L50-L78
- Verdict: fixed. The reference said `createLabelsIfMissing` was only on the batch entries, and that the write response shows `null` for board relations.

## `update_items`

- Skill says: up to 40 updates, can span boards (SKILL.md:200, SKILL.md:256).
- Code: `MAX_UPDATES_PER_CALL = 40`; per-update `boardId`, with missing ones resolved from `itemId`. Each update runs through `ChangeItemColumnValuesTool`, so it returns the same `linked_item_ids`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/update-items-tool/update-items-tool.ts#L29-L58 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/update-items-tool/update-items-tool.ts#L146-L188 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/update-items-tool/constants.ts#L1
- Verdict: matches (the response claim is fixed in the reference, see above).

## Column value formats (shared `COLUMN_VALUES_FORMAT_GUIDE`)

- Skill says: status `{"label"}`/`{"index"}`, dropdown `{"labels": [...]}`/`{"ids": [...]}`, date, timeline, people with kind person/team/agent, board_relation `item_ids`, tags `tag_ids`, checkbox `{"checked": "true"}`, link/email with `text`, location with string lat/lng, phone digits + ISO-2, `null` clears (SKILL.md:112-167, references/column-value-formats.md:7-188, 258-264).
- Code: the same shapes, plus "null clears a column". — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/column-values.utils.ts#L1-L13
- Verdict: matches. The review's change from `""` to `null` for clearing is kept. `rating`, `country`, `world_clock`, `week`, `hour`, `color_picker` and date-with-`time` are not in the guide (see open questions).

## `create_update`

- Skill says: `itemId`, `body` in HTML, not markdown; mention with `mentionsList`, not `@name` (SKILL.md:189-192).
- Code: `body` described as "Do not use @ to mention users, use the mentionsList field ... use html tags ..., dont use markdown"; `mentionsList` is a JSON string of `{id, type}` with types User/Team/Board/Project. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-update-tool/create-update-tool.ts#L24-L43
- Verdict: fixed. Added `mentionsList` and the "not markdown" note.

## `list_users_and_teams`

- Skill says: pass `name` or `userIds`; no-parameter calls return the whole account (SKILL.md:198, SKILL.md:233). `getMe: true` returns the user's email and account URL (SKILL.md:243, SKILL.md:259).
- Code: `name` and `getMe` are standalone (`PARAMETER_CONFLICT` otherwise); no-parameter calls fetch up to 1,000 users. `getMe` returns full user details (including `email`) and `url: https://<slug>.monday.com/teams/all`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/list-users-and-teams-tool/list-users-and-teams-tool.ts#L126-L193 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/list-users-and-teams-tool/constants.ts#L1-L9
- Verdict: fixed. This tool is now used for the account check.

## `move_item_to_group` / `all_monday_api`

- Skill says: if a `move_item_to_group` tool is listed, call it with `itemId`, `groupId` (same board only); otherwise use the `all_monday_api` mutation; `move_object` is not for items (SKILL.md:202, SKILL.md:258).
- Code: toolkit 5.78.11 ships `move_item_to_group` (`itemId`, `groupId`; "cannot move an item to a different board"), but it is **not** in the live tools/list. `all_monday_api` validates the operation against the introspected schema, and its description points to `get_graphql_schema` / `get_type_details`. `move_object` handles folder, board and overview only. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/move-item-to-group-tool.ts#L12-L31 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/all-monday-api-tool.ts#L51-L53 , https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/move-object-tool/move-object-tool.ts#L10-L54
- Verdict: fixed. Removed "the MCP toolkit has no direct move item tool" and made the dedicated tool conditional. Code and live list disagree: the tool exists in code but is not exposed live.

## `link_board_items_workflow`

- Skill says: use it if listed, otherwise write `item_ids` directly (references/column-value-formats.md:158).
- Code: exists in the toolkit (`pat/link-board-items-workflow-tool/link-board-items-workflow-tool.ts`) and is referenced by the `update_items` / `change_item_column_values` descriptions; absent from the live tools/list.
- Verdict: matches (already conditional).

## `show-assign` / `show-table`

- Skill says: use `show-assign` only when it is in the tool list and the user wants help choosing assignees; otherwise text. Do not use `show-table` for a single confirmation (SKILL.md:98-102, SKILL.md:260).
- Code: live tools/list. `show-assign` "Renders an interactive smart assignment interface", input `title` + `assignments[{itemId, itemName, user{id, name, avatarUrl, jobTitle}}]`; `show-table` is a UI table widget with MCP Apps / OpenAI `_meta`.
- Verdict: restored. The review deleted the section; it is back, phrased conditionally, with the text fallback.

## `submit_bug_or_feature_request`

- Skill says: offer to report a persistent write failure only if the tool is in the tool list, otherwise suggest monday.com support (SKILL.md:241, SKILL.md:261).
- Code: `pat/submit-bug-or-feature-request-tool/submit-bug-or-feature-request-tool.ts`; present in the live tools/list.
- Verdict: restored conditionally. The review's support fallback is kept.

## Open questions

- Formats the shared guide does not cover, so code cannot confirm them: `rating`, `country`, `world_clock`, `week`, `hour`, `color_picker`, date with `time`, and `{"checked": true}` (boolean) (references/column-value-formats.md:32-35, 170, 194-240).
- Plain-string behavior: the guide says plain strings fail for email and phone. The skill also says this of link, and says a digits-only phone string "saves with no country" (SKILL.md:167, references/column-value-formats.md:116, 126). That is API behavior, not tool code.
- API behaviors the tools pass through unchecked: the lazy subitems-board race (SKILL.md:223), duplicate dropdown labels with `createLabelsIfMissing` in a batch (SKILL.md:225), formula/mirror plan gating (references/column-value-formats.md:277-282), and the write-failure signatures "Resource not found" / `USER_UNAUTHORIZED` / 403 (SKILL.md:241).
- `move_item_to_group` ships in toolkit 5.78.11 but is not in the live tools/list. The skill handles both cases.
- `get_board_items_page` is served outside mondaycom/mcp. Two claims cannot be checked against code: that `searchTerm` returns relevance-ranked non-matches instead of an empty result (SKILL.md:79), and that board-relation columns list linked items with `includeColumns: true` (references/column-value-formats.md:160).
