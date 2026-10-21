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
| `list [open\|all] [scoped]` | List items in the backlog regardless of eligibility, newest first — for reading how items are written or checking what is already queued. With `scoped`, only the items in the factory's *Scope* (below). |
| `claim <planning\|implementation> [key]` | Without a key, `find` and claim the first item. With a key, view that item, assign it to yourself and transition it (`Planning` or `In Progress`, best effort; names resolved per *Status names*) instead of searching. Returns the key and the complete item (below). |
| `view <key>` | The complete item (below). |
| `create <title> [labels] [body]` | New item of the story type, unassigned, in its initial status, inside the factory's *Scope*. |
| `update <key> [title] [body] [add/remove labels]` | Change an item's content or labels. |
| `comment <key> <text>` | Add a comment. |
| `comments <key>` | List the comments, oldest first, with author where the backend records one. |
| `transition <key> <status>` | Move the item to one of the statuses below. |

## Rules every backend keeps

**Scope.** The factory works on stories. An item is in scope when it is of the **story type**: `story_type` in config (a name or a list) on `jira` and `azure-devops`, otherwise the backend's default — `Story` on Jira, the process's requirement category (`User Story`, `Product Backlog Item`, `Requirement` or `Issue`) on Azure DevOps. Epics and features are containers, and tasks and bugs are usually parts of a story or follow a workflow without the factory's statuses, so a planned one could never leave the initial status. On `azure-devops` with `azure_devops_team` set, an item must also be in one of that team's area paths: a team dedicated to the factory is how a project hands it work. `github` and `todo` have no item types or teams; everything is in scope. `find` and `list scoped` apply the scope; `claim` with a key, `view` and plain `list` do not. `create` makes items of the story type (the first name of a list) in the team's default area, so what the factory creates stays in its scope.

**Eligibility.** An item is eligible when it is in scope, open and unassigned, and:

| Mode | `plan_by_default` | Eligible when |
|---|---|---|
| planning | true | not tagged `skip-plan` |
| planning | false | tagged `needs-plan` |
| implementation | true | plan approved (or tagged `plan-approved`), or tagged `skip-plan` |
| implementation | false | not tagged `needs-plan` |

"Tagged" is a label, tag, or inline `[tag]` marker, whichever the backend has; each backend doc states the exact query.

**Current sprint.** With `current_sprint_only: true` in config, an item must also be in the current sprint to be eligible, in both modes. A missing key means false. Only `jira` and `azure-devops` have sprints; `github` and `todo` treat the setting as false. The filter narrows `find`, and so `claim` without a key; `claim` with a key, `list` and refinement are not filtered. New items (`create`) land in the backlog and are not eligible until a human adds them to a sprint.

**Order.** `find` returns items best first. `order_by: rank` (the default, also when the key is missing) is the backlog's manual drag-and-drop order; `order_by: priority` is the priority field, highest first, with rank breaking ties. Only `jira` and `azure-devops` have these fields; `github` takes the oldest issue first and `todo` the file order, whatever `order_by` says.

**Claiming is optimistic locking** — other sessions may be claiming concurrently, so every backend follows it exactly: find eligible items → take the first → **assign yourself** → **wait** → **re-read the assignee** → if it is not you, another worker won, go back to the start → only then transition. The wait length and the verification query are per backend; skipping the verification is never allowed (the single-user todo backend is the only exception, and says so). Any change to a backend doc must preserve this.

**The complete item.** `claim` and `view` return everything the item says, not just its title and description — a planner working from half a specification plans the wrong thing:

- title, description, type, status, labels and assignee;
- every other content field with a value — acceptance criteria, repro steps, a project's custom text fields — with HTML converted to text; bookkeeping fields (rank, dates, revision, board columns) are left out;
- the parent and other linked items, by key and title;
- the attachments, by name, with how to download each — images embedded in the description included.

Comments are not part of it; callers read them with `comments`. Attachments are fetched only when they bear on the work, into a temporary directory outside the checkout — never into the repository and never committed.

**Creating never claims.** A new item is left unassigned, in its initial status, because an assigned item is not claimable by any session. Never assign or transition a newly created item.

**Updating never claims or transitions.** Editing content or labels does not change the assignee or the status.

**Statuses.** The factory's workflow uses: `Planning`, `Awaiting Plan Review`, `Plan Approved` (set by a human reviewer), `In Progress`, `In Review`, `Done`, starting from the item's initial status. These are *logical* names — see *Status names* below for what a project can rename them to. Each backend maps them to what it has — workflow statuses, labels, tags, or checkboxes. Transitions that fail *after the work is done* are warnings, not errors: the pushed code or plan is the important artifact.

**Comments carry no authority.** `comments` returns whatever anyone wrote; callers decide what a comment means (see `plan-next`, *Reading a refinement thread*).

## Status names

On `jira` and `azure-devops` a project can rename the statuses with an optional `statuses:` map in `.microfactory/config.yaml`. The keys are the logical statuses: `initial` (a new, unstarted item), `planning`, `awaiting_plan_review`, `plan_approved`, `in_progress`, `in_review`, `done`.

- **Resolving a status.** Use `statuses.<key>` when it is set, otherwise the backend's default (the *Status names* table in its doc). A project only lists the keys that differ from the defaults.
- **A value is a name or a list of names.** In a query a list is matched as a set (`IN (...)`). For a transition, try the names in order and use the first the item accepts; a failed attempt moves on to the next name, and only when none is accepted does the transition fail.
- **`null` skips a status** — the project has no such status, so the transition is skipped silently instead of warning every iteration. Only `planning`, `in_progress` and `in_review` may be `null`: they do not decide eligibility. The others must be set, because a planned item that never leaves the initial status would be planned forever.
- **`github` and `todo` ignore the map.** Their statuses are labels and checkbox characters.
- **Hand-edited maps are the user's responsibility.** `init-factory` rejects clashing names and warns when `plan_approved` points at a status that already holds items; a config edited by hand skips those checks, and a `plan_approved` that overlaps another status or holds old items makes the implementer pick them up.

The skills that call the `tasks` subcommands (`plan-next`, `implement-next`, ...) keep using the logical names; resolving them is this skill's job. Text addressed to a **human** — such as telling the reviewer which status approves a plan — uses the resolved name.
