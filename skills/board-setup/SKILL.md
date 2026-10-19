---
name: board-setup
description: |
  Creates and configures monday.com boards with proper structure including groups,
  columns, and initial items. monday.com boards are flexible enough to manage any
  workflow - project trackers, CRM pipelines, bug trackers, content calendars,
  and more. Use when user asks to "create a board", "set up a project", "new board
  for", "build a tracker", "organize my project", "create a board for tracking",
  or "set up a workflow".
license: MIT
metadata:
  author: monday.com
  version: "0.2.0"
---

# Board Setup

Create and configure monday.com boards with proper structure - the right column types, meaningful groups, and initial items. In monday.com, a board is the central workspace where a team tracks any workflow: project delivery, sales pipeline, bug triage, content production, event planning, and more. This skill provides the domain knowledge needed to build well-organized boards that match how teams actually work.

## Workflow

### Step 1: Understand requirements

Before creating anything, clarify with the user:

1. **What are you tracking?** (tasks, projects, bugs, leads, requests, events, etc.)
2. **What stages does work go through?** (e.g., To Do > In Progress > Review > Done)
3. **What information do you need per item?** (owner, deadline, priority, status, budget, etc.)
4. **How many people will use this board?**

If the user gives a brief request like "create a project board", use reasonable defaults based on the use case and confirm the structure before creating.

### Step 2: Design the board structure

Based on requirements, plan:

- **Board name** - clear and descriptive
- **Groups** - represent workflow stages, categories, or time periods
- **Columns** - the data fields for each item (see Column Type Selection below)

### Step 3: Create the board

**Strongly recommended: build the board in one call with `creationPrompt`.** `create_board` accepts a free-text `creationPrompt`, and monday designs the groups, columns, and labels from it. This is faster and more reliable than building the structure call by call, and avoids the partial boards and duplicate columns that long build sequences leave behind when a call fails midway. Use it for every new board unless the user needs an exact structure the prompt cannot express.

Call `create_board` with:
- `boardName`: descriptive name
- `boardKind`: `"public"` (visible to all team members) or `"private"` (visible only to subscribers)
- `workspaceId` (and `folderId` if the user named one)
- `creationPrompt`: the design from Step 2 in plain language - what the board tracks, the workflow stages, the fields each item needs (with label sets for Priority, Stage, etc.), and any starter items. **Leave automations out of the prompt** - they are not created from it. Add each one afterwards with `create_automation` and verify it with `list_automations`. For example:

  > A bug tracker for a mobile team. Bugs move through New, Triaged, In Progress, Fixed, Verified. Each bug needs an assignee, a severity (Low, Medium, High, Critical), a due date, steps to reproduce, and the affected platform (iOS, Android, Web). Starter items: Login button unresponsive on Android, Crash when uploading a photo.

**After it returns, verify the board against the design - `creationPrompt` gets most of it right but not all.** Call `get_board_info`, `get_board_items_page` (with `includeColumns: true`), and `list_automations`, then check:

1. **Group order and `top_group`.** Groups can come back out of order, with a terminal group such as "Done" on top - and new items land in `top_group`. Reorder with `update_group` (`relative_position_before`, see Step 4) so the first workflow stage is on top.
2. **Column types.** A field can come back as a different type (e.g. a website as `text` instead of `link`). Add a column of the right type with `create_column`, copy any values over, and tell the user the old column can be deleted.
3. **Labels.** Fix a wrong or missing label set (see "Changing labels on an existing status column").
4. **Starter items.** Check every value you asked for - some can be left blank. Fill the gaps in one `update_items` call.
5. **Unrequested automations.** `create_board` can add automations nobody asked for, such as monday AI "categorize" rules on a dropdown, which run on every new item and use AI credits. Tell the user what was added and offer to turn each one off; workflows can be deactivated with `manage_automations`, but some cannot be changed through the API - point the user to the board's Automations panel for those.
6. **The Name column title** - rename it if the design calls for "Deal", "Account", etc.

Fill only the gaps with Steps 4-6. Do not rebuild the board. Tell the user in one line what you corrected.

**If `create_board` fails or times out, do not call it again straight away** - the board may already exist. Call `search` for the board name first and only retry if it is not there; re-sending creates duplicate boards.

Without `creationPrompt` you get an empty board; build it with Steps 4-6.

### Step 4: Create groups

Skip this step if `creationPrompt` already created the groups you need.

Call `create_group` for each workflow stage or category.

