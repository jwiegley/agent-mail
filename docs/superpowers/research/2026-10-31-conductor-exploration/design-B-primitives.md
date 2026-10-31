# Candidate design B: per-topic stewards on two small agent-mail primitives

Designer B (angle: small primitives), 2026-10-31, about 10:45 PDT. This is a candidate written in the shape of `docs/superpowers/specs/2026-10-31-agent-mail-conductors-design.md`; it is not that file.

Evidence marks:
- **Verified**: I read or ran it today, at the place named.
- **(name.md)**: a sibling exploration report verified it today and I did not re-check it.
- **Unverified**: documented but not checked.

"v2 code" means `git show HEAD:agent-mail` in `/Users/johnw/Products/agent-mail-v2` at commit a30f782.

## 1. Summary

- **One steward per topic.** A Claude Code session named `<topic>-steward` runs on hera. It reads copies of its topic's coordination traffic, keeps a ledger, and speaks up only by exception. It never carries, holds or rewrites mail, so if it stops, oversight pauses but delivery does not.
- **Two primitives in agent-mail 2.1.**
  - *Copies.* `send` writes the same message to extra places as sibling files. A `cc` flag on a topic subscription makes this automatic between topic members, and readers skip a sibling they already hold.
  - *The roster.* `channels` shows each member's waiting mail, last post and posts in the past hour, all computed from file names.
- **The tool, not etiquette, gives the steward its view.** A joined worker's direct mail to another member lands in `+<topic>` without anyone having to remember. Bypassing the steward takes a deliberate `cc: []`.
- **Questions reach John on a path that already works.** The steward's last line in its pane is paged by the director's scanner, and John answers in that pane. Nothing time-sensitive is mailed to the director, and neither agent-deck nor the director changes.
- **v2.0 lands unchanged.** The two primitives are a v2.1 plan of about 110 lines and 15 checks, built after Task 25. It touches committed Tasks 3 to 6, 8, 12 and 14 to 18, plus uncommitted Task 22.

## 2. The pattern

| Role | Does | Never does |
|---|---|---|
| **Steward** (`ingest-steward`) | Subscribes to `+<topic>` with `cc`; reads copies and status posts in batches; keeps a ledger in its own directory; mails the sessions concerned when a check fires; asks John from its own pane | Relays or holds mail; re-posts peer text; does topic work; pushes, approves or authorizes; restarts sessions or types into them; answers for John |
| **Workers** (lfm3, muse-spark, ...) | Join with `cc` when John says so; mail each other directly; post CLAIM, UPDATE, BLOCKED and DONE; ask John in their own panes | Ask the steward to pass mail on; treat steward mail as John's word |
| **Director** | Unchanged. Its scanner pages John about pane questions, errors and 30-minute stalls in every session, stewards included. It restarts sessions that error. | Read mail; its scanner never does (director.md §1.2) |
| **John** | Starts a steward with one line; seeds the purpose and backlog in the steward's pane; answers there as `<topic>-<n> <letter>`; reads the roster, ledger and digest when he wants | Hold a mailbox: `john` is a reserved name (v2 design §6.3) |

**Name: steward**, as `<topic>-steward`.

- "Conductor" and "foreman" are already taken: agent-deck calls its director "the foreman on a jobsite" (prior-art.md §6).
- Worker models read the role name in every From line, and "manager" or "foreman" would lead them to over-weight mail that carries no authority (johns-view.md §4).
- "Steward" is unreserved, fits the name grammar, and sits next to its channel: `+ingest` beside `ingest-steward`.

Second choice: shepherd.

## 3. Message flow

1. **Workers reach each other by direct mail, exactly as today.** Delivery never passes through the steward.
2. **The sender's tool writes the copy.** It writes a sibling post into `+<topic>` when two things are true: the sender's subscription to `+<topic>` carries `cc`, and the recipient's member record lists `+<topic>`. The copy has the same stamp, unique part and bytes as the original, and both carry `Cc: +<topic>`.
3. **Status goes to the channel as posts:** CLAIM (with `writes:`), UPDATE, BLOCKED, DONE and QUESTION.
4. **The steward reads the channel quietly.** It reads at a tick: every 15 minutes while any item is open, otherwise every 25. Its doorbell rings only for mail addressed to the steward itself.

