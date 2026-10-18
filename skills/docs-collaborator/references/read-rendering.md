# Rendering a doc for read requests

Detail for Step 3 of "Workflow: Read a doc" in `SKILL.md`.

For read responses to "show me / what's in / read this doc" prompts:

- **Always resolve mentions.** Map every USER mention id to a display name (via cached `list_users_and_teams` or a scoped lookup). Map every DOC mention to its doc name. Map every BOARD mention to its board name. Render inline as `**@Alex Polonsky**` (or similar) rather than `@{USER-38288909}`.
- **Resolve embeds when data is cached or cheap to fetch.** If the doc references items / columns already touched in the conversation, use the cached value. If not, fetch with `get_board_items_page` scoped to the specific items + columns. Render as the resolved column value ("**Working on it**", "**No value**", "**2026-05-15**") inline. Skip resolution only if it would require many extra MCP calls for a long doc.
- **Always keep the structured enumeration table as a secondary affordance.** List every inline non-text element (mentions, embeds, notice_boxes, code blocks, tables) with block IDs and IDs. Useful for downstream edit operations and for users who want to target a specific element.
