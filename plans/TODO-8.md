# TODO-8: create a skill that finds possible UI/UX improvements for the project

## Summary

Add `find-ui-improvements`, the fourth and last finder. Same conventions as `plans/TODO-5.md`
(read-only, no backend writes, shared finding shape, ranked, ≤10, evidence required, handoffs).

What is different here: **the evidence is visual**. Reading component source tells you what the UI
is made of, not what it is like to use. So this finder must run the application and look at it —
the same stance `plan-next` already takes for UI features ("run the relevant part of the
application and take screenshots of the affected screens"). This skill makes that the primary
method rather than a side note, and each finding should point at the screenshot that shows it.

Screenshots go to a temp directory, never into the repo — the finder is read-only and its output is
a report, not artifacts. If the application cannot be run (no dev command, missing services,
headless-only environment), the skill degrades to reading templates/components and says clearly
that the pass was source-only, because that materially weakens the findings.

Accessibility gets its own emphasis: it is where the cheapest concrete wins usually are (contrast,
focus order, labels, keyboard reachability, hit-target size), and it is checkable rather than
matter of taste.

## Files to Change

| File | Change |
|---|---|
| `skills/find-ui-improvements/SKILL.md` | New skill |
| `CLAUDE.md` | Add to the repository-layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; the finder family is now complete |
| `README.md` | Extend the one-sentence finder list |

## Implementation Steps

1. Write `skills/find-ui-improvements/SKILL.md`:
   1. **Orient** — find the UI: framework, screens/routes, shared components, design system or token
      file, and any design documentation. Find how to run it (a project run/dev skill, `README`, the
      dev script in the manifest).
   2. **See it** — run the app; walk the main flows; screenshot each significant screen and state at
      a desktop and a narrow width, into a temp directory. Include the states that get forgotten:
      empty, loading, error, and long-content. If it will not run, fall back to source and label the
      pass source-only.
   3. **Hunt** — flow friction (avoidable steps, dead ends, no way back); navigation and information
      architecture (where am I, how do I get back, is the hierarchy honest); missing states (empty,
      loading, error, partial, offline); feedback and latency (unacknowledged actions, no progress,
      silent failures); consistency drift (spacing, type scale, button variants, two components for
      one job); forms (validation timing, unhelpful error messages, lost input); destructive actions
      without confirmation or undo; responsiveness at narrow widths; copy (jargon, unclear labels,
      error text that blames the user); accessibility (contrast, focus visibility and order,
      labels and alt text, keyboard reachability, semantics, hit targets).
   4. **Filter** — evidence (screenshot or file), user-visible payoff, incremental (no redesigns),
      in scope: structural refactors hand off to `find-design-improvements`, missing UI tests to
      `find-testing-improvements`, component-library adoption to `find-library-improvements`.
   5. **Report** — same shape, plus the screenshot each finding is visible in; rank by user impact
      over effort, and note accessibility findings as such.
   6. **Stop** — no code, no committed screenshots, no backend writes.
2. Update the doc files; note in `docs/ARCHITECTURE.md` that the family is complete.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line.
- Behavioral smoke is weak here by nature: **this repo has no UI at all**, so the only honest local
  check is that the skill exits cleanly on a project with no UI rather than inventing findings.
  Read the skill against a project that does have one to sanity-check the flow (orient → run →
  screenshot → hunt) is executable without project-specific knowledge.

## Risks / Edge Cases

- **Taste dressed as findings.** The failure mode is subjective polish advice. Mitigated by
  requiring visual evidence, by ranking on user impact, and by favouring checkable categories
  (missing states, accessibility, dead ends) over aesthetics.
- **Cannot run the app** — the common case in CI-like environments; must degrade explicitly, not
  silently.
- **No UI at all** — exit early and say so.
- **Screenshots in the repo** — explicitly forbidden; temp directory only.
- **Overlap with `plan-next`'s UI analysis step** — that runs per-issue while planning a feature;
  this is a standing sweep of the whole product. Worth one line in the skill so the two do not read
  as duplicates.
