# Investigating runs

Goal: given a run id (`run_000123`), reconstruct what happened and why.

## 1. Locate the run

```sh
lk runs            # id  state  workflow  depth=N  session=…  [note]
```

- **state** is one of `queued`, `running`, `waiting`, `paused`, `completed`, `failed`,
  `aborted`. `waiting` means a blocking attention is open; `paused` covers both an
  explicit pause and a take-over.
- **depth** is trigger-chain depth. 0 means a human or a plain trigger started it;
  rising depth on related runs is the signature of a workflow chain.
- **session** is the claude session id, or `(none)`. Its presence is what makes
  take-over and resume able to continue the same conversation.
- The **note** in brackets is where the important distinctions live, and there is
  exactly one per run:
  - `[pending · waiting for your approval — decide in your inbox]` — the run was held
    before it started (`launch: confirm`). Nothing has been spent.
  - `[pending · needs a terminal — decide in your inbox]` — an `attended` run found no
    multiplexer and is waiting for you to choose headless or wait.
  - `[interrupted · resume to continue]` — it ended early and can be picked up again.
  - `[interrupted]` — it ended early.

Two traps: a held run shows `queued` like an ordinary concurrency-queued run, and the
note is the only thing that tells them apart. And `interrupted` is a **flag on an
aborted run**, not a state of its own — the same is true of a taken-over run, which
shows as `paused`.

## 2. Read the event log (the primary evidence)

`.loopkeep/events.db` is an append-only SQLite log. **Read-only — never
INSERT/UPDATE/DELETE.** (The floor forces an approval prompt on writes to it anyway.)
Two columns, the whole event as JSON in `body`:

```sql
CREATE TABLE events (seq INTEGER PRIMARY KEY AUTOINCREMENT, body TEXT NOT NULL);
```

Timeline of one run:

```sh
sqlite3 .loopkeep/events.db \
  "SELECT seq, json_extract(body,'$.ts'), json_extract(body,'$.type'),
          json_extract(body,'$.payload')
   FROM events WHERE json_extract(body,'$.run_id')='run_000123' ORDER BY seq"
```

Each `body` also carries `id`, `device_id`, `workflow`, `origin`
(`agent` / `router` / `human` / `system` / `trigger`), and `payload_class`
(`meta` / `summary` / `content` — `content` holds raw diffs and commands and is not
synced to the Console).

There is no `lk` command that dumps the log; read the file, or use the terminal UI.
The home workspace (run-less `lk notify`) keeps its log at
`~/.config/loopkeep/events.db` instead of under a project.

Every event type that exists:

| type | what it tells you |
|---|---|
| `trigger.fired` | what started it — `payload.source` is `cron`, `file-watch`, `git`, `manual`, `run-completed`, `dispatch` or `remote` |
| `run.created` | workflow, worktree path, budget at start |
| `run.mode_resolved` | **only** when `attended` was requested: `{requested, resolved, host_kind}`. `resolved: headless` with `host_kind: none` is the record of a degradation |
| `run.state_changed` | `from` / `to` / `reason` — follow these to find where it ended |
| `run.session_started` | the agent session began |
| `action.proposed` | tool, `raw_command`, paths, tags, `self_reported_level` |
| `policy.clamped` | the policy overrode the self-reported level — read `payload` for the rule |
| `policy.gate_bypassed` | the gate was not consulted for this action |
| `attention.requested` | where it waited for a human; always names the rule that stopped it |
| `attention.resolved` | the verdict: `approve`, `deny`, `steer`, `timeout`, `superseded`, `ack`, or `take-over` |
| `run.human_input` | a sanitized preview of what a human typed into an attended run's pane |
| `action.executed` | the action actually ran |
| `checkpoint.created` | commit sha + step summary |
| `run.completed` / `run.failed` / `run.aborted` | terminal state |
| `digest.emitted` | a scheduled inbox digest went out |

Take-over has no event type of its own: it is `run.state_changed` with
`reason: "take-over"`, and hand-back is the same with `reason: "hand-back"`.

## 3. Answer the common questions

- **Why is it stuck?** Look for the last `attention.requested` without a matching
  `attention.resolved`, then `lk inbox`, `lk inbox <seq>` to read what it is asking for,
  and `lk approve <seq>` to decide. If the run is `queued` with a `[pending · …]` note,
  the attention is about starting the run at all — a `launch: confirm` gate, or an
  attended run with no terminal.
- **Why was an action blocked or downgraded?** Find `policy.clamped` events, then
  reproduce the verdict offline with `lk policy explain` (see `policies.md`).
- **What did it change?** `checkpoint.created` events carry commit shas. The work lives
  in the run's worktree (`.loopkeep/worktrees/run_000123`); after cleanup the
  checkpoint branches remain — inspect with normal git (`git log`, `git show`).
- **Why did it die?** `run.state_changed` with the failing `reason`, plus the last
  events before the terminal one. Reasons that mean "ended early, resumable":
  `daemon stopped before this run finished`, `the session ended before this run
  finished` (an attended pane was closed), `no activity — the attended session timed
  out`, and `finished from outside — resume it to continue` (`lk finish` with nothing
  to keep).
- **Why did an attended run go quiet?** Attended runs can be answered in the pane
  instead of the inbox: that shows up as `attention.resolved` with
  `decision: superseded` and then `run.state_changed` with
  `reason: "answered directly in the session"`. No `action.executed` follows — the
  action was neither approved nor denied.
- **Nothing in `events.db` explains it?** Daemon-level failures — a run that never
  produced events, adapter spawn errors, Console connection problems, policy load
  warnings — go to the daemon's diagnostic log, not the event log:
  `~/.local/state/loopkeep/loopkeepd.log` (honors `XDG_STATE_HOME`; rotated at 5 MB
  with one `.old` generation, so it is not a long history). `lk start` discards the
  daemon's stderr, which makes this file the only record.

## 4. Acting on findings

- `lk rerun run_000123` — run the same workflow again, reusing the trigger context.
- `lk pause|resume|abort run_000123` — control a live run. `abort --rollback` resets
  to the last checkpoint first; the worktree is kept either way.
- `lk take-over run_000123` — drive the agent yourself; only the floor stays in effect.
  Finish with `lk hand-back run_000123 --summary "…"`, which commits a checkpoint for
  your changes and tells the agent what you did. Take-over refuses an **attended** run:
  it already runs in a session you control, so open its terminal instead.
- `lk finish run_000123` — end an attended run you are steering. It detaches rather
  than kills: with a deliverable the run is `completed`, without one it is left
  interrupted and resumable. It refuses a headless run.
- `lk inbox <seq>` — print that attention's exact change: the raw diff or command,
  unformatted, ready to pipe into `less` or a diff viewer. The listing doesn't carry it,
  so this is how you read what you are about to approve rather than trusting the
  one-line question. Items that have one are marked in `lk inbox`.
- `lk approve <seq> [--steer --instruction <t>|--abort|--ack]` — decide a waiting
  attention. Notifications take nothing but `--ack`; items needing a decision refuse it.
- Cross-workspace view: `lk watch --json` streams new attentions as NDJSON, covering
  every workspace unless `--workspace` narrows it. Each item carries `has_raw` rather
  than the change itself — fetch it per item with `lk inbox <seq>`.
