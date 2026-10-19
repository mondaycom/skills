# Errors and remediation

Detail for "Surface errors with remediation" in `SKILL.md`.

Common failures and how to remediate:

- **"block_id must be UUID"** - the ID is malformed (not a valid UUID). The API validates UUID format before lookup. Re-read with `include_blocks: true` and use the actual block IDs returned.
- **"Block not found" / no-op** - the block_id is well-formed but stale. Re-read with `include_blocks: true` and retry. Block IDs change when blocks are deleted and recreated (e.g. via `replace_block`).
- **"Cannot use update_block on a divider / table / image"** - schema mismatch. Switch to `replace_block`.
- **"parent_block_id references a block not yet created"** - notice_box nesting attempted in the same call. Split into two calls.
- **"CellLimitExceededException" on `create_doc location=item`** - the target doc column cell is already populated (1-doc-per-cell limit). Fall back to a workspace-level doc with a bi-directional DOC mention cross-link to the existing attached doc. See the Step 1 preflight in `SKILL.md` ("Workflow: Create a doc").
- **Live embed renders blank** - either the item / column was deleted, or the viewer lacks permission on the source board. Verify with `get_board_info`. If it's a permissions issue, suggest moving the source data to a less-restricted board.
- **INTERNAL_SERVER_ERROR on a long `create_doc` or `add_markdown_content`** - large markdown bodies, especially with many tables, fail on import, and `create_doc` can leave a partly created doc behind. Do not resend the whole body. Call `read_docs` to see what landed, then append the rest in chunks of a few sections each, one `add_markdown_content` operation per `update_doc` call (several in one call can interleave). For a new doc, create it with the first section only and append the rest.
- **"Plan limit reached"** on `create_doc` - Free / Basic 3-doc cap. Suggest upgrading to Standard+ or deleting an unused doc.
- **"Cannot create BOARD / WIDGET / DOC / GIPHY block"** - these block types have no public-create path. Suggest the closest alternative (a markdown link to the board, or `add_markdown_content` with a board URL).
- **Orphan comment anchor** - if `read_docs` shows `attributes.comments: [<update_id>]` on a block but the comment is missing from the comments array, the comment was deleted but the anchor remained. Separately update the block's delta to drop the `comments` attribute for full cleanup.
