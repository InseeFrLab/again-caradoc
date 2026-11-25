# Setup

Run every command from the repo root.

## Backend

### FastAPI coding assistance

- Source: https://fastapi.tiangolo.com/ai-development/#install-without-prompts
- Gives: the `fastapi` skill for Claude Code. It ships inside the `fastapi` package, so it matches the installed
  version.
- Needs: FastAPI installed in the project (`uv sync`), because the command looks for the skill in `.venv`.

```bash
uvx library-skills install --skill fastapi --yes --claude
```

- Then: restart Claude Code.
- Writes: a link in `.claude/skills/`. Commit it.

### Temporal coding assistance

- Source: https://docs.temporal.io/with-ai
- Gives: the Temporal plugin for Claude Code, with the `temporal-developer` skill, and the `temporal-docs` MCP server,
  which searches the Temporal docs.
- Needs: nothing.

```bash
claude plugin marketplace add temporalio/claude-temporal-plugin --scope project
claude plugin install temporal@temporal-marketplace --scope project
claude mcp add --scope project --transport http temporal-docs https://temporal.mcp.kapa.ai
```

- Then: restart Claude Code, run `/mcp` and log in to `temporal-docs` with a Google or GitHub account.
- Writes: `.claude/settings.json` and `.mcp.json`. Commit them.
