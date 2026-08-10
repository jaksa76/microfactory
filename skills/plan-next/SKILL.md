---
name: plan-next
description: Run one planning iteration of the microfactory — claim the next planning-eligible issue from the configured backend, explore the codebase, write plans/<KEY>.md, push it, and put the issue up for plan review. Intended as a /loop target (e.g. /loop 10m /microfactory:plan-next); also takes an optional issue key to plan a specific issue.
---

# Plan the next issue

One planning iteration: claim → explore → write plan → push → request review. Do **not** implement anything in this mode.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop.

Load the backend skill matching `backend` (`jira-tasks`, `github-tasks`, or `todo-tasks`) for all task operations below.

## 2. Prepare the working tree

- If the working tree has uncommitted changes, report it and stop this iteration — never plan on top of someone's half-done work.
- Pull the latest default branch.

## 3. Claim an issue

- **No argument**: claim the next planning-eligible issue using the backend skill's claim protocol (planning mode). If there is no eligible issue, say "No planning work available." and end the iteration — the loop cadence handles waiting.
- **With an issue key argument**: view that issue, assign it to yourself, and transition it to `Planning` (best effort) instead of searching.

**Then read the issue's comments** through the backend skill's list-comments operation. Refinement happens in the thread, not in the issue body — `refine-story` posts questions there and the product owner answers beneath them, so the thread is part of the specification. See *Reading a refinement thread* at the end of this skill.

## 4. Analyze the change

Before writing the plan, explore the codebase to understand what the issue touches and do the analysis the change calls for. Depending on the nature of the issue:

- **UI feature** — run the relevant part of the application and take screenshots of the affected screens, then decide on the best UX/UI approach based on what already exists.
- **Database changes** — look at the existing schema and the rationale behind it before proposing modifications, so the change stays consistent with current design.
- **Software design changes** — when the change requires altering the application's design, evaluate the pros and cons of the candidate approaches and pick one deliberately.
- **Anything else** — do whatever additional analysis you deem necessary to plan the change well.

Fold the conclusions of this analysis into the plan below (in the Summary and Risks sections as appropriate).

## 5. Write the plan

Write `plans/<KEY>.md` with these sections:

- **Summary** — what the issue asks for and the chosen approach
- **Files to Change** — concrete files, with what changes in each
- **Implementation Steps** — discrete, ordered steps
- **Testing** — tests to add or modify
- **Risks / Edge Cases** — risks, edge cases, dependencies

The plan must be executable by another agent with no context beyond the repo and the plan itself.

## 6. Publish and request review

1. Commit **only** the plan file (`Add plan for <KEY>`) and push to the default branch.
2. Comment on the issue with a link to the plan file (derive the blob URL from the git remote when it is a GitHub repo; otherwise mention the repo path `plans/<KEY>.md`).
3. Transition the issue to `Awaiting Plan Review`.

Steps 2–3 are best effort: if they fail, warn and finish — the pushed plan is the important artifact. A human reviews the plan and transitions the issue to `Plan Approved` (or edits the plan); `implement-next` picks it up from there.

Handle **one issue per iteration**.

## Reading a refinement thread

`refine-story` never edits the issue — it only comments — so an issue whose body still reads as vague may already have been settled in its thread. Its comment opens with the marker line `**Refinement questions**` and closes with a `_Posted by microfactory refine-story…_` footer. Everything between the two is what was asked; everything after the footer is somebody's reply. Rely on the marker rather than on comment authorship — the todo backend records no authors, and on a shared project a reply may come from any account.

Resolve the thread like this:

- **An answer wins.** Replies are terse (`1: yes`, `2: b`) and refer to questions by number. Where free text contradicts the options that were offered, follow the free text — the human is right.
- **An unanswered question takes the default stated in the question.** Silence was declared a safe answer when it was asked.
- **A "Taken as given" line nobody corrected stands** as a statement about the product.
- **Ignore other factory comments** — plan links, blocked notes, PR URLs. They are not requirements.

Nothing removes the `needs-refinement` tag automatically; a human does that when the answers satisfy them. Its presence on a claimed issue is not a reason to stop.

Write the settled answers into the plan's **Summary**, so the plan stands on its own and `implement-next` does not have to re-derive them.
