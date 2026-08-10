---
name: refine-story
description: Run one refinement iteration of the microfactory — pick an item that is too vague to plan or implement, resolve everything the codebase already answers, post the few genuine product questions left as a comment on the task, and stop. Runs unattended; intended as a /loop target (e.g. /loop 30m /microfactory:refine-story); also takes an issue key to refine a specific item.
---

# Refine the next story

One refinement iteration: pick an item → answer what the code answers → **post** the remaining questions as a comment → stop.

This skill runs **unattended**. Never use `AskUserQuestion` and never wait for a reply — the human answers in the task backend, in their own time. All communication goes through issue comments, because the backend is the single source of truth.

**This skill never edits the item.** It asks; the product owner decides. The questions and the answers live in the comment thread, and `plan-next` and `implement-next` read that thread when they claim the item. Rewriting somebody's story into decisions is not this skill's job, and neither is deciding when a story is ready — a human removes the `needs-refinement` tag when they are satisfied with the answers.

Act on **at most one item per iteration**.

## 1. Read configuration and pick an item

Read `.microfactory/config.yaml`; if it does not exist, tell the user to run `/microfactory:init-factory` and stop. Load the backend skill matching `backend` — you need its **view**, **list comments**, and **comment** operations, and nothing else.

**With an issue key argument**: work that item.

**Without one**: list open, unassigned items that are eligible for refinement, in backlog order —

- tagged `needs-refinement`, or
- items whose entire specification is their title (no body at all).

Refinement never assigns the item, never transitions it, and never updates it. An assigned item is not claimable by `plan-next` or `implement-next`, and refining is not claiming. The questions comment is what tells another session this item has been handled.

## 2. Skip anything already asked

Fetch the item and **its comments**. A questions comment posted by this skill always opens with the marker line `**Refinement questions**` and closes with the `_Posted by microfactory refine-story…_` footer.

**If that marker is there, skip the item** — asked is asked, whether or not anybody has replied yet. (Given an explicit issue key, say the item has already been asked, point at the existing comment, and stop; do not post a second round.) Re-asking a question a human already answered wastes their attention, and asking a second round on top of an unanswered first one buries the questions that mattered. One round per item is the whole of this skill's budget; anything the round did not settle is the product owner's to pursue.

Rely on the marker, not on comment authorship — the todo backend records no authors, and on a shared Jira or GitHub project a teammate's reply may come from any account. Ignore comments other factory skills left (plan links, implementation notes).

If every eligible item already carries the marker, say "No refinement work available." and end the iteration — the humans have the ball, and the loop cadence handles waiting.

## 3. Answer what you can yourself

**Never ask a human something the repository already answers.** Before drafting a single question:

- Read the story as written, and anything it links to.
- Read the code it would touch, the conventions the project documents, and — most useful — a feature already built that resembles it. Existing precedent settles most "how should this behave?" questions.
- Look at the data model, the validation and error handling already in use, and how similar screens or endpoints behave.

Keep a note of what you concluded and what you concluded it from. These are not questions — they go into the comment as a short list of what you took as given, so the product owner can correct any of them in the same reply.

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

- The answer is **obvious** — one answer is what any reasonable person would say, or the project has already done it that way five times. Take the obvious answer and list it under what you took as given.
- The answer **would not change what gets built**. That is curiosity, not refinement.
- It is **cheaper to discover during implementation** than to resolve now.
- A convention, an existing feature, or the story's own text already settles it.

**Asking nothing is a perfectly good outcome.** If nothing survives the prune, still post a comment — the marker line, what you took as given, and one line saying no human input is needed. That records the reasoning, closes the item to further refinement rounds, and tells the product owner they can drop the tag.

**Cap it at 4 questions.** If more survive, keep the four whose answers change the most and drop the rest — earlier answers usually kill later questions anyway, and there is no second round. A story with two open trivia and a clear goal is refined; twelve questions is an interrogation, and a worse outcome than a slightly vague story.

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
Skip anything you have no opinion on — the default applies. Correct anything wrong under
"Taken as given" the same way. Drop the `needs-refinement` tag when this is settled enough to build.

**Taken as given** (from the code — say so if any of these is wrong):
- Invites are sent by email only; there is no invite link to share (matches `ProjectInvite`).
- An invite expires after 7 days, like every other token in the project.

**1.** Should an expired invite link show the sign-up page with an "expired" notice, rather than a 404?
*Default: yes.*

**2.** What does the member list show for a team with nobody in it yet?
  **a.** Empty state with an "Invite someone" button — matches the projects list *(recommended, default)*
  **b.** The list with only the current user in it
  **c.** Hide the list until there are two members — fewer states, but no obvious way to invite

_Posted by microfactory refine-story. Answering here unblocks this story._
```

Leave out **Taken as given** when there is nothing worth stating, and leave out the questions when none survived the prune — the marker line and the footer are the only fixed parts.

Then end the iteration. Do not plan, do not implement, do not touch the item — an unanswered question is not a decision, and the story belongs to whoever wrote it. The product owner answers in the thread and removes `needs-refinement` when they are satisfied; `plan-next` and `implement-next` read the thread when they claim the item.
