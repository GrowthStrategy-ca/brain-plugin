# Company Brain for JobTread (Codex and other agents)

This plugin's instructions are plain markdown skills. Codex and other agents use the same files:

| Command | Instructions |
|---|---|
| setup | `skills/setup/SKILL.md` |
| interview | `skills/interview/SKILL.md` |
| briefing | `skills/briefing/SKILL.md` |

When the user asks for one of these by name (with or without the `/brain:` prefix), read the
matching file and follow it exactly. The spec schema is `spec/company-spec.schema.yaml`.

JobTread access comes from a JobTread MCP server attached to your agent (`codex mcp add`).
Find it by description ("JobTread Pave API" query tool), not by name.
