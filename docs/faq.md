# Questions

Short answers to what people ask before installing. For what to do when
something is wrong, see [troubleshooting](troubleshooting.md).

## Why not just pueue?

If one person at one terminal wants to queue shell commands, use
[pueue](https://github.com/Nukesor/pueue). As a general job queue it has more
in it than skram does: groups with their own parallelism, dependencies
between tasks, delayed start, pause and resume, callbacks, per-task logs, and
a daemon you can drive over the network. Skram has none of that.

Two things skram treats as the point, which pueue does not set out to do:

- **A lane belongs to a shared resource, not to a terminal.** Projects in
  different repositories that name the same `resource:` share one line, so
  one cluster or one Docker daemon gets one job at a time no matter which
  repo, shell, or agent asked. pueue's groups are a way to limit parallelism
  inside one daemon's list of commands; they are not a name for the thing
  outside that must not be used twice at once.
- **Every item carries who asked for it, and a pause carries why.** `status`
  says an item is `by claude-code` and that the queue is `HELD by you:
  debugging the cluster by hand`; an agent enqueueing while that hold stands
  is told so, with the reason, in the JSON it gets back. pueue can pause, but
  a pause has no author and no reason attached to it.

There is also a smaller difference in what gets queued: skram queues a
*target a project declares* — read out of the repo's own ops script,
`Makefile`, `Taskfile.yml`, or `justfile` — rather than an arbitrary command
string, which is why an agent can ask what exists before it asks for it.

What is **not** a reason to choose skram: `--json` output, blocking until a
job finishes, and per-job logs. pueue has had all three for years, and so do
several tools in this space. A hook that stops an agent from running a
dangerous command is table stakes too — the recipe is in every agent's own
hook documentation. Skram's version of that hook knows the project's real
targets and names the `skram run` line to use instead, which is a nicety, not
a category.

## Why not a lock file?

A lock file gives you mutual exclusion and nothing else. What it does not
give you is everything that makes a shared resource bearable:

- **Order.** Two callers blocked on a lock race for it when it opens. A lane
  is first in, first out, and `skram queue list` shows the order before it
  happens.
- **An answer to "who has it".** A lock file says taken. `skram status` says
  which target, whose, how long it has been going, and roughly how long is
  left.
- **A way to wait that a program can use.** `skram wait <item> --json` blocks
  and exits with the job's own code — an agent can wait without polling and
  branch on the result.
- **A record.** Every run leaves its output, its events, and its exit code on
  disk, so "whose failure was that" is answerable afterwards.
- **A safe crash.** A stale lock file blocks everyone until someone notices.
  A lane whose processor died is taken up by the next one; see
  [troubleshooting](troubleshooting.md).

Under the covers skram *is* a file and a lock: the queue is a JSON file
guarded by `flock`. The lock is the mechanism. The queue, the actor, the
estimate, the log, and `wait` are the part you were going to have to write.

## Does it work with worktrees, and with agent managers like Orca?

Yes, and they are worth using together — they solve different halves of the
same complaint.

Worktrees, and the tools built on them (an agent manager such as
[Orca](https://github.com/stablyai/orca), a session multiplexer, your
editor's own worktree support), isolate **files**. Each agent gets its own
checkout and its own branch, and they stop editing each other's work.

They do not isolate what is outside the repository. The cluster, the Docker
daemon, the database, the port your dev server wants — those are one thing,
and a second worktree does not make a second one. The symptom people report
is always the same shape: two agents both took port 3000; you can only
realistically run the app in one place. Orca has gone further than most at
making that visible, with a panel of live ports and a per-worktree proxy, but
its approach is to show and multiplex rather than to serialise. Skram is the
other half: a lane per shared resource, so the second caller waits instead of
colliding.

Practically, inside any worktree:

- `skram run build` (or `skram run apps build`) from inside a worktree of the
  project's repo runs in that worktree: its own Makefile or script, its own
  code, still through the project's lane. From the configured checkout, or
  from a directory that is not a checkout of the repo, it runs in the
  project's configured `path:`.
- `-C <dir>` names the checkout explicitly and always wins, so
  `skram run apps build -C ~/dev/apps` from a worktree runs in the configured
  checkout. See [running in another checkout](how-it-works.md#running-in-another-checkout--c).
- `skram discover` in a worktree reads the worktree's own file, so a target
  only that branch has is listed, and it says which checkout it read.
- Omitting the project name works when the directory you are in belongs to
  exactly one project; a subdirectory of a worktree resolves to the same
  project as that subdirectory of the configured checkout.

## Is the source available?

No. Releases publish the binaries for macOS and Linux, the install script,
the changelog, the licence and notices, and these pages. The source
repository is not published.

The binary is licensed under Apache-2.0: `LICENSE` and `NOTICE` ship in every
release, and [THIRD_PARTY_NOTICES.md](../THIRD_PARTY_NOTICES.md) carries the
licences of the Go modules built into it.

## Does it send anything anywhere?

No. There is no telemetry, no analytics, no crash reporting, and no update
check. Nothing in the binary makes an outbound request on its own — not at
startup, not when a job ends, not ever. Your config file, your queue, and
your job logs stay on the machine.

Three parts of skram involve a network socket at all, and each of them only
when you ask for it:

- **`skram dashboard`** with no `--local` opens a tunnel through
  [ngrok](https://ngrok.com), using *your* ngrok authtoken — from
  `NGROK_AUTHTOKEN` or your own ngrok config file — and prints a public URL
  for it. That is a real outbound connection to a third party, and it is the
  only one in the product. `skram dashboard --local` opens no tunnel and
  binds to this machine.
- **Checking the monitoring stack** makes one request to
  `http://localhost:2112/health` on this machine, to see whether the
  Prometheus exporter is answering. It goes to loopback and nowhere else.
- **The Prometheus exporter**, once you start it, listens on a local address
  for a scraper to come to it. It sends nothing itself.

Two more things worth naming because they look like network calls and are
not. The MCP server (`skram mcp`) talks to the agent that started it over
that process's own standard input and output — there is no socket and no
port. Notifications and herdr sidebar rows are local programs skram runs
(`terminal-notifier`, `osascript`, the herdr executable), handed the project,
target, and status of a job that just finished.

The install script is the one thing that downloads: it fetches the release
archive and its checksums from GitHub, verifies the checksum, and unpacks the
binary. That is the installer, not the product.

## What happens to a running job if the processor dies?

It keeps running and finishes normally. A job runs in its own process group,
not as a child that dies with the background processor, and it writes its own
exit code and log.

What is lost is the bookkeeping: the item's queue row keeps saying `running`,
and nothing takes the next item until a new processor starts, which the next
`skram run` or `skram release` does. The stale rows are cleared by `skram
queue reset`. [Troubleshooting](troubleshooting.md) has the full sequence.

Skram does nothing special about the machine sleeping. Processes suspend and
resume with it, and elapsed time is counted on the clock, so a job that
spanned a sleep reads as overdue against its estimate and records a duration
that includes the time asleep. That run then makes future estimates for the
same command worse until newer runs outweigh it.

## Can two machines share one queue?

No. One machine, one queue. The queue is a file on local disk guarded by
`flock`, and whether the processor is alive is decided by signalling a local
process id — neither survives being pointed at a shared filesystem, where
locking is not dependable and a process id from another host means nothing.

Two machines that share a cluster therefore do not take turns through skram
today. Each machine serialises its own callers against it.

## Where do I report a problem?

Open an issue on the public repository with the output of `skram version` and
`skram doctor`, your operating system, and what you expected instead.
