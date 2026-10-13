# TODO-34: a skill that reviews the work done in a session and sends feedback for improving the microfactory

## Summary

Add one skill, `feedback`, that looks back over what the factory did and turns the **recurring failures**
into concrete proposals for the skills themselves — what misled the planner, what the reviewer kept
catching, where a story was the wrong size, which backend command broke. The item says the main thing
to settle is **where the proposals go**. No refinement thread exists and the item carries no
`needs-refinement` tag, so these decisions are mine.

### Where feedback goes: a local draft first, GitHub issues on approval

Two destinations were on offer — issues in the microfactory repo, or a local report file. Neither is
enough alone, so the skill uses both in sequence, and each does the job it is good at:

- **Local draft, always.** Proposals are written to `.microfactory/feedback.md` in the *target* project.
  It is the skill's memory: what was already drafted or sent, so a second run never says it again — the
  same "read the destination first" rule `implement-next` step 10 and `refine-story`'s marker already
  apply. It is local-only, so the skill appends it to `.git/info/exclude` (never `.gitignore`, which is
  tracked) and never commits it; `implement-next` and `plan-next` commit by explicit path, but a human's
  `git add -A` must not publish it either.
- **GitHub issues in `jaksa76/microfactory`, only with approval.** That is where a plugin author reads
  feedback. Filing is outward-facing and the text derives from someone's private project, so the user
  sees each draft and picks what goes. It is sent with `gh issue create --repo jaksa76/microfactory`.
  If `gh` is missing or unauthenticated the drafts stay in the file and the skill says so — it never
  installs or logs in (that is `init-factory`'s job, and this skill may run unattended).

Rejected: a backend of its own for feedback (a second `tasks` configuration for one append-only stream is
machinery with no second use); filing in the project's own backlog (the feedback is about the factory,
not the product, and would land in front of people who cannot act on it); writing into the project's
`docs/LESSONS.md` (that queue is for judgements about the *project* — facts and judgements about the
factory's own skills belong upstream, where the skills live).

### Two modes, because TODO-35 will call this unattended

- `/microfactory:feedback` — interactive. Analyse, write drafts, show them, **ask which to send**, send.
- `/microfactory:feedback draft` — never prompts and never sends. Analyse and write drafts, then stop.
  This is the form an idle `implement-next` iteration can run inside a `/loop` (TODO-35); it is the only
  difference between the two modes, so TODO-35 needs no change to this skill.

The "do not repeat" guard TODO-35 asks for comes along for free: drafts already in the file are skipped,
and the file header records `Reviewed through: <commit sha>` so a later run only reads history newer
than that.

### What it reads — and why this is not "iteration notes"

`implement-next` no longer writes iteration notes to issues (step 10 routes lessons into project files),
so the evidence is spread over four places, each of which sees something the others cannot:

1. **This session's own context** — reviewer findings (accepted *and* rejected), plan corrections made
   while implementing, blocked iterations, anything the user corrected. This exists nowhere else and is
   lost when the session ends, so it is the most valuable source and the reason to run the skill at the
   end of a session.
2. **Git history since the last review** — factory commits (`[<KEY>] ...`, `Add plan for <KEY>`). A plan
   file touched again by the commit that implemented it is a plan that was wrong; a fix-up commit after
   an implementation is a defect the iteration's own review missed.
3. **The backlog** (`tasks list all`, `tasks comments <key>` on items touched since the last review) —
   refinement threads that needed answers the skill could have asked earlier, `blocked` comments,
   items stuck in `Planning` / `In Progress`, stories split or re-opened.
4. **`docs/LESSONS.md` and `CLAUDE.md`** — judgements and facts earlier iterations recorded; a lesson
   that points at a skill's wording is a feedback candidate.

### What counts as a proposal

A feedback item is about **the factory** (a skill, a backend doc, a config setting, the architecture),
never about the project. It qualifies when it is a pattern — seen in at least two issues or iterations
— or a single failure severe enough to stop work (a backend command that errors, a skill instruction
that cannot be followed on this OS, a claim-protocol hole). A one-off ("this plan was wrong") is
recorded nowhere, the same bar as `implement-next` step 10. Capped at five, ranked by how much
iteration cost the problem caused.

Each proposal: **Title**, **Where** (skill and step, backend doc, or setting), **What happened**
(project-neutral evidence: counts, the step that misled, the error text), **Proposed change** (the
wording or rule to change — concrete enough to become a plan), **Severity**.

### Privacy

The proposals leave the machine. The skill must strip anything project-specific before drafting: no
source code, file paths from the project, issue titles or keys, names, URLs, secrets. Evidence is stated
in terms of the factory ("a `needs-plan` story's plan named a file that did not exist; the implementer
found out at step 6"). The user reads each draft before it is sent; that review is the safety net, not
the stripping.

## Files to Change

- **`skills/feedback/SKILL.md`** (new) — the skill, described in *Implementation Steps*. Frontmatter
  `description`: reviews what the factory did in this session and the history since the last review,
  drafts feedback on the microfactory skills, and with the user's approval files it as issues on the
  microfactory repo; use when a session's work is done or the user wants to report factory problems.
- **`docs/ARCHITECTURE.md`** — add `feedback/` to the repository layout, and a short *Feedback* section
  after *Lessons* covering the destination decision (local draft → upstream issue on approval), the two
  modes, and why the draft file is the repeat guard. Update the `[Git]` / sources prose only if it now
  reads wrong.
- **`CLAUDE.md`** — add the `feedback/` line to the repository layout block (it mirrors ARCHITECTURE's).
- **`README.md`** — one sentence next to the `find-improvements` paragraph: `/microfactory:feedback`
  drafts feedback on the factory itself and sends it only after you pick what to send.
- **`TODO.md`** — on completion, flip TODO-34 to `[x]`; leave TODO-35 as is (it depends on this skill
  and needs no change to it).

Not changed: `implement-next` (TODO-35 owns calling this from the idle path), `init-factory` (no new
config key — the upstream repo is a constant of the plugin, and the draft file needs no setting),
`tasks` (no new subcommand; it only uses `list`, `comments`).

## Implementation Steps

1. Create `skills/feedback/SKILL.md` with this structure, matching the surrounding skills' voice and
   density (terse, rationale stated once, no brittle one-liners):
   - **Title and one-line flow**: `gather evidence → find patterns → draft → (approve → send)`.
   - **Modes**: argument `draft` = never `AskUserQuestion`, never send; no argument = interactive.
   - **1. Read configuration** — `.microfactory/config.yaml` (stop with the `init-factory` pointer if
     absent); load `tasks` for `list` and `comments` only. Read `.microfactory/feedback.md` if it
     exists: its entries are *already said*, and `Reviewed through: <sha>` bounds the history to read.
     If the file does not exist yet, create it with a header and add it to `.git/info/exclude`.
   - **2. Gather evidence** — the four sources above, in that order, each scoped to "since the last
     review". Name the signals (plan edited by its implementing commit, fix-up commits, rejected
     review findings, blocked comments, items stuck in a status, stories that needed splitting).
   - **3. Find the patterns** — the proposal bar (pattern or severe single failure; about the factory,
     not the project), the cap of five, ranking, and the instruction to drop anything already in the
     file or in the open issues of the upstream repo (step 5 check).
   - **4. Draft** — write each proposal to `.microfactory/feedback.md` under the format above, with a
     `status: drafted` line, and update `Reviewed through` to the current HEAD. State the privacy
     rule here: project-neutral evidence only.
   - **5. Send (interactive only)** — print the drafts; ask which to send (`AskUserQuestion`,
     multi-select). For each chosen: check `gh` is installed and authenticated (detect only — never
     install or log in); search `gh issue list --repo jaksa76/microfactory --state all --search <title
     keywords>`; if an open issue already covers it, add the new evidence as a comment on it, else
     `gh issue create --repo jaksa76/microfactory --title ... --body ...`. Record `status: sent <url>`
     in the file. If `gh` is unusable, leave the drafts and say how to send them manually.
   - **6. Report** — what was drafted, what was sent (URLs), what was skipped as already said, and the
     path to the file. Say plainly when the review found nothing worth reporting; that is a result, and
     the `Reviewed through` marker still advances.
   - **Rules block** (short): never edits skills or the project; never commits the draft file; never
     sends in `draft` mode; never sends without approval; one review per invocation.
2. Add `feedback/` to the layout blocks in `docs/ARCHITECTURE.md` and `CLAUDE.md`, and the *Feedback*
   section to ARCHITECTURE as described. Keep it to what the other sections do: the decision, the
   rejected alternatives in a sentence, the cost.
3. Add the README sentence.
4. Re-read each touched file end to end (CLAUDE.md convention) — in particular ARCHITECTURE's
   *Lessons* section, which says feedback signals "are discarded when the session ends"; it should now
   point at this skill as the one place that reads across iterations, so the two sections do not
   contradict each other.

## Testing

No automated suite exists for skills; verify as `CLAUDE.md` describes, with a scratch repo.

1. **Draft mode, todo backend.** Scratch git repo with a `TODO.md` of three done items, a
   `.microfactory/config.yaml` (`backend: todo`), two `[<KEY>] ...` commits where the second edits a
   `plans/<KEY>.md` the first added, and a `blocked` note on one item. Run
   `claude -p "/microfactory:feedback draft"` and check: `.microfactory/feedback.md` exists with
   `Reviewed through` equal to HEAD; it holds a plan-was-wrong proposal with project-neutral wording;
   `.git/info/exclude` lists the file; `git status` does not show it; nothing was sent and no prompt
   appeared.
2. **Repeat guard.** Run the same command again with no new commits: the report says nothing new and
   the file gains no duplicate entry.
3. **Interactive send without `gh`.** Run the bare command where `gh` is absent: drafts are kept, no
   install is attempted, the message says how to send manually.
4. **Interactive send with `gh`** (live smoke, needs auth): pick one draft; confirm one issue appears
   in the upstream repo with no project-specific text, the file records `status: sent <url>`, and a
   second run finds it and comments instead of opening a duplicate. Use a throwaway repo by temporarily
   pointing the skill at it rather than filing noise upstream.
5. **Privacy read.** Read the drafts from run 1 and check that no path, issue key or title from the
   scratch project survived.

## Risks / Edge Cases

- **Outward-facing writes.** Filing publishes text derived from private work, possibly indexed even if
  deleted. Mitigated twice: stripping, then per-draft user approval. `draft` mode cannot send at all,
  so an unattended loop can never publish.
- **Session context is thin when run cold.** In a fresh session the first source is empty and the skill
  works from git and the backlog only. That is acceptable and should be said in the report ("no session
  context; evidence is from history") so the user knows to run it at the end of a working session.
- **Weak signal from the todo backend.** It has no authors or timestamps, so "items touched since the
  last review" falls back on git history of the TODO file. The skill should say it used that fallback.
- **Pattern bar vs. one data point.** A session with one iteration cannot show a pattern; the bar will
  often produce an empty report. That is intended — it matches `implement-next`'s "one iteration is
  not a mandate" — and the severe-single-failure exception covers outright breakage.
- **`.git/info/exclude` is per-clone.** A second clone does not inherit it, and a fresh clone has no
  draft file anyway; the only exposure is someone running the skill there, which re-adds the line.
- **Hardcoded upstream repo.** If the plugin moves, the skill's constant is stale. Cheap to fix in one
  place; promoting it to a config key is not worth it until a fork wants to point elsewhere.
- **Depends on nothing; is depended on by TODO-35**, which should call `feedback draft` (not the bare
  form) from its idle path.
