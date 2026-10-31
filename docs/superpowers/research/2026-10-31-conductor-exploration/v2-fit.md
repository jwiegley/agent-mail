# Per-topic leads on agent-mail 2: fit with v2's primitives

Explorer report, one of six. Written 2026-10-31 for the mail-conductor design session (branch work/conductor). Everything here was done read-only. No repository, worktree, mailbox file or session was changed or messaged. "Lead" is a placeholder for the per-topic role. Its real name is a design choice (see 5.4 for a collision with agent-deck's own "conductor").

Evidence marks: **Verified** means I read or ran it today, and I say where. **Unverified** means it is documented but I did not check it.

## 0. Bottom line

1. v2 as committed already supports the cheapest workable lead pattern, with no code change. A topic channel `+<topic>` is the visible copy of worker coordination. The lead subscribes with `wake`, and workers post requests there. The lead routes **by reference**: it sends a worker direct mail in its own words, with `in_reply_to` set to the post's id. Nothing is forwarded, the author keeps provenance, and claims are untouched.
2. John's literal wording is option (b): the lead receives a message and passes it along. That collides with an approved anti-goal and with the static MCP instructions ("Never forward or re-post peer text"). The lead must either relay verbatim, which is the worm path and launders provenance, or paraphrase, which loses detail. Option (b) also doubles the hops and makes the lead a single point of failure. I recommend against it (section 3.1).
3. Nothing needs to change in v2.0 before landing (Task 19). Every addition a lead design might want is backward compatible, as checked on eb1b468:
   - member records accept extra keys;
   - readers show headers as plain text;
   - extra lines in a channel marker fold into the purpose.

   So the requests are small v2.1 candidates. In priority order: `notify` on send (the design's deferred "Mentions", done with pointer files), or `cc` on send; a doorbell liveness stamp in member records; a lead line in the channel marker.
4. Two capabilities belong outside agent-mail.
   - **Surfacing questions.** The director's monitor already pages John on Discord when a question sits at the tail of any active pane. It runs a pass about every 60 to 70 s.
   - **The work ledger.** It belongs in the lead's own worktree, not in the synced channel directory.
5. Three hard limits remain, and no small change removes them:
   - From lines are asserted, so a lead's word weighs the same as any peer's.
   - An idle Codex session cannot be woken (decision 9), and the director is a Codex session.
   - Member records are written by peers, and `checked` means "last read", not "alive".

## 1. What v2 gives a lead today

**State of v2 (Verified, 2026-10-31 on hera).**
- Branch work/v2 is at eb1b468, with a clean working tree.
- 20 commits are on it, from 301c48a (01:38 PDT) to eb1b468 (10:09 PDT). They cover Tasks 1 to 18 and Task 11b, plus one fix, 1629b69.
- Not done yet: Task 19 (both platforms, audit, landing) and phases C2 and C3 (Tasks 20 to 25).
- I made a scratch copy of `agent-mail` whose SHA-256 matches the worktree file (cad6a088...). Its `python3 -I -B ./agent-mail self-test` passed with 335 checks, with HOME redirected to the scratchpad.

**Primitives** (Verified by reading the v2 script at eb1b468; line numbers refer to that file):

| Primitive | What a lead can rely on |
|---|---|
| Direct mail | Claimed by its single reader, archived 7 days in `.<name>-read/`. It always rings the doorbell (`ringable`, l.1969). |
| Posts | Live in `+<ch>/<ch>_<UTC>_from-<from>_<hex8>.txt`, at most 8 KiB, never claimed. Each reader keeps its own seen set and an archive copy of each delivered post. A reader's own posts are skipped (l.1152), so a sender never receives or archives what it posted. |
| Subscriptions | Quiet or wake. At most 20 per session. `backlog` (at most 20) applies only on the first subscribe; subscribing again changes only `wake` (l.838). Pinned channels come from config and no tool can remove them. Runtime subscriptions live in host-local state (l.826, l.481). |
| Headers | A fixed set: From, To, UTC, Subject, optional In-Reply-To (exactly one id, grammar-checked), and X-Agent-Mail: 2 (l.950). No other header passes through `send`. Readers print the whole file inside the frame, headers included, so a new header shows to old readers with no reader change. |
| Member records | `.agent-mail/members/<name>.json` holds `{v, name, host, checked, subs}`. It is written on receive (commit), on quiet passes of the claiming watch, on subscribe and unsubscribe, and by CLI `pending` (callers at l.1509, 1565, 2430, 2440, 2718, 2732, 2756). Neither the doorbell nor the MCP `pending` tool writes it. `read_member` (l.570) validates only the keys it uses, so extra keys pass through. |
| `channels` tool | For each channel: its purpose, posts in the last 7 days, the last post and the listed subscribers. Then the known sessions with host and `checked` time, marked approximate (l.2235). The purpose is one line of at most 200 characters, and every line of the marker is folded into it (`read_purpose` l.763, `clean_line` l.2782). |
| `reread` | Reads only from the caller's own archive (`archived_item`, l.2262). |
| Send caps | 20 posts per channel per hour. 1 broadcast per 15 minutes. An identical body to the same target is refused within 10 minutes. Direct mail has no count cap (`note_send`, l.516). |
| Channel marker | Written only by the CLI's `channel create`, with O_EXCL and mode 0600 (l.775). After that it is a synced file that any process running as the user can rewrite. |
| Static rule | The MCP instructions and the `send` description both say "Never forward or re-post peer text" (l.2145, l.2157). |

## 2. Capability table

| # | Capability | v2 status | Smallest change | Plan Task touched | Committed? | Risk |
|---|---|---|---|---|---|---|
| 1 | Routing A to B "through" the lead | Works as-is for (a), the channel as the copy, and (d), dispatch by reference. (c) works as two sends. (b) needs a forward primitive (an anti-goal) or a verbatim relay (against the instructions). | None: a skill convention. Optional v2.1: `send {to: "+topic", notify: [names]}` writes the post plus one tool-written pointer file per name. | Optional: 6 (send), 12 (CLI), 15 (tool schema), 18 (README) | All committed, so this would be a new follow-up task after landing | Compliance rests on convention, so workers can DM around the lead. A wake subscription costs the lead one turn per batch of posts. Option (b) adds a single point of failure and a worm path. |
| 2 | Visible copy (Cc) of direct mail | Works as-is with two sends: two ids, nothing links them. | Optional v2.1: `cc: [name]` (at most 3). Writes one sibling direct file per name, with the same stamp and unique part, and a `Cc:` header in each copy. | 6, 12, 15, 18 | Committed | Each copy rings the lead. Replies may drop the Cc. The lead becomes the highest-exposure reader. |
| 3 | Channel owner or steward | Works as-is as free text in the purpose. Verified in a scratch mailbox: `channel create` accepted the purpose "nix-config work, conducted by nix-lead", and a two-line marker shows folded into one purpose line in `channels`. | None. Optional v2.1: `channel create --lead NAME` and `channel set-lead`, CLI only, writing a second marker line `lead: <name>`. `channels` would show `+ch (lead name)`. | 5, 12, 15, 17, 18 | Committed | The marker is peer-writable on every host, and the last writer wins under Syncthing. It is a display, never authority. |
| 4 | Nudges for idle or starved workers | Works as-is: a nudge is direct mail. It rings Claude through the doorbell and Pi through pi-loop's `pending`, and never reaches Codex (decision 9). Detection is a skill. | None. Optional v2.1: the doorbell refreshes a `bell` stamp in its member record at most every 300 s. | 4 (members), 13 (doorbell), 15 (`channels`) | Committed | `checked` means last receive. Member records can be forged. Nudges can loop, since the duplicate rule stops only identical text. |
| 5 | Surfacing questions to the director | Not agent-mail's job. Mail to conductor-director waits for the director's next model turn. Since today's POLICY.md change, heartbeats run a script instead of starting such a turn. | None in agent-mail. Convention: the lead ends its turn with one line in its own words that contains a question mark, and the director's monitor pages John. For the record, it also mails conductor-director with a subject starting `question:`. | Skill text in 22 | Not committed (C2) | Mail cannot carry John's answer back with authority. The Discord page carries the lead's paraphrase. |
| 6 | Digests | Works as-is: a post to `+topic` (at most 8 KiB) or direct mail to the director (at most 1 MiB). | None. Optional v2.1: the sender keeps a copy of its own posts in its archive, so `reread` can show its digests. | 6, 7 | Committed | A digest that quotes peer text is a re-post. Digests should hold the lead's own words plus message ids. |
| 7 | Work ledger | Not in v2's model. A file in `+topic/` is tolerated. A non-hidden one is a doctor finding on every run on every host, and receive reports it once per reader per host. A hidden one is silent everywhere and never pruned (Verified, section 3.7). | None: keep the ledger in the lead's worktree and publish digests as posts. | None (a doc line in 18 or 19, request R8) | n/a | A ledger in the channel is a synced, multi-writer, mutable file. One side's edit is lost silently under `maxConflicts=0`, any session can tamper with it, and workers read it outside the frames. |
| 8 | Identity of the lead | No, by design: From lines are asserted (a non-goal). | None. Skill: the lead's mail is peer mail, and the lead mirrors its decisions on the channel so a forged "lead" message can be cross-checked. | Skill text in 22 | Not committed | Forged lead messages can misdirect work. They cannot authorize anything. |
| 9 | Membership of a topic | Partial: subscriptions in member records, display only and self-asserted. | None. The roster lives in the lead's ledger, and a durable lead pins its topic in config. | 22 (config.json) plus a John decision | Not committed; decision 26 says nothing is pinned | Cannot tell an offline host from an idle or departed session. Runtime subscriptions are lost when a session is re-created on another personal host. |
| 10 | Retention and history | Works as-is: posts kept 30 days, archive copies 7 days, a 24-hour replay after lost state. A restart on the same host keeps runtime subscriptions. | None. Optional v2.1: a `history {channel, last}` tool. | Optional: 14, 15 | Committed | The real loss on a restart is the lead's model context. Direct Cc copies vanish after 7 days. |

## 3. Notes per capability

### 3.1 Routing a worker's message to another worker "through" the lead

| | (a) Channel is the copy | (b) Lead relays | (c) Cc in one send | (d) Dispatch by reference |
|---|---|---|---|---|
| How, in v2 | A posts on `+topic` with B named in the subject; B and the lead subscribe | A mails the lead with "for B"; the lead re-sends to B | A sends to B and to the lead | A posts on `+topic`. The lead, on a wake subscription, decides and mails B its own words with `in_reply_to: <post id>`. |
| Code change | None | Verbatim relay breaks the instructions; a provenance-keeping relay needs a forward tool (an anti-goal) | None (two sends) or `cc` (R5) | None (optionally `notify`, R4) |
| Lead sees it | Yes | Yes | Yes | Yes |
| B is woken | Only on a wake subscription, which rings for every post. Otherwise at B's next turn. | Yes, by the lead's direct mail | Yes, direct mail rings | Yes, by the lead's direct mail. A's post arrives in the same batch if B subscribes. |
| Lead can stop it before B acts | No | Yes | No | Only if the skill says workers wait for dispatch ("gated") |
| Provenance | From A | From the lead, so A is laundered | From A | A's text from A; the routing decision from the lead |
| Claims | None (posts are unclaimed) | The lead claims A's file; B never sees it | Each copy is claimed by its own reader | B claims only the dispatch |
| Latency | One hop | Two hops plus the lead's turn | One hop | One hop when ungated; plus the lead's turn when gated |
| Single point of failure | No | Yes | No | Only when gated. A fallback such as "act after N minutes of the lead's silence" removes it. |
| Fit with the anti-goals | Fits | Collides | Fits | Fits |

**Weighing (b) honestly.** Option (b) gives the lead one thing the others do not: it can intercept a request before B acts on it. Delivery takes seconds and B's work takes minutes, though, so a correction sent after the fact ("hold: C already has this") nearly always lands before much work is lost. Against that gain, (b) has these costs:

1. If the lead copies A's text, it breaks the static instruction "Never forward or re-post peer text" (l.2145). The research behind that rule found self-replicating prompts spreading fastest through relays (Prompt Infection, design section 2). The lead is the ideal relay: it reads everything and writes to everyone.
2. If the lead paraphrases instead, technical details are lost (paths, hashes, exact errors), and an injected instruction can survive the paraphrase.
3. Either way, B sees the request as coming from the lead. That raises its effective weight, against non-goal NG1.
4. A's file is claimed by the lead. If the lead's session dies or compacts before it re-sends, the request lives only in the lead's 7-day archive and in its intent. That is message state outside the files, against anti-goal AG4.
5. Latency doubles, and every topic stops while the lead is down or busy.

A provenance-keeping forward tool, `send {to, forward_of: <id>}`, would remove costs 2 and 3. But it is exactly the "no tool that forwards or re-posts" anti-goal, which John approved. Only he can amend it.

**Recommendation from the v2 side.** Use (d) over (a). The channel holds the text once, and the lead dispatches by reference. Whether workers wait for the lead's dispatch (gated) or act at once (watcher) is John's call (question 1). The design's own Phase 6 item "Mentions" (design section 9) is the natural v2.1 primitive here (R4). If it writes a pointer file per named session, routing stays on file names, as design 6.3 requires ("Readers route only on a file's location and name, never on its headers").

### 3.2 A visible copy (Cc) of direct mail

- **Today:** two sends. The duplicate rule does not fire, because the target differs (`note_send` keys on target and body hash). Costs: two tool calls, two unrelated ids, and copies that can differ by mistake.
- **A `Cc:` header plus a second file (R5)** is cheaper and safer than a `Via:` header.
  - A `Via: <lead>` header on mail to the lead that names the final recipient is option (b): readers never route on headers, so only the lead's model re-sending could deliver it.
  - With Cc, the sender writes both files itself. Each is ordinary direct mail at the top level, named for its own reader, so claims stay one reader per file.
  - B's claim does not affect the lead's copy, and vice versa.
  - Both copies show `Cc: <the other recipients>` with no reader change, because frames print the headers.
  - Sharing the stamp and unique part makes the ids siblings, so the lead can cite B's id.
- **Cost of Cc:** direct mail always rings, so a lead copied on every message is woken for every one. A quiet Cc would be a new primitive (a direct file that does not ring) and goes against "direct mail always rings". A channel subscription lets the lead choose wake or quiet, and that is a reason to prefer posts for routine traffic.

### 3.3 A channel owner or steward

- **Today:** the marker holds one purpose line. Every line of the marker is folded into one, capped at 200 characters.
- **Verified in a scratch mailbox:** a two-line marker "nix-config work / owner: nix-lead" appears in `channels` as `+nix-work: nix-config work owner: nix-lead; 0 post(s) in 7 days; ...`. Writing "lead: <name>" into the purpose at `channel create` therefore works now, with zero code.
- **A structured lead line (R7)** would show as `+nix-config (lead nix-lead): <purpose>; ...`. It changes nothing else.
- **Who may change it:** anyone with a shell as the user, on any Mailbox host, can rewrite `.agent-mail-channel`.
  - The marker is a hidden synced file, and no `.stignore` line matches it.
  - Syncthing carries the newest write. Under `maxConflicts=0`, concurrent writes lose one side silently. That is Unverified on this fleet; design section 11 lists it as unmeasured.
  - agent-mail writes the marker only at creation. A `set-lead` should stay a CLI admin command, like `channel create`. An MCP tool for it would "edit configuration" (AG2).
  - A sturdier home for the mapping is nix-generated `config.json`. Design 6.5 says "No policy setting lives in the synced folder". But `load_settings` rejects unknown keys with `bad_config`, so the code must learn any new key before nix can write it.

### 3.4 Nudges for idle or starved workers

**What the lead can see:**
1. `channels`: subscribers per channel, and each known session's host and `checked` time.
   - `checked` moves only when that session reads (receive or commit), subscribes, runs CLI `pending` (pi-loop does every 30 s), or completes a quiet pass of a claiming watch. The record is rewritten at most every 300 s.
   - The doorbell never moves it. So a healthy Claude session that is idle on the v2 doorbell looks the same as a dead one after an hour.
2. Warnings on `send`, when the recipient's mail was last checked over an hour ago.
3. Outside agent-mail (Verified on hera with read-only commands):
   - `agent-deck list --json` gives 49 local sessions with `status`, `last_activity_at` and `conductor_name`. Today: 27 stopped, 12 waiting, 7 running, 3 idle; 38 Claude, 10 Pi, 1 Codex.
   - Remotes pallas, vulcan, andoria-08, athena, clio and delphi-3bd4 are configured, so `agent-deck remote sessions --json` reaches them from hera. I did not run it, and I did not check whether other hosts have remotes configured.
   - `agent-deck session children --json` and `--follow` report children's status and completion sentinels. This works only for sessions whose agent-deck parent is the lead. I did not check whether a parent link can cross hosts.

**What the lead can do:**
- Direct mail. It rings a Claude session through its background watcher or doorbell; that exit-wake is Verified as being in daily use, per design 6.8. It reaches Pi at the next 30-second `pending` loop, and Codex at its next turn only.
- No new primitive is needed: starvation and stuck detection is a skill. It compares the lead's ledger of assignments with `checked`, agent-deck status and the topic's recent posts.
- The one gap is liveness. R6 (a `bell` stamp written by the doorbell) closes it for sessions on the same mailbox, on any host.
- The lead must never nudge with `agent-deck session send`. Section 5 of the design forbids relaying peer requests that way, and a nudge built from peer text would be a user-role prompt carrying peer bytes (AG3).

### 3.5 Surfacing questions to the director

What the director can actually take in (Verified by reading `/Users/johnw/.local/share/agent-deck/conductor/director/{AGENTS.md,POLICY.md,heartbeat.sh,monitor_sessions.py}` and the tail of task-log.md):

1. **Its Codex model reads agent-mail only at turn start.**
   - Until yesterday the model also had heartbeat turns that read mail. The task log has "Heartbeat mailbox coordination 2026-10-30 16:07 PDT", where it consumed peer mail.
   - POLICY.md, as modified at 09:48 PDT today, says heartbeats run `monitor_sessions.py` directly, "rather than sending input to this Codex TUI", and heartbeat.sh execs it.
   - The latest log entries are script passes, labelled "Scheduled heartbeat", about every 68 s; the last I saw was 10:18:17 PDT.
   - agent-deck's conductor record for director shows `heartbeat_enabled: true`, `heartbeat_interval: 0`. I did not check whether agent-deck still starts heartbeat turns.

   So mail to conductor-director waits for the director's next model turn. That is John's turn, and possibly a heartbeat turn.
2. **Its monitor script already pages for questions.**
   - Each pass scans every active local pane and every pane on a configured remote.
   - It looks at the last 15 lines after the last assistant bullet. A line there containing "?", other than code, URLs and a long list of exclusions, becomes a Discord message `NEED(<where>): <question>` with John mentioned, deduplicated per session and text.
   - The same pass turns on Hera's `indicator 1`.
   - The scan does not look for `NEED:` prefixes; that prefix is agent-deck's parent-wake convention.
   - Titles ending `-review` page only on blocking words.
   - `-review` and `-conv` titles are exempt from the 30-minute stall page.
   - I did not observe a Discord delivery today.
3. **The script drains the director's agent-deck inbox on each pass and only logs it.** Inbox items, such as a child's completion sentinel, are not sent to Discord.

How the options compare:
- A subject prefix (`question:`), a header (`X-Question: yes`) or a `+questions` channel the director subscribes to all change nothing about **when** the director reads, because that is turn start.
  - A header is not routing in v2 at all.
  - A `+questions` channel adds a durable record that other leads can see, and John could read it in the director's next turn.
- **A tool is not needed.** The existing path costs nothing: the lead ends its turn with one line in its own words that contains a question mark and names the worker and the message id. The monitor pages John within about a minute, provided the lead is an active local or configured-remote agent-deck session and its title does not end in `-review`.
- `agent-deck conductor notify --tier urgent|info "<text>"` exists and "queues a message for the human (bridge forwards urgent now, info as a digest)". This is Verified in `agent-deck conductor notify --help`, v1.16.34. Whether it works for a session that is not an agent-deck conductor, and whether the bridge delivers today, is Unverified.

A consequence for authority: if John answers in the lead's pane and the lead mails the answer on, the worker receives peer mail, which cannot carry John's approval. Anything needing authority must be answered in the worker's own pane, with the lead pointing John there (question 3).

### 3.6 Digests

- **To the topic:** `send {to: "+topic"}`, up to 8 KiB, rate-capped at 20 per hour. Subscribers get it; the lead does not, because its own posts are skipped.
- **To the director:** direct mail up to 1 MiB, read at the director's next turn.
- **To John:** not through agent-mail. `john` and its variants are reserved (design 6.3), and John speaks to sessions through their harnesses (design decision 15). His view is the lead's pane, a file in the lead's worktree, or the paths in sections 3.5 and 3.4.
- **What v2 already covers:**
  - Ids in a digest's body are plain text, which is enough. A `References:` list or a `Thread:` header is not needed; threads beyond In-Reply-To are a non-goal (NG3).
  - `reread {id}` re-renders anything the lead received in the last 7 days.
- **The gap:** a lead cannot `reread` its own digests or dispatches, because `send` keeps no copy and own posts are never delivered (R12, optional). It can keep them in its worktree instead.

### 3.7 A work ledger

Verified on a scratch copy of the v2 script, in a scratch mailbox with HOME redirected; only `init`, `channel create`, `channels` and `doctor` were run:

- **A non-hidden `ledger.md` in `+nix-work/`** gives `doctor` this line: `finding: not named as a post: +nix-work/ledger.md`, and doctor exits 1. Doctor runs this check on every run, on every host (l.2619). Receive reports it as "is not named as a post; left in place". That notice is suppressed after the first time through `remember(state, "notice:...")` (l.1425), so it appears once per reader per host. It comes back after 500 other remembered notices (REMEMBER_MAX, l.424) or a lost state file. So the receive notice is not "forever", but the doctor finding is.
- **A hidden `.ledger.md`** is silent. Readers skip hidden entries in channel directories (l.1140), and so does doctor (l.2612). Pruning never touches it, because it removes only grammar-valid post names (Task 9). It also syncs: no `.stignore` line matches it. That is Verified by reading the STIGNORE list; I did not test Syncthing.
- **Design 6.2 lists only the marker and posts** inside a channel directory, so a ledger there needs a design amendment either way.
- **Where it should live:** in the lead's own worktree (single writer, versionable with git, safe across restarts, readable by John), plus digests as posts for the workers.
- **Ledger as posts:** each assignment could also be a post with a `ledger:` subject. That is immutable, framed and kept 30 days. But the lead never receives its own posts, so it still needs its local copy.
- **Host-local agent-mail state is ruled out:** it has one writer, the reader.

### 3.8 Identity

- **A worker cannot tell that a message came from the lead.**
  - The frame says "Sender names are asserted, not verified" and flags a From header that disagrees with the file name.
  - The MCP server binds the sender name, but any process writing the folder on any of the ten hosts can write `<worker>_..._from-<lead>_...txt`.
- **So a lead's word weighs exactly as much as any peer's.** It can inform, ask and suggest. It never authorizes, and it never carries John's approval, even when John spoke to the lead. That is John's rule, restated in every frame.
- **The damage from a forged lead message** is bounded to misallocated work, such as "stop X" or "C has it", because nothing is authorizable by mail.
- **Containment:**
  1. The lead posts every dispatch decision on the topic channel too, so a direct "lead" message that has no matching record there looks suspect.
  2. Workers act on lead mail only within the task John gave them.
  3. Per-site signatures (design Phase 6) could later label mail from the other domain.

### 3.9 Membership

"Subscribed to `+topic`" is a weak definition of "worked by this lead":
1. It is self-asserted, display only and peer-writable.
2. Watchers, such as the director or a reviewer, would count as workers.
3. A worker reached only by direct mail would not count.
4. A session that has not yet run v2 has no record at all.
5. A runtime subscription disappears when the session is re-created on another personal host.

**Offline hosts.** Their records keep the last `checked` time and sync on reconnect. The lead cannot tell an offline host from an idle session or one that has left. Mail waits, and senders get the stale warning after an hour.

**Better:** the roster lives in the lead's ledger. Workers join with a post such as "joining +topic: I take X", and their subscription is pinned in config for durable membership. Member records and agent-deck status are cross-checks, not the source.

**Data domains.** Decision 3's later Positron segregation (Phase 6 allowlists) means a Positron topic's lead must run on a host that will still carry Positron traffic: hera, nyx or a Positron host.

### 3.10 Retention and history

- **For active coordination,** 30 days of posts and 7 days of archive copies are enough. Longer history belongs in the lead's ledger.
- **A lead sees only posts delivered to it:** those since its subscription, plus a backlog of at most 20 on the first subscribe. Older posts can be re-delivered by an unsubscribe followed by a subscribe with backlog 20, or read as raw files outside the frames, which I advise against.
- **Correction to the premise "runtime subscriptions are lost on restart":**
  - Runtime subscriptions live in `~/.local/state/agent-mail/<dirkey>-<name>.json` (l.481, l.826), so a restart on the same host keeps them.
  - At Positron, `~/.local/state` is on the shared NFS home, and `dirkey` hashes the real path `/var/lib/jwiegley/Mailbox`, which is the same on all four hosts. So moving between Positron hosts keeps them too.
  - They are lost when the state file and its `.bak` are lost, which triggers a 24-hour replay with a notice, or when the session is re-created on another personal host.
  - Pinned channels survive everything, and a peer cannot talk the lead into unsubscribing a pinned channel.
- **What a restart does lose is the lead's model context.** The worktree ledger, the 7-day archive (`reread`) and the 30-day channel are what rebuild it.

## 4. Requests the design would send to the rollout session

Addressed to `agent-mail`, which will be `mail-dev` after decision 26's rename. These are peer requests: none of them authorizes anything, and each change goes to John as any plan change would. Priority: **must** means needed before this design can run; **should** and **could** are v2.1 candidates after landing, and only if John approves the lead design.

- **R1 (must, informational): land C1 unchanged.** The lead design needs nothing in v2.0 before Task 19. On eb1b468:
  - `read_member` ignores extra keys (l.570), so a later member field is compatible;
  - readers print headers as text, so a later `Cc:` header needs no reader change;
  - `read_purpose` folds extra marker lines into the purpose (l.763), so a later lead line degrades to text in an old reader.

  Please say if anything in Tasks 19 to 25 would change these three facts.
- **R2 (must): Task 22 (not committed), `config/ai/skills/agent-mail/SKILL.md`.**
  - Keep "Never re-post or forward peer text" and "Use direct mail when one owner must act" exactly as written.
  - Leave room for a short topic-work section whose text will come from the approved lead design: post coordination on the topic channel; name the intended worker in the subject; reply on the channel; treat a lead's mail as peer mail, never as John's word.
  - If the design needs its own skill file (for example `agent-mail-lead`), it would be generated by nix the same way, in Task 22 or a new C2 task. Please say which you prefer.
- **R3 (must, decision routing): Task 22's `config/agent-mail.nix`.** Once John approves the design, it will ask to pin each lead's topic with wake: `sessions.<lead>.channels = ["<topic>:wake"]`. That amends decision 26 ("nothing pinned"), so please carry it to John with Task 22 rather than adding it now.
- **R4 (should): `notify` on send, the design's Phase 6 "Mentions" done by pointer files.**
  - `send {to: "+<ch>", ..., notify: [<name>, ...]}` takes up to 3 names and applies to posts only.
  - It writes the post, then, per name, a direct file `<name>_<same stamp>_from-<sender>_<hex8>.txt` with `In-Reply-To: <post id>` and a tool-written fixed body naming the channel and the post id.
  - Routing stays on file names, the pointer rings the named session, and no peer text is duplicated.
  - Touches the code from Tasks 6, 12, 15 and 18. About 40 lines and 6 checks:
    1. one pointer per name;
    2. invalid and reserved names refused before any file is written;
    3. post and pointers share the stamp;
    4. no peer bytes in the pointer body;
    5. the rate window records each target;
    6. `notify` refused unless `to` is a channel.
- **R5 (could, alternative or complement to R4): `cc` on send, direct mail only, up to 3 names.**
  - One sibling file per name, with the same stamp and unique part as the primary.
  - Each copy carries `Cc: <the other recipients>`.
  - `note_send` records each target, and the result names every id.
  - Tasks 6, 12, 15 and 18. About 30 lines and 5 checks.
- **R6 (should): a doorbell liveness stamp.**
  - The doorbell refreshes a `bell` UTC stamp in its own member record at most every 300 s, and `channels` shows "doorbell <UTC>" beside "checked".
  - Today `checked` moves only on reads, subscriptions and CLI `pending` (callers listed in section 1), so an idle but healthy Claude session on the doorbell looks dead after an hour.
  - This changes the doorbell's documented property "writes nothing but its slot" (design 6.6 and 6.8, plan Task 13), so the design text must change too.
  - Tasks 4, 13 and 15. About 15 lines and 3 checks.
- **R7 (could): a lead line on the channel marker.**
  - `channel create --lead NAME` and `channel set-lead NAME|--none`, CLI only. They store a second marker line `lead: <name>`.
  - `channels` shows `+<ch> (lead <name>)`, and doctor reports a lead name that fails the grammar.
  - Display only.
  - Tasks 5, 12, 15, 17 and 18. About 30 lines and 4 checks.
  - Until then, the lead is named in the purpose text, which works today (Verified, section 3.3).
- **R8 (should, docs, small enough for Task 19's audit pass): say what a channel directory may hold besides posts and the marker.** On eb1b468:
  - readers and doctor skip every hidden entry (l.1140, l.2612);
  - a non-hidden non-post file is reported once per reader per host by receive, and is a doctor finding on every run on every host (l.2619, exit 1);
  - pruning touches neither.

  Either document this, or have doctor also list hidden files other than the marker. Otherwise a hidden synced file in a channel is invisible to every check.
- **R9 (could, only if the director's monitor adopts it): `pending NAME --format human` stops writing the member record.** Keep the write for `--format loop`, which is pi-loop's presence (`cmd_pending`, l.2750, Task 13). Otherwise a monitor that only peeks at waiting mail makes conductor-director look as if it read its mail, and senders lose the stale-recipient warning.
- **R10 (should, plan note): topic channels outlive their topics.** v2 has no retire or delete. A deleted channel directory can also be held on a host where an ignored `.<file>.tmp` sits inside it (the vulcan audit's note in the handoff). Please record under "Not in this plan" that topic channels are kept, and that retiring one means John rewriting its purpose, not deleting the directory.
- **R11 (could): a per-session hourly cap on direct mail** in config, `sessions.<name>.direct_per_hour`, unset by default. v2 caps posts and broadcasts but not direct mail (l.516), so an injected lead could mail every worker repeatedly as long as each body differs. Tasks 3, 4 and 6. The code must reach a host before nix writes the key: `read_config` rejects unknown keys at every level with `bad_config` (l.267). The MCP server would then answer every tool except `channels` with that error.
- **R12 (could): `send` keeps the sender's own copy of each post in its archive**, so `reread` can show a lead its own digests and dispatches. Today a sender never receives or archives its own post (l.1152). Tasks 6 and 7.

## 5. Collisions with v2

### 5.1 Goals, non-goals and anti-goals (design section 4)

| v2 item | Collision | Real, or dissolves? |
|---|---|---|
| Goal: any session reaches any other | Routing "through the lead" as an enforced rule would narrow direct reach. | Dissolves when routing is a convention, as in (a), (c) and (d). Only an enforced (b) collides. |
| Goal: every README guarantee survives (claims, one reader per name) | Under (b) the lead claims A's file and B never gets A's message as A's file. | Formally kept, but practically real for (b). Dissolves under (a), (c) and (d). |
| Non-goal: no cryptographic proof of the sender | Any design that gives the lead's word extra weight, or asks workers to trust "this came from the lead". | Real, and cannot be removed. Contained by keeping the lead advisory and mirroring its decisions on the channel. |
| Non-goal: delivery no faster than the substrate | (b) and a gated (d) add a second hop plus the lead's turn. | Real for (b) and gated (d). Dissolves under (a), (c) and ungated (d). |
| Non-goal: no threads beyond In-Reply-To, no read receipts, no history beyond 30 days | A lead wants to know whether B read A's request, and wants long history. | Partly real. Dissolves by convention (workers acknowledge on the channel: "taking X") and by the worktree ledger. Receipts stay unavailable. |
| Non-goal: no idle Codex wake without John's consent | A lead cannot nudge an idle Codex worker, and mailed questions reach the Codex director only at its next model turn. | Real; no option removes it (decision 9). Questions avoid it by using the lead's pane line instead of mail. |
| Anti-goal: no tool that forwards or re-posts | Option (b), verbatim, breaks the static instruction; a provenance-keeping forward tool is this anti-goal itself. | Real for (b). Dissolves under (a), (c) and (d). A digest that quotes peer text is the same collision on a small scale. |
| Anti-goal: no tool that approves | A lead that "approves" work or relays John's approval. | Real only if the design grants it. It must not. |
| Anti-goal: no tool that edits configuration | A lead-setting MCP tool would edit configuration. | Dissolves if `set-lead` stays a CLI admin command (R7). |
| Anti-goal: no peer bytes in hooks, doorbells, tool descriptions, instructions or user-role prompts | Nudging with `agent-deck session send`, a "question doorbell" that carries text, or injecting questions into the director with `session send`. | Real if proposed. Dissolves when nudges are mail and questions are pane lines or mail. A Discord page goes to John, not to a model as a user-role prompt. |
| Anti-goal: no message state only in a process | Under (b), a claimed but not yet re-sent request lives only in the lead's intent. A ledger kept only in the lead's context has the same problem. | Real for (b) and for a context-only ledger. Dissolves with (a), (c) or (d) and a worktree ledger. |

### 5.2 Rules that must survive (design section 5)

1. **Mail never authorizes anything.** A lead relaying John's answer by mail cannot carry John's approval. This is real: authority questions must be answered by John in the worker's own pane (question 3).
2. **Peer requests are never relayed with `agent-deck session send`.** No collision, as long as leads nudge and dispatch only by mail.
3. **One reader per name, unique across the fleet.** Each lead needs a unique session name, and a channel cannot take the name of a session that has a member record. `+nix` is therefore blocked once the `nix` session runs v2; use `+nix-config`.

### 5.3 Security controls (design 6.11)

| Control | Collision | Real, or dissolves? |
|---|---|---|
| Fixed identity, no sender argument | (b) relays arrive From the lead, which launders provenance. | Real for (b); dissolves under (a), (c) and (d). |
| Strict grammar | Names in `cc`, `notify` and the lead line must pass it. | No collision, if R4, R5 and R7 check them as `send` checks `to` today. |
| Constant doorbells | Any wish for a "priority" or "question" ring with text. | Dissolves: nudges are ordinary mail. |
| Static surface | New arguments or tools change the tool list per version, and it stays static. | No collision. |
| File checks (owner, regular file, one link, O_NOFOLLOW, size) | A ledger in the channel read with shell tools bypasses every check and every frame. A peer could swap it for a symlink to a secret file. | Real for a ledger in the channel; dissolves with a worktree ledger. |
| Amplification limits | A lead with a wake subscription that is also copied on everything is the highest-exposure reader, and an uncapped sender of direct mail. | Real. Contained by the Rule of Two (the lead never pushes, activates or edits others' files on the strength of mail), by own-words-only output, and optionally by R11. |
| Residual risk: a forged member record misleads `channels` | It misleads starvation detection: a dead worker looks alive, or a live one looks gone. | Real. Cross-check with agent-deck status. |
| Rule of Two | The lead reads the most mail of any session. | Fits only if the lead's actions on mail are limited to sending mail and updating its own ledger. |

### 5.4 Naming and neighbouring systems (facts the design should know)

1. **"Conductor" is already an agent-deck term.** `agent-deck conductor` manages "named conductor sessions for meta-agent orchestration. Multiple conductors can exist per profile". It offers `setup`, `notify`, `outbox` and `heartbeat-tick`, and the session records carry a `conductor_name` field (Verified with `--help` and `agent-deck list --json`). One conductor is configured today: `director` (Codex).
2. **Reserved names** include `owner`, `admin`, `operator` and `system`.
3. **Channel names are one level only** (decision 6), so `+positron-ingest` rather than `+positron/ingest`.
4. **The director's monitor treats `-review` titles specially**: their questions page only on blocking words. It also exempts `-review` and `-conv` titles from the stall page. A lead's title should avoid both suffixes, or the director's policy should name leads explicitly; that is John's file.

## 6. Implications for agent-mail

1. **No v2.0 change.** Task 19 can land as planned, and every lead-related addition is backward compatible (R1).
2. **The lead pattern is a skill and conventions over existing primitives:** a topic channel, a pinned wake subscription for the lead, quiet subscriptions for workers, and dispatch by reference with `in_reply_to`. The skill lands with Task 22, which is not committed.
3. **The cost of ringing grows with traffic.** Each wake batch costs the lead a turn. v2's receive budget (12,000 bytes by default, 24,000 at most) bounds one receive, not the lead's context growth, so the lead needs periodic digests and a ledger to survive compaction.
4. **Leads should run on Claude Code or Pi,** which can be woken, and never on Codex (decision 9).
5. **Pinning a lead's topic in config** both survives re-creation and stops a peer from talking the lead into unsubscribing. It needs John to amend decision 26.
6. **agent-mail needs no question primitive.** The lead's own pane line already reaches John through the director's monitor. Mail to the director is a record, not an alert. If John wants mailed questions to page him, that is a director-side change, and R9 then matters.
7. **The work ledger stays out of `~/Mailbox`.** If it is ever wanted in the channel, the design needs an amendment and a doctor rule first (R8).
8. **v2.1 candidates in order:** R4 `notify` (or R5 `cc`), R6 `bell`, R7 lead line. Optional: R11 direct cap, R12 sent copy, and a `history` tool. Each touches code from committed tasks, so each is a new follow-up task after landing.
9. **Documentation gaps** to close regardless: what may sit in a channel directory (R8), and the lifecycle of topic channels (R10).
10. **The lead cannot see direct mail it is not copied on.** "Most traffic goes through the lead" holds only as far as workers follow the convention. Under one Unix user nothing can enforce it, and v2 should not try.

## 7. Open questions for John (most important first)

1. **Gate or watcher?** Should workers wait for the lead's dispatch before acting on a topic request? That lets the lead stop duplicated or conflicting work before it starts, at the cost of latency and a single point of failure. Or should workers act at once while the lead watches and corrects?
2. **Visible or relayed?** Do you accept "through the lead" meaning "visible to the lead" (the channel copy, or a Cc), rather than "relayed by the lead"? A true relay conflicts with the approved anti-goal and the instruction "Never forward or re-post peer text". Only you can amend those.
3. **Where do you answer?** When a lead surfaces a worker's question, do you answer in the worker's own pane, as anything needing authority requires, with the lead only pointing you there? Or may the lead pass on answers that need no authority, as peer information?
4. **Where should questions reach you?** The choices:
   - the lead's pane line, which the director's monitor already pages to Discord;
   - `agent-deck conductor notify` (urgent at once, info in a digest);
   - mail to conductor-director, read only when you open the director;
   - or a director change that pages you when mail waits for it.
5. **Pinning.** May each lead's topic channel be pinned with wake in nix config? That amends decision 26.
6. **Form of a lead.** Should leads be agent-deck named conductors, which brings the notify outbox and heartbeats, or plain sessions? And what is the role called, given that agent-deck already uses "conductor"?
7. **First topics.** Which topics come first, under what channel names (`+nix-config`, since `+nix` is blocked; `+positron-ingest`), and on which host does the Positron lead run, given decision 3's later segregation?
8. **A lead's powers.** Does a lead get anything beyond mail and its own ledger? For example, read access to workers' worktrees to detect duplication. Push access should never be given.
9. **Codex workers.** Accept that a lead cannot wake an idle Codex worker (decision 9), or allow a user-shaped nudge for leads only?
10. **v2.1 additions.** Do you want any v2.1 additions at all (R4 to R7), or only the zero-code conventions for a first trial?

## 8. Evidence log

**Verified today (2026-10-31, on hera):**
- v2 worktree state, git log and the self-test (335 checks) on a scratch copy.
- v2 code paths cited by line number at eb1b468.
- Scratch probes of `channels` and `doctor`: a two-line marker folds into the purpose; a non-hidden `ledger.md` is a doctor finding with exit 1; a hidden `.ledger.md` is silent.
- The v2 plan's task list and "Needs John"; the design's sections 2 to 11; decisions 1 to 27; the handoff, both the worktree copy and main's 0365162 entries on the cutover and this thread.
- The director's AGENTS.md, POLICY.md, heartbeat.sh, heartbeat_watchdog.py and `monitor_sessions.py`: the question scan, Discord delivery code, inbox draining and the stall rules.
- The tail of the director's task-log, which shows script passes about every 68 s, and the director model's last logged mail-reading heartbeat turn, "Heartbeat mailbox coordination 2026-10-30 16:07 PDT".
- `agent-deck --help`, `session --help`, `session children --help`, `conductor --help`, `conductor notify --help` and `conductor heartbeat-tick --help`.
- `agent-deck list --json` field names and counts, `remote list --json` and `conductor list --json`.
- Names in `~/Mailbox` and its presence directory: 13 presence names including `mail-conductor`; no `+` channels and no `.agent-mail/` yet.

**Unverified or not checked:**
- Syncthing's behaviour under `maxConflicts=0` on this fleet, and how it names a conflict copy of a hidden file.
- Whether `agent-deck conductor notify` works from a session that is not a conductor, and whether the bridge delivers today.
- What `heartbeat_interval: 0` does for director.
- Whether agent-deck parent links can cross hosts.
- `agent-deck remote sessions` output.
- What Claude Code 2.1.312 shows in a pane while a background doorbell runs, which decides whether the director's monitor counts the lead as live work.
- Whether Discord pages were delivered today.