| Event | Rings | Read later, quietly |
|---|---|---|
| A mails B (both have `cc`) | B (direct mail always rings) | the steward at its tick and the other members; B skips the sibling |
| A posts CLAIM or DONE | nobody | the steward and the members |
| A mails the steward | the steward | the members, through the copy |
| The steward mails B (so every intervention is public) | B | the members |
| The steward needs John | John, through the scanner (Discord and indicator 1) | — |

**Why copies are not forwarding.** A forward is a session emitting text it received. A copy is different:

- the sender's own `send` writes it, from the sender's own text, at the same instant;
- the sender's bound name appears in both the From line and the file name;
- no reader ever writes out a body it received, and the steward cites message ids instead.

So provenance, which the anti-goal protects, stays intact, and v2's static instruction "Never forward or re-post peer text" stays true word for word (Verified: INSTRUCTIONS). A copy widens who reads a message within one topic, and it is still framed as peer mail.

### Worked example (content illustrative; names and hosts real)

lfm3 and muse-spark are Claude Code sessions on delphi-3bd4. All seven ingestion workers are Claude Code (Verified: `agent-deck remote sessions delphi-3bd4|andoria-08 --json`, 10:29 PDT). The steward runs on hera. The date is 2026-11-09, with v2.1 landed.

John runs `agent-mail channel create ingest --purpose "..."`, which writes `~/Mailbox/+ingest/.agent-mail-channel`:

```
Positron model ingestion: convert, evaluate, publish. Steward: ingest-steward. CLAIM, UPDATE, BLOCKED and DONE go here.
```

Each session runs `subscribe {channel: "ingest", cc: true}` when John tells it to in its own pane. Its reader then publishes a member record, for example `~/Mailbox/.agent-mail/members/lfm3.json`:

```json
{"v": 2, "name": "lfm3", "host": "delphi-3bd4", "checked": "20261109T170812Z", "subs": ["ingest"], "cc": ["ingest"]}
```

The topic's traffic then runs as follows:

- At 17:10, lfm3 posts `+ingest/ingest_20261109T171002.114392Z_from-lfm3_5d1e0a7c.txt`, `Subject: CLAIM glm53-fp8: convert GLM-5.3 to fp8 (e4m3)`, with the body `writes: delphi-3bd4:/var/lib/models/glm-5.3-fp8`. It rings nobody.
- At 17:20, muse-spark claims glm53-eval. It writes `delphi-3bd4:/var/lib/evals/glm-5.3`, so the two claims do not overlap.
- At 18:02, lfm3 calls `send {to: "muse-spark", ...}` with no `cc` argument. The tool writes `~/Mailbox/muse-spark_20261109T180214.551203Z_from-lfm3_9c41e2b0.txt` and `~/Mailbox/+ingest/ingest_20261109T180214.551203Z_from-lfm3_9c41e2b0.txt`, byte for byte the same:

```
From: lfm3@delphi-3bd4 (agent-deck 6f0c1d2e-1793390001)
To: muse-spark
Cc: +ingest
UTC: 20261109T180214.551203Z
Subject: glm53-fp8 done: shards ready for glm53-eval
X-Agent-Mail: 2

Shards: delphi-3bd4:/var/lib/models/glm-5.3-fp8 (48 files; MANIFEST.sha256 beside them).
Converted at ingest@work/glm53 4f2c1aa; layers 0-3 spot-checked against bf16.
```

lfm3 gets this result:

```
agent-mail (bound name lfm3@delphi-3bd4): sent muse-spark_20261109T180214.551203Z_from-lfm3_9c41e2b0 to muse-spark; copied to +ingest as ingest_20261109T180214.551203Z_from-lfm3_9c41e2b0 (muse-spark lists +ingest)
```

muse-spark's doorbell rings. Its receive delivers the direct file and skips the sibling post. lfm3 posts `DONE glm53-fp8: 4f2c1aa`.