**Position the new groups above the default group.** A new monday board ships with a default empty group named "Group Title" (id `topics`). To put your workflow groups in the right order at the top of the board, pass `positionRelativeMethod: "before_at"` and `relativeTo: "topics"` on each `create_group` call. Create groups in workflow order (the first group you create becomes the first group on the board, and `top_group` - which is where new items will land by default).

**Clean up the default group at the end.** Once your workflow groups exist, the default `topics` group is a leftover empty group that confuses users. Delete it by calling `all_monday_api` with the `delete_group` mutation: `mutation { delete_group(board_id: <id>, group_id: "topics") { id } }`. Skipping this leaves the board with a dangling empty group below the workflow.

**Rename or reorder groups** with the `update_group` mutation through `all_monday_api` (there is no `change_group_title` mutation, and `update_group` needs a selection set or it fails):

- Rename: `mutation { update_group(board_id: <id>, group_id: "<group_id>", group_attribute: title, new_value: "New name") { id title } }`
- Move above another group: `group_attribute: relative_position_before, new_value: "<other_group_id>"` (or `relative_position_after`), selecting `{ id position }`.

Renaming the default `topics` group into your first workflow group is an alternative to deleting it.

`create_group` only accepts the exact hex values listed in its `groupColor` enum - a similar shade is rejected.

Common group patterns:

**By workflow stage** (most common):
- "To Do", "In Progress", "In Review", "Done"

**By priority:**
- "Critical", "High Priority", "Medium", "Low Priority"

**By time period:**
- "This Week", "Next Week", "Backlog"

**By team/department:**
- "Engineering", "Design", "Product", "QA"

**By category:**
- "Bugs", "Features", "Improvements", "Technical Debt"

### Step 5: Add columns

If you used `creationPrompt`, only add the columns `get_board_info` shows are missing. Otherwise, call `create_column` for each data field. For each column, specify:
- `boardId`: the board ID from Step 3
- `title`: the display name
- `columnType`: the monday.com column type (see Column Type Selection below)
- `columnSettings`: JSON-stringified label config when the column needs custom labels (see "Custom labels for status columns" below)

**Add columns in this order** (left to right on the board):
1. Status column (workflow state)
2. Person column (owner/assignee)
3. Date or Timeline column (deadline/duration)
4. Priority column (if needed)
5. Additional data columns (text, numbers, links, etc.)

**Renaming the Name (first) column.** `update_column` rejects title changes on the `name` column ("Name columns can only be updated with the width parameter"). Use `all_monday_api` instead: `mutation { change_column_title(board_id: <id>, column_id: "name", title: "Task") { id title } }`. This is how to label it "Deal", "Bug", "Request", etc. to match the template.

#### Custom labels for status columns

`status` columns created without `columnSettings` always get the default monday labels: "Working on it" (orange), "Done" (green), "Stuck" (red). That is fine for the primary workflow Status column, but it is wrong for any other status column the template needs - Priority, Severity, Stage, and Risk all end up with confusing default labels unless you pass custom ones at creation time.

Pass `columnSettings` as a JSON string with the shape:

```json
{"labels": [{"label": "<name>", "color": "<color_token>", "index": <0..n>}, ...]}
```

Each entry needs `label`, `color`, and `index` (other fields are optional). The `color` token is a string from monday's StatusColumnColors enum. Useful tokens by category:

- Calm / low: `sky`, `bright_blue`, `aquamarine`, `saladish`
- Mid / progress: `working_orange`, `egg_yolk`, `purple`
- High / warning: `dark_orange`, `sunset`, `lipstick`
- Critical / blocked: `stuck_red`, `dark_red`, `bright_green` (only for "approved")
- Done / success: `done_green`, `grass_green`

**Reference label sets to use in the templates below:**

- **Priority:** `[{"label":"Low","color":"sky","index":0},{"label":"Medium","color":"working_orange","index":1},{"label":"High","color":"dark_orange","index":2},{"label":"Critical","color":"stuck_red","index":3}]`
- **Severity:** same as Priority.
- **Pipeline stage (CRM):** label set should match the named groups (Lead, Qualified, Proposal, Negotiation, Closed Won, Closed Lost) so item-status and group can stay in sync.
- **Risk:** `[{"label":"Low","color":"saladish","index":0},{"label":"Medium","color":"egg_yolk","index":1},{"label":"High","color":"dark_orange","index":2}]`

