# Microfactory

An AI coding factory that fits in a Claude Code session. A plugin that turns a task backlog into a continuous stream of planned and implemented changes: agents pull issues from a task manager, implement them, and push code — continuously. Inspired by [AI Coding Factories](https://jaksa.wordpress.com/2025/08/07/ai-coding-factories/).

## How it works

The factory is a set of **skills** — markdown instructions, no scripts and no containers. A Claude Code session in your project runs one skill per work iteration:

- `/microfactory:implement-next` — claim the next eligible issue, implement it, run the tests, push (directly or via a feature branch + PR), and update the issue.
- `/microfactory:plan-next` — claim the next issue that needs a plan, write `plans/<KEY>.md`, push it, and put the issue up for human review.

Continuous operation comes from Claude Code's `/loop`, which re-runs a skill on an interval. Parallelism comes from running several sessions — the claim protocol (assign, wait, verify) makes concurrent workers safe because the task manager is the single source of truth.

```
[Task backend: Jira | GitHub Issues | TODO.md]
   ↑↓ claim, comment, transition
[Claude Code sessions running /loop]
   ↓ push code and plans
[Git]
```

## Installation

Install the plugin in Claude Code:

```
/plugin marketplace add jaksa76/microfactory
/plugin install microfactory
```

Backend prerequisites: `acli` for Jira, `gh` for GitHub Issues, nothing for the TODO.md backend.

## Setup

Open Claude Code in the project you want the factory to work on and run:

```
/microfactory:init-factory
```

It interviews you (backend, project, planning and branching policy, loop intervals), authenticates the backend CLI, writes `.microfactory/config.yaml` (no secrets — safe to commit), and starts the work loop in the current session.

## Running

A typical setup uses two sessions in the same project:

```
# session 1 — implementer
/loop 20m /microfactory:implement-next

# session 2 — planner (only needed if you use planning)
/loop 10m /microfactory:plan-next
```

Add more implementer sessions to scale out. You can also run a single iteration by invoking a skill directly, optionally for a specific issue:

```
/microfactory:implement-next MYPROJ-42
```

## Workflow

Issue state drives everything; the board is your monitor.

1. An issue starts in **To Do** (or as an open unassigned GitHub issue / `[ ]` TODO item).
2. If it needs planning (`needs-plan` label, or `plan_by_default: true` in config), a planner session claims it, writes `plans/<KEY>.md`, pushes it, and transitions the issue to **Awaiting Plan Review**.
3. A human reviews the plan and transitions the issue to **Plan Approved**.
4. An implementer session claims it, implements (following the plan when present), runs the tests, pushes, and transitions the issue to **Done** — or opens a PR and transitions to **In Review** when feature branches are enabled.

Labels tune behavior per issue: `needs-plan` / `skip-plan` for planning, `needs-branch` / `skip-branch` for feature branches (`skip-*` wins).

On GitHub, statuses map to labels (`in-progress`, `in-planning`, `awaiting-plan-review`, `plan-approved`, `in-review`); on the TODO.md backend they map to checkbox characters. See the skill files under `skills/` for the exact conventions.

## Legacy

The previous implementations are kept for reference: `legacy/bash-factory/` (bash scripts + Docker worker containers, with its full test suite) and `legacy/hub/` (the original hub-based design).
