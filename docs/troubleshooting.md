# Troubleshooting

Start here when something you asked for is not running. Three commands answer
most of it:

```bash
skram status                    # held? what is running, and who asked for it
skram queue list                # every item in order, with its id and actor
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
ETA: no queue estimate yet (2 pending, 2 without history)
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

A drained queue says so, in text as well as `--json`:

```text
Queue: 1 pending, 1 running, 0 completed, 0 failed
Processor: running (PID 64455)
Draining: processor stops after the running job(s); pending items stay queued
```

Draining is only meaningful while a processor is running — it is a message to
that processor. Asking to drain when none is running does nothing and says so,
rather than leaving a flag that would stop the next processor before it takes
an item:

```text
No processor is running, so there is nothing to drain; 1 pending item(s) stay queued.
```

### 3. Is another job holding the lane?

A lane is the single-file line in front of one shared resource. Every project
that names that resource shares the lane, whatever repo it lives in, and one
job at a time runs in it. Lanes run in parallel with each other.

So an item can be pending while other jobs run — that is correct, not stuck.
`skram status` shows the lanes when there is more than one:

```text
Running:
  apps/build  J20260921_195625_3be1c0  (PID 64458, 1s elapsed)  by claude-code
    Lane: kind-dev
  api/build  J20260921_195625_9a07d4  (PID 64457, 1s elapsed)  by claude-code
    Lane: kind-staging

Queue: 1 pending, 2 running, 0 completed, 0 failed
Processor: running (PID 64455)
Lanes
  kind-dev      running apps/build (est. unknown) · 1 pending (1 without history)
  kind-staging  running api/build (est. unknown) · idle after
```

`skram queue list` gives the order within the whole queue:

```text
  1. [running] apps/build [kind-dev]  q_5c0e2a71 → J20260921_195625_3be1c0  by claude-code
  2. [pending] apps/test [kind-dev]  q_a49121b2  by claude-code
  3. [running] api/build [kind-staging]  q_d3f86b19 → J20260921_195625_9a07d4  by claude-code

1 pending, 2 running, 0 completed, 0 failed
```

Item 2 waits because item 1 has `kind-dev`. Item 3 runs anyway because its
lane is a different one. If you did not expect two projects to share a lane,
check their `resource:` values; if you expected them to share one and they do
not, they are in different lanes by accident.

A job sent to another machine waits in that runner's own lane,
`<resource>@<runner>` (`kind-dev@studio`), so it never waits behind this
machine's jobs on the resource. A pending item that `queue list` marks
`waiting for …` or `waiting for lease …`, or that `status` lists under
"Waiting for a runner:" or "Waiting for a lease on a runner:", is waiting on
the other machine, not stuck here: see [When the runner is
off](runners.md#when-the-runner-is-off) and [Sharing a resource with the
runner](runners.md#sharing-a-resource-with-the-runner).

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

The tell is a `Queue:` line with pending work but no `Processor:` line under
it: nothing is scheduled to take that pending item.

```text
No running jobs.

Queue: 1 pending, 0 running, 3 completed, 0 failed, 1 killed
ETA: queue clears in ~3s (1 pending)
```

What happened to the job that was running when the processor died:

- **If its own process survived, it kept running.** A job is its own process
  group, not a child that dies with the processor, so it finishes and writes
  its own result; it shows in the `Running:` block with its row still
  `running`, correctly. Losing the processor loses the scheduler, not the work.
- **If its own process died too** — a reboot, rather than the processor alone
  — the next `skram status` reconciles the row: it takes the job's own result
  when the job wrote one, otherwise it is marked killed. That is the `1 killed`
  above; the counts no longer lie, so `skram queue reset` is rarely needed.
- **The pending item starts on the next enqueue.** `skram release` also
  starts a processor when anything is pending.

If you would rather start from an empty queue, archive it and start fresh.
History already written to job directories is untouched:

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
that starts in column 0. An ops script's arm can open a multi-line body, or
be written entirely on one line — either way, a trailing comment becomes the
target's help:

```bash
build) # Build the image
    docker build -t app .
    ;;

push) docker push app ;; # Push the built image
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
apps/boom  J20260921_195642_7417fd  [failed] exit=3 (<1s)  by you
Lane: kind-dev
Reason: unknown — could not reach the cluster
Last 2 lines (all streams):
  bash /home/you/dev/apps/ops.sh boom
  could not reach the cluster
Next:
  - no rule recognised this failure; reason_detail is the last line that looked like an error, and the tail above has more: skram logs J20260921_195642_7417fd --tail 80
  - re-run: skram run apps boom
