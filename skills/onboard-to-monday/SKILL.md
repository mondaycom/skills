---
name: onboard-to-monday
description: |
  First-run onboarding for people who just connected monday.com and have not
  set anything up yet. Builds a ready-to-use board tailored to the user's work
  in a few minutes, using what you already know about them to skip generic
  questions. Use only when the user asks to "onboard me to monday", "get me
  started with monday.com", "set up monday for me", "help me get started",
  or sends the post-connect starter prompt. Do not use for requests about an
  existing board, item, doc, or workflow - use the matching skill instead.
license: MIT
metadata:
  author: monday.com
  version: "0.1.0"
---

# Onboard to monday.com

Get a brand-new monday.com user from "just connected" to a working board they recognize as their own, in one short conversation. The goal is a first win in minutes: a board built around the user's real work, with real items, one automation, and a clear next action.

The board itself is built by `create_board` with a `creationPrompt` - monday designs the groups and columns from a free-text description. This skill decides *what* to ask for, then checks the result and fills any gaps. Use the **task-management** skill for column value formats and write-failure handling.

## Principles

- **Propose first, ask second.** Do not open with a questionnaire. Draft a setup from what you already know, then ask one confirmation question.
- **Be transparent about what you inferred.** Say what you are basing the proposal on so the user can correct it.
- **Work context only.** Use the user's role, team, company, current projects, and tools. Never put personal or sensitive details (health, family, finances, private conversations) into the creation prompt, board names, items, or descriptions.
- **Fast.** Aim for one confirmation, one build, one first action. Use smart defaults and state them; do not ask about things that have a reasonable default.

## Workflow

### Step 1: Confirm this is a new user

Call `get_user_context`. Tell the user, in one line, which account and workspace they are connected to.

**The user asked to be onboarded - honor that.** Run onboarding unless there is strong evidence they already use monday.com for real work. Having boards is not that evidence: new accounts come with boards monday created at signup.

**Ignore starter boards.** A board does not count as real usage if:

- its name is a starter name such as "My first project", "My first board", "Getting started", "Welcome to monday", or "Start here", or
- it came from a signup template - check it with `get_board_items_page` (`limit: 10`, no columns): items with generic or sample names ("Project 1", "Task 1", "Item 1"), all created within minutes of each other, and none edited since (`updated_at` close to `created_at`).

Check at most two boards this way, so the step stays fast.

Then decide:

- **New user** - for example a trial or 1-2 active members, no relevant docs or people, and no boards left once starter boards are excluded: continue with Step 2. If it is unclear, treat the user as new.
- **Established user** - several signals together: boards with real activity (items edited after they were created, or many items), several active members, relevant docs or people, a paid plan. One board, or a couple of starter boards, is never enough. Do not stop: name one or two of their real boards and offer to set up a new board for work they are not tracking yet, or to help with an existing one - e.g. "You already run Sales Pipeline and Hiring here. Want me to set up a board for something you're not tracking yet, or work on one of those?"
- **Unexpected account** (the user says it is not the right one, or expected boards are missing): stop and ask them to disconnect and reconnect the monday.com connector, choosing the right account.

**Never describe starter boards as "a starting setup" or a reason to skip onboarding.** Mention them lightly at most - "I see the sample boards monday added when you signed up - let's build one around your actual work" - and continue. A signup template board (e.g. "Travel Expense Tracker") shows what the user picked at signup; use it as a hint in Step 2 and Step 3.

Note the plan tier and trial status for Step 6.

### Step 2: Build a picture of the user's work

Use what you already know about the user from this conversation and from earlier conversations or memory, if available: their role, team, company or industry, and what they are working on now (projects, launches, clients, hiring, events).

A signup template board from Step 1 is a hint too: confirm it rather than assume it - "You picked travel expenses when you signed up - is that what you want to track, or something else?"

- **If you know enough** to guess what they would track, go straight to Step 3.
- **If you know little or nothing,** ask at most three short questions, in one message:
  1. What kind of work do you want to manage here? (e.g. projects, clients/deals, content, requests, hiring, an event)
  2. What is one real thing you are working on right now?
  3. Is this just for you, or for a team?

### Step 3: Propose a tailored setup

Decide on:

