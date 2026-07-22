# Authoring workflows

A workflow is one markdown file in `.loopkeep/workflows/<name>.md`: YAML frontmatter
(GitHub Agentic Workflows-compatible) + the prompt body.

## Anatomy

```markdown
---
on:
  manual: {}                 # always allow `lk run <name>`
  schedule:
    - cron: "0 9 * * *"      # minute granularity
engine:
  id: claude                 # or `api` with an explicit model id
  model: haiku
x-loopkeep.budget:
  daily_runs: 3
  daily_tokens: 200000
x-loopkeep.concurrency:
  max: 1
  on_limit: skip             # don't pile up behind a slow previous run
safe-outputs:                # the only side-channels the run may use to publish
  add-labels: {}
  add-comment: {}
  emit-output: {}            # structured JSON for downstream workflows to chain on
  notify: {}
---
Prompt body, with interpolations — see the next section.
```

Study the shipped examples before writing from scratch — `workflows/examples/` in the
loopkeep repository, or existing files in this workspace's `.loopkeep/workflows/`.

## Interpolations (`{{ … }}` in the body)

Exactly three token kinds are expanded; any other `{{ … }}` is left untouched.

- `{{ trigger }}` — replaced with the trigger context (event payload, schedule id).
  It is a **placement marker, not an opt-in**: if the body has no `{{ trigger }}`,
  the trigger context is prepended to the top of the prompt anyway. Write it only to
  control where the context lands.
- `{{ call:<canonical-tool> }}` — reference a tool by canonical name. It expands to a
  short inline instruction (`call <tool>`), and for the three built-in loopkeep tools
  a usage snippet is appended to the end of the prompt once per tool:
  - `mcp:loopkeep/ask_human` — block and ask a question via the inbox
  - `mcp:loopkeep/notify_human` — post a non-blocking notification to the inbox
  - `mcp:loopkeep/emit_output` — publish structured JSON for downstream workflows
    (pair with `safe-outputs: emit-output`)
  Unknown targets are not expanded and the editor flags them as likely typos — the
  spelling must be the canonical form (`mcp:<server>/<tool>` for MCP tools).
- `{{ secret:<name> }}` — expands to the reference form `secret:<name>`; the value is
  **never inlined into the prompt**. Manage values with `lk secret set` /
  `lk secret list`.

## Authoring flow

1. Start from the closest existing workflow or example; keep names kebab-case.
2. Write the prompt as instructions to an agent that cannot ask questions mid-run:
   state the goal, the exact commands or tools to prefer, hard prohibitions, and how
   to report (`notify` / `emit-output`), ideally exactly once.
3. Test by hand first: `lk run <name>`, then `lk runs` and the inbox. Manual runs
   work regardless of trust.
4. Automatic triggers fire only once the workspace is trusted (`lk trust`).
5. Iterate with `lk rerun <run_id>` — it reuses the trigger context, so trigger-shaped
   bugs reproduce.

## Operating notes

- `lk disable <workflow>` / `lk enable <workflow>` — toggle automatic triggers per
  workspace; in-flight and manual runs are unaffected.
- `lk pause --all` / `lk resume --all` — global stop for trigger-driven runs.
- Budgets and `depth`: chained workflows (one run's `emit-output` triggering another)
  are cut off by chain depth and budgets. If a chain "mysteriously" stops, check
  budgets before suspecting the trigger.
- GitHub Agentic Workflows interop: `lk export --to-gh-aw <workflow> [--json]` reports compatibility as
  green/yellow/red; `lk import <gh-aw-file>` converts, and triggers it cannot run
  become `manual` instead of failing.
