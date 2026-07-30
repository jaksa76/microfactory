# TODO-9: create a skill that uses the above skills to create a list of tasks for the project

## Summary

Add `fill-backlog`: run the four finders, merge and rank their findings, get the user's approval,
then create the approved ones as tasks in the configured backend.

This is the skill the finder family was built for, and it is the first one that **writes** to the
backend. None of the three backend skills documents a create operation today — they cover find,
claim, view, comment, transition — so each needs one added. That is the bulk of this change.

Two decisions that shape it:

- **Approval before creation.** The skill presents the merged list and creates only what the user
  picks. Silently opening 30 issues from an automated sweep would poison a real backlog, and it is
  outward-facing in the Jira/GitHub cases. Nothing is written before the user chooses.
- **Findings never touch disk.** The aggregator loads the finders in its own session and holds their
  findings in context, then writes the backend. No `findings.md` handoff files — the backend stays
  the single source of truth, per CLAUDE.md.

Effort maps to routing: `L` findings (and anything the finder flagged as risky) get the
`needs-plan` label/tag so they go through planning before implementation; `S`/`M` findings go
straight to the implementable pool.

While in `jira-tasks`: its documented comment command (`acli jira comment add --issue …
--comment …`) contradicts the bundled `acli-reference.md` (`acli jira workitem comment create --key
… --body …`). The reference is transcribed CLI help, so the skill is wrong. Fix it in passing — it
is two lines and it would break every Jira comment the factory tries to post.

## Files to Change

| File | Change |
|---|---|
| `skills/fill-backlog/SKILL.md` | New skill: sweep → merge → approve → create → report |
| `skills/todo-tasks/SKILL.md` | Add a "Creating an item" section (append a `- [ ]` line) |
| `skills/github-tasks/SKILL.md` | Add a "Creating an issue" section (`gh issue create`) |
| `skills/jira-tasks/SKILL.md` | Add a "Creating a work item" section; fix the comment command |
| `CLAUDE.md` | Add to the layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; extend the finder section with the aggregator; note create in the backend-operations line |
| `README.md` | Document the sweep-to-backlog flow next to the finder sentence |

## Implementation Steps

1. **Backend create operations.** Add one section to each backend skill, in the same voice as the
   existing operations, taking a title, a body, and optional labels:
   - `todo-tasks`: append `- [ ] <title>` to the end of the TODO file, with `[tag]` markers inline;
     the key is the new line number. Note that the body has nowhere to live in this backend — put a
     one-line rationale in an indented note line under the item, and keep the title self-contained.
   - `github-tasks`: `gh issue create --repo owner/repo --title … --body … --label …`; create missing
     labels with `gh label create` and retry, matching how transitions already handle that.
   - `jira-tasks`: `acli jira workitem create --project <KEY> --type Task --summary … --description …`,
     with the skill's existing instruction to confirm exact flags via `--help` (the bundled reference
     lists the subcommand but not its flags).
   Each must state that creation is **not** part of the claim protocol and must not assign the new
   item to anyone — new tasks have to stay unassigned to be claimable.
2. **The aggregator**, `skills/fill-backlog/SKILL.md`:
   1. Read config, load the backend skill.
   2. Choose areas — all four by default; accept an argument (`design`, `testing`, `libraries`, `ui`)
      or ask when the user's intent is narrower.
   3. Run each finder by loading its skill and following it. Keep each area's findings separate at
      first so a failed area does not lose the others.
   4. **Merge**: drop duplicates across areas (the same file showing up in design and testing is one
      task), collapse findings that would be one commit, and re-rank the survivors globally by
      payoff over effort — each finder ranked only within its own area.
   5. **Check the backlog first**: read the existing open items and drop findings that are already
      there. Re-proposing existing work is the fastest way to make the skill useless.
   6. **Present and get approval** — the ranked list with area, effort, and payoff, and let the user
      select. Create nothing until they have.
   7. **Create** the selected findings through the backend skill: title from the finding's title,
      body from Problem / Proposed change / Payoff / Risk, an area label, and `needs-plan` for `L` or
      risky ones. Leave them unassigned.
   8. **Report** the created keys/URLs, and what was skipped as duplicate or already-present.
3. Update the doc files.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line, across all skills.
- Todo-backend create is exercisable locally and safely: append a probe item to a scratch copy of
  `TODO.md` (not the real one) and confirm the documented shape matches what `todo-tasks`'s
  eligibility rules would then pick up as claimable.
- Do **not** exercise the GitHub or Jira create paths — creating issues is outward-facing and this
  repo's backend is `todo`.
- Do **not** run the full sweep as part of the iteration: it would create tasks in this project's
  own backlog, which is the user's decision, not the loop's.

## Risks / Edge Cases

- **Backlog flooding.** The core risk. Mitigated by approval-before-creation, the duplicate check
  against existing items, and the finders' own ≤10 cap (≤40 total worst case, presented, not created).
- **Duplicate tasks across areas** — handled in the merge step.
- **A finder finding nothing / failing** — the sweep must continue with the remaining areas and say
  which area produced nothing.
- **Todo backend has no body field** — the finding's detail cannot be preserved verbatim; the title
  must therefore carry enough on its own, with a one-line note beneath.
- **Line-number keys shift** when the todo backend gains items, invalidating existing `plans/<KEY>.md`
  associations. Pre-existing (already hit and reported in this repo); creation makes it easier to
  trigger. Out of scope here — it needs its own item, since fixing it changes the key contract in all
  three backends.
