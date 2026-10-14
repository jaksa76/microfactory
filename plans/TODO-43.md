# TODO-43: add a `statuses:` override map so a project can map the factory's statuses to its own

## Summary

Today the factory's seven statuses are hardcoded by name in the Jira and Azure DevOps backend docs
(`To Do`, `Planning`, `Awaiting Plan Review`, `Plan Approved`, `In Progress`, `In Review`, `Done`;
on Azure DevOps `New`/`To Do`/`Proposed` as the initial state, with fallbacks for In Progress and
Done). A project whose workflow already has `Active`, `Ready` or `In QA` has to add duplicates next
to them or cannot use the factory. Add one optional, backend-neutral `statuses:` map in
`.microfactory/config.yaml` that renames them.

```yaml
statuses:                      # optional; omitted keys keep the backend's default
  initial: [New, To Do]        # "not started"
  planning: In Planning
  awaiting_plan_review: Awaiting Plan Review
  plan_approved: Ready
  in_progress: Active
  in_review: In QA
  done: Closed
```

### Design decisions

**Keys are logical, values are backend-native.** The seven keys are the factory's statuses. A value
is a Jira status or an Azure DevOps state, whichever `backend` names. `github` and `todo` ignore the
map (labels and checkbox characters are not renamed by this item — see Risks) and `init-factory`
does not ask for it there.

**Every value is a name or a list of names.** A list is *matched* with `IN (...)` in a query and
*tried in order* for a transition (first the item accepts wins). This one rule replaces today's
special cases: Azure DevOps's initial state `New`/`To Do`/`Proposed`, its In Progress fallback
`Active`/`Doing`/`Committed`, and its Done fallback `Done`/`Closed` become the **defaults**, written
as lists, instead of prose in the backend doc. (In the design discussion only `initial` was a list;
making the rule uniform is simpler than singling it out.)

**Defaults are today's behaviour.** With no `statuses:` the queries and transitions the backends
issue must select and do the same as the current ones. *As built:* they are semantically identical,
not byte-identical — `status = "To Do"` is now `status IN ("To Do")` so a list fits the same shape,
and the Jira implementation/`plan_by_default: true` row is wrapped in parentheses. That row was a bare
`status = "Plan Approved" OR (...)` appended after `AND assignee is EMPTY`, so the OR bound looser than
the `AND`s and let assigned items through; the parentheses fix that as a side effect.

**As built, where this plan said otherwise:**

- The Azure DevOps state check is *not* a `setup` step. `setup` runs before any mapping exists, so
  it only lists each type's states; the check is a separate section, *Checking the mapping*, run by
  `init-factory` after the mapping is settled.
- `init-factory` validates the **resolved** values (defaults included), so `plan_approved: Active`
  against the default `in_progress` list is a clash. It warns about a populated `plan_approved` only
  when that name is new (neither the default nor already in the config) and counts a status that does
  not exist yet as zero. It pre-fills from an existing config's `statuses:`.
- The config example is commented out, so copying the template writes no mapping.
- The old "ask the user once if neither `Done` nor `Closed` works" fallback is gone: it cannot run
  inside an unattended loop, and a failed transition after the work is already a warning.
- Live check against `checkitlabsdev` / `CheckIT Learning`: the overridden queries parse, a
  `plan_approved` of `Ready` holds 141 items (so the warning would fire), and **WIQL returns no rows,
  not an error, for a state name that does not exist** — recorded in `azure-devops.md`.
- A line in the `tasks` skill says a hand-edited map skips `init-factory`'s checks.

| Key | Jira default | Azure DevOps default |
|---|---|---|
| `initial` | `To Do` | `New`, `To Do`, `Proposed` |
| `planning` | `Planning` | `Planning` |
| `awaiting_plan_review` | `Awaiting Plan Review` | `Awaiting Plan Review` |
| `plan_approved` | `Plan Approved` | `Plan Approved` |
| `in_progress` | `In Progress` | `In Progress`, `Active`, `Doing`, `Committed` |
| `in_review` | `In Review` | `In Review` |
| `done` | `Done` | `Done`, `Closed` |

**Where names are resolved: the `tasks` skill.** `plan-next`, `implement-next` and `refine-story`
keep naming logical statuses ("transition it to `In Progress`"), so they stay backend-agnostic as
CLAUDE.md requires. `skills/tasks/SKILL.md` gains a short *Status names* section: the logical name →
`statuses.<key>` → the backend default. The backend docs stop hardcoding names; their eligibility
tables, claim step, setup check and transition table say "the configured `plan_approved` status".
Text addressed to a **human** (the reviewer who must move an item to Plan Approved) uses the
resolved name, not the logical one.

