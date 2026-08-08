# Writing workflow frontmatter

A workflow is one markdown file in `<workspace>/.loopkeep/workflows/<name>.md`: YAML
frontmatter between `---` fences, then a prompt body. Name the file in kebab-case; the
file name without `.md` is the workflow name used by `lk run`, `lk disable`,
`run-completed`, and `dispatch-workflow`.

The format is GitHub Agentic Workflows (gh-aw) compatible. Keys that exist upstream
keep their upstream meaning, and everything loopkeep adds beyond gh-aw goes nested
under a single `x-loopkeep:` key. Unknown keys are kept as written and warned about,
never dropped — so a typo is silent in effect but visible in the warnings. Run
`lk trigger explain <workflow>` after editing to see what actually registered.

```markdown
---
on:
  schedule:
    - cron: "0 9 * * *"
  manual: {}
engine:
  id: claude
  model: claude-haiku-4-5-20251001
x-loopkeep:
  budget:
    daily_runs: 30
  concurrency:
    max: 1
    on_limit: skip
safe-outputs:
  notify: {}
  emit-output: {}
---
Review yesterday's commits and report anything user-visible.

{{ trigger }}

Finish by publishing `{"severity": "<low|medium|high>", "summary": "<one line>"}`
with emit_output.
```

## `on:` — local triggers

These fire from your own machine and need nothing else set up.

```yaml
on:
  schedule:
    - cron: "*/15 * * * *" # 5 or 6 fields; hourly / daily / weekly are also accepted
  file-watch:
    paths: ["inbox/**/*.pdf"]
    events: [create, modify] # create | modify | delete
    debounce_ms: 2000 # collapse a burst of saves into one firing
    min_size_kb: 1 # skip empty or half-written files
    coalesce: latest # latest | all
  git:
    events: [commit, branch-created] # these two only
    branch: ["main", "release/*"] # glob
    author_not: ["loopkeep[bot]"] # the standard guard against self-triggering
  run-completed:
    workflow: "test-fixer"
    status: [failed] # done | failed
    if: event.output.attempts >= 3
  manual: {} # declares the workflow as one a human starts
```

Notes that bite:

- **`manual: {}` is what marks a workflow as human-startable.** The desktop app and the
  Console only offer a Run button for workflows that declare it. `lk run <name>` starts
  any workflow that exists, declared or not — so a workflow missing `manual` is not
  broken from the CLI, just invisible as a manual action everywhere else. Declare it on
  anything a person is meant to trigger.
- `cron` is evaluated on your machine at minute granularity, in local time.
  `hourly` / `daily` / `weekly` are normalized to cron expressions when the file is
  parsed. Anything that is neither an alias nor a 5-or-6 field expression is kept
  verbatim and then never matches, so it fires silently never — write real cron.
- `git` knows exactly two events: `commit` and `branch-created`. Any other string in
  `events:` is dropped.
- `if:` is a small expression language: `event.` paths, `== != > >= < <=`, `contains`,
  `&& || !`, and parentheses. No function calls. It reads the upstream run's output as
  `event.output`.
- A firing rejected by a filter is still recorded, so "it didn't run" is always
  answerable from the event log.

`coalesce` decides what happens to firings that piled up while the machine was busy or
asleep: `latest` runs once for the newest, `all` replays each. Defaults are `latest`
for `schedule` and `file-watch`, `all` for `git` and `run-completed`.

## `on:` — triggers delivered through the Console

GitHub events, Slack messages and webhook posts reach the daemon through the Console,
which matches your declaration before anything is delivered. What you write here is the
whole subscription — the daemon does not re-check the event.

```yaml
on:
  github:
    repos: ["org/other-repo"] # omit to bind to this workspace's git origin
    issues: { actions: [opened], labels: [bug], assignee: me }
    issue_comment: { mentions: me, on: my-pr, from_not: me }
    pull_request: { actions: [review_requested], reviewer: me }
    pull_request_review: { on: my-pr, actions: [submitted] }
    workflow_run: { workflows: [ci], branches: [main], conclusions: [failure] }
    push: { branches: [main] }
    projects_v2_item: { actions: [edited], content_type: [Issue] }
    pull_request_review_comment: { from_bot: ["renovate[bot]"] } # bots fire only when listed
  slack:
    channels: ["C0123ABCD"] # default for every event in the block
    mention: {} # someone @-mentions the loopkeep app
    dm: {} # a direct message to the app
    message: { keywords: [deploy], mentions: me } # needs at least one filter
    reaction: { emoji: [pushpin], on: my-message }
  webhook: # no value at all: every hook URL on your account
    - deploy-done # or the object form { name: deploy-done }
```

