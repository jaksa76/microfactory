# TODO-6: create a skill that finds possible testing improvements for the project

## Summary

Add `find-testing-improvements`, the second member of the finder family. It inherits every
convention settled in `plans/TODO-5.md` — read-only, no backend writes, the same finding shape,
ranked and capped at ~10, evidence required, handoffs instead of overlap — and differs only in
what it hunts for.

What makes a testing finder distinct from the design finder: the interesting findings are about
the **shape** of the suite, not its size. Coverage percentage is a weak signal; the useful
questions are which behaviors would survive a bug, which tests fail for reasons unrelated to the
code (flakiness, shared state, ordering, clock/network dependence), which tests are slow enough
that people stop running them, and whether the pyramid is inverted (all e2e, no units — or all
units, nothing that proves the app boots).

Two project-specific points this skill must handle, because this repo is the first thing it will
be pointed at:

- A project may legitimately have **no automated tests** (this one has none — the skills are prose).
  The finding then is about what would be worth testing and how, not "add tests" as a platitude.
- A project may have tests that **exist but cannot run** in the current environment. That is a
  finding in its own right, and a high-payoff one.

## Files to Change

| File | Change |
|---|---|
| `skills/find-testing-improvements/SKILL.md` | New skill |
| `CLAUDE.md` | Add to the repository-layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; extend the Finder skills section |
| `README.md` | Fold into the finder mention rather than adding a third paragraph |

## Implementation Steps

1. Write `skills/find-testing-improvements/SKILL.md`:
   1. **Orient** — find the test suites, how they are invoked (CI config is the authority on what
      is actually run, not the README), how long they take, and which layers exist. Try running
      them if it is cheap and safe; a suite that does not run is the first finding.
   2. **Hunt** — coverage of *behaviors* rather than lines (critical paths, error paths, boundaries);
      inverted or missing pyramid layers; flakiness sources (shared state, ordering dependence,
      real clocks, real network, sleeps); slow tests; tests asserting implementation rather than
      behavior; missing regression tests for past bugs (git log for fixes with no accompanying
      test); untestable seams that need a design change to test at all.
   3. **Filter** — same bar as the design finder: evidence, incremental, payoff over risk, in scope
      (a needed seam change hands off to `find-design-improvements`).
   4. **Report** — same finding shape, ranked, ≤10, opening with the suite's overall health: what
      layers exist, what runs in CI, how fast, how trustworthy.
   5. **Stop** — no code, no backend writes.
2. Update the doc files. Keep the README to a single finder sentence covering the family, so the
   remaining two finders do not each add a paragraph.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line.
- Behavioral smoke: this repo is the hard case — no tests at all — so read the skill against it
  and confirm the steps produce something honest rather than dead-ending at "find the test suite".

## Risks / Edge Cases

- **Coverage-number worship.** Must be explicitly discouraged; a coverage report is an input, not a
  finding.
- **Test-count padding.** "Add tests for X" repeated per file is not 10 findings; findings should be
  about classes of gap or specific high-value behaviors.
- **No suite at all** and **suite that will not run** — both handled explicitly, per Summary.
- Overlap with the agentize plugin's readiness criteria (C5.2 unit coverage, C5.3 integration/e2e
  maturity, C7.1 test isolation). Different job — agentize scores readiness against fixed criteria,
  this proposes concrete work — but worth a line in the skill so a user with both installed knows
  which to reach for.
