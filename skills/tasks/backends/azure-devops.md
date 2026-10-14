# Azure DevOps Boards backend (via az)

All operations use `az boards` from the `azure-devops` extension of the Azure CLI. Configuration comes from `.microfactory/config.yaml`: `project` = the Azure DevOps project name, `azure_devops_org` = the organization URL (`https://dev.azure.com/myorg`), `azure_devops_user` = your identity (sign-in email / UPN) for self-assignment.

Pass `--org <azure_devops_org>` on every `az boards` and `az devops` call, and `--project <project>` on `query` and `create` (`show` and `update` take only the item id — confirm with `az boards work-item update --help`). Do not rely on `az devops configure` defaults: one machine may serve several projects.

The issue key is `ADO-<id>`, where `<id>` is the work item's integer id; strip the prefix before passing `--id`. If the caller gives a bare number, treat it as the id. Use `-o json` and adapt quoting to the user's shell (WIQL contains single quotes).

## Setup

Prerequisites: the Azure CLI, its `azure-devops` extension, and a login.

- **Detect** (also the check run before every other subcommand): `az --version`, `az extension show --name azure-devops`, then `az account show`.
- **Install `az`** if missing, after the user approves: `winget install --exact --id Microsoft.AzureCLI` on Windows, `brew install azure-cli` on macOS, Microsoft's documented package-repository or script method on Linux (https://learn.microsoft.com/cli/azure/install-azure-cli). A fresh install may need a new shell before `az` is on `PATH` — say so rather than retrying blindly.
- **Install the extension** if missing, after the user approves: `az extension add --name azure-devops`. Do **not** enable `extension.use_dynamic_install` — it would make `az` install extensions silently later, which is the opposite of asking.
- **Log in** if `az account show` fails: suggest `! az login` (or `! az devops login` with a personal access token).
- **Config values**: ask for the organization URL → `azure_devops_org` and the project name → `project`; take the user identity from `az ad signed-in-user show --query userPrincipalName -o tsv`, or ask → `azure_devops_user`.
- **List the states**: the factory's statuses are work item states (next section), and a process has to be customized to have them. For each type the factory will work on (`Task` and the story type), list its states with `az devops invoke --area wit --resource workitemtypes --route-parameters project=<project> type="<type>" --org ... --api-version 7.1 --query "states[].name" -o tsv`, and give the lists to `init-factory`, which maps the factory's statuses onto them.

`setup` stops after listing the states: no mapping exists yet at that point, so whether the process has the states the factory needs is checked later, under *Checking the mapping*.

## Status names

The factory's statuses are the work item's own `System.State`, as on Jira; only the opt-in flags (`needs-plan`, `skip-plan`, `needs-refinement`, …) are **tags** (`System.Tags`), as Jira's are labels. Each is overridable under `statuses:` in config (see the `tasks` skill, *Status names*); a default that is a list is how the standard processes' own names are covered.

| Key | Default state(s) |
|---|---|
| `initial` | `New`, `To Do`, `Proposed` (`To Do` in Basic, `Proposed` in CMMI) |
| `planning` | `Planning` |
| `awaiting_plan_review` | `Awaiting Plan Review` |
| `plan_approved` | `Plan Approved` |
| `in_progress` | `In Progress`, `Active`, `Doing`, `Committed` |
| `in_review` | `In Review` |
| `done` | `Done`, `Closed` |

For a transition, a list is tried in order and the first name the work item accepts wins (see *Other operations*).

`System.Tags` is one `; `-separated string and a write **replaces** it. To add or remove a tag, re-`show` the item, compute the new list, and write all of it: `az boards work-item update --id <N> --fields "System.Tags=a; b; c"`. This is safe because only the item's current owner edits its tags.

## Checking the mapping

