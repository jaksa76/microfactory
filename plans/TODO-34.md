# TODO-34: a skill that analyses the factory's own work and sends feedback for improving the microfactory

## Summary

Add one skill, `feedback`, with two subcommands that run at different moments and for different people:

- **`analyse`** — runs **unattended**, when the backlog is empty. Gathers evidence about how the factory
  has been performing, finds where time or tokens are wasted, which features are missing and which
  behaviours are inadequate, and writes the findings to a local file. Never prompts, never sends.
- **`send`** — runs **interactively**, when a user is present and doing something else. Tells the user
  which problems with the microfactory were found, states that **no project-specific information is
  sent**, asks whether to send them to the developers, and files the approved ones as GitHub issues.
  Sent findings are **marked as sent, not deleted**, so nothing is ever reported twice.

The item left open where feedback goes and when to trigger it. The destination and the trigger are
decided below; the trigger in particular is a **proposal for the owner to confirm** (see *Triggers*).

### Where feedback goes: a local findings file, then GitHub issues on approval

- **`.microfactory/feedback.md` in the target project** is the skill's memory. `analyse` appends to it;
  `send` updates it. Every finding carries a status — `drafted`, `sent <url>` or `declined` — and
  entries are **never removed**. That is the whole repeat guard: both subcommands read the file first
  and skip anything already in it (the same "read the destination first" rule `implement-next` step 10
  and `refine-story`'s marker apply), and a `declined` finding is not offered again. The file is
  local-only: the skill appends it to `.git/info/exclude` (never `.gitignore`, which is tracked) and
  never commits it.
- **GitHub issues on `jaksa76/microfactory`** are where a plugin author reads feedback, sent with
  `gh issue create --repo jaksa76/microfactory`. If `gh` is missing or unauthenticated the findings stay
  `drafted` and the skill says so; it never installs or logs in (that is `init-factory`'s job, and
  `analyse` runs in a loop).

Rejected: a backend of its own for feedback (machinery for one append-only stream); filing in the
project's own backlog (the feedback is about the factory, and would land in front of people who cannot
act on it); `docs/LESSONS.md` (that queue is for judgements about the *project*).

### What `analyse` gathers

Findings are only as good as the evidence, so `analyse` first builds a small **evidence pack** — numbers
first, then a few real sessions read closely. Order matters: the numbers say *where* to look, the
sessions say *why*.

1. **Throughput** — stories done since the last analysis (backlog `list all`, cross-checked against
   factory commits `[<KEY>] ...` and `Add plan for <KEY>`), plans written, items stuck in `Planning` /
   `In Progress`, items blocked.
2. **Duration** — per iteration and on average, for `implement-next` and `plan-next` separately. An
   iteration is one invocation of the skill, measured from the invocation to the last message before
   the next one, so idle gaps between loop ticks are not counted.
3. **Tokens** — per iteration and on average: total, output, and the share spent in subagents (the
   cold review is the usual suspect). Cache-read and fresh input are kept apart, since they cost
   differently.
4. **Configuration** — only the keys that change factory behaviour: `backend`, `plan_by_default`,
   `auto_plan`, `feature_branches`, `deep_review`, and the two intervals. A fixed whitelist: `project`,
   `jira_site`, account ids, usernames and organisation names identify the user or the project and are
   never read into the pack.
5. **Sample sessions, read closely** — for each mode with at least two iterations, the **longest**
   (by duration, ties by tokens) and the one **nearest the median**. Read their structure — the
   sequence of steps, repeated test runs, retries, failed commands, review rounds, re-reads of the same
   files, time between steps — to say where the time and tokens went and whether the long one differs
   from the average one in a way a skill instruction could prevent.
6. **What the repo already recorded** — a plan file touched again by the commit that implemented it
   (the plan was wrong), fix-up commits after an implementation (a defect the iteration's own review
   missed), refinement threads that needed answers the skill could have asked earlier, `docs/LESSONS.md`
   entries that point at a skill's wording.
7. **This session's context**, when there is any — reviewer findings accepted and rejected, plan
   corrections, blocked iterations, things the user corrected. It exists nowhere else.

