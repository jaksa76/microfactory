---
name: plan-next
description: Run one planning iteration of the microfactory — claim the next planning-eligible issue from the configured backend, explore the codebase, write plans/<KEY>.md, push it, and put the issue up for plan review. Intended as a /loop target (e.g. /loop 10m /microfactory:plan-next); also takes an optional issue key to plan a specific issue.
---

# Plan the next issue

One planning iteration: claim → explore → write plan → push → request review. Do **not** implement anything in this mode.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop.

Load the `tasks` skill for all task operations below; each is one of its subcommands.

## 2. Prepare the working tree

Plan in the checkout as it is. It is often shared with an implementer session or a person, so it may have uncommitted changes or be on another branch. Neither stops planning. Planning never stashes, resets or switches branches, and never commits anything but the plan.

- `git fetch origin`. When the checkout is on the default branch, also pull it (`git pull --no-rebase`). A pull that fails because of local changes is not fatal: warn and continue with what is there.
- If the checkout is not on the default branch, or has uncommitted changes, warn: name the branch and the changed files. The code you explore then includes work that is not on the default branch, but the plan still describes a change to the default branch. Do not build on that work as if it were there. Where it matters, read the default branch's version of a file (`git show origin/<default>:<path>`), and say in the plan's **Risks / Edge Cases** which branch and uncommitted changes it was written next to.
- Then load the `overrides` skill for `plan-next` and follow what it returns — after the pull, so a change to the override files that was just merged applies to this iteration.

## 3. Claim an issue

- **No argument**: claim the next planning-eligible issue with the `tasks` skill's `claim planning`. If there is no eligible issue, say "No planning work available." and end the iteration — the loop cadence handles waiting.
- **With an issue key argument**: `claim planning <key>` — view that issue, assign it to yourself, and transition it to `Planning` (best effort) instead of searching.

**Then read the issue's comments** with the `tasks` skill's `comments`. Refinement happens in the thread, not in the issue body — `refine-story` posts questions there and the product owner answers beneath them, so the thread is part of the specification. See *Reading a refinement thread* at the end of this skill.

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

1. Commit **only** the plan file (`Add plan for <KEY>`) to the default branch, or to the branch a `plan-next` override names, and push it. Nothing else in the checkout goes into the commit, except the item itself on the `todo` backend (below).
   - **The checkout is on that branch:** `git commit -m "Add plan for <KEY>" -- plans/<KEY>.md` commits that path alone, whatever else is staged or modified. Push; if the push is rejected, `git pull --no-rebase` and push again.
   - **The checkout is on another branch:** commit without switching, so the checkout's branch, index and files stay as they are. Build the commit on `origin/<branch>` with a temporary index: point `GIT_INDEX_FILE` at a scratch file, then `git read-tree origin/<branch>`, `git update-index --add --cacheinfo 100644,<blob>,plans/<KEY>.md` (the blob id from `git hash-object -w plans/<KEY>.md`), `git write-tree`, `git commit-tree <tree> -p origin/<branch> -m "Add plan for <KEY>"`, and `git push origin <commit>:refs/heads/<branch>`. If the push is rejected, fetch and start again from `read-tree`. Then delete the local `plans/<KEY>.md`: the plan now lives on that branch, and an untracked copy would block a later checkout or pull of it.
2. Comment on the issue with a link to the plan file (derive the blob URL from the git remote when it is a GitHub repo; otherwise mention the repo path `plans/<KEY>.md`).
3. Transition the issue to `Awaiting Plan Review`.

Steps 2–3 are best effort: if they fail, warn and finish — the pushed plan is the important artifact.

**On the `todo` backend, do steps 2–3 before step 1**, and commit the item with the plan. Its claim, note and status are edits to the TODO file, so they are not visible to anyone until they are pushed. The item's *block* is its line plus the indented note lines beneath it. Only that block goes into the commit; other edits to the file stay out.

- **On the target branch:** if the item's block is the only local change to the TODO file, add the file to the commit (`git commit -m "Add plan for <KEY>" -- plans/<KEY>.md <todo-file>`). If the file has other local changes, leave it out and warn that the item's status is only in the checkout.
- **On another branch:** add a second entry to the temporary index. Its content is `origin/<branch>`'s version of the TODO file with the item's block replaced by the checkout's block. Leave the checkout's TODO file as it is, because it is what the next iteration reads. A human reviews the plan and transitions the issue to its approved status — `Plan Approved`, or whatever `statuses.plan_approved` names on Jira and Azure DevOps — (or edits the plan); `implement-next` picks it up from there.

Handle **one issue per iteration**.

## Reading a refinement thread

`refine-story` never edits the issue — it only comments — so an issue whose body still reads as vague may already have been settled in its thread. Its comment opens with the marker line `**Refinement questions**` and closes with a `_Posted by microfactory refine-story…_` footer. Everything between the two is what was asked; everything after the footer is somebody's reply. Rely on the marker rather than on comment authorship — the todo backend records no authors, and on a shared project a reply may come from any account.

Resolve the thread like this:

- **An answer wins.** Replies are terse (`1: yes`, `2: b`) and refer to questions by number. Where free text contradicts the options that were offered, follow the free text — the human is right.
- **An unanswered question takes the default stated in the question.** Silence was declared a safe answer when it was asked.
- **A "Taken as given" line nobody corrected stands** as a statement about the product.
- **Ignore other factory comments** — plan links, blocked notes, PR URLs, and any `**Iteration note**` comments left in older threads before iterations began writing what they learned into the project's own files. They are not requirements; where one disagrees with the thread's answers, the answers win.

Nothing removes the `needs-refinement` tag automatically; a human does that when the answers satisfy them. Its presence on a claimed issue is not a reason to stop.

Write the settled answers into the plan's **Summary**, so the plan stands on its own and `implement-next` does not have to re-derive them.
