# TODO-45: a `/microfactory:customize` skill to add, change, extract, remove and explain overrides

## Summary

TODO-44 adds project override files (`.microfactory/overrides/general.md` and `<skill>.md`) that the
workflow skills read unattended. This item adds the interactive counterpart: a skill a developer
runs to *maintain* those files without having to know the precedence rules, the invariants or the
step titles. It is to the override files what `init-factory` is to `config.yaml`.

```
/microfactory:customize <plain-language change>     add or change an override
/microfactory:customize list [skill]                 what is overridden, where
/microfactory:customize explain [skill|"<text>"]     what each override does and why it applies
/microfactory:customize remove <skill|"<text>">      delete an override
/microfactory:customize extract <skill>              move a section of general.md into <skill>.md
```

The subcommand is inferred from the request, so `/microfactory:customize always run make e2e-smoke
after implementing` is an *add*. One skill, one job: **maintaining the override files**. It never
runs a factory iteration and the workflow skills never load it.

**Depends on TODO-44**: it reads and writes the files that item defines and relies on the `overrides`
skill for the reading rule and the invariants. Implement TODO-44 first; if `skills/overrides/SKILL.md`
does not exist, the skill says so and stops.

### Design decisions

**It does not commit.** Override files are project policy that changes how unattended loops behave,
which makes the diff worth a human's attention in a PR, and "commit or push only when asked" is the
default everywhere else. The skill shows the diff, writes the file after approval, and ends with
`git status` and a one-line "commit this with your usual change" reminder. Contrast `implement-next`,
which commits because a commit *is* its deliverable.

**It finds step titles in the sibling skill, relative to its own location.** A skill is loaded from
wherever the plugin is installed (the project, a plugin cache, a marketplace checkout), so the
project's tree cannot be assumed to contain the skill files. The skill is given its own base
directory when it loads; every other microfactory skill is a sibling:
`<base>/../<skill>/SKILL.md`. It reads that file's `##` headings to get the real step titles.
If the sibling cannot be read, it does not guess: it writes the override without a step reference
and says so.

**One source of truth for the rules.** The invariants, the precedence order and the list of skills
that read overrides are read from `../overrides/SKILL.md` at run time, never copied into this
skill. When TODO-44's rules change, this skill follows without an edit.

