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

## Statuses are tags

Azure DevOps states differ by process template (Basic: To Do/Doing/Done; Agile: New/Active/Resolved/Closed; Scrum: New/Approved/Committed/Done; CMMI: Proposed/Active/Resolved/Closed) and cannot be extended from the CLI, so the factory's statuses are stored as **tags** (`System.Tags`), as GitHub stores them as labels. Only **Done** changes the real state. That is also why eligibility uses `[System.StateCategory] = 'Proposed'`, which means "open, not started" in every process.

`System.Tags` is one `; `-separated string and a write **replaces** it. To add or remove a tag, re-`show` the item, compute the new list, and write all of it: `az boards work-item update --id <N> --fields "System.Tags=a; b; c"`. This is safe because only the item's current owner edits its tags.

## Finding eligible issues

`az boards query --wiql "<WIQL>" --org ... --project ... -o json` returns the matching ids; `az boards work-item show --id <N>` gives the fields. Common WIQL (`Epic` and `Feature` are containers, never one-iteration units):

```
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = '<project>'
  AND [System.StateCategory] = 'Proposed'
  AND [System.AssignedTo] = ''
  AND [System.WorkItemType] NOT IN ('Epic', 'Feature')
  AND <tag conditions>
ORDER BY [Microsoft.VSTS.Common.StackRank] ASC, [System.Id] ASC
```

Tag conditions by mode and `plan_by_default`. `NOT CONTAINS` also matches items with no tags:

| Mode | plan_by_default | Tag conditions |
|---|---|---|
| planning | true | `[System.Tags] NOT CONTAINS 'skip-plan' AND [System.Tags] NOT CONTAINS 'awaiting-plan-review' AND [System.Tags] NOT CONTAINS 'plan-approved'` |
| planning | false | `[System.Tags] CONTAINS 'needs-plan' AND [System.Tags] NOT CONTAINS 'awaiting-plan-review' AND [System.Tags] NOT CONTAINS 'plan-approved'` |
| implementation | true | `([System.Tags] CONTAINS 'plan-approved' OR [System.Tags] CONTAINS 'skip-plan')` |
| implementation | false | `[System.Tags] NOT CONTAINS 'needs-plan'` |

The `awaiting-plan-review` / `plan-approved` exclusions in planning mode matter: a planned item stays in the `Proposed` category and is unassigned again once the planner hands it over, so without them it would be planned forever.

## Claiming an issue

Other factory sessions may be claiming concurrently, so follow this optimistic-locking protocol exactly:

1. Query eligible items as above. If none, there is no work — stop.
2. Take the **first**.
3. Assign yourself: `az boards work-item update --id <N> --assigned-to <azure_devops_user> --org ...`
4. **Wait about 5 seconds**, then `az boards work-item show --id <N>` and read `System.AssignedTo.uniqueName`.
5. If it is not you (compare case-insensitively), another worker won the race — go back to step 1.
6. Only now add the tag — `in-progress`, or in planning mode `in-planning` instead — by read-modify-write. Tags are written only after the assignment is verified, so racing workers never overwrite each other's.
7. Read the title and description from the `show` output (`System.Title`, `System.Description`, HTML).

## Creating an issue

```
az boards work-item create --type "Task" --title "<title>" --description "<body>" --fields "System.Tags=label-a; label-b" --org ... --project ...
```

`Task` exists in every standard process. When the caller wants a story, use the process's story type (`User Story`, `Product Backlog Item`, `Issue`) — take it from the types of existing backlog items rather than guessing. Drop `--fields` when there are no labels. Leave the item **unassigned and in its initial state**: creation is not part of the claim protocol, and an assigned item is not claimable.

## Updating an issue

`az boards work-item update --id <N> --title "<title>" --description "<body>"` — pass only what you are changing. Labels are tags: read-modify-write as above. Updating never changes the assignee, the status tags or the state.

## Other operations

- **List**: `az boards query` with the WIQL above minus the assignee and tag conditions, `ORDER BY [System.Id] DESC`; add `AND [System.StateCategory] <> 'Completed'` for open items only (`'Completed'` is the category name for Done/Closed states).
- **View**: `az boards work-item show --id <N> -o json` (`System.Title`, `System.Description`, `System.Tags`, `System.State`, `System.AssignedTo`).
- **Comment**: `az boards work-item update --id <N> --discussion "<text>" --org ...`
- **List comments**: `az devops invoke --area wit --resource comments --route-parameters project=<project> workItemId=<N> --api-version 7.1-preview --org ...` — oldest first, each with `createdBy.displayName`, `createdDate`, `text`. The text is **HTML**: strip the tags and decode entities before looking for a marker line such as `**Refinement questions**`.
- **Transition**:

| Status | Change |
|---|---|
| In Progress | add `in-progress` |
| Planning | add `in-planning`, remove `in-progress` |
| Awaiting Plan Review | add `awaiting-plan-review`, remove `in-planning`, **clear the assignee** (`--assigned-to ""`) |
| Plan Approved | done by a human reviewer: they add `plan-approved`, remove `awaiting-plan-review` |
| In Review | add `in-review`, remove `in-progress` |
| Done | remove `in-progress` / `in-review`, then set `--state` to the process's completed state: `Done`, or `Closed` if that is rejected (ask the user once if neither works) |

Treat failed tag or state changes after the work is done as warnings, not errors.
