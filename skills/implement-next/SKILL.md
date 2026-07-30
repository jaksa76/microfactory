---
name: implement-next
description: Run one implementation iteration of the microfactory — claim the next implementation-eligible issue from the configured backend, follow its approved plan (or write one first and treat it as approved), implement it, run the tests, push (directly or via a feature branch + PR), and update the issue. Intended as a /loop target (e.g. /loop 20m /microfactory:implement-next); also takes an optional issue key to implement a specific issue.
---

# Implement the next issue

One implementation iteration: claim → plan (if there is no plan yet) → implement → test → push → update the issue.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop.

Load the backend skill matching `backend` (`jira-tasks`, `github-tasks`, or `todo-tasks`) for all task operations below.

## 2. Prepare the working tree

- If the working tree has uncommitted changes, report it and stop this iteration.
- Pull the latest default branch.

## 3. Claim an issue

- **No argument**: claim the next implementation-eligible issue using the backend skill's claim protocol (implementation mode). If there is no eligible issue, say "No implementation work available." and end the iteration — the loop cadence handles waiting.
- **With an issue key argument**: view that issue, assign it to yourself, and transition it to `In Progress` (best effort) instead of searching.

## 4. Decide on a feature branch

Use a feature branch when `feature_branches: true` in config **or** the issue has a `needs-branch` label; a `skip-branch` label overrides both and forces direct-to-default. When branching, create `feature/<KEY>` from the latest default branch (reset it to the default branch if it already exists).

## 5. Make sure there is a plan

- If `plans/<KEY>.md` exists, it is the **approved plan** — use it as written.
- Otherwise write one now: load the `plan-next` skill and follow its analysis and plan-writing steps (its steps 4 and 5) to produce `plans/<KEY>.md`. Treat that plan as **approved** and carry straight on to the implementation — do not stop for review and do not transition the issue to `Awaiting Plan Review`.

## 6. Implement

- Follow the plan. If implementing reveals the plan is wrong, correct the plan file too so it matches what was built.
- Implement the change following existing style and conventions, and add or update tests.
- Run the project's tests; the work is not done until they pass. Commit with a brief, descriptive message, including the plan file when this iteration wrote it.

## 7. Push and update the issue

**Feature-branch mode:**
1. Push the feature branch.
2. Open a PR against the default branch: title `[<KEY>] <summary>`, body referencing the issue.
3. Comment the PR URL on the issue and transition it to `In Review`.

**Direct mode:**
1. Push the default branch.
2. Comment `Implemented: <summary>` on the issue and transition it to `Done`.

Issue comments and transitions after the code is pushed are best effort: if they fail, warn and finish — the pushed code is the important artifact.

If you cannot complete the implementation (e.g. tests cannot be made to pass), do **not** transition the issue to Done: push nothing broken, comment on the issue describing what blocked you, and leave it In Progress for a human.

Handle **one issue per iteration**.
