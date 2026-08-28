---
name: implement-next
description: Run one implementation iteration of the microfactory — claim the next implementation-eligible issue from the configured backend, follow its approved plan (or, with auto_plan on, write one first and treat it as approved), implement it, run the tests, verify the behaviour (UI, e2e), review the code, push (directly or via a feature branch + PR), and update the issue. Intended as a /loop target (e.g. /loop 20m /microfactory:implement-next); also takes an optional issue key to implement a specific issue.
---

# Implement the next issue

One implementation iteration: claim → settle the plan → implement → test → verify → review → push → update the issue → record what you learned.

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

## 5. Settle the plan

Three cases, in this order:

1. **`plans/<KEY>.md` exists** — it is the **approved plan**, use it as written. An existing plan is followed whatever the setting below says.
2. **No plan and `auto_plan: true`** — write one now: load the `plan-next` skill and follow its analysis and plan-writing steps (its steps 4 and 5) to produce `plans/<KEY>.md`. Treat that plan as **approved** and carry straight on to the implementation — do not stop for review and do not transition the issue to `Awaiting Plan Review`.
3. **No plan and `auto_plan` off** — implement without a plan file. The issue text and its resolved refinement thread are the specification. Still do `plan-next`'s **analysis** (its step 4) and carry the conclusions into the implementation: what is dropped here is the document and its self-approval, never the thinking.

A missing `auto_plan` key means **false** — planning happens in the planner loop unless a project asks for it here. Note that this is the opposite of `deep_review`, where an absent key means true; the two booleans sit next to each other in config and do not default the same way.

This setting never lets an issue skip planning that was meant to have one: a `needs-plan` issue is not implementation-eligible until its plan is approved, whatever `auto_plan` says. Eligibility is the backend skill's rule and this does not touch it.

## 6. Implement

- Follow the plan. If implementing reveals the plan is wrong, correct the plan file too so it matches what was built. Where step 5 settled on no plan file, the issue and its thread play that role and there is nothing to correct — step 10 decides whether the divergence is worth recording anywhere.
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

**Who reviews depends on configuration.** A *cold review* — handing the diff to a fresh agent — is the default; `deep_review: false` in config turns it off for the project. Labels override the config per issue: `needs-review` forces the cold review on, `skip-review` forces it off, and `skip-review` wins if an issue somehow carries both. A missing `deep_review` key means **true** — the cold review is what an unconfigured factory does.

This setting chooses *who* reviews, never *whether*. When the cold review is off, review the diff yourself against the same list of concerns below, and say in your report that the cold review was skipped and which setting or label skipped it — so a later reader can tell a deliberate skip from a missed step.

**When it is on, hand the review to a fresh agent, not to yourself.** You know why every line is there, so you read the diff as you intended it rather than as it is written; an agent seeing it cold does not. Give it only what a reviewer would legitimately have:

- the issue text and the answers settled in its refinement thread,
- `plans/<KEY>.md`,
- the diff of this iteration,
- the project's style guide, `CLAUDE.md`/`AGENTS.md`, or whatever conventions the repo documents.

Do **not** give it your reasoning for the choices you made — that is the part being tested. Ask it for findings only: correctness, security problems in the code this iteration added, drift from the project's conventions, and simplicity, decoupling and clarity where the project states no rule. If the environment cannot run a separate agent, review the diff yourself against the same list and say so — that is a capability the environment lacks, not a configuration choice, and the two are worth telling apart in the report.

**Wait for the findings.** The review is worth having only if it lands before the commit does, so nothing in step 9 begins while a review is outstanding — a reviewer still reading while the push goes out has produced a post-mortem, not a review. Where delegated agents run detached unless told otherwise, ask for the form that hands you the findings before you continue. If you use more than one reviewer, launch them together and await all of them; the rule is that findings precede the commit, not that reviewers run one at a time.

The requirement outlives this step: what has to be true before the commit is that no finding is still outstanding. Step 9.1 re-runs the tests when the review changed something, so a finding arriving after that run invalidates it — act on the finding and run the tests again before committing.

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
2. Transition the issue to `Done`. The commit is the record; nothing is commented on the issue.

Issue updates after the code is pushed are best effort: if the PR comment or the transition fails, warn and finish — the pushed code is the important artifact.

If you cannot complete the implementation (e.g. tests cannot be made to pass), do **not** transition the issue to Done: push nothing broken, comment on the issue describing what blocked you, and leave it In Progress for a human. That comment is about this issue's state, which is the backlog's business; step 10 still applies separately to anything durable you learned on the way.

## 10. Record what you learned

An iteration that went as predicted records nothing — finish and stop. But when it diverged from what the issue, the thread or the plan led you to expect, that divergence exists nowhere but in your context right now, and the next iteration starts cold. Write it where it will be read again.

**Nothing goes on the issue.** The backlog is a progress indicator, not a filing cabinet; a comment there is read once, by nobody. Route what you learned by its shape:

| What you learned | Where it goes |
|---|---|
| A durable fact about this project, short and relevant to any future change | `CLAUDE.md` — or `AGENTS.md`, whichever the project uses |
| A procedure — how something is done here, in steps | a new skill, whose `description` names when to use it |
| Reference material too bulky for either | `docs/LESSONS-<subject>.md`, pointed at from whichever file above names the trigger |
| A judgement about what ought to change | `docs/LESSONS.md` |

Four rules make this safe to do unattended:

- **Facts go to the instruction file; judgements go to `LESSONS.md`.** A fact is something a future agent could verify in the repo — this build needs that step, that module contradicts the convention it documents. A judgement is what *should* be true. You have seen one iteration, which is not enough to set policy, so write what you found and leave what to do about it to a human.
- **Integrate, do not append.** A fact belongs in the section that already covers its subject, in a line or two of the surrounding prose. No `## Lessons` heading, no dated entries, no issue keys — git already records when and why. If it cannot be said that briefly inside prose that already exists, it is not a fact yet; it is a judgement.
- **Read the destination first, and say nothing if it is already there.** This is what stops a standing condition — an environment that cannot run subagents, a service unavailable in this checkout — from writing the same thing every iteration for as long as the condition lasts. Recurring is not the same as new.
- **Not about this diff, and not about this session.** The commit records the diff. Where you happened to be running is a fact about the environment, not about the project, and belongs in neither destination.

When a `LESSONS-*.md` file is the destination, the pointer to it must name the **situation**, not the topic: "when changing a config key, read `docs/LESSONS-config.md` first" gets followed; "see `docs/LESSONS.md`" does not, because nothing tells a reader when it applies.

A one-off — this plan was wrong, this story did not fit — is usually not durable and is recorded nowhere. It becomes a judgement worth writing down only once you can say it as a pattern: plans here are consistently wrong about X.

Commit what you wrote with the same change, so the lesson and the work that produced it arrive together.

Handle **one issue per iteration**.
