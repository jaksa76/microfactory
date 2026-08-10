---
name: fill-backlog
description: Sweep the project for improvements and turn the chosen ones into backlog tasks — runs the design, testing, library, and UI finders, merges and ranks their findings, then creates the ones the user approves in the configured task backend. Use when the user wants to generate a backlog, find work for the factory, or top up the task list.
---

# Fill the backlog

One sweep: choose areas → run the finders → merge and rank → **get approval** → create tasks → report.

This is the only skill that creates tasks. The finders themselves never write to the backend.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop.

Load the backend skill matching `backend` (`jira-tasks`, `github-tasks`, or `todo-tasks`) — you will need its **create** operation and its eligibility rules.

## 2. Choose the areas to sweep

All four by default:

| Area | Finder |
|---|---|
| design | `find-design-improvements` |
| testing | `find-testing-improvements` |
| libraries | `find-library-improvements` |
| ui | `find-ui-improvements` |

An argument narrows it (`design`, `testing`, `libraries`, `ui`, or several). Ask only if the user's intent is ambiguous — a bare invocation means all four.

## 3. Run the finders

Load each area's skill in turn and follow it. Keep each area's findings separate at this stage so one area failing or finding nothing does not cost you the others. Note explicitly which areas produced nothing and why (no UI, no dependency manifest, no tests) — an honest empty area is a useful result.

## 4. Merge and rank

Each finder ranked only within its own area, so the combined list needs work:

- **Drop duplicates** — the same file or the same underlying problem surfacing in two areas is one task. Keep the framing that makes the work clearest.
- **Collapse** findings that would land as a single commit.
- **Re-rank globally** by payoff relative to effort, across all areas.
- **Check the backlog first**: read the existing open items from the backend and drop anything already there. Re-proposing work that is already queued is the fastest way to make this skill useless.

## 5. Get approval before creating anything

Present the merged list — title, area, effort, payoff, one line of problem — and let the user choose which become tasks. **Create nothing until they have chosen.** An automated sweep that opens thirty issues by itself ruins a real backlog, and on Jira or GitHub those writes are visible to the user's whole team.

If the user wants everything, that is their call to make explicitly.

## 6. Create the approved tasks

Through the backend skill's create operation, one task per finding:

- **Title** — the finding's title, self-contained (the todo backend has nowhere else to put context).
- **Body** — the finding's Problem, Proposed change, Payoff, and Risk.
- **Labels/tags** — the area (`design`, `testing`, `libraries`, `ui`), plus `needs-plan` when the finding is `L` effort or the finder flagged it as risky, so it goes through planning before implementation. `S` and `M` findings go straight to the implementable pool.
- **Leave every new task unassigned and in its initial status** so factory sessions can claim it.

## 7. Report

Tell the user:

- The tasks created, with keys or URLs.
- What was skipped as a duplicate or as already in the backlog.
- Which areas produced nothing.
- That the factory will pick these up on its next iterations (`/microfactory:start`), and that `needs-plan` ones go through `plan-next` and wait for their approval first.
