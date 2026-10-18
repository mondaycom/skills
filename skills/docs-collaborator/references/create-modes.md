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
- Board name -> `search` with `searchType: BOARD`
- Owner name -> `list_users_and_teams` with `name:` (scoped per cross-skill rule)
- Ambiguous match -> ask once with the top 3 candidates; do not silently pick

**Active mode communication contract.** When the skill picks items / users in active mode, always state the choice and the reasoning: "Picked items X, Y, Z because <reason>. Override by naming different items."