**Where durations and tokens come from.** Claude Code writes every session to
`~/.claude/projects/<project path, encoded>/<session-id>.jsonl`, one JSON object per line with a
timestamp, the token `usage` of each model reply, and `<command-name>` markers for slash-command
invocations; subagent transcripts sit under `<session-id>/subagents/`. Iterations are found by the
`/microfactory:implement-next` and `/microfactory:plan-next` command markers. Two consequences:

- **Aggregate with the shell, do not read the transcripts into context** — `jq`, Python or PowerShell,
  whichever exists. A single long session is megabytes; reading it to count tokens would make the
  analysis the largest waste in the project. Only the two sample sessions per mode are inspected, and
  even those through a condensed view (step names, tool names, exit codes, timestamps), not full text.
- **The format is Claude Code's, not the factory's, and is not a documented interface.** The skill
  treats it as best effort: if the directory is missing (another machine, a cleaned profile, a
  container) or a field has moved, it falls back to commit timestamps for coarse durations, reports
  tokens as unavailable, and **says so in the report** rather than inventing numbers.

### What counts as a finding

About **the factory** — a skill, a backend doc, a setting, the architecture — never about the project.
It qualifies when it is one of:

- **Waste**: time or tokens spent where a skill instruction could have saved them, with the measurement
  that shows it ("the cold review was 40% of each implementation's tokens and changed nothing in four
  of five").
- **Missing feature**: a thing the user or an iteration had to work around by hand.
- **Inadequate behaviour**: a skill that misled, an instruction that could not be followed on this OS,
  a claim-protocol hole, a backend command that failed, a story size the breakdown got wrong.

and it is a **pattern** (at least two iterations or issues) or a **single severe failure** that stopped
work. A one-off ("this plan was wrong") is recorded nowhere — the bar `implement-next` step 10 sets.
Capped at five per analysis, ranked by measured cost. Each finding has: **Title**, **Where** (skill and
step, backend doc, or setting), **Evidence** (the measurement or observed behaviour, project-neutral),
**Proposed change** (concrete enough to become a plan), **Severity**.

### What `send` sends — and says

`send` prints the `drafted` findings and then states, in these words or close to them: *"Sending these
will open issues on the microfactory repository. They contain no project-specific information — no
code, file names, story titles or keys, names, URLs or credentials; only the findings above and a short
block of aggregate numbers (how many stories, average duration and tokens, and which of the settings
`backend`, `plan_by_default`, `auto_plan`, `feature_branches`, `deep_review` you use). Send them?"* The
user picks which to send (multi-select, so a single finding can be declined). The exact issue text is
shown before sending. Each chosen finding is searched for upstream first (`gh issue list --repo
jaksa76/microfactory --state all --search ...`): if an open issue covers it the new evidence is added as
a comment, otherwise an issue is created. The file then records `sent <url>`; refused findings record
`declined`.

The skill strips project specifics when it *drafts*, so the file itself is already clean, but the user's
read of the exact text is the real safety net — the transcripts the evidence came from are full of
project content, so the drafting rule is the one that matters most and is stated first in the skill.

### Triggers

**`analyse` — recommended: the idle iteration of `implement-next`.** This is what you asked for (run
when the backlog is empty) and is exactly TODO-35, which should call `feedback analyse` — never
`send`, which would block a loop on a question nobody is there to answer. Two guards keep an idle
factory from re-analysing every 20 minutes: `analyse` records `Analysed through: <commit sha>` and
stops with "nothing new" when no story has been done since; and TODO-35's call should additionally
require a minimum of new finished stories (suggest five — a handful of iterations is not a pattern).
The skill does not own the threshold; the caller does, so a user running `analyse` by hand is never
refused. Users can always run it manually too.

**`send` — the proposals, for you to choose between:**

1. **Mention it in the skills a user runs from their own session** *(recommended)* — `start`,
   `start-planning`, `find-improvements`, `breakdown-feature`. When `.microfactory/feedback.md` holds
   `drafted` findings, the skill ends with one line: *"N microfactory findings are waiting — run
   `/microfactory:feedback send` to review them."* It is a mention, not a prompt, so it never blocks
   the work the user came to do, and it stops once the findings are sent or declined. Cost: one
   sentence in each of four skills, no machinery.
2. **A push notification from `analyse`** when it writes new findings. It reaches a user who is away
   from the terminal, which is the "doing something else" case literally, but needs the notification
   tool configured, and is noise for users who never send. Reasonable as an add-on to 1, not instead
   of it.
3. **A plugin hook** (e.g. on session start) — rejected: the plugin is pure markdown with no scripts
   (CLAUDE.md), and a hook would run in every session, factory or not.
4. **Ask from inside the loop** — rejected: it blocks every later iteration on a question, and the
   loop cannot tell whether anyone is watching.
5. **Only the manual command** — the floor, and what the plan delivers if 1 is declined: findings
   accumulate in the file and nobody learns they exist. Not recommended on its own.

The plan implements **1**, and mentions 2 only as a possible follow-up. If you prefer a different set
of skills for the mention, that is a one-line change per skill.

### Invocation

`/microfactory:feedback analyse` and `/microfactory:feedback send`. With no argument the skill runs
`analyse` and then `send` — the form for a user who is present and wants the whole loop at once; with
nothing new to analyse it goes straight to `send`.

## Files to Change

- **`skills/feedback/SKILL.md`** (new) — the skill, per *Implementation Steps*. `description` names both
  subcommands and when each applies: analyse the factory's performance when the backlog is empty (also
  by hand); send the findings to the microfactory developers when the user is present.
