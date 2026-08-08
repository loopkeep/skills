---
name: loopkeep
description: >-
  Operate loopkeep from the terminal with the lk CLI: investigate runs (ids like
  run_000123), author workflows and their frontmatter, tune policies, and manage the
  daemon. Use when the user mentions loopkeep, lk, a run id, .loopkeep/ files, or asks
  about workflows, triggers, policies, trust, the inbox, attentions, take-over, or why
  a run stalled, failed, or was blocked.
---

# loopkeep

loopkeep runs agentic workflows (Claude and others) on the user's own machine, gated by
a policy that decides per action whether to auto-approve, notify, ask first, or deny.
The `lk` CLI is the terminal interface to the daemon (`loopkeepd`).

## First steps, always

1. Run `lk --help` — it prints the full command reference, grouped. (A bare `lk` opens
   an interactive terminal UI when stdout is a terminal, so ask for help explicitly.)
   Any command run with no arguments prints its own usage. Trust those over this skill
   when they disagree.
2. Run `lk status` — daemon liveness, workflow/run counts, open attentions, trust
   state, Console connection.
3. Commands target the workspace of the current directory by default; `--workspace
   <id|path>` or `LOOPKEEP_WORKSPACE` retargets them. `lk workspaces` lists them.
   `init`, `trust`, `untrust`, `secret` and `export` always act on the current
   directory and ignore both.

## Vocabulary

- **workspace** — a registered project directory containing `.loopkeep/`.
- **workflow** — a markdown file in `.loopkeep/workflows/` (GitHub Agentic
  Workflows-compatible frontmatter + prompt body) that a **trigger** turns into a
  **run**. Triggers are either local (cron, file-watch, git, run-completed, manual) or
  delivered through the Console (GitHub, Slack, webhooks).
- **run** — one execution (`run_000123`), isolated in a git worktree under
  `.loopkeep/worktrees/`, leaving **checkpoint** commits.
- **headless / attended** — a run's agent is either a subprocess you supervise from the
  inbox, or a terminal multiplexer pane you can watch and type into directly.
- **policy** — layered rules mapping proposed actions to a level: `auto-approve`,
  `notify`, `ask-first`, `deny`. The **floor** is a built-in rule that forces
  `ask-first` on writes to loopkeep's own files; no layer can relax it.
- **trust** — human approval of the current policy (`lk trust`); automatic triggers
  only fire in trusted workspaces. Editing a policy file invalidates trust.
- **inbox / attention** — where runs wait for a human. `lk inbox` lists them,
  `lk inbox <seq>` prints that one's exact change, `lk approve <seq>` decides it.
- **take-over / hand-back** — drive a headless run's agent yourself, then return it.

## The `.loopkeep/` directory

Per project, created by `lk init` (idempotent — it never overwrites what is there):

| Path | What it is |
|---|---|
| `.loopkeep/workflows/*.md` | Workflows. Yours to edit; commit them. |
| `.loopkeep/policy.yaml` | Project policy. Yours to edit; commit it. |
| `.loopkeep/policy.local.yaml` | Your personal overrides for this project. Don't commit. |
| `.loopkeep/config.yaml` | Project defaults for which multiplexer attended runs open in. |
| `.loopkeep/secrets` | The **names** of secrets used here. Values live in the OS keychain. |
| `.loopkeep/trust` | The digest you approved. Written by `lk trust`, never by hand. |
| `.loopkeep/trust.snapshot` | The policy text as trusted, so the app can diff what changed. |
| `.loopkeep/events.db` | The append-only event log. Read-only evidence. |
| `.loopkeep/state/runs/` | Daemon run state. Never edit. |
| `.loopkeep/worktrees/run_000123/` | One run's isolated git worktree. |

`lk init` writes exactly two files: a starter `policy.yaml` (`deploy`-tagged actions
ask first, everything else notifies) and `workflows/nightly.md`.

Machine-wide state lives elsewhere and is not part of any project: `~/.loopkeep/`
holds `global-pause.json`, `disabled-workflows.json` and `pending-flood.json`, and
`~/.config/loopkeep/` holds `config.yaml`, the personal `policy.yaml`,
`workspaces.yaml`, and the home workspace's `events.db`. Don't look for those under a
project.

## Task routing

- Investigate a run (why did it stall/fail/get blocked?) → `references/investigate-runs.md`
- Write or fix workflow frontmatter (`on:`, `x-loopkeep:`, safe-outputs, `{{ … }}`) → `references/workflow-frontmatter.md`
- Create, test and ship a workflow end to end → `references/workflows.md`
- Adjust policy / explain a verdict → `references/policies.md`
- Anything else → the `lk --help` screen; prefer the narrowest command over restarts.

## Safety rules

- Never weaken a policy to unblock an action without telling the user what you are
  relaxing and why. Prefer the narrowest possible rule; verify with `lk policy test`.
- Treat `.loopkeep/events.db` as **read-only evidence**. Never write to it.
- Do not edit `.loopkeep/state/`, `.loopkeep/trust`, `.loopkeep/trust.snapshot`, or
  anything under `.loopkeep/worktrees/` by hand; use `lk` commands. The floor forces
  an approval prompt on those writes anyway, so hand-editing them just interrupts
  somebody.
- `lk stop` ends in-flight runs and marks them interrupted; they come back resumable
  on the next start. Say so before proposing a restart while runs are active.
- Never call `lk trust` on the user's behalf as a step in your own work. It is the
  human's approval of what the policy currently says.
