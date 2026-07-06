---
name: init-factory
description: Set up the microfactory for the current project. Interviews the user, writes .microfactory/config.yaml, authenticates the task backend (Jira via acli, GitHub via gh, or a local TODO.md), and starts the work loop. Use when the user wants to initialize, configure, or reconfigure the factory.
---

# Initialize the Microfactory

Set up the current project so `plan-next` and `implement-next` can run continuously. All commands below are examples — adapt them to the user's OS and shell (Windows/macOS/Linux).

## 1. Interview the user

Ask (with AskUserQuestion where options fit, free text otherwise):

1. **Task backend**: `jira`, `github`, or `todo` (a local TODO.md file).
2. **Project identifier**: Jira project key (e.g. `MYPROJ`), GitHub `owner/repo`, or the path to the TODO file.
3. **Plan by default?** If yes, every issue needs an approved plan before implementation (issues labeled `skip-plan` are exempt). If no, only issues labeled `needs-plan` go through planning.
4. **Feature branches?** If yes, implementation happens on `feature/<KEY>` branches with a pull request (issues labeled `skip-branch` are exempt). If no, work lands on the default branch (issues labeled `needs-branch` still get a branch).
5. **Loop intervals**: how often to look for work. Defaults: implementation every 20 minutes, planning every 10 minutes.
6. **This session's role**: planner or implementer (the other role runs in a second session).

## 2. Authenticate the backend

- **jira**: check `acli jira auth status`. If not authenticated, the user must log in interactively — suggest they run `! acli jira auth login` (they will need their site, email, and an API token). Also ask for the Jira site host (e.g. `mycompany.atlassian.net`). Determine the user's Atlassian account id (visible in `acli jira auth status` output, or ask the user); it is needed for self-assignment.
- **github**: check `gh auth status`. If not authenticated, suggest `! gh auth login`. Determine the username with `gh api user --jq .login`.
- **todo**: no authentication. Verify the TODO file exists; if not, offer to create it with a couple of example items (see the `todo-tasks` skill for the format).

Never write tokens or secrets anywhere — the backend CLIs store their own credentials.

## 3. Write the configuration file

Create `.microfactory/config.yaml` in the project root (settings only, no secrets — safe to commit):

```yaml
backend: jira              # jira | github | todo
project: MYPROJ            # Jira key | owner/repo | path to TODO.md
plan_by_default: false
feature_branches: false
implement_interval: 20m
plan_interval: 10m
# jira backend only:
jira_site: mycompany.atlassian.net
jira_account_id: "712020:abc..."
# github backend only:
github_username: octocat
```

Include only the keys relevant to the chosen backend. If a config file already exists, show it and ask before overwriting.

## 4. Start the loop

For the role the user chose for this session, invoke the loop skill:

- implementer: `/loop <implement_interval> /microfactory:implement-next`
- planner: `/loop <plan_interval> /microfactory:plan-next`

Then tell the user how to run the other role in a second Claude Code session in the same project, e.g.:

```
claude
/loop 10m /microfactory:plan-next
```

If planning is not used (no plan-by-default and no `needs-plan` issues expected), a single implementer session is enough — say so instead of prescribing two sessions.