**`null` skips a status.** `in_review: ~` means "this project has no such status": the transition is
skipped silently instead of warning every iteration. Allowed only for `planning`, `in_progress` and
`in_review`, the three that do not decide eligibility. `initial`, `plan_approved`,
`awaiting_plan_review` and `done` must be set: skipping `awaiting_plan_review` would leave a planned
item in the initial state to be planned forever.

**Validation is `init-factory`'s job**, at the one moment a human is present:

1. Names are **distinct** across keys (case-insensitive, and no overlap between lists). If
   `plan_approved` and `initial` share a name, approved items look unplanned and the claim logic
   breaks.
2. **Warn when `plan_approved` maps onto a state that already holds items.** Count them (a JQL/WIQL
   count) and ask for confirmation: mapping it to a state with 119 existing items makes the
   implementer pick up all 119 as approved. Other keys do not need the warning; existing items in
   `Active` are the point of mapping `in_progress` there.

**`init-factory` proposes the mapping, never makes the user author YAML.** After backend setup it
shows the defaults next to the project's real statuses and writes **only the keys that differ**
under `statuses:`. Where the backend can list them: Azure DevOps via the state-listing call already
in its Setup section. Jira has no list-statuses command in `acli` (see `jira-acli-reference.md`:
"just try transitioning"), so for Jira it asks for the names with the defaults pre-filled.

## Files to Change

| File | Change |
|---|---|
| `skills/tasks/SKILL.md` | New *Status names* section (logical name → `statuses.<key>` → default; list and `null` semantics; skip rule). The `Statuses` paragraph under "Rules every backend keeps" points at it and says the names listed there are logical. |
| `skills/tasks/backends/jira.md` | Replace the literal names in the eligibility table, claim step 6, the transition line and the closing "Statuses used by the factory workflow" sentence with the configured statuses; add the defaults table. Search/transition commands are unchanged. |
| `skills/tasks/backends/azure-devops.md` | Replace the *Statuses are states* table with the defaults table above; the `INITIAL` macro becomes "the `initial` list"; the *Check the states* setup step checks the **configured** names, not the literal four; the Transition bullet resolves through the config. The In Progress / Done fallback prose is deleted (it is now the default list). |
| `skills/init-factory/SKILL.md` | New interview step (statuses, Jira and Azure DevOps only) after backend setup; validation per Design decisions; config example gains a commented `statuses:` block; the "include only the keys relevant" sentence covers it. |
| `skills/plan-next/SKILL.md`, `skills/implement-next/SKILL.md`, `skills/refine-story/SKILL.md` | Only where a sentence is addressed to a human and names a status (e.g. "A human reviews the plan and transitions the issue to `Plan Approved`"): say "the configured plan-approved status". Logical names in instructions to the agent stay. |
| `docs/ARCHITECTURE.md` | Config block gains `statuses:`; the *Status mapping* paragraph describes the override and which backends honour it. |
| `README.md` | Status-mapping sentence (~line 92) and the workflow list (lines 71–72) mention that the names are configurable on Jira and Azure DevOps. |

Not changed: `github.md` and `todo.md`, `CLAUDE.md`, `plugin.json` (no version bump).

## Implementation Steps

0. **Prerequisite.** This builds on the states-based rewrite of `azure-devops.md` (plus the matching
   `README.md` / `ARCHITECTURE.md` lines). Make sure that change is committed first; if it is not,
   implement it as part of this item.
1. Read `docs/LESSONS.md` (empty at the time of writing) and the current `tasks/SKILL.md`,
   `backends/jira.md`, `backends/azure-devops.md`, `init-factory`, `plan-next`, `implement-next`,
   `refine-story` end to end.
2. `tasks/SKILL.md`: write *Status names* — the seven keys, resolution order, name-or-list rule
   (matched with `IN`, tried in order for transitions), `null` rule and which keys allow it,
   "`github` and `todo` ignore `statuses:`".
3. `backends/jira.md`: add the defaults table; rewrite the eligibility table with the configured
   statuses (`status IN (<initial>)`, `status IN (<plan_approved>)`); claim step 6 and the
   transition bullet use the configured `planning` / `in_progress`. Keep the transition command and
   `--yes` exactly as they are. A transition to a status the item cannot reach stays a warning after
   the work, as today.
