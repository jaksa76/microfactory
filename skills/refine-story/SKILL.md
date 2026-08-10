---
name: refine-story
description: Run one refinement iteration of the microfactory — pick an item that is too vague to plan or implement, resolve everything the codebase already answers, post the few genuine product questions left as a comment on the task, and once a human has answered them rewrite the item with acceptance criteria. Runs unattended; intended as a /loop target (e.g. /loop 30m /microfactory:refine-story); also takes an issue key to refine a specific item.
---

# Refine the next story

One refinement iteration: pick an item → answer what the code answers → **post** the remaining questions as a comment → next iteration (or a later one) folds the human's answers back into the story.

This skill runs **unattended**. Never use `AskUserQuestion` and never wait for a reply — the human answers in the task backend, in their own time, and a later iteration picks the answers up. All communication goes through issue comments, because the backend is the single source of truth.

Act on **at most one item per iteration**.

## 1. Read configuration and pick an item

Read `.microfactory/config.yaml`; if it does not exist, tell the user to run `/microfactory:init-factory` and stop. Load the backend skill matching `backend` — you need its **view**, **list comments**, **comment**, and **update** operations.

**With an issue key argument**: work that item.

**Without one**: list open, unassigned items that are eligible for refinement, in backlog order —

- tagged `needs-refinement`, or
- items whose entire specification is their title (no body, no acceptance criteria).

Refinement never assigns the item and never transitions it. An assigned item is not claimable by `plan-next` or `implement-next`, and refining is not claiming. The questions comment is what tells another session this item is already being handled.

## 2. Read the item's comments and decide the state

Fetch the item and **its comments**. A questions comment posted by this skill always opens with the marker line `**Refinement questions**` (round 2 adds `(round 2)`) and closes with the `_Posted by microfactory refine-story…_` footer. Everything between those two lines is ours; anything after the footer is somebody's reply. Rely on that, not on comment authorship — the todo backend records no authors, and on a shared Jira or GitHub project a teammate's answer may come from any account. Ignore comments other factory skills left (plan links, implementation notes): they are not answers.

From that:

| State | What you see | What to do |
|---|---|---|
| **A — not asked** | no marker comment | steps 3–7: analyze, draft, prune, post |
| **B — answered** | marker comment, and at least one later comment that is not one of ours | steps 8–9: fold the answers in and rewrite |
| **C — waiting** | marker comment, nothing after it | skip this item, move to the next eligible one |

**Take state B before state A.** Answers that are already in should be turned into a refined story before new questions go out to anybody — otherwise unanswered rounds pile up faster than the humans clear them.

If every eligible item is in state C, say "No refinement work available." and end the iteration — the humans have the ball, and the loop cadence handles waiting.

## 3. Answer what you can yourself

**Never ask a human something the repository already answers.** Before drafting a single question:

- Read the story as written, and anything it links to.
- Read the code it would touch, the conventions the project documents, and — most useful — a feature already built that resembles it. Existing precedent settles most "how should this behave?" questions.
- Look at the data model, the validation and error handling already in use, and how similar screens or endpoints behave.

Keep a note of what you concluded and what you concluded it from. These become **statements** in the refined story, not questions.

## 4. Draft the questions: functionality and experience, not implementation

Ask about what the product does and what the user sees:

- **Scope edges** — what is in, and what a reader might reasonably assume is in but is not.
- **Unhappy paths** — what the user gets when it fails, times out, is already done, or is done twice.
- **Data and validation rules** — required, optional, limits, formats, what happens to data that already exists.
- **Permissions** — who can do this, who can see it.
- **What the user sees** — the states, the wording, where it lives in the product.
- **Non-functional expectations** — scale, latency, retention, auditing, when any of them actually bite.

**Do not ask implementation questions.** Which library, which pattern, how to structure the module, where the code lives — those are yours to decide, or `plan-next`'s. The exception is a story that is *itself* technical (a performance target, a refactoring, a dependency upgrade, a migration): there the technical choices are the product choices, so ask about them — expected gain, acceptable regressions, what must not change, what may break.

## 5. Prune hard — every question costs a human

Now delete questions. This step matters more than the drafting.

