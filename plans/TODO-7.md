# TODO-7: create a skill that installs the agentize plugin (github.com/jaksa76/agentize)

## Summary

Add a `install-agentize` skill that installs the [agentize](https://github.com/jaksa76/agentize)
plugin — skills for assessing and improving a project's agent readiness — into the
user's Claude Code installation.

`github.com/jaksa76/agentize` is a **marketplace** repo (`.claude-plugin/marketplace.json`,
marketplace name `agentize`) containing a single plugin at `plugins/agentize`
(plugin name `agentize`), so the install target is `agentize@agentize`.

Its README documents the interactive route (`/plugin marketplace add …`, `/plugin install …`),
which an agent cannot execute — `/plugin` is a built-in REPL command, not a shell command.
The skill therefore uses the non-interactive CLI, which does the same thing:

```
claude plugin marketplace add jaksa76/agentize
claude plugin install agentize@agentize --scope user
```

Scope choice: `user` (all of the user's projects) by default, since the agentize skills are
assessment tools the user runs, not project infrastructure. `project` scope is mentioned as
the alternative for teams that want it committed with the repo.

## Files to Change

| File | Change |
|---|---|
| `skills/install-agentize/SKILL.md` | New skill: check, add marketplace, install, verify, report |
| `CLAUDE.md` | Add the skill to the repository-layout listing |
| `docs/ARCHITECTURE.md` | Add the skill to the repository-layout listing |
| `README.md` | Mention the skill where the other convenience skills are listed |

## Implementation Steps

1. Write `skills/install-agentize/SKILL.md` with frontmatter (`name`, `description` phrased so
   the model loads it on "install agentize" / "assess agent readiness" style requests) and steps:
   1. **Check** — `claude plugin list`; if `agentize` is already installed, say so and offer
      `claude plugin update agentize` instead of reinstalling.
   2. **Add the marketplace** — `claude plugin marketplace add jaksa76/agentize`; treat an
      "already exists" failure as success (idempotent), and `claude plugin marketplace list`
      to confirm.
   3. **Install** — `claude plugin install agentize@agentize --scope user` (note `--scope project`
      as the shared alternative).
   4. **Verify and report** — `claude plugin list` shows agentize; tell the user the skills load
      in the **next** session (restart required) and name the entry point
      `/agentize:assess-readiness`, plus `/agentize:verify-c<N>-<M>` for a single criterion.
2. Note in the skill that the repo also ships unpackaged skills under `.claude/skills/` that the
   plugin does not include, with the README's copy-based install as an optional extra.
3. Update the three doc files listed above.

## Testing

No automated suite in this repo (see CLAUDE.md). Verification:

- Structural: frontmatter `name` matches the directory, single `description` line — same check
  the other skills pass.
- Command-level: confirm `claude plugin marketplace add`, `claude plugin install`, and
  `claude plugin list` exist with the flags used (`--scope`) via `claude plugin … --help`.
- Do **not** run the install as part of the iteration — it mutates the user's `~/.claude`
  installation state, which is the user's call, not the loop's.

## Risks / Edge Cases

- **Marketplace already added** — `add` fails; the skill must tolerate it rather than abort.
- **Already installed** — reinstalling is wasteful; branch to `update`.
- **Restart requirement** — plugins load at session start, so the freshly installed skills are
  not invocable in the session that installed them. Must be stated or the user will think it failed.
- **CLI availability** — `claude plugin` subcommands exist in current Claude Code; if the CLI is
  older, fall back to telling the user to run the two `/plugin` commands interactively.
- Scope flag defaults to `user` in the CLI, but pass it explicitly so the behavior is obvious.