- **Board name** from their real work (e.g. "Q4 Product Launch", "Acme Client Pipeline"), not "My Board".
- **What the board tracks and how work flows** (stages, the key fields per item - owner, due date, priority, value, etc.).
- **3-5 starter items from their actual work** - real milestones, clients, or tasks you know about or that they just mentioned - never "Task 1". Spread due dates over the next two weeks.
- **One starter automation** (see "Starter automations" below).

**Show the proposal as a table with `show-table-preview`.** It renders the same interactive table as `show-table` from mock data - nothing is created in monday.com - so the user sees their board before you build it. Pass one `board` object:

- `name` - the proposed board name.
- `columns` - up to 20, each with a unique `id` (e.g. `"status"`, `"owner"`), a `title`, and a `type`: `text`, `numbers`, `date`, `status`, `dropdown`, `timeline`, `checkbox`, or `people`. Give `status` and `dropdown` columns their `labels`. Do not include the Name column - it is always shown first. Map other field types to the closest preview type (a link or email as `text`, a priority as `status`).
- `items` - the 3-5 starter items, each with a `name` and `values` keyed by column `id`: a status takes one label string, a dropdown an array of labels, a date `"YYYY-MM-DD"`, a timeline `{"from": "...", "to": "..."}`, a checkbox a boolean, numbers a number, and people an array of names (shown as text; real assignments happen after the build).

For example:

```json
{"board": {
  "name": "Q4 Launch Tracker",
  "columns": [
    {"id": "status", "title": "Status", "type": "status", "labels": ["To Do", "In Progress", "In Review", "Done"]},
    {"id": "owner", "title": "Owner", "type": "people"},
    {"id": "due", "title": "Due Date", "type": "date"},
    {"id": "priority", "title": "Priority", "type": "status", "labels": ["Low", "Medium", "High", "Critical"]}
  ],
  "items": [
    {"name": "Finalize messaging", "values": {"status": "In Progress", "owner": ["Dana"], "due": "2026-10-27", "priority": "High"}},
    {"name": "Launch landing page", "values": {"status": "To Do", "due": "2026-10-30", "priority": "Critical"}},
    {"name": "Partner email", "values": {"status": "To Do", "due": "2026-11-03", "priority": "Medium"}}
  ]
}}
```

Then, in one short message under the preview, say what you based it on, name the starter automation, and ask one question, for example:

> Here's a preview based on what I know - you lead marketing at Acme and you are planning the Q4 launch. Each task moves through To Do / In Progress / In Review / Done, and I'd add an automation that notifies the owner when a task gets stuck. Should I build it, or change anything first?

If they change something, update the proposal and build - call `show-table-preview` again only if the structure changed (columns or stages), not for small edits. Do not re-confirm after small changes.

If `show-table-preview` is not in your tool list or fails, present the same proposal as text instead - board name, stages, key fields, starter items, and the automation - and ask the same question.

### Step 4: Create the board

1. **Check for a name collision.** Call `search` with the board name. If a board with that name exists, ask whether to use it or create the new one with a different name.
2. **Call `create_board`** with `boardName`, `boardKind: "public"` (or `"private"` if the user asked), the workspace from Step 1, and a `creationPrompt` that describes the board in plain language: what it tracks, the workflow stages, the fields each item needs, and the starter items. Leave the automation out - it is not created from the prompt; Step 6 adds it. Write the prompt from the agreed proposal, for example:

   > A launch tracker for a marketing team planning a Q4 product launch. Work moves through To Do, In Progress, In Review, Done. Each task needs an owner, a due date, and a priority (Low, Medium, High, Critical). Starter tasks: Finalize messaging, Launch landing page, Partner email, Launch day checklist.

3. **If `create_board` fails or times out, do not call it again straight away** - the board may have been created. Call `search` for the board name first, and only retry if it does not exist. Re-sending creates duplicate boards.

### Step 5: Check the result, fill gaps, and show the live board

`creationPrompt` gets most of the board right but not all. Call `get_board_info`, `get_board_items_page` (`includeColumns: true`), and `list_automations`, and compare with what you proposed:

