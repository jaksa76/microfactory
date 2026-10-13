# TODO-42: add support for Azure DevOps Boards through the `az` CLI

## Summary

Add a fourth task backend, `azure-devops`, alongside `jira`, `github` and `todo`. It is one new
backend skill, `skills/azure-devops-tasks/SKILL.md`, that documents the same operations the other
three do (find eligible, claim with race verification, view, comment, list comments, transition,
create, update) using `az boards` from the `azure-devops` extension of the Azure CLI. The workflow
skills (`plan-next`, `implement-next`, `refine-story`, ...) stay backend-agnostic; they only need to
learn that the name exists. The claim protocol is preserved exactly (assign → wait → verify → retry).

No refinement thread exists on this item and it carries no `needs-refinement` tag, so there are no
product questions to settle; the design choices below are mine.

### Design decisions

**Statuses are tags, like GitHub's labels — not Azure DevOps states.** ADO states depend on the
process template (Basic: To Do/Doing/Done; Agile: New/Active/Resolved/Closed; Scrum:
New/Approved/Committed/Done; CMMI: Proposed/Active/Resolved/Closed) and cannot be extended from the
CLI, so the factory's statuses (`Planning`, `Awaiting Plan Review`, `Plan Approved`, ...) have no
portable state to map to. Tags (`System.Tags`) are available in every process and are what the
labels of `github-tasks` already model. The mapping is the same one GitHub uses (`in-progress`,
`in-planning`, `awaiting-plan-review`, `plan-approved`, `in-review`), so the labels the README and
the override rules already name (`needs-plan`, `skip-plan`, `needs-branch`, `needs-refinement`, ...)
mean the same thing on every backend.

**Done is the one real state change.** Setting the work item to a completed state is what makes the
card leave the board and the item leave the eligible set. The completed state name varies by
process (`Done` in Basic and Scrum, `Closed` in Agile and CMMI); the skill says to try `Done`, then
`Closed`, and to use whichever the work item's process accepts. Eligibility is expressed through
the process-independent **`[System.StateCategory]`** field — `= 'Proposed'` means "open, not
started" in every process — so the WIQL needs no per-process state names.

**Key = `ADO-<id>`.** The work item id is a bare integer, which would produce `plans/123.md`,
branch `feature/123` and PR title `[123]` — ambiguous next to the numeric GitHub keys and
collision-prone if a repo ever switches backend. The factory key is `ADO-<id>` (the same shape as
`TODO-<n>`); the backend skill strips the prefix when calling `az --id`. When a user passes a bare
number to `plan-next`/`implement-next`, treat it as the id.

**Configuration.**

```yaml
backend: azure-devops
project: MyProject                       # Azure DevOps project name
azure_devops_org: https://dev.azure.com/myorg
azure_devops_user: me@example.com        # identity used for self-assignment
```

Every `az boards` call passes `--org` and `--project` from config explicitly rather than relying on
`az devops configure` defaults, so one machine can serve several projects. Credentials stay in
`az`'s own auth store (`az login`, or `az devops login` with a PAT); nothing secret goes in config.

**Eligibility** mirrors `github-tasks`, with one deliberate addition. Common WIQL preamble:

```
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = '<project>'
  AND [System.StateCategory] = 'Proposed'
  AND [System.AssignedTo] = ''
  AND [System.WorkItemType] NOT IN ('Epic', 'Feature')
ORDER BY [Microsoft.VSTS.Common.StackRank] ASC, [System.Id] ASC
```

(`Epic`/`Feature` are excluded because they are containers, never one-iteration units.) Tag
conditions are appended with `[System.Tags] CONTAINS 'needs-plan'` / `NOT CONTAINS`. Table, same
shape as the other backends:

| Mode | plan_by_default | Extra WIQL conditions |
|---|---|---|
| planning | true | `NOT CONTAINS 'skip-plan'` |
| planning | false | `CONTAINS 'needs-plan'` |
| implementation | true | `CONTAINS 'plan-approved'` OR `CONTAINS 'skip-plan'` |
| implementation | false | `NOT CONTAINS 'needs-plan'` |

Addition: planning mode also requires `NOT CONTAINS 'awaiting-plan-review'` and `NOT CONTAINS
'plan-approved'`. On Jira the status leaves `To Do` after planning, so a planned issue drops out of
the planning query by itself; with tags-as-status, state stays `Proposed` and the item would be
re-planned forever once the planner removes itself as assignee. (See Risks: `github-tasks` may have
the same flaw.)

**Tags are read-modify-write.** `az boards work-item update` has no `--add-tag`; tags are the single
field `System.Tags`, a `; `-separated string that a write *replaces*. The skill must say so loudly:
re-fetch the item, compute the new tag list, write the whole string via
`--fields "System.Tags=a; b; c"`. Two sessions doing this concurrently is safe only because the
claimer owns the item (it is assigned) — the claim itself adds `in-progress` *after* verification,
never before, so racing workers cannot clobber each other's tags.

