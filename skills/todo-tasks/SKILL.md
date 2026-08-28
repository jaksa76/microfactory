---
name: todo-tasks
description: Local TODO.md backend operations for the microfactory — find eligible items, claim, create, update, comment, and transition by editing checkbox states in a markdown file. Used by plan-next and implement-next when the configured backend is todo. Good for trying out the factory without Jira or GitHub.
---

# TODO.md task operations

Tasks live in a local markdown file (the `project` path in `.microfactory/config.yaml`). No authentication, no CLI — operations are file edits. The issue key is `TODO-<line-number>` (1-based line number of the item in the file).

## File format

One item per line, a checkbox list. The checkbox character encodes the status; labels are inline `[tag]` markers in the text:

```markdown
- [ ] Add a health check endpoint
- [ ] Rework the auth flow [needs-plan]
- [>] Fix the flaky integration test
- [x] Update the README
  - Note: Implemented: Update the README
```

| Checkbox | Status |
|---|---|
| `[ ]` | Open (claimable) |
| `[>]` | In Progress / In Review |
| `[~]` | Planning |
| `[?]` | Awaiting Plan Review |
| `[p]` | Plan Approved (claimable for implementation) |
| `[x]` | Done |

## Finding eligible items

By mode and the `plan_by_default` setting:

| Mode | plan_by_default | Eligible lines |
|---|---|---|
| planning | true | `[ ]` without a `[skip-plan]` tag |
| planning | false | `[ ]` with a `[needs-plan]` tag |
| implementation | true | `[p]`, or `[ ]` with a `[skip-plan]` or `[plan-approved]` tag |
| implementation | false | `[ ]` without a `[needs-plan]` tag |

## Claiming an item

1. Find eligible lines (top to bottom). If none, there is no work — stop.
2. Take the first one; its line number gives the key `TODO-<n>`.
3. Flip its checkbox: to `[~]` (planning mode) or `[>]` (implementation mode).
4. The item text (minus checkbox and tags) is the issue summary; there is no separate description.

This backend is single-user, so no race verification is needed.

## Creating an item

Append a new line to the end of the file: `- [ ] <title>`, with any labels as inline `[tag]` markers (e.g. `- [ ] Rework the auth flow [needs-plan]`). Its line number is the new item's key.

This backend has nowhere to put a description, so the title has to be self-contained; add a single indented note line beneath it for the rationale only if necessary — the backlog is a progress indicator, and a title that needs explaining is usually a title that needs rewriting. Never assign or transition a newly created item — creation is not part of the claim protocol, and an item that is not `[ ]` cannot be claimed.

## Updating an item

Rewrite the item's line text in place, keeping its checkbox character and inline `[tag]` markers. Longer content (context, decisions) goes in indented lines beneath the item, since there is no description field.

Updating never changes an item's checkbox or claims it — status and refinement are separate concerns. Remember that keys are line numbers: **every line you add beneath an item shifts the keys of all items below it**, so re-read the file to get current keys after an update.

## Other operations

- **View**: read line `<n>` of the file.
- **Comment**: insert an indented note line **below the item and below any notes already there**, so notes stay in the order they were written: `  - Note: <text>`. A multi-line comment becomes one indented `- Note:` line per line of text, keeping the same indentation.
- **List comments**: the indented `- Note:` lines beneath the item, in file order. This file has no authors or timestamps, so a reader tells notes apart by their content alone — which is why `refine-story` marks its own with a fixed first line.
- **Transition**: set the checkbox character per the status table above (In Progress and In Review both map to `[>]`; Done is `[x]`). A human reviewer approves a plan by changing `[?]` to `[p]`.

Commit changes to the TODO file together with the related work so the backlog history stays in git.
