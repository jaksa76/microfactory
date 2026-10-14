---
name: customize
description: Add, change, list, explain, extract and remove the project's microfactory override files (.microfactory/overrides/general.md and <skill>.md) from a plain-language request such as "always run make e2e-smoke after implementing". Interactive — shows the exact change and asks before writing; checks each override against the factory's invariants and the real step titles. Use when the user wants to add extra instructions for a microfactory skill, or asks what an existing override does or why one applies or is being ignored. Not run by loops; not for config.yaml settings (that is init-factory) or for general questions about how the factory works.
---

# Customize the factory

The project's overrides are plain markdown under `.microfactory/overrides/` that the workflow skills read unattended. This skill is how a person maintains them without knowing the precedence rules, the invariants or the step titles — what `init-factory` is to `config.yaml`. Its one job is **the override files**: it never runs a factory iteration, never edits a skill, never touches `config.yaml`, and **never commits** (override files are project policy for unattended loops; the diff deserves a human's review like any other change).

## 1. Load the rules

Every other skill of the plugin is a sibling of this one: `<this skill's base directory>/../<skill>/SKILL.md`. Use that, not the project's tree — the plugin may be installed anywhere.

- Read `../overrides/SKILL.md`. It is the single source of truth for the file locations, the `## <skill>` / `## all` scoping, the precedence order, the invariants an override can never weaken, the list of skills that read overrides, and what belongs in overrides rather than elsewhere. **Do not copy its rules from memory or restate them from here**; apply what it says. If it cannot be read, you cannot check anything safely: say you could not find the `overrides` skill next to this one (an older plugin version, or an agent that does not tell a skill where it lives) and stop.
- The **real step titles** of a skill come from the `##` headings of `../<skill>/SKILL.md`, read only for the skills this request touches. If that file cannot be read, do not guess and do not drop the step: keep the user's own wording of the step in the rule, mark it unverified in what you show, and say why.
- The **config keys** a request might really be a setting for are the keys in the config example in `../init-factory/SKILL.md`; read them there each time rather than remembering them.

## 2. Work out what is asked

Infer the mode from the request; ask once if it is genuinely ambiguous.

| Request | Mode |
|---|---|
| A rule or behaviour to add or change ("always run…", "plans must include…", "never touch…") | **add / change** |
| `list [skill]` — what is overridden | **list** |
| `explain [skill \| "<text>"]` — what an override does, or why the factory does something | **explain** |
| `remove <skill \| "<text>">` | **remove** |
| `extract <skill>` — move a section of `general.md` into `<skill>.md` | **extract** |

## Add or change