- **`skills/start/SKILL.md`**, **`skills/start-planning/SKILL.md`**, **`skills/find-improvements/SKILL.md`**,
  **`skills/breakdown-feature/SKILL.md`** — one closing sentence each: if `.microfactory/feedback.md`
  holds `drafted` findings, mention that `/microfactory:feedback send` is waiting. Read each file end to
  end first and place the sentence where the user will see it before a loop takes over their session.
- **`docs/ARCHITECTURE.md`** — `feedback/` in the layout; a *Feedback* section after *Lessons*
  covering the destination, the two subcommands, the evidence pack and its transcript dependency, the
  trigger decision and the status-not-deletion repeat guard; and a pointer from *Lessons*, whose
  opening says feedback signals "are discarded when the session ends", so the two sections do not
  contradict each other. Add the new line to the claim that only `implement-next` writes lessons if
  that sentence now reads wrong.
- **`CLAUDE.md`** — `feedback/` in the layout block (it mirrors ARCHITECTURE's).
- **`README.md`** — a short paragraph next to `find-improvements`: what the two subcommands do, that
  `send` asks first and sends no project-specific information.
- **`TODO.md`** — on completion, TODO-34 to `[x]`; rewrite TODO-35's note to say it calls
  `feedback analyse` and applies the minimum-stories gate.

Not changed: `implement-next` (TODO-35 owns the idle call), `init-factory` (no new config key; the
upstream repo is a constant of the plugin), `tasks` (uses only `list` and `comments`).

## Implementation Steps

1. Create `skills/feedback/SKILL.md`, in the voice and density of the neighbouring skills (rationale
   stated once, no brittle one-liners — name the tool to use, not the exact command, except the `gh`
   calls which are part of the contract):
   - **Opening** — what the two subcommands are, and which one may run unattended (`analyse`: never
     `AskUserQuestion`, never sends) and which may not (`send`: interactive only).
   - **Shared: read configuration and the findings file** — read `.microfactory/config.yaml` (stop with
     the `init-factory` pointer if absent); read `.microfactory/feedback.md`, creating it with a header
     if missing and adding it to `.git/info/exclude`. Every entry in it is *already said*.
   - **`analyse`** — (a) if nothing was done since `Analysed through`, say "nothing new" and stop;
     (b) build the evidence pack as above, each item scoped to "since the last analysis", with the
     transcript-handling rules (aggregate with the shell, sample sessions through a condensed view,
     fallback and a stated gap when the transcripts are unavailable); (c) apply the finding bar and
     the cap of five; (d) drop what the file already holds; (e) write each finding with
     `status: drafted` and a short aggregate block (counts, averages, whitelisted settings); (f) advance
     `Analysed through` to the current HEAD, even when nothing was found — an empty result is a result;
     (g) report in two or three lines.
   - **`send`** — (a) if no `drafted` findings, say so and stop; (b) print them and the
     no-project-information statement; (c) ask which to send; (d) detect `gh` and its login (detect
     only; on failure leave everything `drafted` and say how to send manually); (e) show the exact text;
     (f) search upstream, comment or create; (g) record `sent <url>` or `declined`; (h) report URLs.
   - **Rules** — never deletes an entry; never commits the file; never edits skills or the project;
     privacy rule first: project-neutral evidence only, and the config whitelist.
2. Add the one-line mentions to `start`, `start-planning`, `find-improvements`, `breakdown-feature`.
3. Update ARCHITECTURE, CLAUDE.md, README, and the TODO-35 note in TODO.md as listed.
4. Re-read each touched file end to end (CLAUDE.md convention: a new capability invalidates sentences
   that never name it) — especially ARCHITECTURE's *Lessons*, the *Getting work into the backlog*
   count of skills, and `start`/`start-planning`'s endings.

## Testing

No automated suite exists for skills; verify as `CLAUDE.md` describes.

1. **Analyse on real history.** Run `claude -p "/microfactory:feedback analyse"` in this repo, which has
   real session transcripts and factory commits. Check against a manual count: stories done, iteration
   count, and the duration and token totals for one session; check the longest and median sessions it
   picked are the right ones. Confirm no transcript was read into context wholesale (the run's token
   use stays small) and that `.microfactory/feedback.md` exists, is listed in `.git/info/exclude`, and
   does not appear in `git status`.
2. **Repeat guard.** Run it again with no new commits: "nothing new", no duplicate entries.
3. **Fallback.** Point the skill at a project with no transcript directory (a scratch repo): it falls
   back to commit timestamps, says tokens are unavailable, and fabricates no numbers.
4. **Whitelist.** Put a fake `jira_site`/`github_username` in a scratch config; confirm neither appears
   anywhere in the findings file.
5. **Send without `gh`.** Findings stay `drafted`, nothing is installed, the message explains the
   manual route.
6. **Send with `gh`** (live smoke, needs auth, against a throwaway repo by temporarily changing the
   target rather than filing noise upstream): the no-project-information statement is shown first; one
   chosen finding becomes an issue, one declined stays in the file as `declined`; a second `send` shows
   neither again; a rerun of `analyse` that finds the same problem does not re-add it.
7. **Privacy read.** Read the drafted issue text and check no path, key, title or name from the project
   survived.
8. **Mention.** With a `drafted` finding present, `/microfactory:start` ends with the one-line mention
   before the loop begins; with none, nothing is said.

## Risks / Edge Cases

- **Transcripts are an undocumented interface.** The format can change between Claude Code versions;
  duration and token numbers are the headline of the analysis and the thing most likely to silently go
  wrong. Mitigated by the stated fallback and by the skill saying what it could not measure. Another
  agent environment (the open copilot/codex items) has no such files at all — `analyse` then runs on
  git and the backlog alone and says so.
- **Transcripts hold the project's code and conversation.** The analysis reads them locally and only
  conclusions leave, but one careless quote would leak. The drafting rule (project-neutral evidence
  only), the setting whitelist and the user reading the exact text before sending are three separate
  barriers; `analyse` itself never sends, so an unattended loop can never publish.
- **The analysis can cost more than it saves.** Aggregating with the shell and sampling only two
  sessions per mode is what prevents that; the skill must state it, and the minimum-stories gate in
  TODO-35 keeps it from running often.
- **Small samples.** An average over three iterations is weak. Findings must show their `n`, and the
  pattern bar will often produce an empty result — intended, since one iteration is not a mandate to set
  policy.
- **Median and "average" session** differ on skewed data; the plan picks the one nearest the median
  and labels it so.
- **Todo backend has no timestamps or authors**, so "done since the last analysis" falls back on git
  history of the TODO file and factory commits. The report says when it used the fallback.
- **The mention nags if findings are never dealt with.** It is one line and only while `drafted`
  findings exist; `declined` removes it. If that proves too quiet, option 2 (a notification) is the
  follow-up.
- **`.git/info/exclude` is per clone**, and the findings file is per working copy; running in a second
  clone starts a fresh file and may re-propose the same findings. The upstream search before creating
  an issue is what keeps that from producing duplicates.
- **Hardcoded upstream repo.** If the plugin moves, one constant changes; a config key is not worth it
  until a fork wants to point elsewhere.
- **Depends on nothing; TODO-35 depends on this.**