4. `backends/azure-devops.md`: same treatment. `INITIAL` is replaced by `[System.State] IN (<initial>)`;
   delete the fallback sentences now carried by the default lists; the setup state check compares the
   states each type has against the configured names for `planning`, `awaiting_plan_review`,
   `plan_approved` and `in_review` (a `null` `in_review` is not checked).
5. `init-factory`: add the interview step and validation. Flow: for Jira or Azure DevOps, show the
   defaults and the project's real statuses/states, ask which factory status each real one plays
   (default = keep), reject duplicate names, run the `plan_approved` populated-state count and ask
   for confirmation, then write only the differing keys under `statuses:` in step 3. If a config
   already exists, keep its `statuses:` block in the "show it and ask before overwriting" view.
6. Update the three workflow skills' human-facing sentences, `ARCHITECTURE.md` and `README.md`.
7. Per the CLAUDE.md rule on optionality, re-read end to end every file touched plus `start`,
   `start-planning` and `breakdown-feature`. Grep for each literal default name (`Plan Approved`,
   `Awaiting Plan Review`, `In Progress`, `In Review`, `Planning`, `To Do`) and decide for every hit
   whether it is a default, a logical name or a stale hardcode.
8. Commit; no version bump.

## Testing

No automated suite exists for skills (see CLAUDE.md). Verification:

- **No-override equivalence (the main check).** Render the Jira JQL and Azure DevOps WIQL for all
  four mode × `plan_by_default` rows with an empty `statuses:` and diff them against the text in
  today's backend docs. They must be identical.
- **Override rendering.** With the example map above, render the same eight queries and read them
  back: `initial` becomes `IN ('New', 'To Do')`, `plan_approved` becomes `'Ready'`, and so on.
- **Azure DevOps, read-only against the live org** (`checkitlabsdev` / `CheckIT Learning`):
  run the overridden eligibility queries and the populated-state count for `Ready` (expect the
  existing ~119 items to trigger the warning). Also confirm what WIQL does with a state name that
  does not exist for a type (error, or silent empty result): that decides whether a stale name after
  a process rename fails loudly. Record the answer in `azure-devops.md`.
- **Jira** (if a scratch project is available; otherwise state that it was not run): run the
  overridden JQL read-only, then one planning claim → transition with a renamed `planning` status.
  Check that an unreachable status warns and does not abort.
- **Validation.** Run `init-factory` against a mapping with a duplicate name and one that maps
  `plan_approved` onto a populated state: both must stop or ask, and nothing must be written.
- **Other backends.** With `backend: todo` and a `statuses:` block present, `implement-next` behaves
  exactly as before (the end-to-end from CLAUDE.md); `init-factory` does not ask the question.
- **Skipping.** `in_review: ~` with `feature_branches: true`: the PR is opened and no transition or
  warning is attempted.

## Risks / Edge Cases

- **Mapping onto populated states.** The one dangerous mistake is `plan_approved` (the implementer
  starts everything in it). The count-and-confirm step covers `init-factory`; a hand-edited config
  is the user's responsibility, so the schema doc says so in one line.
- **Renamed states later.** If an admin renames a mapped status, Jira's JQL errors loudly, while
  Azure DevOps may return nothing and the loop would look idle. The live check above settles which;
  if it is silent, the backend doc should tell `find` to report "no items in state X" when a query
  returns nothing for an unknown name rather than add a validation pass.
- **Quoting.** Status names with quotes or `\` must be escaped in JQL/WIQL; the docs already say
  "adapt quoting to the user's shell", and the name list is the new unquoted input. Say so once.
- **Case.** JQL status names are case-insensitive, WIQL state names are matched as written; compare
  names case-insensitively in validation, write them as the project spells them.
- **List ordering as a transition rule** (try each, first accepted wins) can mask a wrong name by
  silently landing on the next. Acceptable for the defaults; if it bites, restrict lists to
  `initial` only and keep the Azure DevOps fallbacks as prose — this plan's one design choice that
  is easy to roll back.
- **GitHub labels are out of scope.** `in-progress`, `in-planning`, ... could take the same keys
  (`in_progress: wip`) with no new mechanism. Left for a follow-up so this item stays reviewable;
  the *Status names* section must say GitHub and todo ignore the map so nobody assumes otherwise.
- **Config surface.** This adds the first non-scalar config value. The check is whether
  `init-factory` keeps the file readable: only differing keys are written, never all seven.
