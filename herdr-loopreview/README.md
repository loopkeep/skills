# herdr-loopreview skill

Agent skill for opening a review in front of the human from inside herdr: point
the tab's single loopreview (`lr`) pane at your worktree diff, branch diff, or a
pull request via the `loopkeep.loopreview` plugin's headless `open` action. Pairs
with the `loopreview-session` skill, which steers the review once it is open.

`SKILL.md` here is the source of truth, mirrored to the public `loopkeep/skills`
repo from the `herdr-plugin-loopreview` repository on each release — so the
published skill always matches a released plugin binary.

Install just this skill:

```sh
npx skills add loopkeep/skills -s herdr-loopreview
```
