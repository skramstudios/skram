# Setting up the agent surface

Skram's agent surface is four things, and each is installed once by hand:

1. the **MCP server**, registered at your editor's user scope, so every
   project gets the tools without a file in any checkout;
2. the **machine-local agent files** `skram agent install-rules` writes into
   each checkout, so an agent reading that checkout's instructions learns the
   queue exists and what its targets are;
3. the **guard**, a pre-tool shell hook that stops a command which would
   bypass the queue and replies with the `skram run` line to use instead;
4. the **actor**, which needs no setup on Claude Code or Cursor and one
   environment variable everywhere else.

There is no one-command plugin that installs all of this yet. Everything below
is a command or a snippet you can paste.

`skram doctor` reports on all four and is the way to check your work:

```bash
skram doctor
```

## 1. Register the MCP server

For Claude Code, once per machine:

```bash
claude mcp add --scope user skram -- skram mcp
claude mcp list          # skram should be listed
```

Any other client that launches stdio servers from a JSON file takes the same
definition. For Claude Code that file is `~/.claude.json`; other clients use
their own:

```json
{
  "mcpServers": {
    "skram": { "command": "skram", "args": ["mcp"] }
  }
}
```

The client must be able to find `skram` on its `PATH`; give the absolute path
(`/home/you/.local/bin/skram`) if it cannot. To point the server at a config
file other than `~/.config/skram/config.yaml`, put `--config <file>` before
`mcp` in the arguments.

`skram doctor` reads `~/.claude.json` and reports the registration as missing:

```text
[--] mcp: skram is not registered at user scope (run: claude mcp add --scope user skram -- skram mcp)
```

It checks that a server named `skram` is registered, not what that server
runs, so a registration pointing somewhere else still reads as present.

Restart the editor session after registering: a client reads its server list
at startup.

What the server then offers — six tools, two resources, the instructions it
sends at connect time — is [the MCP page](mcp.md).

## 2. Write the agent files into a checkout

```bash
skram agent install-rules              # every checkout the config knows about
skram agent install-rules --here       # only the checkout you are in
skram agent install-rules --check      # write nothing; exit 1 if anything is missing or edited
```

