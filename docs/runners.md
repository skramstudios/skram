# Runners

A runner is another machine a job can be sent to. The queue stays on this
machine: you enqueue, `wait`, and read `logs` here as for any job, and the
processor sends the job to the runner over SSH when its lane lets it start.
The code goes as a commit, the output comes back as the job runs, and the
job's files come home when it ends, so every command that works on a local
job works on this one.

A runner needs `git`, the tools the target itself runs, an SSH login this
machine can use without a prompt, and the same operating system and CPU
architecture as this machine. It needs no Skram config and no Skram install:
this machine copies its own binary there.

The job gets its `PATH` from your login shell on the runner (`$SHELL -l`),
so what `~/.zprofile` or `~/.bash_profile` adds there, such as Docker
Desktop's `~/.docker/bin`, reaches the job the way it reaches a local one.
`~/.zshrc` is read only by interactive shells, so a tool put on `PATH` there
alone is missing: check with `ssh <runner> "$SHELL -lc 'command -v docker'"`,
or pass `PATH` for one job with `--env PATH=…`, which wins. The rest of the
environment is the SSH command's.

## Declaring a runner

```yaml
runners:
  build-box: {}                  # reached as `ssh build-box`
  studio:
    ssh: me@studio.example.net   # any destination your SSH program accepts
```

A runner's name is its SSH destination unless `ssh:` says otherwise, and
`ssh:` is one destination (a host, an alias, or `user@host`), never a
command. A name starts with a lowercase letter or digit and continues with
lowercase letters, digits, `.`, `_`, and `-`. `local` is this machine and
cannot be declared.

## Sending a job

```bash
skram run --runner build-box api test --json   # queued here, runs there
skram wait <item> --json                       # exits with the target's code
skram logs -F <item>                           # the target's output, live
skram run -f --runner build-box api test       # attach, stream, exit with its code
skram run --runner local api test              # run it here after all
```

`--runner` goes before or after the target, and `--runner=NAME` works too. An
unknown name is refused at once, naming the runners you have. An ephemeral
target runs here at once, so `--runner build-box` on one is refused;
`--runner local` on one is fine.

