# Column Value Formats Reference

Complete formatting reference for all monday.com column types. Use this when constructing the `columnValues` JSON string for `create_item` or `change_item_column_values`.

The `columnValues` parameter is always a JSON string. Each key is a column ID (from `get_board_info`), and each value follows the format specific to that column's type.

## Status

Sets the status label. Use the exact label string from the board's status column settings (returned by `get_board_info`).

```json
{"status_column_id": {"label": "Done"}}
```

Common labels: "Working on it", "Done", "Stuck", "Not Started". Labels are case-sensitive and board-specific.

## Date

Sets a date value. Format: `YYYY-MM-DD`.

```json
{"date_column_id": {"date": "2025-06-15"}}
```

With time (optional):
```json
{"date_column_id": {"date": "2025-06-15", "time": "14:30:00"}}
```

## Timeline

Sets a date range. Both `from` and `to` are required.

```json
{"timeline_column_id": {"from": "2025-06-01", "to": "2025-06-30"}}
```

## People

Assigns one or more people or teams.

Single person:
```json
{"person_column_id": {"personsAndTeams": [{"id": 12345678, "kind": "person"}]}}
```

Multiple people:
```json
{"person_column_id": {"personsAndTeams": [
  {"id": 12345678, "kind": "person"},
  {"id": 87654321, "kind": "person"}
]}}
```

Team:
```json
{"person_column_id": {"personsAndTeams": [{"id": 98765, "kind": "team"}]}}
```

Get valid user/team IDs from `list_users_and_teams`.

## Text

Plain string value.

```json
{"text_column_id": "Any text content here"}
```

## Long Text

Rich text content. Supports basic HTML.

```json
{"long_text_column_id": {"text": "Detailed description with <b>bold</b> and <i>italic</i>"}}
```

## Numbers

String representation of the number.

```json
{"numbers_column_id": "42.5"}
```

## Email

Object with email address and display text.

```json
{"email_column_id": {"email": "user@example.com", "text": "Contact"}}
```

The `text` field controls the display label. If omitted, the raw email is shown.

## Phone

Object with phone number and country code.

```json
{"phone_column_id": {"phone": "+15551234567", "countryShortName": "US"}}
```

The `countryShortName` is the ISO 3166-1 alpha-2 code. Include country code in the phone number.

## Link

URL with optional display text.

```json
{"link_column_id": {"url": "https://example.com", "text": "Example Site"}}
```

## Dropdown

One or more labels from the column's predefined options.

Single option:
```json
{"dropdown_column_id": {"labels": ["High Priority"]}}
```

Multiple options:
```json
{"dropdown_column_id": {"labels": ["Frontend", "Backend"]}}
```

Use exact label strings from the column's settings (returned by `get_board_info`).

## Checkbox

String-based checked state.

```json
{"checkbox_column_id": {"checked": "true"}}
```

Use the string `"true"` or `"false"`, not boolean `true`/`false`.

## Rating

Integer from 1 to 5 (or the column's configured max).

```json
{"rating_column_id": {"rating": 4}}
```

## Tags

Array of tag IDs.

```json
{"tags_column_id": {"tag_ids": [12345, 67890]}}
```

## Country

Country code and name.

```json
{"country_column_id": {"countryCode": "US", "countryName": "United States"}}
```

## Location

Geographic location.

```json
{"location_column_id": {"lat": 40.7128, "lng": -74.0060, "address": "New York, NY"}}
```

## World Clock

Timezone identifier.

```json
{"world_clock_column_id": {"timezone": "America/New_York"}}
```

## Week

Start and end date of a week.

```json
{"week_column_id": {"week": {"startDate": "2025-06-09", "endDate": "2025-06-15"}}}
```

## Hour

Time value.

```json
{"hour_column_id": {"hour": 14, "minute": 30}}
```

## Color Picker

Nested hex color object.

```json
{"color_picker_column_id": {"color": {"hex": "#FF5733"}}}
```

## Combining Multiple Columns

Pass all column updates in a single JSON string:

```json
{
  "status": {"label": "Working on it"},
  "date4": {"date": "2025-06-15"},
  "person": {"personsAndTeams": [{"id": 12345678, "kind": "person"}]},
  "text0": "Design the new landing page",
  "numbers6": "8",
  "dropdown3": {"labels": ["High Priority"]},
  "checkbox9": {"checked": "false"}
}
```

## Clearing a Column Value

To clear a column, pass an empty string or null:

```json
{"column_id": ""}
```

Or for object-type columns:
```json
{"column_id": null}
```

## Common Mistakes

1. **Wrong status label** - Labels are case-sensitive and board-specific. Always check `get_board_info` first.
2. **Using column name instead of ID** - The key must be the column ID (e.g., `status`, `date4`), not the display name (e.g., "Status", "Due Date").
3. **Missing nested structure** - Status requires `{"label": "..."}`, not just `"Done"`. Date requires `{"date": "..."}`, not just `"2025-06-15"`.
4. **Invalid person ID** - Always verify user IDs with `list_users_and_teams` before assigning.
5. **Incorrect number format** - Numbers must be strings: `"42"` not `42`.

## Plan-tier note

Most column types are universal (Free / Basic / Standard / Pro / Enterprise). Two are gated:

- **Formula columns** (`formula`): Pro and Enterprise plans only. If the user is on Free / Basic / Standard, the create_column call will fail. Suggest a `numbers` column they update manually, or a downstream calculation in a docs/dashboards skill.
- **Mirror columns** (`mirror`): Available on most paid plans but with caveats. Verify before recommending.
