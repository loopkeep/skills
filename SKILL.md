---
name: loopkeep
description: >-
  Operate loopkeep from the terminal with the lk CLI: investigate runs (ids like
  run_000123), author workflows, tune policies, and manage the daemon. Use when the
  user mentions loopkeep, lk, a run id, .loopkeep/ files, or asks about workflows,
  triggers, policies, trust, the inbox, attentions, or why a run stalled, failed,
  or was blocked.
---

# loopkeep

loopkeep runs agentic workflows (Claude and others) on the user's own machine, gated by
a policy that decides per action whether to auto-approve, notify, ask first, or deny.
The `lk` CLI is the terminal interface to the daemon (`loopkeepd`).

## First steps, always

1. Run `lk` with no arguments — it prints the full command reference. Subcommands do
   not have `--help`; this one screen is the whole surface. Trust it over this skill
   when they disagree.
2. Run `lk status` — daemon liveness, workflow/run counts, open attentions, trust
   state, Console connection.
3. Commands target the workspace of the current directory by default. Use
   `lk --workspace <id|path> ...` to target another one; `lk workspaces` lists them.

## Vocabulary

- **workspace** — a registered project directory containing `.loopkeep/`.
- **workflow** — a markdown file in `.loopkeep/workflows/` (GitHub Agentic
  Workflows-compatible frontmatter
  + prompt body) that a **trigger** (cron, manual, webhook, …) turns into a **run**.
- **run** — one execution (`run_000123`), isolated in a git worktree under
  `.loopkeep/worktrees/`, leaving **checkpoint** commits.
- **policy** — layered rules mapping proposed actions to a level: `auto-approve`,
  `notify`, `ask-first`, `deny`. The **floor** is the built-in set of
  deny rules no policy layer can relax.
- **trust** — human approval of the current policy (`lk trust`); automatic triggers
  only fire in trusted workspaces. Editing policy invalidates trust.
- **inbox / attention** — where runs wait for a human (`ask-first` and friends);
  `lk inbox`, `lk approve <seq>`.
- **take-over / hand-back** — drive a run's agent interactively, then return it.

## Task routing

- Investigate a run (why did it stall/fail/get blocked?) → `references/investigate-runs.md`
- Create or edit a workflow → `references/workflows.md`
- Adjust policy / explain a verdict → `references/policies.md`
- Anything else → the `lk` help screen; prefer the narrowest command over restarts.

## Safety rules

- Never weaken a policy to unblock an action without telling the user what you are
  relaxing and why. Prefer the narrowest possible rule; verify with `lk policy test`.
- Treat `.loopkeep/events.db` as **read-only evidence**. Never write to it.
- Do not edit files under `.loopkeep/state/`, `.loopkeep/trust`, or
  `.loopkeep/worktrees/` by hand; use `lk` commands.
- `lk stop` restores in-flight runs as `interrupted` on the next start — say so
  before proposing a restart while runs are active.
