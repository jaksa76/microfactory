---
name: find-ui-improvements
description: Run the project's UI, look at it, and report candidate UI/UX improvements — flow friction, navigation and information architecture, missing empty/loading/error states, inconsistency, form and copy problems, accessibility defects. Read-only: it proposes, it does not change the UI or create tasks. Use when the user asks how the product's UI, UX, or accessibility could be improved.
---

# Find UI/UX improvements

One analysis pass: orient → **see it running** → hunt → filter → report. **Change nothing.**

Reading component source tells you what the UI is made of, not what it is like to use. So this pass looks at the running application; findings that cannot be seen are usually not findings.

## 1. Orient

- Find the UI: framework, screens or routes, shared components, design tokens or design system, and any design documentation.
- Find how to run it: a project run/dev skill if the repo has one, the README, the dev script in the manifest, the compose file.
- Note who the users are and what the product is for. Friction only matters relative to a job someone is trying to do.

If the project has no UI, say so and stop. Do not manufacture findings from templates that nothing renders.

## 2. See it

- Start the application and walk its main flows end to end, as a user would.
- Screenshot every significant screen — and the states that get forgotten: **empty, loading, error, partial data, long content**.
- Look at a narrow width as well as a desktop one.
- Put screenshots in a temp directory. **Never commit them** — this pass produces a report, not artifacts.

If the application cannot be run here (no dev command, missing services or credentials, headless-only environment), fall back to reading templates and components — and **say in the report that the pass was source-only**. It materially weakens what you can claim.

## 3. Hunt

- **Flow friction** — steps that could be removed or deferred, work the product could do for the user, dead ends with no way back or forward.
- **Navigation and information architecture** — can the user tell where they are, how to get back, and what belongs to what? Does the hierarchy match how users think about the domain, or how the database is shaped?
- **Missing states** — empty, loading, error, partial, offline, "nothing found". A screen that only looks right with ideal data is unfinished.
- **Feedback and latency** — actions that give no acknowledgement, slow operations with no progress, failures that fail silently, optimistic updates that lie.
- **Consistency drift** — spacing and type scale wandering, several button or card variants doing one job, two components built for the same purpose.
- **Forms** — validation that fires at the wrong moment, error messages that do not say how to fix it, lost input on error, unclear required fields.
- **Destructive actions** — deleting or overwriting with no confirmation, and no undo.
- **Responsiveness** — what breaks, overflows, or becomes unreachable at narrow widths.
- **Copy** — jargon, ambiguous labels, error text that blames the user, inconsistent terminology for one concept.
- **Accessibility** — contrast, focus visibility and order, labels and alt text, keyboard reachability of everything clickable, semantic structure, hit-target size. These are checkable rather than matters of taste, and they are usually the cheapest real wins in the report.

## 4. Filter

Drop a candidate unless it survives all of these:

- **Evidence** — a screenshot showing it, or the file that causes it.
- **User-visible payoff** — someone's task gets easier, faster, or possible at all. Polish for its own sake is out.
- **Incremental** — it lands as a change, not a redesign. "Rebuild the navigation" is out; "give the settings page a back path" is in.
- **In scope** — the interface. Structural refactors hand off to `find-design-improvements`, missing UI tests to `find-testing-improvements`, adopting a component library to `find-library-improvements`.

## 5. Report

Open with a short read on the product's overall UX — what it does well, where it makes users work — then list at most **10** findings, ranked by user impact relative to effort, each as:

- **Title** — one line, imperative ("Add an empty state to the issue list")
- **Area / files** — screen or flow, and the components behind it
- **Problem** — what the user experiences and what it costs them, with the screenshot or file that shows it
- **Proposed change** — the smallest change that fixes it
- **Effort** — S / M / L
- **Payoff** — whose task gets better, and how
- **Risk** — what could regress, what to check

Mark accessibility findings as such, so they can be prioritised as correctness rather than preference.

## 6. Stop

Do **not** change the UI, do **not** commit screenshots, and do **not** create backlog items — this skill only proposes. The task-list skill turns findings into tasks.

*Related:* `plan-next` also runs the UI and takes screenshots, but per issue, while planning one feature. This skill is a standing sweep of the whole product.