1. **Find the target.** A rule for the whole factory is general; otherwise it belongs to one skill from the `overrides` skill's list. If more than one skill fits ("run the linter" could be `implement-next` or the finders), ask once and default to the narrowest. A skill that does not read overrides (`tasks`, `start`, `init-factory`, ...) is not a target: say so and list the ones that are.
2. **Run the checks.** Each failure is shown to the user with its reason, never silently fixed:
   - **Invariants.** A request that weakens one of those the `overrides` skill lists ("skip the review", "commit before verifying", "write the token into the plan") is refused with the reason, and the closest permitted alternative is offered. Write nothing for the refused part.
   - **Unattended.** Judged by where the rule will run, using the `overrides` skill's rule. A rule that makes a loop target (`plan-next`, `implement-next`, `refine-story`) stop and wait for the user ("ask me before pushing") is flagged — and so is a *general* rule of that kind, because it reaches the loop targets too. `breakdown-feature` and `find-improvements` are run by a person, so a rule that adds to their conversation is fine; the finders have none. Offer a rewrite that works unattended — "post the question as a comment on the issue and stop" — or refuse.
   - **Real steps.** A step the request names must be a heading in that skill, or in a skill it borrows steps from (`implement-next` borrows `plan-next`'s), matched by title ignoring the number and case; otherwise show the real titles and ask which is meant. Write the step by title (`After *Implement*, …`), never by number. Step titles repeat across skills, so a rule that names a step is never general: put it with the one skill whose step it is. If the user asked for a scope the rule cannot have ("for every skill" with a rule about the *Implement* step), do not narrow it yourself — say why, ask which skill is meant, and with no one to ask write nothing. Restating it without a step name so it stays general is an option to offer, not to take.
   - **Settings first.** A plain on/off or a value that a config key already controls (read the keys, see step 1) is answered by naming that key and pointing at `/microfactory:init-factory`; write no prose override for it.
   - **Belongs elsewhere.** What the `overrides` skill says does not belong in overrides — how the project builds, tests or is laid out ("our tests run with `pytest -x`"), lessons — is answered by pointing at the project's `CLAUDE.md` instead. Overrides are for how the *factory* behaves.
   - **Contradictions and duplicates.** Compare with the existing text **including precedence, in both directions**. If a more specific line already overrides the new one — for a general rule, any `<skill>.md` or `## <skill>` section that contradicts it — show that line and do not write the losing text. If a line at the *same* level contradicts the new one (same file and section), what happens depends on what the user asked. A request that **changes an existing rule** ("change the smoke command to…") replaces that line: show old and new, and never leave the two side by side. A **new** rule that contradicts a line the user did not mention never deletes it: show both lines and ask which wins; with no one to ask, write nothing. Removing a rule the user did not name is not covered by "apply without asking". If the rule is already there, say so and write nothing.
3. **Place it.** A rule for every skill goes at the top of `general.md`, or under `## all`. A rule for one skill goes in that skill's `<skill>.md` if the file exists, so a skill never has its rules in two files; otherwise in a `## <skill>` section of `general.md`, created if needed. Create the directory and `general.md` as required — the "never create" rule in the `overrides` skill binds the skills that read them, not this one — and put no placeholder text in a file you create.
4. **Draft** one rule per line or short paragraph, imperative, specific to a step by title where it matters, and without the reason for the change — an override is an instruction, not a discussion. Say what the user asked and no more: a requirement you think is missing ("and fix any failures") is offered as a separate suggestion in step 5, not added to the line.
5. **Show and confirm** (see *Approval*).
6. **Offer extraction** when the `## <skill>` section you wrote into has passed about fifteen lines, or `general.md` about sixty (see *Extract*).
7. **Close** with `git status` and a one-line reminder that the change is not committed.

## Approval

Add, remove and extract all work the same way. Show the exact lines that will change and the file and section they are in, ask (AskUserQuestion: apply / edit / cancel), and write only after approval. If the user's own request already says to apply it without asking, that approves **what they asked for, as drafted** — it does not cover an alternative or rewrite you offered, and it never waives the checks above: a refused part stays refused. When there is no one to ask and the request does not say to apply, print the change and write nothing.

## List

For each override give the file and section, the skill it applies to, and the rule in one line. Group by skill, general rules first. With a skill argument, show only what applies to that skill, in precedence order. Also list what **no skill will ever read**: a `##` section of `general.md` named neither `all` nor a skill that reads overrides (a typo such as `## implement_next` included), and a `<skill>.md` for a skill that does not read overrides or whose name is misspelled. No files means no overrides: say so and mention `/microfactory:customize <what you want>`.

## Explain

Read-only, and the way to answer "why is the factory doing this?". For each override (or only those matching the argument), report:

- the **skill and step** it changes, by real title (read the skill's headings; for a step the skill does not have, say so);
- what the skill does **without** it, quoted briefly from that step;
- the **file and section** it comes from;
- how it **interacts** — which of several overrides wins by precedence, which invariant bounds it, and that a `plan-next` override also applies when `implement-next` runs `plan-next`'s steps (the `overrides` skill's rule for borrowed steps);
- whether it is **being ignored**, and why: it weakens an invariant, names a step that does not exist, needs a human in an unattended run — or is never read at all, because it sits in a `##` section named neither `all` nor a skill, or in a `<skill>.md` for a skill that does not read overrides (the *List* case). A section that is never read is not live, however it looks.

`explain <skill>` also prints the **effective instruction set**: what the skill will follow after precedence is applied, ignored overrides excluded. Say the explanation is *as written* — it comes from reading the skill and the overrides, and whether an agent obeys an instruction is only visible in a run.

## Remove

Remove only what the user described, and show it first (*Approval*), including the section or file that becomes empty. A description that matches several lines or places is shown in full and the user picks; with no one to ask, list the matches and remove nothing. `remove <skill>` means that skill's own rules — its `<skill>.md` and its `## <skill>` section — not general rules that merely mention it. An emptied `<skill>.md` is deleted; an emptied `## <skill>` section is removed with its heading.

## Extract

Move a `## <skill>` section of `general.md` into `.microfactory/overrides/<skill>.md` (*Approval*), unchanged apart from dropping the heading. The skill must be one that reads overrides, and the file is named with the skill's exact lowercase name even if the heading was spelled `## \`Implement-Next\``; `all` and unknown names are refused. If `<skill>.md` already exists, append below its present content and flag any line that now contradicts one already there. Leave `general.md` without the section.

## Boundaries

Settings belong to `init-factory`; this skill sends them there. It does not decide what is *good* policy — only that the override is expressible, safe and says what the user meant.
