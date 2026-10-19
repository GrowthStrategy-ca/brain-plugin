---
name: briefing
description: Owner morning briefing for a JobTread contractor. Reads company-spec.yaml, queries JobTread for past-due invoices, expiring proposals, idle jobs, yesterday's daily logs and today's tasks, and posts one pinned comment "Morning briefing — YYYY-MM-DD" on the internal "Office" job (updating it if it already exists today). Use for "/brain:briefing", "morning briefing", "what needs my attention in JobTread today", or when a scheduled routine asks for the briefing.
---

# /brain:briefing

Produce the owner's morning briefing from JobTread and post it as one comment on the internal
Office job. Read-only on everything except: the Office job setup (first run only) and the one
briefing comment per day.

## 0. Find the JobTread tool

Do not assume a tool name. Look through your available tools for one whose description says it
executes queries against the **JobTread Pave API** and takes a single `query` object. It may be a
claude.ai connector or a Claude Code MCP server (names like `query` under a JobTread server). If
tools are deferred, search for "jobtread query" first.

- None found: stop. Say "No JobTread connection in this session. Attach the JobTread connector
  (or MCP server) and run again." Do not invent data.
- More than one found: run step 1 on each and use the one whose organization name matches
  `company.name` in the spec. If none or several match, stop and say which orgs you saw.
- Exactly one found: still compare the organization name from step 1 with `company.name`.
  If they differ, stop and say so: the wrong JobTread connector is attached.

## 1. Context

1. Run `{"currentGrant": {"organization": {"id": {}, "name": {}}, "user": {"name": {}}}}`.
   Use the returned organization id as `<ORG_ID>` everywhere below.
2. Read `company-spec.yaml` from the working directory. If it is missing, use these defaults
   and say so in the briefing's last line ("Using default settings; run /brain:interview"):
   timezone = the machine's, office_job "Office", idle_job_days 5, expiring_window_days 7,
   past_due_grace_days 0, pin_comment true, max_rows_per_section 10.
3. Compute dates in the spec's timezone, as `YYYY-MM-DD`:
   - `<TODAY>`, `<YESTERDAY>`
   - `<PAST_DUE_BEFORE>` = TODAY minus past_due_grace_days
   - `<EXPIRES_BY>` = TODAY plus expiring_window_days
   - `<IDLE_SINCE>` = TODAY minus idle_job_days
4. Quiet hours: if this is a scheduled run and the local time is inside
   `communication.quiet_hours`, print the briefing but do not post it, and say why.
   A person running the command by hand overrides quiet hours.

## 2. Pull the data (one query)

Validated against a live org on 2026-10-17. Every list is capped at `size: 10`; `count` gives
the true total. Replace `<ORG_ID>`, the dates, and `"Office"` with the spec's `company.office_job`.

```json
{"organization": {"$": {"id": "<ORG_ID>"}, "name": {},
  "pastDueInvoices": {"_": "documents",
    "$": {"size": 10, "where": {"and": [["type", "customerInvoice"], ["status", "!=", "draft"], ["balance", ">", 0], ["dueDate", "<", "<PAST_DUE_BEFORE>"]]},
          "sortBy": [{"field": "dueDate", "order": "asc"}]},
    "count": {}, "balanceTotal": {"_": "sum", "$": "balance"},
    "nodes": {"id": {}, "fullName": {}, "balance": {}, "dueDate": {}, "job": {"id": {}, "name": {}}}},
  "expiringProposals": {"_": "documents",
    "$": {"size": 10, "where": {"and": [["type", "customerOrder"], ["status", "pending"], ["dueDate", ">=", "<TODAY>"], ["dueDate", "<=", "<EXPIRES_BY>"]]},
          "sortBy": [{"field": "dueDate", "order": "asc"}]},
    "count": {}, "nodes": {"id": {}, "fullName": {}, "priceWithTax": {}, "dueDate": {}, "job": {"id": {}, "name": {}}}},
  "idleJobs": {"_": "jobs",
    "$": {"size": 10,
          "with": {"recentLogs": {"_": "dailyLogs", "$": {"where": ["date", ">=", "<IDLE_SINCE>"]}, "count": {}},
                   "recentComments": {"_": "comments", "$": {"where": ["createdAt", ">=", "<IDLE_SINCE>"]}, "count": {}},
                   "recentTime": {"_": "timeEntries", "$": {"where": ["startedAt", ">=", "<IDLE_SINCE>"]}, "count": {}}},
          "where": {"and": [["closedOn", null], ["name", "!=", "Office"], ["createdAt", "<", "<IDLE_SINCE>"],
                            [["recentLogs", "count"], 0], [["recentComments", "count"], 0], [["recentTime", "count"], 0]]},
          "sortBy": [{"field": "createdAt", "order": "asc"}]},
    "count": {}, "nodes": {"id": {}, "name": {}, "number": {}}},
  "yesterdayLogs": {"_": "dailyLogs",
    "$": {"size": 10, "where": ["date", "<YESTERDAY>"]},
    "count": {}, "nodes": {"id": {}, "notes": {}, "job": {"name": {}}, "user": {"name": {}}}},
  "todayTasks": {"_": "tasks",
    "$": {"size": 10, "where": {"and": [["isToDo", false], ["progress", "!=", 1], ["startDate", "<=", "<TODAY>"], ["endDate", ">=", "<TODAY>"]]},
          "sortBy": [{"field": "endDate", "order": "asc"}]},
    "count": {}, "nodes": {"id": {}, "name": {}, "endDate": {}, "job": {"name": {}},
                           "assignedMemberships": {"$": {"size": 3}, "nodes": {"user": {"name": {}}}}}}}}
```

