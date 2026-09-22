# Troubleshooting

Start here when something you asked for is not running. Three commands answer
most of it:

```bash
skram status                    # held? what is running, and who asked for it
skram queue list                # every item in order, with its actor
skram doctor                    # what this machine still needs
```

Pages: [index](README.md) · [configuration](config.md) ·
[how it works](how-it-works.md) · [agent setup](agent-setup.md) ·
[questions](faq.md)

## An item never starts

Work through these in order. The first two are far more common than the rest.

### 1. Is the queue held?

A hold pauses the whole queue, for everyone. Jobs that are already running
finish; nothing new starts. Enqueueing still works, so items pile up behind
it. `skram status` says so, with who held it, why, and for how long:

```text
No running jobs.
HELD by you: debugging the cluster by hand (0s)

Queue: 2 pending, 0 running, 0 completed, 0 failed
ETA: queue clears in ≥<1s (2 pending, 2 without history)
```

An agent that enqueued with `--json` was told at the time. The reason travels
with the hold, so the agent can say why it is waiting instead of retrying:

```json
{
  "item_id": "q_a49121b2",
  "project": "api",
  "target": "build",
  "actor": "claude-code",
  "pending": 3,
  "held": true,
  "hold": {
    "actor": "you",
    "reason": "debugging the cluster by hand",
    "since": "2026-09-21T19:56:25.312483-04:00"
  },
  "wait": "skram wait q_a49121b2"
}
```

Lift it when you are done. Anything pending starts immediately:

```bash
skram release
```

```text
Queue processor started (PID 64455)
Released (held by you for 0s: debugging the cluster by hand)
```

Nothing in the queue bypasses a hold: not `-f`, not `wait`, not another repo,
not another person. That is the point of it. The one thing that still runs is
an ephemeral target, because an ephemeral target never enters the queue — it
is a read-only command like `status`, so a hold has nothing to say about it.

A hold you set with a reason is the polite way to take a shared resource for
an hour:

```bash
skram hold --reason "rebuilding the cluster by hand"
```

### 2. Is the queue draining?

A drain is one-shot: the processor finishes the jobs it has and exits, and
pending items stay queued. The next `skram run` starts a fresh processor and
the queue moves again.

```bash
skram queue drain
```

```text
Drain signal set. Processor will stop after the running job(s); 0 pending item(s) stay queued.
```

Two things to know:

- A drain is only visible in `skram status --json`, which carries it as
  `"drain": true`. The text output has no drain line, so a drained queue and
  a queue whose processor is gone read the same there.
- A drain set while no processor is running is consumed by the *next* one.
  That processor starts, sees the flag, clears it and exits without taking an
  item, so the enqueue that started it appears to do nothing. Enqueue again
  (or `skram release`, which also respawns) and the item runs.

### 3. Is another job holding the lane?

A lane is the single-file line in front of one shared resource. Every project
that names that resource shares the lane, whatever repo it lives in, and one
job at a time runs in it. Lanes run in parallel with each other.

So an item can be pending while other jobs run — that is correct, not stuck.
`skram status` shows the lanes when there is more than one:

```text
Running:
  apps/build  J20260921_195625  (PID 64458, 1s elapsed)  by claude-code
    Lane: kind-dev
  api/build  J20260921_195625  (PID 64457, 1s elapsed)  by claude-code
    Lane: kind-staging

Queue: 1 pending, 2 running, 0 completed, 0 failed
Processor: running (PID 64455)
Lanes
  kind-dev      running apps/build (est. unknown) · 1 pending (1 without history)
  kind-staging  running api/build (est. unknown) · idle after
```

`skram queue list` gives the order within the whole queue:

```text
  1. [running] apps/build  by claude-code
  2. [pending] apps/test  by claude-code
  3. [running] api/build  by claude-code

1 pending, 2 running, 0 completed, 0 failed
```

Item 2 waits because item 1 has `kind-dev`. Item 3 runs anyway because its
lane is a different one. If you did not expect two projects to share a lane,
check their `resource:` values; if you expected them to share one and they do
not, they are in different lanes by accident.

To stop waiting: wait for the lane, take the item back, or end the job ahead
of it.

```bash
skram wait --resource kind-dev --timeout auto
skram queue cancel q_a49121b2
skram kill --resource kind-dev
```

### 4. Is the processor gone?

One background process — the processor — takes items off the queue and runs
them. It starts on demand when you enqueue and exits when the queue is idle.
If it is killed while jobs are running (a reboot, a `killall`, closing a
session that owned it), nothing takes the next item until something enqueues
again.

The tell is a `Queue:` line that counts work with no `Processor:` line under
it, and no job in the `Running:` block:

```text
No running jobs.

Queue: 1 pending, 1 running, 3 completed, 0 failed
ETA: queue clears in ~3s (1 pending)
```

One pending, one "running", and nothing running. What happened to each:

- **The job that was running kept running.** A job is its own process group,
  not a child that dies with the processor, so it finishes and writes its own
  result. Losing the processor loses the bookkeeping, not the work.