If the user requests a status column whose meaning is not workflow ("Priority", "Severity", "Risk", "Stage", "Health", "Confidence", etc.), default to a Low/Medium/High/Critical scale unless they tell you the labels they want. State the defaults you used so they can override.

For the primary workflow Status column, leave `columnSettings` off - the defaults match what most users expect.

**Shape and colors.** Send the flat `{"labels": [...]}` object. `get_column_type_info` shows the schema wrapped in a `settings` key - do not copy that wrapper: `create_column` tolerates it, `update_column` rejects it. `color` must be a token from the list above (or its numeric value); hex codes like `#00c875` are rejected.

**If `create_column` with custom labels returns INTERNAL_SERVER_ERROR,** do not resend the same payload. Create the column without `columnSettings`, then set the labels with `update_column` (below).

#### Changing labels on an existing status column

`update_column` replaces the column's whole label list:

1. Call `get_board_info` with `filters.columns.ids` for the column to get its `revision` and current `settings.labels`.
2. Send **every label you want to keep**, each with its existing `id`, `label`, `color`, and `index`. **A label you leave out is deleted.**
3. Add new labels **without** an `id` - an invented id is rejected ("Label ids: 5 do not exist"). A new label may reuse a color that is already taken.
4. Use the new `revision` from the response for any further update.

Changing the color of an existing label is not supported: the call reports success but the color stays the same. To change a color, tell the user to do it from the column's label editor in monday.

### Step 6: Seed initial items (optional)

If the user provides initial items and the board does not have them yet, create them in one `create_items` call (up to 20 per call) rather than one `create_item` per item. Follow the column value formatting rules from the task-management skill.

### Step 7: Present the result

Report what was built in plain language - don't narrate the API calls. Include the direct URL returned by the MCP tool.

Summarize what was created:

**Board Created: [Board Name]**
- URL: [board URL]
- Type: [public/private]

**Groups:**
1. To Do
2. In Progress
3. In Review
4. Done

**Columns:**
| Column | Type | Purpose |
|--------|------|---------|
| Status | Status | Workflow state |
| Owner | People | Item assignee |
| Due Date | Date | Deadline |
| Priority | Status | Importance level |
| Notes | Text | Additional context |

Then suggest 2-3 logical next steps:
- "Want me to add some initial items to get started?"
- "Should I invite team members to this board?"
- "Want me to create a dashboard to track this project?"
- "Should I set up a form for intake requests?"

## Column Type Selection

Choosing the right column type is critical. Using a text column where you need a status column means losing filtering, grouping, and automation capabilities.

For the complete column type reference, see **`references/column-types-guide.md`**.

Quick selection guide for the most common needs:

| What you need | Column type | Why not text? |
|--------------|-------------|---------------|
| Workflow state (To Do, In Progress, Done) | `status` | Enables color-coding, filtering by status, automations |
| Who's responsible | `people` | Links to monday.com users, enables workload views |
| Deadline | `date` | Enables calendar view, timeline view, overdue detection |
| Date range (start-to-end) | `timeline` | Shows duration bars in timeline view |
| Importance level | `status` (labeled "Priority") | Color-coded, filterable, sortable |
| Effort estimate | `numbers` | Enables sum, average, and formula calculations |
| Category tags | `dropdown` | Multi-select from predefined options |
| Yes/no flag | `checkbox` | Visual toggle, filterable |
| Free-form notes | `text` | Simple text input |
| Long description | `long_text` | Rich text with formatting |
| File attachments | `file` | Upload and preview files inline |
| Web link | `link` | Clickable URL with display text |
| Email address | `email` | Clickable mailto link |
| Phone number | `phone` | Clickable tel link |
| Budget/cost | `numbers` | Supports currency formatting and formulas |
| Rating | `rating` | 1-5 star visual rating |
| Progress | `progress` | Visual progress bar based on subitems or linked column |

## Plan-tier awareness

Most column types work on every monday plan. Two are gated and the skill should handle this gracefully:

- **Formula column** - Pro and Enterprise plans only. If the user is on Free / Basic / Standard and asks for a formula column (e.g., "I want a column that calculates total cost from quantity and price"), explain the gating and suggest one of:
  - A `numbers` column the user updates manually
  - A `text` column with a calculated value the user pastes from elsewhere
  - Adding the formula calculation to a docs/dashboards skill output instead
- **AI / Automation features at the board level** - some are tier-gated. Create automations with `create_automation`, one trigger per call, and read each back with `list_automations` to confirm the trigger and conditions match the request. If the call is rejected because of the plan or trial, say automations need a paid plan and point the user to the board's Automations panel.

