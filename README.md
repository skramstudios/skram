# Skram

**One job queue you and your coding agents share: one job at a time per shared resource, every run logged.**

You have one local cluster, one Docker daemon, one port 3000, and three
checkouts, a couple of agent sessions, and yourself all wanting to build,
deploy, and test against it. A worktree per agent isolates the files and
nothing else: two agents that both start the app take the same port, and you
can only realistically run the app in one place. Skram puts a lane in front of
each shared resource, so work queues instead of colliding, every run is kept,
and `status` says who asked for it.

```text
$ skram run apps deploy
Queued apps/deploy (item q_6aa510b2, 1 pending)
Estimate: ~3s (from 7 runs · exact); queue clears in ~3s (1 pending)
$ skram run web deploy
Queued web/deploy (item q_2c31477f, 1 pending)
Estimate: ~3s (from 5 runs · exact); queue clears in ~6s (1 pending)
$ skram status
Running:
  apps/deploy  J20260921_212950  (PID 45006, 0s elapsed)  ~3s remaining (est ~3s · from 7 runs · target)  by you
    Lane: kind-dev
By actor: you (sam): 1 running · claude-code: 1 pending
ETA: queue clears in ~6s (1 pending)
```

Two projects, two callers, one cluster: the second run waited its turn.

![Two callers enqueue the same target, a hold with a reason, and status naming who asked](docs/demo.gif)

## What holds its own

- **A lane per shared resource.** `resource:` on a project names what its
  targets must not run against at the same time. Every project naming it
  shares that lane and takes turns; lanes with nothing in common run at once.
  The resource is what gets serialised, not the repository or the agent.
- **The actor on every item.** Whoever enqueued an item is recorded on it and
  carried into the job's own record, so `status`, `queue list`, and `explain`
  say `by you` or `by claude-code` instead of leaving you to work out whose
  failure this is.
- **A hold with a reason, which agents see.** `skram hold --reason "upgrading
  the cluster"` pauses the queue for everyone until `skram release`. Nothing
  bypasses it, and a caller who enqueues during one is told why:

  ```text
  Queued apps/deploy (item q_51203633, 2 pending)
  HELD by you: upgrading the cluster — starts after 'skram release'
  ```

  With `--json` the same enqueue reports `"held": true` with the hold's actor
  and reason, so an agent can stop and say so rather than block.
- **`wait`.** `skram wait <item>` blocks until the job is done and exits with
  the job's own code, so an agent never polls. `--queue` waits for everything,
  `--resource` for one lane, `--timeout auto` takes its deadline from the
  estimate.
- **`explain`.** After a failure: the status, the classified reason, the tail
  of what the job printed, and the commands to run next.

`--json` everywhere and a hook that blocks the raw build command are table
stakes: pueue had `--json` and `wait` years ago, and the editors' own hook
documentation gives the same block-and-suggest recipe. Skram has both; the
list above is what it adds.

## Install

Releases are built for macOS and Linux, on amd64 and arm64.

```bash
curl -fsSL https://raw.githubusercontent.com/skramstudios/skram/main/install.sh | sh
skram version
```

