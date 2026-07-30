---
name: find-design-improvements
description: Analyze the project and report candidate software design improvements — refactorings, architectural fixes, misplaced responsibilities, missing abstractions, pattern inconsistencies. Read-only: it proposes, it does not change code or create tasks. Use when the user asks what could be improved in the project's design, structure, or architecture.
---

# Find design improvements

One analysis pass: orient → hunt for smells → filter → report ranked findings. **Change nothing.**

## 1. Orient

- Read what the project says about itself: `CLAUDE.md`/`AGENTS.md`, `README`, `docs/`, ADRs.
- Read the build and manifest files to learn the module boundaries the project intends.
- Map the entry points and the main flows between modules. State this map to yourself before judging anything — most bad design advice comes from not knowing what the code is for.
- Use git to find where the design actually costs money: files with the highest churn, files that keep changing together (a coupling signal), and files many others import.

Adapt to what exists. A small repo, a markdown-only repo, or one with no build file still has a design; skip the steps that have nothing to read rather than inventing findings.

## 2. Hunt for smells

Look for evidence of:

- **Duplication** — the same logic or knowledge in several places, especially if the copies have already drifted.
- **Oversized units** — modules, classes, or functions doing several unrelated jobs; the ones that appear in every other change.
- **Leaked boundaries** — layers reaching past their neighbors, business rules in transport or UI code, infrastructure types in the domain.
- **Missing abstraction** — the same conditional or the same 5-line dance repeated; a concept the code clearly has but never names.
- **Inconsistent treatment of one concern** — errors, configuration, logging, validation, or persistence handled three different ways.
- **Dead weight** — unreachable code, unused exports, abandoned config, features nothing calls.
- **Drift from the project's own conventions** — the repo documents a rule and the code stopped following it. These are the cheapest, safest findings.

## 3. Filter

Drop a candidate unless it survives all of these:

- **Evidence** — you can name the files (and lines) that show the problem. No finding may rest on a general principle alone.
- **Not a rewrite** — it can land incrementally, with tests still passing at each step. "Move to microservices", "switch frameworks", "restructure everything" are out.
- **Payoff beats risk** — it makes upcoming work easier, or removes a recurring bug source. Aesthetic-only churn is out.
- **In scope** — this is design. Hand off missing tests to `find-testing-improvements`, dependency work to `find-library-improvements`, and screen-level concerns to `find-ui-improvements`; mention the handoff, do not absorb it.

## 4. Report

Open with one short paragraph on the project's overall design health, then list at most **10** findings, ranked by payoff relative to effort, each as:

- **Title** — one line, imperative ("Extract issue-state handling out of the report builder")
- **Area / files** — concrete paths
- **Problem** — what is wrong and what it costs, with the evidence
- **Proposed change** — the smallest change that fixes it
- **Effort** — S / M / L
- **Payoff** — what gets better, and for whom
- **Risk** — what could go wrong, what to watch

If nothing meets the bar, say so plainly. A short honest list beats a padded one.

## 5. Stop

Do **not** implement anything and do **not** create backlog items — this skill only proposes. To turn findings into tasks, the user runs the task-list skill; to implement one, they pick it up as a normal backlog item.
