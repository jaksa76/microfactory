---
name: todo-tasks
description: Local TODO.md backend operations for the microfactory — find eligible items, claim, create, update, comment, and transition by editing checkbox states in a markdown file. Used by plan-next and implement-next when the configured backend is todo. Good for trying out the factory without Jira or GitHub.
---

# TODO.md task operations

Tasks live in a local markdown file specified as `project` path in `.microfactory/config.yaml` (default `TODO.md`). No authentication, no CLI — operations are file edits. The issue key is `TODO-<n>`, a stable id written at the start of the item's text (before any `[tag]` markers). Items from before this convention have no id; for those, fall back to the item's line number as its key, and give it a real id the next time it's touched (see Updating).

## File format

One item per line, a checkbox list. The checkbox character encodes the status; the item text starts with its `TODO-<n>` id, then any inline `[tag]` markers, then a `- ` separator and the description:

```markdown
- [ ] TODO-1 - Add a health check endpoint
- [ ] TODO-2 [needs-plan] - Rework the auth flow
- [>] TODO-3 - Fix the flaky integration test
- [x] TODO-4 - Update the README
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
2. Take the first one; its `TODO-<n>` id is the key (or, for a legacy item with no id, its line number — add the id now, per Updating below).
3. Flip its checkbox: to `[~]` (planning mode) or `[>]` (implementation mode).
4. The item text (minus checkbox, id, and tags) is the issue summary; there is no separate description.

This backend is single-user, so no race verification is needed.

## Creating an item

Find the highest existing `TODO-<n>` id in the file (checking every item, not just eligible ones) and append a new line with the next one: `- [ ] TODO-<n+1> - <title>`, with any labels as inline `[tag]` markers right after the id (e.g. `- [ ] TODO-5 [needs-plan] - Rework the auth flow`). That id is the new item's key, permanently — it won't change as other lines are added or removed.

This backend has nowhere to put a description, so the title has to be self-contained; add a single indented note line beneath it for the rationale only if necessary — the backlog is a progress indicator, and a title that needs explaining is usually a title that needs rewriting. Never assign or transition a newly created item — creation is not part of the claim protocol, and an item that is not `[ ]` cannot be claimed.

## Updating an item

Rewrite the item's line text in place, keeping its checkbox character, `TODO-<n>` id, and inline `[tag]` markers (id first, then tags, then the `- ` description). If the item has no id yet (a legacy item, keyed by line number), add one now — figure out the next free id the same way Creating does, and prepend `TODO-<n>` to its text, ahead of any existing tags. Longer content (context, decisions) goes in indented lines beneath the item, since there is no description field.

Updating never changes an item's checkbox or claims it — status and refinement are separate concerns. Once an item has a `TODO-<n>` id its key is stable, unaffected by lines added or removed elsewhere in the file. A legacy item without one is still keyed by line number until it gets an id, so re-read the file to get its current key after an update.

## Backfilling ids into a legacy file

If the file predates the id convention (no item has a `TODO-<n>` prefix) — or the user asks to add ids to an existing file — do a single pass over the whole file rather than relying on Updating to backfill items one at a time:

1. Read every item in the file, and every plan file under `plans/` (named `TODO-<n>.md` or `TODO-<n>-<slug>.md` — see `plan-next`).
2. Match each item to a plan file by content (its title/summary against the item's text), not by assuming the item still sits at the line number its plan was written for — items move as the file is edited over time, so a line-number match can be wrong.
3. Assign each matched item the id from its plan file's name. If two plan files claim the same id (a historical collision, from a time when two different items briefly shared that line number), keep the id on whichever item's content matches the plainly-named `TODO-<n>.md`; for the other item, pick a fresh id (per step 4) and rename its plan file to match — `git mv` it to `TODO-<newn>.md` and update its `# TODO-<newn>: ...` heading.
4. Give every item that matched no plan file a fresh id, continuing upward from the highest id used anywhere (in the file or in `plans/`), assigned in file order so ids stay ascending top to bottom.
5. Prepend each item's id (and, for a renamed plan file, its new id) per the File format above, leaving checkbox, tags, and wording otherwise unchanged.

This is a bulk, one-time correction — normal operation only ever adds one id at a time, per Creating and Updating above.

## Other operations

- **View**: find the line whose text starts with `TODO-<n>` (or, for a legacy item, line `<n>` itself).
- **Comment**: insert an indented note line **below the item and below any notes already there**, so notes stay in the order they were written: `  - Note: <text>`. A multi-line comment becomes one indented `- Note:` line per line of text, keeping the same indentation.
- **List comments**: the indented `- Note:` lines beneath the item, in file order. This file has no authors or timestamps, so a reader tells notes apart by their content alone — which is why `refine-story` marks its own with a fixed first line.
- **Transition**: set the checkbox character per the status table above (In Progress and In Review both map to `[>]`; Done is `[x]`). A human reviewer approves a plan by changing `[?]` to `[p]`.

Commit changes to the TODO file together with the related work so the backlog history stays in git.