**Comments.** `az boards work-item` has no comment-list command. Add with
`az boards work-item update --id N --discussion "<text>"`; list with the REST comments API through
`az devops invoke --area wit --resource comments --route-parameters project=<project> workItemId=<N> --api-version 7.1-preview`
(oldest first; each has `createdBy.displayName`, `createdDate`, `text`). Comment text comes back as
**HTML**: the skill says to strip tags and decode entities before looking for the
`**Refinement questions**` marker, and to treat the marker line, not the markup, as the contract.

## Files to Change

| File | Change |
|---|---|
| `skills/azure-devops-tasks/SKILL.md` | **New.** The backend skill, same section order as `github-tasks` (Finding eligible, Claiming, Creating, Updating, Other operations incl. transition table) plus a short "Identity and tags" note. Frontmatter `description` precise: "Azure DevOps Boards backend operations ... via the az CLI. Used by plan-next and implement-next when the configured backend is azure-devops. Not for general Azure DevOps questions." |
| `skills/plan-next/SKILL.md` | Line 14: add `azure-devops-tasks` to the list of backend skills. |
| `skills/implement-next/SKILL.md` | Line 14: same. |
| `skills/find-improvements/SKILL.md` | Line 16: same. |
| `skills/init-factory/SKILL.md` | Frontmatter description and interview Q1/Q2 gain `azure-devops` (project = ADO project name, plus org URL); step 2 gets an `azure-devops` auth bullet (`az extension add --name azure-devops` if missing, `az account show` / suggest `! az login`, user identity from `az ad signed-in-user show --query userPrincipalName` or ask); config example gains `azure_devops_org` / `azure_devops_user` under an "azure-devops backend only" comment. |
| `docs/ARCHITECTURE.md` | Diagram label `[Task backend: jira \| github \| todo \| azure-devops]`; layout list entry; config block (`backend` comment + two keys); claim-wait note (~5s for ADO); status-mapping paragraph (ADO = tags + a real Done state). |
| `README.md` | Backend prerequisites line (`az` + `azure-devops` extension); the one sentence on status mapping (line ~92) mentions Azure DevOps tags. |
| `CLAUDE.md` | Repository layout block: add `azure-devops-tasks/`; conventions text mentions backends generically so needs no change. |
| `.claude-plugin/plugin.json` | `description`: "(Jira, GitHub Issues, Azure DevOps Boards, or a TODO.md file)". Do **not** bump `version` — releases are the maintainer's call. |

Not changed: `todo-tasks`, `github-tasks`, `jira-tasks` (their claim protocols are untouched),
`refine-story`, `breakdown-feature` (it reads existing items generically; its Jira/GitHub
template examples need no ADO entry).

## Implementation Steps

1. Read `docs/LESSONS.md` (currently empty of entries) and re-read `skills/github-tasks/SKILL.md`
   end to end; the new skill is a deliberate sibling of it.
2. Write `skills/azure-devops-tasks/SKILL.md` containing:
   - **Preamble**: config keys; every command takes `--org <azure_devops_org> --project <project>`
     (`--project` is not accepted by `update`/`show`, which take the id only — check
     `az boards work-item update --help` and omit where unsupported); auth check `az account show`
     and `az extension show --name azure-devops`, on failure stop and tell the user to run
     `! az login` / `az extension add --name azure-devops`; key = `ADO-<id>`.
   - **Finding eligible issues**: `az boards query --wiql "<WIQL>" --org ... --project ... -o json`
     with the preamble and table above. Note the result lists ids; fetch fields with
     `az boards work-item show --id N` for the ones needed. Include a worked example and the same
     "adapt quoting to the user's shell" caveat (WIQL contains single quotes; PowerShell/bash differ).
   - **Claiming**: steps identical to `github-tasks`: query; take first; `az boards work-item update --id N --assigned-to <azure_devops_user>`; wait ~5 s; re-`show` and compare `System.AssignedTo.uniqueName` to `azure_devops_user` (case-insensitive); lost race → back to step 1; then add the `in-progress` tag (and in planning mode `in-planning` instead, never both); finally `show` for title/description. Explicit sentence: tags are added only after the assignment is verified.
   - **Creating**: `az boards work-item create --type "<type>" --title ... --description ... --fields "System.Tags=a; b" --org ... --project ...`, leave unassigned. Type: default `Task` (exists in all four standard processes); if the user's breakdown wants a story, use the process's story type (`User Story`/`Product Backlog Item`/`Issue`) — discover via the existing backlog's item types rather than guessing.
   - **Updating**: `update --id N --title/--description`; never touches assignee, status tags or state. Tag edits via read-modify-write (see above).
   - **Other operations**: View (`show --id N`, `System.Tags`, `System.State`, `System.AssignedTo`); Comment (`--discussion`); List comments (`az devops invoke ... comments`, HTML note); Transition table:

     | Status | Change |
     |---|---|
     | In Progress | add tag `in-progress` |
     | Planning | add `in-planning`, remove `in-progress` |
     | Awaiting Plan Review | add `awaiting-plan-review`, remove `in-planning`, **clear the assignee** (`--assigned-to ""`) |
     | Plan Approved | done by a human: they add `plan-approved`, remove `awaiting-plan-review` |
     | In Review | add `in-review`, remove `in-progress` |
     | Done | remove `in-progress`/`in-review`, set `--state` to the process's completed state (`Done`, else `Closed`) |

     Failed tag/state changes after the work is done are warnings, not errors.