**Placement rule for a new override** (mirrors TODO-44's "start in `general.md`"):

1. A rule that applies to every skill goes at the top of `general.md`, or under `## all`.
2. A rule for one skill goes in the `## <skill>` section of `general.md`, creating it if needed
   (creating `general.md` too, with no placeholder text).
3. If `<skill>.md` already exists, the rule goes there instead, so a skill never has its rules in two
   files.
4. Offer **`extract`** when a `## <skill>` section passes about fifteen lines, or `general.md` about
   sixty: say that the section would be clearer as `<skill>.md` and move it only on a yes.

**Checks before anything is written** (each failure is shown, not silently fixed):

- **Invariants.** A request that weakens one listed in `overrides/SKILL.md` ("skip the review",
  "commit before verifying") is refused with the reason, and the closest permitted alternative is
  offered.
- **Unattended.** Text that needs a human mid-loop ("ask me before pushing") is flagged. Offer a
  rewrite that does not (for example "stop and post the question as an issue comment") or refuse.
- **Real steps.** A step the request names must exist in that skill's headings; otherwise ask which
  of the real steps is meant, and list them.
- **Known skill.** The target must be one of the skills that read overrides; list them otherwise.
- **Settings first.** If the request is a plain on/off or a value that a config key already
  controls (`plan_by_default`, `auto_plan`, `feature_branches`, `deep_review`, the intervals, and
  `statuses` once TODO-43 lands), point at that key (and `init-factory`) instead of writing prose.
- **Contradiction and duplicates.** Compare against the existing text for that skill, including
  precedence: an addition that an existing `<skill>.md` line already overrides is shown, with the
  winning line quoted.

**Explain is read-only** and is the path for "why is the factory doing this?". For each override
(or each one that matches the argument) it reports:

- the **skill and step** it changes, by real title;
- what the skill does **without** it (quoted from the skill's own step);
- the **file and section** it comes from;
- how it **interacts** with other overrides (which one wins by precedence) and with the invariants;
- whether it is **being ignored**, and why: it names a step that no longer exists, it conflicts with
  an invariant, or it needs a human mid-loop.

`explain <skill>` also prints the *effective* instruction set after precedence is applied, which is
the view a debugging developer wants.

**Remove** shows the exact lines (and the file left empty, if so), asks, then deletes. An emptied
`<skill>.md` is deleted; an emptied `general.md` section header is removed with it.

**As built, where this plan said otherwise** (found by the cold review and the scratch runs from Testing):

- **Approval is one section** shared by add, remove and extract. "Apply without asking" approves only
  what the user asked for as drafted — never an offered alternative, never past the checks — and with
  no one to ask and no such instruction, the skill prints the change and writes nothing (this
  non-interactive rule was add-only in the plan).
- **If the `overrides` skill cannot be read, the skill stops** with an accurate diagnosis rather than
  checking blind. *Step titles* unreadable is a separate, softer fallback: keep the user's wording of
  the step in the rule and mark it unverified, rather than the plan's "write it without a step
  reference", which would change the rule from "at this step" to "throughout".
- **More checks than planned:** a rule that names a step is never general (titles repeat across skills),
  and borrowed steps count (`implement-next` may name `plan-next`'s); "belongs elsewhere" points
  project facts (build, tests, layout) at `CLAUDE.md`; general rules are compared against every
  skill-specific file they would be shadowed by, a same-level contradiction **replaces** the old line
  instead of sitting beside it, and "unattended" is judged per target, including for general rules.
- **A new rule never deletes an existing line the user did not name.** Found in a scratch run: for
  "for every skill, always run make e2e-smoke" against an `implement-next.md` containing "Never run
  the e2e suite", the first version replaced that line and quietly narrowed the scope. Now only a
  request that *changes an existing rule* replaces its line; a contradicting new rule shows both and
  asks (writing nothing when nobody can answer), and a scope the rule cannot have is asked about, not
  narrowed.
- **List and explain also report text no skill will read** (a `##` section named neither `all` nor a
  skill, a `<skill>.md` for a skill that does not read overrides); extract refuses `all` and unknown
  names and normalises the file name.
- The config-key list is not copied into the skill: it is read from `init-factory` each time.
- Draft says what was asked and no more; extra requirements are offered as suggestions.
- `ARCHITECTURE.md` says the skill works wherever the plugin is installed *provided the agent tells a
  skill where it lives*; otherwise it stops.

## Files to Change

| File | Change |
|---|---|
| `skills/customize/SKILL.md` | **New.** Frontmatter `description`: add, change, list, explain, extract and remove the project's microfactory override files (`.microfactory/overrides/`); interactive, not run by loops. Body per the design decisions above. |
| `skills/init-factory/SKILL.md` | The closing line TODO-44 adds about `.microfactory/overrides/` also names `/microfactory:customize`. |
| `docs/ARCHITECTURE.md` | Repository layout gains `customize/`; the *Overrides* section (TODO-44) says files are maintained with it. |
| `README.md` | The *Customizing* paragraph (TODO-44) shows `/microfactory:customize`. |
| `CLAUDE.md` | Repository layout block gains the `customize/` entry. |

Not changed: `skills/overrides/SKILL.md` (it stays the single source of truth this skill reads), the
nine skills that load it, `tasks`, `.claude-plugin/plugin.json` (no version bump).

## Implementation Steps

1. Confirm TODO-44 has landed: `skills/overrides/SKILL.md` and the load lines in the nine skills.
   If not, stop and report; this item cannot be built first.
2. Read `docs/LESSONS.md`, `skills/overrides/SKILL.md` end to end (the rules this skill defers to)
   and `skills/init-factory/SKILL.md` (the model for an interactive skill that writes project files:
   interview, show, approve, write).
3. Write `skills/customize/SKILL.md` in this order: how it reads the rules (`../overrides/SKILL.md`)
   and finds step titles (`../<skill>/SKILL.md`); the subcommand inference table; the placement
   rule; the checks; explain's report; remove and extract; the closing `git status` reminder.
   Keep it about the length of `init-factory`; the checks are one bullet each.
4. `init-factory` closing line, `ARCHITECTURE.md`, `README.md`, `CLAUDE.md` per the table.
5. Per the CLAUDE.md rule on optionality, re-read the files touched end to end, and check that no
   sentence from TODO-44 ("`init-factory` does not create the files", "the project writes them by
   hand") is now false.
6. Commit; no version bump.

## Testing

No automated suite exists for skills (see CLAUDE.md). Verify with a scratch git repo
(`backend: todo`, `.microfactory/config.yaml`) in which TODO-44's `overrides` skill is present:

- **Add, general.** `claude -p "/microfactory:customize after implementing, append the word
  override-ran to the iteration note"` with `-p` approving the diff: `general.md` gains it under
  `## implement-next` and no `<skill>.md` is created. Run `implement-next` on a todo item and check
  the note, which closes the loop with TODO-44.
- **Placement.** With an existing `implement-next.md`, a second request for the same skill lands
  there, not in `general.md`. A request naming no skill lands at the top of `general.md`.
- **Real step.** "after the Review step" resolves to the heading *Review the code*; "after step 99"
  is refused or clarified, with the real titles listed.
- **Invariants.** "skip the code review" and "commit before verifying" are refused with the reason
  and nothing is written.
- **Unattended.** "ask me before pushing" is flagged and not written verbatim.
- **Settings.** "turn off the cold review" is answered with `deep_review: false`, and no override
  is written.
- **Contradiction.** A request that `implement-next.md` already contradicts reports the winning line
  and does not write the losing one silently.
- **Explain.** With one valid override, one naming a removed step and one violating an invariant,
  `explain` lists all three, the last two as ignored with the right reason; `explain implement-next`
  prints the effective set.
- **Extract and remove.** Past the size thresholds extract is offered and moves the section
  unchanged; remove shows lines, asks, and deletes an emptied `<skill>.md`.
- **No commit.** After every mode `git log` is unchanged and `git status` shows the file changes.
- **Plugin installed elsewhere.** Run it with the plugin loaded from outside the project (a
  `--plugin-dir` checkout) and confirm step titles still resolve via the base directory. If the
  harness does not provide the base directory, the skill must fall back to "no step reference" and
  say so; confirm that path too.

## Risks / Edge Cases

- **Base directory assumption.** Finding sibling skills relative to the skill's own location depends
  on the harness giving the skill its path. If it does not on some agent (Copilot, Codex: TODO-36 and
  TODO-37), step titles cannot be checked and the fallback applies. Say that in the skill rather
  than invent a lookup; do not hardcode a plugin cache path.
- **Rules drift.** Reading the invariants from `overrides/SKILL.md` avoids copying them, but the
  skill's *checks* still assume what kinds of rules exist (invariant, unattended, step, settings).
  If TODO-44 adds a category, this skill needs a matching check; the plan for that item should say
  so when it changes.
- **Settings list.** "Settings first" names today's config keys. A hardcoded list rots, so the skill
  reads the keys from `docs/ARCHITECTURE.md`'s configuration block or `init-factory`'s config
  example instead, whichever exists in the plugin, and falls back to saying it cannot check.
- **Natural-language ambiguity.** A request can mean several skills ("always run the linter" is
  `implement-next`, but also arguably the finders). Ask once when more than one skill fits, and
  default to the narrowest.
- **Explain is advice, not truth.** It states what a skill does by reading its text and the
  overrides; whether an agent actually follows an override is only observable in a run. Do not
  promise otherwise; the explain output says "as written".
- **Overlap with `init-factory`.** Both are interactive config-adjacent skills. Keep the boundary:
  `init-factory` owns `config.yaml` and backend setup; `customize` owns `overrides/` and sends
  settings questions back there.
- **File size.** Each `customize` run reads the nine skills' headings only when needed (the target
  and explain targets), never all of them eagerly.
