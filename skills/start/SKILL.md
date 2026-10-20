---
name: start
description: Start the microfactory implementation loop in the current session — reads the interval from .microfactory/config.yaml and runs /loop <implement_interval> /microfactory:implement-next. Use when the user wants to start the factory, start implementing, or turn this session into an implementer.
---

# Start the implementation loop

Turn this session into an implementer: claim and implement one issue per iteration, repeatedly, until the user stops it.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop — the loop has no backend to work against.

Take `implement_interval` from the config; if the key is missing, use `20m`.

If `current_sprint_only` is `true` (Jira and Azure DevOps), mention that only issues in the current sprint will be picked up, so the loop may find nothing to do between sprints.

## 2. Update the plugin

Run `claude plugin update microfactory@microfactory --json`. If the result's `updateOutcome` is not `up_to_date` and the update succeeded, tell the user "microfactory updated to <newVersion>: run `/reload-plugins`, then `/microfactory:start` again" and **stop — do not start the loop**, since this session still holds the old skills. If the command fails, skip this step silently and continue.

## 3. Start the loop

Invoke the loop skill with that interval and a dispatch instruction as the target, so each iteration runs in a fresh subagent and this session's context does not grow with every story:

```
/loop <implement_interval> If the implementation subagent you launched earlier is still running, do nothing and end this tick. Otherwise launch one subagent in the background whose whole job is to run /microfactory:implement-next and end with a single line: the issue key and outcome. Print the number of tasks completed in this session, the duration of the session and the rate (task every N minutes).
```

The loop runs the first tick immediately and repeats it on the interval. The session keeps only each subagent's one-line report.

If a subagent cannot be launched, run `/microfactory:implement-next` directly in this session instead.

## 4. Report

Tell the user the interval, that the backlog is the place to watch progress, and — if planning is in use (`plan_by_default: true`, or issues labeled `needs-plan`) — that a planner belongs in a second session in the same project:

```
/microfactory:start-planning
```
