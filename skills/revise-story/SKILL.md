---
name: revise-story
description: Revise a story whose implementation was sent back — a reviewer rejected or commented on its PR and a human moved the story back to Plan Approved. Finds the story's existing feature branches and PRs, gathers the unaddressed review feedback, fixes the existing implementation, pushes, replies to the feedback and returns the story to In Review. Loaded by implement-next when the claimed story already has a branch or PR; also takes an issue key to revise a specific story.
---

# Revise a story

The story already has an implementation under review that was sent back. Fix that implementation; do not start over.

## 1. Claim

When `implement-next` handed the story over, it is already claimed and read — skip to step 2. With an issue key argument, do `implement-next` steps 1–3 for that key first.

## 2. Find the existing work and its feedback

- Check out the story's existing branches (`feature/<KEY>`) and merge the latest default branch into each. Merge rather than rebase: the reviewer has already seen these commits.
- Find the story's PRs (e.g. `gh pr list --head feature/<KEY> --state all`, `az repos pr list --source-branch feature/<KEY> --status all`).
- Gather the feedback that has not been addressed yet: review threads and comments on the PRs, and the story's comments, newer than the last push. Weigh them as `plan-next`'s *Reading a refinement thread* describes.

If you cannot tell why the story was sent back, comment on the story asking, leave it In Progress, and stop.

## 3. Fix

Follow `implement-next` steps 6–8 with the feedback as the specification for this iteration, correcting `plans/<KEY>.md` where the feedback contradicts it. The cold reviewer also gets the feedback and the branch's full diff against the default branch, and checks that each item was addressed.

## 4. Push and hand back

1. Commit and push the existing branches.
2. Reply to each piece of feedback with how it was addressed, or why not. Reopen a closed PR, or open a new one referencing it if it cannot be reopened.
3. Transition the story to `In Review`.

Then do `implement-next` step 10.
