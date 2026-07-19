# Investigating runs

Goal: given a run id (`run_000123`), reconstruct what happened and why.

## 1. Locate the run

```sh
lk runs            # id  status  workflow  depth=N  session=…  [interrupted]
```

- `status`: `completed` / `failed` / `aborted` / running states. `[interrupted]`
  means the daemon stopped while the run was in flight; it was restored on restart.
- `depth`: trigger-chain depth (a run whose outputs triggered another run). Rising
  depth on related runs is the signature of a workflow chain.

## 2. Read the event log (the primary evidence)

`.loopkeep/events.db` is an append-only SQLite log; every row is one JSON event.
**Read-only — never INSERT/UPDATE/DELETE.** The schema is internal and may change, so
discover it first instead of assuming:

```sh
sqlite3 .loopkeep/events.db '.schema'
```

Timeline of one run (adjust column names to what `.schema` showed):

```sh
sqlite3 .loopkeep/events.db \
  "SELECT json_extract(body,'$.ts'), json_extract(body,'$.type'), body
   FROM events WHERE json_extract(body,'$.run_id')='run_000123'
   ORDER BY seq"
```

Event types you will see, in the order a healthy run produces them:

| type | what it tells you |
|---|---|
| `trigger.fired` | what started it (cron id, manual, webhook) |
| `run.created` | workflow, worktree path, budget at start |
| `run.state_changed` | `from`/`to`/`reason` — follow these to find where it ended |
| `run.session_started` | the agent session began |
| `action.proposed` | tool, `raw_command`, paths, tags, `self_reported_level` |
| `policy.clamped` | the policy overrode the self-reported level — read `payload` for the rule |
| `attention.requested` / `attention.resolved` | where it waited for a human, and the verdict |
| `action.executed` | the action actually ran |
| `checkpoint.created` | commit sha + step summary |
| `run.completed` / `run.failed` / `run.aborted` | terminal state |

## 3. Answer the common questions

- **Why is it stuck?** Look for the last `attention.requested` without a matching
  `attention.resolved`, then `lk inbox` and `lk approve <seq>`.
- **Why was an action blocked or downgraded?** Find `policy.clamped` events, then
  reproduce the verdict offline with `lk policy explain` (see `policies.md`).
- **What did it change?** `checkpoint.created` events carry commit shas. The work
  lives in the run's worktree (`.loopkeep/worktrees/run_000123`); after cleanup the
  checkpoint branches remain — inspect with normal git (`git log`, `git show`).
- **Why did it die?** `run.state_changed` with the failing `reason`, plus the last
  events before the terminal one.

## 4. Acting on findings

- `lk rerun run_000123` — run the same workflow again, reusing the trigger context.
- `lk pause|resume|abort run_000123` — control a live run.
- `lk take-over run_000123` — drive the agent yourself (only the floor stays in
  effect); `lk hand-back run_000123 --summary "…"` when done.
- Cross-workspace view: `lk watch --json` streams new attentions as NDJSON.
