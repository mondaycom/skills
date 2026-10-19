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
  version: "0.3.0"
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

Call `get_user_context`.

- Tell the user, in one line, which account and workspace they are connected to.
- **New user** (trial or recently created account, few or no boards of their own): continue with Step 2.
- **Existing user** (the account already has boards they use): do not run onboarding. Say they already have boards set up, name a few of their most relevant ones, and offer to help with those instead (status update, add items, new board).
- **Unexpected account** (the user says it is not the right one, or expected boards are missing): stop and ask them to disconnect and reconnect the monday.com connector, choosing the right account.

Note the plan tier and trial status for Step 6.

### Step 2: Build a picture of the user's work

Use what you already know about the user from this conversation and from earlier conversations or memory, if available: their role, team, company or industry, and what they are working on now (projects, launches, clients, hiring, events).

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

Present it compactly and ask one question, for example:

> Based on what I know - you lead marketing at Acme and you are planning the Q4 launch - I would set up a **Q4 Launch Tracker** that tracks each launch task through To Do / In Progress / In Review / Done, with an owner, due date and priority, starting with: Finalize messaging, Launch landing page, Partner email, Launch day checklist - plus an automation that notifies the owner when a task gets stuck. Should I build it, or change anything first?

If they change something, adjust and build. Do not re-confirm after small changes.

### Step 4: Create the board

1. **Check for a name collision.** Call `search` with the board name. If a board with that name exists, ask whether to use it or create the new one with a different name.
2. **Call `create_board`** with `boardName`, `boardKind: "public"` (or `"private"` if the user asked), the workspace from Step 1, and a `creationPrompt` that describes the board in plain language: what it tracks, the workflow stages, the fields each item needs, and the starter items. Write the prompt from the agreed proposal, for example:

   > A launch tracker for a marketing team planning a Q4 product launch. Work moves through To Do, In Progress, In Review, Done. Each task needs an owner, a due date, and a priority (Low, Medium, High, Critical). Starter tasks: Finalize messaging, Launch landing page, Partner email, Launch day checklist.

3. **If `create_board` fails or times out, do not call it again straight away** - the board may have been created. Call `search` for the board name first, and only retry if it does not exist. Re-sending creates duplicate boards.

### Step 5: Check the result and fill gaps

Call `get_board_info` on the new board. Compare it with what you proposed:

- **Columns** - if a key field from the proposal is missing (e.g. no owner or due date column), add it with `create_column`. Do not rebuild the board.
- **Items** - if the starter items are missing, create them in one `create_items` call, following the task-management column formats. Assign them to the current user only if they agreed.

If a write tool fails after a successful read, follow the task-management rule: retry once, then stop and ask the user to reconnect the connector. Do not keep rebuilding.

### Step 6: Add the starter automation

Ask before creating it (it can be part of the Step 3 confirmation).

1. Call `create_automation` with the `boardId` and a structured description (Trigger / Conditions / Actions) in the user's terms, using the real column and group names from Step 5. If it returns `needs_clarification`, ask for the missing details and call it again.
2. **Read it back.** Call `list_automations` for the board and check that the trigger and conditions match the request - in particular that a "status changes to X" trigger does not also require the previous status to be X, and that conditions are not inverted. If it is wrong, deactivate it with `manage_automations` (`action: "deactivate"`) and recreate it.
3. If `manage_automations` returns `USER_UNAUTHORIZED` for an automation you just created, do not retry - tell the user to turn it off from the board's Automations panel.
4. If `create_automation` is rejected because of the plan or trial, skip it, mention automations are available on paid plans, and continue. Do not let this block the board.

### Step 7: Hand over with a first win

1. **Show the board.** Call `show-table` with the new board's `boardId` so the user sees their board - with their starter items - as an interactive table right in the chat. Do not pass `filters`; show the whole board.
2. **Summarize in a few lines** below the table: what was built, the automation (if created), and the board URL from the tool response. Do not repeat the items as a text list - the table already shows them.
3. **Give one concrete thing to try right now,** phrased as something they can just say, using one of their own items:
   - "Try it: say **mark Finalize messaging as done**."

   When they do, make the change (task-management skill), then call `show-table` again so they see the board update.

If `show-table` is unavailable or fails, skip it and list the starter items in plain text with the board URL instead.

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

- `get_user_context` - account, workspace, plan and trial status, and whether the user already has boards
- `search` - board name collision check, and checking whether a failed `create_board` actually created the board
- `create_board` - build the board from a `creationPrompt`
- `get_board_info` - read back the generated columns, groups, and labels
- `create_column` - add a key column the generated board is missing
- `create_items` - add the starter items if the generated board does not have them
- `create_automation` - create the starter automation
- `list_automations` - verify the automation's trigger and conditions
- `manage_automations` - deactivate an automation that was created wrong
- `show-table` - display the new board as an interactive table, and again after the user's first change
