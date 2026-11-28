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
this machine copies its own binary there. List the tools under the runner's
[`tools:`](config.md#runners) and `skram doctor` and `skram runner check`
check they are there.

The job gets its `PATH` from your login shell on the runner (`$SHELL -l`),
so what `~/.zprofile` or `~/.bash_profile` adds there, such as Docker
Desktop's `~/.docker/bin`, reaches the job the way it reaches a local one.
`~/.zshrc` is read only by interactive shells, so a tool put on `PATH` there
alone is missing: check with `ssh <runner> "$SHELL -lc 'command -v docker'"`,
or pass `PATH` for one job with `--env PATH=…`, which wins. The rest of the
environment is the SSH command's.

## Setting up a machine as a runner

Each requirement below has one command to run by hand from this machine, the
one with the queue, and the fix when it fails. `<dest>` is the runner's SSH
destination. For the login-shell `PATH`, where most first jobs fail, see the
paragraph above.

| Requirement | Check, run here | Fix |
|---|---|---|
| SSH logs in with no prompt of any kind | `ssh -o BatchMode=yes -o ConnectTimeout=10 <dest> true; echo $?` prints `0` | On the runner turn Remote Login on (macOS: System Settings, General, Sharing), put this machine's public key in its `~/.ssh/authorized_keys`, and log in once by hand to accept the host key. Skram never lets SSH prompt, so a password, passphrase, or host-key question fails the call. |
| Same OS and CPU architecture | `uname -sm; ssh <dest> uname -sm` prints two identical lines | None: use another machine. An Intel runner cannot serve an Apple Silicon queue. |
| Skram on the runner | none needed | Nothing to install or configure: the first job copies this machine's binary. |
| It accepts jobs | `ssh <dest> 'grep accept_runner_jobs ~/.config/skram/config.yaml'` prints nothing or `true` | Remove `accept_runner_jobs: false` from the runner's own config ([Declining runner jobs](#declining-runner-jobs)). |
| `git` and `git-receive-pack` on the plain SSH `PATH` | `ssh <dest> 'command -v git git-receive-pack'` prints two paths | Install git (on macOS, the Command Line Tools: `xcode-select --install`, at the machine). The push and the checkout that precede every job run under the bare SSH `PATH`, not the login shell's. |
| The login-shell `PATH` is what the jobs need | `ssh <dest> '$SHELL -lc "echo \$PATH"'` | Set it in `~/.zprofile` or `~/.bash_profile`, not `~/.zshrc`, or pass `--env PATH=…` on each job. |
| Each tool the target runs is on that `PATH` | `ssh <dest> '$SHELL -lc "command -v node pnpm"'` (name your tools) prints a path for each | The same fix. Name the tools under [`tools:`](config.md#runners) so `doctor` and `runner check` look for them every time. |
| On a Mac, the runner may sleep between jobs | none needed | Nothing: each job holds it awake ([Keeping machines awake](#keeping-machines-awake)). |

A job that uses Docker Desktop on a Mac needs a logged-in desktop session on
the runner, since Docker Desktop runs only inside one: log in once after each
reboot. While this machine sleeps, nothing new is dispatched and a streaming
connection drops; on a Mac it holds itself awake while any of its jobs runs,
here or on a runner.

### Keeping machines awake

On macOS, every job holds the machine that runs it awake for exactly its
length: Skram starts `caffeinate -i -m -s -w <pid>` bound to the job's
process, so idle and system sleep wait until the job ends by exit, kill, or
failure. For a job sent to a runner, the runner's Skram holds the runner
awake, and the processor on this machine holds this one awake while it streams
the job. A pending item, or one waiting for a runner or a lease, holds
nothing. The display may still sleep, and closing a laptop's lid still sleeps
it. Nothing needs `sudo` and no `pmset` setting changes. A Mac without
`caffeinate`, and any other operating system, runs the job without it.

`skram runner check` runs all of it, from here, in one command, and prints
the fix under each row that fails (see [Checking a runner](#checking-a-runner)):

```bash
skram runner check studio
```

`skram runner add` declares the runner (see [Declaring a
runner](#declaring-a-runner)) and ends by running that check:

```bash
skram runner add studio --ssh me@studio.example.net --tools node,pnpm
```

## Declaring a runner

```yaml
runners:
  build-box: {}                  # reached as `ssh build-box`
  studio:
    ssh: me@studio.example.net   # any destination your SSH program accepts
    tools: [node, pnpm, docker]  # commands its jobs need on its login-shell PATH
```

A runner's name is its SSH destination unless `ssh:` says otherwise, and
`ssh:` is one destination (a host, an alias, or `user@host`), never a
command. A name starts with a lowercase letter or digit and continues with
lowercase letters, digits, `.`, `_`, and `-`. `local` is this machine and
cannot be declared. `tools:` is optional; what it does is in
[the config reference](config.md#runners).

`skram runner add` writes the entry for you:

```bash
skram runner add studio --ssh me@studio.example.net --tools node,pnpm
skram runner add build-box --yes                                # no ssh: (the name is the destination), no question
skram runner add studio --ssh me@studio.example.net --dry-run   # print the lines, write nothing
```

It prints the lines it would add under `runners:` (with the `runners:` key
when the config has none yet), then asks `Add to <config>? [Y/n]` in a
terminal. Anywhere else it writes only with `--yes`; without one it prints the
lines, writes nothing, and exits 1 with "Nothing written" on stderr, so a
script does not mistake the proposal for a write. Answering `n` at the prompt
exits 0. `--dry-run` writes nothing and exits 0, even with `--yes`, and so
does `--json`, which prints `name`, `destination`, `tools`, `config`, `lines`
(the exact text), and `written` (always false). The rest of the file, comments
and key order included, is left as it was. It refuses a name already declared,
`local`, a name outside the rule above, an `--ssh` with spaces in it, and a
`--tools` entry that is not a command name, and writes nothing. After a write
it runs `skram runner check` on the new runner (see [Checking a
runner](#checking-a-runner)) and exits with its code, so a runner that cannot
be reached yet is still declared and the command exits 1.

```bash
skram runner list          # one line per runner: SSH destination, tools, projects that default to it
skram runner list --json   # the same, keyed by runner name
```

`skram runner list` reads the config and this machine's record of each
runner's last answer, and contacts no runner:

```text
RUNNER     SSH                    TOOLS               DEFAULT FOR  LAST PROBE             FREE DISK  DOCKER
build-box  build-box              -                   -            unreachable, 2h5m ago  -          -
mini       mini                   -                   -            never                  -          -
studio     me@studio.example.net  node, pnpm, docker  api          ok, 3m ago             120GB      OrbStack (context orbstack), up, 31GB free
```

`LAST PROBE` is the outcome of the last hello (the first call Skram makes to a
runner before a job, and in `runner check` and `doctor`) this machine made to
the runner (`ok`, `unreachable`, `declined`, or `never` when there has been
none) and how long ago it was, so a machine that went offline shows as
`unreachable` rather than looking fine. `FREE DISK` is the free space that
hello measured on the volume holding Skram's data there. `DOCKER` is the
Docker engine the runner's `docker` context reached (`Docker Desktop`,
`OrbStack` or `Docker Engine`), the context, `up` or `down`, and the free
space inside its storage; Skram measures the engine only in a hello for a
project whose [`tools:`](config.md#tools) names `docker`. A later hello that
did not ask keeps what the last one found and adds how old it is (`as of 3h
ago`). `-` is a value no hello has measured (an unreachable runner, an older
Skram there, or Docker never asked for), never zero. To refresh the columns,
run `skram runner check <name>` (see [Checking a runner](#checking-a-runner)).

With `--json` it prints one object keyed by runner name, each with
`destination`, `tools`, and `projects` (the projects whose `runner:` is that
runner), plus the same fields as `skram status --json`'s `runners` (`outcome`,
`at`, `last_contact_seconds`, `host_free_bytes`, `docker_answered`,
`docker_engine`, `docker_context`, `docker_error`, `docker_free_bytes` and the
rest; see [`status`](mcp.md#status)); `tools` and `projects` are always lists,
empty when there are none, a field not measured is absent, and the object is
`{}` when no runner is declared. With none declared, the plain output says so
and names `skram runner add <name> --ssh <dest>`, and `skram doctor` prints
`[--] runners: none declared` as information. `skram discover --json` carries
the same names under `runners`, each with `destination` and `tools`.

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
an empty `--runner=`) runs one here. In the Launch tab, `ctrl+r` cycles the
runner (`local` and every declared runner, the project's default chosen first)
before `Enter`; the footer shows the chosen runner and its lane, and an
ephemeral target offers no choice. An ephemeral target always runs here. A
name that is not in `runners:` is a config error. `skram discover`
(`runner` in `--json`) and the project's MCP resource
`skram://projects/<name>` show the default. A workflow's steps run here.

### Seeing where a job runs

A job sent to a runner carries `@<runner>` after its label wherever a label is
shown: `skram status`, `skram queue list`, `skram logs`, the TUI's Jobs and
Queue tabs, the dashboard, and the herdr sidebar (`build (api--T-3) @build-box
1m20s/4m10s`). The item, its `status.json`, and the JSON of `run`, `status`,
`queue list`, `wait`, and `explain` carry `runner`. A job's estimate prefers
the history of the machine it runs on: that runner's own past runs when it has
any, else every run (see [Estimates](how-it-works.md#estimates)), and `wait
--timeout auto` follows it.

A job sent to a runner records its phase on its item as it goes: `checking`
(the runner's hello, and sending it this machine's skram when it differs),
`shipping` (pushing a checkout's commit), `preparing` (resetting the runner's
worktree to it; with inputs, each repo is shipped then prepared in turn),
`setup` (the project's [`setup:`](config.md#setup) target, when it has one),
`running` (the target), and `homing` (bringing the job's files home). `skram
status` prints it on the running job's line, and `status --json` and `queue
list --json` carry it as `phase` on the running job and item. A job that
stays `shipping` is pushing a large commit, not hung. A local job is
`running`; a pending item, one waiting for its runner or a lease, and a
finished job have no phase.

The TUI shows both: a running row in Jobs carries its phase (a local job's plain
`running` is left out), and the Queue tab's `state` column says why a pending
item waits, in the same words as `skram status`.

Lanes name the runner too: `skram status`'s Lanes block and each running
job's `Lane:` line, `cluster@studio` in `queue list` after a live item's label
(`lanes` on every item in `--json`), `lanes` in `run --json`, `explain
--json`, and each running job in `status --json` (their `resources` stay the
resource names), and the TUI's Queue tab. A lane that only repeats the
label, a project with no `resource:` sent to `studio`, is not printed.
`skram wait --resource cluster@studio` and `skram kill --resource
cluster@studio` address that lane alone; `--resource cluster` is this
machine's.

## Splitting a suite across runners

To shorten a suite, send one job per destination: one here and one to each
runner. Each runs in its own lane, so they run at the same time.

```bash
skram run api test --suite a --json                       # this machine, lane cluster
skram run --runner studio api test --suite b --json       # lane cluster@studio
skram run --runner build-box api test --suite c --json    # lane cluster@build-box
skram wait <item-a> --json --timeout auto                 # one wait per item
skram wait --queue                                        # or wait for every lane
```

- Arguments after the target pass through as they are, so the split (a suite
  name, a shard, a filter) is whatever the target already accepts.
- `--runner` goes before or after the target. A project's `runner:` default
  avoids the flag, and `--runner local` overrides it for one job.
- Each runner needs its own copy of whatever the suite talks to (a database,
  containers, a cluster, ports). Bring it up with a job sent to that runner,
  never by hand.
- Two jobs sent to one runner on one `resource:` take turns, and a job's
  estimate is the same wherever it ran.

## What is not true about runners

Each of these is a wrong assumption that sends a reader looking for something
that is not there.

| If you assume | What is true |
|---|---|
| A runner has a Skram command to run, or something keeps running there | The `skram runner` commands run here. Nothing runs on a runner between jobs, apart from a [reap you ask for](#reaping-a-runners-docker). See [Declaring a runner](#declaring-a-runner). |
| The runner needs Skram installed and configured | It needs neither: this machine sends its binary and a config for each job, and a `skram` already on the runner is never used for a sent job. See [What goes to the runner](#what-goes-to-the-runner). |
| Declaring runners makes the queue spread jobs | Nothing picks a runner. A job goes where `--runner` or its project's `runner:` says, else it runs here. To use three machines, enqueue three jobs: see [Splitting a suite across runners](#splitting-a-suite-across-runners). |
| You run `skram`, or the tests, on the runner | Enqueue, `wait`, and read logs here only. Nothing needs a login on the runner after setup. |
| The runner uses its own checkout of the project | Its own checkouts are never read. The job runs in a worktree Skram keeps, on the exact commit sent. See [Where things live on a runner](#where-things-live-on-a-runner). |
| Installs, `.env` files, and build output come along | Files git ignores never leave this machine. The job installs what it needs; pass values with `--env`, and secrets by name (see [Secrets on a runner](#secrets-on-a-runner)). |
| An ephemeral target can go to a runner | It always runs here, and `--runner` on one is refused. See [Sending a job](#sending-a-job). |
| A workflow's steps can go to runners | A workflow's steps run here. |
| Any machine can be a runner | It must have this machine's OS and CPU architecture, or the job fails as `runner_refused`. |

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
3. Each git submodule initialised in the checkout goes the same way, nested
   ones too: see [Submodules](#submodules).
4. The runner checks that commit out into a worktree it keeps for this
   checkout: tracked files are reset to it and untracked files left by an
   earlier job are removed, but files git ignores stay for the next job, so
   caches persist between jobs of one checkout. Two jobs that would use one
   worktree on one runner (two projects in one checkout, or a job reading
   as an input the checkout another runs in) never run at once: the second
   stays pending until the first ends.
5. The job runs there detached, so it survives the SSH session, with a config
   Skram sends for that job. It never enters the runner's own queue. A
   project with a [`setup:`](config.md#setup) target runs it first in that
   worktree, as its own step; when it fails, the job ends as `setup_failed`
   and the target never starts.
6. While it runs, its output streams back line by line into this machine's
   job directory, so `skram logs -F`, `skram logs --json`, and `skram run -f`
   show it as it happens, and `skram status` shows the job running on its
   runner.
7. When it ends, its `raw.log`, `events.jsonl`, and `status.json` are copied
   into this machine's job directory, so `logs`, `explain`, history, and
   estimates read it like any other job. So is its `artifacts.tar.gz`: the
   runner packs the project's
   [`artifacts:`](config.md#artifacts-and-artifacts_limit) from its worktree
   as the job ends, even after a kill, so a trace the next job there would
   wipe is read here, never on the runner.

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
uncommitted changes on top of `head`, in the checkout or in one of its
submodules, and when it is `false`, `commit` is `head`. A checkout with
submodules adds `submodules`, below.

### Secrets on a runner

A job sent to a runner receives its [`secrets:`](config.md#secrets) and
`--secret` names like a local one, and the runner keeps nothing. This machine
runs `skram-vault` before anything is sent, so a secret that does not resolve
fails the job here as `missing_secret`, with no call to the runner and the
detail naming this machine. The values then travel inside the SSH session
that starts the job and live only in the memory of the job's process and the
target's environment: no file on the runner holds one, not the job's config,
its `status.json`, its logs, or what comes home, where each appears as
`‹secret:NAME›`. The runner never runs `skram-vault`, needs no vault, and has
nothing to clean up when the job ends. The job's own environment never shows
`SKRAM_SECRET_NAMES`, the list of names Skram sets to tell the job's process
that its values arrived this way, and removes at once.

A runner whose Skram is too old to take secrets this way is refused before
anything is sent, as `runner_refused`; the detail says to upgrade Skram there.
A job whose copy of Skram is replaced first (see
[What goes to the runner](#what-goes-to-the-runner)) never meets this.

### Submodules

A submodule initialised in the checkout travels as the checkout does, from
this machine over the same SSH connection: its commit (the one the
checkout records for it), or a commit on it holding its uncommitted changes,
is pushed into the runner's own clone of that submodule, and the runner sets
the submodule in the checkout's worktree to it. A submodule inside a
submodule goes the same way. The runner never fetches: it needs no network
access and no credentials for a submodule's remote. Files git ignores in a
submodule stay here, as in the checkout.

Shallow clones ship. A submodule cloned at `--depth 1` (or `shallow = true`
in `.gitmodules`) needs no `git fetch --unshallow` first, and neither does a
shallow checkout of the project or of an input: the runner's clone takes the
history your clone has. A job there sees no more history than that. The
runner's clone also remembers where the history stopped, so a later job from a
full clone of the same repo runs as usual, but `git log` or `git describe` in
its runner worktree may stop at the same place. To have the full history sent
again, remove both `repos/<repo>.git` and `worktrees/<repo>` there (see
[Where things live on a runner](#where-things-live-on-a-runner)) while none of
its jobs runs.

The runner's worktree mirrors your checkout. A submodule with uncommitted
changes makes the job's `sent` `dirty`, so a [ledgered](config.md#ledger)
target records nothing. A submodule you have not initialised here
(`git submodule update --init --recursive`) is absent there too, an empty
directory, and the job's events hold one warning tagged `missing_submodule`
naming each one; the job runs anyway, since the same build would fail here.

`sent` lists each submodule shipped, by its path from the checkout's root:

```json
"sent": {"head": "3f9c…", "commit": "3f9c…", "dirty": true, "submodules": [
  {"path": "deps/common", "head": "8b21…", "commit": "c07d…", "dirty": true},
  {"path": "deps/common/inner", "head": "51e0…", "commit": "51e0…", "dirty": false}]}
```

### Where things live on a runner

Everything a runner holds is under `$XDG_DATA_HOME/skram/runner` there
(`~/.local/share/skram/runner` by default), so removing that directory removes
all of it:

| Path | Holds |
|---|---|
| `bin/skram` | the binary this machine sent |
| `repos/<repo>.git` | one bare clone per repo, named by its `repos:` entry, else its folder; every checkout of a repo shares it, and a submodule gets one too |
| `worktrees/<repo>/<checkout>` | one worktree per checkout, named by the checkout's folder, where its ignored caches live; each submodule in it is a worktree of the submodule's clone; removed once idle (see [Runner storage](#runner-storage)) |
| `logs/` | the runner's copy of each job's `raw.log`, `events.jsonl`, `status.json`, and `artifacts.tar.gz` |
| `runs/<job>` | the config sent for one job and the detached process's output |

The leases are beside it, in `$XDG_DATA_HOME/skram/leases`: one file per
resource, shared by the runner's own queue and the jobs sent to it. On the
machine that sends the jobs, `$XDG_DATA_HOME/skram/runners/<name>.json` is
what it last heard from each runner (see [Checking a runner](#checking-a-runner)).

Two checkouts of one repo whose folders share a name would share a worktree, so
a job that reads both is refused; rename one.

### Runner storage

Each job sent to a runner first removes, on that runner, what has been idle
longer than the runner's [`prune_after:`](config.md#runners) (default `72h`;
`0` turns this off):

- a worktree whose last job ended, and which no job has been prepared in,
  longer ago than that;
- the `logs/<job>` and `runs/<job>` of a job whose files came home longer ago
  than that.

A worktree a running job uses, a job that is running, and a job whose files
never came home are kept, and so are the clones under `repos/`, so the next
job for a removed checkout name gets a fresh worktree from the clone. What was
removed (a count and the bytes) is one line in this machine's
`queue-processor.log`; the job's own events do not mention it. Nothing there
fails the job.

A removed worktree loses its ignored files: an install your setup target
rebuilds comes back on the next job, slower once; anything you copied into the
worktree by hand is gone after it sat idle for `prune_after`.

### Inputs

A project with [inputs](config.md#inputs) sends every checkout it reads. Each
input's checkout, the one the job resolved here (`-C`, your working directory,
or the repo's default), goes to the runner like the project's own: one commit
into the runner's clone of that repo, checked out into a worktree kept for
that checkout, with its [submodules](#submodules). On the runner each input
variable names that worktree, and `SKRAM_INPUT_<VAR>_DEFAULT` is set only when
the input is on its default checkout (to the same worktree), since the default
checkout itself does not travel. Each input in the `inputs` of the item and
`status.json` records its own `sent`:

```json
"inputs": {"LIB_DIR": {"repo": "lib", "dir": "/src/lib--T-1", "sent": {"head": "…", "commit": "…", "dirty": true}}}
```

The config the runner receives holds the job's project, with its path and
every input's pointing at the runner's worktrees, and only the `repos:`
entries the job reads, plus this machine's `reaper:` section; no other project
or repo leaves this machine. The target sees the same `env:` values, `--env`
values, `SKRAM_ACTOR`, and herdr workspace and session as it would here. An
input checkout removed before the job starts fails it as `checkout_removed`,
and the runner is never contacted.

## Reaping a runner's Docker

Image builds fill a runner's Docker disk as they fill this machine's. The
[`reaper:` section](config.md#reaper) of this machine's config goes with every
sent job, so a job whose project and target match `after_targets:` and that
succeeds reaps on the runner afterwards, inside the job: the reap's lines are
in the job's log before it ends, and it never fails the job.
`SKRAM_REAP_DISABLE=1` here skips that reap there too.

To reap a runner now, name it:

```bash
skram reap --runner mini --dry-run    # what would be removed there
skram reap --runner mini              # remove it
```

The reap runs on the runner with this machine's `reaper:` section and the
command's own flags (`--dry-run`, `--keep`, `--cache-budget`, `--images-only`,
`--cache-only`); its output streams back and the exit code is the reap's. The
runner needs no Skram config, and nothing is queued, so it can run while a job
builds there: an image that job is using survives, because its removal fails
and is skipped. A name not under `runners:` is refused with the declared ones;
`--runner local` is the plain reap on this machine. A runner whose owner
[declines runner jobs](#declining-runner-jobs) declines this too.

`image_repos:` globs match the repository name as `docker image ls` prints it
on the machine being reaped, and `*` never crosses a `/`. Images a runner
builds for itself (`skram-e2e-app`, say) have no registry prefix, so they need
their own glob (`skram-e2e-*`) beside `localhost:6001/*`.

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
waiting for the lease, and Skram asks the runner again about once a minute. It
is sent as soon as the lease is free. If the runner stops answering while the
item waits, it waits for the runner instead (`waiting_for_runner`, as in [When
the runner is off](#when-the-runner-is-off)), and once the runner answers it
is sent, or waits for the lease again. On the runner, an item of its own queue
whose resource is leased to a job sent there waits the same way and starts by
itself when that job ends.

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
machine last booted is taken over by the next job that wants the resource, so
a dead job never holds one for ever. Each takeover is a `lease_reclaimed` line
in `$XDG_DATA_HOME/skram/leases/events.jsonl`, and the queue processor's log
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
say), the job fails with the last error rather than waiting again: as
`runner_refused` when the runner rejected the `git push`, else as
`runner_unreachable`.

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
| `runner_refused` | The runner cannot take the job as things stand: another OS or architecture, a runner no longer in `runners:`, a checkout with nothing to send, a copied binary that will not run there, a job with secrets for a runner whose Skram cannot take them, or a `git push` the runner rejected (`reason_detail` has git's message). Nothing ran. |
| `runner_unreachable` | The runner answered, but a call there kept failing before the job started. Nothing ran. |
| `runner_lost` | The job started, but contact was lost and every reattach failed. It may still be running there. |
| `runner_declined` | The runner's owner set `accept_runner_jobs: false` there (see [Declining runner jobs](#declining-runner-jobs)). Nothing ran, and nothing was retried. |
| `setup_failed` | The project's [`setup:`](config.md#setup) target failed in the runner worktree, so the target never started. The tail shows the setup's output. |
| `missing_tool` | A command the project's [`tools:`](config.md#tools) names is not on the runner's login-shell `PATH`. Nothing was sent. |
| `disk_full` | The runner is short of disk: its host disk, or for a project that uses Docker, the engine's storage. Nothing was pushed. |
| `docker_down` | The project uses Docker and the engine the runner's `docker` context reaches did not answer. Nothing was pushed. |

The last three come from the checks Skram makes in its first call to the
runner, before anything is pushed, and `reason_detail` names the runner and
what it found.

- **`missing_tool`**: put the tool on the runner's login-shell `PATH`
  (`~/.zprofile` or `~/.bash_profile`), confirm with `skram runner check
  <name>`, then run the job again.
- **`disk_full`**: the floor is this machine's
  [`preflight.min_free_disk`](config.md#preflight) (default 5GB) for the
  volume holding Skram's data on the runner, or
  [`preflight.docker_min_free`](config.md#the-docker-engine) (default 10GB)
  inside the Docker engine. With the [`reaper:`](config.md#reaper) block on,
  Skram has already [reaped there once](config.md#reaping-once-before-failing)
  and `reason_detail` ends with what that freed. Free space on the runner
  (`skram reap --runner <name>`, see [Reaping a runner's
  Docker](#reaping-a-runners-docker)), or run the job here.
- **`docker_down`**: for Docker Desktop and OrbStack, Skram has already
  [started the engine there once](config.md#starting-a-stopped-engine), and
  `reason_detail` names the command and how it ended. Restart the engine on
  the runner, or run the job here. A job that had started and then printed
  that the engine stopped answering gets the same reason, with that line as
  `reason_detail`.

A runner that cannot be reached at all is not a failure: see
[When the runner is off](#when-the-runner-is-off).

## Checking a runner

`skram doctor` asks each runner what it is (giving up after 15 s) and prints
one line per runner:

```text
[ok] runner build-box: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0); data ~/.local/share/skram/runner; last contact 3m ago
[--] runner studio (me@studio.example.net): reachable, darwin/arm64, no skram yet (the first job sends this machine's v0.19.0 (engine 0.19.0)); data ~/.local/share/skram/runner; last contact never
[!!] runner old-box: unreachable through ssh: runner call hello exited 255: ssh: connect to host old-box port 22: Connection refused; last contact 2h5m ago (unreachable)
[--] runner lab-mac: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0), declines runner jobs (accept_runner_jobs: false there); data ~/.local/share/skram/runner; last contact 1d3h ago (declined)
[!!] runner mini: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0); data ~/.local/share/skram/runner; login PATH /usr/bin:/bin:/usr/sbin:/sbin; missing on login PATH: node, pnpm; disk: 120GB free (preflight.min_free_disk 5GB); last contact just now
[ok] runner pc: reachable, darwin/arm64, skram v0.19.0 (engine 0.19.0); data ~/.local/share/skram/runner; login PATH /opt/homebrew/bin:/usr/bin:/bin; disk: 120GB free (preflight.min_free_disk 5GB); docker: OrbStack (context orbstack), up, 31GB free; last contact 1m ago
```

A reachable runner's line gives the `PATH` a job there gets from the login
shell. Every line ends with the last contact before this one: how long ago
this machine last said hello to the runner (`never` when it has not), with the
outcome when that hello did not get an ordinary answer. A runner that is
unreachable, or of another OS or architecture, is an issue and makes `doctor`
exit 1, and so is one missing a tool its `tools:` names from that `PATH`, or
missing `git` or `git-receive-pack` from the plain SSH `PATH` every job's push
runs under. One with no Skram or another build is information only: the next
job sends this machine's. A runner's free disk follows, against this machine's
[`preflight.min_free_disk`](config.md#preflight); below it is an issue. When a
project that defaults to the runner, or the runner's own `tools:`, names
`docker`, the hello also asks for the Docker engine there, and the line names
it and its context: an engine that does not answer, or one below
[`preflight.docker_min_free`](config.md#the-docker-engine), is an issue. A
value the runner's Skram did not measure is left out. A runner still on an
older Skram build is probed through plain `sh` instead, so its `PATH`, `git`
and tools are checked too; "not probed" appears only when that probe could not
run at all. `skram runner check` is the row-by-row form of that line, with the
fix under each failed row.

The queue keeps each runner's last answer in
`$XDG_DATA_HOME/skram/runners/<name>.json` on this machine, written after
every hello it makes: before a dispatch, while an item waits for the runner,
and in `runner check` and `doctor`. `skram status --json` carries those files
as `runners`, each with `last_contact_seconds`, without contacting anyone (the
fields are under [`status`](mcp.md#status)). The hello before a job of a
project whose `tools:` names `docker` also measures the runner's Docker
engine, so the file records which engine its `docker` context reached, whether
it answered, and its free space; a runner on an older Skram reports none of
that, which fails nothing.

```bash
skram runner check                            # every declared runner, in name order
skram runner check studio --tools node,pnpm   # look for these tools instead of its tools:
skram runner check studio --json              # the same rows as data
```

It asks each runner once (a second call asks a macOS runner's `pmset` about
sleep) and prints one row per item: SSH, OS and architecture, the Skram build
there, whether it accepts jobs, `git` on the plain SSH `PATH`, the login-shell
`PATH`, each tool, free disk, the Docker engine when the tools it looks for
name `docker`, and sleep. `check` sends nothing and pushes nothing, so it
is safe to run at any time.

```text
runner studio (me@studio.example.net)
[ok] SSH connects without a prompt
[ok] darwin/arm64, same as this machine
[--] no skram yet (the first job sends this machine's v0.21.0 (engine 0.21.0))
[ok] accepts jobs
[ok] git and git-receive-pack on the SSH PATH
[ok] login PATH /opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin
[ok] node on the login PATH
[!!] pnpm missing from the login PATH
     fix: add its directory to PATH in the runner's login profile (~/.zprofile or ~/.bash_profile; ~/.zshrc is not read), or pass --env PATH=… on each job
[ok] disk: 120GB free (preflight.min_free_disk 5GB)
[ok] docker: OrbStack (context orbstack), up, 31GB free
[--] sleep allowed (pmset sleep 10): jobs hold the machine awake while they run
```

`[!!]` is a problem a job would hit, and makes `check` exit 1; `[--]` is
information. When SSH fails, that row and its fix are all a runner gets, since
nothing else can be asked. With no name it checks every declared runner. The
tools it looks for are the runner's own `tools:` plus the
[`tools:`](config.md#tools) of every project whose default runner it is, each
once; `--tools a,b` replaces that list for this one call. The disk row is the
free space on the volume holding Skram's data there, against this machine's
[`preflight.min_free_disk`](config.md#preflight): below it, the row fails, as
a job sent there would. The Docker row names the engine the runner's `docker`
context reached (on a machine with both Docker Desktop and OrbStack, the one
its current context picks) and fails when the engine does not answer (`docker:
down (context desktop-linux): <its error>`) or has less free space inside than
[`preflight.docker_min_free`](config.md#the-docker-engine). A runner whose
Skram is too old to measure either gets a `[--] … not measured` row, which
fails nothing; the next job sends it this machine's. `--json` prints an object
keyed by runner name, each a list of rows in the order above: `name` (`ssh`,
`platform`, `build`, `accepts`, `git`, `login_path`, `tool:<name>`, `disk`,
`docker`, `sleep`), `ok` (false only for `[!!]`), `warn` (true for `[--]`),
`detail`, and `fix` (absent when there is none).

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
-o ServerAliveCountMax=3 -o ControlMaster=auto -o ControlPath=<dir>/%C
-o ControlPersist=60s <destination> <command>`: it never prompts, gives up
on a connect after 10 s, notices a dead connection in about 15 s, and shares
one connection per runner, so the calls of a job after the first skip the
login (about 0.06 s instead of 0.2 s each). `<dir>` is `ssh/` in Skram's data
directory, or `/tmp/skram-<uid>` when that path is too long for a socket
(macOS allows 104 bytes); if neither is a private directory of yours, no
sharing options are added. A
program you set in `SKRAM_SSH` is run exactly as above, with nothing added.
Point `SKRAM_SSH` at a wrapper to add options, or to test a setup without
a second machine.
