# Lessons

Judgements an iteration reached but had no mandate to act on: things that look like they ought to
change, observed while doing something else. One iteration is one data point, which is enough to
record an observation and not enough to set policy — so these wait here for a human.

**What belongs here**: what *should* be true, where acting on it would widen the change that noticed
it. **What does not**: facts about the project, which go into `CLAUDE.md` where agents load them;
procedures, which become skills; and work that is already decided, which goes in the backlog.

Entries are removed when acted on or rejected — this is a queue, not a log. Git keeps the history.

---

## `implement-next` step 8 conflates two reasons for skipping the cold review

Step 8 distinguishes a **capability** failure ("the environment cannot run a separate agent") from a
**configuration** choice (`deep_review: false`, or a `skip-review` label), and requires the report to
say which applied. There is a third case it does not name: a session whose **policy** forbids spawning
subagents even though the harness is perfectly capable of it.

Observed on three consecutive iterations in this repo, where every cold review fell back to
self-review for exactly that reason. Reported as a capability failure, it misleads — it suggests
fixing an environment that is not broken.

Worth naming as its own case, so the report says "not permitted here" rather than "not possible here".
Cheap: one clause in step 8 and one in the report line.

## `implement-next` does not check for a dirty working tree

`plan-next` step 2 stops the iteration when the working tree has uncommitted changes — *"never plan on
top of someone's half-done work"*. `implement-next` step 2 only pulls, and will happily build on
whatever it finds.

The asymmetry looks accidental rather than considered: the reasoning that justifies stopping is at
least as strong for implementation, which commits and pushes. It also gets worse under the plan in
`plans/TODO-29.md`, where each iteration runs in its own context — an agent inheriting a predecessor's
failed working tree has no idea what is in it.

The counter-argument is that implementation is the unattended path that must keep running, and
stopping on a dirty tree could wedge a loop indefinitely. So the right answer may be "stop and say
so", or may be "stash and continue" — which is exactly why it is a judgement and not a fix.

## Two settings default in opposite directions with no rule to follow

An absent `deep_review` key means **true** (preserving what a config written before the setting
existed used to do). An absent `auto_plan` key means **false** (changing it). Both were deliberate,
both are documented, and together they leave the next setting with two contradictory precedents.

The natural principle is *"an absent key means whatever preserves the behaviour of a config written
before it existed"* — which is what `deep_review` does, and which would have made `auto_plan` default
to true. Adopting it therefore means revisiting a decision already made on purpose, which is why this
is here rather than in `CLAUDE.md`.

The alternative principle — *"an absent key means the better default, and upgrades are documented"* —
is equally defensible and needs no revisiting. What matters is that one of them is written down before
a third setting picks a third answer.

## Line-number keys orphan plan files, and the todo backend shifts them itself

`todo-tasks` keys an item by its line number, and `plan-next` writes the plan to `plans/<KEY>.md`. But
line numbers move: the backend's own **comment** operation inserts a note line beneath an item, which
shifts every item below it. So an item can be planned as `TODO-29`, drift to line 39 as notes accumulate
above it, and arrive at line 24 after a tidy-up — while its plan sits at `plans/TODO-29.md`, findable by
nothing.

Observed on the one item in this repo that has a plan awaiting review. Its plan file and every
cross-reference inside it (five keys, pointing at four other items) had to be renumbered by hand. The
failure is silent in the direction that matters: `implement-next` step 5 case 3 reads "no plan file" and,
with `auto_plan` off, implements from the issue text — so a plan that was written, reviewed and approved
is skipped without anyone being told.

`todo-tasks` already warns that adding a line shifts the keys below it, but only about *re-reading to get
current keys*. It does not say that anything durable was named after the old one.

The counter-argument is that line numbers are exactly what makes this backend free — no ID allocation, no
registry, no state outside the file — and that stable keys mean either writing an ID into every line or
keeping a side table, which is the ceremony this backend exists to avoid. It is also the try-it-out
backend; Jira and GitHub have real IDs and none of this applies. So the fix may be narrower than a new key
scheme: have the comment operation *not* shift keys, or have step 5 look for a plan whose title names the
item rather than trusting the filename.