Drop a question when:

- The answer is **obvious** — one answer is what any reasonable person would say, or the project has already done it that way five times. Take the obvious answer and record it as a decision.
- The answer **would not change what gets built**. That is curiosity, not refinement.
- It is **cheaper to discover during implementation** than to resolve now.
- A convention, an existing feature, or the story's own text already settles it.

**Leaving a story exactly as it is, is a perfectly good outcome.** If nothing survives the prune, do not post a questions comment: refine the story from what you concluded (step 9), and note in the comment that no human input was needed.

**Cap it at 4 questions.** If more survive, keep the four whose answers change the most and defer the rest — earlier answers usually kill later questions anyway. A story with two open trivia and clear acceptance criteria is refined; twelve questions is an interrogation, and a worse outcome than a slightly vague story.

## 6. Shape each surviving question — cheapest form that works

In this order, always preferring the form higher up the list:

1. **Yes/no confirmation.** When one option is clearly dominant, do not make anyone compare a menu. Propose it and ask for a yes: *"Should an expired invite show the sign-up page with an 'expired' notice, rather than a 404?"* This is the form to reach for by default.
2. **Multiple choice, 2–4 options.** When you are genuinely unsure which way to go. Every option must be a decision that could ship — concrete behaviour, not a direction ("Reject the upload and show which rows failed" beats "improve error handling"). Recommend one, put it first, say why, and state what each option costs or rules out.
3. **Open question.** Last resort, only when you cannot enumerate the plausible answers at all. If you are writing more than one of these, you have not done step 3 properly.

Every question states its **default** — what gets built if nobody answers. That is what makes silence a safe answer.

## 7. Post the questions and stop

Post one comment through the backend's comment operation:

```markdown
**Refinement questions**

Reply in a comment, answering by number (e.g. `1: yes`, `2: b`, `3: keep the old ones read-only`).
Skip anything you have no opinion on — the default applies.

**1.** Should an expired invite link show the sign-up page with an "expired" notice, rather than a 404?
*Default: yes.*

**2.** What does the member list show for a team with nobody in it yet?
  **a.** Empty state with an "Invite someone" button — matches the projects list *(recommended, default)*
  **b.** The list with only the current user in it
  **c.** Hide the list until there are two members — fewer states, but no obvious way to invite

_Posted by microfactory refine-story. Answering here unblocks this story._
```

Then end the iteration. Do not plan, do not implement, do not rewrite the story yet — an unanswered question is not a decision.

## 8. Fold the answers in (state B)

Read every comment after the marker comment. Then:

- Apply each answer. Where an answer arrives as free text that contradicts the options offered, the human is right — follow what they said.
- For questions nobody answered, **apply the stated default** and record it as such.
- If an answer opens a genuinely new question that changes what gets built, you may post **one** round-2 comment (marker `**Refinement questions (round 2)**`) and end the iteration there. Two rounds is the limit — after that, decide, and put anything still open under **Deferred**.

## 9. Rewrite the story and write it back

```markdown
**Goal** — one sentence: who gets what, and why.

**Context** — why now, what exists already, what it touches.

**Acceptance criteria**
- [ ] checkable statements of behaviour, including the unhappy paths
- [ ] ...

**Out of scope** — what this deliberately does not do.

**Decisions** — each choice made during refinement, what was chosen, and why (repository precedent, or the human's answer, or the default applied because nobody objected).

**Deferred** — anything left open on purpose, and when it must be answered.
```

A story created by `breakdown-feature` arrives in this shape already, with an **Open questions** section — resolving those is exactly this skill's job, so replace that section with **Decisions** and **Deferred**.

Preserve every concrete detail the original already had; refinement adds precision, it does not overwrite the author's intent. Where an answer contradicts the original text, record that in **Decisions** rather than silently dropping it.

Then:

- Update the item through the backend skill's update operation.
- Remove the `needs-refinement` tag if the project uses one.
- Comment a one-line summary of what changed, so the humans who answered can see the result.
- Leave status and assignee untouched.

Then end the iteration. Do **not** plan and do **not** implement — `plan-next` and `implement-next` take it from here.
