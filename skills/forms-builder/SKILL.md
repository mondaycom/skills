---
name: forms-builder
description: |
  Creates and edits monday.com WorkForms - public intake forms that capture
  responses as new items on a backing board. Useful for request intake, bug
  reports, lead capture, event signups, internal submissions, and any flow
  that turns a user-submitted form into a tracked work item.
  Use when user asks to "create a form", "build an intake form", "make a
  WorkForm", "add a question to my form", "edit form questions", "publish
  a form", "share a form link", "set up a request form", "add a form to a
  board", "remove the monday branding from my form", or "deactivate the
  form".
  Don't use for editing the items a form creates (use task-management) or for monday docs (use docs-collaborator).
license: MIT
metadata:
  author: monday.com
  version: "0.1.3"
---
# Forms Builder

Create, edit, and publish monday.com WorkForms. Every WorkForm is paired with a backing board: each form submission becomes a new item on that board, with the form's questions mapped to the board's columns. The form is the front door, the board is the destination.

## Core constraint - read this first

**WorkForms only create new items. They cannot update existing items.** This is a hard platform constraint, not a skill limitation. If a user asks for a form that "lets people update their tickets" or "edit existing entries", redirect them: a new submission can be linked to an existing item via a connected-boards question, but submitting the form will always create a new row on the form's board. For status-update style flows, suggest a different approach (an item-level update field, a separate update-tracking board, or an automation that finds-and-updates by reference).

## Plan-tier note

Forms ship in every monday plan, including Free and Basic. **WorkForms branding ("Powered by monday.com") shows by default and can only be removed on Pro plans and above.** If a user asks to remove the branding, check the plan first (`get_user_context` returns `account.tier`) or warn them: it requires `updateAppearance` with `form.appearance.hideBranding: true` and will silently fail on Free / Basic / Standard.

## Workflow: Create a form

### Step 1: Confirm the destination workspace

Forms must live in a workspace. Ask if not specified. `get_user_context` returns the user's favorite workspaces; call `list_workspaces` if you need to choose from all of them. The workspace ID is the only required parameter for `create_form`.

### Step 2: Decide the form's purpose, then call `create_form`

The form's purpose dictates the question set. Common shapes:

- **Bug report:** title, severity, environment, steps to reproduce, screenshot upload, reporter email.
- **Feature request:** title, description, use case, business impact, customer count, link.
- **Lead capture:** name, work email, company, role, what brought you here.
- **Event signup:** name, email, dietary needs, session preferences.
- **Internal request:** requester, request type (single-select), context, deadline, priority.