`--projects a,b` narrows to the checkouts where those projects apply, and
`--repos x,y` to the checkouts of those `repos:` entries. (The other tools the
same command runs, [below](#one-command-for-skram-skram-vault-skram-tunnel-and-skram-chat),
take `--repos` too, but never `--projects`.) Every checkout
carries its linked worktrees along automatically — a `git worktree add`
sibling, one nested inside the checkout, or any other linked worktree gets
the same two files, with no extra config; narrowing applies to them the same
way. A worktree whose directory has been removed but not pruned from git is
skipped without a line.

```text
== skram ==
  /home/you/dev/apps
    OK   AGENTS.local.md
    OK   CLAUDE.local.md
    OK   .git/info/exclude

Installed 3 artifact(s) (engine v0.13.0).

skram-vault: not on PATH, skipped

skram-tunnel: not on PATH, skipped

skram-chat: not on PATH, skipped
```

A checkout where no project applies gets no Skram section: it is listed with a
`SKIP` line, and that is not a failure. A file that cannot be written or removed
gets an `ERROR` line and the command exits 1, however many others succeeded.

Three files, and **none of them is committed**:

| File | What it is |
| --- | --- |
| `AGENTS.local.md` | a `## Shared-resource jobs (skram)` section: the queue rules, then a `### Targets` table for each project that applies in this checkout |
| `CLAUDE.local.md` | one line, `@AGENTS.local.md` — Claude Code loads `CLAUDE.local.md` by itself |
| `.git/info/exclude` | the two filenames above, under a `# skram machine-local agent files` comment |

`.git/info/exclude` is the checkout's own private ignore file: it is not
tracked and never reaches a commit. `.gitignore` is never touched. So this
works in a repo you have no right to change, and `git status` stays clean. A
linked worktree gets its own `AGENTS.local.md`/`CLAUDE.local.md`, and its
lines land once in the exclude file it shares with the main checkout, not
once per worktree.

The section is a function of your config alone — no timings, nothing
machine-specific — so `--check` compares bytes and gives the same answer on
every machine:

```text
== skram ==
Agent rules current (engine v0.13.0): 3 artifact(s) checked.
```

An edited or out-of-date file makes it print the difference and exit 1, and
`skram doctor` reports the same thing as one line per checkout. Re-run
`install-rules` to fix it.

### One command for Skram, Skram Vault, Skram Tunnel, and Skram Chat

[Skram Vault](https://github.com/skramstudios/skram-vault),
[Skram Tunnel](https://github.com/skramstudios/skram-tunnel), and
[Skram Chat](https://github.com/skramstudios/skram-chat) keep their own sections in the same files. After writing or checking its own,
`skram agent install-rules` runs `skram-vault agent install-rules`, then
`skram-tunnel agent install-rules`, then `skram-chat agent install-rules`, so
one command after any upgrade refreshes all four.
Each tool still writes and checks only its own section and lines.

- A tool is found on Skram's own `PATH`. One that is not there gets a one-line
  note and counts as success.
- It runs in your current directory with your environment, and what it prints
  appears under a `== skram-vault ==`, `== skram-tunnel ==`, or `== skram-chat ==`
  heading.
- It gets `--repos`, `--here`, and `--check` as you gave them, and `--config`
  (made absolute) when you gave Skram one. `--projects` narrows Skram only: the
  other tools run without it.
- The command exits non-zero when any of the four does; under `--check`, that
  is a difference reported by any of them. One tool failing does not stop the
  others.
- When Skram cannot make sense of what you asked for, nothing runs and nothing
  is written: a config it cannot read, an unknown project or repo name,
  `--here` outside git or in a checkout Skram does not know, or a selection
  that matches no checkout.

A tool's exit code alone decides whether it passed or failed, so each tool has
to exit 0 where it has nothing of its own to write. With a release of any
of them that exits non-zero there instead, this command fails too; upgrade that
tool. With `--check --json`, Skram also puts what each tool printed into its
document.

With `--check --json`, standard output is one document: Skram's report, in
which `ok` covers all four tools, and a `siblings` list. Standard error is
passed through. Here Skram Vault reports a difference and Skram Tunnel and
`skram-chat` are not installed:

```json
{
  "ok": false,
  "engine": "v0.13.0",
  "checked": 3,
  "differences": [],
  "siblings": [
    {
      "tool": "skram-vault",
      "found": true,
      "exit_code": 1,
      "report": { "ok": false }
    },
    { "tool": "skram-tunnel", "found": false, "exit_code": 0 },
    { "tool": "skram-chat", "found": false, "exit_code": 0 }
  ]
}
```

| Key in `siblings` | What it holds |
| --- | --- |
| `tool` | `skram-vault`, `skram-tunnel`, or `skram-chat` |
| `found` | whether it is on Skram's `PATH` |
| `exit_code` | its exit code; `-1` when it could not start or a signal ended it |
| `report` | the JSON object it printed, as it printed it (the one above is cut short) |
| `error` | without a report, what it said instead; absent when it succeeded and said nothing |

`--json` without `--check` changes nothing: a write prints the same text.

**If the checkout has an `AGENTS.md` and no `CLAUDE.md`:** Claude Code's
documentation says a `CLAUDE.local.md` makes it stop reading `AGENTS.md`
there. Set Claude Code's *Project instructions* setting to
`claude-md-and-agents-md` to keep both.

## 3. Install the guard

The guard is `skram agent guard`: it reads a hook payload on stdin, decides
whether the command bypasses the queue, and answers in the hook protocol of
whichever client sent it.

### Claude Code

Add it as a global `PreToolUse` hook on `Bash` in `~/.claude/settings.json`.
This is the whole of what `skram doctor` looks for:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "skram agent guard" }]
      }
    ]
  }
}
```

If the file already exists, add the entry to the existing `PreToolUse` list
rather than replacing the file. Until it is there, `doctor` says:

```text
[--] guard: no PreToolUse Bash hook running "skram agent guard" in ~/.claude/settings.json
```

The line may carry a trailing note about where that hook usually comes from.
The snippet above is all the check wants.

A blocked command exits 2 with the reason on stderr, which is how Claude Code
learns the tool call was refused and why.

### Cursor

Cursor sends `beforeShellExecution` with the command at the top level of the
payload, and expects a verdict on stdout. The guard detects that shape and
answers it, with no configuration of its own:

```json
{ "permission": "deny", "user_message": "skram: …", "agent_message": "skram: …" }
```

or `{"permission":"allow"}`. Register `skram agent guard` as Cursor's
`beforeShellExecution` hook command. `skram doctor` does not check Cursor's
files.

### What it blocks

Given a project whose script is `ops.sh` with a read-only `status` target:

| The command | What happens |
| --- | --- |
| `./ops.sh build` | blocked — a configured project's script, run outside the queue |
| `./ops.sh status` | allowed — `status` is in the project's `ephemeral:` list |
| `make build`, `task build`, `just build` | blocked when a project in this checkout uses that tool *and* defines that target |
| `kubectl apply -f deploy.yaml` | blocked — a cluster mutation, anywhere |
| `kubectl get pods` | allowed — reads are not mutations |
| `docker build -t app .` | blocked — an image build, anywhere |
| `docker compose up`, `docker compose down` | blocked |

The reason it prints is the command to run instead:

```text
skram: /home/you/dev/apps/ops.sh runs outside the queue. Use: skram run apps build --json,
then skram wait <item_id>. Ephemeral (read-only) targets are exempt; see skram discover.
```

The guard parses the command as a shell would and judges every command in
it: each part of a `;`, `&&`, `||`, or `|` list, and what runs inside
`( … )`, `$( … )`, and backticks. Wrappers (`sudo`, `env`, `bash`, `time`,
…) and leading `NAME=value` assignments are stripped first, so
`sudo docker build .` is still a `docker build`.

- **Quoted text and heredocs are data.** `echo "make deploy"`, a commit
  message, or `cat <<EOF` naming a script is not judged. Text a shell runs
  is: the string after `bash -c`/`sh -c`/`zsh -c` and `eval`, and a heredoc
  fed to a shell given no script (`bash <<EOF`).
- **A syntax check runs nothing.** `bash -n`, `sh -n`, and `zsh -n` (also
  bundled, as in `-nx`) are allowed.
- **`cd` moves where the rest is judged.** `cd ../apps && ./ops.sh build` is
  judged in `../apps`. After a `cd` the guard cannot follow (`cd $DIR`,
  `cd -`, a directory that does not exist), only full paths are matched.
- **A script's full or relative path is guarded from any directory; a bare
  name only inside its own checkout** (or a worktree of it). Another repo's
  own `scripts/ops.sh` is not yours. A word holding an expansion (`$OPS`) is
  skipped, and globs are not expanded.
- A command the guard cannot parse is allowed, with one line on stderr.

The `make`/`task`/`just` rule and the allow list only apply to the projects
that belong where the command runs — the checkout's own, worked out from the
`repos:` block and the project paths. The `kubectl` and `docker` rules apply
from any directory.

### Letting something through

A target that is genuinely not shared-resource work goes in the project's
[`guard.allow`](config.md), a list of
[RE2](https://github.com/google/re2/wiki/Syntax) patterns matched against the
command:

```yaml
projects:
  apps:
    guard:
      allow:
        - '^kubectl delete pod debug-'
