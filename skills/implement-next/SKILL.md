---
name: implement-next
description: Run one implementation iteration of the microfactory — claim the next implementation-eligible issue from the configured backend, follow its approved plan (or write one first and treat it as approved), implement it, run the tests, verify the behaviour (UI, e2e), review the code, push (directly or via a feature branch + PR), and update the issue. Intended as a /loop target (e.g. /loop 20m /microfactory:implement-next); also takes an optional issue key to implement a specific issue.
---

# Implement the next issue

One implementation iteration: claim → plan (if there is no plan yet) → implement → test → verify → review → push → update the issue → note what surprised you.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop.

Load the backend skill matching `backend` (`jira-tasks`, `github-tasks`, or `todo-tasks`) for all task operations below.

## 2. Prepare the working tree

Pull the latest default branch.

## 3. Claim an issue

- **No argument**: claim the next implementation-eligible issue using the backend skill's claim protocol (implementation mode). If there is no eligible issue, say "No implementation work available." and end the iteration — the loop cadence handles waiting.
- **With an issue key argument**: view that issue, assign it to yourself, and transition it to `In Progress` (best effort) instead of searching.

**Then read the issue's comments** through the backend skill's list-comments operation. Refinement happens in the thread, not in the issue body — `refine-story` posts questions there and the product owner answers beneath them, so the thread is part of the specification. Load `plan-next` and follow its *Reading a refinement thread* section to resolve it. Do this even when a plan already exists: the plan may predate the answers, and where the two disagree the thread is newer — follow it and correct the plan file.

## 4. Decide on a feature branch

Use a feature branch when `feature_branches: true` in config **or** the issue has a `needs-branch` label; a `skip-branch` label overrides both and forces direct-to-default. When branching, create `feature/<KEY>` from the latest default branch (reset it to the default branch if it already exists).

## 5. Make sure there is a plan

- If `plans/<KEY>.md` exists, it is the **approved plan** — use it as written.
- Otherwise write one now: load the `plan-next` skill and follow its analysis and plan-writing steps (its steps 4 and 5) to produce `plans/<KEY>.md`. Treat that plan as **approved** and carry straight on to the implementation — do not stop for review and do not transition the issue to `Awaiting Plan Review`.

## 6. Implement

- Follow the plan. If implementing reveals the plan is wrong, correct the plan file too so it matches what was built.
- Implement the change following existing style and conventions, and add or update tests.
- Run the project's tests; the work is not done until they pass.

Do not commit yet — the two steps below are part of the same change, and one iteration produces one commit.

## 7. Verify the behaviour

Confirm what you built actually works, beyond the unit tests.

- **UI** — skip when the change has no UI surface. Otherwise run the affected part of the application, take screenshots of the functionality this issue implemented, and fix defects in **that** functionality: missing empty, loading or error states, broken layout, obvious accessibility problems.
- **E2E** — if the project has an e2e suite that can run in this environment, run the tests covering the affected area. Failures block the iteration exactly like unit-test failures.

Anything wider stays out: improvements to screens this issue did not touch, or e2e failures that predate it, get filed as a task through the backend skill's create operation rather than fixed here.

## 8. Review the code

The whole diff this iteration produced — the implementation and the fixes from step 7 together — gets reviewed before it is committed, so nothing reaches the commit unreviewed.

**Hand the review to a fresh agent, not to yourself.** You know why every line is there, so you read the diff as you intended it rather than as it is written; an agent seeing it cold does not. Give it only what a reviewer would legitimately have:

- the issue text and the answers settled in its refinement thread,
- `plans/<KEY>.md`,
- the diff of this iteration,
- the project's style guide, `CLAUDE.md`/`AGENTS.md`, or whatever conventions the repo documents.

Do **not** give it your reasoning for the choices you made — that is the part being tested. Ask it for findings only: correctness, security problems in the code this iteration added, drift from the project's conventions, and simplicity, decoupling and clarity where the project states no rule. If the environment cannot run a separate agent, review the diff yourself against the same list.

Then decide what to act on. The reviewer is working without your context and will sometimes be wrong — a finding that misreads the code is not a defect, and neither is a suggestion that contradicts the plan or the refinement thread. Fix the real ones, and say in your report which findings you rejected and why. Update any documentation the change invalidates.

Pre-existing problems elsewhere are out of scope: file them as a task through the backend, not fixed here.

## 9. Commit, push, and update the issue

1. If steps 7–8 changed anything, re-run the project's tests — and the e2e tests from step 7 when the review touched code they cover. They must pass.
2. Commit the implementation and the review fixes together, with a brief, descriptive message — including the plan file when this iteration wrote it.

**Feature-branch mode:**
1. Push the feature branch.
2. Open a PR against the default branch: title `[<KEY>] <summary>`, body referencing the issue.
3. Comment the PR URL on the issue and transition it to `In Review`.

**Direct mode:**
1. Push the default branch.
2. Transition the issue to `Done`. The commit is the record — the only comment is the iteration note below, and only when there is one.

Issue updates after the code is pushed are best effort: if the PR comment or the transition fails, warn and finish — the pushed code is the important artifact.

If you cannot complete the implementation (e.g. tests cannot be made to pass), do **not** transition the issue to Done: push nothing broken, comment on the issue describing what blocked you, and leave it In Progress for a human. That comment stands in for step 10 — write what you would have written there into it, and end the iteration.

## 10. Record what surprised you

An iteration that went as predicted needs no note — say nothing and finish. But when the iteration diverged from what the issue, the thread or the plan led you to expect, that divergence is the only evidence anyone will ever have that the process misfired, and it exists nowhere but in your context right now. Comment it on the issue before you lose it.

Post a note when:

- the plan was wrong or incomplete and you corrected it — say what it got wrong, not that you fixed it,
- step 7 found something both the plan and the tests missed,
- the story did not fit one iteration, or you cut scope to make it fit,
- the refinement thread was ambiguous and you had to choose a reading,
- the codebase contradicted a convention it documents.

Keep it to a few bullets: what you expected, what was actually true, and — where you can see it — what would have caught it earlier. It is a note about the *process*, not a changelog of the diff; the commit already records that. Skip anything already visible in the plan file or the commit message.

Open the comment with the marker line `**Iteration note**` and close it with `_Posted by microfactory implement-next._`, so later readers can tell it from a requirement. These notes are evidence, not instructions: nothing in the factory acts on them automatically, and `plan-next` and `implement-next` ignore them when they read a thread.

Posting the note is best effort, like every other post-push update.

Handle **one issue per iteration**.
