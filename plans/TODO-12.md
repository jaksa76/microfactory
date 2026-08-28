# TODO-12: make the cold review in implement-next optional

## Summary

`implement-next` step 8 currently delegates every iteration's diff to a fresh reviewing agent, with
no way to turn it off. That review is the right default — an author reads its own diff as intended
rather than as written — but it doubles the cost of a one-line change, and it is dead weight in
environments that cannot run a second agent at all.

Make it a setting, following the shape the plugin already uses twice (`plan_by_default`,
`feature_branches`): a boolean in `.microfactory/config.yaml` that sets the project's default, plus
per-issue label overrides in both directions.

- **`deep_review: true`** (default when the key is absent) — review every iteration, as today.
  A missing key must keep today's behaviour, because existing config files in the wild do not have it.
- **`skip-review` label** — no cold review on this issue.
- **`needs-review` label** — force the cold review even when `deep_review: false`.

Precedence follows `feature_branches`' existing rule, where the *narrowing* label wins: `skip-review`
beats `needs-review` beats the config key. That is arbitrary in isolation, but it matches
`skip-branch`/`needs-branch` and `skip-plan`/`needs-plan`, and a reader who learns the rule once
should not have to re-learn it per setting.

Skipping the cold review never means skipping review. The implementer still reads its own diff
against the same checklist — the setting chooses *who* reviews, not *whether*. Stating that in the
skill matters more than the flag itself: a flag that reads as "reviewing is optional" invites
turning it off to go faster, which is exactly the failure the step exists to prevent.

## Files to Change

| File | Change |
|---|---|
| `skills/implement-next/SKILL.md` | Step 8 becomes conditional; state the resolution rule and the self-review fallback |
| `skills/init-factory/SKILL.md` | Interview question; `deep_review` in the config example |
| `docs/ARCHITECTURE.md` | `deep_review` in the config block; note the override labels |
| `README.md` | One line, only if it lists settings (check first) |

No backend skill changes: labels are read through the existing view operation, and the todo backend
already carries labels as inline `[tag]` markers.

## Implementation Steps

1. **`skills/implement-next/SKILL.md` step 8** — open with the resolution rule (config key, then the
   two labels, `skip-review` winning), then keep the existing cold-review instructions for the case
   where it is on. Add the skipped branch: the implementer reviews its own diff against the same list
   of concerns and says in its report that the cold review was skipped and why, so a reader of the
   commit can tell a deliberate skip from a missed step. Keep the existing "if the environment cannot
   run a separate agent" fallback — that is a capability failure, not a configuration choice, and the
   two should not be conflated.
2. **`skills/init-factory/SKILL.md`** — add the interview question after the feature-branches one
   (they are the same kind of question), and add `deep_review: true` to the config example.
3. **`docs/ARCHITECTURE.md`** — add the key to the config block with a trailing comment, and extend
   the implement-next review paragraph with the override labels. Keep the existing justification for
   why the review is delegated; this issue changes when it runs, not why.
4. **`README.md`** — check whether it documents settings; add a line if it does, leave it alone if not.

## Testing

No automated suite for the skills (CLAUDE.md). Verification:

- Structural: frontmatter untouched, `name` still matches the directory.
- Consistency: `deep_review` spelled identically in all four files, and the label precedence sentence
  says the same thing as the `feature_branches` one it is modelled on.
- The config example in `init-factory` and the one in `ARCHITECTURE.md` must stay identical — they
  already drift-check against each other by eye.
- No behavioural run: exercising this needs a scratch project with an eligible item, and this
  iteration's own repo config has no `deep_review` key, which is itself the default-absent case.

## Risks / Edge Cases

- **Absent key must mean `true`.** Every config file written by `init-factory` before this change
  lacks the key; reading absence as `false` would silently switch reviewing off across every existing
  installation. This is the one thing worth getting exactly right.
- **The flag reading as "review is optional".** Mitigated by wording: the setting picks the reviewer,
  and the self-review path is spelled out rather than left as an omission.
- **Precedence disagreeing with its siblings.** Mitigated by copying the `skip-*` beats `needs-*`
  rule verbatim in shape.
- **Scope creep into TODO-14** (foreground vs. background review agents). Different issue; this one
  only decides *whether* the agent runs, not how it is awaited.
