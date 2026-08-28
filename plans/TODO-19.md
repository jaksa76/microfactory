# TODO-19: run the implement-next review subagents in the foreground

## Summary

Step 8 says to hand the diff to a fresh agent but never says to *wait* for it. Nothing in the skill
stops step 9 from committing and pushing while the reviewer is still reading, and in harnesses where
a delegated agent runs detached by default, that is what an implementer following the words literally
would do. The whole value of the step is that findings land before the commit does; a review whose
findings arrive after the push is not a review, it is a post-mortem.

The fix is one sentence of sequencing, not a mechanism. Per the plugin's conventions, the skill states
the requirement — the reviewer's findings are in hand before anything is committed — and leaves the
executing agent to map that onto its harness, which for harnesses that default to detached delegation
means explicitly asking for an awaited run.

Two details are worth stating beyond the bare rule:

- **The multi-reviewer case.** If more than one reviewer is used, awaiting each one in turn serializes
  work that has no reason to be serial. The shape that keeps both properties is: launch them together,
  await all of them, then commit. Saying only "run reviewers in the foreground" would forbid the
  parallel launch for no gain, which is the wrong lesson to encode.
- **Where the boundary actually is.** It is the commit, not step 8's end — step 9.1 re-runs the tests
  when the review changed anything, so a late finding invalidates the test run too, not just the diff.

This is the concern one level down from TODO-29, which has the dispatcher await its iteration agent.
Same rule, different altitude; that plan references this one rather than restating it, and this change
should keep the wording close enough that the two read as one policy.

## Files to Change

| File | Change |
|---|---|
| `skills/implement-next/SKILL.md` | Step 8: await the reviewer(s) before committing; the parallel-launch shape for multiple reviewers |
| `docs/ARCHITECTURE.md` | One clause in the review paragraph — findings precede the commit |

`README.md` is not in scope: it describes that a review happens, not how it is sequenced.

## Implementation Steps

1. **`skills/implement-next/SKILL.md`, step 8** — after the paragraph that hands the diff to the fresh
   agent, add the sequencing rule: wait for the reviewer's findings before going on; nothing from step
   9 starts while a review is outstanding. Name the harness-adaptive part (where delegated agents run
   detached unless told otherwise, ask for the awaited form) without naming a parameter, which would
   be exactly the brittle harness detail `CLAUDE.md` warns against.
2. **Same step** — one sentence for the multi-reviewer case: launch together, await all, then commit.
3. **Same step** — make the boundary explicit: the gate is the commit in step 9, so a finding that
   arrives after the tests were re-run means re-running them again.
4. **`docs/ARCHITECTURE.md`** — extend the sentence that already explains why review comes last with
   the fact that the iteration blocks on it. The doc currently says review runs before the commit;
   it does not say the iteration waits, which is the same gap being closed in the skill.

## Testing

No automated suite for the skills (CLAUDE.md). Verification:

- Structural: frontmatter untouched, `name` still matches the directory.
- Consistency: the sequencing sentence in `implement-next` and the clause in `ARCHITECTURE.md` must
  agree on where the boundary is (the commit, not the end of step 8).
- Read step 8 and step 9 straight through as an implementer would, checking there is no longer a
  reading in which the push can precede the findings.
- No behavioural run: this changes the order of operations inside an iteration, which is only
  observable by running a full iteration against a scratch project, and this iteration's own review
  path is a worse test than reading it — see the risk below.

## Risks / Edge Cases

- **Encoding a harness parameter.** The obvious way to write this is to name the flag that makes a
  delegated agent run in the foreground. That is the brittle-one-liner failure the conventions call
  out, and the default it refers to has already varied between harness versions. State the ordering
  requirement instead.
- **Forbidding parallel reviewers by accident.** Mitigated by step 2; the rule is "await before
  commit", not "one at a time".
- **Overlap with TODO-29.** That plan makes the dispatcher await its iteration agent. Keep this change
  to the reviewer-inside-an-iteration level and let TODO-29 own the level above, or the two will
  contradict each other on which layer enforces the wait.
- **This iteration cannot demonstrate the fix.** The session implementing it does not spawn review
  agents at all (see TODO-12's iteration note), so the change ships reviewed by its own author against
  the checklist, and unexercised. That is worth saying plainly rather than implying a verification that
  did not happen.
