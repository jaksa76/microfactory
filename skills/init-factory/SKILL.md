---
name: init-factory
description: Set up the microfactory for the current project. Interviews the user, writes .microfactory/config.yaml, sets up the task backend (Jira via acli, GitHub via gh, Azure DevOps via az, or a local TODO.md), installing missing CLIs with approval, and starts the work loop. Use when the user wants to initialize, configure, or reconfigure the factory.
---

# Initialize the Microfactory

Set up the current project so `plan-next` and `implement-next` can run continuously. All commands below are examples — adapt them to the user's OS and shell (Windows/macOS/Linux).

## 1. Interview the user

Ask (with AskUserQuestion where options fit, free text otherwise):

1. **Task backend**: `jira`, `github`, `azure-devops`, or `todo` (a local TODO.md file).
2. **Project identifier**: Jira project key (e.g. `MYPROJ`), GitHub `owner/repo`, Azure DevOps project name, or the path to the TODO file.
3. **Plan by default?** If yes, every issue needs an approved plan before implementation (issues labeled `skip-plan` are exempt). If no, only issues labeled `needs-plan` go through planning.
4. **Auto-plan?** When an implementation-eligible issue has no plan, should the implementer write one for itself and treat it as approved? Default no — planning then happens only in the planner loop, and the implementer works from the issue and its refinement thread. Independent of question 3: `needs-plan` issues still wait for an approved plan either way.
5. **Feature branches?** If yes, implementation happens on `feature/<KEY>` branches with a pull request (issues labeled `skip-branch` are exempt). If no, work lands on the default branch (issues labeled `needs-branch` still get a branch).
6. **Deep review?** If yes (the default), each implementation iteration hands its diff to a fresh agent for a cold review before committing — an author reads its own diff as intended rather than as written. If no, the implementer reviews its own diff instead; issues labeled `needs-review` still get the cold review, and `skip-review` exempts an issue either way.
7. **Loop intervals**: how often to look for work. Defaults: implementation every 20 minutes, planning every 10 minutes.
8. **This session's role**: planner or implementer (the other role runs in a second session).

## 2. Set up the backend

Run the `tasks` skill's `setup` subcommand for the chosen backend. Its backend doc says what the tool needs: it detects the CLI, extension and login, **offers to install whatever is missing and waits for the user's approval before each install**, helps with the login (suggesting `! <login command>` where the user must do it interactively), and gives you the values the config file needs (`jira_site` and `jira_account_id`, `github_username`, `azure_devops_org` and `azure_devops_user`). For the todo backend it verifies the TODO file exists and offers to create it.

If the user declines an install, stop and tell them what to run themselves; do not continue to write a config for a backend that cannot work.

Never write tokens or secrets anywhere — the backend CLIs store their own credentials.

## 3. Write the configuration file

Create `.microfactory/config.yaml` in the project root (settings only, no secrets — safe to commit):

```yaml
backend: jira              # jira | github | azure-devops | todo
project: MYPROJ            # Jira key | owner/repo | Azure DevOps project | path to TODO.md
plan_by_default: false
auto_plan: false
feature_branches: false
deep_review: true
implement_interval: 20m
plan_interval: 10m
# jira backend only:
jira_site: mycompany.atlassian.net
jira_account_id: "712020:abc..."
# github backend only:
github_username: octocat
# azure-devops backend only:
azure_devops_org: https://dev.azure.com/myorg
azure_devops_user: me@example.com
```

Include only the keys relevant to the chosen backend. If a config file already exists, show it and ask before overwriting.

## 4. Start the loop

For the role the user chose for this session, invoke the loop skill:

- implementer: `/microfactory:start`
- planner: `/microfactory:start-planning`

Then tell the user how to run the other role in a second Claude Code session in the same project, e.g.:

```
claude
/microfactory:start-planning
```

If planning is not used (no plan-by-default and no `needs-plan` issues expected), a single implementer session is enough — say so instead of prescribing two sessions.
