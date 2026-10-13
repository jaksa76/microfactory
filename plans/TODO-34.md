# TODO-34: a skill that analyses the factory's own work and sends feedback for improving the microfactory

## Summary

Add one skill, `feedback`, with two subcommands that run at different moments and for different people:

- **`analyse`** — runs **unattended**, from the implementation loop once the backlog has run dry (the
  call itself is TODO-35's). Gathers evidence about how the factory has been performing, finds where
  time or tokens are wasted, which features are missing and which behaviours are inadequate, and
  writes the findings to a local file. Never prompts, never sends.
- **`send`** — runs **interactively**, offered at the end of `find-improvements` (and by hand). Tells
  the user which problems with the microfactory were found, states that **no project-specific
  information is sent**, asks whether to send them to the developers, and files the approved ones as
  GitHub issues. At most **one send per week**. Sent findings are **marked as sent, not deleted**, so
  nothing is ever reported twice.

The skill is **agent-neutral**. It says *what* evidence it needs and in what order to fall back, never
*where a particular agent keeps it* — no paths, no file formats, no tool names for reading session
records. Locating the session history, token usage and timings is the executing agent's job, which
keeps the skill valid when the agent's internals change and when it runs under an agent other than
the one it was written on (copilot and codex support are already in the backlog).

### Where feedback goes: a local findings file, then GitHub issues on approval

- **`.microfactory/feedback.md` in the target project** is the skill's memory. `analyse` appends to it;
  `send` updates it. Every finding carries a status — `drafted`, `sent <url>` or `declined` — and
  entries are **never removed**. That is the repeat guard: both subcommands read the file first and
  skip anything already in it (the same "read the destination first" rule `implement-next` step 10 and
  `refine-story`'s marker apply), and a `declined` finding is not offered again. Its header carries
  two bookkeeping lines — `Analysed through: <commit sha>` and `Last sent: <date>` — which the gates
  below read. The file is local-only: the skill appends it to `.git/info/exclude` (never
  `.gitignore`, which is tracked) and never commits it.
- **GitHub issues on `jaksa76/microfactory`** are where a plugin author reads feedback. If `gh` is
  missing or unauthenticated the findings stay `drafted` and the skill says so; it never installs or
  logs in (that is `init-factory`'s job, and `analyse` runs in a loop).

Rejected: a backend of its own for feedback (machinery for one append-only stream); filing in the
project's own backlog (the feedback is about the factory, and would land in front of people who cannot
act on it); `docs/LESSONS.md` (that queue is for judgements about the *project*).

### What `analyse` gathers

Findings are only as good as the evidence, so `analyse` first builds a small **evidence pack** — numbers
first, then a few real sessions read closely. The numbers say *where* to look, the sessions say *why*.
Everything is scoped to "since the last analysis".

1. **Throughput** — stories done (backlog `list all`, cross-checked against factory commits
   `[<KEY>] ...` and `Add plan for <KEY>`), plans written, items stuck in `Planning` / `In Progress`,
   items blocked, iterations by mode and how each ended (done, blocked, abandoned).
2. **Duration** — per iteration and on average, for `implement-next` and `plan-next` separately. An
   iteration is one invocation of the skill, measured from the invocation to the end of its work, so
   idle gaps between loop ticks are not counted. Where it is available, also the **share by phase**
   (claim, analysis, implement, test, verify, review, push) — the single most useful number for
   finding waste.
3. **Tokens** — per iteration and on average: total, output, and the share spent in subagents (the
   cold review is the usual suspect); cached and fresh input kept apart, since they cost differently.
4. **Process quality** — claim races lost, failed task-backend commands, test runs per iteration, cold
   review findings raised / acted on / rejected, plans corrected by the iteration that implemented them,
   plans a human edited before approving, stories that needed refinement, and the typical size of a
   story (files and lines changed, in buckets). Also the time items spend in `Awaiting Plan Review`,
   which says whether the loop is idling on humans rather than on its own inefficiency.
5. **Factory and environment** — see *What travels with the feedback*.
6. **Sample sessions, read closely** — for each mode with at least two iterations, the **longest** (by
   duration, ties by tokens) and the one **nearest the median**. Read their structure — sequence of
   steps, repeated test runs, retries, failed commands, review rounds, re-reads of the same files, time
   between steps — to say where time and tokens went and whether the long one differs from the typical
   one in a way a skill instruction could prevent.
7. **What the repo already recorded** — a plan file touched again by the commit that implemented it
   (the plan was wrong), fix-up commits after an implementation (a defect the iteration's own review
   missed), refinement threads that needed answers the skill could have asked earlier, `docs/LESSONS.md`
   entries that point at a skill's wording.
8. **This session's context**, when there is any — reviewer findings accepted and rejected, plan
   corrections, blocked iterations, things the user corrected. It exists nowhere else.

**How to obtain it is the agent's problem.** The skill states the need — per-iteration timings and
token usage, the steps taken, and subagent usage — and a fallback ladder, nothing more:

1. whatever record the agent environment keeps of its past sessions, if it keeps one the agent can
   read;
2. otherwise commit and backlog timestamps, for coarse durations only;
3. otherwise report the figure as **unavailable**.

The report **says which rung each number came from** and never invents one. Two rules hold on every
rung:

- **Aggregate with tooling, do not read session records into context.** A long session is megabytes;
  reading it to count tokens would make the analysis the largest waste in the project. Only the two
  sample sessions per mode are inspected, and those through a condensed view (steps, tool names,
  outcomes, timestamps), not full text.
- **The records stay local.** They are full of the project's code and conversation; only conclusions
  and aggregates are ever drafted.

### What travels with the feedback

Beyond the findings themselves, each send carries one short **context block**, so the developers can
tell what a finding was observed *on* and weigh it:

| Group | Contents |
|---|---|
| Plugin | microfactory version — from the plugin's own manifest, found relative to the skill. When the installed version differs from the one the running session loaded (a long loop can lag an update), both |
| Agent | the agent product and version, and the model(s) used for the main session and for subagents such as the reviewer — whatever the agent can report about itself |
| Environment | operating-system family and shell family; whether subagents were available; whether the work ran interactively or headless / in a loop, and the loop intervals |
| Backend | which backend, and the version of its CLI (`acli`, `gh` or `az`) |
| Configuration | `plan_by_default`, `auto_plan`, `feature_branches`, `deep_review` |
| Volume | the number of stories done, iterations by mode, the evidence window as a length of time |
| Per-mode averages | duration, tokens, phase shares, test runs, review findings raised/acted on/rejected, plan corrections, claim races, backend command failures, story-size buckets, time awaiting plan review |
| Data quality | which rung of the fallback ladder each number came from |

A **fixed whitelist**, not a filter: the block contains exactly these fields and nothing else. Config
values that identify a person or a project — `project`, site and organisation names, account ids,
usernames — are never read into the pack at all. Deliberately **left out**: language, framework,
repository size or name, project type, file names, story titles or keys, exact timestamps (the window is
a length, not dates), and anything quoted from a session record. Language or project type would help
the developers, but they are the project, and the promise to the user is simpler and safer without
exceptions.

### What counts as a finding

About **the factory** — a skill, a backend doc, a setting, the architecture — never about the project.
It is one of:

- **Waste**: time or tokens spent where a skill instruction could have saved them, with the measurement
  that shows it ("the cold review was 40% of each implementation's tokens and changed nothing in four
  of five").
- **Missing feature**: a thing the user or an iteration had to work around by hand.
- **Inadequate behaviour**: a skill that misled, an instruction that could not be followed in this
  environment, a claim-protocol hole, a backend command that failed, a story size the breakdown got
  wrong.

and it is a **pattern** (at least two iterations or issues) or a **single severe failure** that stopped
work. A one-off ("this plan was wrong") is recorded nowhere — the bar `implement-next` step 10 sets.
Capped at five per analysis, ranked by measured cost. Each finding has: **Title**, **Where** (skill and
step, backend doc, or setting), **Evidence** (the measurement or observed behaviour, project-neutral,
with its `n`), **Proposed change** (concrete enough to become a plan), **Severity**.

### What `send` says and does

1. **Weekly limit first.** If `Last sent` in the file is less than seven days ago, say when the next
   send is possible and stop. One send is one batch of findings; a hard cap on a stream that reaches
   real people. It is not overridable from the skill — a user who truly needs to report something
   sooner can open an issue by hand, which the message says. The check comes before any question, so a
   user inside the limit is never asked anything.
2. If there are no `drafted` findings, say so and stop.
3. Print the `drafted` findings, ranked, at most five — the rest wait for the next week — together
   with the exact **context block** that will accompany them, and then state, in these words or close
   to them: *"Sending these will open issues on the microfactory repository. They contain no
   project-specific information — no code, file names, story titles or keys, names, URLs or
   credentials; only the findings above and the context block shown. Send them?"*
4. The user picks which to send (multi-select, so any single finding can be declined). Declined ones are
   marked `declined` and not offered again.
5. Detect `gh` and its login — detect only; on failure leave everything `drafted` and explain the manual
   route. For each chosen finding, search the upstream issues first: when an open issue already covers
   it, add the new evidence as a comment on it; otherwise open an issue. Each finding is its own issue
   so the developers can track and close them separately.
6. Record `sent <url>` on each, set `Last sent` to today, report the URLs.

The skill strips project specifics when it *drafts*, so the file is already clean, but the user reading
the exact text is the safety net.

### Triggers

**`analyse` — from the implementation loop when no work is left, guarded.** The call belongs to
TODO-35 (the idle path of `implement-next`), which should be a one-liner: `feedback analyse if-due`.
Plain `analyse` runs whenever asked (a user by hand is never refused); `analyse if-due` is the
unattended form and applies two gates. The gates live in the skill rather than in the caller so
there is one place that decides, and TODO-35 cannot get them wrong:

1. **Enough new work** — at least **five** stories finished since `Analysed through`. A handful of
   iterations is not a pattern, so below that there is nothing worth analysing.
2. **No backlog of findings** — no finding is still `drafted`. Findings nobody has dealt with are a
   reason to wait, not to produce more: they only pile up until a send happens.

When a gate fails the skill says nothing and ends, so an idle loop stays quiet. After every analysis,
including one that finds nothing, `Analysed through` moves to the current HEAD — an empty result must
not make the gate fire again on the same stories. Five is a constant of the skill, not a config key,
until someone needs a different number.

**`send` — at the end of `find-improvements`, interactively.** `find-improvements` is the skill a user
runs from their own session and already asks them questions, so someone is there. After its report,
if `drafted` findings exist and the weekly limit allows, it hands over to `feedback send`. It does the
weekly check itself first, so inside the limit it adds no line at all rather than nagging. If
`find-improvements` ever runs unattended (TODO-32 plans a loop), the offer is skipped — it asks a
question — and TODO-32 should say so. Nothing is added to `start`, `start-planning` or
`breakdown-feature`. The user can always run `/microfactory:feedback send` directly.

Rejected as before: a plugin hook (the plugin is pure markdown, and a hook runs in every session,
factory or not) and asking from inside the loop (it blocks every later iteration on a question).

### Invocation

`/microfactory:feedback analyse [if-due]` and `/microfactory:feedback send`. With no argument the skill
runs `analyse` and then `send` — for a user who is present and wants the whole round at once.

## Files to Change

- **`skills/feedback/SKILL.md`** (new) — the skill, per *Implementation Steps*. `description` names both
  subcommands and when each applies: analyse the factory's performance (unattended, when the backlog
  is empty, or by hand); send the findings to the microfactory developers (interactively, at most
  weekly).
- **`skills/find-improvements/SKILL.md`** — a closing step after *Report*: if `.microfactory/feedback.md`
  holds `drafted` findings and its `Last sent` date is a week or more old (or absent), load `feedback`
  and run `send`; otherwise say nothing. Update the opening line of the file (the flow summary) and
  the sentence "This is the only skill that creates tasks" if it now reads wrong.
- **`docs/ARCHITECTURE.md`** — `feedback/` in the layout; a *Feedback* section after *Lessons*
  covering the destination, the two subcommands, the agent-neutral evidence pack, the context block,
  both triggers and the three guards (status-not-deletion, the two `analyse` gates, the weekly send
  limit); and a pointer from *Lessons*, whose opening says feedback signals "are discarded when the
  session ends", so the two sections do not contradict each other.
- **`CLAUDE.md`** — `feedback/` in the layout block (it mirrors ARCHITECTURE's).
- **`README.md`** — a short paragraph next to `find-improvements`: what the two subcommands do, that
  `send` asks first, sends no project-specific information and at most once a week.
- **`TODO.md`** — on completion, TODO-34 to `[x]`; rewrite TODO-35's note to say it calls
  `feedback analyse if-due` and nothing else; add a line to TODO-32's note that its unattended sweep
  must skip the feedback offer.

Not changed: `implement-next` (TODO-35 owns the idle call), `start`, `start-planning`,
`breakdown-feature`, `init-factory` (no new config key; the upstream repo and the thresholds are
constants of the skill), `tasks` (uses only `list` and `comments`).

## Implementation Steps

1. Create `skills/feedback/SKILL.md`, in the voice and density of the neighbouring skills (rationale
   stated once; name what is needed, not where one agent keeps it):
   - **Opening** — the two subcommands, which one may run unattended (`analyse`: never asks, never
     sends) and which may not (`send`: interactive only), and the `if-due` form.
   - **Shared: read configuration and the findings file** — read `.microfactory/config.yaml` (stop with
     the `init-factory` pointer if absent); read `.microfactory/feedback.md`, creating it with a header
     (`Analysed through`, `Last sent`) if missing and adding it to `.git/info/exclude`. Every entry in it
     is *already said*.
   - **`analyse`** — (a) with `if-due`, apply the two gates and end silently if either fails; (b) if
     nothing was done since `Analysed through`, say "nothing new" and stop; (c) build the evidence pack
     — need and fallback ladder, aggregate-with-tooling, records stay local; (d) apply the finding bar
     and the cap of five; (e) drop what the file already holds; (f) write each finding with
     `status: drafted` and the context block; (g) advance `Analysed through` to HEAD even when nothing
     was found; (h) report in two or three lines, naming which rung each figure came from.
   - **`send`** — the six steps above, in order.
   - **Rules** — never deletes an entry; never commits the file; never edits skills or the project;
     the context block is the whitelist and nothing else is added; privacy first: project-neutral
     evidence only.
2. Add the closing step to `find-improvements`.
3. Update ARCHITECTURE, CLAUDE.md, README, and the TODO-32 / TODO-35 notes as listed.
4. Re-read each touched file end to end (CLAUDE.md convention: a new capability invalidates sentences
   that never name it) — especially ARCHITECTURE's *Lessons* and *Getting work into the backlog*, which
   count the skills that feed the backlog, and `find-improvements`' opening.

## Testing

No automated suite exists for skills; verify as `CLAUDE.md` describes.

1. **Analyse on real history.** Run `claude -p "/microfactory:feedback analyse"` in this repo, which has
   real sessions and factory commits. Check against a manual count: stories done, iteration count, and
   the duration and token totals for one session; check the longest and median sessions it picked are
   the right ones, and that the report names the source of each figure. Confirm the run did not read
   session records into context wholesale (its own token use stays small) and that
   `.microfactory/feedback.md` exists, is listed in `.git/info/exclude`, and is absent from `git status`.
2. **Agent-neutrality read.** Read the finished `SKILL.md` end to end and confirm it names no path,
   file format or tool for reading session history.
3. **Gates.** `analyse if-due` with four finished stories since the marker: silent, no file change.
   With five: runs. Run it again straight after: silent (marker advanced). With one `drafted`
   finding present and five new stories: silent.
4. **Fallback.** In a scratch repo with no readable session history, the analysis falls back to commit
   timestamps, reports tokens as unavailable, and fabricates no numbers.
5. **Context block.** The drafted block contains the whitelist and nothing else: plugin version, agent
   and model(s), OS and shell family, backend CLI version, the four settings, aggregates. Put a fake
   `jira_site` / `github_username` / `project` in a scratch config and confirm none appears anywhere in
   the file or the issue text.
6. **Send without `gh`.** Findings stay `drafted`, nothing is installed, `Last sent` is **not**
   updated, the message explains the manual route.
7. **Send with `gh`** (live smoke, needs auth; point it at a throwaway repo rather than filing noise
   upstream): the no-project-information statement and the exact context block are shown before the
   question; one chosen finding becomes an issue, one declined stays as `declined`; `Last sent` is
   today; an immediate second `send` stops on the weekly limit without asking anything; after faking
   `Last sent` back eight days, `send` offers nothing already sent or declined; an `analyse` that finds
   the same problem again does not re-add it.
8. **Privacy read.** Read the issue text and check no path, key, title or name from the project survived.
9. **Offer.** Finish a `find-improvements` run with a `drafted` finding and `Last sent` old: the send
   step follows the report. With `Last sent` three days ago, or no findings: nothing is said.

## Risks / Edge Cases

- **Session history is the agent's, not the factory's.** Leaving retrieval to the agent buys
  forward and sideways compatibility and costs reproducibility: two agents may find different
  amounts of evidence. The fallback ladder and the "which rung" disclosure make that visible instead
  of silent, but the headline numbers (duration, tokens) are the thing most likely to be absent or
  coarse. A run with no readable history still produces findings from git and the backlog.
- **Session records hold the project's code and conversation.** The analysis reads them locally and only
  conclusions leave, but one careless quote would leak. The drafting rule, the whitelist context block
  and the user reading the exact text are three separate barriers; `analyse` itself never sends, so an
  unattended loop can never publish.
- **The analysis can cost more than it saves.** Aggregating with tooling, sampling two sessions per
  mode, and the two gates are what prevent that; the skill must say so.
- **Small samples.** An average over five iterations is weak. Findings show their `n`, and the pattern
  bar will often yield an empty result — intended, since one iteration is not a mandate to set policy.
- **The gates can starve the analysis.** If a user never sends, findings stay `drafted` and the second
  gate blocks any further analysis forever. That is deliberate backpressure, but it should be visible:
  the `find-improvements` offer is the nudge, and a manual `analyse` always works.
- **The weekly limit counts sends, not findings.** One batch of up to five issues per week. A severe
  finding discovered after a send waits up to a week; a manual issue is the escape hatch.
- **`Last sent` is per working copy.** A second clone starts with no marker and could send again within
  the week, and may re-propose findings. The upstream search before opening an issue stops duplicates;
  the weekly cap is therefore a courtesy per clone, not a global guarantee.
- **Median and "typical" session** differ on skewed data; the plan picks the one nearest the median and
  labels it so.
- **The todo backend has no timestamps or authors**, so "done since the last analysis" falls back on git
  history of the TODO file and factory commits; the report says when it used the fallback.
- **The context block is fixed** — new fields need a skill change, on purpose: every addition widens
  what leaves the user's machine.
- **Hardcoded upstream repo and thresholds.** If the plugin moves, or five stories / one week prove
  wrong, one constant changes; a config key is not worth it until someone needs a different value.
- **Depends on nothing; TODO-35 depends on this**, and TODO-32 needs a one-line note about the offer.
