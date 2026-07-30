# TODO-11: create a skill to break down features/requirement documents into stories

## Summary

Add `breakdown-feature`: read a feature description or requirements document, slice it into stories
that the factory can actually execute, and — after the user approves the set — create them in the
backend.

It completes the front of the pipeline: `breakdown-feature` (a document becomes stories) →
`refine-story` (a vague story becomes a specified one) → `plan-next` → `implement-next`.

Three decisions shape it:

- **Vertical slices, not layers.** Each story must deliver something a user can observe and must be
  shippable on its own. "Create the database schema", "add the API endpoint", "build the form" is a
  layer decomposition — it produces three stories none of which can be released, and it defeats the
  factory's one-issue-per-iteration model. The first slice should be a walking skeleton: the thinnest
  end-to-end path through the whole feature.
- **Sized to one iteration.** This plugin's unit of work is a single `implement-next` run — claim,
  implement, test, push. A story that cannot land that way is not a story yet, it is an epic, and
  must be split further. This is the concrete sizing rule the skill uses, and it is more useful than
  story points because it is checkable.
- **Gaps become `needs-refinement`, not questions asked here.** Requirement documents are always
  underspecified. Rather than duplicating `refine-story`'s interview, this skill records each story's
  open questions and tags the affected ones so `refine-story` handles them with its proposal-shaped
  questions. One skill, one job.

Approval before creation, exactly as `fill-backlog` does it — a document can easily produce twenty
stories, and on Jira or GitHub those writes are visible to the user's whole team.

No new backend operation is needed: `create` arrived with TODO-9.

## Files to Change

| File | Change |
|---|---|
| `skills/breakdown-feature/SKILL.md` | New skill |
| `CLAUDE.md` | Add to the layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; document the intake pipeline |
| `README.md` | Show the document → stories → plan → implement flow |

## Implementation Steps

1. Write `skills/breakdown-feature/SKILL.md`:
   1. **Take the input** — a file path, a URL, or text the user pastes. Read all of it before slicing;
      partial reads produce stories that contradict later sections.
   2. **Ground it in the codebase** — what already exists that this builds on or duplicates. A
      breakdown written without reading the code proposes work that is already done.
   3. **Find the user-visible outcomes** — the jobs the feature lets someone do. These become the
      slices; each is a story.
   4. **Slice vertically** — walking skeleton first, then one story per outcome, then the enrichments
      (validation, edge cases, admin views, migration of existing data). Split anything that cannot
      land in one `implement-next` iteration.
   5. **Sequence** — dependency order, with each story's prerequisites named, so implementer sessions
      claiming top-down do not hit a wall.
   6. **Write each story** — the `refine-story` template (Goal, Context, Acceptance criteria, Out of
      scope, Open questions), since the two skills feed the same backend and the same readers.
   7. **Record what the document does not say** — per story, and tag those `needs-refinement`.
   8. **Present the set for approval** — the ordered list with sizes and dependencies. Create nothing
      until the user agrees; let them cut or merge.
   9. **Create** through the backend skill: unassigned, a shared label naming the feature so the set
      stays identifiable, `needs-plan` for the ones that warrant a plan, `needs-refinement` where
      questions remain.
   10. **Report** what was created, in order, and what to do next.
2. Update the doc files.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line.
- Consistency: the story template must match `refine-story`'s section names, so a story created here
  and later refined does not change shape. Check them against each other.
- Do **not** run a breakdown as part of the loop iteration — it would create tasks in this project's
  backlog, which is the user's decision.

## Risks / Edge Cases

- **Layer decomposition.** The dominant failure mode; guarded by the vertical-slice rule and the
  walking-skeleton-first ordering.
- **Stories too big.** Guarded by the one-iteration sizing rule.
- **Twenty stories from one document.** Approval gate, plus explicit permission for the user to cut.
- **Document contradicts the codebase** — say so rather than silently designing around it; that is a
  finding for the user, not something to resolve unilaterally.
- **No document, just a sentence** — the skill should still work, but say that the breakdown rests on
  a thin input and lean harder on `needs-refinement`.
- **Overlap with `fill-backlog`** — that sweeps an existing codebase for improvements; this turns a
  stated intention into work. Different inputs, same create path. Worth one line so they do not read
  as duplicates.