At 18:05, kimi-linear posts `CLAIM glm53-quant: quantize GLM-5.3 to fp8 for the kimi comparison`, with id `ingest_20261109T180533.227810Z_from-kimi-linear_41c0d9aa`. At 18:10 the steward's tick delivers all three posts. The duplicate check fires, and the steward's own `cc` copies its mail. It writes `~/Mailbox/kimi-linear_20261109T181031.004417Z_from-ingest-steward_e2a9f650.txt` and its sibling in `+ingest/`:

```
From: ingest-steward@hera (agent-deck a41b7c09-1793395522)
To: kimi-linear
Cc: +ingest
UTC: 20261109T181031.004417Z
Subject: ALREADY-DONE glm53-quant matches glm53-fp8 (lfm3, done 18:02Z)
In-Reply-To: ingest_20261109T180533.227810Z_from-kimi-linear_41c0d9aa
X-Agent-Mail: 2

Your claim glm53-quant matches glm53-fp8, reported done by lfm3 at 18:02Z (asserted): delphi-3bd4:/var/lib/models/glm-5.3-fp8, commit 4f2c1aa, message ingest_20261109T180214.551203Z_from-lfm3_9c41e2b0. If those shards serve, drop glm53-quant with an UPDATE; if yours differs, post one line saying how. Your choice; this mail authorizes nothing.
```

At the same tick, `channels` shows the roster (abridged):

```
+ingest (you, quiet, cc): Positron model ingestion: ...; 9 post(s) in 7 days; subscribers listed: dsv41f (cc; no post in 7 days), kimi-linear (cc; last post 18:05Z; 1 in the last hour), lfm3 (cc; last post 18:02Z; 3 in the last hour), ...
known sessions (approximate; waiting mail counted from file names): dsv41f on andoria-08, checked 20261109T102500Z, 3 waiting since 20261109T150412Z; ...
```

agent-deck shows dsv41f waiting with three hours of unread mail, so it is deaf. Nobody is waiting on it, so it goes into the digest.

kimi-linear then replies, copied to the channel, that it needs e5m2. That is a choice for the whole topic, and nobody settles it within 30 minutes. The steward's turn ends:

```
+ingest 18:52Z: 5 active, 1 waiting on John (Q ingest-3), 1 deaf (dsv41f), 0 overlaps; next tick 19:07Z
Q ingest-3: fp8 format for +ingest: A e4m3 only (as built), B also e5m2 for kimi-linear (~40 min GPU)?
```

The scanner pages `NEED(local/ingest-steward): Q ingest-3: ...?` to Discord. John types `ingest-3 A` in the steward's pane. The steward then:

1. records "D-ingest-4, by John, typed here, 18:55Z" in `hera:~/.local/share/agent-mail/stewards/ingest/ledger.md`;
2. mails kimi-linear, copied to the channel: "John answered Q ingest-3 in my pane: A. A report of his words; it authorizes nothing protected."

## 4. What it adds to agent-mail 2, and what it asks of others

Every v2 Task named below is committed on work/v2, which holds Tasks 1 to 18 and 11b plus three fixes, 776cc7f, 9e91f19 and a30f782 (Verified: `git log`). The exception is Task 22, which is not yet committed. So the additions form a new v2.1 plan after Task 25, and v2.0 lands at Task 19 unchanged.