```

A pattern matches if it matches either the one part of the command being
judged or the whole line. Allow lists are read from the projects that belong
where the command runs.

To turn the guard off for one command, one shell, or one session, set
`SKRAM_GUARD` to `0`, `false`, or `off`:

```bash
SKRAM_GUARD=0 ./ops.sh build
```

**The guard fails open.** Unreadable stdin, a payload it cannot parse, a
config it cannot load — each is reported on stderr and the command is allowed.
It is a guard rail, not a permission system: it exists to stop an agent from
walking past the queue by accident, and it never stops you working.

### Checking it without an editor

`skram agent guard` is an ordinary command that reads a payload on stdin, so
you can try it from a plain shell. The command under test sits inside a
quoted payload, so the guard treats it as data even from inside a guarded
agent session:

```bash
echo '{"tool_input":{"command":"docker build -t app ."},"cwd":"'"$PWD"'"}' \
  | skram agent guard; echo "exit $?"
```

Exit 2 with a reason on stderr is a block; exit 0 and silence is an allow.

## 4. The actor

Every queue item and every job's `status.json` records who enqueued it, so
`skram status` can say who is waiting and `skram explain` can say who held the
queue. It is detected from the environment, first match winning:

| Checked | Actor |
| --- | --- |
| `SKRAM_ACTOR` | its value |
| `CLAUDECODE` or `CLAUDE_CODE` set | `claude-code` |
| `CURSOR_TRACE_ID` or `CURSOR_AGENT` set | `cursor` |
| `USER` | its value |
| none of the above | `unknown` |

So Claude Code and Cursor name themselves with no setup. For any other agent,
set `SKRAM_ACTOR` in the environment you launch it in:

```bash
SKRAM_ACTOR=review-bot skram run apps test --json
```

It flows through everything: the item, the job's `status.json`, `by_actor` in
`skram status`, `cancelled_by` on a cancelled item, and the `actor` on a hold.
The queue's processor exports the enqueuing actor into each job, so a job's
own status names whoever asked for it, not the processor.

## The rules an agent reads on demand

The rules `install-rules` writes are short, at most 60 lines: the loop, the
guard, what `held`, `still_running`, `skipped` and `in_flight` mean, and what to
do after a failure. The rest is read only when it is needed, one topic at a
time: `ledger`, `reasons`, `runners`, `files`, and `config`.

```bash
skram rules              # the topics, one line each
skram rules reasons      # one topic, as markdown
```

An unknown topic exits 1 and names the topics. The MCP server serves the same
text as `skram://rules/<topic>`. When a job fails because the machine was unfit
(`disk_full`, `docker_down`, `missing_tool`, `missing_secret`, `setup_failed`,
`start_failed`), `explain`'s `next` tells the agent not to enqueue it again and
to report the machine and `reason_detail` to you.

