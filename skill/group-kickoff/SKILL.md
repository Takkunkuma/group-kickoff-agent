---
name: group-kickoff
description: Group Kickoff Agent. Scans a Canvas course for upcoming GROUP assignments (Radar), then reads one assignment's page + rubric and publishes a shared one-page Kickoff Brief so every teammate walks into the first meeting aligned. Use when the user says "kickoff", "radar", "what group work is coming up", or names a Canvas group assignment to prep a meeting for.
---

# Group Kickoff Agent

Goal: remove the "what are we even supposed to do?" part of a group meeting so the team spends its time on the decisions that need real thinking.

**Core rule: the agent prepares the team, it never does the assignment.** No drafting slides, answers, or content. It surfaces requirements, points, traps and the questions the team must decide together.

The user talks to this agent in plain language ("prep our final project kickoff", "what group work is coming up?", "make a brief for the press release one and post it to our channel"). They should never need an assignment ID. Default course: https://canvas.<your-school>.edu/courses/<COURSE_ID> (ask if the request is about another course).

## How the agent decides what to do

1. **Work out the goal** from the request: Radar only, Brief, or Brief + share to Slack.
2. **Work out which assignment** (Step 1.5). Never guess silently between two candidates; ask.
3. **Run the steps**, adapting to what it finds (login wall → ask the user to log in; truncated text → keep reading; references to earlier work → go fetch those too).
4. **Check its own output** against the rubric before publishing (Step 3.5).
5. **Ask before anything leaves the user's hands**: show the Slack message and channel and wait for a "yes" before posting.

## Step 0 — Connect to Canvas (Claude in Chrome)

1. `tabs_context_mcp` (createIfEmpty), navigate the tab to `https://canvas.<your-school>.edu/courses/<COURSE_ID>/assignments`.
2. If the page shows a login screen, stop and ask the user to log in themselves. Never type credentials.
3. All reads use the Canvas REST API from inside the logged-in page with `javascript_tool` (read-only GETs only, never POST/PUT). Helper:

```js
window._g = async u => JSON.parse((await (await fetch(u,{headers:{Accept:'application/json'}})).text()).replace(/^while\(1\);/,''));
```

The tool truncates output around ~1,000 characters, so stash long text on `window` and read it in slices.

## Step 1 — Radar (find upcoming group work)

```js
const a = await _g('/api/v1/courses/<COURSE_ID>/assignments?per_page=100&order_by=due_at&include[]=submission');
const now = Date.now();
a.filter(x => x.group_category_id && x.due_at && new Date(x.due_at) > now)
 .map(x => ({id:x.id, name:x.name, due:new Date(x.due_at).toLocaleString(), daysLeft:((new Date(x.due_at)-now)/864e5).toFixed(1), pts:x.points_possible, submitted:x.submission?.workflow_state}))
```

- "Group" = Canvas `group_category_id` is set (the official group-assignment flag), NOT a guess from the title.
- Report a short table: name, due (local time), days left, points, submitted?. Flag anything due in < 5 days as ⚠️.
- Also list (separately) items whose title contains "group" but have no group flag — possible misses.

## Step 1.5 — Figure out which assignment the user means

Match the user's words against the full assignment list (names, plus due dates):
- "next", "upcoming", "this week" → the soonest unsubmitted group assignment from Radar.
- A nickname ("final project", "press release one", "A4") → fuzzy-match on the name.
- If two or more match (e.g. "Final Group Project" and "Final Group Presentation" share a due date and files), say so and ask: brief one, the other, or both combined.
- If the best match is not a group assignment, warn: "This one is individual. Brief it anyway?"
State the match in one line ("I'm briefing: <name>, due <date>") before continuing.

## Step 2 — Read the assignment

For the chosen assignment ID:
- `x = await _g('/api/v1/courses/<COURSE_ID>/assignments/<ID>')`
- Description → plain text via a temporary `div.innerHTML` → `innerText`; store in `window._desc` and read all slices to the end.
- Rubric: `x.rubric` → criterion, `long_description`, points.
- Attached files: list `/files/<id>` links from the description with `_g('/api/v1/files/<id>')` (name, size). Do not download them; note in the brief that attachments were not read.
- Dependencies: scan the text for references to earlier work ("Week 3", "prior assignment", "worksheet", "bring forward"). For each, find the matching earlier assignment in the list and name it so the team knows what to bring.

## Step 3 — Write the Kickoff Brief

Publish it as a shared doc (Claude Docs connector; load the docs skill first) titled `Kickoff Brief — <assignment name>`. Sections, in order:

