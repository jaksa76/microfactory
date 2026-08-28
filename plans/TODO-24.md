# TODO-24: run each implement-next iteration in a separate context

## Summary

`/loop 20m /microfactory:implement-next` runs every iteration in one session, so iteration N carries
the full context of iterations 1..N-1. That is pure cost — the factory is already designed so that
nothing needs to carry: the backend is the single source of truth, and all cross-session coordination
goes through issue state. The execution shape simply does not match the design yet.

Three approaches were considered.

**A. Delegate the iteration body to a fresh agent; the loop session becomes a thin dispatcher.**
Each tick spawns an agent that does claim → implement → verify → review → push → update, and returns
a short report. The plugin already does exactly this one level down — step 8 hands the diff to a
fresh reviewing agent — so the idiom is established.

**B. A fresh CLI session per iteration**, driven by an OS scheduler running
`claude -p "/microfactory:implement-next"` headless. Genuinely zero accumulation. But it abandons the
`/loop` model the whole plugin is built on, needs per-OS scheduling instructions, non-interactive auth
and headless tool permissions, and takes the live session view away from the user.

**C. Clearing or compacting context between iterations.** Rejected: not controllable from inside a
skill, and `/clear` inside a loop session would take the loop with it.

**Choose A.** Beyond fitting the existing shape, it is the only option that leaves the factory a place
to stand for the two feedback items already in the backlog. TODO-27 (a skill that reviews all the work
done in a session) and TODO-29 (run it when out of work) both need *something* that has seen more than
one iteration. Under A that is the dispatcher, holding one compact report per iteration; under B
nothing accumulates anywhere and those two items lose their raw material entirely. Isolating the
iterations and keeping a thin trace of them are the same design decision, so this plan makes the
report a deliberate artifact rather than a side effect.

**Where it belongs: `start`, not `implement-next`.** `implement-next` describes *what one iteration
does*; `start` decides *how iterations are driven*. One skill, one job. `start` builds the loop, so
`start` is where the loop learns to dispatch rather than to execute.

**No new config key.** The last three landed items each added a setting, and the reflex is worth
resisting here: unlike `deep_review`, this has no cost a user would trade against — a fresh context is
strictly better when available, and the only reason not to use it is that the environment cannot, which
is a fallback, not a preference. Adding `iteration_isolation: true` would be a switch nobody has a
reason to flip.

## Files to Change

| File | Change |
|---|---|
| `skills/start/SKILL.md` | Step 2 dispatches each iteration to a fresh agent; add the report contract and the fallback |
| `skills/implement-next/SKILL.md` | State that an iteration assumes a fresh context and owns its own claim/push; name what its closing report must contain |
| `docs/ARCHITECTURE.md` | New subsection under continuous operation: dispatcher/worker shape, why, and the nesting limit |
| `README.md` | The `/loop` examples currently teach the accumulating form — correct them |
| `CLAUDE.md` | One line in Conventions: iterations are context-isolated; the backend stays the only durable state |

## Implementation Steps

1. **`skills/start/SKILL.md` step 2** — the loop target becomes a dispatch instruction rather than a
   direct skill invocation: each tick launches a fresh agent whose whole job is one `implement-next`
   iteration in the project directory, awaits it, and keeps only its report. Specify what the
   dispatcher retains per iteration — issue key, outcome (done / blocked / no work), and the iteration
   note if one was posted — and that it retains nothing else, since retaining the diff would re-create
   the accumulation this removes.
2. **Fallback, in the same step** — where an agent cannot be spawned, run the iteration inline exactly
   as today and say so once, rather than failing. Same distinction the review step now draws: a
   capability the environment lacks is not a configuration choice.
3. **Name the nesting limit explicitly.** The dispatched iteration agent must itself spawn the cold
   review agent (step 8), which is an agent spawning an agent. Where a harness forbids that, the
   iteration agent falls back to self-review *and says which constraint caused it* — the same report
   line TODO-12 just added. Do **not** resolve this by turning `deep_review` off: trading the review
   away to buy context isolation is a real loss disguised as a setting, and if a harness forces the
   choice the user should be told, not quietly charged.
4. **`skills/implement-next/SKILL.md`** — one short paragraph near the top: this skill is one
   iteration, it assumes a cold context, and it must not rely on anything a previous iteration knew.
   Then extend step 10 so the closing report carries the three fields the dispatcher keeps. Step 10
   currently governs the *issue comment*; the report to the caller is a second, smaller thing and the
   difference needs saying, or implementers will conflate them.
5. **`docs/ARCHITECTURE.md`** — subsection under *Continuous operation and scaling*: the dispatcher
   holds no work state, one iteration is one agent, and this is the same statelessness the claim
   protocol already assumes, now enforced by the execution shape rather than by convention. Note the
   dependency: TODO-27's feedback skill reads the dispatcher's reports.
6. **`README.md`** — the two `/loop` examples under Workflow and the one in Running teach the
   accumulating form. Update them, or the documented path contradicts the skill.
7. **`CLAUDE.md`** — one line under Conventions.

## Testing

No automated suite for the skills (CLAUDE.md). Verification:

- **Behavioural, and worth doing here** — this change alters how the loop executes, which prose review
  cannot confirm. In a scratch project with a `TODO.md` and two trivial `[ ]` items, run `start` and let
  two iterations pass. Both items should be completed, each by its own agent, with the dispatcher
  holding two short reports and no diff content. The second iteration proving it never saw the first is
  the actual acceptance criterion.
- **Fallback path** — force the no-agent case and confirm the loop still completes an iteration inline
  and says once that it did.
- Structural: frontmatter untouched, `name` still matches each directory.
- Consistency: the report fields named in `start` and in `implement-next` step 10 must be the same
  three, in the same words.

## Risks / Edge Cases

- **Nested agents may not be supported.** The most likely failure, and it lands squarely on the review
  step this backlog just made configurable. Handled by step 3 — report the constraint, never silently
  drop the review.
- **The dispatcher accumulates reports.** Smaller than diffs, but not nothing over fifty iterations.
  Bounding the report to three fields is the mitigation; if it still grows, the fix belongs to TODO-27
  (which consumes them), not here.
- **Interaction with TODO-18** (capture lessons in `CLAUDE.md`). That item exists *because* iteration
  context is lost, and this change makes the loss structural rather than incidental. It strengthens the
  case for TODO-18 rather than conflicting with it — worth one sentence in the ARCHITECTURE subsection
  so the two are read together.
- **A dispatched agent inheriting a dirty working tree** from a failed predecessor. `implement-next`
  step 2 pulls but does not check for uncommitted changes, unlike `plan-next` step 2 which stops. Out
  of scope to fix here, but the isolation makes it likelier — file it rather than widening this issue.
- **Loss of the live view.** The user currently watches an iteration work; afterwards they watch a
  dispatcher report on it. The backlog is still the monitor, per the plugin's own line, but this is a
  real change in feel and belongs in the README wording.
- **Scope creep into TODO-14** (foreground vs. background review agents). This plan says the dispatcher
  *awaits* its iteration agent, which is the same underlying concern one level up; keep the decision in
  TODO-14 and reference it rather than restating it.
