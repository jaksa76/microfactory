# TODO-5: create a skill that finds possible software design improvements for the project

## Summary

Add a `find-design-improvements` skill: an analysis-only pass over the project that reports
candidate refactorings, architectural fixes, and pattern cleanups — ranked, justified, and
scoped so each one could become a backlog item.

This is the first of a family the backlog asks for: design, testing, libraries, UI/UX finders,
plus a fifth skill that "uses the above skills to create a list of tasks". Decisions made here
that the later four inherit:

- **Read-only.** A finder never edits code and never writes to the task backend. It reports.
  Creating tasks belongs to the aggregator skill, which keeps "one skill, one job" intact and
  keeps the backend as the single source of truth (no findings files on disk for cross-skill
  coordination — the aggregator loads the finders in its own session and writes the backend
  directly).
- **A shared finding shape**, repeated inline in each finder rather than factored into a shared
  doc — the three backend skills already duplicate their common operation list, so this matches
  the repo's existing convention and keeps each skill self-contained:

  `Title` · `Area / files` · `Problem` · `Proposed change` · `Effort S/M/L` · `Payoff` · `Risk`

- **Capped and ranked.** At most ~10 findings, ordered by payoff-to-effort, so the output is a
  work queue rather than an audit dump.
- **No speculative rewrites.** Findings must name concrete evidence in this repo; "consider
  microservices" style suggestions are explicitly out.

## Files to Change

| File | Change |
|---|---|
| `skills/find-design-improvements/SKILL.md` | New skill |
| `CLAUDE.md` | Add to the repository-layout listing |
| `docs/ARCHITECTURE.md` | Add to the repository-layout listing; note the finder family |
| `README.md` | Mention alongside the other convenience skills |

## Implementation Steps

1. Write `skills/find-design-improvements/SKILL.md`:
   1. **Orient** — read `CLAUDE.md`, `docs/`, and the build/manifest files; map the modules and
      entry points; use git history (churn, files changed together) to find hot spots, since those
      are where design debt actually costs something.
   2. **Look for specific smells** — duplication, oversized modules with mixed responsibilities,
      leaked boundaries (layers reaching past each other), missing abstraction behind repeated
      conditionals, inconsistent handling of the same concern (errors, config, logging), dead code,
      and drift from the conventions the project itself documents.
   3. **Filter** — discard anything without evidence in the code, anything that is a rewrite, and
      anything whose payoff does not clearly beat its risk. Prefer changes that unblock other work.
   4. **Report** — the finding shape above, ranked, capped at ~10, with a one-paragraph summary of
      the project's overall design health first.
   5. **Stop there** — state explicitly: do not implement, do not create backlog items, point at
      the task-list skill for that.
2. Update the three doc files.

## Testing

No automated suite in this repo (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line — the check the other
  skills pass.
- Behavioral smoke: the skill is analysis-only, so it is safe to exercise. Reading it against this
  repo, confirm the steps are answerable here (a small markdown-only repo) and not silently
  assuming a compiled application — a finder that only works on large codebases is a bad finder.

## Risks / Edge Cases

- **Generic output.** The failure mode for this kind of skill is plausible-sounding advice that
  applies to any repo. Mitigated by requiring concrete file/line evidence per finding and by the
  filter step.
- **Scope creep into the other finders.** Design overlaps testing and libraries; the skill should
  hand those off ("testing gaps → find-testing-improvements") rather than absorb them.
- **Small or unusual projects.** Steps must degrade gracefully when there is no build file, no
  layered architecture, or (as here) no code at all beyond markdown.
- **Naming.** `find-design-improvements` fits the family (`find-<area>-improvements`) that the
  remaining three items will follow.
