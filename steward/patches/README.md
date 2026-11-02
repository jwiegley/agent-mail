# Patches for the director's scanner and the Discord bridge

Two patches to John's unversioned files, for him to apply, each on its own: the scanner's, below, and the Discord bridge's, in the last section.

`monitor_sessions-steward.patch` changes John's scanner, `~/.local/share/agent-deck/conductor/director/monitor_sessions.py`, in three ways (spec sections 6.9 and 6.12, and John's decision 23 of 2026-11-02):

- **The STATUS rule.** Each pass reads `~/.local/share/agent-deck/conductor/*-steward/STATUS`. When the file is older than twice the `tick=` it declares, and `open=` is above zero or the steward's channel holds a post stamped after `last-post=`, the pass pages Discord through the existing delivery, once per stale episode: `NEED(steward): conductor-<name> STATUS unchanged since <UTC> (tick <m>m, <n> open)`. A closed topic's `STATUS-final-<date>` never pages. The rule reads post file names in `~/Mailbox/+<channel>/` and never opens a message; a post agent-mail never delivers, such as one stamped more than a day ahead, does not count. A steward whose STATUS, TOPIC.md or channel cannot be read goes into the pass's errors, which page once like the scanner's other errors, and the other stewards are still checked.
- **MCP tool calls.** A bullet of the form `⏺ <server> - <tool> (MCP)(...`, or `⏺ <server> - <tool> (MCP)` for a call with no arguments, counts as a tool call, so a peer's question inside an agent-mail `receive` result is no longer paged.
- **Joined lines.** Both captures of a pane, the local one and the one over ssh, pass `-J` to `tmux capture-pane`. It joins the lines that tmux wrapped at the pane's edge, so that a question line that tmux wrapped pages whole, from its id to its question mark, at any pane width (John's decision 23 of 2026-11-02). It also keeps the spaces at the end of each line, which a capture without it drops. They would widen the gaps that the scanner's error rules, `SAFE_PATTERNS`, allow between the words of an error that a line break parts, so the patch takes them out: every check reads what it read before, with a line that tmux wrapped joined. A line that the program in the pane broke itself has no wrapped rows, and `-J` leaves it as it is. Claude Code breaks its own rows: on hera on 2026-11-02, `-J` joined no line in any Claude Code or Codex pane. So a steward's question line still pages whole only while it fits the pane, which the cap of CLAUDE.md section 9, under 76 characters, keeps.

The patch was made against the scanner of 2026-11-02 09:46 (74,199 bytes, sha256 `8d8d7ca67413040384f3b550f0912f8d6cc989721d5af5e9d6985c3d9d42a398`). Run every command below from the root of the agent-mail checkout.

## Apply

Keep a copy of the live scanner, then check, then apply:

```sh
cp -n -p ~/.local/share/agent-deck/conductor/director/monitor_sessions.py ~/.local/share/agent-deck/conductor/director/monitor_sessions.py.pre-steward
patch -N -F0 -p0 --dry-run -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch &&
    patch -N -F0 -p0 -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch
```

