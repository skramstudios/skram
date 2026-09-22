# Skram documentation

One job queue you and your coding agents share: one job at a time per shared
resource, every run logged.

These pages are written for someone who has only the binary. Start at the
[README](../README.md) for what Skram is, how to install it, and the first
loop; come back here for the detail. Every page is published from the same
release as the binary, so it describes the version you are running:

```bash
skram version                   # which release this is
skram doctor                    # what this machine still needs
skram discover                  # every project, target, and typical duration
```

## Pages

- [Configuration reference](config.md) — every key of the config file, with its
  default and what changes when you set it: the project blocks and their
  backends, resources, ephemeral targets, guard allow lists, workflows, the
  reaper's budgets, and the repo entries that say what is true inside a
  checkout.
- [How it works](how-it-works.md) — what a job is on disk, the queue file and the lock in
  front of it, the processor that starts on demand and exits when idle, how
  an item that names more than one resource is scheduled, a hold against a
  drain against a kill, what an ephemeral target is and what `--now` really
  does, what `--on-error stop` cancels and what it leaves alone, and how an
  estimate falls back from a flag combination to the target to the project.
- [Agent setup](agent-setup.md) — registering Skram's MCP server for an agent, what
  `skram agent install-rules` writes into a checkout and why none of it is
  committed, the guard hook and which command lines it stops, how the actor
  on each item is detected and overridden, and the exit codes an agent should
  branch on.
- [MCP](mcp.md) — the tools `skram mcp` serves, the JSON each one takes and
  returns, and the resources it exposes.
- [herdr](herdr.md) — the sidebar row the processor keeps, the plugin actions, and
  the key bindings for them.
- [Notifications](notifications.md) — the one-time macOS setup that makes job-end
  notifications appear, and how to turn them off.
- [Troubleshooting](troubleshooting.md) — an item that never starts (is the queue held, and who
  held it), a processor that died and where its log is, a quarantined binary,
  silent notifications, how to read what `skram doctor` prints, and every
  `SKRAM_*` variable in one table.
- [Questions](faq.md) — why this rather than a single-terminal job queue or a lock
  file, whether it works alongside worktrees, whether the source is available,
  and what it sends anywhere (nothing).

## Reporting a problem

Open an issue on the public repository with the output of `skram version` and
`skram doctor`, your operating system, and what you expected instead.
