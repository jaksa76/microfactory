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