1. **Header**: due date/time (local), points, deliverables, Canvas link, team meeting length (ask the user; default 60 min).
2. **The assignment in 3 sentences.** Plain language.
3. **5-minute pre-read: what every teammate does before the meeting**: read this brief + which earlier work to open (from dependencies).
4. **Where the points are**: table of rubric criterion → points → what earns it (from rubric text) → *the trap* (the easiest way to lose it, inferred; mark inferred items "(inferred)").
5. **Easy-to-miss requirements**: hard rules buried in the description (time limits, # of presenters, cameras, page limits, required named items). Quote short phrases exactly.
5b. **Suggested workspace (inferred)**: the shared files the team should set up, one job each (same rules as the Slack section in Step 4).
6. **Decide together**: 3–5 open questions only the team can answer, each tied to a rubric criterion. Questions, never answers.
7. **Suggested work split** by *workstream* (not by person), plus two cross-cutting roles that always exist in group work: **Story owner** (one narrative, consistent voice/titles across everyone's parts) and **Clock owner** (deadlines, integration checkpoint, rehearsal). Include an integration checkpoint and a rehearsal date counted back from the due date.
8. **Meeting agenda** time-boxed to the meeting length: ≤10% on context (the brief already did it), most time on "Decide together", end with owners + dates.
9. **Definition of done** checklist built from the deliverables + rubric.
10. **What this brief can't know**: things said only in class/Slack, attached files not read, judgment calls marked (inferred).

Style: short, scannable, no filler, no invented requirements. If something is ambiguous in Canvas, put it under "Decide together" or "Ask the instructor", don't guess.

## Step 3.5 — Self-check before publishing

Re-read the finished brief against the raw Canvas text and rubric:
- Every rubric criterion appears in "Where the points are".
- Every hard rule (time limits, counts, required items) appears in "Easy-to-miss requirements".
- No requirement in the brief is missing from Canvas (no invented rules). Inferences are labeled (inferred).
- "Decide together" contains questions, not answers.
Fix anything that fails, then publish.

These checks are Evals 2–4 in `~/group-kickoff-agent/EVALS.md` (guardrail, rubric coverage, faithfulness). Apply their pass bars. After each run, append one row to that file's "Results log" with the scores you measured; write "pending" for anything that needs a human (Eval 5 needs the grade).

## Step 4 — Share with the team (Slack)

1. Draft the message in exactly this shape (Slack markdown, under ~1,500 characters, every line taken from the brief):

```
*Kickoff brief: <assignment name>* (due <weekday, date, time>, <points> pts)
Please read before our meeting (~5 min): <brief link>

*Deliverables*
• <deliverable 1, with its hard limit, e.g. "Slide deck, 6 slides in fixed order">
• <deliverable 2>
• <deliverable 3 if any>

*Suggested workspace* (inferred)
• <one shared file per job, e.g. "1 working doc: brainstorm + interview notes + sources">
• <e.g. "1 shared slide deck for the final slides (one template = one voice)">
• <e.g. "1 doc for the written deliverable / AI prompt log", only if the assignment needs it>

*Top things to decide together*
1. <decide-together question 1>
2. <decide-together question 2>
3. <decide-together question 3>

*What may come up next*
• After the meeting: <the owner/date decisions to lock, e.g. "assign Story owner + Clock owner, set rehearsal date">
• Checkpoints: <integration checkpoint and rehearsal, counted back from the due date>
• Coming up on Canvas: <the next assignment this one feeds into and how, if Canvas says so; otherwise the next group assignment from Radar>
```

"Suggested workspace" is inferred from the deliverables and always labeled (inferred). Rules: suggest the fewest shared files that cover the work, each with one job; slides due → ONE shared deck everyone edits (not separate files merged later); a written report/PDF due → one shared doc; research, brainstorm or analysis steps → one working doc to collect notes, quotes and sources; an AI-usage write-up required → a running prompt log inside the working doc. Name the file type (doc, slide deck, spreadsheet), not a specific brand, unless the course or team already uses one. Only suggest; never create files.

"What may come up next" uses only Canvas facts (linked or upcoming assignments) and the brief's own checkpoints; mark anything else (inferred).
2. Ask which channel or DM if not already known (search channels with the Slack connector's channel search).
3. **Show the exact message + destination and wait for the user's explicit yes.** Then send with the Slack connector.
4. If Slack is not connected, give the message ready to paste.

Reply with: the Radar table (if run), the brief's link, and what was posted where.

## Background mode (scheduled runs, nobody watching)

Runs at 12 AM and 12 PM daily. Nobody is there to answer questions, so:
- **Memory:** `~/group-kickoff-agent/briefed.json` lists assignments already briefed. Skip those; append new ones after briefing.
- **Window:** group assignments due in the next 10 days, not yet submitted, not in the log.
- **Ambiguity:** if two group assignments share a due date (e.g. a project + its presentation), make ONE combined brief covering both.
- **Never post to the team channel directly.** Save the team message (Step 4 format: Deliverables / Suggested workspace / Top things to decide together / What may come up next) as an unsent **draft** in #<team-channel> (channel `<TEAM_CHANNEL_ID>`) and send a **DM to the user** (<your name>, `<YOUR_SLACK_USER_ID>`) with the brief link, the drafted team message, and "Open #<team-channel> to review and send the draft."
- **Nothing new → do nothing** (no DM).
- **Something broke** (Chrome not connected, Canvas login screen, Slack error) → DM the user one line saying what broke and how to fix it (e.g. "Log into Canvas in Chrome, then I'll retry at the next run"). Never type credentials.
