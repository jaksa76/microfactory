---
name: find-library-improvements
description: Analyze the project's dependencies and report candidate library improvements — vulnerable or unmaintained packages, worthwhile upgrades, redundant or oversized dependencies to drop, and gaps a library would genuinely fill. Read-only: it proposes, it does not touch manifests or create tasks. Use when the user asks about dependencies, upgrades, or library choices.
---

# Find library improvements

One analysis pass: orient → gather facts by command → hunt → filter → report. **Change no manifest, install nothing.**

Bias toward **subtraction**. Every dependency is a permanent liability — supply chain risk, an upgrade treadmill, transitive weight — so "drop this", "we already have something that does this", and "this is vulnerable" outrank "adopt this". A report that mostly recommends new libraries has made the project worse.

## 1. Orient

- Find the manifests and lockfiles, and which ecosystems are in play (a repo often has more than one).
- Note whether versions are pinned and whether lockfiles are committed.
- Note what automation already runs: Dependabot, Renovate, `npm audit` in CI. Findings must not duplicate work a bot already does.

If the project has no dependency manifest at all, say so and stop early — there is nothing here to find, and inventing dependency findings is worse than an empty report.

## 2. Gather facts by command

Get the current state from the tools, not from memory. Examples, adapt to the ecosystem and OS:

| Ecosystem | Outdated | Advisories |
|---|---|---|
| npm / pnpm / yarn | `npm outdated` | `npm audit` |
| Python | `pip list --outdated`, `uv pip list --outdated` | `pip-audit` |
| Rust | `cargo outdated` | `cargo audit` |
| Go | `go list -m -u all` | `govulncheck ./...` |
| Maven | `mvn versions:display-dependency-updates` | `mvn org.owasp:dependency-check-maven:check` |
| Gradle | `./gradlew dependencyUpdates` (versions plugin) | `./gradlew dependencyCheckAnalyze` |

Also useful: `gh api repos/{owner}/{repo}/dependabot/alerts` where Dependabot is enabled.

**Never state a version number, release date, or vulnerability from your own knowledge** — it is stale by construction. Every such claim in the report must trace to command output or an advisory you read. If a command cannot run (offline, restricted network, missing toolchain), report that as a limitation of the pass instead of guessing.

## 3. Hunt

In this priority order:

1. **Known vulnerabilities with a fix available** — the version that fixes it, and whether it is a breaking change.
2. **Unmaintained or abandoned dependencies** — no releases or commits for years, archived repo, deprecated on the registry. Note the live alternative.
3. **Dependencies carrying their weight badly** — a large package used for one trivial function; a transitive-heavy package where a small one would do.
4. **Redundancy** — two libraries doing the same job (two HTTP clients, two date libraries, two test runners), usually a merge artifact.
5. **Reinvention** — hand-rolled code that a dependency **the project already has** does better and safer.
6. **Reproducibility gaps** — missing or uncommitted lockfile; unpinned versions where builds must be reproducible.
7. **Upgrades the project is far behind on** — several majors back, where each further release widens the gap. Stage these; never propose a multi-major jump as one step.
8. **Licence problems** — a dependency whose licence conflicts with how the project is distributed.
9. **Only then, a new library** — and only with what it replaces, what it costs, and why the liability is worth paying.

## 4. Filter

Drop a candidate unless it survives all of these:

- **Evidence** — command output, an advisory, a registry page, or the code that shows the problem.
- **Breaking changes checked** — no upgrade goes in the report without a look at its release notes for what breaks. "Bump to latest" without that is not a finding.
- **Worth the liability** — no new dependency to replace a few lines of code.
- **Not bot work** — if Dependabot/Renovate already handles routine bumps, say so and leave them out.
- **In scope** — dependencies. Structural changes hand off to `find-design-improvements`, test framework gaps to `find-testing-improvements`.

## 5. Report

Open with the dependency picture in a few lines — ecosystems, how many direct dependencies, pinned or not, what automation exists — then list at most **10** findings, ranked security first, then risk reduction, then convenience, each as:

- **Title** — one line, imperative ("Drop left-pad in favour of String.padStart")
- **Area / files** — manifest, lockfile, and the code affected
- **Problem** — what is wrong and what it exposes, with the evidence
- **Proposed change** — the specific version or removal, staged if it cannot land in one step
- **Effort** — S / M / L
- **Payoff** — risk removed, weight removed, or capability gained
- **Risk** — what breaks, what to test after

## 6. Stop

Do **not** edit manifests or lockfiles, do **not** run installers, and do **not** create backlog items — this skill only proposes. The task-list skill turns findings into tasks.
