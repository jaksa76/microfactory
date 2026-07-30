---
name: install-agentize
description: Install the agentize plugin (github.com/jaksa76/agentize) — skills that assess and improve a project's agent readiness and adoption. Use when the user wants to install agentize, assess agent readiness, or check how ready this project is for agentic coding.
---

# Install the agentize plugin

[agentize](https://github.com/jaksa76/agentize) is a marketplace repo (marketplace name `agentize`) holding one plugin (`agentize`), so the install target is `agentize@agentize`.

Its README documents the interactive route (`/plugin marketplace add …`, `/plugin install …`). Those are REPL commands you cannot run yourself — use the `claude plugin` CLI below, which does the same thing. Commands are examples; adapt them to the user's OS and shell.

## 1. Check what is already there

```
claude plugin list
```

If `agentize` is already installed, do not reinstall it — offer `claude plugin update agentize` instead and skip to step 4.

## 2. Add the marketplace

```
claude plugin marketplace add jaksa76/agentize
```

If this fails because the marketplace is already configured, that is success — carry on. `claude plugin marketplace list` confirms it is there.

## 3. Install the plugin

```
claude plugin install agentize@agentize --scope user
```

`user` scope makes the skills available in all of the user's projects, which suits assessment tools they invoke by hand. Use `--scope project` instead when the user wants it committed with the repo so their team gets it too — ask if it is not obvious which they want.

## 4. Verify and report

Confirm with `claude plugin list`, then tell the user:

- The skills load at session start, so they must **restart Claude Code** before invoking them — they are not available in this session.
- The entry point is `/agentize:assess-readiness` for a full assessment, or `/agentize:verify-c<N>-<M>` for a single criterion (e.g. `/agentize:verify-c5-2` for unit test coverage).

If the `claude plugin` subcommands are unavailable (older CLI), fall back to telling the user to run the two interactive commands themselves:

```
/plugin marketplace add jaksa76/agentize
/plugin install agentize@agentize
```

## Note on the unpackaged skills

The agentize repo also ships work-in-progress skills under `.claude/skills/` that the plugin does not include (adoption assessment, readiness improvement). They install by copying, per the repo's README — mention this only if the user asks for more than the readiness assessment.