An event's value can also be a **list of filter sets**, which is an OR — the trigger
fires if any one set matches, and filters inside a set still have to match together:

```yaml
on:
  slack:
    message:
      - { channels: [C0123ABCD], keywords: [deploy] }
      - { mentions: me }
      - { channels: [C0ALERTS], from_bot: ["B0123ALERT"] } # this set fires on that bot alone
```

### The four predicates

Every remote event takes the same four questions, plus filters of its own.

| Predicate | Asks | Example |
|---|---|---|
| `from` / `from_not` | which person did it | `from_not: [octocat]` |
| `from_bot` | which bots may fire it at all | `from_bot: ["renovate[bot]"]` |
| `mentions` | who the text names | `mentions: me` |
| `on` | whose thing it happened to | `on: my-pr` |

**No bot fires a trigger unless the filter set lists it under `from_bot`** — on GitHub or
on Slack, `from_bot: ["*"]` for every bot. `from` and `from_not` match people only, so no
spelling of `from_not` keeps a bot out or lets one in. On GitHub the value is the bot
account's login (`renovate[bot]`); on Slack it is any identifier the event carries for
that bot — its bot ID (`B…`), app ID (`A…`), or the bot's user ID (`U…`).

Within one filter set, `from` / `from_not` decide which people fire it and `from_bot`
decides which bots. A set naming bots and no people is a **bot-only** subscription:
`{ channels: [C0ALERTS], from_bot: [B0SENTRY] }` fires on that bot's posts and not on a
person talking in the channel. Add `from` or `from_not` to bring people back —
`{ from_not: [noisy-human], from_bot: [B0SENTRY] }` is everyone but that person, plus
that bot. Non-actor filters (`keywords`, `mentions`, `labels`, `channels`) apply either
way. `from_bot` goes in the filter set; it cannot sit beside the Slack events.

Values are `me` or a literal (a GitHub login, a Slack user ID); a single value can skip
the list. `from` and `from_not` accept `*` as a wildcard. `on` takes exactly `my-pr`,
`my-issue`, or `my-message`; to point at someone else's, write the object form with the
key that names the thing — `{ pr: octocat }`, `{ issue: octocat }`, or
`{ message_from: U0123ABCD }`. Those three keys and three tokens are the entire
vocabulary. `me` resolves to the identity linked when the app was connected, and
matches nothing until one is linked.

### Which filters each event accepts

An event only accepts filters it can evaluate against its own payload. Write one it
can't — `reviewer` on `issues`, `conclusions` on `push`, `emoji` on a Slack `mention` —
and it is **refused with a warning up front** rather than registered and then never
matched. Get this wrong and the trigger never fires.

GitHub:

| Filter | Accepted on |
|---|---|
| `actions` | every event whose payload carries an `action` — not `push`, `create`, `delete`, `gollum`, `fork`, `status`, `page_build`, `public`, `workflow_call` |
| `labels` | issues, pull requests, discussions, and their comments |
| `assignee`, `on` | issues and pull requests, and their comments and reviews |
| `mentions` | anything with a body: issues, PRs, comments, reviews, commit comments, discussion comments |
| `reviewer` | `pull_request` / `pull_request_target`, on `review_requested` |
| `branches` | `push`, `create`, `delete`, `workflow_run`, and a pull request's base branch |
| `workflows`, `conclusions` | `workflow_run` only |
| `from`, `from_not`, `from_bot` | every event except `projects_v2_item` |

Slack: `reaction` takes `on`, `from`, `from_not`, `from_bot`, `emoji`, `channels`. Every
other Slack event takes `from`, `from_not`, `from_bot`, `mentions`, `keywords`,
`channels`. `keywords` is a case-insensitive substring of the message text. `channels`,
`from` and `from_not` may also sit beside the events as the block's default, which a set
overrides; `from_bot` may not.

`projects_v2_item` is its own world: it takes only `actions` and `content_type`
(`Issue` / `PullRequest` / `DraftIssue`), because the payload carries node IDs rather
than field or option names. Its `{{ trigger }}.body` is the raw webhook JSON.