- **Its queue row still says `running`**, because updating that row is the
  processor's job and the processor is gone. It is a stale row: the counts
  are wrong, and it stays until `skram queue reset`. While the job's own
  process still runs, a new processor keeps its lane taken, so nothing else
  starts on that resource; once that process is gone the lane is free. If
  the job's own process really did die — a reboot, rather than the processor
  alone — the next `skram status` marks that job killed in its job
  directory.
- **The pending item starts on the next enqueue.** `skram release` also
  starts a processor when anything is pending.

To clear the stale rows, archive the queue and start a fresh one. History
already written to job directories is untouched:

```bash
skram queue reset
```

### The processor's log

The processor writes what it does — and anything a job printed on its way
past — to `queue-processor.log`, beside the queue file:

```text
$XDG_DATA_HOME/skram/queue/queue-processor.log
```

`XDG_DATA_HOME` is usually unset, in which case the path is
`~/.local/share/skram/queue/queue-processor.log`. If the config file sets
`queue_dir`, the log is in that directory instead. `skram doctor` prints the
queue directory it is using, so this resolves it on any machine:

```bash
skram doctor
```

This log is not pruned. Delete it whenever you like; it is recreated.

Per-job output is somewhere else: each job has its own directory under the
logs directory with `raw.log`, `events.jsonl` and `status.json` in it. Read
those through `skram logs <job>` rather than by hand.

## The command does nothing at all (macOS)

If `skram` prints nothing, exits immediately, and `echo $?` says `137`, macOS
has quarantined it. Any file a browser downloaded carries a quarantine
attribute, and an unsigned binary carrying one is killed before it runs — so
there is no error message to search for.

Check and clear it:

```bash
xattr -l "$(command -v skram)"        # com.apple.quarantine listed?
xattr -d com.apple.quarantine "$(command -v skram)"
skram version
```

The install script does not produce this, because `curl` does not set the
attribute. Downloading the release archive in a browser does.

## Notifications never appear

Queued jobs notify when they end. Inside a herdr session that goes through
herdr and needs nothing installed. Otherwise it is a macOS notification,
which only shows once `terminal-notifier` is installed and registered as an
application — the fallback skram uses without it is usually silent.

`skram doctor` names the path in use:

```text
[ok] notify: terminal-notifier (queued jobs notify on completion; herdr inside a session)
[--] notify: terminal-notifier not on PATH; completion notifications use osascript, which macOS may not show
[--] notify: disabled (SKRAM_NOTIFY=0); queued jobs will not notify on completion
```

The one-time setup is in [notifications](notifications.md). `SKRAM_NOTIFY=0` turns
notifications off.

## Targets are missing, or the project is ambiguous

**No targets, or not the ones you expected.** `skram discover` lists what
each project offers. Discovery is a static read of the project's entry point
— it never runs the file — so a target only appears if it is written where
the reader looks: a rule in a `Makefile`, a task in a `Taskfile.yml`, a
recipe in a `justfile`, or, in an ops script, an arm of the `case … in` block
that starts in column 0. An ops script's arm has to be on a line of its own,
optionally followed by a comment that becomes the target's help:

```bash
build) # Build the image
    docker build -t app .
    ;;
```

A project with an `expose:` list shows only the targets on it. `skram doctor`
reports the same thing with the file it read:

```text
[ok] apps: 4 target(s) via script (/home/you/dev/apps)
[!!] broken: script file not found: /home/you/dev/api/nope.sh
```

**Ambiguous project.** Leaving the project name out asks skram to work it out
from the directory you are in. When two projects claim that directory it
stops rather than guess:

```text
Error: cwd matches projects apps and apps-staging; name one
```

Name the project: `skram run apps build`. The same applies when the directory
belongs to no project at all — a worktree away from the configured checkout,
for instance. Naming the project always works, from anywhere; the job runs in
the project's own directory either way, not in yours.

## After a failure

`skram explain` is the short version of a failed job: what it was, who asked,
the lane, why it is classified as it is, the tail of what it printed, and the
commands to go further. With no argument it explains the most recent job.

```bash
skram explain
```

```text
apps/boom  J20260921_195642  [failed] exit=3 (<1s)  by you
Lane: kind-dev
Reason: unknown — could not reach the cluster
Last 2 lines (all streams):
  bash /home/you/dev/apps/ops.sh boom
  could not reach the cluster
Next:
  - full log: skram logs J20260921_195642_7417fd
  - re-run: skram run apps boom
```

It takes a job id or an item id, `--lines N` for a longer tail, and `--json`
for the same fields as an object. When the tail is not enough, the full log
is `skram logs <job>`.

Exit codes a caller can branch on: the job's own code, or `124` for a
`--timeout` that expired, `125` for an item cancelled before it ran, and
`137` for a job that was killed. `skram wait` and `skram run -f` use the same
three; [how it works](how-it-works.md) has the table and its one caveat.

## Reading `skram doctor`

`skram doctor` is the first thing to run on a machine that is behaving
oddly. Every line starts with one of three markers:

