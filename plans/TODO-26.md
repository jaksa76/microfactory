# TODO-26: make the auto-plan step of implement-next optional

## Summary

Step 5 writes `plans/<KEY>.md` and self-approves it whenever no plan exists. That is right for a team
that wants a written plan behind every change, and wrong for one that wants planning to happen only in
the planner loop, under human approval — there, a self-approved plan is ceremony: written and blessed
by the same agent in the same breath, and read by nobody.

Add `auto_plan`. With it on, step 5 behaves as today. With it off, the implementer works directly from
the issue and its refinement thread and writes no plan file.

**The default is off, per the issue.** This is the significant part, and it differs from `deep_review`
in a way worth stating plainly rather than burying: `deep_review`'s absent key means *true*, which
preserves the behaviour of every config file written before it existed. `auto_plan`'s absent key means
*false*, so this change alters what an existing installation does — issues that used to get a
self-approved plan will simply be implemented. That is the intent (the issue was edited from "default
on" to "default off" deliberately), but it is a behaviour change on upgrade and the docs should say so
rather than let users discover it.

**No new per-issue labels.** `needs-plan` already is the per-issue override: it makes an issue
ineligible for implementation until a real plan is approved, and that rule is untouched here. A
`needs-auto-plan` label would be a second way to say the same thing.

**What replaces the plan when it is off.** Nothing formal, and that is the point — the issue text plus
the resolved refinement thread is the specification, which is what step 3 already loads. Step 5's
value in that mode is not a document; it is the thinking, which still has to happen before code is
written. The skill should say that, or "no plan" will read as "no analysis".

## Files to Change

| File | Change |
|---|---|
| `skills/implement-next/SKILL.md` | Step 5 branches on `auto_plan`; state what the off path uses instead |
| `skills/init-factory/SKILL.md` | Interview question; `auto_plan: false` in the config example |
| `docs/ARCHITECTURE.md` | Config block; correct the "every implemented issue ends up with a plan file" claim, which stops being true |
| `README.md` | The Workflow step 4 sentence promises a self-written plan; qualify it |

## Implementation Steps

1. **`skills/implement-next/SKILL.md` step 5** — restructure into three cases, in this order so the
   strictest is read first:
   1. `plans/<KEY>.md` exists → it is the approved plan, use it as written. (Unchanged, and stays
      first: an existing plan is followed whatever `auto_plan` says.)
   2. `auto_plan: true` → write one now via `plan-next`'s steps 4–5, treat it as approved, carry on.
      (Today's behaviour.)
   3. Otherwise → no plan file. Do `plan-next`'s **analysis** (its step 4) anyway and carry the
      conclusions into the implementation; the issue and the resolved thread are the specification.
2. **State the absent-key rule** in the same step: a missing `auto_plan` means **false**, and note that
   this is the opposite of `deep_review` so a reader comparing the two settings is not surprised.
   Two adjacent booleans with opposite absent-key semantics is a genuine trap; naming it costs a
   clause and saves a misreading.
3. **Keep `needs-plan` untouched.** Add one sentence that eligibility is unaffected: a `needs-plan`
   issue is not implementable without an approved plan regardless of `auto_plan`, so turning auto-plan
   off never causes an issue to skip planning that was meant to have it.
4. **`skills/init-factory/SKILL.md`** — interview question after the plan-by-default one (they are the
   same subject), and `auto_plan: false` in the config example.
5. **`docs/ARCHITECTURE.md`** — add the key to the config block; rewrite the "Every implemented issue
   therefore ends up with a plan file" paragraph, which is exactly the sentence this change falsifies.
   Note the upgrade consequence.
6. **`README.md`** — Workflow item 4 says the implementer writes "its own plan first (and treating it
   as approved)"; qualify it as the `auto_plan` path rather than the only path.

## Testing

No automated suite for the skills (CLAUDE.md). Verification:

- Structural: frontmatter untouched, `name` still matches the directory.
- Consistency: `auto_plan` spelled identically in all four files; the config examples in `init-factory`
  and `ARCHITECTURE.md` stay key-for-key identical, as they already do.
- The three cases in step 5 must be mutually exclusive and ordered, with the existing-plan case first.
  Read them as an implementer with a `needs-plan` issue and no plan, and confirm the answer is "not
  implementable" and not "implement without a plan".
- No behavioural run: this iteration's own config has no `auto_plan` key, which is itself the
  absent-key case, and the iteration is being implemented under the old behaviour by definition.

## Risks / Edge Cases

- **Behaviour change on upgrade.** The main risk, accepted deliberately. Mitigated by documenting it
  in `ARCHITECTURE.md` rather than only in the skill.
- **"No plan" read as "no analysis".** The likeliest way this gets implemented badly downstream —
  guarded by step 1.3 requiring `plan-next`'s analysis regardless.
- **Opposite absent-key semantics to `deep_review`.** Guarded by naming it explicitly; the alternative,
  making them agree, would contradict the issue.
- **`plans/` stops being a complete record.** Under the new default, the directory holds only
  human-approved plans. Arguably an improvement in signal, but anything that assumed one file per
  implemented issue is now wrong; the ARCHITECTURE rewrite is where that gets said.
- **Interaction with TODO-28** (capture lessons in `CLAUDE.md`). With no plan file, the iteration note
  and any captured lesson become the only written trace of an iteration's reasoning. Worth a sentence
  when TODO-28 is planned, not a change here.