### Declarations that are refused

These do not error the file — they are dropped with a warning, and the trigger stays
silent. Check for them with `lk trigger explain`.

- `on.slack.message` with no `keywords`, `mentions`, `channels`, `from` or `from_bot`. It
  would match every message the app can see. A lone `from_not` does not count as
  narrowing — excluding a few people still leaves nearly everything.
- A bot account (`renovate[bot]`, anything ending `[bot]`) under `from`. `from` only
  matches people, so it could never fire; write `from_bot: ["renovate[bot]"]`.
- `from_bot: []`. An empty allowlist lets no bot through, which is what leaving it out
  already does. List the bots, or write `from_bot: ["*"]`.
- `from_not: ["*bot*"]`. Bots do not fire unless `from_bot` lists them, so there is
  nothing to exclude, and as a plain glob it drops people such as `abbott`. Remove it.
- `from_bot` written beside the Slack events instead of inside a filter set. A
  block-level allowlist would turn every event under it, `dm: {}` included, into a
  bot-only subscription.
- An event declared with an empty list (`issues: []`). It can never match.
- Slack `channels` given a name (`#ops`) instead of an ID. IDs start with `C`, `G` or
  `D` and are uppercase alphanumeric; copy one from the channel's details.
- An `on:` predicate naming an unknown token or key (`on: my-prs`,
  `{ message: U0123 }`). Only that condition is dropped; the rest of the event still
  registers — which is why a typo here widens nothing.
- `on.webhook` given an empty list, or an entry carrying `secret:`. Nobody holds the
  sender's signing key, so a signature can't be checked.
- A hook name that isn't a name: it must start with a letter or digit and use only
  letters, digits, dots, underscores and hyphens.

The four bot refusals drop the whole filter set that carries them, not just the offending
key — a set you meant to narrow must not quietly go wide. Sibling sets on the same event
still register.

### What never fires

- **Any bot the workflow didn't list.** Bots are off by default on both apps, and
  `from_bot` is the only way in. An existing file that kept bots out with `from_not` is
  now refused rather than honoured — rewrite it.
- Events raised by loopkeep's own GitHub App, so a comment a run posts can't wake that
  run again. This one cannot be opted back in: listing it under `from_bot` changes
  nothing.
- Anything the account can't see: a private repository it isn't in, a private channel
  it isn't a member of.

loopkeep no longer injects `from_not: ["*bot*"]` into Slack subscriptions. The conditions
written in the file are the conditions that run.

A bare gh-aw event (`on: issues`, `on: [issues, push]`, `on.issues:`) is accepted as
shorthand for `on.github.<event>` with no filters. Keys written under it — gh-aw's own
`types:`, for instance — are **not** read as loopkeep filters. Filters only belong
under `on.github.<event>`.

## `engine`

```yaml
engine:
  id: claude
  model: claude-sonnet-5
  params: {}
```

loopkeep runs only the `claude` engine — the Claude Code CLI as a subprocess. Omit
`engine` entirely and a run uses `claude` anyway (gh-aw's own default is `copilot`,
which loopkeep has no engine for, so it logs a notice and proceeds). Any other engine
id **fails the run at start** with a message rather than silently substituting.
`model` is per-workflow, so a heartbeat can run a small model while the heavy work
runs a big one.

## `tools`

```yaml
tools: [edit, bash, web-fetch]
```

The subset of agent tools this workflow may use. Recognized: `edit`, `bash`,
`web-fetch`, `web-search`, `read`, `write`, `view`, `fetch`, `glob`, `grep`, `github`,
`playwright`. Anything else warns and is ignored.

## `safe-outputs`

Declares the effects the workflow is allowed to produce. Declaring one **never**
bypasses policy evaluation — it says the effect is in scope, not that it is approved.

```yaml
safe-outputs:
  create-pull-request: {} # gh-aw native; runs through your gh auth
  add-comment: {}
  add-labels: {}
  local-commit: {} # commit in the run's worktree
  local-file-write: {} # write into the workspace
  run-command: {}
  notify: {} # post to the inbox
  dispatch-workflow:
    allowed: ["incident-triage", "inbox-processor"] # explicit allowlist, required
  emit-output: {} # structured JSON for downstream workflows
```

