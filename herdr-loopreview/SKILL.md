---
name: herdr-loopreview
description: Use when you are working inside herdr (HERDR_ENV=1) with the loopkeep.loopreview plugin installed and you want the human to SEE a diff review of your work — open or switch the tab's single loopreview (lr) pane to your worktree diff, branch diff, or a pull request, so the human watches it in their own tab. Pairs with the loopreview-session skill for steering the review once it is open.
---

# Open a loopreview pane in herdr

You are running inside a herdr pane, and the human is watching your tab. When you
have a change worth showing — a working tree, a branch against its base, or a
pull request — open it in a **loopreview (`lr`)** pane so the human can review it
without leaving their tab.

**You never run the `lr` TUI yourself.** The TUI belongs to the human — do not
run `lr`, `lr diff`, or `lr pr`. Instead, ask the `loopkeep.loopreview` plugin to
open (or reuse) the tab's single named loopreview pane for you, then steer that
review over its control plane with the `loopreview-session` skill.

The plugin keeps exactly one loopreview pane per herdr tab and swaps its contents
in place: each open points that one pane at the new target rather than spawning
more panes. Opening in **your** tab is the point — that is where the human sees
your pane.

## Prerequisites

- `HERDR_ENV=1` — you are inside herdr. (If it is unset, this skill does not
  apply; fall back to asking the human to open `lr` themselves.)
- The plugin is installed: `herdr plugin list` shows `loopkeep.loopreview`.

Resolve the launcher once (its path is reported by herdr):

```bash
PLUGIN_DIR="$(herdr plugin list --json | jq -r '.result.plugins[] | select(.plugin_id=="loopkeep.loopreview") | .plugin_root')"
```

## Opening a review

Run the launcher's headless `open` action from inside your worktree. It resolves
the target repo from your cwd, opens the review in your tab's loopreview pane, and
prints a one-line result; on failure it exits non-zero with the reason on stderr.

```bash
# Branch diff of the current worktree against its base (the default view):
"$PLUGIN_DIR/bin/herdr-loopreview" open

# The plain working-tree diff instead of the branch diff:
"$PLUGIN_DIR/bin/herdr-loopreview" open --view worktree

# A pull request (number, #123, owner/repo#123, or a PR URL):
"$PLUGIN_DIR/bin/herdr-loopreview" open --pr 123

# A worktree other than your cwd (review the worktree containing <dir>):
"$PLUGIN_DIR/bin/herdr-loopreview" open --path /path/to/worktree
```

- `--view` applies only to worktree targets; `--pr` is a pull request and takes
  neither `--view` nor `--path`.
- Check the exit code: non-zero means the open failed (e.g. `lr` not installed,
  not inside a git repository, or an invalid `--pr` value) — read stderr and fix
  the cause before continuing.

## Golden flow

```text
1. finish your change (commit, or leave it in the working tree)
2. "$PLUGIN_DIR/bin/herdr-loopreview" open        # show it in the human's tab
3. lr session ...                                  # steer the review (see below)
4. lr session wait --for reply                     # hand the turn to the human
```

Once the pane is open there is a live `lr` session; steer it — read the diff,
navigate the human's view, leave local notes, and wait for replies — with the
**`loopreview-session`** skill (`lr session review`, `navigate`, `comment add`,
`wait`). Defer to that skill for the control-plane commands; this skill's job is
only to get the review in front of the human.
