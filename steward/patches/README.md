# Patches for the director's scanner

`monitor_sessions-steward.patch` changes John's scanner, `~/.local/share/agent-deck/conductor/director/monitor_sessions.py`, in two ways (spec sections 6.9 and 6.12):

- **The STATUS rule.** Each pass reads `~/.local/share/agent-deck/conductor/*-steward/STATUS`. When the file is older than twice the `tick=` it declares, and `open=` is above zero or the steward's channel holds a post stamped after `last-post=`, the pass pages Discord through the existing delivery, once per stale episode: `NEED(steward): conductor-<name> STATUS unchanged since <UTC> (tick <m>m, <n> open)`. A closed topic's `STATUS-final-<date>` never pages. The rule reads post file names in `~/Mailbox/+<channel>/` and never opens a message; a post agent-mail never delivers, such as one stamped more than a day ahead, does not count. A steward whose STATUS, TOPIC.md or channel cannot be read goes into the pass's errors, which page once like the scanner's other errors, and the other stewards are still checked.
- **MCP tool calls.** A bullet of the form `⏺ <server> - <tool> (MCP)(...`, or `⏺ <server> - <tool> (MCP)` for a call with no arguments, counts as a tool call, so a peer's question inside an agent-mail `receive` result is no longer paged.

The patch was made against the scanner of 2026-10-31 11:46 (73,768 bytes, sha256 `c636237a6d83ed310ff5693751a54f05578e3b1b09c771b93c505bd30eb3047b`). Run every command below from the root of the agent-mail checkout.

## Apply

Keep a copy of the live scanner, then check, then apply:

```sh
cp -p ~/.local/share/agent-deck/conductor/director/monitor_sessions.py ~/.local/share/agent-deck/conductor/director/monitor_sessions.py.pre-steward
patch -p0 --dry-run -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch
patch -p0 -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch
```

If the dry run reports a rejected hunk, the scanner has changed since the patch was made: regenerate the patch against the new file rather than forcing it.

The running scanner keeps the old code until the restart below.

## Test

The steward tests, against the patched scanner. They replace Discord, agent-deck and the scanner's state and log with fixtures, so they page nobody and leave `state.json`, `task-log.md` and `~/Mailbox` alone:

```sh
STEWARD_PATCHED_MONITOR=$HOME/.local/share/agent-deck/conductor/director/monitor_sessions.py \
    ~/.local/share/agent-deck/conductor/venv/bin/python -B -m unittest steward/patches/test_steward_patches.py
```

John's own scanner tests, in the director's directory:

```sh
(cd ~/.local/share/agent-deck/conductor/director && ~/.local/share/agent-deck/conductor/venv/bin/python -B -m unittest test_monitor_sessions)
```

Run the steward tests again after any change to `monitor_sessions.py`: they hold the fixtures spec 6.12 relies on (the question-line template, a flake input in a question line, an MCP receive block and the STATUS rule), and John's own suite does not run them. If a dry run of the patch then rejects a hunk, regenerate the patch against the changed file.

## Restart the scanner

The scanner is one long-running process (`heartbeat.sh` runs it with `--loop` under the launchd job `com.agentdeck.conductor-heartbeat.director`), so a patched file takes effect only when that process restarts. This waits until no pass holds `.monitor.lock`, then restarts the job while the scanner sleeps between passes:

```sh
(cd ~/.local/share/agent-deck/conductor/director && ~/.local/share/agent-deck/conductor/venv/bin/python -I -c 'import fcntl, os, subprocess; lock = open(".monitor.lock", "a"); fcntl.flock(lock, fcntl.LOCK_EX); subprocess.run(["launchctl", "kickstart", "-k", f"gui/{os.getuid()}/com.agentdeck.conductor-heartbeat.director"], check=True)')
```

The new process's first pass may log `Monitor already running; skipping overlapping pass.` while the command above still holds the lock; the next pass runs a minute later.

## Check the next pass

```sh
tail -n 3000 ~/.local/share/agent-deck/conductor/director/heartbeat.log | grep -E '^Traceback|"steward_alerts"|"steward_status"|"steward/|"completed_at"' | tail -n 4
```

A normal pass ends with a `"steward_alerts"` line (`[]` while every steward's STATUS is fresh) followed by a `"completed_at"` from the last minute or two. A `Traceback` line, or a `"steward_status"` entry in the pass's errors, means the rule failed: undo the patch as below. A `"steward/conductor-<name>"` entry names a steward whose STATUS, TOPIC.md or channel could not be read: the fix is to that steward's file, not to the patch.

## Undo

```sh
patch -R -p0 -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch
```

`patch -R` assumes the scanner has not changed since the patch went in. If it has, or the reverse is rejected, put back the copy kept before applying, which also drops any edit made since:

```sh
cp -p ~/.local/share/agent-deck/conductor/director/monitor_sessions.py.pre-steward ~/.local/share/agent-deck/conductor/director/monitor_sessions.py
```

Then restart the scanner as above.
