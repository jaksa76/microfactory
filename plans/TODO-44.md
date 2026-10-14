# TODO-44: project override files — extra instructions for any microfactory skill

## Summary

A project cannot change how a microfactory skill behaves without forking the plugin. Add a way for
the project to supply **additional instructions** that the workflow skills read and follow, kept in
the project and versioned with it.

```
.microfactory/overrides/
  general.md           # start here: applies to every workflow skill
  implement-next.md    # skill-specific; created when general.md grows
  plan-next.md
```

Plain markdown, written by the project. Examples: "After *Implement*, run `make e2e-smoke`."
"Plans in this repo must include a rollout section." "Never touch `legacy/`."

### Design decisions

**One new skill, `overrides`, holds the rules; the workflow skills only point at it.** Same shape as
`tasks`: the workflow skills say "load the `overrides` skill" in their first step, and
`skills/overrides/SKILL.md` is the single place that defines where the files live, what precedence
means and what cannot be overridden. This keeps the guardrails in one file instead of repeated in
nine skills, and keeps the workflow skills otherwise unchanged ("one skill, one job").

**`general.md` is the first place overrides go; skill-specific files are the extraction target.**
The reading rule for a skill named `<s>`:

1. Read `.microfactory/overrides/general.md`, then `.microfactory/overrides/<s>.md`. Missing files
   are fine and mean "no overrides"; never create them.
2. In `general.md`, a section headed `## <s>` (e.g. `## implement-next`) applies **only** to that
   skill; text before the first `##` heading, or under `## all`, applies to every skill. This lets a
   developer scope rules inside the one file, and "extract when it grows" is a cut-and-paste of a
   section into `<s>.md`.
3. Precedence, most specific wins: `<s>.md` > a `## <s>` section of `general.md` > general text >
   the skill's own steps.

**Overrides add, skip or adjust steps; they never outrank the invariants.** `overrides/SKILL.md`
lists what a project cannot override, and says to ignore (and mention once in the iteration's
output) any override that tries:

- the claim protocol: assign → wait → verify → retry; creation never claims; updating never claims
  or transitions (`tasks`);
- nothing is committed or pushed before the review findings are in (`implement-next`, *Review the
  code* and *Commit, push, and update the issue*); planning mode implements nothing;
- no secrets are written anywhere;
- the backend stays the only source of truth for work, and durable knowledge still goes to
  `CLAUDE.md` / `LESSONS.md`, not into override files.

**Overrides refer to steps by title, not number**, so renumbering a skill does not silently break
them; the skill's step titles are therefore stable names. The doc says so, and that an override
which names a step that no longer exists is reported, not guessed at.

**Overrides must not need a human.** Skills run unattended in `/loop`; an override that asks
a question or waits for approval would hang the iteration. The reading rule says to treat that as an
instruction the skill cannot follow, and skip it with a note.

**Which skills read overrides:** `plan-next`, `implement-next`, `refine-story`, `breakdown-feature`,
`find-improvements`, and the four `find-*-improvements` skills (each can be run on its own). Not
`tasks` (backend mechanics are not project policy), `start`, `start-planning`, `init-factory` or
`install-agentize`.

**`init-factory` does not create the files.** It mentions the directory in its closing message so
developers know it exists. Empty placeholder files would suggest they are expected.

**As built, where this plan said otherwise** (found in the cold review and by running the scratch repo
from Testing):

- `plan-next` and `implement-next` load `overrides` at the end of step 2, **after** the pull, not in
  step 1 — before it, a just-merged override change would apply one iteration late. The other seven
  skills keep it in step 1.
- `implement-next` loads the overrides for `plan-next` as well, because it borrows `plan-next`'s
  analysis, plan-writing and thread-reading steps; both apply to those steps and the borrower wins on
  conflict. The load sentence in `implement-next` therefore names two skills.
- The review invariant says every implementation is reviewed and an override can change who
  reviews, never whether, so "skip the review" cannot be read as "no findings outstanding".
- Scoping is spelled out: a leading `# Title` is general text, `###` belongs to its `##` section, a
  `##` section named neither `all` nor a skill is ignored, headings match case-insensitively.
- "Unattended" means a `/loop` or `claude -p`, and the loop targets always count. Asking *through the
  backend* (a comment) is allowed; stopping to wait for the user is not. The finders have no approval
  conversation, so only `breakdown-feature` and `find-improvements` may add to one.
- `docs/ARCHITECTURE.md`'s status paragraph said `setup` checks the Azure DevOps states; that was
  made false by TODO-43's restructuring and is corrected here.

## Files to Change