## Picking up where the last session stopped

A new session needs no hand-off if the queue's record is read first. Two calls
answer "what is deployed?" and "how did the last green run?":

```bash
skram status --json            # most_recent: the last job, with each checkout's and input's head
skram explain q_6aa510b2 --json   # any past item: env, checkouts, rerun
```

`most_recent.inputs.<VAR>.head` is the commit that input was on at enqueue
(`head_at_start` when the job began), and `checkout` names the project's own
when it ran outside its configured path. Compare it with where the repo is now
(`git diff --stat <head> HEAD`); when nothing that the target reads has
changed, skip the rebuild.

`explain` works on a green job as well as a red one. Its `env` lists the
variables the job was given with `--env`, `checkout` and `inputs` the directory
each repo resolved to (with its head), and `rerun` is the whole line, `--env`
and `-C` flags included, so sending it again reproduces the job instead of
approximating it. Run that line, then
`skram wait <item> --json --timeout auto`. The estimate that comes back from
`run --json` says what it rests on, for example "~11m38s (from 3 runs ·
exact)": the number of runs, and whether the history matched the exact line, only
its target, or only the project.

## Exit codes

`skram wait` and `skram run -f` both block and then exit with the outcome, so
an agent can branch on the code without parsing anything:

| Code | Meaning |
| --- | --- |
| 0 | the job completed |
| the target's own | it ran and failed; the code is the target's |
| 124 | the timeout elapsed — the job is still running |
| 125 | the item was cancelled before it ran; `cancelled_by` says by whom |
| 137 | the job was killed |

124 is not a failure. Call `wait` again, or pass a longer `--timeout`.

```bash
skram wait q_6a2071c8 --json --timeout 30m
skram wait --last --timeout auto
skram wait --queue
```

`--timeout auto` is twice the estimate for whatever is being waited on, never
less than ten minutes; the deadline it chose goes to stderr and, under
`--json`, into `timeout_seconds`. [How it works](how-it-works.md) has the same
table with its one caveat, and [troubleshooting](troubleshooting.md) covers an
item that never starts.

The guard's own exit codes are different and belong to the hook protocol, not
to a job: 2 means blocked, 0 means allowed.

## When it is all in place

```bash
skram doctor                            # no [!!] lines
skram agent install-rules --check       # exits 0
skram discover --json                   # the projects an agent will see
```

Then start a session and ask the agent to run something. It should reach for
the `run` tool, report an item id, and call `wait`.