`emit-output` is the one always-auto-approved output: it only produces data for
downstream workflows and is never shown to a human. The gh-aw-native GitHub outputs
(`create-pull-request`, `create-issue`, `create-discussion`, `add-comment`,
`add-labels`, `update-issue`, `push-to-pull-request-branch`,
`create-pull-request-review-comment`, `create-code-scanning-alert`, `missing-tool`)
execute through your `gh` auth. Everything else — including any output name loopkeep
doesn't recognize — goes through policy evaluation.

## `mcp-servers`

```yaml
mcp-servers: [notion] # in a workflow: names only
```

Connection details (`command`, `url`, `args`, `env`, `container`, `registry`, `type`)
belong in workspace or daemon configuration, never in a workflow. Writing them inline
is a **validation error**, not a warning, and the values are never echoed back into
logs or messages. MCP tool calls are policy-evaluated like anything else; in policy
rules they match as `tool: mcp:notion/create-page`.

## `x-loopkeep`

Everything loopkeep adds beyond gh-aw nests here. Exactly seven subkeys are
recognized — `engine`, `concurrency`, `budget`, `tags`, `execution_mode`, `mux`,
`coalesce` — and anything else warns and has no effect. There is no flat form: a
top-level `x-loopkeep.budget:` key is warned about and ignored, so nest everything
under one `x-loopkeep:` block.

```yaml
x-loopkeep:
  engine: claude # overrides root engine: entirely (string, or { id, model })
  concurrency:
    max: 1
    on_limit: queue # queue (default) | skip | replace
  budget:
    tokens: 200000
    time_ms: 900000
    steps: 30
    daily_runs: 300
    daily_tokens: 500000
  tags: [deploy, migration]
  coalesce: 5m # 30s | 5m | 2h — fold remote firings in a window into one run
  execution_mode: attended # auto (default) | headless | attended
  mux: herdr # herdr | tmux | zellij | cmux
```

- **`budget`** — integers only. A run that exceeds one fails, recorded as `failed` with
  a budget reason. Time spent waiting on a human doesn't count against `time_ms`.
- **`concurrency`** — `queue` holds new runs until a slot frees, `skip` drops the
  firing (recorded, not silent), `replace` aborts the running one in favor of the new.
  The gh-aw-native root `concurrency:` is also honored: `cancel-in-progress: true`
  behaves like `replace`, anything else like `queue`.
- **`tags`** — semantic labels policy rules can match. Self-declared by the workflow,
  so use them for escalation, not for safety: hard rules should match `paths` or
  `tool`, which a workflow can't lie about.
- **`coalesce`** — a window that applies to GitHub, Slack and webhook firings only. The
  first event runs immediately and the rest are held, then run together when the window
  closes; the run receives every event it stood for. Local triggers ignore it and keep
  their own `coalesce: latest | all`. Leave it off for a workflow that replies to
  whatever triggered it — one run can only answer one thread. An unreadable duration
  falls back to no window, with a warning.

### `execution_mode` and `launch`

```yaml
x-loopkeep:
  execution_mode:
    mode: attended # auto (default) | headless | attended
    on_no_host: ask # ask (default for attended) | headless
    launch: confirm # auto (default) | confirm
```

`auto` picks by trigger: a manual run becomes attended, an unattended firing stays
headless. `headless` runs the agent as a subprocess supervised from the inbox.
`attended` runs it in a multiplexer pane you can watch and type into — the policy gate
still applies to every tool.

When `attended` is declared and no multiplexer is running, `on_no_host: ask` (the
default) **holds the run** and puts a choice in the inbox: run headless now, or wait
for a terminal. "Wait" really waits — the run starts attended on its own when a
multiplexer appears. `on_no_host: headless` falls back quietly instead. `auto` never
asks.

`launch: confirm` is independent of the mode: it holds the run before the agent starts
and puts a card in the inbox showing what wants to run and what triggered it. Nothing
is spent if you abort. Reach for it when the trigger is outside your control — a Slack
mention, an issue anyone can open.

Held runs show up as state `queued` and take no concurrency slot.

### `mux`

```yaml
x-loopkeep:
  mux:
    driver: tmux # herdr | tmux | zellij | cmux
    space: loopkeep # multiplexer session/workspace to place the pane in
    tab: automation # tab/window within it
```