```

It takes a job id or an item id, `--lines N` for a longer tail, and `--json`
for the same fields as an object. When the tail is not enough, the full log
is `skram logs <job>`.

The reason comes from one ordered list of rules over the last 200 lines of
each stream, and `reason_detail` is the line the winning rule matched — for a
red test suite, the test runner's own summary line. The list is in
[MCP](mcp.md#explain).

A job that ended before its target ran has the reason `start_failed`, and the
detail names what stopped it: the job's process could not be started, the
config no longer loads, the checkout or the project's Makefile, script,
Taskfile or justfile is gone, or the target's tool (`make`, `bash`, `task`,
`just`) is not on the job's `PATH`. Nothing of the target ran. Fix that cause,
run `skram doctor` to check the rest of the machine's setup, and re-run the
job.

Exit codes a caller can branch on: the job's own code, or `124` for a
`--timeout` that expired, `125` for an item cancelled before it ran, and
`137` for a job that was killed. `skram wait` and `skram run -f` use the same
three; [how it works](how-it-works.md) has the table and its one caveat.

## Runner jobs

A job sent to a runner ([runners](runners.md)) fails or waits for reasons a
local job does not have. `skram runner list` shows, without contacting
anyone, what the last hello to each runner found: whether it answered and
how long ago, its free disk, and its Docker engine. `skram runner check
<name>` asks again and finds most problems; the rest are below.

### `error saving credentials` (-25308) or `User interaction is not allowed`

`docker login` or `helm registry login` in a runner job fails with
`error saving credentials … (-25308)`. A process started over SSH cannot
reach the macOS keychain, which Docker's default credential store uses. On
the runner, copy `~/.docker/config.json` without its `credsStore` key, and
the `~/.docker/contexts/` directory, into a second directory:

```bash
mkdir -p ~/.docker-jobs
cp -R ~/.docker/contexts ~/.docker-jobs/
grep -v credsStore ~/.docker/config.json > ~/.docker-jobs/config.json
```

Then pass that directory on each job, with `HELM_REGISTRY_CONFIG` as well
when the job uses helm (use the absolute path of the directory):

```bash
skram run --runner studio api push --env DOCKER_CONFIG=/home/ci/.docker-jobs --env HELM_REGISTRY_CONFIG=/home/ci/.docker-jobs/config.json
```

### Items sit in `waiting_for_runner`

A runner that is asleep, off, or not answering SSH is waited for, not failed:
the item stays pending and is tried again about once a minute. `skram runner
check <name>` says which: when SSH works but the sleep row reads `sleep
allowed`, the runner slept between jobs (a running job holds it awake, see
[Keeping machines awake](runners.md#keeping-machines-awake)). Waking the
runner is enough for the item to start; do not enqueue it again. See [When the
runner is off](runners.md#when-the-runner-is-off).

## Reading `skram doctor`

`skram doctor` is the first thing to run on a machine that is behaving
oddly. Every line starts with one of three markers:

| Marker | Meaning |
|---|---|
| `[ok]` | Checked, and fine. |
| `[!!]` | A problem. Counted in the total at the end. |
| `[--]` | Information: something optional that is off, absent, or worth knowing. Never counted. |

The `disk` line is the free space on the volume holding Skram's data, measured
now, against [`preflight.min_free_disk`](config.md#preflight). The `docker`
line appears whenever a `docker` program is on `PATH`: it names the engine the
program's current context reaches (on a machine with both Docker Desktop and
OrbStack, the one it picks), the context, and the free space inside the
engine. When a project that runs on this machine names `docker` in its
[`tools:`](config.md#tools), an engine that does not answer, one below
[`preflight.docker_min_free`](config.md#the-docker-engine), or no `docker` on
`PATH` is a `[!!]` line with a `fix:` under it, since a job of that project
would fail the same way. When no project uses Docker, the same line is `[ok]`
or `[--]` and never counts as a problem. Each runner's line shows the same two
values as that runner reported them.

Indented lines belong to the line above: an indented `[!!]` is a problem with
that project in particular, and an indented `run:` line is the command that
fixes what precedes it.

```text
[ok] Engine v0.13.0 (binary v0.13.0-11-gefe49fe)
[ok] Config loaded (3 project(s), 0 workflow(s))
[ok] Data directory: /home/you/.local/share/skram/logs
[ok] Data directory: /home/you/.local/share/skram/queue
[ok] disk on this machine (the volume holding /home/you/.local/share/skram): 120GB free (preflight.min_free_disk 5GB)
[ok] docker on this machine: OrbStack (context orbstack), up, 31GB free
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
counting the `[!!]` lines. `skram doctor` exits 1 when that count is above
zero and 0 when every check passed, so a script can test it; `[--]` lines
never change the exit code. If the config file itself cannot be read, that is
the only `[!!]` line and the run stops there, with exit 1:

