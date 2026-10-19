# brain-plugin

**Company Brain for JobTread**, by GrowthStrategy. A Claude Code and Codex plugin that interviews
your company once, then posts an owner briefing inside JobTread every morning.

## Install (Claude Code)

```
/plugin marketplace add GrowthStrategy-ca/brain-plugin
/plugin install brain@brain-plugin
```

You also need JobTread connected: the JobTread connector in claude.ai, or JobTread's MCP server
in Claude Code or Codex.

## Commands

| Command | What it does |
|---|---|
| `/brain:interview` | About 25 questions, one at a time, each with a default you can accept with enter. Writes `company-spec.yaml`. Re-run any time; it only asks what changed. |
| `/brain:briefing` | Reads JobTread (past-due invoices, proposals expiring this week, idle jobs, yesterday's daily logs, today's schedule) and posts one pinned comment on your internal "Office" job. Re-running the same day updates that comment instead of adding another. |
| `/brain:setup` | Checks the JobTread connection and your spec, then walks you through scheduling the briefing as a daily Claude routine. |

Codex: see `plugin/AGENTS.md`.

## How it fits together

The plugin runs inside your own Claude or Codex subscription, on your machine or as a routine in
your own claude.ai account, and talks to JobTread through your own JobTread connection.
GrowthStrategy never stores or sees your Claude login. A GrowthStrategy service at
brain.growthstrategy.ca (coming) will hold your company spec, workflow templates, a ledger of
every change the Brain makes, and approvals; the plugin will log into it, but your subscription
does the running.

## Status

v0.1, October 2026. Read-only on JobTread except the Office job and its one daily comment.