Call `create_form` with:
- `destination_workspace_id` (required)
- `destination_name` (the backing board's name)
- `destination_folder_id` or `destination_folder_name` if the user wants the board placed in a specific folder
- `board_kind` (`public`, `private` or `share`); use `private` if the form's responses are sensitive

Capture the returned `form_token` (and `board_id`) - every other forms tool takes it as `formToken`.

**`create_form` does not accept a form title parameter.** The form header defaults to "New Form" regardless of `destination_name` (which only names the backing board). Always follow `create_form` with an `update_form` call using `action: "updateFormHeader"` to set `form.title` (and optional `form.description`) before showing the URL to the user. Skipping this leaves every form labeled "New Form" in the WorkForms UI - confusing for end users and for the form owner browsing their forms list.

### Step 3: Build the question set

Use `form_questions_editor` with `action: "create"` for each question. Decide question type from the field's intent:

| Intent | Type |
|---|---|
| Free text, single line | `ShortText` |
| Free text, multi-line | `LongText` |
| Yes/no | `Boolean` |
| One-of-many | `SingleSelect` (set `settings.display: "Dropdown"` for long lists, `"Vertical"` for short) |
| Many-of-many | `MultiSelect` |
| Numeric | `Number` |
| Money / quantity / score | `Number` |
| Date | `Date` |
| Date range | `DateRange` |
| Email | `Email` |
| Phone | `Phone` |
| Web URL | `Link` |
| Country | `Country` |
| Address / map point | `Location` |
| File upload | `File` |
| Rating (1-5) | `Rating` |
| Person / assignee | `People` |
| Static instructions | `DISPLAY_TEXT` |
| Section heading | `PAGE_BLOCK` |
| Linked existing item | `ConnectedBoards` |
| Submitter name | `Name` |
| Comment / update field | `Updates` |
| E-signature | `Signature` |

Set `required: true` only for fields that genuinely block submission. Excess required fields cut response rate.

For `SingleSelect` and `MultiSelect`, `options` is required. Each option needs a `label` and a stable `value`. The platform may normalize string `value`s into numeric strings (`"0"`, `"1"`, `"2"`...) on creation, so the `value` you send is best-effort - always call `get_form` after creation to read back the canonical option values before any later edit. The canonical value must not change after the form has live submissions - changing it breaks data integrity for past responses.

### Step 4: Show the user the question set before publishing

After all questions are created, summarize them in a Markdown table (question text, type, required yes/no, and options if applicable). Ask the user to confirm or edit before sharing the form.

### Step 5: Confirm public access and share

Forms are created **active by default**. The form URL works the moment `create_form` returns. If the user wants a draft state (questions still being shaped), call `update_form` with `action: "deactivate"` immediately, finish the questions, then `activate` when ready. Otherwise, share the URL as-is.

`create_form` and `get_form` do not return the public form URL (`get_form` returns `features.shortenedLink.url` once the URL is shortened). Build it from the token. The `?r=` value is the account's data region; `use1` is the example `get_form` documents:

```
https://forms.monday.com/forms/{formToken}?r=use1
```

Include both the form URL and the backing board URL (`get_board_info` with the `board_id` returns its `url`) in the response. The user will want both - the form for sharing, the board for monitoring submissions.

### Step 6: Suggest follow-ups

End with 2-3 next-step suggestions:

- **"Want the form on a single scrolling page (Classic) instead of one question per screen (OneByOne)?"** - Classic is the better default for most operational and internal forms. Surface this choice every time. The MCP tools cannot change it (see below); the suggestion points the user to Form Settings -> Layout in the monday UI.
- "Want me to shorten the form URL?"
- "Should I set a password on the form?"
- "Should I set a submission limit or close date?"
- "Want to remove the WorkForms branding? (Pro+ only.)"

**Layout is two settings, and the MCP tools reach only one.** `update_form` `action: "updateAppearance"` sets the frame with `form.appearance.layout.type`: `"CARD"` renders the form inside a bordered card that scrolls within the card, `"FLAT"` renders it edge-to-edge and the whole page scrolls. Optional `alignment` (`FullLeft`, `Left`, `Center`, `Right`, `FullRight`) and `direction` (`LtR`, `Rtl`) sit beside it. The question flow (OneByOne or Classic) has no parameter in `update_form`, so it cannot be changed via MCP: direct the user to Form Settings -> Layout in the form editor. Do not present a CARD/FLAT change as a OneByOne/Classic change.

## Workflow: Edit an existing form

### Step 1: Resolve the formToken

If the user pastes a form URL, extract the token: `https://forms.monday.com/forms/{TOKEN}?r=...` -> `{TOKEN}` is the alphanumeric string between `/forms/` and `?` (or the end of the URL).

If the user names the form, call `search` with `searchType: "BOARD"` and `searchTerm: "<form or board name>"` to find the backing board. The results carry the board's `id` and `url`, not a formToken, and the MCP has no form-search tool: ask the user for the form URL and mark this as a known gap if it blocks them.

### Step 2: Fetch current state

Call `get_form` with the formToken. This returns questions, structure, appearance, and feature flags. Always read before writing - patches are merge-style and missing context leads to broken updates.

### Step 3: Apply the change

Pick the right tool:

- **Add / update / delete a single question:** `form_questions_editor`.
- **Reorder questions:** `update_form` with `action: "updateQuestionOrder"` and `form: {questions: [{"id": "<q1>"}, {"id": "<q2>"}, ...]}` (objects with required `id` and optional `page_block_id`, with every existing question included).
- **Header (title and description):** `update_form` with `action: "updateFormHeader"` (`form.title` and optional `form.description`).
- **Branding, layout, colors, logo:** `update_form` with `action: "updateAppearance"`.
- **Submission rules (anonymous, response limit, close date, login required, captcha):** `update_form` with `action: "updateFeatures"`.
- **Activate / deactivate / shorten URL / set password:** corresponding actions.

### Step 4: Confirm

Plain-language summary: what changed, what stayed the same, and the form URL. Do not narrate the API calls.

## Common edits

**Add a question to the end:** `form_questions_editor` with `action: "create"`, no `insert_after_question_id`. The question appends. `insert_after_question_id: null` puts it first.

**Insert a question between two existing ones:** call `get_form` to get the predecessor's ID, then create with `insert_after_question_id: "<predecessor_id>"`.

**Make a field required:** `form_questions_editor` with `action: "update"`, `questionId: "<id>"`, and `question: {type: "<existing type>", required: true}`. `question.type` must always be sent on updates.

**Update a select question's options without losing past responses:** call `get_form` first, copy each existing option's `value` into the update, then add or modify only the labels. Omitting an existing option deletes it.

**Show / hide a question conditionally:** use `show_if_rules` on the question. Rules are **OR-only** (every operator — top-level, rule-level, and condition-level — must be `"OR"`). Each condition has the shape `{building_block_id: "<question_id>", operator: "OR", values: ["<answer_value>"]}`.

**Reorder questions:** `update_form` with `action: "updateQuestionOrder"` and `form: {questions: [{"id": "<q1>"}, {"id": "<q2>"}, ...]}` (include optional `page_block_id` if using pages) - every existing question ID must appear, even unchanged ones. After the reorder lands, call `get_form` to re-sync question IDs - any `PAGE_BLOCK` (section heading) IDs are reassigned by the platform on reorder (e.g. `page_block__classic_default` becomes `page_block77dy0px4`). Cached PAGE_BLOCK IDs will 404 on the next edit if you skip the re-sync.

**Sync question titles with column names:** `update_form` with `action: "updateFeatures"` and `form.features.monday.syncQuestionAndColumnsTitles: true`. Useful when the user is editing the backing board and wants the form to follow.

**Drop the WorkForms branding (Pro+):** `update_form` with `action: "updateAppearance"` and `form.appearance.hideBranding: true`.

**Close the form to new responses:** `update_form` with `action: "deactivate"`. Reactivate with `action: "activate"`. To close on a date, use `updateFeatures` with `form.features.closeDate.enabled: true` and `form.features.closeDate.date: "<ISO timestamp>"`.

**Password-protect the form:** `update_form` with `action: "setFormPassword"` and `formPassword: "<password>"`. Communicate the password separately - it does not appear in the form URL. To remove it, `updateFeatures` with `form.features.password.enabled: false`.

## Smart defaults

When the user does not specify all values:

- **Form name and board name:** if the user gives the form a name, default the backing board's name to match. State the default in the response.
- **Workspace:** ask once if not provided. Do not guess.
- **Required flag:** false by default, except for the obvious primary identifier (email on a lead form, title on a bug report).
- **Question order:** the order the user listed them, top to bottom.
- **Form is active after creation:** yes - `create_form` returns an active, publicly-accessible form. If the user wants to preview before responses can come in, call `deactivate` immediately and reactivate when ready.
- **Form title:** `create_form` defaults the header to "New Form". Always set `form.title` explicitly via `update_form` `action: "updateFormHeader"` after creation - use the user's stated form name (or, if missing, the form's purpose, e.g. "Bug Intake"). State the title in your confirmation.
- **Layout:** new forms default to OneByOne (card-per-question). For most operational and internal forms, Classic (single scrolling page) is the better default - surface this in Step 6 follow-ups, do not silently rely on OneByOne. The choice is not reachable via MCP; point the user to Form Settings -> Layout in the form editor.
- **Branding:** leave WorkForms branding on by default; mention Pro+ removal as a follow-up rather than auto-toggling it.

