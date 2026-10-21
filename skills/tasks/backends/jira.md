# Jira backend (via acli)

All operations use the `acli` CLI. Read `jira-acli-reference.md` in this directory for command details, and use `acli <command> --help` when unsure of exact flags. Configuration comes from `.microfactory/config.yaml` (`project` = Jira project key, `jira_account_id` = your assignee id, `story_type` = the issue type(s) the factory works on, default `Story`).

## Setup

Prerequisites: `acli` installed and logged in to the Jira site.

- **Detect** (also the check run before every other subcommand): `acli --version`, then `acli jira auth status`.
- **Install `acli`** if missing, after the user approves: follow Atlassian's instructions at https://developer.atlassian.com/cloud/acli/guides/install-acli/ for the user's OS (Homebrew on macOS: `brew tap atlassian/homebrew-acli && brew install acli`; a downloaded binary on Windows and Linux). Open a new shell if the binary is not on `PATH` afterwards.
- **Log in** if `auth status` fails: the user must do it interactively — suggest `! acli jira auth login` (they need the site, their email, and an API token).
- **Config values**: ask for the Jira site host (e.g. `mycompany.atlassian.net`) → `jira_site`; take the account id from `acli jira auth status` output, or ask → `jira_account_id` (needed for self-assignment).
- **Story type**: ask which issue type the project's stories are, pre-filled with `Story`. Many projects work in `Task`s instead, or in both; only an answer other than `Story` goes into `story_type` (a name or a list).

## Status names

The workflow's statuses, each overridable under `statuses:` in config (see the `tasks` skill, *Status names*):

| Key | Default status |
|---|---|
| `initial` | `To Do` |
| `planning` | `Planning` |
| `awaiting_plan_review` | `Awaiting Plan Review` |
| `plan_approved` | `Plan Approved` |
| `in_progress` | `In Progress` |
| `in_review` | `In Review` |
| `done` | `Done` |

Below, `<initial>` and `<plan_approved>` stand for the configured names, double-quoted and comma-separated (`"To Do"`; with a list, `"To Do", "Open"`). Escape a `"` or `\` inside a name. `<story_type>` is formatted the same way (`"Story"` by default).

## Finding eligible issues

Search with `acli jira workitem search --jql "<JQL>" --json --paginate`. Build the JQL by mode and the `plan_by_default` setting, and always require `assignee is EMPTY` and `issuetype IN (<story_type>)`:

| Mode | plan_by_default | JQL conditions |
|---|---|---|
| planning | true | `status IN (<initial>) AND (labels NOT IN ("skip-plan") OR labels is EMPTY)` |
| planning | false | `status IN (<initial>) AND labels IN ("needs-plan")` |
| implementation | true | `(status IN (<plan_approved>) OR (status IN (<initial>) AND labels IN ("skip-plan")))` |
| implementation | false | `status IN (<initial>) AND (labels NOT IN ("needs-plan") OR labels is EMPTY)` |

With `current_sprint_only: true`, add `AND sprint in openSprints()`. It matches an issue in any open sprint, and nothing when no sprint is open, as on a Kanban board or between sprints.

End with `ORDER BY rank ASC`, or `ORDER BY priority DESC, rank ASC` when `order_by: priority` (Jira sorts priorities in the priority scheme's order, so `DESC` puts the highest first).

When a `find` returns no items, say which issue types and statuses the query used: a project whose backlog is made of `Task`s looks idle under the default `Story` until `story_type` names them.

Example (implementation, plan_by_default false, default statuses, current_sprint_only true, order_by priority):

```
acli jira workitem search --jql "project = \"MYPROJ\" AND assignee is EMPTY AND issuetype IN (\"Story\") AND status IN (\"To Do\") AND (labels NOT IN (\"needs-plan\") OR labels is EMPTY) AND sprint in openSprints() ORDER BY priority DESC, rank ASC" --json --paginate
```

## Claiming an issue

Other factory sessions may be claiming concurrently, so follow this optimistic-locking protocol exactly:

1. Search with the eligibility JQL above. If the result is empty, there is no work — stop.
2. Take the **first** issue.
3. Assign it to yourself: `acli jira workitem assign --key <KEY> --assignee <jira_account_id> --yes`
4. **Wait about 10 seconds**, then re-fetch the issue and check the assignee's account id.
5. If the assignee is not you, another worker won the race — go back to step 1.
6. Transition the issue: to the `planning` status (planning mode) or the `in_progress` status (implementation mode). If the transition is unavailable, warn and continue; if the status is `null`, skip it.
7. Read the complete item (*Viewing a work item*).

## Viewing a work item

`acli jira workitem view <KEY> --fields "*all" --json` returns every field (confirm the `--fields` syntax with `acli jira workitem view --help`). From it, build the complete item (the `tasks` skill, *The complete item*):

- **Fields**: `summary`, `description`, `issuetype`, `status`, `labels`, `assignee`, and every `customfield_*` with a text value — acceptance criteria are usually one. A custom field's id does not say what it is; `acli jira field --help` shows how to list the fields with their names. Leave out bookkeeping: rank, dates, sprint, story points, watchers, votes.
- **Links**: `parent`, `subtasks` and `issuelinks` (each with its link type and the other issue's key and summary).
- **Attachments**: `attachment` lists each file's name and content URL; `acli jira workitem attachment --help` shows how to list and download them.

## Creating a work item

```
acli jira workitem create --project "<project key>" --type "<story type>" --summary "<title>" --description "<body>" --label "label-a,label-b"
```

The story type is the first name of `story_type`, `Story` by default. `--label` takes a comma-separated list; drop it if the project uses no labels. Leave the new item **unassigned** in its initial status: creation is not part of the claim protocol, and an assigned item is not claimable by any session.

## Updating a work item

```
acli jira workitem edit --key "<KEY>" --summary "<title>" --description "<body>" --yes
```

Pass only what you are changing. Labels are `--labels "a,b"` to set and `--remove-labels "c"` to remove — that is how a `needs-plan` or `needs-refinement` tag comes off once the work it asked for is done.

**Always pass `--yes`.** Without it `edit` prompts for confirmation, which hangs a session running unattended in a `/loop`. The same goes for `assign` and `transition`.

Updating an item's content never changes its assignee or status — editing is not claiming.

## Other operations

- **List**: `acli jira workitem search --jql "project = \"MYPROJ\" ORDER BY created DESC" --json --paginate`, adding `AND statusCategory != Done` for open items only and `AND issuetype IN (<story_type>)` for `scoped`.
- **View**: as *Viewing a work item*.
- **Comment**: `acli jira workitem comment create --key <KEY> --body "<text>"`
- **List comments**: `acli jira workitem comment list --key <KEY> --json --paginate` (oldest first by default; `--order` takes `created`/`updated` with a `+`/`-` prefix).
- **Transition**: `acli jira workitem transition --key <KEY> --status "<status>" --yes`, with the status resolved from the table under *Status names* (for a list, try each name in order and use the first the workflow accepts; for a `null` status, do nothing).

Statuses used by the factory workflow are those in the table under *Status names*; `plan_approved` is set by a human reviewer. Transitions may not exist in every workflow — treat failed transitions after work is done as warnings, not errors.
