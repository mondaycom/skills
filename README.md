# monday.com Agent Skills

[![skills.sh](https://skills.sh/b/mondaycom/skills)](https://skills.sh/mondaycom/skills)

Official [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
for [monday.com](https://monday.com) - the customizable AI work platform for work
management, CRM, dev, service, and campaigns. They equip any skills-compatible
agent to manage boards, items, docs, and forms correctly, including the right
column formatting, board structure, and platform conventions.

> These skills run through the [monday.com Platform MCP](https://developer.monday.com/api-reference/docs/monday-mcp-overview)
> (`https://mcp.monday.com/mcp`, OAuth-based - no API tokens required).

## Installing

These skills work with any agent that supports the Agent Skills standard,
including Claude Code, Cursor, OpenAI Codex, and OpenCode.

### npx skills

Install using the [`npx skills`](https://skills.sh) CLI:

```
# Install all skills
npx skills add mondaycom/skills

# Install a specific skill
npx skills add mondaycom/skills --skill task-management
```

### Clone / Copy

Clone this repo and copy the skill folders into the appropriate directory for your agent:

| Agent | Skill Directory | Docs |
|-------|-----------------|------|
| Claude Code | `~/.claude/skills/` | [docs](https://code.claude.com/docs/en/skills) |
| Cursor | `~/.cursor/skills/` | [docs](https://cursor.com/docs/context/skills) |
| OpenCode | `~/.config/opencode/skills/` | [docs](https://opencode.ai/docs/skills/) |
| OpenAI Codex | `~/.codex/skills/` | [docs](https://developers.openai.com/codex/skills/) |

```
git clone https://github.com/mondaycom/skills.git

# Copy to your agent's skill directory, e.g.:
cp -r skills/task-management ~/.claude/skills/
cp -r skills/task-management ~/.cursor/skills/
```

## Skills

| Skill | Useful for |
|-------|------------|
| `task-management` | Create, update, assign, and comment on items across any board, with correct column-value formatting handled automatically |
| `project-status-report` | Generate status reports surfacing overdue items, blockers, workload distribution, and recent activity across one or more boards |
| `board-setup` | Create and configure boards with the right groups, column types, and templates for any workflow |
| `onboard-to-monday` | First-run onboarding for new users: previews a board tailored to their work as an interactive table, then builds it with starter items and an automation once they confirm |
| `docs-collaborator` | Read, draft, and update monday docs with live board embeds and user mentions |
| `forms-builder` | Build WorkForms for intake workflows that capture responses as items on a backing board |

## License

MIT. See [`LICENSE`](LICENSE).
