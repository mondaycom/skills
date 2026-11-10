# Duplicate detection before writing

Detail for "Duplicate detection before writing" in `SKILL.md`.

**Before any `create_block`, `add_markdown_content`, `add_comment`, or content-replacing `update_block`, compare the proposed content against existing blocks in the doc.** Default to skip + report on match; surface options to the user; never silently duplicate.

Match heuristics by block type:

| Block type | Match heuristic |
|---|---|
| text / heading / quote / code | exact delta text + same subtype |
| list_item | exact text + same list type + adjacency (multi-bullet sections matched as a group) |
| notice_box | same theme + same child text |
| table | same dimensions + same cell content |
| mention | same user_id at same block position |
| column_value embed | same item_id + column_id at same block position |
| comment | same body text + same anchor (excluded by default - see below) |

Skill behavior on match found:

1. **Default: skip + report.** "X already exists at <position>. Skipping to avoid duplicate."
2. **Surface options** (do not silently pick):
   - **(a) skip** - keep existing, do nothing
   - **(b) duplicate** - proceed as requested, accept redundancy
   - **(c) replace** - delete existing, create fresh
   - **(d) move** - offered only when the new request has explicit positional intent ("at the top / at the bottom / after X") that conflicts with the existing match's position

Exclusions where duplication is normal and detection should NOT trigger:

- **Comments are excluded** except for a short stutter window. Same comment body submitted to the same scope (doc or block) within 60 seconds is almost certainly a double-submit / network retry; beyond that, treat as legitimate (legitimate re-review, sentiment echo).
- Adjacent same-style blocks in a list - intended structure, not duplication.
- Repeating section names in different parents (e.g. two "Action Items" H3s under different H2s).

Apply this duplicate-detection rule to every write path in the skill.