## Error prevention and remediation

- **WorkForms cannot update existing items.** If asked, redirect to either an internal item-update field, a connected-boards question that links to an existing item, or a separate update-tracking workflow. Do not promise behavior the platform does not support.
- **Form URL returns "form is not available":** the form has been deactivated, hit a response limit, or passed a close date. Check `get_form` features state. Reactivate with `update_form` `action: "activate"`, raise the response limit, or extend the close date.
- **`updateQuestionOrder` rejects the call:** `form.questions` must be an array of `{"id": "<q>"}` objects containing every existing question ID. Call `get_form` first, collect all IDs, then send the full ordered array.
- **PAGE_BLOCK question IDs change after `updateQuestionOrder`.** The platform reassigns IDs for `PAGE_BLOCK` section-heading questions on every reorder (e.g. `page_block__classic_default` becomes `page_block77dy0px4`). Always call `get_form` immediately after `updateQuestionOrder` and refresh any cached PAGE_BLOCK IDs before the next edit, otherwise subsequent `form_questions_editor` calls referencing the old ID 404.
- **Select option deletion is destructive:** updating a `SingleSelect` or `MultiSelect` and omitting an existing option deletes it, breaking past responses that used that option. Always send all options when editing.
- **`hideBranding: true` does not stick:** the account is on a plan below Pro. Tell the user the constraint; do not retry.
- **Question type cannot change after creation:** to "convert" a question type, delete and recreate. Past responses for the original question are lost. Confirm with the user before deleting.