Notes:
- Always give nested connections a small `size` (as on `assignedMemberships`). An unbounded
  nested list can fail with "Request Entity Too Large".
- `progress` is null on tasks never started; `["progress", "!=", 1]` keeps them.
- Document statuses are `draft | pending | approved | denied`. Proposals are `customerOrder`.
- Review 2026-10-18: drafts are never past due; expired proposals are excluded so they cannot
  crowd out the ones expiring this week; a job younger than `idle_job_days` is not idle;
  "active today" means start <= today <= end, so long tasks appear every day until done.
- If the query errors, introspect the failing field with
  `{"schema": {"$": {"path": "root.organization.<field>", "expand": true}}}`, fix it, retry once.

## 3. Write the briefing

Comment **name** (exact, the idempotency key): `Morning briefing — <TODAY>` (an em dash).
Comment **message**: plain text, at most 4096 characters, in this order. Use the spec's `tone`.
Show at most `max_rows_per_section` rows per section and "+N more" when `count` is larger.
Name a section with "None." when it is empty; never drop a section.

```
Good morning <owner>. <one sentence: the single most important thing today>

PAST-DUE INVOICES (<count>, $<balanceTotal> outstanding)
- <fullName> · <job name> · $<balance> · due <dueDate>

PROPOSALS EXPIRING BY <EXPIRES_BY> (<count>)
- <fullName> · <job name> · $<priceWithTax> · expires <dueDate>

JOBS IDLE <idle_job_days>+ DAYS (<count>)
- #<number> <name>

YESTERDAY'S DAILY LOGS (<count>)
- <job name> · <user name>: <first 120 chars of notes>
(If daily_logs_expected is true and this is 0 while jobs are active, say so.)

TASKS ACTIVE TODAY (<count>, in progress or due)
- <task name> · <job name> · due <endDate> · <assignees or "unassigned">

NEEDS A DECISION
- <exactly one line: the one call only the owner can make today, picked from the data above,
  e.g. "Close or revive the 15 idle jobs?" or "Approve chasing the oldest past-due invoice?">

Posted by the Brain from JobTread data at <HH:MM tz>. Read-only except this comment.
```

## 4. Post it (idempotent)

1. **Find the Office job:**
   ```json
   {"organization": {"$": {"id": "<ORG_ID>"},
     "officeJob": {"_": "jobs", "$": {"size": 1, "where": {"and": [["name", "Office"], ["closedOn", null]]}},
       "nodes": {"id": {}, "name": {},
         "comments": {"$": {"size": 5, "where": ["name", "Morning briefing — <TODAY>"]},
                      "nodes": {"id": {}, "name": {}, "createdAt": {}}}}}}}
   ```
2. **If the Office job does not exist**, create it once (three calls, each using the id returned
   by the one before). The names make it obvious this is internal:
   ```json
   {"createAccount": {"$": {"organizationId": "<ORG_ID>", "name": "Brain (internal - do not invoice)", "type": "customer"}, "createdAccount": {"id": {}}}}
   {"createLocation": {"$": {"accountId": "<ACCOUNT_ID>", "name": "Office (internal)", "parseAddress": false}, "createdLocation": {"id": {}}}}
   {"createJob": {"$": {"locationId": "<LOCATION_ID>", "name": "Office", "description": "Internal job used by the Company Brain for the daily owner briefing. Not a customer job."}, "createdJob": {"id": {}, "number": {}}}}
   ```
   Before creating the account, check for an existing account with that exact name and reuse it.
3. **If a comment named `Morning briefing — <TODAY>` already exists** on the Office job,
   update it instead of creating another:
   ```json
   {"updateComment": {"$": {"id": "<COMMENT_ID>", "message": "<MESSAGE>"}}}
   ```
4. **Otherwise create it:**
   ```json
   {"createComment": {"$": {"targetType": "job", "targetId": "<OFFICE_JOB_ID>",
     "name": "Morning briefing — <TODAY>", "message": "<MESSAGE>", "isPinned": true,
     "isVisibleToInternalRoles": true, "isVisibleToCustomerRoles": false, "isVisibleToVendorRoles": false},
     "createdComment": {"id": {}, "name": {}}}}
   ```
   Then unpin older briefings so only today's stays on top:
   ```json
   {"job": {"$": {"id": "<OFFICE_JOB_ID>"},
     "comments": {"$": {"size": 10, "where": {"and": [["isPinned", true], ["name", "like", "Morning briefing%"], ["name", "!=", "Morning briefing — <TODAY>"]]}},
                  "nodes": {"id": {}}}}}
   {"updateComment": {"$": {"id": "<OLD_ID>", "isPinned": false}}}
   ```
5. Never post anywhere else. Never create tasks, send messages, or change any record other
   than the Office job and its briefing comment.

## 5. Report

Print the briefing text, then one line: `created` or `updated`, the comment id, and the Office
job link `https://app.jobtread.com/jobs/<OFFICE_JOB_ID>`.
