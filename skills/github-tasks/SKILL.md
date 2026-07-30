---
name: github-tasks
description: GitHub Issues backend operations for the microfactory — find eligible issues, claim one with race verification, create, update, view, comment, and transition (statuses are mapped to labels), all via the gh CLI. Used by plan-next and implement-next when the configured backend is github. Not for general GitHub questions.
---

# GitHub Issues task operations (via gh)

All operations use the `gh` CLI against the repo from `.microfactory/config.yaml` (`project` = `owner/repo`, `github_username` = your assignee login). Issue numbers are the issue keys.

Before the first operation, verify authentication with `gh auth status`; if unauthorized, stop and tell the user to run `! gh auth login`.

## Finding eligible issues

List **open, unassigned** issues, oldest first (sort by issue number ascending). Eligibility by mode and the `plan_by_default` setting:

| Mode | plan_by_default | Eligible when |
|---|---|---|
| planning | true | not labeled `skip-plan` |
| planning | false | labeled `needs-plan` |
| implementation | true | labeled `plan-approved` or `skip-plan` |
| implementation | false | not labeled `needs-plan` |

Example (implementation, plan_by_default true):

```
gh issue list --repo owner/repo --state open --json number,title,labels,assignees \
  --jq '[.[] | select((.assignees | length) == 0)
             | select([.labels[].name] | (index("plan-approved") != null or index("skip-plan") != null))]
       | sort_by(.number)'
```

Adapt quoting to the user's shell (jq filters need different escaping on Windows).

## Claiming an issue

Other factory sessions may be claiming concurrently, so follow this optimistic-locking protocol exactly:

1. List eligible issues as above. If none, there is no work — stop.
2. Take the **first** (lowest number).
3. Assign yourself and mark in progress: `gh issue edit <N> --repo owner/repo --add-assignee <github_username> --add-label "in-progress"`
4. **Wait about 5 seconds**, then re-fetch the issue's assignees.
5. If you are not among the assignees, another worker won the race — go back to step 1.
6. Planning mode only: swap labels — `gh issue edit <N> --add-label "in-planning" --remove-label "in-progress"`.
7. Fetch title and body: `gh issue view <N> --repo owner/repo --json title,body`

## Creating an issue

```
gh issue create --repo owner/repo --title "<title>" --body "<body>" --label "<label>"
```

If a label does not exist the command fails — create it (`gh label create <name> --repo owner/repo`) and retry, as with transitions. Leave the new issue **unassigned and open**: creation is not part of the claim protocol, and an assigned issue is not claimable by any session.

## Updating an issue

```
gh issue edit <N> --repo owner/repo --title "<title>" --body "<body>"
```

Pass only the parts you are changing. Updating an issue's content never changes its assignee or status labels — editing is not claiming.

## Other operations

- **View**: `gh issue view <N> --repo owner/repo --json number,title,body,labels,assignees`
- **Comment**: `gh issue comment <N> --repo owner/repo --body "<text>"`
- **Transition**: GitHub has no issue statuses; the factory maps them to labels:

| Status | Label changes |
|---|---|
| In Progress | add `in-progress` |
| Planning | add `in-planning`, remove `in-progress` |
| Awaiting Plan Review | add `awaiting-plan-review`, remove `in-planning`, **remove yourself as assignee** |
| Plan Approved | done by a human reviewer: they add `plan-approved`, remove `awaiting-plan-review` |
| In Review | add `in-review`, remove `in-progress` |
| Done | remove `in-progress`/`in-review`, then `gh issue close <N>` |

Missing labels on the repo make `--add-label` fail — create the label (`gh label create <name> --repo owner/repo`) and retry. Treat failed label changes after work is done as warnings, not errors.