`cp -n` never replaces a copy that is already there, so running this block a second time cannot make the patched scanner the copy that Undo restores. A dry run that needs fuzz, finds the patch already applied, or rejects a hunk stops the block with exit 1 before any file changes, and asks nothing: `-F0` allows no fuzz, `-N` refuses a patch that is already in (without it, a second run asks whether to reverse the patch, and macOS's `patch` defaults to yes), and the `&&` runs the real patch only after a clean dry run. GNU patch and macOS's `/usr/bin/patch` behave the same with these flags.

If the dry run says a hunk failed (`Hunk #<n> FAILED at <line>.` from GNU patch, `<n> out of 10 hunks failed while patching monitor_sessions.py` from macOS's), the scanner has changed since the patch was made: regenerate the patch against the new file rather than forcing it. If it reports the patch previously applied (`Reversed (or previously applied) patch detected!  Skipping patch.` from GNU patch, `Ignoring previously applied (or reversed) patch.` from macOS's), the patch is already in.

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
patch -R -N -F0 -p0 --dry-run -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch &&
    patch -R -N -F0 -p0 -d ~/.local/share/agent-deck/conductor/director < steward/patches/monitor_sessions-steward.patch
```

The dry run stops the undo with exit 1, changing nothing, leaving no reject file and asking nothing, in two cases. When the patch is not in, it says so (`Unreversed patch detected!  Skipping patch.` from GNU patch, `Ignoring previously applied (or reversed) patch.` from macOS's), and there is nothing to undo. When the scanner has changed since the patch went in, it says a hunk failed: then put back the copy kept before applying, which also drops any edit made since:

```sh
cp -p ~/.local/share/agent-deck/conductor/director/monitor_sessions.py.pre-steward ~/.local/share/agent-deck/conductor/director/monitor_sessions.py
```

Then restart the scanner as above.

## The Discord bridge

`bridge-steward-routing.patch` changes John's Discord bridge, `~/.local/share/agent-deck/conductor/bridge.py`, so that an answer to a steward reaches it in the two forms that go to the director today (spec sections 6.7 and 8):

- **Any case, and space before the colon.** A message that starts with a conductor's name and a colon goes to that conductor whatever the case of the name: `Ingest-steward : ingest-3 A` reaches conductor-ingest-steward as `ingest-3 A`.
- **A reply to a page.** A reply to the bot's message `NEED(local/conductor-<name>): …`, the page the scanner posts for what it finds in that conductor's pane, goes to conductor `<name>` with the reply's whole text. A conductor prefix at the start of the reply still wins. A reply to one of discord_qna's menu questions is still answered by discord_qna first, and any other message without a prefix, a reply to anything else included, still goes to the first conductor, `director`.

This holds for every page the scanner posts with the place `local/conductor-<name>`, a stall page (`No meaningful pane progress for the policy threshold.`) or an error page (`session status is error`) as much as a question: a reply to it without a prefix of its own is delivered to that conductor as text in its pane. A page with more after the name, such as `local/conductor-<name>/restart`, and the stale-STATUS page `NEED(steward): …` are not among them, so a reply to either goes to the director.

The patch was made against the bridge of 2026-10-31 10:43 (118,714 bytes, sha256 `47e7629d26b186708843ddf1b85cd84336aa55a6df778e8b1e7bd9fe7533d5e0`). `agent-deck conductor setup` overwrites `bridge.py` with the copy built into agent-deck (spec section 5.3), which drops this patch and the Discord Q&A hook alike, so until the fork fix of spec section 8 lands, apply the patch again after any setup run. The copy that the first Apply kept is then still the bridge from before the patch, Q&A hook included, and that is what Undo by copy restores.

### Apply

```sh
cp -n -p ~/.local/share/agent-deck/conductor/bridge.py ~/.local/share/agent-deck/conductor/bridge.py.pre-steward
patch -N -F0 -p0 --dry-run -d ~/.local/share/agent-deck/conductor < steward/patches/bridge-steward-routing.patch &&
    patch -N -F0 -p0 -d ~/.local/share/agent-deck/conductor < steward/patches/bridge-steward-routing.patch
```

The copy, `cp -n`, `-N -F0` and the `&&` work as in the scanner's Apply. If the dry run says a hunk failed, the bridge has changed since the patch was made, as it will have after a setup run: regenerate the patch against the new file rather than forcing it.

### Test

Each block below is one command, so paste one block at a time. With the bridge patch applied, whether or not the scanner's is, run the bridge tests alone (`-k Bridge`); the file still loads the scanner that STEWARD_PATCHED_MONITOR names, patched or not:

```sh
STEWARD_PATCHED_MONITOR=$HOME/.local/share/agent-deck/conductor/director/monitor_sessions.py \
STEWARD_PATCHED_BRIDGE=$HOME/.local/share/agent-deck/conductor/bridge.py \
    ~/.local/share/agent-deck/conductor/venv/bin/python -B -m unittest -k Bridge steward/patches/test_steward_patches.py
```

With both patches applied, run the whole file instead:

```sh
STEWARD_PATCHED_MONITOR=$HOME/.local/share/agent-deck/conductor/director/monitor_sessions.py \
STEWARD_PATCHED_BRIDGE=$HOME/.local/share/agent-deck/conductor/bridge.py \
    ~/.local/share/agent-deck/conductor/venv/bin/python -B -m unittest steward/patches/test_steward_patches.py
```

Then run John's own bridge tests, which are in `test_discord_qna.py` and load the `bridge.py` beside the director's directory:

```sh
(cd ~/.local/share/agent-deck/conductor/director && ~/.local/share/agent-deck/conductor/venv/bin/python -B -m unittest test_discord_qna.BridgeTests)
```

The steward tests import the bridge with its home, config and conductor directory in an empty temporary directory, so the import opens no file of John's, and they build its Discord bot without logging in, with agent-deck and Discord's API replaced by recorders. John's tests import the bridge as it stands. Before the patch and after it alike, they print discord.py's DeprecationWarning that `asyncio.iscoroutinefunction` is deprecated and six `Discord bot initialized (guild=1, channel=42)` lines, and add those six lines to the live `bridge.log`: neither is a patch failure.

### Restart the bridge

The running bridge keeps the old code until its launchd job restarts it. Restart it while no Discord message is waiting for a conductor's reply, since the process holds that wait and a reply still pending when it stops is never posted:

```sh
launchctl kickstart -k gui/$(id -u)/com.agentdeck.conductor-bridge
```

### Check

```sh
tail -n 40 ~/.local/share/agent-deck/conductor/bridge.log | grep 'Discord bot ready'
```

A `Discord bot ready` line stamped after the restart means the bridge is back.

Then, with conductor-ingest-steward running (spec section 6.4), send three messages from Discord: `ingest-steward: digest`, then `Ingest-steward : digest`, then `digest` as a reply to one of the steward's pages. If the steward has posted no page yet, make the reply check on its first one. Each works when the steward's digest comes back tagged `[ingest-steward]` and `bridge.log` records it:

```sh
grep -E 'Discord message -> \[ingest-steward\]|Failed to resolve Discord reply target' ~/.local/share/agent-deck/conductor/bridge.log | tail -n 6
```

Each of the three should show a line ending `Discord message -> [ingest-steward]: digest`. An answer tagged `[director]` instead means the bridge did not take that message for the steward's; for the reply, a `Failed to resolve Discord reply target` line says the page could not be read. Until it is resolved, answer the steward with new messages that start `ingest-steward:`, which reach it either way, and undo the patch or report what you found.

### Undo

```sh
patch -R -N -F0 -p0 --dry-run -d ~/.local/share/agent-deck/conductor < steward/patches/bridge-steward-routing.patch &&
    patch -R -N -F0 -p0 -d ~/.local/share/agent-deck/conductor < steward/patches/bridge-steward-routing.patch
```

The dry run stops the undo as in the scanner's Undo: when the patch is not in, there is nothing to undo; when it says a hunk failed, the bridge has changed since the patch went in, and you put back the copy kept before applying, which also drops any edit made since:

```sh
cp -p ~/.local/share/agent-deck/conductor/bridge.py.pre-steward ~/.local/share/agent-deck/conductor/bridge.py
```

Then restart the bridge as above.
