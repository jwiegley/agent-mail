# What exists today: agent-deck's director and conventions, and where a per-topic conductor sits

Explorer perspective: the director. Written 2026-10-31, about 10:20 PDT, from read-only inspection on hera.
Every factual claim carries its source. "Verified" means I read the file or ran the read-only command today.
"Unverified" means documented but not exercised. "I did not check" marks what I did not look at.

Abbreviations: `D/` is `/Users/johnw/.local/share/agent-deck/conductor/director/`; `C/` is
`/Users/johnw/.local/share/agent-deck/conductor/`; `handoff` is
`/Users/johnw/Products/agent-mail-conductor/docs/superpowers/handoff-mailbox-rollout.md`; `spec` is the agent-mail 2
design in the same `docs/superpowers/specs/`; `fork` is the agent-deck source at `/Users/johnw/src/fork/agent-deck`
(agent-deck 1.16.34 is installed; the fork's head says it is rebased on 1.16.34).

## 0. The answer in six lines

1. The director is two things: a Codex session (`conductor-director`) that acts only when something gives it a turn, and a launchd scanner (`monitor_sessions.py --loop`) that runs about every 69 s, reads catalogs and pane text, applies mechanical recoveries and pages John on Discord. Since 09:44 today the scanner gives the Codex session no turns.
2. It watches session health and pane-visible questions for every session in the fleet. It never reads mail or the content of work, so it catches no duplication, no conflict, no drift and no starvation.
3. The fastest existing path from any session to John: end a turn with a one-line question in its own pane. The scanner pages Discord with a mention and lights Hera's indicator within one pass.
4. Mail to `conductor-director` has no bounded latency today. Two cutover notices sent at 09:55 and 09:57 were still unread at 10:18. It has no presence record in `~/Mailbox`, and its last activity was 09:50.
5. A parent wake into the director measured 4.4 s. But it races the scanner, which drains the director's inbox every pass, and it depends on typing into a Codex TUI. The old heartbeat failed at that about 4,600 times.
6. A topic conductor gets the director's health cover for free: restarts, Continue recoveries, stall and question pages. It must not repeat any of them, and it must avoid tripping them by accident: a trailing `?` pages John, and a quiet pane draws a stall alert.

## 1. What the director does today

### 1.1 The two halves, and what wakes each

| Part | What runs | When it acts | Evidence (Verified) |
|---|---|---|---|
| Codex session `conductor-director` (id `e78b9a6d-1784130784`, gpt-5.5) | reads `AGENTS.md`, `POLICY.md`, `LEARNINGS.md`, then works | Only when it gets a turn. Sources: John typing in the TUI; John on Discord, which the bridge forwards with `agent-deck session send --no-wait`, or queues while the session is busy; the notify-daemon's typed `[INBOX] urgent · <child>` wake for one of its children; a CLI send. | `D/AGENTS.md:3,7,12`; `C/bridge.py:469-529`; `agent-deck triggers --session conductor-director --json` shows one trigger: launchd `com.agentdeck.transition-notifier`, origin push, last fire 2026-10-30T20:23:40Z, last line `[INBOX] urgent · uhj (f801e22b-1793391816): idle · details are in this turn's c…`, fires_24h 2. `--all` adds a `cli` sender (18 h ago) and session `8188c389` (failing, 37 h). `agent-deck list --json`: last_activity_at 2026-10-31T16:50:14Z (09:50 PDT). |
| Scanner | launchd `com.agentdeck.conductor-heartbeat.director` (KeepAlive, RunAtLoad), which runs `heartbeat.sh`, which execs `monitor_sessions.py --loop --label "Scheduled heartbeat"` | Every pass, then a 60 s sleep. An flock on `.monitor.lock` prevents overlapping passes. | `~/Library/LaunchAgents/com.agentdeck.conductor-heartbeat.director.plist` (plutil); `D/heartbeat.sh:2-6`; `D/monitor_sessions.py:1612-1630`; `D/POLICY.md:53-60`: "invokes `monitor_sessions.py` directly rather than sending input to this Codex TUI" |
| Watchdog | launchd `com.agentdeck.conductor-watchdog.director`, running `heartbeat_watchdog.py --loop` | Checks every 300 s. If no scan has completed for 600 s, sends one deduplicated Discord NEED. | `D/heartbeat_watchdog.py:18-27,37-47,58-62` |
| Bridge | launchd `com.agentdeck.conductor-bridge`, running `C/bridge.py` | Carries Discord to and from the conductor sessions. Its own heartbeat loop is disabled because an OS heartbeat plist exists. | `C/bridge.py:5-8,2605-2637`; `C/bridge.log:15439-15440` "OS heartbeat daemon detected, bridge heartbeat loop disabled" |

Stale text inside the director's own files (Verified):

1. `D/AGENTS.md:13` says "You receive periodic `[HEARTBEAT]` messages". That stopped with today's change (`D/POLICY.md:53-60`; `D/task-log.md:4-5`).
2. `D/AGENTS.md:37` says "The mailbox is `~/Products/mailbox`". That path now links to `~/Mailbox` (`readlink ~/Products/mailbox`).
3. `D/AGENTS.md:43` says "SSH delivery is not implemented; do not claim cross-host delivery". Syncthing carries the mail between hosts now.

### 1.2 Signals the scanner reads on each pass (Verified in code)

1. **Its own agent-deck inbox, drained at the start and end of every pass.** It runs `agent-deck inbox drain --json e78b9a6d-1784130784` at `D/monitor_sessions.py:1172` and `:1570`. A drain consumes the records ("Reading clears the inbox", `agent-deck inbox --help`). The records are only written to `task-log.md`; inbox items are never paged.
2. **The local catalog.** `agent-deck list --json`, minus the director itself (`:94,1183-1187`). Stopped and deleted sessions are skipped (`:1198-1200`), and so are sessions John retired in `state.json` (`:1010-1022`).
3. **Remote catalogs.** `agent-deck remote list --json`, then for each remote `ssh <host> agent-deck list --json`, falling back to `agent-deck remote sessions --json` (`:157-167,334-398`). The last pass covered 13 remote active sessions: andoria-08 (dsv41f, glm-5.2, pipeline), delphi-3bd4 (glm-5.3-flash, kimi-linear, lfm3, muse-spark), athena, clio/huququllah, pallas and vulcan (automation, monitor, nixos). Source: `D/state.json` `last_full_monitor_pass.session_ids`.
4. **Pane text.** The last 260 lines, by `tmux capture-pane` locally, over ssh and tmux remotely, or through a throwaway `agent-deck remote attach` client (`:203-207,401-409,1491-1567`).
5. **What it parses from the pane:**
   - five mechanical signatures (`:35-72`);
   - the session-limit reset line (`:73-76,543-569`);
   - the Codex "thinking a bit more" prompt (`:646-649`);
   - permission and model menus, and question lines within the last 15 lines after the latest assistant bullet (`:735-912`; the 15-line window is at `:783`, and only `line[-500:]` is kept at `:910-911`);
   - live-work markers such as background shells, agents, workflows and "running in the background" (`:1033-1064`);
   - a progress fingerprint that ignores token and timer telemetry (`:930-942`).
6. **Per-session directives** in `state.json` `session_directives`, where the tag is `continue-until-complete` (`:1025-1030`).

What it does not read (Verified: no `agent-mail` or `mailbox` string appears in `monitor_sessions.py`, `heartbeat.sh`, `heartbeat_watchdog.py` or `notify_discord.py`):

- mail;
- transcripts;
- git state;
- anyone else's children or inbox;
- agent-deck's `substate`, `background_work` or `last_activity_at` fields.

### 1.3 Cadence (Verified)

| Thing | Cadence | Source |
|---|---|---|
| Scanner pass | median 69 s between starts (min 67, max 160); a pass takes about 7 s; 28 passes from 09:44:31 to 10:17:10 | headings `## Scheduled heartbeat` in `D/task-log.md`; `last_full_monitor_pass` 10:17:10 to 10:17:17 |
| Stall alert | after 30 min with no meaningful pane change (5 h for vulcan/nixos); once per active lifetime; live work resets the clock | `D/POLICY.md:45-49`; `D/monitor_sessions.py:945-995` |
| Watchdog | checks every 300 s, alerts after 600 s | `D/heartbeat_watchdog.py:25,62` |
| Codex session | no cadence; turn-driven | sections 1.1 and 1.8 |

History that matters for the design:

- Until 09:44 today the heartbeat typed a check-in into the Codex TUI. `D/heartbeat.log` holds three kinds of failure from that era:
  - 3,983 "message reached 'conductor-director' but was never confirmed submitted" (issue #1793);
  - 355 "message typed but not submitted" (issue #1413);
  - 299 "cannot establish exact Codex turn acceptance".
- The last completed scan before the repair was 2026-10-30 16:06:03. John found the gap himself: "User reported missed requests for input" (`D/task-log.md:13-14`). That is about 17.6 hours with no fleet monitoring. The watchdog was added in the same repair (`D/task-log.md:4-5`).

### 1.4 What it does on its own (Verified in code and log)

The counts come from `D/task-log.md` since 2026-07-15. They count lines in per-pass logs, not unique events.

| Action | Trigger | Source | Pass entries |
|---|---|---|---|
| `agent-deck session restart`, then `Continue` | catalog status `error` | `D/POLICY.md:11,227-231`; `D/monitor_sessions.py:296-306,1205-1218,1331-1349` | 217 |
| `Continue` | waiting, with a `continue-until-complete` directive and no live work (local sessions only) | `:1288-1301`; the `state.json` directives date from July (nix-review) | 169 |
| `Continue` | API Error 529 Overloaded | `D/POLICY.md:175-185` | 96 |
| `Enter` | a `Continue` still sitting unsubmitted in the input box | `:1256-1262` | 35 |
| `Continue`, on every pass while visible | Quota exceeded | `D/POLICY.md:199-209` | 22 |
| `Continue` | Codex model at capacity; API 500 | `D/POLICY.md:117-131,187-197` | 3 and 2 |
| `2` | Codex extended-thinking prompt | `D/POLICY.md:133-144` | 2 |
| `Enter` | `ct` session's `.vo` cleanup confirmation | `D/POLICY.md:109-115` | 2 |
| `/compact`, then `/goal resume` or `Continue` | Codex "This content can't be shown" | `D/POLICY.md:146-173`; `:1233-1249` | (rare) |
| a `Continue` scheduled for one minute after the reset | Claude session limit | `D/POLICY.md:211-221`; `:572-632` | (jobs in `state.json`) |

In all, 618 of the 8,276 passes that logged an `Actions` line took any action (7.5%).

**Content replies are almost never sent.** `D/POLICY.md:8,235-254` permits answers to "Should I proceed?"-style questions. But `:12` forbids ordinary messages to running sessions, and `:29` forbids auto-answering human questions. The log's norm is "No safe auto-response" (for example `D/task-log.md:133367,133391`). The scanner itself never sends content.

### 1.5 What it escalates and how it reaches John (Verified)

1. **Discord.** It posts to the configured `#agent-deck` channel with an explicit mention, through `D/notify_discord.py` (`D/POLICY.md:31-41`; `D/notify_discord.py:29-57`).
   - A send counts as delivered only when the receipt carries a message id and the mention (`D/monitor_sessions.py:1133-1135`).
   - The dedup key is sha256 of `[session id, item]`, and it is kept only after a confirmed send, so a failed send retries on the next pass (`:1098-1142`).
   - Paged items are questions, stalls and errors. Inbox items are never paged.
   - Since 09:44 today: 14 pages (7 question, 4 stall, 3 error).
2. **Hera's indicator 1.** It turns on when any question is found. It turns off only after a complete scan finds none, and stays unchanged when coverage was incomplete (`D/POLICY.md:20-25`; `D/monitor_sessions.py:1072-1095`).
3. **The Codex TUI**, as lines of the form `NEED(<location>): <action>` (`D/POLICY.md:43`). The scanner skips the director's own pane (`SELF_TITLES`, `:94`). So John sees these lines only if he looks at the TUI, or if the Codex session also runs `notify_discord.py`, as its policy says it must (`D/POLICY.md:27-35`).
4. **Never mail.** It escalates to John on Discord and answers peers by mail (`D/task-log.md:134170,134181,134186`).

### 1.6 What it logs (Verified)

- **`D/task-log.md`**: 134,702 lines and 12,144 sections since 2026-07-15.
  - Each pass appends one block: Inbox, Coverage, Counts, Waiting, Actions, Questions, Stalled, Errors, Discord, Indicator (`D/monitor_sessions.py:1593-1608`).
  - The Codex session adds narrative entries (for example `:1-17`, `:134141-134205`).
- **`D/state.json`** (125 KB):
  - `last_full_monitor_pass`;
  - `progress_watch` (238 entries; it already tracks `local/mail-conductor`);
  - `question_watch` (8 open questions, for example andoria-08/glm-5.2 "On in both, or off in both?" and local/nix's pinning question);
  - `attention_delivery` (28 entries), `attention_indicator`, `session_directives`, `retired_monitoring_sessions` and `session_limit_continue_jobs`.
  - The Codex session also wrote ad hoc coordination records there. `mcp_plan_coordination` records owners, boundaries and publication holds across nix, mcp-servers, catalog and nyx. `session_communication_watch` follows pi's mailbox prototype. `pending_triggers` holds "When local/ares-attn has announced completion ... instruct local/ares-refine to rebase". This is topic-conductor work done by hand, kept as free-form notes.
- **`D/heartbeat.log`**: the scanner's JSON for each pass, plus the old send errors. **`D/watchdog.log`** is empty.

### 1.7 What it has learned (`D/LEARNINGS.md`, Verified)

1. Never lifecycle-manage a session as a form of recovery: it once restarted two errored sessions without authority (`:31-37`). The current policy makes error-status restart the single exception (`D/POLICY.md:11`).
2. Remote coverage needs an attach fallback when raw ssh and tmux hang (`:39-45`).
3. Typed input to Codex needs a pause before Enter (`:27`; `D/POLICY.md:99-107`).
4. A Discord channel post without a user mention notifies nobody (`:63-69`).
5. Review sessions must not produce routine NEED noise (`:71-77`).
6. Refresh the catalog on every pass (`:79-85`).
7. An inbox signal waits for the scan in flight (`:95-101`).
8. A Codex session can die on send when it is bound to a subagent thread (`:47-53`).
9. An entry is promoted to POLICY after 3 recurrences (`:8`).

Nothing in it concerns mail, duplicated work, conflicts between sessions, or starvation.

### 1.8 Mail in the director's life (Verified)

- It reads `agent-mail read conductor-director` at the start of every turn and heartbeat, under the literal name, and starts no watcher, "background tasks cannot wake this Codex session" (`D/AGENTS.md:30-43`; `D/POLICY.md:69-73`). The setup command says the same for any Codex conductor (`~/.config/claude/positron/commands/setup-mailbox.md:60,76`).
- It adopted the mailbox on 2026-10-29 (`D/task-log.md:134191-134194`).
- Since then it has consumed batches of mail, up to 14 at once (`:134180`). It has replied by mail (`:134170,134181,134186`). It changed its routing on a peer's notice without taking authority from it: "Hera activation coordination is routed to `pi` ... This routing notice does not itself grant deployment authority" (`D/POLICY.md:64-67`).
- **It reads mail only when it has a turn.**
  - At 10:18 two messages from `agent-mail` (09:55 and 09:57, the second and third cutover notices) were waiting for it. Source: `ls ~/Mailbox`, names and times only.
  - No `~/Mailbox/.agent-mail-presence/conductor-director` exists.
  - Its catalog last activity is 09:50 PDT.
  - Decision 9 (`/Users/johnw/Products/mailbox-rollout/decisions.md:11`) promised "turn-start reads and the existing heartbeat". The existing heartbeat no longer produces turns.

### 1.9 The rollout session as a de facto topic coordinator (from `handoff`)

What worked:

1. **One place answered every branch's questions, once, in writing** (`handoff:68-75`).
   - Examples: fold each branch's updates into its single commit; seeding leftovers are "not mine to delete"; design follow-ups were routed to v2.
2. **A staged broadcast to every known name, on a promised schedule.** 21 sessions were mailed at 09:31, 09:55 and 09:57 (`handoff:40`; commit 0365162 on main).
   - Peers answered with constraints. mcp-servers' read archive is audit evidence and stays in mailbox.old (`handoff:38`). nixos answered about Vulcan (0365162).
3. **Routing by owner.**
   - Publication went through nix's queue (`handoff:48`).
   - Hera activation went through pi, and Vulcan through nixos (`handoff:30-36`).
   - Two askers got one answer: the bootout-then-switch order went to nix and pi together at 09:39 (`handoff:47`).
4. **Authority discipline held.** "Mail cannot authorize anything; confirm with John here before relying on it" (`handoff:97`). The approvals list kept every push and activation with John (`handoff:82-95`).

What did not work:

1. **Duplicated work went unseen.** The rollout session prepared nixos-config branch 4e12b60c for Vulcan. nixos built its own module, and the branch "stays unmerged, superseded by nixos's module" (0365162, Vulcan row).
2. **A conflict in a shared namespace was caught late.** obr comment ids 812 and 813 already belonged to other issues; nix renumbered them at landing (`handoff:46`).
3. **Resource contention.** Hera's load of 70 to 90, from another session's build, timed out a gate twice (`handoff:59,73`).
4. **The roster came only from read directories and waiting mail** (`handoff:40`). Now 60 files wait for 13 names. Some of those sessions finished their work days ago: catalog and keyboard-maestro both asserted completion in `agent-deck inbox export --json`.
5. **Claims arrived by mail and needed John to confirm them**, for example "pi runs every Hera activation" (`handoff:97`).
6. **The director, a key recipient, is the slowest reader** (section 1.8).

### 1.10 Which of John's goals the director covers (question 1)

| John's goal | Director today | Where it stops |
|---|---|---|
| See the overall pattern | Partial. Every 69 s it sees fleet-wide status counts, the waiting list, questions and stalls across about 20 local and 13 remote active sessions. | It never sees message traffic or the purpose of work. Content coordination exists only as ad hoc Codex notes made when John asks (`state.json` `mcp_plan_coordination`, `pending_triggers`). |
| Catch duplication | No. There is no such logic in `monitor_sessions.py`. The 743 "duplicat" hits in `task-log.md` are about duplicate alerts. | It missed the Vulcan branch duplicate (1.9). |
| Catch conflicts | No. It only relays a session's own question, for example "pause the competing GLM build" (`D/task-log.md:75324-75334`). | It missed the obr id collision and the Hera load contention (1.9). |
| Keep work on purpose | No. It is forbidden to steer (`D/POLICY.md:12,29`). | — |
| Nobody stuck | Yes for mechanical stuck states and error status (1.4), plus a one-shot page after 30 minutes of quiet. | Not for "waiting on a peer's answer", "blocked by another session's work" or "waiting on unread mail". |
| Nobody starved | No. Waiting at an empty prompt is not an attention item (`D/POLICY.md:22-23,87`). The only keep-going mechanism is John's per-session `continue-until-complete` directive. | — |
| Surface questions proactively | Yes for a question visible in a pane: a Discord page with a mention plus the indicator within one pass (about 69 s). | It misses questions asked by mail, `NEED:` lines without a `?`, and anything above the last 15 lines. It reports only the last `?` line, cut to 500 characters. It was blind for 17.6 hours before today's watchdog. |

## 2. Reusable (question 2)

All Verified today unless marked.

1. **The catalog**, from `agent-deck list --json`. Fields seen:
   - `status`, and `status_source` (live or cached);
   - `substate`: `idle-at-empty-prompt`, `interactive-menu`, `background-work`, `running`;
   - `substate_detail`, and `background_work` `{kind, task, step, steps, elapsed}`, for example `workflow conductor-exploration 0/6`;
   - `last_activity_at`, `parent_session_id`, `parent_project_path`, `group`, `conductor_name`;
   - `viewers`, which lists attached terminals: a cue that John is watching that session.

   The scanner uses none of the substate or activity fields. A conductor can.
2. **`agent-deck status --json`** gives counts. **`agent-deck status --stale --json`** gives `last_activity_age_seconds` and `reasons` per session and is read-only. Today: 49 sessions, 34 stale at 24 h.
3. **`agent-deck session children --json`**: live status plus the last asserted completion for each child; read-only.
   - `--follow` streams JSONL: snapshot, added, status transitions, done, removed, error, and a heartbeat every minute.
   - `--until-done` exits once every child needs input or has finished. This is a ready-made stuck-or-finished feed for a conductor that is the parent of its workers.
4. **`agent-deck inbox export --json`**: this host's completion and transition records, without consuming them.
   - Today 38 records: 31 `finished` records with `done_status` and `done_summary` from 25 children, and 7 running-to-waiting transition records, most carrying `tier` (info), `trigger` (task or inbox) and the child's turn `text`.
   - `--after <cursor>` reads only what is new.
   - `inbox stats --json --all` and `inbox writer-status` show wake counts and latency.
   - For other hosts the help names `agent-deck remote drain <name>`. Whether it consumes: I did not check.
5. **The parent conventions** (from `agent-deck session primer`, Verified):
   - the completion sentinel `===AGENTDECK_DONE=== status=<ok|fail> summary=<one line>`, which reaches the parent as "one urgent record";
   - "start a line with NEED: or end your message with the question (a trailing ?); that wakes it";
   - "Anything else you print is delivered as progress on its next turn".

   Measured parent-wake latency (`last_urgent_latency_ms`, which is "last urgent record commit -> delivery", `fork/internal/session/inbox_stats.go:65-66`):

   | Parent | Latency |
   |---|---|
   | director | 4,412 ms (23 urgent wakes since 2026-10-20) |
   | 1592a8dd | 1,422 ms |
   | b36d13d3 | 32,465 ms |
   | agent-mail (this session's parent) | 125,729 ms |
   | nix | 333,562 ms |

   A busy parent waits.
6. **Named conductors in agent-deck itself** (Verified from help and code; not run).
   - `agent-deck conductor setup <name> -agent claude|codex|hermes|pi` creates a conductor with its own directory, POLICY.md, HEARTBEAT_RULES.md and heartbeat. "Multiple conductors can exist per profile".
   - The session is titled `conductor-<name>` (`C/bridge.py:315-317`).
   - The stock bridge heartbeat scopes each conductor to the sessions whose agent-deck group equals its name or sits under `name/` (`C/bridge.py:2664-2675`). That loop is disabled on hera, though (section 1.1).
   - Discord messages that start `<name>: ` are routed to that conductor (`C/bridge.py:1097-1110,2498`).
7. **The director's escalation path costs nothing to reuse.** Any question line a topic conductor leaves at the tail of its own pane reaches Discord and the indicator within one pass (section 3, path C). The director already tracks new sessions automatically: `local/mail-conductor` is in `progress_watch`.
8. **The director's health cover.** If a topic conductor errors, the director restarts it and sends `Continue`. If it hits a 529, quota or capacity message, the director sends `Continue`. If it sits quiet for 30 minutes without live work, one stall page goes out (1.4).
9. **`agent-deck triggers`** answers "what wakes this session". A conductor can audit whether each worker can be woken at all: Claude by a background watcher, Pi by a loop, Codex by nothing.
10. **Other helpers (Verified from help only):**
    - `agent-deck ask choice|text` asks John in the Mac app or TUI dialog and returns the answer to the asker. It waits up to 30 minutes, and only one request may be open at a time (`fork/docs/ask.md`).
    - `agent-deck session annotate` records durable tags (purpose, ticket, why, decision, outcome).
    - `agent-deck session set-parent` sets a parent link, and `agent-deck group move` regroups a session.
11. **Patterns worth copying.**
    - The watchdog: an independent liveness check that pages once per stale state (`D/heartbeat_watchdog.py`).
    - The LEARNINGS entry format and its promote-after-three rule.
    - Discord delivery confirmed by receipt.
    - Dedup kept only after a confirmed send.

## 3. Hand-off to the director and to John (question 3)

Paths a topic conductor could use to raise a question, compared:

| Path | Who acts | Latency today | Risk |
|---|---|---|---|
| **A. Mail to `conductor-director`** | the director's Codex session, at its next turn | Unbounded. No turn since 09:50; two notices from 09:55 and 09:57 still unread at 10:18; no presence record (1.8). v2 changes nothing here: the plan wires Codex for "turn-start reads and heartbeat" (`agent-mail-v2/.../2026-10-31-agent-mail-mcp-channels.md:780`) and leaves Codex hooks out (`:976`). | Questions pile up silently. A read consumes the mail. The scanner never sees mail. The director escalates only if it judges the mail important. |
| **B. Parent wake.** The topic conductor is a child of `conductor-director` (`agent-deck session set-parent`) and prints `NEED:` or ends with `?`. | the notify-daemon types `[INBOX] urgent · <child> …` into the director's pane | 4.4 s measured for the director (inbox stats); 1.4 s to 334 s for other parents | 1. The scanner drains the director's inbox every pass (`D/monitor_sessions.py:1172,1570`), so the Codex turn's own drain (`D/POLICY.md:78`) can find the record gone; it survives only in `task-log.md`. Whether the typed line carries the question text: I did not check. 2. Typing into this Codex TUI failed about 4,600 times under the old heartbeat (1.3). Whether the notify-daemon uses the same submission path: I did not check. 3. A session has one parent, so a conductor that is the director's child no longer reports to whoever launched it. 4. The question text is peer text placed in the director's context. 5. A trailing `?` also fires path C, so John is paged twice if the director pages too. |
| **C. Pane question.** The topic conductor ends a turn with one line ending in `?`. The scanner pages Discord with a mention and turns on indicator 1. | launchd scanner; no Codex turn needed | one pass: median 69 s, at most 160 s observed, plus the Discord post. Proven today: `local/nix` and `andoria-08/glm-5.2` questions paged with receipts (`D/task-log.md:1-9` and the tail). | 1. Detection is heuristic (`:735-912`, about 60 exclusion rules): it needs a `?`, looks only at the last 15 lines, and keeps only the last line, cut to 500 characters. 2. If the worker's own pane also shows the question, John is paged twice. 3. It bypasses the director's judgment. 4. The text goes to Discord, a third party, so no secrets. 5. A trailing `?` also wakes the conductor's own agent-deck parent, if it has one. |
| **D. `agent-deck ask choice/text`** | John, in the app or TUI dialog | as fast as John answers; at most 30 min | Help-only (Unverified). Whether it notifies the phone: I did not check. One open request at a time. The asker blocks. It bypasses the director. |
| **E. Run the director's `notify_discord.py` directly** | the topic conductor | seconds | It duplicates the director's path, skips its dedup and the indicator, and reuses the director's bot configuration. Do not. |
| **F. `agent-deck session send conductor-director`** | — | — | Forbidden for relaying peer requests (README:43; setup-mailbox.md:31). It is user-role injection, which spec section 2 rules out (`spec:39`), and it hits Codex submission failures. Do not. |

From the director to John, today:

1. **Discord with a mention.** The scanner sends it automatically. The Codex session sends it only when it takes a turn and runs `notify_discord.py`.
2. **Hera's indicator 1.** The scanner sets it.
3. **NEED lines in the TUI.** The scanner never reads them (1.5).

When John answers, his reply reaches the director in its TUI or through Discord (the bridge). It reaches a topic conductor by Discord only if that conductor is an agent-deck conductor and John prefixes `name:` (2.6).

Default suggested for the designer:

1. **Anything John must answer goes out by path C,** as one line in the topic conductor's pane. Only the conductor surfaces it, and it adds context and dedups across workers.
2. **Workers that route a question through the conductor end their own turn without that `?`**, so John is not paged twice.
3. **Only non-urgent digests go by mail to `conductor-director`.** Nothing time-sensitive travels by mail alone.
4. **Path B only if John wants the director's judgment in the loop,** and only after one drainer is chosen. Either the scanner uses `inbox peek`/`export`, or the Codex session stops draining.

## 4. Do not duplicate or fight (question 4)

1. **Mechanical recoveries belong to the director:** `Continue` and `Enter`, `2`, `/compact`, scheduled `Continue` and restart-on-error (1.4). A topic conductor never types into a pane, restarts, stops or forks a session. If it notices a worker sitting on a 529 or quota message, it leaves it alone; the scanner handles it within about 69 s.
2. **Two nudgers on one session.** The director injects `Continue` into waiting sessions that carry John's `continue-until-complete` tag (`:1288-1301`). A topic conductor nudges only by mail, which the worker reads at its own turn start or doorbell, and only about content ("here is your next item"), never liveness. Before nudging a session that has a directive, it should know about the directive. Today the directives live only in the director's private `state.json`, which a conductor should not read. That is an open question below.
3. **Double paging.** The scanner pages each new question text once per session (`:1121-1123`). A conductor that re-surfaces a worker's visible question pages John a second time. The conductor must surface only what it adds: merged, de-duplicated or newly blocking questions. Workers must not leave the routed question as their trailing line.
4. **Stall alerts against a quiet conductor.** A conductor is quiet by design. With no live-work marker it draws a one-shot stall page after 30 minutes (`:945-995`). A Claude conductor with a visible background watcher may count as live; the patterns match "N shell(s)", "background shell", "running in the background" (`:1033-1064`). Whether a Claude pane with one background Bash task matches: I did not check. agent-deck itself reports the watcher-running `agent-mail` session as `idle-at-empty-prompt`. Exemption today comes only from the `-review` or `-conv` title suffix (`:677-682`), which is the wrong signal for a conductor.
5. **Accidental pages.** The scanner treats any trailing `?` line in a pane as a question for John (`:714-723,783-911`). A conductor musing "Shall I nudge glm-5.2?" pages John. The conductor's skill must forbid trailing questions unless paging John is the intent.
6. **The inbox.** `inbox drain` consumes. A topic conductor never drains another session's inbox. It uses `inbox peek`, `inbox export` or `session children` for anything that is not its own.
7. **The indicator and the Discord bot.** Both belong to the director: one bot and one channel. No second notifier.
8. **Default Discord routing.** The bridge sends un-prefixed Discord messages to the alphabetically first conductor (`C/bridge.py:294-312,325-328`). A topic conductor made with `agent-deck conductor setup` whose name sorts before `director` would take John's ordinary Discord messages away from the director. Example names that would do this: `agents`, `ingest`, `deck`. Names after `director` are safe, for example `nix` or `positron`.
9. **The director's existing Nix coordination.** `mcp_plan_coordination`, `pending_triggers` and the Hera routing to pi (`D/POLICY.md:64-67`) are de facto topic conducting for Nix. A Nix topic conductor would overlap until John decides who owns it.
10. **Restart policy.** A topic conductor must never restart anyone. The director's one restart rule covers errored conductors too.

## 5. Identity and liveness facts (question 5)

1. **The mailbox name is the agent-deck title, in lower case,** and must be unique across every host that shares the mailbox (`/Users/johnw/src/sys/agent-mail/README.md:11`, Verified).
   - agent-deck does not enforce unique titles. Two sessions titled `agent-cat` existed at once (`D/task-log.md:1089`, "local/agent-cat (two sessions)").
   - v2 warns when the member record shows another host within 600 s (`agent-mail-v2/agent-mail:1112,1363-1364`).
2. **The director reads mail only at turn start,** under the literal name `conductor-director` with `--id e78b9a6d-1784130784` (`D/AGENTS.md:30-39`).
3. **v1 presence.** `~/Mailbox/.agent-mail-presence/<name>` is touched by any `read`, `watch` poll or `pending`, at most once a minute (`agent-mail:46-48,106,248-262`). It syncs between hosts (README:110). `send` warns when it is more than an hour old (`:99,610-615`).
   - Today it exists for 13 names: agent-mail, athena, glm-5.2, glm-5.3-flash, kimi-linear, lfm3, mail-conductor, mcp-servers, muse-spark, nix, nixos, pallas, pi.
   - There is none for `conductor-director`.
4. **v2 member records.** `~/Mailbox/.agent-mail/members/<name>.json` holds `{"v":2,"name","host","checked","subs"}` (`agent-mail-v2/agent-mail:592-617`).
   - It is refreshed when the subscriptions change, or at most every 300 s (`:423`). It also touches the v1 presence file.
   - It is peer-writable and only displayed or warned about (`:570-589`; `spec:230,442`).
5. **What presence means for "who is alive".** Presence says "mail was checked", not "working".
   - A Claude session with a background watcher stays fresh while idle.
   - A Codex session stays invisible while it waits. The director looks dead by this measure while its scanner runs fine.
   - Liveness has to combine agent-deck's catalog (`status`, `substate`, `background_work`, `last_activity_at`), `children --follow` transitions, and presence and members for mail reachability.
   - The director's own liveness is visible in its `state.json` `last_full_monitor_pass.completed_at`, which the watchdog uses. Reading another session's private state is not a documented interface (open question 6).
6. **Cross-host limits.** Parent links, `children` and `inbox export` are per host and per profile ("this host's completion/transition records", `agent-deck inbox export --help`). The Positron ingestion sessions run in the agent-deck instances on andoria-08 and delphi-3bd4. Whether a parent link can span hosts: I did not check. Mail over Syncthing is the only channel known to cross every host.

## 6. Implications for agent-mail

1. **Spec 6.8 is stale on the Codex conductor's heartbeat.** It says "Heartbeat, Verified. `heartbeat.sh` sends a fixed check-in ... through `agent-deck session send` when the session is idle" (`spec:396`). Since 09:44 today `heartbeat.sh` runs the scanner and sends nothing to the TUI. Before that it failed thousands of times.
   - Decision 9 therefore leaves `conductor-director` with turn-start reads only.
   - The v2 README task (plan `:780`, "Codex's turn-start reads and heartbeat") should not promise a heartbeat read.
   - This is a correction the rollout session (mail-dev) owns.
2. **Nothing time-sensitive should depend on mail to the director.** Either John approves a wake path for it, or the conductor design routes John-bound questions through path C. The Codex hooks in spec 6.8 (SessionStart, UserPromptSubmit) run only at turn boundaries, so they would not shorten an idle wait (Unverified, per `spec:397`).
3. **The scanner could cheaply report unread director mail.** Each pass could run `agent-mail pending conductor-director`, which consumes nothing, and page when mail has waited, say, 15 minutes. Caveat: v1 `pending` also refreshes presence (`agent-mail:46-47`), which would tell senders "checked" while the mail sits unread. This is the director's file and John's call.
4. **Questions asked by mail are invisible to the director.** If workers ask the topic conductor by mail, only the conductor sees them. Re-surfacing what needs John into its pane (path C) is the conductor's job, and the skill should say exactly how: one line, a trailing `?`, a topic prefix, no secrets.
5. **Liveness.** Presence and members can't show a Codex session's liveness. A conductor's starvation and stuck checks must use agent-deck's catalog fields plus presence, never presence alone.
6. **Mailbox hygiene is a natural conductor or `doctor` duty.** 60 files wait for 13 names, some for sessions whose work ended. A per-recipient age report, joined with the agent-deck catalog, would find dead or never-joined recipients. The v1 roster problem from 1.9 points the same way.
7. **Names.**
   - A channel cannot share a name with a session that has a member record, so `+nix` is blocked by `nix` (`spec:215`; plan `:987`).
   - `agent-mail` is reserved (`spec:181`).
   - If topic conductors take agent-deck's `conductor-<name>` convention, the name `conductor` means two jobs. The bridge's default routing (section 4, item 8) constrains the choice of name too.
8. **Trust is unchanged.** A topic conductor is a peer. The director's own precedent is the right line: routing knowledge from a peer is accepted ("Hera via pi"), authority is not (`D/POLICY.md:64-67`). Conductor mail carries the same peer banner, and workers must not read a conductor's instructions as John's.
9. **Harness choice for a conductor.** A Codex conductor cannot hold a watcher (`D/AGENTS.md:42`), so it only reacts at turns. A Claude conductor (background `watch`) or a Pi conductor (loop) can react to traffic. A Claude conductor also keeps a live-work marker that may hold off the director's stall alert (unchecked, see section 4, item 4).

## 7. Open questions for John (most important first)

1. When a topic conductor has a question for you, where should it land?
   - (a) Discord and the indicator directly, through the director's scanner: about 70 s, no director judgment.
   - (b) The director's Codex session, which adds judgment but needs a reliable wake.
   - (c) Both, accepting double paging unless policy dedups them.
2. Should topic conductors be agent-deck children of `conductor-director`, so their questions wake it (4.4 s measured)? If so, which one drains the director's inbox: the scanner or the Codex session?
3. Is unbounded mail latency to `conductor-director` acceptable? Decision 9 assumed a heartbeat that no longer reaches the Codex TUI. If not acceptable, which fix?
   - The scanner pages on old unread director mail.
   - Turns are given back to the Codex session.
   - Nobody mails it anything urgent.
4. Does the director hand its ad hoc Nix coordination to a Nix topic conductor? That covers `mcp_plan_coordination`, `pending_triggers` and the Hera-via-pi routing.
5. What draws a topic's boundary? Options:
   - the agent-deck group, as the stock bridge's per-conductor scoping does (`C/bridge.py:2664-2675`). Today's groups (Positron, ledger, my-sessions, category-theory) do not match your topics.
   - a roster each conductor keeps.
   - v2 channel subscribers.
   Note: the Positron ingestion sessions live in other hosts' agent-deck instances.
6. May the director's POLICY.md gain a clause for topic conductors? It would cover three things:
   - no 30-minute stall page for them;
   - no second page for a question a conductor has surfaced;
   - an explicit, documented liveness signal and directive list that conductors may read, instead of the director's private `state.json`.
   Only you, or the director in its own session, edit that file.
7. Should topic conductors be real agent-deck conductors? Through `agent-deck conductor setup`, each gets its own POLICY.md, HEARTBEAT_RULES.md, heartbeat and Discord routing (`name:`). Then "conductor" names two jobs, and a name sorting before `director` would capture your un-prefixed Discord messages. The alternative is ordinary sessions under another word (foreman, steward).

## Appendix: what I ran (all read-only)

- `agent-deck --help`, `session --help`, `inbox --help`, `triggers --help`, `session children --help`, `status --help`, `ask --help`, `conductor setup --help`, `remote --help`.
- `agent-deck status --json`, `status --stale --json`, `list --json`.
- `agent-deck session children --json` (for this session, `agent-mail` and `conductor-director`), `session primer`.
- `agent-deck triggers list [--all]`, `triggers --session conductor-director [--json]`.
- `agent-deck inbox export --json`, `inbox stats --json --all`, `inbox writer-status --json`.
- `agent-deck conductor list`, `conductor status`.
- `plutil -p` on the LaunchAgents.
- Names and times only via `ls` in `~/Mailbox` and `~/Mailbox/.agent-mail-presence`.
- `grep`, `head`, `tail` and line-range reads on `D/` logs and code. `python3 -I` summaries of `state.json`, `list.json` and `inbox-export.json`.
- No `agent-mail` read, watch, pending or send. No `session send`, `launch`, drain or `ask`. Nothing written outside this report.