If the user is unsure of their plan, just create the board with standard columns first and offer to add formula columns later if their plan supports it.

## Board Templates by Use Case

### Project Tracker
- **Groups:** To Do, In Progress, In Review, Done
- **Columns:** Status, Owner (people), Due Date (date), Priority (status, custom labels Low/Medium/High/Critical), Notes (text)

### Bug Tracker
- **Groups:** New, Triaged, In Progress, Fixed, Verified
- **Columns:** Status, Assignee (people), Severity (status, custom labels Low/Medium/High/Critical), Due Date (date), Steps to Reproduce (long_text), Environment (dropdown)

### Sales/CRM Pipeline
- **Groups:** Lead, Qualified, Proposal, Negotiation, Closed Won, Closed Lost
- **Columns:** Stage (status, custom labels matching the group names: Lead/Qualified/Proposal/Negotiation/Closed Won/Closed Lost), Owner (people), Company (text), Deal Value (numbers), Close Date (date), Source (dropdown)

### Content Calendar
- **Groups:** Ideas, In Production, In Review, Published
- **Columns:** Status, Author (people), Publish Date (date), Content Type (dropdown), Channel (dropdown), Link (link)

### Event Planning
- **Groups:** Planning, Logistics, Marketing, Day-Of, Post-Event
- **Columns:** Status, Owner (people), Timeline (timeline), Budget (numbers), Priority (status), Notes (text)

### Sprint Board
- **Groups:** Backlog, Sprint [current], In Progress, Done
- **Columns:** Status, Assignee (people), Story Points (numbers), Sprint (dropdown), Due Date (date), Type (status)

### Request/Intake
- **Groups:** New Requests, Under Review, Approved, In Progress, Completed
- **Columns:** Status, Requester (people), Assigned To (people), Priority (status), Due Date (date), Request Type (dropdown), Description (long_text)

## Error Prevention and Remediation

- If `create_board` fails due to permissions, suggest using an existing workspace the user has admin access to, or ask which workspace they want to use.
- If a column type is rejected (e.g., formula on a non-Pro plan), explain the plan gating and propose an alternative column type.
- **Detect name collisions before creating.** monday allows two boards with the same name in the same workspace, which is a usability trap. Before calling `create_board`, call `search` with the requested board name scoped to the target workspace. If a board with that name already exists, ask the user: use the existing one, create with a suffix (e.g., "Board Name (2)"), or pick a different name. Skipping this check and creating a duplicate is a fail.
- If `columnSettings` JSON is malformed, `create_column` returns an error. The label format requires an array of `{label, color, index}` objects, not a name->color map. See "Custom labels for status columns" above.
- **Date filters in saved views** (`create_view_table` / `update_view_table`) take different values from item filters. Use `between` with two plain `YYYY-MM-DD` dates, `greater_than` / `lower_than` with `"TODAY"`, or `within_the_next` / `within_the_last` with `["DAYS", 7]`. `any_of` with `["EXACT", "<date>"]` is rejected in views, even though `get_column_type_info` guidelines list it - those guidelines are for `get_board_items_page` filters.

## Tools Used

- `search` - preflight check for board name collisions before `create_board`, and checking whether a failed `create_board` created the board anyway
- `get_column_type_info` - understand available column types and their schemas (including the `columnSettings` shape for status columns)
- `create_board` - create the board, preferably with `creationPrompt` so monday builds the structure in one call
- `create_group` - add groups to the board
- `create_column` - add columns with correct types and custom `columnSettings` for Priority / Severity / Stage / Risk / etc.
- `update_column` - change labels on an existing status column (full label list, existing ids)
- `get_board_info` - read a column's `revision` and current labels before `update_column`
- `create_items` - seed initial items in one call (optional)
- `create_view_table` - saved table views with filters (optional)
- `get_board_items_page` - check starter-item values after `creationPrompt`
- `update_items` - fill starter-item values the generated board left blank
- `create_automation` - add each requested automation after the board exists
- `list_automations` - verify new automations and spot unrequested ones `create_board` added
- `manage_automations` - deactivate an unrequested or wrong automation
- `all_monday_api` - delete the default `topics` group, rename or reorder groups (`update_group`), and rename the Name column (`change_column_title`)

## Additional Resources

- **`references/column-types-guide.md`** - Complete guide to all monday.com column types, when to use each, and configuration options