| Marker | Meaning |
|---|---|
| `[ok]` | Checked, and fine. |
| `[!!]` | A problem. Counted in the total at the end. |
| `[--]` | Information: something optional that is off, absent, or worth knowing. Never counted. |

Indented lines belong to the line above: an indented `[!!]` is a problem with
that project in particular, and an indented `run:` line is the command that
fixes what precedes it.

```text
[ok] Engine v0.13.0 (binary v0.13.0-11-gefe49fe)
[ok] Config loaded (3 project(s), 0 workflow(s))
[ok] Data directory: /home/you/.local/share/skram/logs
[ok] Data directory: /home/you/.local/share/skram/queue
[ok] notify: terminal-notifier (queued jobs notify on completion; herdr inside a session)
[--] herdr: installed, no running session
[ok] apps: 4 target(s) via script (/home/you/dev/apps)
[ok] apps-staging: 4 target(s) via script (/home/you/dev/apps)
[!!] broken: script file not found: /home/you/dev/api/nope.sh
[--] /home/you/dev/apps: agent rules: /home/you/dev/apps/AGENTS.local.md missing
[--] /home/you/dev/apps: agent rules: /home/you/dev/apps/CLAUDE.local.md missing
     run: skram agent install-rules

1 issue(s) found.
```

The last line is the summary: `All checks passed.` or `N issue(s) found.`,
counting the `[!!]` lines. **`skram doctor` always exits 0**, including when
it found problems — read the count, do not test the exit code. If the config
file itself cannot be read, that is the only `[!!]` line and the run stops
there:

```text
[ok] Engine v0.13.0 (binary v0.13.0-11-gefe49fe)
[!!] Config: config file not found at /home/you/.config/skram/config.yaml — run 'skram init' to create one

1 issue(s) found.
```

## Running out of disk

Two commands reclaim space, and neither is automatic on its own.

**Old jobs.** Each job keeps three files forever until something removes
them. `skram prune` removes job directories older than a cutoff — 30 days for
completed jobs, 90 days for failed and killed ones by default. Running jobs
and jobs still referenced by the queue are never removed, and neither is a
directory with no `status.json`.

```bash
skram prune --dry-run
skram prune --older-than 720h --keep-failed 2160h
```

**Docker images and build cache.** `skram reap` removes timestamped image
tags beyond a keep count and prunes build caches to a budget, on the host
Docker daemon only: registries, cluster runtimes, and volumes are never
touched. Named tags never match the pattern and are never removed.

```bash
skram reap --dry-run
skram reap --cache-only --cache-budget 20GB
```

It reads the [`reaper:` section of the config](config.md) for which
repositories are eligible and what the budgets are; the flags override it for
one run. If
`reaper.after_targets` is configured, a reap also runs inside a matching job
after it succeeds; `SKRAM_REAP_DISABLE=1` skips that automatic pass and
leaves `skram reap` alone.

## Environment variables

These are the variables the released binary reads. Nothing here needs to be
set for normal use.

| Variable | Effect | Default | Read by |
|---|---|---|---|
| `SKRAM_ACTOR` | Names who enqueued, on every item and in `status`. Overrides the detection below. | unset | every command that enqueues, holds, or reports; the processor puts the item's actor back into the job's environment, so the job's own record names whoever asked for it |
| `SKRAM_GUARD` | `0`, `false`, or `off` turns the guard off for that command, so a command line it would block runs instead. Any other value leaves the guard on. | unset (guard on) | the guard hook |
| `SKRAM_HERDR` | Exactly `0` turns off herdr reporting — the row the processor keeps in each workspace's sidebar. Any other value leaves it on. | unset (on where herdr is installed) | the processor's herdr reporter, and `skram doctor` |
| `SKRAM_NOTIFY` | Exactly `0` turns off job-end notifications, herdr and macOS alike. Any other value leaves them on. | unset (on) | the job-end notification, and `skram doctor` |
| `SKRAM_REAP_DISABLE` | Exactly `1` skips the automatic reap that runs inside a job matching `reaper.after_targets`. `skram reap` run by hand is unaffected. | unset (automatic reap on where configured) | the post-job reap |

When `SKRAM_ACTOR` is unset the actor is worked out from the environment the
caller is already in, first match winning: `CLAUDECODE` or `CLAUDE_CODE` set
gives `claude-code`; `CURSOR_TRACE_ID` or `CURSOR_AGENT` gives `cursor`;
otherwise `USER`; otherwise `unknown`. Inside a herdr session
`HERDR_WORKSPACE_ID` and `HERDR_SESSION` are recorded beside the name.

Variables that are not skram's own but change what it does:
`XDG_CONFIG_HOME` and `XDG_DATA_HOME` move the config file and the data
directory (`--config` overrides the first for one command);
`NGROK_AUTHTOKEN` is the token `skram dashboard` uses for a tunnel; `PAGER`
is what the TUI opens a log in; `HERDR_BIN_PATH` points at the herdr
executable when it is not on `PATH`.

## Still stuck

Open an issue on the public repository with the output of `skram version` and
`skram doctor`, your operating system, and what you expected instead.
