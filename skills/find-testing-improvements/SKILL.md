---
name: find-testing-improvements
description: Analyze the project's tests and report candidate testing improvements — missing coverage of important behaviors, missing or inverted pyramid layers, flaky and slow tests, tests that cannot run. Read-only: it proposes, it does not write tests or create tasks. Use when the user asks how the project's testing could be improved.
---

# Find testing improvements

One analysis pass: orient → hunt for gaps → filter → report ranked findings. **Write no tests, change no code.**

## 1. Orient

- Find the suites and the layers that exist: unit, integration, end-to-end, contract, manual scripts.
- Find how they are actually invoked. The CI configuration is the authority on what runs; the README often describes an older setup.
- Note how long they take and what they need (a database, a browser, network, credentials).
- If it is cheap and safe, run them. **A suite that cannot run in a clean environment is itself a finding**, usually the highest-payoff one in the report.

If the project has no automated tests at all, that is the situation, not a dead end. Skip to what would be worth testing first — the behaviors whose breakage would hurt most — and say why those, in what layer.

## 2. Hunt

- **Behaviors, not lines** — which important behaviors would survive a bug being introduced? Critical paths, error and failure handling, boundaries and edge inputs, permissions, money and data-loss paths. A coverage report is an input to this question, never a finding on its own.
- **Pyramid shape** — all end-to-end tests and no fast feedback; or thousands of unit tests and nothing proving the app starts and serves a request. Both are findings.
- **Trustworthiness** — flakiness sources: shared mutable state between tests, order dependence, real clocks, real network or filesystem, `sleep`-based waits, randomness without a fixed seed. Tests people rerun until green are worse than no tests.
- **Speed** — the slowest tests, and whether they are slow for a good reason. A suite people avoid running is a broken suite.
- **What is asserted** — tests coupled to implementation detail (mock call counts, private internals) break on refactoring without catching bugs; tests asserting behavior do the opposite.
- **Missing regression tests** — scan git history for bug fixes that landed without a test. Each is a bug that can come back.
- **Untestable seams** — code that cannot be tested without a design change (hard-wired dependencies, side effects in constructors, global state). Note it and hand the seam change to `find-design-improvements`.

## 3. Filter

Drop a candidate unless it survives all of these:

- **Evidence** — name the files, the CI job, the timing, or the history that shows the problem.
- **Incremental** — it can land without stopping other work. "Rewrite the suite" is out.
- **Payoff beats risk** — it would catch a real class of bug, or make the suite fast or trustworthy enough that people run it.
- **In scope** — testing. Design seams hand off to `find-design-improvements`, test tooling and framework upgrades to `find-library-improvements`.

Do not pad. "Add tests for file X" repeated per file is one finding about a gap, not ten findings.

## 4. Report

Open with the suite's health in a few lines — which layers exist, what CI actually runs, how long it takes, how much it can be trusted — then list at most **10** findings, ranked by payoff relative to effort, each as:

- **Title** — one line, imperative ("Pin the clock in the retry tests")
- **Area / files** — concrete paths, jobs, or suites
- **Problem** — the gap or defect and what it lets through, with the evidence
- **Proposed change** — the smallest change that closes it, and at which layer
- **Effort** — S / M / L
- **Payoff** — what it would catch or unblock
- **Risk** — what could go wrong, what to watch

If the suite is in good shape, say so and keep the list short.

## 5. Stop

Do **not** write tests and do **not** create backlog items — this skill only proposes. The task-list skill turns findings into tasks; implementation happens through the normal backlog.

*Related:* the [agentize](https://github.com/jaksa76/agentize) plugin scores a project against fixed readiness criteria (C5.2 unit coverage, C5.3 integration/e2e maturity, C7.1 test isolation). Use that to grade, this to get concrete work proposals.
