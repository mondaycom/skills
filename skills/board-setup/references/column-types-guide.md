# Column Types Guide

Complete reference for all monday.com column types. Use this when deciding which column type to add to a board via `create_column`.

## Core Column Types

### status

Visual color-coded labels representing workflow states. The most important column type in monday.com.

**Use for:** Workflow state, priority, approval status, risk level - anything with a fixed set of states.

**Default labels:** "Working on it" (orange), "Done" (green), "Stuck" (red). These ship by default if you do not pass `columnSettings`. Fine for the primary workflow Status column, wrong for Priority / Severity / Stage / Risk.

**Why use status over dropdown:** Status columns power automations, provide color-coded visual indicators, and are the primary driver of monday.com's workflow engine. Dropdown is for categorization; status is for workflow state.

**Custom labels at creation time.** Pass `columnSettings` as a JSON-stringified object:

```json
{"labels": [{"label": "Low", "color": "sky", "index": 0}, {"label": "Medium", "color": "working_orange", "index": 1}, {"label": "High", "color": "dark_orange", "index": 2}, {"label": "Critical", "color": "stuck_red", "index": 3}]}
```

`label`, `color`, `index` are required for each entry. The `color` token is a string from monday's StatusColumnColors enum; useful tokens by category: `sky` / `bright_blue` / `aquamarine` (low/calm), `working_orange` / `egg_yolk` / `purple` (mid), `dark_orange` / `sunset` / `lipstick` (high), `stuck_red` / `dark_red` (critical), `done_green` / `grass_green` / `bright_green` (success). The platform assigns each label a permanent numeric `id` that may not match `index` - use `index` to control display order, `id` is read-only.

**Anti-pattern:** Passing labels as a name->color map (`{"Low": "sky", ...}`) - the API rejects this. Must be an array of objects.

**`is_done` flag:** Each label has an optional `is_done` boolean. Setting `is_done: true` on a label tells reports, automations, and progress widgets that this label means "complete". On user-created boards via the UI, this flag is often left unset on every label including "Done". When creating status columns programmatically and you care about correct progress reporting, set `is_done: true` explicitly on the success label(s).

### people

Links to monday.com user accounts.

**Use for:** Item owner, assignee, reviewer, requester - anyone responsible for or related to the item.

**Supports:** Multiple people per cell, teams (not just individuals).

**Why use people over text:** People columns enable workload views, @mentions, notification routing, and "my items" filtering. A text column with someone's name is just a string.

### date

Single date value with optional time.

**Use for:** Deadline, due date, start date, event date - any single point in time.

**Supports:** Date picker UI, calendar view integration, overdue highlighting.

### timeline

Date range with start and end dates.

**Use for:** Project duration, sprint dates, event period, campaign window - anything with a start and end.

**Supports:** Gantt chart view, duration calculations, dependency visualization.

**When to use timeline vs date:** Use `date` for a single deadline. Use `timeline` when you need to show duration (e.g., "June 1-15").

### numbers

Numeric values with optional units.

**Use for:** Budget, cost, hours, story points, quantity, score - any numeric measurement.

**Supports:** Sum, average, median, min, max calculations at group and board level. Supports unit prefixes/suffixes (e.g., $, hrs).

### text

Short plain text input.

**Use for:** Brief notes, reference IDs, short descriptions - any free-form text under ~100 characters.

### long_text

Rich text with formatting support.

**Use for:** Detailed descriptions, meeting notes, specifications, instructions - longer content that benefits from formatting.

**Supports:** Bold, italic, lists, links, code blocks.

### dropdown

Multi-select from predefined options.

**Use for:** Categories, tags, departments, tools, regions - any categorization with a known set of options.

**Why use dropdown over status:** Use `status` for workflow state (one active state at a time). Use `dropdown` for categorization (can select multiple labels simultaneously).

### checkbox

Boolean toggle.

**Use for:** Approval flags, completion markers, yes/no questions, feature flags.

### file

File upload column.

**Use for:** Attachments, design files, documents, screenshots, deliverables.

**Supports:** Preview for images and common file types, download links.

## Communication Columns

### email

Email address with mailto link.

**Use for:** Contact email, customer email, vendor email.

### phone

Phone number with tel link.

**Use for:** Contact phone, support number, vendor phone.

### link

URL with display text.

**Use for:** External references, documentation links, PR links, design file URLs.

## Advanced Column Types

### rating

1-5 star rating.

**Use for:** Quality scores, satisfaction ratings, effort estimates on a simple scale.

### progress

Visual progress bar.

**Use for:** Completion percentage. Can be configured to auto-calculate based on subitems or linked columns.

### tags

Shareable tags across boards.

**Use for:** Cross-board categorization. Unlike dropdown (board-specific), tags are shared across the account.

### country

Country selector with flag icons.

**Use for:** Geographic location, market region, office location.

### location

Geographic coordinates with map integration.

**Use for:** Addresses, event venues, office locations.

### world_clock

Timezone display.

**Use for:** Team member timezones, meeting time references, global team coordination.

### week

Week date range selector.

**Use for:** Sprint weeks, weekly planning, week-based scheduling.

### hour

Time of day value.

**Use for:** Meeting times, shift start/end, deadline times.

### color_picker

Hex color value.

**Use for:** Brand colors, category colors, design specifications.

### formula

Calculated values based on other columns.

**Use for:** Derived metrics, cost calculations, score computations. Configured in the board UI after creation.

**Plan tier:** Pro and Enterprise only. If the user is on Free / Basic / Standard, recommend a `numbers` column updated manually or an external calculation.

### auto_number

Auto-incrementing ID.

**Use for:** Sequential numbering, ticket IDs, order numbers.

### dependency

Links between items showing dependencies.

**Use for:** Task dependencies in project management, prerequisite tracking.

### mirror

Reflects data from linked items on other boards.

**Use for:** Cross-board data visibility without duplication.

### connect_boards

Links items across different boards.

**Use for:** Relating items across boards (e.g., linking a task to a client record).

## Column Selection Decision Tree

1. **Is it a workflow state?** -> `status`
2. **Is it a person?** -> `people`
3. **Is it a date?**
   - Single date -> `date`
   - Date range -> `timeline`
4. **Is it a number?** -> `numbers`
5. **Is it a choice from a list?**
   - Workflow state -> `status`
   - Category/tag (single board) -> `dropdown`
   - Category/tag (cross-board) -> `tags`
6. **Is it yes/no?** -> `checkbox`
7. **Is it a file?** -> `file`
8. **Is it a URL?** -> `link`
9. **Is it contact info?**
   - Email -> `email`
   - Phone -> `phone`
10. **Is it text?**
    - Short (<100 chars) -> `text`
    - Long (rich text) -> `long_text`
11. **Is it a calculated value?** -> `formula` (Pro+ only)

## Recommended Starter Columns

For most boards, start with these 5 columns and add more as needed:

1. **Status** (`status`) - Every board needs a workflow state
2. **Owner** (`people`) - Who's responsible
3. **Due Date** (`date`) - When it's due
4. **Priority** (`status`, titled "Priority") - How important
5. **Notes** (`text`) - Additional context
