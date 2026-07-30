# TODO-10: create a skill to refine stories

## Summary

Add `refine-story`: take a vague backlog item, resolve its ambiguities by asking the user
**closed, proposal-shaped questions**, and write the sharpened story back to the backend so
`plan-next` and `implement-next` have something unambiguous to work from.

The item states three requirements, and they are the whole design:

1. **Post questions to the user to clarify the story** — interactive by nature, unlike every other
   skill in this plugin. `AskUserQuestion` is the mechanism.
2. **Avoid open questions** — no "what should happen when the token expires?". Every question must be
   answerable by picking one of 2–4 concrete options. (The tool always adds an "Other" escape hatch,
   which is where free-form answers belong — as the exception, not the format.)
3. **Ask questions in the form of proposals** — each option is a decision someone could ship, with
   its consequence stated, and the option the skill recommends comes first. The user should be able
   to answer by recognising rather than by composing.

The rule that makes this worth running at all: **never ask the user what the repository already
answers.** Read the code first, decide everything decidable, and spend the user's attention only on
genuine product choices. A refinement pass that asks six questions the codebase answers is worse
than no refinement.

Writing the result back needs a backend operation that does not exist yet — `create` was added in
TODO-9, but nothing can **edit** an existing item. All three backend skills need an update
operation, symmetric with create.

## Files to Change

| File | Change |
|---|---|
| `skills/refine-story/SKILL.md` | New skill |
| `skills/todo-tasks/SKILL.md` | Add "Updating an item" (rewrite the line text, notes beneath) |
| `skills/github-tasks/SKILL.md` | Add "Updating an issue" (`gh issue edit --title/--body`) |
| `skills/jira-tasks/SKILL.md` | Add "Updating a work item" (`acli jira workitem edit`) |
| `CLAUDE.md` | Add to the layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; document the skill and the update op |
| `README.md` | Document the refine step in the workflow section |

## Implementation Steps

1. **Backend update operations**, one section each, in the existing voice:
   - `todo-tasks`: replace the item's line text, preserving its checkbox and inline tags; put
     acceptance criteria in indented lines beneath, since the backend has no description field.
     Warn that the item's key is its line number, so added lines shift the keys below it.
   - `github-tasks`: `gh issue edit <N> --repo owner/repo --title … --body …`.
   - `jira-tasks`: `acli jira workitem edit --key <KEY> --summary … --description …`, with the skill's
     existing "confirm flags with `--help`" caveat.
   Each must say that updating never changes assignee or status — refinement is not a claim.
2. **`skills/refine-story/SKILL.md`**:
   1. **Pick the story** — a key argument, or list the open unrefined items (`needs-refinement` tag
      where the project uses one, otherwise the thin ones) and ask which.
   2. **Answer what you can yourself** — read the story, then the code, existing conventions, similar
      features already implemented, and the data model. Write down what you concluded and from where;
      these become statements in the refined story, not questions.
   3. **List the remaining ambiguities** — scope edges, behaviour on the unhappy path, validation and
      data rules, permissions, what the user sees, non-functional expectations, and explicitly what
      is *out* of scope. Discard any whose answer would not change the implementation.
   4. **Ask as proposals** — `AskUserQuestion`, at most 4 per round, each with 2–4 shippable options,
      recommended first and labelled as such, each with its consequence. Say what happens by default
      if they pick nothing. Stop asking once the remaining uncertainty is cheaper to discover in
      implementation than to resolve now.
   5. **Rewrite the story** — a template: one-sentence goal, context/why, acceptance criteria as a
      checkable list, out of scope, decisions taken (with what was chosen and why), and any
      deliberately deferred question.
   6. **Write it back** through the backend's update operation; drop the `needs-refinement` tag if
      present; leave status and assignee alone.
   7. **Stop** — do not plan and do not implement; say that `plan-next` or `implement-next` takes it
      from here.
3. Update the doc files.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line.
- Todo-backend update is exercisable safely on a scratch copy: rewrite an item's text with criteria
  lines beneath and confirm the item still parses as claimable under `todo-tasks`'s eligibility rules
  (the checkbox and tags must survive the rewrite).
- Do **not** run the interactive flow as part of the loop iteration — it would interrupt the user with
  questions about a story nobody asked to refine.

## Risks / Edge Cases

- **Asking what the code answers.** The main failure mode; guarded by step 2 coming before step 3.
- **Interrogation.** Twelve questions is a worse outcome than a slightly vague story. Hence the
  per-round cap and the "stop when uncertainty is cheap" rule.
- **Fake choices.** Options that are not really different, or a recommended option with no stated
  reason, waste the user's attention as much as an open question does.
- **Destroying the original text.** The rewrite must preserve any concrete detail already in the
  story; refinement adds precision, it does not replace the author's intent. Where the original is
  contradicted by an answer, the decision list records that.
- **Todo backend line-shifting** — adding criteria lines moves the keys of items below, the known
  wart in this backend. Called out in the backend skill rather than solved here.
- **Interactive skill inside a `/loop`** — worth a line in the skill: this one is meant to be run by
  hand, not as a loop target.
