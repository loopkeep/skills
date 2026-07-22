# loopkeep skills

Agent skills for the loopkeep family of tools, one directory per skill:

- [`loopkeep/`](loopkeep) — operate loopkeep from coding agents: investigate
  runs, author workflows, tune policies, and use the `lk` CLI.
- [`loopreview/`](loopreview) — steer a live loopreview (`lr`) diff-review
  session: read the diff, navigate the reviewer's view, leave draft comments.

**Status: pre-release.** Each skill directory is mirrored from its tool's
repository on release; only this README and the root LICENSE are edited here
by hand.

Once published, install with:

```sh
npx skills add loopkeep/skills                        # pick interactively
npx skills add loopkeep/skills -s loopkeep            # just the lk skill
npx skills add loopkeep/skills -s loopreview-session  # just the lr skill
```