```text
[ok] Engine v0.13.0 (binary v0.13.0-11-gefe49fe)
[!!] Config: config file not found at /home/you/.config/skram/config.yaml — run 'skram init' to create one

1 issue(s) found.
```

## Running out of disk

A `disk_full` job usually never started: the
[preflight](config.md#preflight) found less free space than
`preflight.min_free_disk` (default 5GB) on the machine it would have run on,
named in `reason_detail` with the space measured and the floor. Nothing was
recorded in the ledger for it. Free space on that machine, then run it
again; re-queueing it before then fails it the same way. When the
[`reaper:`](config.md#reaper) block is on, Skram has already reaped once
there before failing the job, and `reason_detail` ends with `a reap was
tried first` and what it freed: what is left is not the reaper's to remove
(other images, volumes, a cluster, old job directories). `skram doctor`
shows this machine's free space now, and `skram runner list` each runner's
as of its last hello.

A `disk_full` job that did start means the disk filled while it ran:
the target printed `no space left on device`, and `reason_detail` is that
line. Inside a container that is the Docker engine's own disk, which can be
full while the host has room. Free space the same way, then run it again.

When `reason_detail` says **the Docker engine's storage**, it is the disk
inside Docker Desktop or OrbStack that is short, against
`preflight.docker_min_free` (default 10GB), whatever the host disk shows.
Free space inside the engine: take down a local cluster that holds images
and volumes (a `cluster-down` target, if the project has one), remove images
and build cache (`skram reap --dry-run` shows what the reaper would remove),
or raise the engine's disk limit in its settings.

### The Docker engine is down

A job that fails with reason `docker_down` never started: its project's
`tools:` names `docker`, and the engine the `docker` program's current
context reaches did not answer within 30 seconds. `reason_detail` names the
machine, the context, and the engine's own error line. Skram has already
[tried to start it once](config.md#starting-a-stopped-engine) when it is
Docker Desktop or OrbStack and `preflight.start_docker` is not `false`;
`reason_detail` then ends with the command it ran and how that ended.
Start or restart the engine on that machine by hand (Docker Desktop,
OrbStack, or the daemon), check it with `docker info`, then run the job
again. Reaping cannot help: it needs the engine to answer.

A job that had already started can fail with `docker_down` too: it printed
that the engine stopped answering (`Cannot connect to the Docker daemon`,
`Docker Desktop is unable to start`), and `reason_detail` is that line. The
fix is the same. Neither reason says anything about the code.

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
skram reap --runner mini --dry-run
```

It reads the [`reaper:` section of the config](config.md) for which
repositories are eligible and what the budgets are; the flags override it for
one run. If `reaper.after_targets` is configured, a reap also runs inside a
matching job after it succeeds; `SKRAM_REAP_DISABLE=1` skips that automatic
pass and leaves `skram reap` alone. `--runner <name>` reaps a
[runner](runners.md#reaping-a-runners-docker) instead, with this machine's
`reaper:` section; a matching job sent to a runner reaps there.

## Environment variables

These are the variables the released binary reads. Nothing here needs to be
set for normal use.

| Variable | Effect | Default | Read by |
|---|---|---|---|
| `SKRAM_ACTOR` | Names who enqueued, on every item and in `status`. Overrides the detection below. | unset | every command that enqueues, holds, or reports; the processor puts the item's actor back into the job's environment, so the job's own record names whoever asked for it |
| `SKRAM_GUARD` | `0`, `false`, or `off` turns the guard off for that command, so a command line it would block runs instead. Any other value leaves the guard on. | unset (guard on) | the guard hook |
| `SKRAM_HERDR` | Exactly `0` turns off herdr reporting — the row the processor keeps in each workspace's sidebar. Any other value leaves it on. | unset (on where herdr is installed) | the processor's herdr reporter, and `skram doctor` |
| `SKRAM_NOTIFY` | Exactly `0` turns off job-end notifications, herdr and macOS alike. Any other value leaves them on. | unset (on) | the job-end notification, and `skram doctor` |
| `SKRAM_REAP_DISABLE` | Exactly `1` skips the automatic reaps: the one inside a job matching `reaper.after_targets`, and the one the [preflight](config.md#reaping-once-before-failing) tries before failing a job on disk. `skram reap` run by hand is unaffected. Set on the machine that sends a job, it skips the job's reaps on the runner too. | unset (automatic reap on where configured) | the post-job reap, and the processor (the preflight's reap, and when it sends a job to a runner) |
| `SKRAM_SSH` | The program every call to a [runner](runners.md) goes through, git's included, invoked as `<program> <destination> <command>`. | unset (`ssh`) | the processor, when it sends a job to a runner |

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
