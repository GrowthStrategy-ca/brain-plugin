---
name: setup
description: Checks that the JobTread Company Brain is ready to run (JobTread query tool present, company-spec.yaml present) and prints the steps to schedule the daily briefing as a Claude routine. Use for "/brain:setup", "set up the brain", "schedule my morning briefing", or "is my brain configured".
---

# /brain:setup

Run these checks in order and print a short checklist with a pass or fail mark for each.
Do not write anything to JobTread.

## 1. JobTread connection

Look through your available tools for one whose description says it executes queries against
the **JobTread Pave API** and takes a single `query` object (claude.ai connector or Claude Code
MCP server; search "jobtread query" if tools are deferred). If found, run
`{"currentGrant": {"organization": {"name": {}}, "user": {"name": {}}}}` and show the org and user.

- Fail: say "Attach the JobTread connector in claude.ai (Settings > Connectors), or add JobTread's
  MCP server to Claude Code, then run /brain:setup again."
- Pass but the user is a person, not a dedicated agent user: recommend creating a JobTread user
  named after `company.agent_user` (default "Office") so the Brain's work is easy to spot.

## 2. Company spec

Check for `company-spec.yaml` in the working directory.
- Missing: say "Run /brain:interview first (about 10 minutes)." The briefing still runs on
  defaults without it.
- Present: show `company.name`, `updated`, `briefing.run_at`, and the count of `# assumed` lines.
- If the org name from step 1 differs from `company.name`, flag it.

## 3. Schedule the daily briefing (routine)

Print these steps for the user. They run on the user's own Claude subscription; GrowthStrategy
never sees or stores the Claude login.

1. Open **claude.ai/code** and go to **Routines**. Create a new routine.
2. Schedule: weekdays at `briefing.run_at` in `company.timezone` (default 07:07). It must be
   after `communication.quiet_hours.end` or the briefing prints but never posts. Routines run
   at most hourly, which is fine for a daily briefing.
3. Add this marketplace to the routine: `GrowthStrategy-ca/brain-plugin`, and enable the
   plugin `brain`. If the routine does not keep the plugin (the API dropped it on 2026-10-18),
   add this to the prompt instead: "If /brain:briefing is not available, run `git clone
   --depth 1 https://github.com/GrowthStrategy-ca/brain-plugin.git /tmp/brain-plugin` and
   follow `plugin/skills/briefing/SKILL.md` as a SCHEDULED run."
4. Attach the **JobTread** connector to the routine.
5. Put `company-spec.yaml` where the routine can read it (for example, commit it to the repo
   the routine checks out). If it is absent the briefing uses defaults and says so.
6. Prompt: `Run /brain:briefing.`
7. Run it once by hand and check the pinned comment on the Office job in JobTread.

## 4. License check (Day 2)

Stub. This plugin will log into **brain.growthstrategy.ca** (OAuth, remote MCP) to fetch the
company spec, templates and approvals, and to record each run in a ledger. Not active yet:
print "License check: not required in v0.1".
