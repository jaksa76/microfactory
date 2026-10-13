# TODO-42: add support for Azure DevOps Boards through the `az` CLI

## Summary

Add a fourth task backend, `azure-devops`, alongside `jira`, `github` and `todo`, using `az boards`
from the `azure-devops` extension of the Azure CLI. It supports the same operations as the other
three (find eligible, claim with race verification, view, comment, list comments, transition,
create, update). The workflow skills (`plan-next`, `implement-next`, `refine-story`, ...) stay
backend-agnostic. The claim protocol is preserved exactly (assign → wait → verify → retry).

Two structural additions were requested on top of the Azure DevOps work, and are part of this plan
(see *Additions*): **backend prerequisites are documented in each backend's own doc and triggered by
`init-factory`**, and **the four backends become nested documents of a single `tasks` skill with
explicit subcommands**. Because of the second, the Azure DevOps doc is written as
`skills/tasks/backends/azure-devops.md`, not as a skill of its own; "the backend skill" below means
that document.

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

**Installing prerequisites needs the user's approval, and is triggered only from `init-factory`.**
That is the one interactive skill; everything that runs in a `/loop` only *checks* and stops. For
each missing piece, show the exact command and ask (AskUserQuestion: install / I'll do it myself)
before running it; a "no" means stop with instructions, never a silent install. For Azure DevOps:
`az` itself is installed with the platform's package manager, adapted to the user's OS as elsewhere
in the skills (Windows: `winget install --exact --id Microsoft.AzureCLI`; macOS: `brew install
azure-cli`; Linux: Microsoft's documented package-repo/script method). The extension is
`az extension add --name azure-devops`. Do not set `extension.use_dynamic_install`: it would make
`az` install extensions silently later, which is the opposite of asking. A fresh `az` install may
need a new shell to be on `PATH`; say so rather than retrying blindly. *Where those instructions
live* is addition 1 below.

## Additions

### Addition 1 — install instructions live in each backend doc; `init-factory` only triggers them

Each backend doc gets a **Setup** section, the single place that knows its tool: what the
prerequisites are, how to detect each (`az --version`, `gh --version`, `acli --version`, the
`azure-devops` extension check, the auth-status command), how to install each per OS, and the login
step (`! az login`, `! gh auth login`, `! acli jira auth login`; todo: verify the file exists).
`init-factory` step 2 shrinks to "run the `tasks` skill's `setup` subcommand for the chosen
backend" — today it hard-codes per-backend auth text (lines 25–27) that would otherwise be
duplicated in the docs.

Setup has two modes, because installing needs a human and loops have none:

- **`setup`** (interactive, called by `init-factory`): detect → for each missing prerequisite show the
  command and ask before running it → log in → record any derived config values (Jira account id,
  GitHub username, ADO user principal name).
- **check** (the first step of every other subcommand; never installs, never prompts): run only the
  detect commands; on failure stop and say "run `/microfactory:init-factory`".

This also fixes an inconsistency: `gh` and `acli` were assumed installed and only their *auth* was
checked; they are now offered the same approved install as `az`. `acli` has no common
package-manager route, so its section links Atlassian's install instructions and the agent follows
them for the user's OS, still asking first.

### Addition 2 — one `tasks` skill, backends as nested documents, explicit subcommands

Today `jira-tasks`, `github-tasks` and `todo-tasks` repeat the same contract in three dialects, and
each workflow skill enumerates them ("`jira-tasks`, `github-tasks`, or `todo-tasks`") — which is why
adding a fourth touches several files. Replace them with one skill:

```
skills/tasks/
  SKILL.md                 dispatcher + the backend-independent contract
  backends/
    jira.md                (+ jira-acli-reference.md, the old acli-reference.md)
    github.md
    todo.md
    azure-devops.md
```

`SKILL.md` reads `.microfactory/config.yaml`, loads **only** `backends/<backend>.md`, and defines the
explicit subcommands. Callers say "run the `tasks` skill's `claim planning`"; a user can run
`/microfactory:tasks view TODO-3` too.

| Subcommand | Purpose |
|---|---|
| `setup` | Addition 1: detect/install/log in (interactive, called by `init-factory`) |
| `find <planning\|implementation>` | List eligible items, best first |
| `claim <planning\|implementation> [key]` | Find + claim with race verification; with a key, claim that item; returns key, title, description |
| `view <key>` | Summary, description, labels, assignee, status |
| `create <title> [labels] [body]` | New item, unassigned, initial status |
| `update <key> [title] [body] [add/remove labels]` | Edit content/labels; never claims or transitions |
| `comment <key> <text>` | Add a comment |
| `comments <key>` | List comments, oldest first |
| `transition <key> <status>` | Move to a canonical status |

What goes where:

- **`SKILL.md` (shared, written once)**: the subcommand contract above; the *semantic* eligibility
  table (`plan_by_default` × `needs-plan`/`skip-plan`/`plan-approved` — identical on every backend
  today); the canonical status list (`Planning`, `Awaiting Plan Review`, `Plan Approved`,
  `In Progress`, `In Review`, `Done`); the invariants every backend must keep — **claim =
  assign → wait → verify → retry**, create never assigns, update never changes assignee or status,
  post-work transitions are best effort; the setup vs check split.
- **`backends/<x>.md` (translation only)**: Setup; how to express eligibility in that backend's query
  language; the claim wait time and how to assign/verify; the status→label/state/checkbox mapping;
  the command for each subcommand; quoting and pagination gotchas.

Net effect: a new backend is one file plus one name in a config comment; the enumeration sites in
the workflow skills collapse to "use the `tasks` skill".

Trade-offs, decided deliberately:

- **Claim protocol stays safe.** It moves up into the dispatcher as an invariant, but the *wait
  durations and verification query* stay per backend. Moving text must not weaken it — the CLAUDE.md
  rule that any backend change preserve the protocol applies to the move itself, so the move is a
  pure refactor (no behaviour edits) checked by diffing the old and new text per backend.
- **"One skill, one job"** (CLAUDE.md) still holds: the job is "task-backend operations"; the old
  split was by backend, not by responsibility.
- **Cost:** renaming is a breaking change for anyone who invoked `/microfactory:github-tasks` by
  hand (the old descriptions say they are only loaded by the workflow skills, so this is expected to
  be nobody). Historical plans under `plans/` mention the old names and are left as written.
- **Size:** this is the largest part of the change by far. If the reviewer prefers, split it:
  *A* the consolidation (pure move, no new backend), *B* azure-devops + setup. The steps below are
  ordered A → C → B so either cut works; if A is deferred, write the ADO doc as
  `skills/azure-devops-tasks/SKILL.md` per the first draft and let A move it later.

## Files to Change

| File | Change |
|---|---|
| `skills/tasks/SKILL.md` | **New.** Dispatcher and shared contract (see Addition 2). Frontmatter `description`: task-backend operations (find, claim, view, create, update, comment, comments, transition, setup) for the configured jira, github, azure-devops or todo backend; used by the workflow skills and `init-factory`; not for general Jira/GitHub/Azure DevOps questions. |
| `skills/tasks/backends/jira.md` | `git mv` of `skills/jira-tasks/SKILL.md`, frontmatter and the backend-independent prose removed, **Setup** section added. |
| `skills/tasks/backends/jira-acli-reference.md` | `git mv` of `skills/jira-tasks/acli-reference.md`; the link in `jira.md` updated. |
| `skills/tasks/backends/github.md` | `git mv` of `skills/github-tasks/SKILL.md`, same treatment. |
| `skills/tasks/backends/todo.md` | `git mv` of `skills/todo-tasks/SKILL.md`, same treatment (Setup = verify the file; offer to create it with example items). |
| `skills/tasks/backends/azure-devops.md` | **New.** The Azure DevOps doc, same section order as `github.md`; content in the design decisions above and step 4 below. |
| `skills/jira-tasks/`, `skills/github-tasks/`, `skills/todo-tasks/` | Removed (emptied by the moves). |
| `skills/plan-next/SKILL.md`, `skills/implement-next/SKILL.md`, `skills/find-improvements/SKILL.md` | Line 14 / 14 / 16: replace the backend-skill enumeration with "load the `tasks` skill"; where they name an operation ("claim", "list-comments", "create"), name the subcommand. |
| `skills/init-factory/SKILL.md` | Frontmatter description and interview Q1/Q2 gain `azure-devops` (project = ADO project name, plus org URL); step 2 becomes "run `tasks setup` for the chosen backend" (replacing the three hard-coded bullets); config example gains `azure_devops_org` / `azure_devops_user` under an "azure-devops backend only" comment. |
| `docs/ARCHITECTURE.md` | Diagram label with `azure-devops`; layout block (one `tasks/` entry with `backends/`); a short section describing the subcommand contract and the setup/check split; config block (`backend` comment + two keys); claim-wait note (~5s for ADO); status-mapping paragraph (ADO = tags + a real Done state). |
| `README.md` | Prerequisites line (`az` + `azure-devops` extension) noting that `init-factory` offers to install missing CLIs; status-mapping sentence (~line 92) mentions Azure DevOps tags. |
| `CLAUDE.md` | Repository layout block: replace the three backend entries with `tasks/`; the convention "Workflow skills stay backend-agnostic: they delegate all task operations to the backend skills" → "to the `tasks` skill"; the claim-protocol convention → "any change to a backend doc". |
| `.claude-plugin/plugin.json` | `description`: "(Jira, GitHub Issues, Azure DevOps Boards, or a TODO.md file)". Do **not** bump `version` — releases are the maintainer's call. |

Not otherwise changed: `refine-story`, `breakdown-feature` (generic wording; add "Azure DevOps" to
the "Jira or GitHub" sentences at `find-improvements` line 46 and `breakdown-feature` line 66, a few
words each).

## Implementation Steps

Ordered so that A and (C + B) can land as separate commits if the reviewer splits the item.

1. Read `docs/LESSONS.md` and the three existing backend skills end to end; note every sentence
   that is backend-independent (eligibility semantics, protocol steps, "never assign on create").
2. **A — consolidate.** Create `skills/tasks/`; `git mv` the three backend `SKILL.md` files (and
   `acli-reference.md`) into `backends/`, so history follows. Write `skills/tasks/SKILL.md`: config
   loading, backend selection, subcommand table, shared eligibility table, canonical statuses,
   invariants. Trim the moved docs to translation-only, **without changing any command, wait time,
   query or mapping**; verify with a side-by-side diff of each old file against its new doc plus the
   dispatcher. Update `plan-next`, `implement-next`, `find-improvements` load lines and operation
   names; grep `skills/` for the old names (`jira-tasks`, `github-tasks`, `todo-tasks`) and fix
   every hit (`init-factory` line 27 included).
3. **C — setup.** Add a **Setup** section to each backend doc: prerequisites, detect command, per-OS
   install (Jira: `acli` per Atlassian's instructions; GitHub: `winget install GitHub.cli` /
   `brew install gh` / the distro package; Azure: as in the installing paragraph), login, derived
   config values. Add the `setup` and check semantics to `tasks/SKILL.md` (every other subcommand's
   first step is check; check never installs or prompts). Rewrite `init-factory` step 2 to call
   `tasks setup`.
4. **B — azure-devops doc.** Write `skills/tasks/backends/azure-devops.md` containing:
   - **Preamble**: config keys; every command takes `--org <azure_devops_org> --project <project>`
     (`--project` is not accepted by `update`/`show`, which take the id only — check
     `az boards work-item update --help` and omit where unsupported); key = `ADO-<id>`. Prerequisite
     check comes from Setup (`az --version`, `az extension show --name azure-devops`,
     `az account show`).
   - **Finding eligible issues**: `az boards query --wiql "<WIQL>" --org ... --project ... -o json`
     with the preamble and table above. Note the result lists ids; fetch fields with
     `az boards work-item show --id N` for the ones needed. Include a worked example and the same
     "adapt quoting to the user's shell" caveat (WIQL contains single quotes; PowerShell/bash differ).
   - **Claiming**: steps identical to `github.md`: query; take first; `az boards work-item update --id N --assigned-to <azure_devops_user>`; wait ~5 s; re-`show` and compare `System.AssignedTo.uniqueName` to `azure_devops_user` (case-insensitive); lost race → back to step 1; then add the `in-progress` tag (and in planning mode `in-planning` instead, never both); finally `show` for title/description. Explicit sentence: tags are added only after the assignment is verified.
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
5. Update `init-factory` interview/config example/description, `docs/ARCHITECTURE.md`, `README.md`,
   `CLAUDE.md`, `plugin.json` as in the table.
6. Per the CLAUDE.md rule on optionality, re-read end to end `init-factory`, `plan-next` (step 6
   derives a blob URL "when it is a GitHub repo"; for an ADO-hosted repo the fallback "mention the
   repo path" already applies — leave as is), `implement-next`, `find-improvements` and
   `ARCHITECTURE.md` for sentences that assumed three backends or named the old skills.
7. Commit (single commit per landed phase); no version bump.

## Testing

No automated suite exists for skills (see CLAUDE.md). Verification:

- **Static**: grep that `azure-devops` spellings are identical everywhere; that no reference to
  `jira-tasks` / `github-tasks` / `todo-tasks` survives outside `legacy/` and historical `plans/`;
  that every config key named in `init-factory` appears in `ARCHITECTURE.md`; that each backend doc
  has the same top-level sections (Setup, Finding eligible, Claiming, Creating, Updating, Other
  operations) so the dispatcher's subcommands map onto all four.
- **Consolidation is a pure move (phase A)**: for jira, github and todo, every command, wait time,
  JQL/label/checkbox mapping in the old file still appears verbatim in the new doc. Then the todo
  end-to-end from CLAUDE.md (scratch repo, `backend: todo`, `claude -p "/microfactory:implement-next"`
  and the `plan-next` equivalent) — the only backend testable without external services — must behave
  as before.
- **Setup/check split**: with a CLI absent, `init-factory` asks before installing it and stops
  cleanly on "no"; a loop-run subcommand with the CLI absent stops with "run init-factory" and
  installs nothing.
- **Azure prerequisite install**: on a scratch machine, `az extension remove --name azure-devops`, then run `init-factory` and confirm it asks before installing and stops cleanly on "no".
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
- **Scope.** With the two additions this is three changes in one item: a pure refactor (A), setup/
  install (C), and a new backend (B). They are ordered so a reviewer can cut after A. Keep each
  backend doc about as long as `github.md` plus its Setup section; if the dispatcher or a doc grows
  much past that, something has gone beyond "the simplest thing".
- **Dispatcher indirection.** Workflow skills now go through two loads (`tasks`, then the backend
  doc). Only one backend doc is loaded per run, so context cost is about what it was; but a workflow
  skill that names a subcommand the dispatcher does not define will silently improvise — step 6's
  re-read must check every operation name used by `plan-next`, `implement-next`, `refine-story`,
  `breakdown-feature` and `find-improvements` against the subcommand table.