3. Edit the three load-line mentions (`plan-next`, `implement-next`, `find-improvements`) to include
   `azure-devops-tasks`. Verify with a grep for `github-tasks` across `skills/` that no other file
   enumerates the backends (the earlier grep found only these three plus `init-factory`).
4. Update `init-factory` (interview, auth bullet, config example, frontmatter description).
5. Update `docs/ARCHITECTURE.md`, `README.md`, `CLAUDE.md`, `plugin.json` as in the table.
6. Per the CLAUDE.md rule on optionality, re-read `init-factory`, `plan-next`'s step 6 (it derives
   a blob URL "when it is a GitHub repo" — for an ADO-hosted repo the fallback "mention the repo
   path" already applies; leave as is) and `ARCHITECTURE.md` end to end for sentences that assumed
   three backends ("Jira or GitHub" in `find-improvements` line 46 and `breakdown-feature` line 66
   name only the team-visible backends — add Azure DevOps there, a few words each).
7. Commit with the implementation (single commit); no version bump.

## Testing

No automated suite exists for skills (see CLAUDE.md). Verification:

- **Static**: grep that `azure-devops` / `azure-devops-tasks` spellings are identical everywhere;
  that every config key named in `init-factory` appears in `ARCHITECTURE.md`; that the new skill
  has the same top-level sections as `github-tasks`.
- **CLI dry-checks** (read-only, no org needed): `az boards query -h`, `az boards work-item
  update -h`, `az devops invoke -h` — confirm each flag the skill names exists in the installed
  extension (1.0.8) and that `--project` is accepted exactly where the skill uses it.
- **Live smoke** (needs an ADO org with a scratch project; if none is available the implementer
  states that this was not run): create a work item with tags `needs-plan`; run
  `/microfactory:plan-next` and check the assignee, the `in-planning` → `awaiting-plan-review` tag
  swap and the cleared assignee; approve it as a human would (add `plan-approved`, remove `needs-plan` and `awaiting-plan-review`, since
  implementation with `plan_by_default: false` excludes `needs-plan`), run `/microfactory:implement-next`, check Done state. Also check that a
  second concurrent claim attempt loses (assign the item to someone else during the 5 s wait).
- Check the comment round trip: post a `**Refinement questions**` comment via `--discussion` and
  confirm `az devops invoke ... comments` returns it so the marker is still findable after
  stripping HTML.

## Risks / Edge Cases

- **Assignee identity formats.** `--assigned-to` accepts a display name or UPN; the readback
  `System.AssignedTo` is an object (`displayName`, `uniqueName`). Compare on `uniqueName`;
  ambiguity on display names is why config holds the UPN/email.
- **HTML comments.** `--discussion` text is stored as HTML, and line breaks in the marker comment
  may be flattened. The skill must tell the reader to match the marker line after stripping
  markup; if the thread's structure turns out unreliable in the live smoke test, the `refine-story`
  footer convention may need a note — flag it rather than widen this change.
- **Tag races.** Read-modify-write on `System.Tags` is not atomic. Safe only because tags are
  edited by whoever currently owns the item; humans editing tags during a transition can lose an
  update (the work-item revision check in the REST API would catch it but `az boards` does not
  expose it). Documented, not engineered around.
- **Process variance.** Completed-state names, available types, and whether `StackRank` exists vary
  by process; the skill names the fallbacks (`Done`/`Closed`; ordering falls back to `[System.Id]`).
  A custom process with a non-standard `Done` category state is the user's to configure; the skill
  should say to ask the user once rather than guess.
- **`NOT CONTAINS` on empty tags.** In WIQL, `[System.Tags] NOT CONTAINS 'x'` does match items with
  no tags, unlike Jira's `labels NOT IN`, so no `OR labels is EMPTY` clause is needed — verify in
  the smoke test and drop the note if wrong.
- **PRs.** `implement-next` step 9 opens PRs; whether that uses `gh` or `az repos pr create`
  depends on where the code is hosted, not on the task backend. Out of scope here; a repo hosted on
  Azure Repos with `feature_branches: true` needs the implementer to adapt the PR command, as it
  already does per OS/shell.
- **Possible existing bug, not fixed here.** In `github-tasks`, planning mode with
  `plan_by_default: false` matches any open unassigned issue labelled `needs-plan`, and the
  Awaiting Plan Review transition unassigns the planner without removing that label, so a planned
  issue looks claimable for planning again. This plan avoids it for ADO via the extra exclusions;
  whether to fix GitHub is a separate judgement — record it in `docs/LESSONS.md` when implementing
  if it is confirmed, do not widen this change.
- Scope discipline: one new skill plus mechanical enumerations. If the skill grows past roughly
  the length of `github-tasks` (~70 lines) something has gone beyond "the simplest thing".
