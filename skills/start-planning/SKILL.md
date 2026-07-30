---
name: start-planning
description: Start the microfactory planning loop in the current session — reads the interval from .microfactory/config.yaml and runs /loop <plan_interval> /microfactory:plan-next. Use when the user wants to start planning, start the planner, or turn this session into a planner.
---

# Start the planning loop

Turn this session into a planner: claim one planning-eligible issue per iteration, write its plan, and put it up for review. This session never implements anything — that is what `/microfactory:start` is for.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop — the loop has no backend to work against.

Take `plan_interval` from the config; if the key is missing, use `10m`.

If `plan_by_default` is `false`, mention that only issues labeled `needs-plan` will be picked up, so the loop may find nothing to do until such an issue exists.

## 2. Start the loop

Invoke the loop skill with that interval and `plan-next` as the target:

```
/loop <plan_interval> /microfactory:plan-next
```

The loop runs the first iteration immediately and repeats it on the interval; each iteration ends quietly when no work is available.

## 3. Report

Tell the user the interval, that finished plans land in `plans/<KEY>.md` and wait for their approval (the backlog shows which issues are awaiting plan review), and that an implementer belongs in a second session in the same project:

```
/microfactory:start
```