The script takes the latest release for your machine, checks it against the
release's `checksums.txt`, and puts `skram` in `~/.local/bin`, which must be on
your `PATH`; `VERSION=` picks a release and `BIN_DIR=` another directory. By
hand, unpack the archive from the
[releases page](https://github.com/skramstudios/skram/releases); shell
completions are in its `completions/`, and `skram completion zsh` prints one on
demand. macOS quarantines a binary downloaded in a browser, so clear the flag
once:

```bash
xattr -d com.apple.quarantine ~/.local/bin/skram
```

## Quickstart

**1. Check the machine.** `skram doctor` is the first thing to run, and the
thing to run after every config edit: the config file, the data directories,
each project with the targets it exposes, and anything missing.
[Reading its output](docs/troubleshooting.md) is the way out of most problems.

**2. Give a project an entry point.** A Makefile, Taskfile, or justfile works
as it stands; a shell script with one column-0 `case` arm per target is the
lowest common denominator and wraps anything else. Skram parses the file
rather than running it, so discovery works without `make` on the machine:

```bash
case "$1" in
  status)   # Show cluster status — this comment is the help text
    kubectl get pods -A
    ;;
esac
```

**3. Register it.** `skram init --detect`, run in the checkout, finds that
file and writes the entry below once you confirm:

```yaml
projects:
  apps:
    path: /home/you/dev/apps
    script: ops.sh           # or makefile: Makefile, taskfile: Taskfile.yml, justfile: justfile
    resource: kind-dev       # optional; projects sharing a resource take turns
    ephemeral: [status]      # add by hand: read-only targets run at once, no queue, no history
```

`--backend` picks a different file, `--name` renames the project, `--yes`
skips the prompt, and `--dry-run` and `--json` never write. Every key, and
where each backend's targets and help text come from, is in the
[configuration reference](docs/config.md).

**4. Use it.**

```bash
skram discover                               # every project, target, help, and typical duration
skram run apps deploy                        # enqueue; prints the estimate
skram run deploy                             # inside the checkout the project name is optional
skram run apps test --scope=auth             # everything after the target goes to the script
skram status                                 # running job, time remaining, queue ETA
skram logs -F                                # follow the running job
skram run apps status                        # ephemeral: runs now, no queue, no job directory
skram run apps test -f                       # attach: block, stream, exit with its code
skram kill                                   # stop the running job (its whole process group)
```

Outside any checkout, `skram context use apps` sets the project a bare
`skram run <target>` means; `skram context list` and `skram context current`
show it.

## The agent loop

An agent never drives the ops script. The loop is enqueue, wait, read:

```bash
skram run apps test --json                   # → item_id, estimate, "wait": "skram wait q_…"
skram wait q_6aa510b2 --json --timeout auto  # blocks; exit code is the job's
skram explain q_6aa510b2 --json              # after a failure: reason, stderr tail, what to run next
skram logs q_6aa510b2 --json --tail 40       # events.jsonl, one JSON object per line
skram status --json                          # what is running, who is waiting, when it clears
```

`wait` and `run -f` exit with the job's own code, or `124` on a timeout, `125`
for an item cancelled before it ran, and `137` for one that was killed.
`skram wait --queue` blocks until nothing is pending or running, and
`skram queue cancel <item>` withdraws a pending item.

`skram mcp` serves the same commands over MCP with the same JSON, and
`skram agent install-rules` writes the rules into each checkout as a
machine-local `AGENTS.local.md` that is never committed, so it works in a
repository you cannot change. [Agent setup](docs/agent-setup.md) is the whole
path — server at user scope, files, guard, exit codes — and
[MCP](docs/mcp.md) is the tool-by-tool reference.

## Sharing a resource with your agents

One session per checkout, an agent in each, all deploying to one cluster:

- Give every project that touches the cluster the same `resource:`; one that
  touches two lists both. `skram wait --resource kind-dev` and
  `skram kill --resource kind-dev` then address that lane alone.
- Agents enqueue and `wait`; you enqueue with `-f` when you want to watch.
- Only a target you listed as `ephemeral:` skips the queue, which is your
  promise that it is read-only.
- `--on-error stop` cancels the pending items in the failed job's lanes — every
  project sharing the resource, and no other lane.

## Everything else, once each

| | | |
| --- | --- | --- |
| `skram agent guard` | a Claude Code and Cursor shell hook: it blocks the ops script, a known `make`/`task`/`just` target, and `kubectl` and `docker` mutations, and prints the `skram run` line instead | [installing it](docs/agent-setup.md) |
| `skram tui` | running jobs and their logs, a launcher, and the queue, in the terminal | [as a popup](docs/herdr.md) |
| `skram workflow list`, `skram workflow run` | a DAG of targets, defined under `workflows:`, queued as one job that holds every lane its steps need | [the keys](docs/config.md) |
| `skram estimate` | what a command usually takes, and from which runs | [key fallback](docs/how-it-works.md) |
| `skram prune` | remove old job directories; running jobs and anything the queue still refers to are kept | [defaults](docs/troubleshooting.md) |
| `skram reap` | reclaim Docker disk space so repeated image builds cannot fill the daemon's disk | [budgets](docs/config.md) |
| `skram dashboard` | a monitoring UI with live log streaming; `skram dashboard status` and `skram dashboard stop` manage it | [details](docs/how-it-works.md) |
| `skram metrics up` | a Prometheus, Grafana, Loki, and Vector stack from a compose file you supply | [what it needs](docs/how-it-works.md) |
| herdr | the processor keeps the running job in each workspace's sidebar row, and the TUI opens as a popup | [bindings](docs/herdr.md) |
| notifications | on job end, through herdr in a session or macOS otherwise | [setup](docs/notifications.md) |

[Skram Vault](https://github.com/skramstudios/skram-vault) keeps lore, specs,
and tickets in a git repository you own;
[Skram Tunnel](https://github.com/skramstudios/skram-tunnel) shares a local app
and the identity provider in front of it on one public URL. Each installs on
its own and reads its own blocks of the same config file.

## What it is not

Skram is not a CI system, not a durable workflow engine, and not a
per-project task runner. It does not do parallel groups, remote execution,
or input-hash caching (yet, or ever, respectively).

If you only need a shell job queue for one terminal, use
[pueue](https://github.com/Nukesor/pueue). Skram is for when several
callers, some of them not human, share one thing.

## Similar projects

- **[pueue](https://github.com/Nukesor/pueue)** — the closest thing: groups,
  `pause`, `wait`, `--json`. No record of who enqueued an item and no lane per
  shared resource, because it is built for one person's terminal.
- **Worktree and agent managers** — Claude Code's own worktrees, claude-squad,
  [Orca](https://github.com/stablyai/orca), Conductor. They give each agent its
  own files, and say plainly that ports and the network are not isolated. Use
  both: a worktree for the files, a Skram lane for the cluster.
- **[Tilt](https://tilt.dev), [process-compose](https://github.com/F1bonacc1/process-compose)**
  — one developer's inner loop: start the services, watch them, rebuild on
  save. They run things; they do not queue callers.
- **make, [Task](https://taskfile.dev), [just](https://github.com/casey/just),
  [mise](https://mise.jdx.dev)** — what Skram dispatches *to*: it parses their
  targets and runs them, and does not replace them.

## Documentation

[docs/README.md](docs/README.md) is the index: configuration,
[how it works](docs/how-it-works.md) (the rule for an item with two resources,
hold against drain against kill, what `--now` really does, where estimates come
from), agent setup, MCP, herdr, notifications, troubleshooting, and questions.

## License

Apache-2.0. See `LICENSE` and `NOTICE`; `THIRD_PARTY_NOTICES.md` carries the
licences of the Go modules built into the binary.
