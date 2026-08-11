# Authoring workflows

A workflow is one markdown file in `.loopkeep/workflows/<name>.md`: YAML frontmatter
(GitHub Agentic Workflows-compatible) + the prompt body. The file name without `.md`
is the workflow name everything else refers to.

**Every frontmatter key, trigger, and `{{ … }}` token is in
`workflow-frontmatter.md`.** This page is the loop around it: getting a new workflow
running, iterating on it, and moving files in and out of gh-aw.

## Writing the prompt body

The body is instructions to an agent that will not have you sitting next to it. State
the goal, the exact commands or tools to prefer, the hard prohibitions, and how to
report. Two habits matter more than the rest:

- **Give it a way to reach you.** An agent that hits an ambiguous fork with no
  `ask_human` either guesses or stalls. `{{ call:mcp:loopkeep/ask_human }}` in the body
  makes asking a first-class option; `notify_human` covers "you should know this
  happened" without blocking.
- **Say how it ends, once.** Exactly one closing report — a `notify`, or an
  `emit_output` payload downstream workflows read. A workflow that reports three times
  trains its human to stop reading the inbox.

Read `{{ trigger }}` content as untrusted input. Anyone who can open an issue or post
in a channel can put text in it, and that text arrives in the same prompt as your
instructions.

## Getting one running

1. Start from the closest existing workflow — `workflows/examples/` in the loopkeep
   repository, or the files already in this workspace's `.loopkeep/workflows/`.
2. `lk run <name>`, then watch it: `lk runs`, `lk inbox`. This works on any workflow
   file that exists, trusted or not, so the iteration loop needs no approval. Add
   `manual: {}` to `on:` as well if a person should be able to start it from the
   desktop app or the Console — that is what puts a Run button on it.
3. `lk trigger explain <name>` — confirms what registered, which repository a GitHub
   trigger bound to, and what `me` resolved to. Anything that was refused with a
   warning shows up here rather than in the run.
4. Automatic triggers only fire once the workspace is trusted. That is the human's
   call: show them the policy and let them run `lk trust`.
5. Iterate with `lk rerun <run_id>` — it reuses the original trigger context, so
   trigger-shaped bugs reproduce instead of vanishing.

## Chaining workflows

Runs never talk to each other directly. There are three channels:

1. **Run output** — the agent calls `emit_output` with JSON; a downstream workflow's
   `run-completed` trigger filters on it (`if: event.output.severity == "high"`) and
   receives it as `{{ trigger.output }}`.
2. **Dispatch** — an agent fires another workflow through `dispatch-workflow`. The
   target must be named in that output's `allowed` list, and the dispatch is
   policy-evaluated like any other action.
3. **The filesystem** — write a file, let a downstream `file-watch` pick it up. Best
   for multi-stage pipelines, because each stage stays independently runnable.

Chains carry a depth: a `run-completed` or `dispatch-workflow` firing inherits its
parent's depth plus one, capped at 5. Deeper firings are refused and recorded. That
depth limit is the entire loop-prevention model — there is no DAG to configure.

**When a chain mysteriously stops, check budgets and depth before suspecting the
trigger.** `lk runs` prints `depth=` per run, and a budget refusal is recorded as a
failed run rather than as silence.

## Operating an existing workflow

- `lk disable <workflow>` / `lk enable <workflow>` — turn its automatic triggers off
  and on in this workspace. Manual runs and runs in flight are unaffected.
- `lk pause --all` / `lk resume --all` — persisted global trigger gate across every
  workspace. Future automatic firings stop; manual and in-flight runs continue. It is
  independent of trust and does not kill the daemon, terminals, or agent sessions.
- `lk untrust` — this workspace's trigger gate: every automatic trigger here stops
  firing until someone runs `lk trust` again.
- `lk hooks` / `lk hooks add <name>` / `lk hooks rotate <name>` / `lk hooks rm <name>` —
  the personal webhook URLs `on.webhook` listens to. The URL is the credential and is
  printed only when it is issued; `lk hooks` lists names and dates, never URLs.

## GitHub Agentic Workflows interop

`lk export --to-gh-aw <workflow> [--json]` translates a workflow to gh-aw and reports
each key as green (same meaning), yellow (translated, meaning changed), or red (no
equivalent, feature lost). The workflow can be a name under `.loopkeep/workflows/` or a
path to a file. Local triggers, local safe-outputs, and everything under `x-loopkeep`
have no gh-aw equivalent, so expect red and yellow rows for anything loopkeep-specific.

`lk import <gh-aw-file> [--name <name>] [--force]` writes a gh-aw file into the
workspace as `.loopkeep/workflows/<name>.md`, named after the source file unless
`--name` says otherwise, and adds a `manual` trigger so it can always be started by
hand. The file's own triggers are kept as written — a trigger that doesn't fire locally
is not swapped for `manual`. An existing workflow is an error rather than a silent
overwrite; `--force` replaces it. It prints the path it wrote, any warnings, and the
resulting `on:`.
