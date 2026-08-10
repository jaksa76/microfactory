---
name: breakdown-feature
description: Turn a feature description or requirements document into a sequenced set of implementable stories and create the approved ones in the task backend — lean vertical slices, each sized to a single implementation iteration, with open questions marked for refinement. Use when the user has a spec, PRD, feature brief, or requirements doc to break down into backlog items.
---

# Break a feature down into stories

One pass: read the document → ground it in the code → slice → sequence → **get approval** → create.

This is the front of the pipeline: `breakdown-feature` → `refine-story` (for the underspecified ones) → `plan-next` → `implement-next`.

## 1. Read configuration and take the input

Read `.microfactory/config.yaml`; if it does not exist, tell the user to run `/microfactory:init-factory` and stop. Load the backend skill matching `backend` — you need its **list**, **view**, and **create** operations.

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

**Write stories the way this project already writes them.** Before writing the first one, look at how existing items are written — list a dozen recent items in the backend and read a few in full, and check for an issue template (`.github/ISSUE_TEMPLATE/`, a Jira description template, the existing entries in `TODO.md`) or a house style in `CONTRIBUTING.md`. If a convention is there, follow it: the same headings, the same length, the same voice. A backlog whose newest twenty items look nothing like the older ones is worse than an imperfect template.

**If no convention is detectable**, use the agile story form where it fits:

```
As a <who>, 
I want <what>, 
so that <why>.
```

That is the whole story, and it goes in the description; the title stays a short summary of it ("Invite a teammate by email"), because that is what a board shows. Add a sentence or two beneath only when a reader would otherwise get it wrong — what this builds on, or what it deliberately leaves to another story. On the `todo` backend there is no description at all, so the title carries the story and anything extra goes in indented lines beneath it.

**Where that form does not fit, write something simpler.** Plenty of real work has no user in it — a dependency upgrade, a migration, a spike, a build fix. Forcing "As a developer, I want…" onto those adds words and no meaning. A plain imperative title is a perfectly good task.

Whatever the shape, keep it lean: no acceptance criteria — not as a checklist, not as tickable behaviour statements, not under another heading — and no enumeration of cases the document does state in passing. The story is a placeholder for a conversation, and the conversation happens in `refine-story` and in the code.

**Open questions** are the one addition that earns its place: where the document leaves a real product choice open, list those questions at the end so `refine-story` can settle them. Record them honestly rather than inventing answers — a document that specifies everything does not exist. If there are none, the story ends after its first line.

## 7. Present the set for approval

Show the ordered list: title, one-line goal and which have open questions. **Create nothing until the user approves.** Invite them to cut, merge, or reorder — a twenty-story dump helps nobody, and on Jira or GitHub these writes are visible to their whole team.

## 8. Create and report

Through the backend skill's create operation:

- **Unassigned**, in the initial status, so sessions can claim them.
- **A shared label or tag naming the feature**, so the set stays identifiable as one body of work.
- **`needs-plan`** on the ones substantial enough to warrant a plan first.
- **`needs-refinement`** on the ones with open questions, so `/microfactory:refine-story` can settle them before implementation.

Then report the created keys in dependency order, and what happens next: refine the flagged ones, then let the factory work down the list.

*Related:* `find-improvements` also creates tasks, but it sweeps an existing codebase for improvements. This one turns a stated intention into work.
