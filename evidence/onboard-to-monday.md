# onboard-to-monday: tool evidence

Verified against mondaycom/mcp @ 1261f96 (agent-toolkit 5.78.11) and the live tools/list on 2026-11-18.

"The review" below is an external check of these skills against the live tool schemas, which this verification started from.

Base path for links below: `https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/`.

## `get_user_context`

- Skill says: returns the user's name and title, plan tier and trial status, favorites, and up to 10 recently used boards, not a full list of workspaces or boards (SKILL.md:33, 128).
- Code: the query reads `me { id name title account { tier active_members_count is_during_trial products } }`, `favorites`, and `intelligence.relevant_boards(limit: 10)`; output is `user`, `account`, `favorites`, `relevantBoards`, `relevantDocs`, `relevantPeople`. No account name, no workspace list; workspaces appear only if favorited. Takes no input. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context.graphql.dev.ts#L3-L46, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/user-context-tool/user-context-tool.ts#L57-L75
- Verdict: fixed. The previous text (and the review's version) said it returns "the user's account, workspaces"; it returns neither an account name nor a workspace list. Step 1 now names who the user is connected as instead of "which account" (SKILL.md:35).
- Skill says: note plan tier and trial status for Step 6 (SKILL.md:40).
- Code: `tier` and `is_during_trial` are returned (same lines). — Verdict: matches.

## `list_workspaces`

- Skill says: call it to pick the workspace for the new board; the user's own workspaces come first (SKILL.md:33, 129).
- Code: lists workspaces where the user is a member first, falls back to all workspaces if none; returns `id`, `name`, `description`, `url`. In live tools/list. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/list-workspace-tool/list-workspace-tool.ts#L36-L69
- Verdict: fixed (added). Step 4 passes a workspace "from Step 1", which `get_user_context` cannot supply.

## `workspace_info`

- Skill says: list all boards in a workspace by `workspace_id` (SKILL.md:33, 130).
- Code: required `workspace_id` (number); returns boards, docs and folders, up to 100 of each. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/workspace-info-tool/workspace-info-tool.ts#L10-L24
- Verdict: matches. The review's alternative "or `search` with `searchType: "BOARD"`" for a complete list was dropped: `search` requires a non-empty `searchTerm` and has no list-everything mode (see `search`).

## `search`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: `searchType: "BOARD"` and `searchTerm: "<board name>"` for the collision check and the post-failure check (SKILL.md:69, 74, 131).
- Code: `searchType` is an enum (BOARD, DOCUMENTS, FOLDERS, WORKSPACES, UPDATES, ITEMS, TIMELINE_ITEMS, DASHBOARDS); `searchTerm` must be non-empty; limit max 20; BOARD returns id, title, url, workspaceId. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/search-tool/search-tool.ts#L189-L198, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/search-tool/search-tool.ts#L234-L240
- Verdict: matches (the review's fix kept).

## `create_board`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: `boardName`, `boardKind: "public"` or `"private"`, `workspaceId` as a string, and `creationPrompt` (SKILL.md:70).
- Code: live tools/list: required `boardName`, `boardKind` (enum `private`/`public`/`share`, default `public`); optional `workspaceId` (string), `folderId`, `boardDescription`, `boardOwnerIds`, `creationPrompt` ("Omit to create an empty board"), `useMlsTemplate`, `useDatasetTemplate`. mondaycom/mcp @ 1261f96 has the same core fields but no `creationPrompt`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-board-tool.ts#L11-L19; `creationPrompt`: live tools/list
- Verdict: fixed (named `workspaceId` and its string type; was "the workspace from Step 1").
- Skill says: share the board URL from the `create_board` response (SKILL.md:87, 100).
- Code: returns `board_id`, `board_name`, `board_url`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-board-tool.ts#L50-L54
- Verdict: matches (see Open questions).

## `get_board_info`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: read back generated columns, groups and labels (SKILL.md:78, 133).
- Code: required `boardId` (number); query returns columns and `groups { id title }`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-info/get-board-info.graphql.ts#L55-L62
- Verdict: matches.

## `get_board_items_page`
- Served by a monday service outside mondaycom/mcp; checked against live tools/list. Toolkit links are the open-source version of the same tool, cited only where its schema matches the live one.

- Skill says: call with `includeColumns: true`; re-read after the user's first change when `show-table` is not available (SKILL.md:78, 104, 134).
- Code: `includeColumns` boolean, default false; group included by default. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/get-board-items-page-tool/get-board-items-page-tool.ts#L90
- Verdict: matches.

## `all_monday_api`

- Skill says: `update_group(board_id, group_id, group_attribute: relative_position_before, new_value: "<current top group id>") { id }`; on rejection check `get_graphql_schema` / `get_type_details` (SKILL.md:80).
- Code: tool takes `query` and `variables` (JSON string). The schema defines `update_group(board_id: ID!, group_attribute: GroupAttributes!, group_id: String!, new_value: String!): Group` and `GroupAttributes` includes `relative_position_before`. Both helper tools are in live tools/list. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/all-monday-api-tool.ts#L20-L21, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/monday-graphql/schema.graphql#L670-L673, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/monday-graphql/schema.graphql#L11008-L11019
- Verdict: matches (the review's fallback kept).

## `create_column`

- Skill says: add a missing key column, e.g. `link` instead of `text` (SKILL.md:81, 137).
- Code: required `boardId`, `columnType` (enum includes `link`, `text`), `columnTitle`; optional `columnSettings`, `columnDescription`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-column-tool.ts#L9-L20
- Verdict: matches.

## `create_items`

- Skill says: create missing starter items in one call (SKILL.md:82, 138).
- Code: `boardId` and `items[]` (each `name`, `columnValues` string, optional `groupId`), max 20 per call. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-items-tool/create-items-tool.ts#L22-L69, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/create-items-tool/constants.ts#L1
- Verdict: matches (3-5 starter items fit one call).

## `update_items`

- Skill says: fill blank values in one call (SKILL.md:82, 136).
- Code: `updates[]` of `{ itemId, columnValues, boardId? }`, max 40 per call; live description: "null clears a column". — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/update-items-tool/update-items-tool.ts#L33-L73, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/update-items-tool/constants.ts#L1
- Verdict: matches. The skill does not clear values, so the null-vs-"" fix does not apply here.

## `create_automation`

- Skill says: `boardId` plus a Trigger / Conditions / Actions description; on `needs_clarification` ask and call again (SKILL.md:93).
- Code: required `userPrompt` and `boardId` (both non-empty strings); the description documents the format and `status: "needs_clarification"`; the response body is returned as is, and non-2xx becomes `UPSTREAM_HTTP_ERROR`. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/create-automation/create-automation-tool.ts#L11-L18, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/create-automation/create-automation-tool.ts#L53, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/create-automation/create-automation-tool.ts#L125-L138
- Verdict: matches. The text says "structured description" and does not name `userPrompt` (left as is).
- Skill says: skip it if it is rejected because of the plan or trial (SKILL.md:96).
- Code: no plan check in the tool; any rejection comes from the upstream body. — same lines. — Verdict: open question.

## `list_automations`

- Skill says: verify trigger and conditions; source of ids for `manage_automations` (SKILL.md:78, 83, 94, 140).
- Code: required `boardId` (string); returns `workflows` (manageable) and `legacyAutomations` (read-only, first page only); "the only way to resolve an automation id". — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/list-automations/list-automations-tool.ts#L50-L75
- Verdict: fixed (Step 5 now says to take the id from `list_automations` and, when one cannot be changed, to point the user to the Automations panel. This covers read-only automations without labelling them, as the tool description asks).

## `manage_automations`

- Skill says: deactivate with `action: "deactivate"`; on `USER_UNAUTHORIZED` do not retry (SKILL.md:83, 94, 95, 141).
- Code: required `action` (`activate`/`deactivate`/`delete`) and `workflowId`; deactivate calls `deactivate_live_workflow`. API errors are rethrown with their `extensions.code` in the message, so an upstream `USER_UNAUTHORIZED` reaches the model. — https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/manage-automations/manage-automations-tool.ts#L20-L32, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/core/tools/platform-api-tools/automations-tools/manage-automations/manage-automations-tool.ts#L118-L141, https://github.com/mondaycom/mcp/blob/1261f96/packages/agent-toolkit/src/utils/error.utils.ts#L36-L59
- Verdict: matches (the step 5 use was added; the error code itself is the API's, not the toolkit's).

## `show-table`

- Skill says: if `show-table` is in the tool list, call it with `boardId` as a string and no `filters`; otherwise, or on failure, share the board URL with a Markdown list of the items; call it again after the user's first change (SKILL.md:87, 100, 104, 142).
- Code: live tools/list: required `boardId` (string); optional `filters`, `filtersOperator`, `currentlySelectedItemIdForShowingUpdates`; `[UI COMPONENT] Renders an interactive table`; "After adding an update to an item, you MUST display the table AGAIN". Not in mondaycom/mcp.
- Verdict: fixed. Restored as conditional. The review removed it because some clients do not expose it; clients that render it keep the table, and others get the review's Markdown and URL fallback.

## Invite teammates / Slack follow-ups

- Skill says: suggest sharing the board link with teammates (SKILL.md:109).
- Code: live tools/list has no tool to invite users to an account or board (`list_users_and_teams` only reads users and teams). There are no Slack tools.
- Verdict: matches (the review's removal kept).

## Open questions

- `create_board`: the live schema carries `creationPrompt` and `folderId`, which mondaycom/mcp @ 1261f96 does not have. So the response field names (`board_url`) come from the open-source handler and were not checked against the live response.
- `create_automation` plan or trial rejection (SKILL.md:96) and the claim that `create_board` can add unrequested automations (SKILL.md:83) depend on upstream behaviour. Neither shows in toolkit code.