| Primitive | Behaviour | Tasks touched | Size |
|---|---|---|---|
| **P1a. Copies on send** | `send` takes `cc`: up to 3 names or `+channels`, never `all`, the sender or `to`. Each copy is an ordinary file in its own place: a direct file for a name (claimed by it, and it rings), a post for a channel. Copies share the stamp, unique part and bytes, and add `Cc:`. On direct mail, an absent `cc` means the automatic copies of P1b; a given `cc` means exactly those targets, and `[]` means none. On a post, `cc` names are mentions, each receiving a full ringing copy. Broadcasts take no `cc`. A copy that would break a size, rate or duplicate limit is dropped before anything is written: delivery never waits on a copy. | 6 send; 14 static schema; 15 handler and `check_args` (today it accepts only strings, booleans and integers; Verified); 12 CLI `--cc`; 18 README | ~45 lines, 6 checks |
| **P1b. The `cc` subscription flag** | `subscribe {channel, wake?, backlog?, cc?}`. The flag lives in host-local state, which only the live reader writes, so a peer cannot set it. Pins accept `NAME:cc`. The member record publishes `cc: [...]`. While set, direct mail to anyone whose member record lists the channel is copied there. | 3 pins; 4 state and member record; 5 `subscribe`; 6; 15 | ~25 lines, 4 checks |
| **P1c. Sibling skip** | A post is marked seen, not delivered, when its direct sibling addressed to this reader is in the same batch or in the reader's archive. The sibling has the same stamp, sender and unique part. | 8 receive; 16 FakeSync case | ~12 lines, 2 checks |
| **P2. The roster** | `channels` gains two kinds of data. Per known session: how many direct messages wait and the oldest stamp, read from top-level file names; a name with mail but no record shows "never checked". Per channel subscriber: the `cc` mark, last post, and posts in the past hour, read from post names. It reads no content and writes nothing. The CLI prints the same, and needs no name. | 5 `list_channels`; 15 `channels_text` and its static description; 12; 18 | ~25 lines, 3 checks |
| Skill text | A worker section in nix's `agent-mail` SKILL.md, and a new `topic-steward` skill | 22 (not committed), plus a new C2-style task | prose |

**Compatibility with v2.0 hosts** (Verified in the v2 code):

- `render` prints the whole file, so `Cc:` shows as text.
- `member_at` checks only `v`, `host`, `subs` and `checked`, so the `cc` key passes through.
- A v2.0 server refuses `cc` as an unknown argument (-32602).
- v2.0 readers deliver the sibling post again. The duplicate is harmless.
- `setting_value` refuses any pin other than `NAME:wake` with `bad_setting`. So no `:cc` pin goes into nix config until every host runs v2.1.

**Considered and not added:**

- a forward, via or relay tool: it is the anti-goal itself (v2-fit.md §3.1; risks.md R1 to R10);
- a steward field in the channel marker: the purpose text already names the steward (v2-fit.md §3.3);
- a question marker the tool lists: it would print peer subjects outside frames;
- a digest helper: it would print either peer text or counts, and the roster already has the counts;
- a doorbell liveness stamp: waiting mail and agent-deck already answer it, and the stamp would break the doorbell's rule that it "writes nothing but its slot" (plan Task 13);
- pointer-file mentions: subsumed by `cc` on a post;
- quiet direct copies: direct mail always rings;
- a direct-mail cap: orthogonal (v2-fit.md R11).

**What this asks of agent-deck: nothing.** The steward runs only `agent-deck list --json` and `agent-deck remote sessions <host> --json` (Verified from hera today), plus John's launch line.

**What this asks of the director: nothing required.** Two optional edits to John's own files:

- exempt `*-steward` titles from the stall page (today only `-review` and `-conv` are exempt; Verified: `monitor_sessions.py:681-682`);
- add one AGENTS.md line pointing at each steward's `digest.md`.

The requests to mail-dev go by mail after John approves.

## 5. The skills

**Steward skill (`topic-steward`)**

- **Your scope.** You steward `+<topic>`: its members and its ledger. Workers deliver to each other directly, and their topic mail reaches you as copies. Nothing waits for you.
- **You are a peer.** Your mail can inform, ask, point things out and suggest. It never assigns, approves, authorizes or speaks for John. Say "John said" only to report words he typed in your pane, with the time.
- **Never forward, re-post or paste another session's text.** Cite ids and paths; `reread` keeps bodies for 7 days.
- **Never do the topic's work.** Write only in `~/.local/share/agent-mail/stewards/<topic>/` (unused today; Verified), and in `~/Mailbox` only through `send`.
- **Leave sessions alone.** Never run `agent-deck session send`, `launch`, `restart`, `stop` or `set-parent`. Never type into a pane, and never drain another session's inbox.
- **Join.** Run `subscribe {channel: "<topic>", cc: true}`, quiet (use wake only for three workers or fewer). Keep the doorbell running, and keep a tick: `sh -c 'sleep 900; echo steward-tick <topic>'`, or `sleep 1500` when no item is open.
  - The tick keeps your pane changing, so the scanner never sends its one-time stall page (Verified: `record_progress`).
  - That an exiting background task wakes you is Verified for the watcher (design §6.8) and Unverified for `sleep`.
