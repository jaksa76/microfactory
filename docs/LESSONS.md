# Lessons

Judgements an iteration reached but had no mandate to act on: things that look like they ought to
change, observed while doing something else. One iteration is one data point, which is enough to
record an observation and not enough to set policy — so these wait here for a human.

**What belongs here**: what *should* be true, where acting on it would widen the change that noticed
it. **What does not**: facts about the project, which go into `CLAUDE.md` where agents load them;
procedures, which become skills; and work that is already decided, which goes in the backlog.

Entries are removed when acted on or rejected — this is a queue, not a log. Git keeps the history.

---

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
