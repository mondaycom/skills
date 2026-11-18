# Creating docs: passive scaffold vs active mode

Detail for Step 3 of "Workflow: Create a doc" in `SKILL.md`.

**Default mode: passive scaffold.** Reserve visible slots for embeds and mentions in the markdown body using machine-readable markers. Skill (and downstream agents) can later detect and fill these slots when the user names the underlying items / users.

| Slot type | Marker format | Example |
|---|---|---|
| Live column-value embed | `_[embed: <column> of <item-ref>]_` | `_[embed: status of <top-priority item>]_` |
| User mention | `_[mention: <user-ref>]_` | `_[mention: <owner>]_` |
| Doc / board mention | `_[mention-doc: <doc-name>]_` / `_[mention-board: <board-name>]_` | `_[mention-board: Project Tracker]_` |

Italic styling on slot markers gives visual signal that the content is dynamic. The bracket-keyword format is grep-able for downstream automation.

**Active mode: escalate when intent signals are present.** Switch from passive scaffold to wiring real embeds / mentions / cross-links when the user's prompt signals intent through name-based references. Never on item IDs - users speak in names.

Triggers that escalate to active mode:

- Item names ("the Q3 plan", "the launch milestone", "the blocker we discussed")
- Board names ("from the project tracker", "in the bug board")
- Owner names ("Daniel's tasks", "my items", "the design team's")
- Positional refs ("the top 3 priorities", "the most recent stuck items")
- Imperative wiring ("embed live status from", "show current status of", "pull from <board>", "wire up the embeds", "tag me as owner")

Resolution chain (name -> ID, never ask the user for IDs):
- Item name -> `get_board_items_page` with `searchTerm` scoped to the named board
- Board name -> `search` with `searchType: "BOARD"` and `searchTerm`
- Owner name -> `list_users_and_teams` with `name:` (scoped per cross-skill rule)
- Ambiguous match -> ask once with the top 3 candidates; do not silently pick

**Active mode communication contract.** When the skill picks items / users in active mode, always state the choice and the reasoning: "Picked items X, Y, Z because <reason>. Override by naming different items."

## Common doc shapes

Starting points when the user names a use case. Extend freely; they are examples, not boundaries.

- **Meeting notes:** title, date, attendees (with `@mentions`), agenda bullets, decisions, action items table, follow-ups.
- **Weekly update:** highlights, key metrics, risks, next-week plan.
- **RFC:** problem, proposed solution, alternatives considered, open questions, decision log.
- **Runbook:** trigger, prerequisites, numbered procedure, rollback steps, escalation path.
- **Postmortem:** incident summary, timeline, root cause, what went well, what did not, action items.

## Markdown notes

- Markdown headings (`#`, `##`, `###`) map to LARGE_TITLE / MEDIUM_TITLE / SMALL_TITLE blocks.
- Markdown tables work and the cells will be populated with the values you put in. **This is the only way to get pre-filled table content.** Cell-level API population after creation is not supported.
- Markdown images and links work. Markdown cannot reference monday assets: for an image block, use `update_doc` `create_block` with `block_type: "image"` and either `public_url` or `asset_id`. An `asset_id` must already exist - the `assetId` in an item's file column, or the `asset_id` that `finalize_asset_upload` returns after `get_asset_upload_url` (only when the client can PUT the file itself). Do not use the `public_url` from `get_assets` as an image source: it expires after an hour.
- Markdown does not have notice_box, layout, embed, divider-with-spacing, mention, or live column-value syntax. Add those via `update_doc` after the doc is created.
