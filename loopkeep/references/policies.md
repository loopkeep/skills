# Tuning policy

Policy maps each proposed action (tool + paths + tags + command) to a level:

`auto-approve` → `notify` → `ask-first` → `deny`

Layers stack (home `~/.config/loopkeep/policy.yaml`, workspace
`.loopkeep/policy.yaml`, per-workflow rules). The **floor** — built-in deny rules for
genuinely destructive actions — sits under everything and cannot be relaxed by any
layer, including during take-over.

## Explain and test (local, no daemon needed)

```sh
lk policy explain <tool> [paths...] [--tags a,b] [--command <text>] \
                  [--self <level>] [--workflow <name>]
lk policy test    <tool> --expect <level> [...]      # verdict via exit code
```

`explain` prints the matched rule chain — each line shows the layer, the source
`file:line`, and the level it contributed, then the `final` verdict. Note: rules that
need a live run (the running workflow's own definition, paths the run protects at
runtime) are not evaluated in this stateless mode; a clamp seen in a run's event log
can therefore be stricter than what `explain` reproduces.

## Tuning flow

1. Reproduce the verdict: take `tool` / `raw_command` / paths / tags from the run's
   `policy.clamped` or `attention.requested` event and feed them to
   `lk policy explain`.
2. Read the matched chain to find which layer and line produced the level.
3. Edit the narrowest layer that owns the decision — usually the workspace
   `.loopkeep/policy.yaml`. Prefer adding a narrow rule (specific tool + path/tag)
   over changing `defaults:`.
4. Verify both directions with exit codes:
   `lk policy test <tool> ... --expect <level>` for the case you fixed, and a second
   `test` proving you did not relax something unrelated.
5. Re-approve: editing policy invalidates trust, so finish with `lk trust` —
   otherwise automatic triggers stay off.

## Guardrails

- Relaxing review of write/exec/destructive actions is a human decision: propose the
  diff of `policy.yaml` and say what becomes more permissive before applying it.
- If a level seems ignored, check the direction of the stack: layers can tighten a
  verdict; only the owner of the matched rule can loosen it, and nothing loosens the
  floor.
- `--self <level>` simulates what the agent self-reported; use it to check that
  optimistic self-reports get clamped where you expect.