A bare `mux: tmux` is the driver alone. Each key falls back on its own to this
project's `.loopkeep/config.yaml`, then the personal `config.yaml`, then the built-in
`loopkeep` / `automation`. The pane is named after the run id.

## The body

The body is a natural-language instruction to an agent that cannot ask you a question
mid-run unless you give it the means to. Four token kinds are expanded; every other
`{{ … }}` is left exactly as written.

### `{{ trigger }}` and `{{ trigger.<path> }}`

The trigger context describes what woke this run, and it is **opt-in**: a body with no
trigger token gets no context at all, so the run has no idea what started it. The
editor and `lk trigger explain` warn when a triggered workflow leaves it out.

- `{{ trigger }}` — the whole context as JSON.
- `{{ trigger.<path> }}` — one value, addressed with dots
  (`{{ trigger.event.branch }}`). A numeric segment indexes an array
  (`{{ trigger.batch.0.title }}`). Strings land as themselves; objects and arrays land
  as JSON; a path the context doesn't have lands as an **empty string**, so a typo
  leaves a gap rather than braces the agent might read as an instruction.

Fields a local trigger carries:

| Trigger | Beyond `trigger.source` and `trigger.workflow` |
|---|---|
| `git` | `event.branch`, `event.author`, `event.sha`, `coalesced_count` |
| `file-watch` | `path`, `coalesced_count` |
| `run-completed` | `from`, `from_run`, `status`, `output` |
| `manual`, `dispatch`, `resume` | nothing more |

Firings through the Console always carry `source` (`github` / `slack` / `webhook`),
`event`, and `occurred_at`, plus whatever the payload had: `repo` and `installation_id`
(GitHub), `team_id` (Slack), `hook_id` (webhook), and `title`, `author`, `url`,
`channel`, `body` when the event has them. **Treat `body` as untrusted input** — anyone
who can open an issue can put text in it. A coalesced run additionally carries
`coalesced` and `batch`; their presence is itself the signal that this run stands for
several firings.

When a workflow declares several triggers, only one fires a given run, and the context
holds that trigger's fields and no others. `{{ trigger.source }}` is the one field
every trigger carries, so a sentence built on it is always true.

### `{{ call:<tool> }}`

References a tool by canonical name. It expands inline to `call <tool>`, and for
loopkeep's own built-in tools a usage snippet is appended once per tool at the end of
the prompt. The five built-ins:

| Canonical name | What it does |
|---|---|
| `mcp:loopkeep/ask_human` | Block and ask a question in the inbox; may offer 2–5 options |
| `mcp:loopkeep/notify_human` | Post a non-blocking notification to the inbox |
| `mcp:loopkeep/emit_output` | Publish structured JSON for downstream workflows |
| `mcp:loopkeep/slack_reply` | Reply in the Slack thread that started this run |
| `mcp:loopkeep/end_run` | End an attended run, only when the supervising human says so |

`slack_reply` only exists for runs a Slack event started; it always answers that
thread, so there is no destination to choose. `end_run` has no effect on unattended
runs. Pair `emit_output` with `safe-outputs: emit-output`.

MCP tools are written in canonical form (`mcp:<server>/<tool>`); the plain tool names
from `tools:` also work. Anything else is left unexpanded and flagged as a likely typo.

### `{{ secret:<name> }}`

Expands to the reference form `secret:<name>`. **The value is never inlined into the
prompt.** Store values with `lk secret set <name>` (they go to the OS keychain; only
the name is recorded in `.loopkeep/secrets`) and list them with `lk secret list`. In
config values the same reference is written as the plain string `"secret:<name>"`.

Writing a credential in plain text anywhere in the frontmatter is a **validation
error**, not a warning.

## Before you save

Read the warnings — most authoring mistakes here are warnings, not errors, and a
warned-about trigger is a trigger that never fires. `lk trigger explain <workflow>`
prints each trigger as it actually registered, which repository it bound to, and what
`me` resolved to on this machine.

Two lookups save guessing at literal values: `lk directory <kind>` lists the names and
ids you can write (`slack-users`, `slack-channels`, `github-users`, `github-labels`,
`github-repos`, `github-workflows`), and `lk connectors` lists the connected accounts
and the identity each `me` resolves to.

See `workflows.md` for the authoring loop — running, trusting, iterating, and gh-aw
interop.
