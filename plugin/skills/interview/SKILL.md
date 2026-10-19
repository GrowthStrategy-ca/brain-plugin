---
name: interview
description: Company onboarding interview for the JobTread Company Brain. Asks about 25 questions one at a time, each with a recommended default, and writes company-spec.yaml in the working directory. Use when the user says "/brain:interview", "set up my company brain", "interview my company", or wants to change their brain settings. Re-running loads the existing spec and asks only what changed.
---

# /brain:interview

You are interviewing a contractor (usually the owner) so the Brain knows how their company runs.
The output is one file, `company-spec.yaml`, in the current working directory. Its schema and
defaults live in `../../spec/company-spec.schema.yaml` (relative to this file). Read that file first.

## How to run the interview

1. **Check for an existing spec.** If `company-spec.yaml` exists in the working directory, load it.
   Say "I found your spec from <updated>. I'll only ask what might have changed." Then show each
   area's current values in a compact list and ask, one area at a time: "Anything to change here?
   (enter = keep)". Only drill into the questions of an area the user wants to change.
2. **Fresh interview.** Ask the questions below **one at a time**. Never batch questions.
   Each question shows a recommended default in brackets. If the user presses enter, says "yes",
   "default", "sure" or similar, take the default. If the user says "skip" or "don't know", take
   the default and mark it `# assumed` in the file.
3. Keep your own words short. No lectures between questions. One line of context only when the
   question is not obvious.
4. If an answer conflicts with an earlier one (for example the approver is not in the people
   lists), point it out in one line and ask once.
5. You may pre-fill a default from JobTread if a JobTread `query` tool is available (for example
   the company name from `{"currentGrant": {"organization": {"name": {}}}}`, or member names from
   memberships). Never write anything to JobTread during the interview.

## The questions

Number them for the user ("3 of 25") so they can see progress.

### 1. Company and people
1. Company name? [from JobTread org name, else ask]
2. Time zone? [the machine's time zone]
3. Who is the owner the morning briefing is for? [the JobTread user running this]
4. Who approves anything a customer will see (messages, proposals, invoices)? [the owner]
5. Who works in the office? (names, comma separated) [none]
6. Who are your field leads and crew? [none]
7. The Brain acts in JobTread as its own user so its work is easy to spot. What should that user
   be called? Create it in JobTread under Settings > Users if it does not exist yet. ["Office"]

### 2. Lead flow
8. Where do your leads come from? [website form, referrals, phone]
9. How fast do you expect a first reply to a new lead, in hours? [24]
10. What makes a lead "qualified" for you, in one sentence? [in service area, real budget, deciding within 90 days]

### 3. Estimates and proposals
11. Who builds estimates? [the owner]
12. How many days is a proposal valid before it expires? [30]
13. After a proposal goes out, on which days should someone follow up? [2, 7, 14]

### 4. Production and crew
14. Do you expect a daily log on every job day? [yes]
15. Do crews clock in and out in JobTread? [yes]
16. Who owns the schedule? [first office person, else the owner]
17. After how many days with no daily log, comment or time entry should an open job count as
    idle? [5]

### 5. Money
18. What are your invoice terms, in days? [15]
19. How many days after the due date should the briefing call an invoice past due? [0]
20. How do field receipts get to the office? [photo into a JobTread vendor bill]

### 6. Communication and approval rules
21. What, if anything, may the Brain send outside JobTread without a human approving it first?
    [nothing; everything customer-facing waits for approval]
22. Quiet hours: when should the Brain never post or notify? [20:00 to 07:00]
23. What tone should anything the Brain drafts have? [plain, short, friendly]
24. What needs an answer within the hour? List them. Say plainly: "I'll write these down so we
    can plan for them. This version runs at most hourly, so it does not promise to answer these
    within the hour." [none]
25. What time should the morning briefing run? [07:07] (must be after quiet hours end)

## Writing the file

- Write `company-spec.yaml` in the working directory, following the schema's key names and
  order exactly. Set `spec_version: 1` and `updated:` to today's date.
- Answers taken by default after "skip" or "don't know" get a trailing `# assumed` comment.
- `fast_lane:` holds the answers to question 24 as a list of short strings.
- `communication.may_leave_jobtread_without_human:` is an empty list unless the user named
  specific message types in question 21.
- Derived values: `estimates.expiring_window_days: 7`, `briefing.pin_comment: true`,
  `briefing.max_rows_per_section: 10`, `company.office_job: "Office"` unless the user asks otherwise.
- Never put secrets, API keys, grant keys or passwords in the file.
- On a re-run, keep every value the user did not change, and update `updated:`.

## Finish

Print a summary of at most 12 lines: company, owner and approver, agent user, idle-job days,
proposal window, invoice terms and grace, quiet hours, briefing time, fast-lane items (marked
"recorded, not promised in v1"), and the count of `# assumed` answers.

Then say what `/brain:briefing` will do with this spec:

> Each morning at <run_at>, `/brain:briefing` reads this file, checks JobTread for past-due
> invoices, proposals expiring in the next <expiring_window_days> days, open jobs idle for
> <idle_job_days>+ days, yesterday's daily logs and today's scheduled tasks, and posts one
> pinned comment called "Morning briefing — <date>" on your "<office_job>" job. It reads
> JobTread and writes only that one comment. Run `/brain:setup` next to schedule it.
