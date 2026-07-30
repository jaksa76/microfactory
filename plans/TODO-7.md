# TODO-7: create a skill that finds possible library improvements for the project

## Summary

Add `find-library-improvements`, the third finder. Same conventions as `plans/TODO-5.md`
(read-only, no backend writes, shared finding shape, ranked, ≤10, evidence required, handoffs).

The item names "new libraries, library upgrades", so the skill covers both directions — but the
default bias must be **subtraction, not addition**. Every dependency is a permanent liability
(supply chain, upgrade treadmill, transitive bloat), so "remove this" and "we already depend on
something that does this" outrank "adopt this". A finder that mostly recommends new libraries
would make the project worse.

The other thing that separates this finder from its siblings: the facts are **checkable by
command**, per ecosystem — `npm outdated` / `npm audit`, `pip list --outdated` / `pip-audit`,
`cargo outdated` / `cargo audit`, `mvn versions:display-dependency-updates`, `go list -m -u all`,
`gh api` for Dependabot alerts. The skill should name these as examples per the repo's convention
(instructions, not code — the agent adapts to the user's ecosystem and OS) and require that every
version claim comes from actually running one, never from the model's memory of a version number.
Model knowledge of "latest" is stale by construction.

## Files to Change

| File | Change |
|---|---|
| `skills/find-library-improvements/SKILL.md` | New skill |
| `CLAUDE.md` | Add to the repository-layout listing |
| `docs/ARCHITECTURE.md` | Add to the layout listing; update the finder-family sentence |
| `README.md` | Extend the one-sentence finder list |

## Implementation Steps

1. Write `skills/find-library-improvements/SKILL.md`:
   1. **Orient** — find the manifests and lockfiles, the ecosystems in play, whether versions are
      pinned, and whether a lockfile is committed. Note what automation already exists (Dependabot,
      Renovate) so findings do not duplicate a bot's job.
   2. **Gather facts by command** — run the ecosystem's outdated/audit commands; read advisories.
      Never state a version or vulnerability from memory.
   3. **Hunt** — in priority order: known vulnerabilities with a fix available; unmaintained or
      abandoned dependencies; dependencies used for one trivial function; two libraries doing the
      same job; hand-rolled code that a dependency the project *already has* would do better;
      missing or uncommitted lockfile; unpinned versions in anything reproducible; major upgrades
      the project is far behind on (with sequencing, since these must be staged); licence
      incompatibilities. Only then: a genuinely worthwhile new library, and only with what it
      replaces and why the liability is worth it.
   4. **Filter** — same bar, plus: no upgrade proposed without checking its breaking changes, and no
      new dependency proposed to replace ten lines of code.
   5. **Report** — same shape, ranked with security first, then risk-reduction, then convenience.
      Group a big upgrade into stages if it cannot land in one step.
   6. **Stop** — no edits to manifests, no backend writes.
2. Update the doc files.

## Testing

No automated suite (CLAUDE.md). Verification:

- Structural: frontmatter `name` matches directory, single `description` line.
- Behavioral smoke: this repo has **no dependency manifest at all** (markdown skills only), so read
  the skill against it and confirm it reports honestly rather than fabricating dependency findings.
  Also confirm the commands named are correct invocations for their ecosystems.

## Risks / Edge Cases

- **Hallucinated versions.** The dominant failure mode. Mitigated by requiring command output for
  every version and advisory claim.
- **Upgrade-everything advice.** A list of 40 minor bumps is a bot's output, not a finding. If
  Dependabot/Renovate is configured, say so and skip the routine bumps.
- **Dependency addition bias.** Guarded by the subtraction-first ordering in the hunt step.
- **No manifest / vendored dependencies / monorepo with many manifests** — all need graceful
  handling.
- **Offline or restricted environments** — audit commands may fail; report that as a limitation of
  the pass rather than guessing.