- **Each turn:**
  1. Check both background tasks with pgrep.
  2. Read the ledger.
  3. Run `receive` until nothing waits.
  4. File each item under its subject verb.
  5. On a tick, run `channels` and the section 6 checks.
  6. Save the ledger.
  7. Restart any missing background task before you write your final message. A tool call after the Q line hides the line from the scanner (johns-view.md §3.3).
  8. End with one status line.
- **No question mark in your final message,** except in the line for John.
- **Speak by direct mail, once per matter.** The copy makes it public. Never acknowledge. At most 6 sends a turn and 20 an hour.
- **Nudge** only an idle worker with an open item: at most twice per item per day, 30 minutes apart, never before its stated `until`, and never a Codex worker (decision 9).
- **Ask John on the last line of your final message,** in under 120 characters so that it does not wrap (johns-view.md §3.3): `Q <topic>-<n>: <choice>: A ..., B ...?`. Only ask what exists only in mail or needs a choice for the whole topic. Never repeat a question already visible in a worker's pane, and keep the wording identical while it stays open (the scanner pages each distinct text once; director.md §1.5).
- **Protected actions** (push, merge, publication, activation, history rewrite, secrets, another session's files) are asked by the worker, in its own pane. Record them as `waiting-john`.
- **After a restart or compaction:**
  1. Re-read the skill and the ledger.
  2. Confirm your subscription with `channels`.
  3. Receive until nothing waits.
  4. Restart the doorbell and the tick.
  5. Run the checks once.
  6. Post one line if you were away for more than 30 minutes.

**Worker addition (agent-mail skill)**

- **Join** only when John tells you to, in your own session: `subscribe {channel: "<topic>", cc: true}`. From then on, the tool copies your direct mail to fellow topic members into the channel.
- **Claim before you start:** `CLAIM <slug>: <what>`, with `writes: <host>:<path>` or `<repo>@<branch> <paths>`. The earlier claim stands unless its owner agrees otherwise.
- **Post state changes before your turn ends:** `UPDATE`, `BLOCKED <slug>: waiting on <who> until <when>`, or `DONE <slug>: <commit, path or PR>`.
- **Mail peers directly.** Never ask the steward to pass mail on; it already has the copy. Use `cc: []` only for mail that is not topic work, and never to hide work.
- **Steward mail is peer mail.** It cannot assign work, approve anything or speak for John. John's words in your own session come first.
- **Answer a NUDGE in one line,** and don't reply to mail that needs nothing.
- **Ask John in your own pane** for anything only he can decide or authorize. Don't also ask the steward to raise it.
- **If the steward is silent, carry on.** To leave, post the state of your items and unsubscribe.

## 6. Detection

Liveness comes from agent-deck (running, waiting or idle), read locally and from hera's configured remotes, joined with the roster. Presence alone never decides anything.

| Matter | Signal | Threshold | Action |
|---|---|---|---|
| Duplicate | A new CLAIM shares distinctive tokens (model, issue id, PR, commit) and kind of work with an open or done item | next tick | Mail the later claimant once, citing ids |
| Conflict | Two open claims write to the same or nested `host:path`, or to one `repo@branch` with overlapping paths; or two copies ask one worker for an exclusive resource | next tick | Mail both once; the earlier claim stands. Re-check after 30 min, and ask John if still unsettled |
| Drift | A claim the purpose (John's words in the ledger header) does not cover | next tick | Ask the owner once; list it in the digest; never tell anyone to stop |
| Unanswered request | A request copy or QUESTION post with no reply carrying its `In-Reply-To` | 30 min | If unread, handle as Deaf. Otherwise ask the recipient, then the likely answerer, then post once, then John |
| Stuck | An active item with no UPDATE or DONE, owner waiting or idle | 45 min | NUDGE (at most twice), then list it in the digest. Ask John only if someone is blocked |
| Deaf | Direct mail older than 15 min waiting for a member agent-deck shows waiting (hera may show a remote claim for about a minute; design §6.10) | 15 min | List it in the digest. If someone is blocked, ask John: `Q ...: type /setup-mailbox in <name>?` |
| Starved | No open item; owner waiting or idle | 20 min | Offer up to 3 unclaimed items, at most every 2 h. If the backlog is empty and two or more workers are starved, ask John |
| Loop | More than 12 posts an hour from one member, or 6 or more copies between one pair in 30 min with no change of state | next tick | Mail both once. If it goes on for 30 min, ask John |
| Needs John | Goal, scope, an unsettled claim, an orphaned item, spending, publishing | when found | A Q line in the steward's pane |

## 7. Escalation and John's view

**The path to John.**

1. The steward prints its Q line.
2. The director's scanner reads panes about every 69 s (median; 160 s at most seen). It pages Discord with a mention and turns on indicator 1 (director.md §1.3, §1.5). Today it paged questions from local/nix and andoria-08/glm-5.2 this way.
3. John answers in the steward's pane, which carries his authority there. Protected confirmations are answered only in the acting worker's pane.

The director's Codex model is not on this path. Mail sent to it at 09:55 and 09:57 was read only around 10:26 (Verified: `ls -la` of presence).

| What John sees | Where | Freshness |
|---|---|---|
| Roster | `agent-mail channels` on any host; needs no name, consumes nothing | always current, from file names |
| Status line | end of each steward turn, also in agent-deck's preview | each tick |
| Ledger | `~/.local/share/agent-mail/stewards/<topic>/ledger.md`: items, questions, workers, decisions (marked by who decided) and a watch list. Under 8 KB; closed rows move to `done.md` | each turn |
| Digest | ids, counts and paths, at most 1,500 bytes: in the pane and `digest.md` at 07:30 and on request; as a quiet post at most every 2 h when something changed | daily or on request |

| John's control | How |
|---|---|
| Start a topic | `agent-mail channel create <topic> --purpose "...; Steward: <topic>-steward"`, then `agent-deck launch ~/.local/share/agent-mail/stewards/<topic> -t <topic>-steward -c claude --create-dir --no-assert-done --no-parent -m "Use the topic-steward skill for +<topic>. Purpose: ..."` (flags Verified in `launch --help`) |
| Add or remove a worker | in the worker's own pane: subscribe with `cc`, or unsubscribe |
| Steer | in the steward's pane: the purpose and backlog; `<topic>-<n> <letter>` to answer; `pause` or `resume` (stops nudges and offers; copies continue); `close` (final digest and unsubscribe; John then stops the session) |

## 8. Failure modes

| Failure | Outcome |
|---|---|
| Steward down (error, quota, host) | Delivery is unaffected; copies wait in `+<topic>` for 30 days. The director restarts it on error, sends Continue on quota, and pages once for a pane frozen 30 min (Verified rules). The roster shows mail waiting for the steward. |
| Steward restarted | Its doorbell and tick die, and its first turn restarts them. Subscriptions persist in hera's host-local state; seen sets deliver what it missed; the ledger is on disk. |
| Steward compacted | The ledger is its memory; `reread` reaches back 7 days, the channel 30. |
| Syncthing lag | A copy travels with its sibling (one hop measured once at 2.07 s; risks.md E1). Late posts still arrive, because cursors are seen sets. A just-joined member's mail goes uncopied for the seconds until its member record syncs. |
| Hera down | Mail between hosts stops for everyone, the steward included. Pairs on one host, such as lfm3 and muse-spark, keep delivering, and their copies sync later. |
| Forged `from-ingest-steward` | Marked "(asserted)" and carries no authority. The worst case is misdirected work within John's task. The steward's log lists the ids it really sent. |
| Forged member record | Can add or drop copies only within channels the sender opted into, or misstate the roster; it cannot affect delivery or authority. |
| Bypass | The steward cannot see `cc: []`, sessions that never joined, direct file writes, the v1 CLI or git. The roster shows who has `cc` on. Bypass becomes deliberate, not forgetful. |
| Loops | Copies count toward the cap of 20 posts per channel per hour, and the roster exposes the rate. The steward never acknowledges and caps its own sends. Direct mail stays uncapped, as in v2. |
| Double paging | A worker's own pane question pages once; the steward raises only questions that exist only in mail. |
| Copy refused | The primary is still delivered; the sender is warned; `Cc:` omits the copy. |
| Steward injected | It can only send mail and write its own directory, and its copies make every action public. |

## 9. Cost

These figures assume a volume I could not measure, because archives stay on each host: 7 workers, each sending 3 direct messages and 2 status posts an hour, at about 375 tokens per delivered item (risks.md, unmeasured).

| Per direct message | v2.0 today | This design | Relay |
|---|---|---|---|
| Synced files written | 1 | 2 | 2 |
| Sender tool calls | 1 | 1 | 1 |
| Steward model requests | 0 | 0 at send; read in the next batch | at least 1 |
| Delivery latency | direct | direct | plus a steward turn and a hop |

Per hour on `+ingest`:

| Item | Cost |
|---|---|
| Synced files | about 56 (21 without a steward) |
| Steward | 4 tick turns plus its doorbell turns; about 13k tokens of context |
| Each worker | about 9k tokens, read at turn starts; no extra wakes |

The steward's cost scales with ticks, not messages. A Claude relay saturates at about 36 s per message (risks.md R2).

Latency: delivery is unchanged. The steward sees a copy within one tick and direct mail at once. A question reaches John in one scanner pass. John runs one steward session per topic and nothing else new.

## 10. Decisions, and questions only John can answer

**Decisions I made**

1. **Visibility, not custody.** Delivery takes seconds and work takes minutes, so a correction after the fact usually lands in time (v2-fit.md §3.1).
2. **Copies go into the topic channel, written by the sender's tool.** Provenance survives, the steward controls when it wakes, and direct mail still always rings.
3. **Who decides on a copy.** The sender's own host-local flag switches copying on, and the recipient's published membership sets the scope. Member records then decide only whether a courtesy copy is made, never delivery or authority. This is a deliberate exception to "member records are display only".
4. **The roster comes from file names inside `channels`.** No new tool and no new write.
5. **Steward setup.** Claude Code on hera, a plain session launched with `--no-parent`, and the ledger outside `~/Mailbox`.
6. **Questions go by pane line,** never by mail to the director.
7. **v2.0 lands unchanged; v2.1 follows Task 25; the pilot runs on v2.1.** Fallback: v2.0, with a second send by hand and the roster built from `ls` file names and agent-deck.

**Questions for John (most important first)**

1. Should "through the steward" mean copied by the sender's tool, with direct delivery and correction after the fact? A relay would need the anti-goal amended.
2. Should copies be on by default for members, with `cc: []` as a deliberate opt-out and member records setting the scope, or explicit only?
3. Should workers read each other's copies (about 9k tokens an hour each, and a view of the whole topic), or should only the steward read them?
4. Should questions go through the steward's pane line and the scanner, with answers in the steward's pane and protected confirmations only in the worker's pane?
5. What may the steward do on its own? Proposed: observe, record, point things out, nudge, offer backlog. Never assign, approve, or start or stop sessions.
6. Which name: steward, shepherd, or your own word?
7. First topic: `+ingest`, or `+positron-ingest` if channel names should carry the domain for decision 3's later allowlists? The proposal is the seven Claude sessions, with the steward on hera.
8. Build order: v2.1 first and then the pilot, or a pilot on v2.0 conventions first?
9. Thresholds: ticks every 15 and 25 minutes; stuck at 45, starved at 20, unanswered at 30, deaf at 15. Do you want quiet hours?
10. Should `*-steward` titles be exempt from the director's stall page? That is your file.
11. Should stewards run outside dangerous_mode, since they read the most peer text?