The MCP `run` tool takes the same choice as `runner` (`local` runs it here);
see [MCP](mcp.md#run).

A sent job waits in its runner's own lane, `<resource>@<runner>`
(`cluster@studio`), because the runner has its own copy of the resource: two
jobs sent to one runner on one `resource:` run one at a time, while a local
job on that resource, or one sent to another runner, runs beside them. On
the runner it takes that resource's lease, so it never overlaps a job the
runner runs for itself or one another machine sent there: see
[Sharing a resource with the runner](#sharing-a-resource-with-the-runner).
Nothing picks a runner for you; a job goes where its `--runner` or its
project's `runner:` says.

### A project's default runner

```yaml
projects:
  api:
    path: ~/src/api
    makefile: Makefile
    runner: build-box      # every api job goes there unless a run names another
```

With `runner:` set, `skram run api test`, the MCP `run` tool, and the TUI's
Launch tab send the job to that runner with no flag, and `--runner local` (or
an empty `--runner=`) runs one here. An ephemeral target always runs here. A
name that is not in `runners:` is a config error. `skram discover`
(`runner` in `--json`) and the project's MCP resource
`skram://projects/<name>` show the default. A workflow's steps run here.

### Seeing where a job runs

A job sent to a runner carries `@<runner>` after its label wherever a label is
shown: `skram status`, `skram queue list`, `skram logs`, the TUI's Jobs and
Queue tabs, the dashboard, and the herdr sidebar (`build (api--T-3) @build-box
1m20s/4m10s`). The item, its `status.json`, and the JSON of `run`, `status`,
`queue list`, `wait`, and `explain` carry `runner`. A job's estimate is the
same wherever it runs.

Lanes name the runner too: `skram status`'s Lanes block and each running
job's `Lane:` line, `cluster@studio` in `queue list` after a live item's label
(`lanes` on every item in `--json`), `lanes` in `run --json`, `explain
--json`, and each running job in `status --json` (their `resources` stay the
resource names), and the TUI's Queue tab. A lane that only repeats the
label, a project with no `resource:` sent to `studio`, is not printed.
`skram wait --resource cluster@studio` and `skram kill --resource
cluster@studio` address that lane alone; `--resource cluster` is this
machine's.

## What goes to the runner

When a sent job's lane lets it start:

1. Skram asks the runner for its OS, CPU architecture, and Skram build. A
   runner of another OS or architecture is refused, with the reason on the
   job (`skram explain`). A runner with no Skram, or another build, gets this
   machine's binary, once; the job always runs under that copy, never
   whatever `skram` is on the runner's `PATH`.
2. The checkout is pushed to the runner's own clone of the repo, into a ref
   under `refs/skram/`, as one commit: `HEAD` itself when the checkout is
   clean, otherwise a commit on `HEAD` holding your uncommitted edits and the
   untracked files git does not ignore. Files git ignores (`node_modules`,
   build output, `.env` files) never leave this machine. Nothing is committed
   here: your index, stash, branches, tags, and remotes are untouched, in both
   repositories, and after the first job only the objects the runner lacks
   cross the network.
3. The runner checks that commit out into a worktree it keeps for this
   checkout: tracked files are reset to it and untracked files left by an
   earlier job are removed, but files git ignores stay for the next job, so
   caches persist between jobs of one checkout. Two jobs that would use one
   worktree on one runner (two projects in one checkout, or a job reading
   as an input the checkout another runs in) never run at once: the second
   stays pending until the first ends.
4. The job runs there detached, so it survives the SSH session, with a config
   Skram sends for that job. It never enters the runner's own queue.
5. While it runs, its output streams back line by line into this machine's
   job directory, so `skram logs -F`, `skram logs --json`, and `skram run -f`
   show it as it happens, and `skram status` shows the job running on its
   runner.
6. When it ends, its `raw.log`, `events.jsonl`, and `status.json` are copied
   into this machine's job directory, so `logs`, `explain`, history, and
   estimates read it like any other job.

The project must sit in a git checkout, and its script, Makefile, Taskfile, or
justfile must be inside that checkout (a project in a subdirectory of the
repo is fine), since only the repo's commit travels. A checkout with no
commit yet is refused.

The item, its `status.json`, and the JSON of `status`, `queue list`, `wait`,
and `explain` record what went as `sent`:

```json
"sent": {"head": "3f9c…", "commit": "a41e…", "dirty": true}
```

`commit` is exactly the code the job ran; `dirty` says whether it carried
uncommitted changes on top of `head`, and when it is `false`, `commit` is
`head`.

### Where things live on a runner

Everything a runner holds is under `$XDG_DATA_HOME/skram/runner` there
(`~/.local/share/skram/runner` by default), so removing that directory removes
all of it:

| Path | Holds |
|---|---|
| `bin/skram` | the binary this machine sent |
| `repos/<repo>.git` | one bare clone per repo, named by its `repos:` entry, else its folder; every checkout of a repo shares it |
| `worktrees/<repo>/<checkout>` | one worktree per checkout, named by the checkout's folder, where its ignored caches live |
| `logs/` | the runner's copy of each job's `raw.log`, `events.jsonl`, and `status.json` |
| `runs/<job>` | the config sent for one job and the detached process's output |

The leases are beside it, in `$XDG_DATA_HOME/skram/leases`: one file per
resource, shared by the runner's own queue and the jobs sent to it.

Two checkouts of one repo whose folders share a name would share a worktree, so
a job that reads both is refused; rename one.

### Inputs

A project with [inputs](config.md#inputs) sends every checkout it reads. Each
input's checkout, the one the job resolved here (`-C`, your working
directory, or the repo's default), goes to the runner like the project's own:
one commit into the runner's clone of that repo, checked out into a worktree
kept for that checkout. On the runner each input variable names that
worktree, and `SKRAM_INPUT_<VAR>_DEFAULT` is set only when the input is on
its default checkout (to the same worktree), since the default checkout
itself does not travel. Each input in the `inputs` of the item and
`status.json` records its own `sent`:

```json
"inputs": {"LIB_DIR": {"repo": "lib", "dir": "/src/lib--T-1", "sent": {"head": "…", "commit": "…", "dirty": true}}}
```

The config the runner receives holds the job's project, with its path and
every input's pointing at the runner's worktrees, and only the `repos:`
entries the job reads; no other project or repo leaves this machine. The
target sees the same `env:` values, `--env` values, `SKRAM_ACTOR`, and herdr
workspace and session as it would here. An input checkout removed before the
job starts fails it as `checkout_removed`, and the runner is never contacted.

## Sharing a resource with the runner

Every resource is guarded by a lease on the machine that has it. A job takes
the lease for each of its `resource:` names as it starts and gives it back
when it ends, on this machine for a local job and on the runner for a sent
one. The runner's own queue takes the same leases, so a job you send to a
Mac and a job that Mac queues for itself on the same resource never run at
once, in either order. Two machines share a resource only when both configs
give it the same name; a project with no `resource:` uses its own name.
This machine's leases never hold a job sent elsewhere, and a sent job takes
none here: a local job and a sent job on one resource run at once, each on
its own machine's copy.

When a job is sent and the runner's lease is held, nothing starts there and
the job does not fail: the item goes back to pending at the head of its lane,
waiting for the lease, and Skram asks the runner again about once a minute.
It is sent as soon as the lease is free. If the runner stops answering while
the item waits, it waits for the runner instead (`waiting_for_runner`, as in
[When the runner is off](#when-the-runner-is-off)), and once the runner
answers it is sent, or waits for the lease again. On the runner, an item of its own
queue whose resource is leased to a job sent there waits the same way and
starts by itself when that job ends.

```bash
skram status                  # "Waiting for a lease on a runner:" names the resource, runner, holder, and ETA
skram queue list --json       # the item carries waiting_for_lease
skram wait <item> --json      # waiting_for_lease on a timeout; the text wait names the holder on stderr
```

`waiting_for_lease` names who holds the resource and until when they should:

```json
"waiting_for_lease": {"resource": "cluster", "runner": "studio", "machine": "laptop",
  "actor": "claude-code", "job_id": "J20260929_101500_1a2b3c",
  "since": "2026-09-29T10:15:00Z", "eta": "2026-09-29T10:33:00Z"}
```

`machine` is where the holding job was queued, `since` is when it took the
lease, and `eta` is when its machine's estimate says it ends (absent with no
history).

On any machine, `skram status` lists the leases held there, marking each job
sent from another machine as inbound, with that machine, the actor, the job
id, and the estimated release. A lease held by a job already under
"Running:" is left out, so a machine with only its own jobs shows no leases
block; `skram status --json` has every lease as `leases`:

```text
Leases on this machine:
  cluster  job J20260929_101500_1a2b3c inbound from laptop (claude-code) since 10:15:00 (3m0s); free in ~15m
```

A lease whose job died (a crash, `kill -9`) or that was taken before the
machine last booted is taken over by the next job that wants the resource,
so a dead job never holds one for ever.
Each takeover is a `lease_reclaimed` line in
`$XDG_DATA_HOME/skram/leases/events.jsonl`, and the queue processor's log
names it too.

## Stopping a runner job

The controls are the ones for a local job, run on this machine:

```bash
skram kill <item>            # ends the job's processes on the runner; wait exits 137
skram queue cancel <item>    # a pending item: withdrawn here, nothing is sent
skram hold                   # running jobs finish, on runners too; nothing new starts
```

`kill` asks the runner to stop the job's whole process tree there and stores
it as killed, both there and here, before it returns. A job that is still
being sent is stopped as it starts. Stopping the queue processor (SIGINT or
SIGTERM) sends the same signal to every job it has on a runner and waits for
them to end, as it does for local jobs; a job still being sent, or waiting to
retry a call, stops at once and is stored as killed by that signal.

## When the runner is off

Each call made before the job starts is retried at once, then 15 s and 45 s
after it first failed. If the runner still cannot be reached, the job has not
started and does not fail: the item goes back to pending, ahead of everything
else in its lane, and waits for the runner. Other lanes keep running. About
once a minute Skram asks the runner again, and the job starts soon after it
answers. If the connection dropped after the start was sent, the job may
have started there anyway: when Skram next reaches the runner it kills that
job before sending the item again, and leaves its files on the runner. If
the runner answers but the job still cannot be sent there (its disk is full,
say), the job fails as `runner_unreachable` with the last error, rather than
waiting again.

```bash
skram status                                # "Waiting for a runner:" names the item, the runner, and since when
skram queue list --json                     # the item carries waiting_for_runner and unreachable_since
skram wait <item> --timeout 30m             # bounded as for any item (124 on timeout)
skram queue cancel <item>                   # stop waiting; nothing more is sent
```

`run -f` keeps attaching while the item waits.

## When the connection drops

A dropped connection is not a failed job. The job runs on, detached, on the
runner while this machine reattaches at once, then 15 s and 45 s after the
drop. Each reattach resumes the log from the last line this machine
holds, so `logs -F` and `run -f` go on with no line lost or repeated. A
connection that goes quiet counts as dropped too: while the job prints
nothing the runner sends a heartbeat every 5 s, and after 30 s with nothing
from the runner this machine ends the connection and reattaches. A
reattach that gets further starts the three tries again, and bringing the
job's files home at the end is retried the same way.

When all three fail, the job ends here as failed, reason `runner_lost` (even
if it finished there but its files never came home): `wait` and `run -f` exit
1, `explain` names the runner and gives the re-run, and its lane is free: the
next item in it starts, whatever `--on-error` says, since it was the
connection that failed, not the job.

The job may still be running there. Skram remembers it, and the next time it
reaches that runner (the next job sent there, or `skram doctor`) it kills the
job there and copies its `raw.log` and `events.jsonl` home, with the runner's
own `status.json` beside them as `runner-status.json`. The job's `status.json`
here stays `runner_lost`. `doctor` prints a line for each job it reclaims this
way.

## When it goes wrong

`skram explain <item>` names the reason, and its hints give the re-run and a
way to run it here instead:

| Reason | Meaning |
|---|---|
| `runner_refused` | The runner cannot take the job as things stand, and retrying would not help: another OS or architecture, a runner no longer in `runners:`, a checkout with nothing to send, or a copied binary that will not run there. Nothing ran. |
| `runner_unreachable` | The runner answered, but a call there kept failing before the job started. Nothing ran. |
| `runner_lost` | The job started, but contact was lost and every reattach failed. It may still be running there. |
| `runner_declined` | The runner's owner set `accept_runner_jobs: false` there (see [Declining runner jobs](#declining-runner-jobs)). Nothing ran, and nothing was retried. |

A runner that cannot be reached at all is not a failure: see
[When the runner is off](#when-the-runner-is-off).

## Checking a runner

`skram doctor` asks each runner what it is (giving up after 15 s) and prints
one line per runner:

```text
[ok] runner build-box: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0); data ~/.local/share/skram/runner
[--] runner studio (me@studio.example.net): reachable, darwin/arm64, no skram yet (the first job sends this machine's v0.19.0 (engine 0.19.0)); data ~/.local/share/skram/runner
[!!] runner old-box: unreachable through ssh: runner call hello exited 255: ssh: connect to host old-box port 22: Connection refused
[--] runner lab-mac: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0), declines runner jobs (accept_runner_jobs: false there); data ~/.local/share/skram/runner
```

A runner that is unreachable, or of another OS or architecture, is an issue
and makes `doctor` exit 1. One with no Skram or another build is information
only: the next job sends this machine's.

## Declining runner jobs

A machine's owner can refuse every job sent to it, from any machine's queue,
in their own config file (`~/.config/skram/config.yaml` there):

```yaml
accept_runner_jobs: false
```

The Skram that runs a job there reads only this key from that file, so the
file needs nothing else, and a missing or unreadable file accepts. A job sent
to a machine that declines fails at once, with no retries and nothing run
there: reason `runner_declined`, and `skram explain` names the runner and the
setting. `skram doctor` adds `declines runner jobs` to that runner's line, as
information, not an issue. A job already running there when the owner sets it
runs to the end. Remove the key, or set it `true`, and the machine is a runner
again, with no other step.

### Tailscale SSH

On a tailnet, Tailscale SSH is off until a device's owner turns it on for that
device (`tailscale set --ssh`), so leaving it off is the network-level way to
keep a machine from being a runner. `accept_runner_jobs: false` is Skram's own
switch, for a machine that keeps SSH on for other reasons.

Plan limits, as Tailscale's pricing page stated them in September 2026:

- The free Personal plan covers up to 6 users, for non-commercial use only.
- Tailscale SSH and tags work on every plan; the pricing page lists basic SSH
  on Personal as up to 5 hosts.
- Sharing one device with someone does not add a user; inviting them to the
  tailnet does. Tagged devices do not count as users.
- Machines that belong to a company go on a tailnet of the company's own
  domain, on the Standard plan, priced per user.

## The SSH program

Every call to a runner, git's included, goes through one program:
`$SKRAM_SSH`, else `ssh`, invoked as `<program> <destination> <command>`.
It must log in without a prompt (a key, an agent, or Tailscale SSH) and run
the command in the runner's shell, as `ssh` does. Your `~/.ssh/config`
applies, so a `Host` alias works as a runner name. With `SKRAM_SSH` unset,
Skram runs `ssh -o BatchMode=yes -o ConnectTimeout=10 -o ServerAliveInterval=5
-o ServerAliveCountMax=3 <destination> <command>`: it never prompts, gives up
on a connect after 10 s, and notices a dead connection in about 15 s. A
program you set in `SKRAM_SSH` is run exactly as above, with nothing added.
Point `SKRAM_SSH` at a wrapper to add options, or to test a setup without
a second machine.
