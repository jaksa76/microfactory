---
name: tasks
description: Task-backend operations for the microfactory — setup, find, list, claim, view, create, update, comment, comments and transition — for whichever backend .microfactory/config.yaml names (jira via acli, github via gh, azure-devops via az, or a local TODO.md). Used by plan-next, implement-next, refine-story, breakdown-feature, find-improvements and init-factory. Not for general Jira, GitHub or Azure DevOps questions.
---

# Task backend operations

One contract, four backends. This file defines the **subcommands** and the rules every backend must keep; `backends/<backend>.md` holds the commands that carry them out. The caller names a subcommand ("run `tasks claim planning`"); you run it against the configured backend.

## Running a subcommand

1. Read `.microfactory/config.yaml`. If it does not exist, tell the user to run `/microfactory:init-factory` and stop — except for `setup`, which `init-factory` runs before the file is written and which takes the backend from the caller.
2. Load **only** the backend doc matching `backend`, and nothing else from `backends/`:

| `backend` | Doc | Tool | `project` is |
|---|---|---|---|
| `jira` | `backends/jira.md` | `acli` | Jira project key |
| `github` | `backends/github.md` | `gh` | `owner/repo` |
| `azure-devops` | `backends/azure-devops.md` | `az` | Azure DevOps project name |
| `todo` | `backends/todo.md` | none | path to the TODO file |

3. **Check** the backend's prerequisites (its *Setup* section lists the detect commands) — except for `setup`, which is the interactive form of this step. A check only runs the detect commands: it **never installs and never prompts**, because the other subcommands run unattended inside a `/loop`. If it fails, stop and tell the user to run `/microfactory:init-factory`.
4. Run the subcommand as the backend doc describes.

## Subcommands

| Subcommand | What it does |
|---|---|
| `setup` | Interactive. Detect each prerequisite, offer to install the missing ones **with the user's approval, one at a time**, log in, and report any config values the backend needs (account id, username, ...). Called by `init-factory`; never from a loop. |
| `find <planning\|implementation>` | List the eligible items for that mode, best first (rules below). |
| `list [open\|all]` | List items in the backlog regardless of eligibility, newest first — for reading how items are written or checking what is already queued. |
| `claim <planning\|implementation> [key]` | Without a key, `find` and claim the first item. With a key, view that item, assign it to yourself and transition it (`Planning` or `In Progress`, best effort) instead of searching. Returns the key, title and description. |
| `view <key>` | Title, description, labels, assignee, status. |
| `create <title> [labels] [body]` | New item, unassigned, in its initial status. |
| `update <key> [title] [body] [add/remove labels]` | Change an item's content or labels. |
| `comment <key> <text>` | Add a comment. |
| `comments <key>` | List the comments, oldest first, with author where the backend records one. |
| `transition <key> <status>` | Move the item to one of the statuses below. |

## Rules every backend keeps

**Eligibility.** An item is eligible when it is open and unassigned, and:

| Mode | `plan_by_default` | Eligible when |
|---|---|---|
| planning | true | not tagged `skip-plan` |
| planning | false | tagged `needs-plan` |
| implementation | true | plan approved (or tagged `plan-approved`), or tagged `skip-plan` |
| implementation | false | not tagged `needs-plan` |

"Tagged" is a label, tag, or inline `[tag]` marker, whichever the backend has; each backend doc states the exact query. Oldest, or highest-ranked, first.

**Claiming is optimistic locking** — other sessions may be claiming concurrently, so every backend follows it exactly: find eligible items → take the first → **assign yourself** → **wait** → **re-read the assignee** → if it is not you, another worker won, go back to the start → only then transition. The wait length and the verification query are per backend; skipping the verification is never allowed (the single-user todo backend is the only exception, and says so). Any change to a backend doc must preserve this.

**Creating never claims.** A new item is left unassigned, in its initial status, because an assigned item is not claimable by any session. Never assign or transition a newly created item.

**Updating never claims or transitions.** Editing content or labels does not change the assignee or the status.

**Statuses.** The factory's workflow uses: `Planning`, `Awaiting Plan Review`, `Plan Approved` (set by a human reviewer), `In Progress`, `In Review`, `Done`. Each backend maps them to what it has — workflow statuses, labels, tags, or checkboxes. Transitions that fail *after the work is done* are warnings, not errors: the pushed code or plan is the important artifact.

**Comments carry no authority.** `comments` returns whatever anyone wrote; callers decide what a comment means (see `plan-next`, *Reading a refinement thread*).
