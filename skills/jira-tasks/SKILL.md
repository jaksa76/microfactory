---
name: jira-tasks
description: Jira backend operations for the microfactory — find eligible issues, claim one with race verification, create, view, comment, and transition, all via the acli CLI. Used by plan-next and implement-next when the configured backend is jira. Not for general Jira questions.
---

# Jira task operations (via acli)

All operations use the `acli` CLI. Read `acli-reference.md` in this skill's directory for command details, and use `acli <command> --help` when unsure of exact flags. Configuration comes from `.microfactory/config.yaml` (`project` = Jira project key, `jira_account_id` = your assignee id).

Before the first operation, verify authentication with `acli jira auth status`; if unauthorized, stop and tell the user to run `! acli jira auth login`.

## Finding eligible issues

Search with `acli jira workitem search --jql "<JQL>" --json --paginate`. Build the JQL by mode and the `plan_by_default` setting; always `ORDER BY rank ASC` and require `assignee is EMPTY`:

| Mode | plan_by_default | JQL conditions |
|---|---|---|
| planning | true | `status = "To Do" AND (labels NOT IN ("skip-plan") OR labels is EMPTY)` |
| planning | false | `status = "To Do" AND labels IN ("needs-plan")` |
| implementation | true | `status = "Plan Approved" OR (status = "To Do" AND labels IN ("skip-plan"))` |
| implementation | false | `status = "To Do" AND (labels NOT IN ("needs-plan") OR labels is EMPTY)` |

Example (implementation, plan_by_default false):

```
acli jira workitem search --jql "project = \"MYPROJ\" AND assignee is EMPTY AND status = \"To Do\" AND (labels NOT IN (\"needs-plan\") OR labels is EMPTY) ORDER BY rank ASC" --json --paginate
```

## Claiming an issue

Other factory sessions may be claiming concurrently, so follow this optimistic-locking protocol exactly:

1. Search with the eligibility JQL above. If the result is empty, there is no work — stop.
2. Take the **first** issue (highest rank).
3. Assign it to yourself: `acli jira workitem assign --key <KEY> --assignee <jira_account_id> --yes`
4. **Wait about 10 seconds**, then re-fetch the issue and check the assignee's account id.
5. If the assignee is not you, another worker won the race — go back to step 1.
6. Transition the issue: to `Planning` (planning mode) or `In Progress` (implementation mode). If the transition is unavailable, warn and continue.
7. Fetch the issue's summary and description: `acli jira workitem view <KEY> --json`

## Creating a work item

```
acli jira workitem create --project "<project key>" --type "Task" --summary "<title>" --description "<body>"
```

Confirm the exact flags with `acli jira workitem create --help` before relying on them (the bundled reference lists the subcommand but not its flag set), and add labels the same way `edit` does if the project uses them. Leave the new item **unassigned** in its initial status: creation is not part of the claim protocol, and an assigned item is not claimable by any session.

## Other operations

- **View**: `acli jira workitem view <KEY> --json` (summary, description, labels, assignee).
- **Comment**: `acli jira workitem comment create --key <KEY> --body "<text>"`
- **Transition**: `acli jira workitem transition --key <KEY> --status "<status>" --yes`

Statuses used by the factory workflow: `Planning`, `Awaiting Plan Review`, `Plan Approved` (set by a human reviewer), `In Progress`, `In Review`, `Done`. Transitions may not exist in every workflow — treat failed transitions after work is done as warnings, not errors.
