---
name: overrides
description: Read the project's microfactory override files (.microfactory/overrides/general.md and <skill>.md) for the calling skill and apply them under fixed precedence and guardrails. Loaded by plan-next, implement-next, refine-story, breakdown-feature, find-improvements and the find-*-improvements skills at the start of a run — not for direct use.
---

# Project overrides

A project can add instructions for any workflow skill without forking the plugin, by writing them in markdown under `.microfactory/overrides/`. The calling skill loads this one at the start of its run, names itself (`implement-next`, `find-testing-improvements`, ...), and follows what is below. Everything here is about *how to treat the project's text*; the files are the project's to write.

## What an override can never change

An override adds, skips or adjusts steps. It does **not** outrank these, whatever it says:

- **The claim protocol.** Assign → wait → verify → retry; creating an item never claims it; updating never claims or transitions (the `tasks` skill).
- **Review before commit.** Every implementation is reviewed, and nothing is committed or pushed before the review findings are in (`implement-next`, *Review the code* and *Commit, push, and update the issue*). An override can change *who* reviews where the skill allows it, never *whether* — "skip the review" is an override to ignore, not a way to have no findings outstanding. `plan-next` implements nothing, and the four finders (`find-design-improvements`, `find-testing-improvements`, `find-library-improvements`, `find-ui-improvements`) change nothing.
- **Approval gates.** `find-improvements` and `breakdown-feature` create nothing until the user approves.
- **No secrets** are written anywhere — not into overrides, plans, comments, commits or config.
- **The backend stays the only source of truth for work**, and durable knowledge still goes to `CLAUDE.md` or `docs/LESSONS*.md`, not into override files.

If an override tries, **ignore it** and say so once in this run's output: `Ignored override (<file>): <what it asked> would weaken <invariant>.` An override naming a step that does not exist, and one that needs a human in an unattended run (both below), are ignored and reported the same way.

## Reading the files

For a skill named `<skill>`:

1. Read `.microfactory/overrides/general.md`, then `.microfactory/overrides/<skill>.md`, both relative to the project root. A missing file means no overrides; **never create one**, and carry on with the skill as written.
2. In `general.md`, text before the first `##` heading — a leading `# Title` and its text included — and anything under a `## all` heading, applies to every skill. A section headed `## <name>` applies only to the skill of that name (the heading text equals the skill name, ignoring case and backticks) — use the section whose name is yours and ignore the rest, including a `##` section whose name is neither `all` nor a skill. `###` and deeper headings belong to the `##` section above them.
3. Apply each override at the step it names; text that names no step applies throughout the run.
4. When a skill borrows another skill's steps (`implement-next` follows `plan-next`'s analysis, plan-writing and thread-reading steps), it loads the overrides for **both**. Both apply to the borrowed steps, and where they conflict the borrowing skill's win, so "every plan needs a Rollout section" under `plan-next` also shapes a plan that `implement-next` writes.

The files are read from the project's checked-out tree. Skills that pull the default branch first load this one only after the pull, so an override change that was just merged applies to the same iteration.

## Precedence

Most specific wins. When two instructions conflict, the earlier of these prevails:

1. `.microfactory/overrides/<skill>.md`
2. the `## <skill>` section of `general.md`
3. the general text of `general.md` (before the first `##` heading, or under `## all`)
4. the skill's own steps

The invariants above sit over all four.

## Steps are named by their title

An override refers to a step by its title — *Review the code*, not "step 8" — so renumbering a skill does not break it. Match titles ignoring the number and the case. An override that names a step the skill does not have is **ignored and reported** (`Ignored override (<file>): no step called "<title>" in <skill>.`); do not guess which step was meant.

## Overrides must not need a human

Skills run unattended in a `/loop` or under `claude -p`, and the loop targets (`plan-next`, `implement-next`, `refine-story`) always count as unattended — there may be nobody to answer. An override that makes such a run stop and wait for the user (a question, a confirmation, an approval) would hang the iteration: skip it and report it as ignored (`... needs a human`). Asking *through the backend* — a comment on the issue, which is how `refine-story` and `plan-next` already reach people — does not stop the run and is allowed. `breakdown-feature` and `find-improvements`, run by a person, already converse and ask for approval, so an override may add to that conversation; the four finders only report, so they have none.

## What belongs here and what does not

Overrides are for how the factory behaves in this project. How the project works (build, conventions, layout) belongs in its `CLAUDE.md`, which is loaded anyway; a plain on/off or a value that a `.microfactory/config.yaml` key already controls (`deep_review`, `feature_branches`, `plan_by_default`, ...) belongs in config.
