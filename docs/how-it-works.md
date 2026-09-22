# How Skram works

Skram has no server and no database. One config file says what your projects
are, one JSON file holds the queue, and each job gets a directory of its own.
Every command reads and writes those files directly, so one queue is shared by
every shell, checkout, and agent on the machine with nothing to start first.

This page covers the behaviour that `--help` cannot tell you: what is on disk,
when the processor runs, how an item that needs two resources is scheduled,
what a hold stops that a drain does not, what an ephemeral target really is,
what `--on-error stop` cancels, where estimates come from, and what `wait`
exits with.

- [A job on disk](#a-job-on-disk)
- [The queue file](#the-queue-file)
- [The processor](#the-processor)
- [Which project a bare `skram run` means](#which-project-a-bare-skram-run-means)
- [Lanes, and the rule for an item with two resources](#lanes-and-the-rule-for-an-item-with-two-resources)
- [Hold, drain, kill, cancel](#hold-drain-kill-cancel)
- [Ephemeral targets and `--now`](#ephemeral-targets-and---now)
- [When a job fails: `--on-error`](#when-a-job-fails---on-error)
- [Estimates](#estimates)
- [Attaching, waiting, and exit codes](#attaching-waiting-and-exit-codes)
- [The dashboard](#the-dashboard)
- [The metrics stack](#the-metrics-stack)

## A job on disk

An **item** is an entry in the queue that has not started. A **job** is that
item once it has: it has a process group, a directory, and an exit code.

Every job gets a directory named after its job id, under the logs directory
(`~/.local/share/skram/logs` unless `logs_dir` moves it):

```text
~/.local/share/skram/logs/J20260921_194852_f1fca4/
  raw.log        what the target printed, stdout and stderr interleaved,
                 with the exec line and the exit code around them
  events.jsonl   one JSON object per line: job_start, a log event per output
                 line (with its stream), job_end with status and duration
  status.json    the job's record: command, project, target, actor, PID,
                 start and end times, exit code, duration, failure reason
```

The job id is the date and time the job started plus a random suffix, so a
directory listing is in start order. Secrets are redacted on the way into
`raw.log` and `events.jsonl`.

Those three files are the only place job output lives, and everything else
reads them:

```bash
skram logs J20260921_194852_f1fca4     # raw.log for one job
skram logs -F                          # follow the newest job's raw.log
skram logs <job-id> --json             # events.jsonl instead, one object per line
skram status                           # running and recent jobs, from status.json
skram explain <item-or-job>            # status, reason, stderr tail, what to run next
```

Nothing cleans them up on its own. `skram prune` removes old job directories;
a running job, a job the live queue still refers to, and a directory with no
readable `status.json` are always kept.

## The queue file

The queue is one JSON file and a lock beside it:

```text
~/.local/share/skram/queue/
  queue.json            the queue: the hold, the processor's PID, every item
  queue.lock            the advisory lock taken before any read-then-write
  processor.lock        held by the processor for its whole life; a second one exits
  queue-processor.log   the processor's own output
  queue-history/        archived queues
```

`queue.json` holds the queue-wide failure policy, the hold if there is one,
the PID of the processor, and the items. Each item carries its id (`q_…`), the
project and target, the **actor** who enqueued it (a person, or an agent such
as `claude-code`), the lanes it occupies, its status, and once it has run, the
job id, the exit code, and the duration. The actor is stamped at enqueue time
and passed down to the job, so `status.json` names whoever asked for the work
rather than the processor.

```bash
skram queue list            # one line per item, with the actor
skram queue list --json     # the file as stored, plus liveness and counts
```

Every command that changes the queue takes `queue.lock` first, and writes
`queue.json` by replacing it whole, so a concurrent reader never sees half a
queue. That is the entire concurrency mechanism.

When the queue is idle and has grown past 100 items, the next enqueue moves it
into `queue-history/` and starts a fresh one; `skram queue reset` does the same
on demand. Archived items stay readable — `skram explain` looks in the history
as well as the live queue.

`skram queue add` writes an item into the file directly. Its argument is the
`skram` command line the processor should run, without the leading `skram`:

```bash
skram queue add "run --foreground apps test"
```

```text
Added item q_eb096823: run --foreground apps test
```

This is the low-level form of `skram run`, and `skram run` is almost always
what you want. An item added this way carries the actor, and nothing else: no
project, no target, and so no resource, no lane of its own, and no estimate.
It also starts no processor, so it sits pending until something else does —
the next `skram run`, `skram run -f`, or `skram wait`. An argument that is not
a `skram` command line fails the way `skram` would, with the message in
`queue-processor.log`.

## The processor

Nothing is installed and no daemon runs in the background waiting for work.
The first enqueue that finds no live processor starts one, detached from your
terminal, and says so:

```text
Queue processor started (PID 48138)
```

Its output goes to `queue-processor.log` in the queue directory — the reason a
job was cancelled, or the queue was left alone, is written there.

The processor is one loop. On each pass it takes the lock, re-reads the queue,
starts every item that can run now, then waits until a job finishes or five
seconds pass, whichever comes first. When nothing is running and nothing is
pending, it records a processor PID of `0` and exits. A machine that has
finished its work is left with no Skram process running.

Two enqueues at the same moment on an idle queue may both start a processor,
because neither can yet see the other's. The first to take `processor.lock`
is the processor; the other exits at once, before it reads the queue, so
there is never more than one scheduler and a lane never runs two jobs. A
processor also treats a lane as taken while the file shows a running item
there whose job is still alive, even one started by a processor that has
since died.

A processor that dies takes nothing with it: the items stay pending, and the
next enqueue starts a new one. `skram run -f` and `skram wait` also start one
if they find a pending item and no live processor, so blocking on an item can
never wait forever on a processor that is gone.

## Which project a bare `skram run` means

`skram run <project> <target>` never guesses. With the project left out,
Skram works it out in this order, and the first answer wins:

1. `--context <project>`, which beats everything below so that a script
   setting it is not surprised by where it is run from;
2. the project whose `path:` contains the current directory, or that the
   `repos:` entry for this checkout names (with `default:` breaking a tie
   between several);
3. the **active context**, the project last named by `skram context use`.

With none of the three, the run is refused and the message lists what to do.
A checkout that several projects apply to, with no `default:`, is refused the
same way, naming the candidates.

```bash
skram context list           # the projects, with the active one marked
skram context use apps       # set it
skram context current        # print it
```

The active context is a single line in `~/.config/skram/context.yaml`
(`$XDG_CONFIG_HOME/skram/context.yaml` when that is set). It is machine
state, not configuration: it is not part of the config file, it is not read
by anything else, and deleting the file just leaves you with no active
context. It is the fallback for working outside any checkout; inside one,
step 2 answers first and the active context is never consulted.

## Lanes, and the rule for an item with two resources

A **resource** is the shared thing a project's targets must not run against
concurrently — a Docker daemon, a local cluster, a port. A project declares
one or more, and each one is a **lane**: a single-file line of items, one
running job at a time.

```yaml
projects:
  my-app:
    resource: [docker, cluster]   # this project needs both
  other-app:
    resource: [cluster]
  docker-app:
    resource: docker
```

With no `resource:`, a project gets a lane named after itself. Two projects
that name the same resource share its lane and take turns; two projects with
nothing in common run at the same time.

An item may run when it is at the head of **every** lane it names and none of
those lanes is busy. Both halves matter:

- A job in the `cluster` lane blocks `my-app`, because `my-app` needs
  `cluster` as well as `docker`.
- While `my-app` waits, it holds its place in `docker` too. An item behind it
  that only needs `docker` waits as well, even though `docker` is free.

That second part is deliberate. A pending item blocks its lanes for everything
behind it, which keeps each lane in order and stops a two-resource item from
being overtaken forever by single-resource work.

A workflow (`skram workflow run`) is one item under this rule. It names every
lane its steps' projects declare, waits for all of them, and holds all of
them until its last step ends, so a step never runs beside a queued job on a
resource the step's project protects. `skram status` shows it as
`_workflow/<name>` with its lanes.

`skram status` shows a Lanes block whenever there is more than one lane, with
what is running and what is queued in each. A lane can also be addressed
directly:

```bash
skram wait --resource cluster     # block until that lane is empty
skram kill --resource cluster     # kill whatever is running in it
```

## Hold, drain, kill, cancel

Four different things stop work, and they are not interchangeable.

| | What it stops | What it lets finish | How it ends |
|---|---|---|---|
| `skram hold` | the whole queue, for everyone | the running jobs | only `skram release` |
| `skram queue drain` | the processor starting anything new | the running jobs | the next enqueue |
| `skram kill` | one running job | nothing | — |
| `skram queue cancel` | one pending item | — | — |

**A hold** stops the queue for everyone until it is lifted. The jobs already
running finish; nothing else starts. New items still queue normally, and
whoever enqueues one is told why they are waiting:

```bash
skram hold --reason "upgrading the cluster"
skram release
```

```text
Queued my-app/test (item q_83a39d9f, 1 pending)
HELD by alice: upgrading the cluster — starts after 'skram release'
```

With `--json`, the same enqueue reports `"held": true` and the hold's actor,
reason, and start time, so an agent can back off rather than block.

Nothing bypasses a hold. `skram run`, `skram run -f`, and `skram wait` will not
start a processor while one stands, and a running processor exits at its next
pass; the hold is stored in `queue.json`, so it outlives that processor and any
reboot. `skram release` clears it and starts a processor again if anything is
pending.

**A drain** is the one-shot version: the processor finishes the jobs it is
running, starts nothing new, and exits. Pending items stay queued and the flag
is consumed as the processor goes, so the very next enqueue starts a processor
and the queue picks up where it left off. Use a drain to stop the machine
doing work now; use a hold when nobody else should start work either.

**A kill** ends one running job. Skram sends SIGTERM to the job's whole
process group — the script and everything it started — and follows with
SIGKILL if it is still alive after the grace period. With no argument it kills
the single running job, or lists them and asks you to choose.

**A cancel** withdraws pending items, which never started and have no job
directory:

```bash
skram queue cancel q_866bfad7      # one item
skram queue clear                  # every pending item
```

## Ephemeral targets and `--now`

A target listed in a project's `ephemeral:` is one the author has declared
read-only — a `status`, a `logs`, a `ps`. It runs immediately, in your
terminal, with no lane, no item, and no job directory, and exits with the
script's own code. Because it never touches the queue, a hold does not
affect it.

```yaml
projects:
  my-app:
    ephemeral: [status]   # read-only targets, declared by you
```

```bash
skram run my-app status     # runs now; nothing is queued, nothing is logged
```

`--now` is not a way around the queue. On an ephemeral target it changes
nothing, because such a target already runs immediately. On anything else it
is an error:

```text
Error: target "test" is not ephemeral; use -f to attach to the queue,
or add it to ephemeral: in the project config
```

So `--now` is only ever a statement of intent: it says "I expect this to be
free", and Skram tells you when you are wrong instead of running a build
beside someone else's.

## When a job fails: `--on-error`

When a job fails, the policy on its item decides what happens to the items
still waiting. `--on-error stop` is the default; `--on-error continue` leaves
the queue alone.

```bash
skram run my-app build --on-error continue
```

`stop` cancels the pending items in the **lanes of the failed item** — that
means every project sharing those resources, and only those lanes. It is not
scoped to the project. Given the three projects above, with `other-app/fail`
running in `cluster` and three items behind it:

| Item | Lanes | After `other-app/fail` fails |
|---|---|---|
| `my-app/test` | cluster, docker | cancelled — shares `cluster` |
| `other-app/test` | cluster | cancelled — shares `cluster` |
| `docker-app/test` | docker | runs — shares no lane with the failure |

A cancelled item records what cancelled it, so `skram queue list`, `skram wait`,
and `skram explain` all say "after failure of q_…" rather than leaving you to
guess. After a failure, `skram explain` is the fastest way back: it prints the
status, the classified reason, the tail of stderr, and the commands to run
next.

## Estimates

Estimates are read out of the `status.json` files of past jobs. There is no
separate database, so deleting a job directory (or letting `skram prune` do it)
also removes it from the history.

A run is keyed by an array: the project, the target, then the arguments you
passed, then the flags, sorted so that `--a --b` and `--b --a` share a history.
A lookup tries the whole key, then drops one element at a time, down to the
project alone:

```bash
skram estimate my-app build              # key [my-app, build]
skram estimate my-app build --scope api  # a new flag combination
skram estimate my-app deploy             # a target that has never run
```

```text
key: [docker-app, build, api, --scope]
estimate: ~2s  (from 2 runs · target; matched [docker-app, build]; last run 11s ago)
```

So a flag combination you have never used inherits the target's history, and a
brand new target inherits the project's. `skram estimate` always prints which
key it matched, so you can see how specific the answer is.

Like `skram run`, `estimate` takes the project from the current directory (or
`--context`) when you leave it out, and `--json` prints the whole answer — key,
matched key, and the estimate in seconds — as one object:

```bash
skram estimate build                     # project from the current directory
skram estimate --context my-app build    # or named explicitly
skram estimate my-app build --json       # machine-readable
```

The number itself is a weighted average over the ten most recent runs, which
follows a machine that got faster without letting one outlier dominate. Only
runs that count are used:

- failed and killed runs are never used — a build that crashed after three
  seconds says nothing about a real one;
- samples older than 30 days are ignored;
- Skram's own flags (`-f`, `--now`) never become part of a key, so attaching
  to a job does not fragment its history.

The same estimate appears at enqueue, as the remaining time in `skram status`,
as the queue and lane ETAs, and as the deadline behind `skram wait --timeout
auto`.

## Attaching, waiting, and exit codes

`skram run` returns as soon as the item is queued. Two ways to block on it:

```bash
skram run my-app build -f          # enqueue, then attach: stream the job's output
skram wait q_866bfad7              # block on an item or a job already queued
skram wait --last                  # the most recently enqueued item
skram wait --queue --timeout auto  # until nothing is pending or running
```

`-f` does not jump the queue. The item is queued exactly as it would be; your
terminal just follows the job once it starts. Ctrl-C detaches the terminal and
leaves the job running — Skram prints how to get back to it.

`--timeout auto` is twice the estimate, never less than ten minutes (ten
minutes with no history at all). The deadline it chose is printed to stderr,
and reported as `timeout_seconds` with `--json`.

Both `run -f` and `wait` exit with the job's own exit code, so they can be used
directly in a script. Four other codes are Skram's own:

| Exit code | Meaning |
|---|---|
| the job's | the target ran and this is what it returned |
| 124 | `--timeout` elapsed first; the job may still be running |
| 125 | the item was cancelled before it ever ran |
| 137 | the job is recorded as killed |
| 1 | the job ended in a state Skram could not read |

A timeout is worth distinguishing from a failure: the item may simply be
behind a hold, and on a timeout `wait` says so, naming who holds the queue and
why. A job you stop with `skram kill` is recorded as killed, so `wait` and
`run -f` exit 137 for it.

## The dashboard

`skram dashboard` serves the same three files over HTTP: the queue, the
running jobs, and each job's log streamed line by line as it is written.

```bash
skram dashboard                 # start it, print the URL and a QR code
skram dashboard --local         # this machine only: 127.0.0.1, no tunnel, no QR code
skram dashboard --no-qr         # no QR code
skram dashboard --port 9000     # a port other than 8484
skram dashboard status          # is one running, on what URL, since when
skram dashboard stop            # stop it
```

Starting it detaches a child process and returns; the URL, the port, the PID,
and the mode are written to `dashboard-state.json` in the data directory, and
that file is what `status` and `stop` read. The child's own output goes to
`dashboard.log` beside it. One dashboard runs at a time: starting a second
reports the first.

The URL carries a token, and every request needs it, so a link is only as
private as you keep it. `--no-auth` turns that off, which is only sensible on
a network you trust. Without `--local`, Skram opens an
[ngrok](https://ngrok.com) tunnel so the URL works off the machine — that is
the one thing in Skram that talks to a third party, it needs
`NGROK_AUTHTOKEN`, and `--local` never does it. `--local` listens on
`127.0.0.1` and prints a `http://127.0.0.1:<port>` URL, so it opens only in a
browser on the same machine; other machines on your network cannot reach it,
and it prints no QR code.

## The metrics stack

Skram can export its job history as Prometheus metrics and bring up a local
stack to look at them:

```bash
skram metrics up            # docker compose up -d, then print the URLs
skram metrics status        # what the compose project has running
skram metrics down          # stop it
```

Read this part before you try it. **The stack is not in the release
archive.** `skram metrics up` runs `docker compose` against the first
`monitoring/docker-compose.yml` it finds, looking in the data directory
(`~/.local/share/skram/monitoring/docker-compose.yml`), then beside the
binary, then in the directory you are standing in. An archive that carries
only the binary, the licences, the completions, and the herdr plugin has none
of those, so all three commands fail with:

```text
Error: monitoring stack not found: docker-compose.yml not found — ensure
monitoring/ directory exists in skram data dir (…) or current directory
```

To use it, write that compose file yourself. Skram runs `docker compose` on
it with the logs directory passed in as `SKRAM_LOGS_DIR`, so the services can
mount the job records read-only; everything else about the stack is yours to
decide. The URLs `metrics up` prints afterwards assume Grafana on 3333,
Prometheus on 9090, Loki on 3100, and a job-metrics exporter on 2112, so
those are the ports to publish if you want the printed links to work.

Nothing else in Skram depends on any of this. The job records the stack would
read are the same ones `skram status`, `skram logs`, and `skram explain`
already read straight off the disk.

---

Elsewhere: the docs [index](README.md) lists every page, and the
[README](../README.md) has the install steps and the first loop. Every
configuration key — `resource:` and `ephemeral:` above, and the two
directories — has its entry in the [configuration reference](config.md), and an item that
never starts is the first subject of [troubleshooting](troubleshooting.md).
The same behaviour reached by an agent is on the [agent setup](agent-setup.md)
and [MCP](mcp.md) pages.
