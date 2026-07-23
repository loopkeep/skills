# loopkeep skills

Agent skills for the loopkeep family of tools, one directory per skill:

- [`loopkeep/`](loopkeep) — operate [loopkeep](https://github.com/loopkeep/loopkeep)
  from coding agents: investigate runs, author workflows, tune policies, and use
  the `lk` CLI.
- [`loopreview/`](loopreview) — steer a live
  [loopreview](https://github.com/loopkeep/loopreview) (`lr`) diff-review
  session: read the diff, navigate the reviewer's view, leave draft comments.
- [`herdr-loopreview/`](herdr-loopreview) — from an agent pane inside
  [herdr](https://herdr.dev), open or switch the tab's loopreview pane to your
  worktree, branch diff, or PR via the
  [herdr-plugin-loopreview](https://github.com/loopkeep/herdr-plugin-loopreview)
  plugin, so the human sees the review; pairs with `loopreview/` for steering it.

**Status: pre-release.** Each skill directory is mirrored from its tool's
repository on release; only this README and the root LICENSE are edited here
by hand.

Once published, install with:

```sh
npx skills add loopkeep/skills                        # pick interactively
npx skills add loopkeep/skills -s loopkeep            # just the lk skill
npx skills add loopkeep/skills -s loopreview-session  # just the lr skill
npx skills add loopkeep/skills -s herdr-loopreview    # just the herdr pane opener
```
