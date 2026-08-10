---
name: breakdown-feature
description: Turn a feature description or requirements document into a sequenced set of implementable stories and create the approved ones in the task backend — vertical slices, each sized to a single implementation iteration, with acceptance criteria and open questions marked for refinement. Use when the user has a spec, PRD, feature brief, or requirements doc to break down into backlog items.
---

# Break a feature down into stories

One pass: read the document → ground it in the code → slice → sequence → **get approval** → create.

This is the front of the pipeline: `breakdown-feature` → `refine-story` (for the underspecified ones) → `plan-next` → `implement-next`.

## 1. Read configuration and take the input

Read `.microfactory/config.yaml`; if it does not exist, tell the user to run `/microfactory:init-factory` and stop. Load the backend skill matching `backend` — you need its **create** operation.

The input is a file path, a URL, or text the user pastes. **Read all of it before slicing anything** — requirement documents contradict themselves across sections, and a breakdown built from the first half will be wrong.

If the input is one sentence rather than a document, the breakdown still runs, but say plainly that it rests on a thin input and expect to lean on `needs-refinement`.

## 2. Ground it in the codebase

Find what already exists: features this builds on, features it duplicates, the data model it extends, the conventions it must follow. A breakdown written without reading the code proposes work that is already done and misses the reuse that makes half the stories small.

If the document contradicts what the code does, **say so** and let the user decide. Do not quietly design around it.

## 3. Find the user-visible outcomes

List the jobs the feature lets someone do — each one something a user could notice working. These are the slices. Everything else (schema, endpoints, components) is *part of* a slice, never a slice of its own.

## 4. Slice vertically

- **Walking skeleton first** — the thinnest end-to-end path through the whole feature, however crude. It proves the shape and gives every later story something to attach to.
- **Then one story per outcome**, each shippable on its own.
- **Then the enrichments** — validation, edge cases, error states, admin views, migration of existing data, instrumentation.

Never slice by layer. "Create the schema" / "add the endpoint" / "build the form" produces three stories, none of which can be released, and none of which a session can finish and demonstrate.

**Size rule:** a story must be completable in a single `implement-next` iteration — claim, implement, test, push. If it cannot, it is an epic; split it. This is checkable, which is why it beats story points here.

## 5. Sequence

Order the stories so a session claiming them top-down never hits a missing prerequisite. Name each story's dependencies explicitly. Where two stories are independent, say so — they can be worked in parallel by separate sessions.

## 6. Write each story

Use the same template as `refine-story`, so a story created here and refined later does not change shape:

```markdown
**Goal** — one sentence: who gets what, and why.

**Context** — what exists already, what this builds on.

**Acceptance criteria**
- [ ] checkable statements of behaviour, including the unhappy paths
- [ ] ...

**Out of scope** — what this story deliberately leaves to another.

**Open questions** — what the document does not answer.
```

Record open questions honestly rather than inventing answers — a document that specifies everything does not exist.

## 7. Present the set for approval

Show the ordered list: title, one-line goal, size, dependencies, and which have open questions. **Create nothing until the user approves.** Invite them to cut, merge, or reorder — a twenty-story dump helps nobody, and on Jira or GitHub these writes are visible to their whole team.

## 8. Create and report

Through the backend skill's create operation:

- **Unassigned**, in the initial status, so sessions can claim them.
- **A shared label or tag naming the feature**, so the set stays identifiable as one body of work.
- **`needs-plan`** on the ones substantial enough to warrant a plan first.
- **`needs-refinement`** on the ones with open questions, so `/microfactory:refine-story` can settle them before implementation.

Then report the created keys in dependency order, and what happens next: refine the flagged ones, then let the factory work down the list.

*Related:* `find-improvements` also creates tasks, but it sweeps an existing codebase for improvements. This one turns a stated intention into work.
