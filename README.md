# Group Kickoff Agent

An AI assistant that gets a student team aligned **before** the first meeting on a new group assignment.

## The problem

When a new group assignment drops, everyone reads a different part of the instructions (or none). The first third of the meeting goes to figuring out *what we're supposed to do* instead of *how to do it well*. On one assignment, our 5-person team planned a 1-hour meeting that ran 1.5 hours, and the grader later called our presentation "disjointed, as if different people prepared each slide."

## What it does

Twice a day (12 AM and 12 PM), the agent:

1. **Checks Canvas** for new group assignments due in the next 10 days (using Canvas's own group-assignment flag, not a guess from the title)
2. **Reads** the assignment page, the grading rubric, and the earlier assignments it builds on
3. **Writes a one-page Kickoff Brief** as a shared doc: where the points are, easy-to-miss rules, a suggested workspace, questions the team must decide together, a work split and a 60-minute agenda
4. **Checks its own brief** against the rubric (see [EVALS.md](EVALS.md))
5. **Drafts a Slack message** for the team channel (Deliverables / Suggested workspace / Top things to decide together / What may come up next) and DMs you
6. **You press Send.** Nothing reaches the team without a human.

It keeps a small log (`briefed.json`) so it never briefs the same assignment twice. You can also just ask it in plain language: *"prep a kickoff brief for our final project."*

**Design rule:** the agent prepares the team; it never does the assignment. It hands over questions, not answers.

## How the pieces connect

```
Canvas (via your logged-in Chrome) ──► Claude reads assignment + rubric + earlier work
                                            │
                                            ▼
                               Kickoff Brief (shared doc) ──► self-check (Evals 2–4)
                                            │
                                            ▼
                     Slack: unsent draft in team channel + DM to you ──► you press Send
```

## Tools used

- **Claude Code** (Claude desktop app): runs the agent and the twice-daily schedule
- **Claude in Chrome**: reads Canvas through your own logged-in browser (read-only)
- **Canvas LMS**: source of assignments and rubrics
- **Claude Docs**: publishes the shareable Kickoff Brief
- **Slack connector**: creates the draft and the DM

## Setup (about 10 minutes, no coding)

1. Install the Claude desktop app and the Claude in Chrome extension; sign in to both with the same account
2. In the Claude app, go to **Settings → Connectors** and connect **Slack**
3. Log into Canvas in Chrome
4. Copy `skill/group-kickoff/` into `~/.claude/skills/`
5. Make a folder `~/group-kickoff-agent/`, copy `EVALS.md` into it, and copy `briefed.example.json` there as `briefed.json` (empty its `briefed` list)
6. Replace the placeholders in `SKILL.md` and `scheduled-task/PROMPT.md`: `<COURSE_ID>`, `<your-school>`, `<team-channel>`, `<TEAM_CHANNEL_ID>`, `<YOUR_SLACK_USER_ID>`, `<your name>`. Tip: just ask Claude "find my Slack channel ID and my user ID"
7. In the Claude app, create a scheduled task with the contents of `scheduled-task/PROMPT.md`, set to run at 12 AM and 12 PM daily
8. Press **Run now** once and approve the permission prompts, so later runs don't get stuck waiting

## Evaluations

The agent is probabilistic, so it is tested against 5 evaluations: 1 happy path, 1 guardrail, 3 thresholds. Full details, example inputs and the results log are in [EVALS.md](EVALS.md).

| # | Evaluation | Pass bar | First result |
|---|---|---|---|
| 1 | Happy path: scheduled run end to end | All 5 checks true | Pending |
| 2 | Guardrail: never acts for the team | 0 violations | Pass |
| 3 | Rubric and hard-rule coverage | 6/6 criteria, ≥10/11 rules | 6/6, 11/11 |
| 4 | Faithfulness to Canvas | ≥19/20 claims sourced, 0 invented | 1 error caught and fixed |
| 5 | Predicts where points get lost | ≥60% | 70% |

## Limitations

- **Only knows what's on Canvas.** Expectations said in class, on Slack, or only in grader feedback are invisible (it missed "slide titles should state a point" for this reason)
- **Output varies run to run.** One brief borrowed wording from grader feedback instead of Canvas; the faithfulness test caught it. A human reviews before anything is sent
- **Doesn't read attached PDFs** or files that live outside Canvas
- **Runs only while your computer is on** and the Claude app is open, with an active Canvas login in Chrome. If logged out, it DMs you
- **Some parts are guesses** (combining same-day assignments, suggested workspace, checkpoints), always labeled "(inferred)"
- **No help during the meeting**: a live "are we drifting?" check was left out to keep the build testable
- **The first test wasn't blind**: the grader's feedback was seen before the test brief was written; the blind case is pending grading

## Disclaimer

This is a student project created at Carnegie Mellon University for educational purposes only. It is not affiliated with, endorsed by, or associated with Anthropic (Claude, Claude Code, Claude in Chrome, Claude Docs), Instructure (Canvas), Slack Technologies / Salesforce (Slack), or Google (Chrome). All trademarks and brand names are the property of their respective owners.
