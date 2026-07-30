---
name: start
description: Start the microfactory implementation loop in the current session — reads the interval from .microfactory/config.yaml and runs /loop <implement_interval> /microfactory:implement-next. Use when the user wants to start the factory, start implementing, or turn this session into an implementer.
---

# Start the implementation loop

Turn this session into an implementer: claim and implement one issue per iteration, repeatedly, until the user stops it.

## 1. Read configuration

Read `.microfactory/config.yaml` in the project root. If it does not exist, tell the user to run `/microfactory:init-factory` and stop — the loop has no backend to work against.

Take `implement_interval` from the config; if the key is missing, use `20m`.

## 2. Start the loop

Invoke the loop skill with that interval and `implement-next` as the target:

```
/loop <implement_interval> /microfactory:implement-next
```

The loop runs the first iteration immediately and repeats it on the interval; each iteration ends quietly when no work is available.

## 3. Report

Tell the user the interval, that the backlog is the place to watch progress, and — if planning is in use (`plan_by_default: true`, or issues labeled `needs-plan`) — that a planner belongs in a second session in the same project:

```
/loop <plan_interval> /microfactory:plan-next
```