Run by `init-factory` once the `statuses:` map is settled (defaults for any key not in it) — never by `setup`, and never from a loop. Every status that is not `null` must name at least one state that each of the types the factory works on has, judged against the state lists `setup` produced. If one does not, tell the user which states to add and where — a process administrator adds them under Organization settings → Process → (an inherited copy of the project's process) → the type → States — and do not continue until they have. System processes (Basic, Agile, Scrum, CMMI) cannot be edited; the project has to be moved to an inherited process first. Give `plan_approved` the **Proposed** category (not started) and the other new states **In Progress**; the queries match state names, so the categories only affect board columns and reports.

## Finding eligible issues

`az boards query --wiql "<WIQL>" --org ... --project ... -o json` returns the matching ids; `az boards work-item show --id <N>` gives the fields. Common WIQL (`Epic` and `Feature` are containers, never one-iteration units):

```
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = '<project>'
  AND [System.AssignedTo] = ''
  AND [System.WorkItemType] NOT IN ('Epic', 'Feature')
  AND <state and tag conditions>
ORDER BY [Microsoft.VSTS.Common.StackRank] ASC, [System.Id] ASC
```

`[System.StateCategory]` is not a WIQL field, so conditions match state names. `INITIAL` below stands for `[System.State] IN (<initial>)` and `APPROVED` for `[System.State] IN (<plan_approved>)`, where `<initial>` and `<plan_approved>` are the configured names, single-quoted and comma-separated (`'New', 'To Do', 'Proposed'` by default; escape a `'` by doubling it). `NOT CONTAINS` also matches items with no tags:

| Mode | plan_by_default | Conditions |
|---|---|---|
| planning | true | `INITIAL AND [System.Tags] NOT CONTAINS 'skip-plan'` |
| planning | false | `INITIAL AND [System.Tags] CONTAINS 'needs-plan'` |
| implementation | true | `(APPROVED OR (INITIAL AND [System.Tags] CONTAINS 'skip-plan'))` |
| implementation | false | `INITIAL AND [System.Tags] NOT CONTAINS 'needs-plan'` |

Expand `INITIAL` and `APPROVED` in the query; WIQL has no macros. A planned item leaves the initial state, so it cannot be planned twice.

WIQL does **not** error on a state name that no type has — it just matches nothing, so a state renamed after `init-factory` ran makes the factory look idle. When a `find` returns no items, say which state names the query used, so that an idle factory and a stale name can be told apart.

## Claiming an issue

Other factory sessions may be claiming concurrently, so follow this optimistic-locking protocol exactly:

1. Query eligible items as above. If none, there is no work — stop.
2. Take the **first**.
3. Assign yourself: `az boards work-item update --id <N> --assigned-to <azure_devops_user> --org ...`
4. **Wait about 5 seconds**, then `az boards work-item show --id <N>` and read `System.AssignedTo.uniqueName`.
5. If it is not you (compare case-insensitively), another worker won the race — go back to step 1.
6. Transition the item: to the `planning` state (planning mode) or the `in_progress` state (implementation mode). The state is changed only after the assignment is verified, so racing workers never overwrite each other's. If the transition is rejected, warn and continue; if the status is `null`, skip it.
7. Read the title and description from the `show` output (`System.Title`, `System.Description`, HTML).

## Creating an issue

```
az boards work-item create --type "Task" --title "<title>" --description "<body>" --fields "System.Tags=label-a; label-b" --org ... --project ...
```

`Task` exists in every standard process. When the caller wants a story, use the process's story type (`User Story`, `Product Backlog Item`, `Issue`) — take it from the types of existing backlog items rather than guessing. Drop `--fields` when there are no labels. Leave the item **unassigned and in its initial state**: creation is not part of the claim protocol, and an assigned item is not claimable.

## Updating an issue

`az boards work-item update --id <N> --title "<title>" --description "<body>"` — pass only what you are changing. Labels are tags: read-modify-write as above. Updating never changes the assignee or the state.

## Other operations

- **List**: `az boards query` with the WIQL above minus the assignee, state and tag conditions, `ORDER BY [System.Id] DESC`; add `AND [System.State] NOT IN (<done>, 'Removed')` for open items only (`<done>` formatted like `<initial>`).
- **View**: `az boards work-item show --id <N> -o json` (`System.Title`, `System.Description`, `System.Tags`, `System.State`, `System.AssignedTo`).
- **Comment**: `az boards work-item update --id <N> --discussion "<text>" --org ...`
- **List comments**: `az devops invoke --area wit --resource comments --route-parameters project=<project> workItemId=<N> --api-version 7.1-preview --org ...` — oldest first, each with `createdBy.displayName`, `createdDate`, `text`. The text is **HTML**: strip the tags and decode entities before looking for a marker line such as `**Refinement questions**`.
- **Transition**: `az boards work-item update --id <N> --state "<state>" --org ...`, with the state resolved from the table under *Status names*: for a list, try each name in order and use the first the work item accepts (a rejected name moves on to the next); for a `null` status, do nothing. **Awaiting Plan Review** also **clears the assignee** (`--assigned-to ""`), so the implementer can claim the item once a human reviewer sets it to the `plan_approved` state.

Treat failed state changes after the work is done as warnings, not errors.
