---
name: refine-story
description: Sharpen a vague backlog item into an implementable story — resolve what the codebase already answers, put the remaining product choices to the user as closed proposal-shaped questions, then write acceptance criteria back to the task backend. Use when a story is too vague to plan or implement, or when the user asks to refine, clarify, or add acceptance criteria to an item.
---

# Refine a story

One refinement pass: pick the story → answer what the code answers → ask the rest as proposals → rewrite → write it back.

Run this by hand, with the user present. It is **not** a `/loop` target — it asks questions, so it needs someone there to answer them.

## 1. Read configuration and pick the story

Read `.microfactory/config.yaml`; if it does not exist, tell the user to run `/microfactory:init-factory` and stop. Load the backend skill matching `backend`.

Take the issue key from the argument. Without one, list the open items that look unrefined — tagged `needs-refinement` where the project uses that, otherwise the ones whose whole specification is a title — and ask which to work on.

## 2. Answer what you can yourself

**Never ask the user something the repository already answers.** Before writing a single question:

- Read the story as written, and anything it links to.
- Read the code it would touch, the conventions the project documents, and — most useful — a feature already built that resembles it. Existing precedent settles most "how should this behave?" questions.
- Look at the data model, the validation and error handling already in use, and how similar screens or endpoints behave.

Keep a note of what you concluded and what you concluded it from. These become **statements** in the refined story, not questions.

## 3. List what is genuinely undecided

From what is left, the things whose answer would change the implementation:

- **Scope edges** — what is in, and what a reader might reasonably assume is in but is not.
- **Unhappy paths** — what happens when it fails, times out, is already done, or is done twice.
- **Data and validation rules** — required, optional, limits, formats, what happens to existing data.
- **Permissions** — who can do this, who can see it.
- **What the user sees** — the states, the wording, where it lives in the product.
- **Non-functional expectations** — scale, latency, retention, auditing, if any of them bite.

Discard any question whose answer would not change what gets built. Those are curiosity, not refinement.

## 4. Ask as proposals, never as open questions

Use `AskUserQuestion`. Every question must be answerable by **choosing**, not composing:

- **2–4 options per question**, each one a decision that could ship — concrete behaviour, not a direction ("Reject the upload and show which rows failed" beats "improve error handling").
- **Recommended option first**, labelled as such, with the reason it is recommended.
- **State each option's consequence** — what the user gets, what it costs, what it rules out.
- **At most 4 questions per round.** If more remain, resolve the round and ask again with what you learned; earlier answers usually kill later questions.
- **Say what happens by default.** If they pick nothing, they should know what they are getting.
- Free text is the escape hatch the tool already provides, not the format you ask in.

Stop asking when the remaining uncertainty is cheaper to discover during implementation than to resolve now. A story with two open trivia and clear acceptance criteria is refined. Twelve questions is an interrogation, and it is a worse outcome than a slightly vague story.

## 5. Rewrite the story

```markdown
**Goal** — one sentence: who gets what, and why.

**Context** — why now, what exists already, what it touches.

**Acceptance criteria**
- [ ] checkable statements of behaviour, including the unhappy paths
- [ ] ...

**Out of scope** — what this deliberately does not do.

**Decisions** — each choice made during refinement, what was chosen, and why.

**Deferred** — anything left open on purpose, and when it must be answered.
```

A story created by `breakdown-feature` arrives in this shape already, with an **Open questions** section — resolving those is exactly this skill's job, so replace that section with **Decisions** and, for anything left open on purpose, **Deferred**.

Preserve every concrete detail the original already had; refinement adds precision, it does not overwrite the author's intent. Where an answer contradicts the original text, record that in **Decisions** rather than silently dropping it.

## 6. Write it back and stop

- Update the item through the backend skill's update operation.
- Remove a `needs-refinement` tag if the project uses one.
- Leave status and assignee untouched — refining is not claiming.

Then stop. Do **not** plan and do **not** implement: say that the story is ready and that `/microfactory:plan-next` (or `implement-next`) takes it from here.
