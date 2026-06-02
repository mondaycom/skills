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
  version: "0.1.0"
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

Call `create_board` with:
- `boardName`: descriptive name
- `boardKind`: `"public"` (visible to all team members) or `"private"` (visible only to subscribers)

### Step 4: Create groups

Call `create_group` for each workflow stage or category.

**Position the new groups above the default group.** A new monday board ships with a default empty group named "Group Title" (id `topics`). To put your workflow groups in the right order at the top of the board, pass `positionRelativeMethod: "before_at"` and `relativeTo: "topics"` on each `create_group` call. Create groups in workflow order (the first group you create becomes the first group on the board, and `top_group` - which is where new items will land by default).

**Clean up the default group at the end.** Once your workflow groups exist, the default `topics` group is a leftover empty group that confuses users. Delete it by calling `all_monday_api` with the `delete_group` mutation: `mutation { delete_group(board_id: <id>, group_id: "topics") { id } }`. Skipping this leaves the board with a dangling empty group below the workflow.

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

Call `create_column` for each data field. For each column, specify:
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

#### Custom labels for status columns

`status` columns created without `columnSettings` always get the default monday labels: "Working on it" (orange), "Done" (green), "Stuck" (red). That is fine for the primary workflow Status column, but it is wrong for any other status column the template needs - Priority, Severity, Stage, and Risk all end up with confusing default labels unless you pass custom ones at creation time.

Pass `columnSettings` as a JSON string with the shape:

```json
{"labels": [{"label": "<name>", "color": "<color_token>", "index": <0..n>}, ...]}
```

Each entry needs `label`, `color`, and `index` (other fields are optional). The `color` token is a string from monday's StatusColumnColors enum. Useful tokens by category:

- Calm / low: `sky`, `bright_blue`, `aquamarine`, `saladish`
- Mid / progress: `working_orange`, `orange`, `egg_yolk`, `purple`
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

### Step 6: Seed initial items (optional)

If the user provides initial items, call `create_item` for each one. Follow the column value formatting rules from the task-management skill.

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
- **AI / Automation features at the board level** - some are tier-gated. If the user asks the skill to set up an automation, redirect them to the monday automation UI rather than promising it via the plugin.

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

## Tools Used

- `search` - preflight check for board name collisions before `create_board`
- `get_column_type_info` - understand available column types and their schemas (including the `columnSettings` shape for status columns)
- `create_board` - create the board
- `create_group` - add groups to the board
- `create_column` - add columns with correct types and custom `columnSettings` for Priority / Severity / Stage / Risk / etc.
- `create_item` - seed initial items (optional)
- `all_monday_api` - delete the default `topics` group after the workflow groups are in place

## Additional Resources

- **`references/column-types-guide.md`** - Complete guide to all monday.com column types, when to use each, and configuration options