- **Group order** - if a later or terminal group (e.g. "Done") is on top, new items land there. Move the first workflow stage to the top with `all_monday_api`: `update_group(board_id, group_id, group_attribute: relative_position_before, new_value: "<current top group id>") { id }`.
- **Columns** - if a key field is missing or has the wrong type (e.g. a website as `text` instead of `link`), add it with `create_column`. Do not rebuild the board.
- **Items** - if starter items are missing, create them in one `create_items` call; if values you asked for are blank, fill them in one `update_items` call. Follow the task-management column formats. Assign items to the current user only if they agreed.
- **Unrequested automations** - `create_board` can add automations nobody asked for (e.g. monday AI "categorize" rules that use AI credits). Mention them in one line and offer to turn them off.

If a write tool fails after a successful read, follow the task-management rule: retry once, then stop and ask the user to reconnect the connector. Do not keep rebuilding.

**Show the real board right away.** As soon as the gaps are fixed, call `show-table` with the new board's `boardId` - always, even though the user saw the Step 3 preview. The preview was a mock-up; this is their live board, and seeing it is the payoff of the build. Do not pass `filters`; show the whole board. Do this before the automation step, not after it.

Under the table, add one short line with the board URL from the tool response, e.g. "Your Q4 Launch Tracker is live: <url> - adding the starter automation next." If the real board differs from the preview in a way the user would notice (a renamed stage, a missing column you could not add), say so in the same line. If `show-table` is unavailable or fails, share the URL and continue.

### Step 6: Add the starter automation

Ask before creating it (it can be part of the Step 3 confirmation).

1. Call `create_automation` with the `boardId` and a structured description (Trigger / Conditions / Actions) in the user's terms, using the real column and group names from Step 5. If it returns `needs_clarification`, ask for the missing details and call it again.
2. **Read it back.** Call `list_automations` for the board and check that the trigger and conditions match the request - in particular that a "status changes to X" trigger does not also require the previous status to be X, and that conditions are not inverted. If it is wrong, deactivate it with `manage_automations` (`action: "deactivate"`) and recreate it.
3. If `manage_automations` returns `USER_UNAUTHORIZED` for an automation you just created, do not retry - tell the user to turn it off from the board's Automations panel.
4. If `create_automation` is rejected because of the plan or trial, skip it, mention automations are available on paid plans, and continue. Do not let this block the board.

### Step 7: Hand over with a first win

The live board is already on screen from Step 5 - do not call `show-table` again here.

1. **Summarize in a few lines:** what was built, the automation (if created), and the board URL from the tool response. Do not repeat the items as a text list - the table already shows them. (If `show-table` failed in Step 5, list the starter items here instead.)
2. **Give one concrete thing to try right now,** phrased as something they can just say, using one of their own items:
   - "Try it: say **mark Finalize messaging as done**."

   When they do, make the change (task-management skill), then call `show-table` again so they see the board update.

Then offer, briefly, the habits that bring people back:

- "Any time, ask me **what's overdue** or **give me a status update** on this board."
- "Want me to invite your teammates so they can work on this with you?" (only if they said it is for a team)

Stop there. Do not chain further setup unless the user asks.

## Starter automations

Pick the one that fits the board, using its real column, label, and group names:

| Board type | Starter automation |
|---|---|
| Projects / tasks / sprints | When Status changes to Done, move the item to the Done group |
| Clients / deals / CRM | When Stage changes to Closed Won, move the item to the Closed Won group |
| Content | When the publish date arrives, notify the author |
| Requests / intake | When a new item is created, notify the assignee |
| Bugs / issues | When Severity changes to Critical, notify the assignee |
| Events / anything else | When Status changes to Stuck, notify the owner |

## Tools Used

- `get_user_context` - account, workspace, plan and trial status, active members, and the user's boards, docs, and people
- `get_board_items_page` - tell a signup starter board (sample items, never edited) from real usage (Step 1), and check the starter items' values (Step 5)
- `search` - board name collision check, and checking whether a failed `create_board` actually created the board
- `create_board` - build the board from a `creationPrompt`
- `get_board_info` - read back the generated columns, groups, and labels
- `all_monday_api` - reorder groups (`update_group`) when the first stage is not on top
- `update_items` - fill starter-item values the generated board left blank
- `create_column` - add a key column the generated board is missing
- `create_items` - add the starter items if the generated board does not have them
- `create_automation` - create the starter automation
- `list_automations` - verify the automation's trigger and conditions
- `manage_automations` - deactivate an automation that was created wrong
- `show-table-preview` - render the proposed board from mock data before anything is created (Step 3)
- `show-table` - render the real board right after it is built (Step 5), and again after the user's first change (Step 7)