## Presenting form structure and responses

- **Before publish:** summarize the question set (question text, type, required flag) in a Markdown table alongside the form and backing board URLs. If `show-table` is available and the set has more than 5 questions, render the backing board with it so the user sees the question-to-column mapping.
- **After publish:** if the user asks "what came in this week", render the backing board with `show-table` and a date filter when it is available; otherwise read submissions with `get_board_items_page` (`includeColumns: true`, a date filter) and summarize them in a Markdown table with the board URL. For activation, deactivation, and single-question edits, a one-line confirmation plus the form URL is enough - no widget.

## Be conservative on destructive operations

Always confirm before:

- Deleting a question with prior submissions (data is lost).
- Deleting a select option that has been used (past responses for that option become orphaned).
- Deactivating a public-facing form (existing links break).
- Deleting the backing board (deletes the form too).

Single-question creates and non-destructive edits do not need extra confirmation - the user already asked.

## Tools used

- `list_workspaces` - locate the destination workspace if not provided
- `create_form` - create the form and its backing board (returns `form_token` and `board_id`)
- `get_form` - read current form state by formToken (always before editing)
- `update_form` - 11 actions covering header, appearance, accessibility, features, question order, tags, activation, password, URL shortening
- `form_questions_editor` - create / update / delete a single question
- `search` - find a backing board by name (`searchType: "BOARD"`, `searchTerm`) when only the form's purpose is known
- `get_board_info` - read the backing board's `url`
- `get_board_items_page` - read submissions from the backing board to summarize responses
- `get_user_context` - favorite workspaces, and `account.tier` for the branding plan check
- `show-table` (when available) - show the backing board for question-set review or response browsing

## Additional notes

- A `formToken` is the alphanumeric segment of a form URL (e.g., `https://forms.monday.com/forms/{TOKEN}?r=use1`, where the `?r=...` region parameter varies by data region). Treat `formToken` as the form's primary key.
- The backing board's items represent submissions. Editing a board column does not edit the form question - use `form_questions_editor` for the form side. The `monday.syncQuestionAndColumnsTitles` feature flag helps keep them in step.
- Forms that require login (`features.requireLogin.enabled: true`) restrict submissions to monday.com users in the same account.
- For multilingual forms, set `accessibility.language` via `update_form` `action: "updateAccessibility"`. Combine with `features.ai_translate.enabled: true` for AI-generated translations.
