# Tuning policy

Policy maps each proposed action (tool + paths + tags + command + workflow) to a level:

`auto-approve` → `notify` → `ask-first` → `deny`

## The layers

Four layers, highest precedence first:

| Layer | File | Scope |
|---|---|---|
| project-local | `<ws>/.loopkeep/policy.local.yaml` | your personal overrides for this project; not committed |
| home | `~/.config/loopkeep/policy.yaml` | your personal rules everywhere (honors `XDG_CONFIG_HOME`) |
| project | `<ws>/.loopkeep/policy.yaml` | the project's rules; committed |
| org | — | reserved; nothing loads it today |

Under all of them sits the **built-in floor**, which is not a layer and cannot be
relaxed by any of them. It is **`ask-first`, not `deny`** — it forces a human decision,
it does not refuse. It fires on writes to loopkeep's own files: `~/.config/loopkeep/**`,
any `.loopkeep/**`, the definition of the workflow the run is currently executing, and
the daemon binary, config and event log. Run worktrees are carved out, so a run writing
inside its own sandbox is not floored. A bash command trips it only when one of those
paths appears in a **write position** (a redirect target, or an argument to
`rm`/`cp`/`mv`/`tee`/`dd`/`sed -i`) — reading or merely mentioning a protected path
does not. Separately, `ask_human` is floored to `ask-first` and `notify_human` /
`slack_reply` to `notify`.

**There is no per-workflow layer.** Scoping a rule to a workflow is a match axis on the
rule itself (`match: { workflow: [deploy, "release-*"] }`).

## How a verdict is reached

- **A matching rule**: the first match wins, in layer precedence order. So a personal
  layer *can* loosen a project rule for yourself — that is what project-local and home
  are for.
- **A rule with `allow_override: false`**: every such rule in every active layer
  applies, and the strictest wins. These only tighten.
- **No rule matched**: the blanket `defaults.level` is taken as the **strictest across
  all active layers**, not by precedence — a permissive personal default cannot quietly
  relax a cautious project default. With no default declared anywhere, it is `notify`.
- **The agent's self-reported level** can only raise the verdict, never lower it.
- **The floor** clamps upward last.

Untrusted changes what is even considered: the two workspace-scoped layers
(project and project-local) go inert, while home, the floor, and the repo-ask clamp
keep applying. Manual runs still work when untrusted; automatic triggers do not.

## The file

```yaml
version: 1 # optional, but must be 1 if present
policies: # optional named behaviors rules can reference (same file only)
  careful:
    level: ask-first
    timeout: { after: 72h, then: abort } # abort | deny
rules:
  - id: prod-migration # required
    match: # all five axes optional; within a list OR, across axes AND
      paths: ["migrations/**", "infra/prod/**"]
      tool: [bash] # exact, case-sensitive
      tags: [migration]
      workflow: ["release-*"] # `*` wildcard only
      command: "terraform apply" # case-insensitive substring of the raw command
    level: ask-first # or `policy: careful`, never both
    allow_override: false # default true; false means "no layer may relax this"
    reason: "production changes need a human"
defaults:
  level: notify # the only key `defaults` reads
```

`workflow` and `command` fail closed: if you constrain on them and the action carries
no workflow name or no raw command, the rule does not match. The removed key
`mandatory:` is a hard error — write `allow_override: false`.

## Explain and test (local, no daemon needed)

```sh
lk policy explain <tool> [paths...] [--tags a,b] [--command <text>] \
                  [--self <level>] [--workflow <name>]
lk policy test    <tool> [paths...] --expect <level> [...]
```

Flags take their value as the **next argument** — `--tags a,b`, never `--tags=a,b`.
Positional paths stop at the first `--` argument.

`explain` prints the matched rule chain — each line shows the layer, the source
`file:line`, and the level it contributed — then the `final` verdict, and a `[floor]`
line when the built-in floor fired. It also prints a note reminding you that rules
needing a live run (the running workflow's own definition, the paths a run protects at
runtime) are not evaluated in this stateless mode, so a clamp seen in a run's event log
can be stricter than what `explain` reproduces.

`test` is the same evaluation driven by exit code:

| Exit | Meaning |
|---|---|
| 0 / 1 | with `--expect`: matched / did not match |
| 0–3 | without `--expect`: the final level's rank (auto-approve 0, notify 1, ask-first 2, deny 3) |
| ≥ 10 | usage error or the evaluation itself failed |

The rank is deliberately never used for errors, so "exit 0" can't be misread as
auto-approve when nothing was evaluated. The exit code is computed after the floor
clamp.

## Tuning flow

1. Reproduce the verdict: take `tool` / `raw_command` / paths / tags from the run's
   `policy.clamped` or `attention.requested` event and feed them to `lk policy explain`.
2. Read the matched chain to find which layer and line produced the level.
3. Edit the narrowest layer that owns the decision. A project rule everyone should get
   goes in `.loopkeep/policy.yaml`; something that is only your preference on your
   machine goes in `policy.local.yaml` or the home policy. Prefer adding a narrow rule
   (specific tool + path/tag) over changing `defaults:`.
4. Verify both directions with exit codes: `lk policy test <tool> … --expect <level>`
   for the case you fixed, and a second `test` proving you did not relax something
   unrelated.
5. Re-approve. Editing `.loopkeep/policy.yaml` or `.loopkeep/policy.local.yaml`
   invalidates trust, and automatic triggers stay off until someone runs `lk trust`.
   (Editing the **home** policy does not invalidate trust — it is not part of the
   digest.) That approval is the human's; show them the diff rather than running it.

## Guardrails

- Relaxing review of write, exec, or destructive actions is a human decision: propose
  the diff of the policy file and say what becomes more permissive before applying it.
- If a level seems ignored, check which resolution rule applies. A blanket default
  can't be loosened by a lower-precedence layer, an `allow_override: false` rule can't
  be loosened at all, and nothing loosens the floor.
- If a rule seems to do nothing at all in an untrusted workspace, that is why: project
  and project-local layers are inert until `lk trust`.
- `--self <level>` simulates what the agent self-reported; use it to check that
  optimistic self-reports get clamped where you expect.