| File | Change |
|---|---|
| `skills/overrides/SKILL.md` | **New.** Frontmatter `description`: read the project's override files for a microfactory skill (`.microfactory/overrides/general.md` and `<skill>.md`) and apply them under fixed precedence; loaded by the workflow skills, not for direct use. Body: file locations, the `## <skill>` / `## all` scoping, precedence, the invariants list, step-titles rule, unattended rule. |
| `skills/plan-next/SKILL.md`, `skills/implement-next/SKILL.md`, `skills/refine-story/SKILL.md`, `skills/breakdown-feature/SKILL.md`, `skills/find-improvements/SKILL.md`, `skills/find-design-improvements/SKILL.md`, `skills/find-testing-improvements/SKILL.md`, `skills/find-library-improvements/SKILL.md`, `skills/find-ui-improvements/SKILL.md` | One sentence in step 1: "Load the `overrides` skill for `<this skill>` and follow what it returns." Nothing else changes. |
| `skills/init-factory/SKILL.md` | One line in the closing message about `.microfactory/overrides/`. |
| `docs/ARCHITECTURE.md` | Repository layout gains `overrides/`; short section *Overrides* (locations, precedence, invariants). |
| `README.md` | A short *Customizing* paragraph with the `general.md` example. |
| `CLAUDE.md` | Repository layout block gains the `overrides/` entry. |

Not changed: `tasks` and its backends, `start`, `start-planning`, `install-agentize`,
`.claude-plugin/plugin.json` (no version bump).

## Implementation Steps

1. Read `docs/LESSONS.md` and the step-1 text of every skill in the table, and `implement-next`
   end to end (its step titles are the names overrides will use).
2. Write `skills/overrides/SKILL.md` as above. Keep it short; the invariants list must name the
   protocol and the review-before-commit rule explicitly.
3. Add the one-sentence load line to step 1 of each of the nine skills. Where a skill already loads
   `tasks`, put the override load after it.
4. `init-factory` closing line, `ARCHITECTURE.md`, `README.md`, `CLAUDE.md` per the table.
5. Per the CLAUDE.md rule on optionality, re-read each of the nine skills end to end: an override can
   now change any step, so sentences like "always hands the diff to a fresh agent" need to be true
   *given the invariants*, and any step that is unconditional by design should say so.
6. Commit; no version bump.

## Testing

No automated suite exists for skills (see CLAUDE.md). Verification:

- **No override, no change.** With no `.microfactory/overrides/` directory the todo end-to-end from
  CLAUDE.md (scratch repo, `backend: todo`, `claude -p "/microfactory:implement-next"` and the
  `plan-next` equivalent) behaves exactly as before: checkbox, note line, commit.
- **General override.** `general.md` with "append the line `override-ran` to the note" under
  `## implement-next`; run `implement-next` and check the note. Run `plan-next` and check it does
  **not** apply there.
- **Precedence.** Put a conflicting instruction in `general.md` and in `implement-next.md`; the
  skill-specific one wins.
- **Invariants.** An override saying "commit before the review" or "skip the claim verification" is
  ignored and mentioned; the iteration still reviews before committing.
- **Interaction.** An override that says "ask the user before continuing" does not hang a headless
  `claude -p` run; it is skipped with a note.
- **Missing step.** An override naming a step title that does not exist is reported, not guessed at.
- **Static.** All nine skills contain the load sentence; none of `tasks`, `start`, `start-planning`
  does; `skills/overrides/SKILL.md` lists each invariant named above.

## Risks / Edge Cases

- **Weakened safety is the main risk.** Free-form text can say anything. The invariants list is the
  defence, and it only works if the model reads it, so it sits in the first lines of
  `overrides/SKILL.md`, not at the end. It does not make unsafe overrides impossible; review of
  `.microfactory/overrides/` in PRs is the real control, and the README paragraph says so.
- **Drift against skill changes.** Step titles become an interface. A skill rewording a title
  breaks overrides silently, except for the "named step not found" report. Keep titles stable and
  mention it in `CLAUDE.md`'s conventions if this proves a problem.
- **Instruction bloat.** Every iteration reads the files again, so `general.md` growing unboundedly
  costs context in every run. The extraction path (`## <skill>` sections → `<skill>.md`) bounds it
  per skill, not overall. Judgement for a human; not engineered here.
- **Overlap with the project's `CLAUDE.md`.** General project rules already belong there and are
  loaded automatically. Say in the README that overrides are for *how the factory behaves*, and
  `CLAUDE.md` is for *how the project works*, so the same sentence is not written twice.
- **Overlap with config.** A rule that fits a setting (a plain on/off) should be a setting, as
  `deep_review` and `feature_branches` are. Overrides are for behaviour the settings cannot name.
- **Named hooks later.** If the same few overrides recur ("run X before commit"), promoting them to
  named extension points is a compatible follow-up; this change does not preclude it.
