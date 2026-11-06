# Constraints on a metadata send log (agent-mail 2.4, decision 90)

- **Written:** 2026-11-05, 19:4x to 20:1x PST, by the read-only constraints scout of the traffic-log design workflow, for the designer of rollout decision 90's metadata send log and the steward's use of it.
- **Read:** the conductors design (`spec`, both the live tree and the a3 worktree copy, sections 1 to 13 as needed, closely 3, 5.4, 6.11, 8, 11 and 12); the live `README.md`; the program `agent-mail` 2.3.0 by grep and line reads; `decisions.md` entries 1, 2, 4, 10, 17, 19, 26, 45, 57, 63, 69, 72, 80a, 82 to 90 and 86a; `agent-mail-gaps.md` whole; the code map's sections 0, 1, 6.1 and 7; nix-config's `agent-mail` and `mailbox` skills, `config/agent-mail.nix`, `config/syncthing.nix` (grep) and `test/home/agent-mail-mcp.nix`; the steward files of the a3 worktree (`CLAUDE.md`, `WORKER.md`, the helper's header and census lines); the MCP channels design and the topics design (sections 3 to 7 and 10); the exploration's `synthesis.md`; the overview report (sections 0 to 3, 8, 10 to 13) with its F1 and F6 notes in part; the steward-native proposal in part; four of the user's memory notes, named where cited.
- **Not done:** no git in any form; no agent-mail run; no agent-mail MCP tool; no agent-deck command. Under `~/Mailbox` only `ls -la` of the top level (I kept the hidden and channel entries and a count of message-named entries, and recorded no message file name) and `cat .stignore`. No message file was opened. Nothing was written except this file.
- **What this context held:** the workflow brief, the user's memory index (`MEMORY.md`), and what I read; no parent transcript, no mail, no compaction.

**Anchors.** `spec:N` is `docs/superpowers/specs/2026-10-31-agent-mail-conductors-design.md`; lines 1 to 619 are identical in the live tree `/Users/johnw/src/sys/agent-mail` and the worktree `/Users/johnw/Products/agent-mail-a3`. The worktree adds decision 38 (`a3-spec:620-623`), which moves section 12 down by four lines (`spec:623-648` on main is `a3-spec:627-652`). `README:N` is the live `README.md` (identical in a3). `am:N` is the live program `agent-mail`, VERSION 2.3.0 at `am:2559` (identical in a3). `dec:N` is a line of `/Users/johnw/Products/mailbox-rollout/decisions.md`; "decision N" is its entry number. `skill:N` is `/Users/johnw/src/sys/nix/config/ai/skills/agent-mail/SKILL.md`. `CLAUDE.md:N`, `WORKER.md:N`, `signals:N` are `/Users/johnw/Products/agent-mail-a3/steward/` files (every line cited here is the same on main; main differs at CLAUDE.md 250 and 423 and the helper's docs checks near 5709 only). `gaps:N` is `/Users/johnw/Products/steward-native-design/notes/agent-mail-gaps.md` (byte-identical to the vault copy). `map:N` is `/Users/johnw/Documents/Obsidian/agent-mail-overview-design-20261104/00-code-map.md`, whose own `agent-mail:N` numbers are 2.2.1's. `overview:N` is `/Users/johnw/Documents/Obsidian/20261104-agent-mail-overview-design -- agent-mail.md`; `F1:N` and `F6:N` its notes `F1-fleet-table.md` and `F6-handoff-pointer.md`. `proposal:N` is the vault copy `/Users/johnw/Documents/Obsidian/20261105-steward-native-design -- agent-mail.md` (two lines longer than the `~/Products/steward-native-design/proposal.md` working copy from line 5 on). `mcp-spec:N` is `docs/superpowers/specs/2026-10-30-agent-mail-mcp-channels-design.md`, `topics:N` is `docs/superpowers/specs/2026-11-02-mailbox-topics-design.md`, `synthesis:N` is `docs/superpowers/research/2026-10-31-conductor-exploration/synthesis.md`, `handoff-rollout:N` is `docs/superpowers/handoff-mailbox-rollout.md` (all in the live tree). `nix-syncthing:N`, `nix-amconfig:N`, `nix-mcptest:N` are `config/syncthing.nix`, `config/agent-mail.nix` and `test/home/agent-mail-mcp.nix` in `/Users/johnw/src/sys/nix`. "Memory note X" is `~/.config/claude/positron/projects/-Users-johnw-src-sys-agent-mail/memory/X.md`, a record I did not re-verify.

## 0. The short list

1. **Decision 1 keeps its routing and loses two clauses.** Copy, not custody, no relay, and "nothing waits for the steward" all survive. A metadata log reverses "nothing changes in agent-mail 2 for the pilot" outright, and, for metadata, reverses the NOTE convention's role as the only way the steward sees direct mail between others. It also widens decision 9 (file names to subjects) and, if a steward reads beyond its own members, decision 32 (section 1).
2. **Privacy.** Today a direct mail's subject is read by its recipient alone and kept only on the recipient's host for 7 days; its file name (recipient, sender, time, id) is visible in the shared folder only until it is claimed, which takes seconds for a Claude reader. The log makes every subject, with the who-mails-whom graph, durable and present on all ten hosts. Any process of John's user on any host can read it. Nothing can enforce a reader (section 2).
3. **What must survive:** bodies never; no relay and no re-post of peer text; mail authorizes nothing, and neither does a log line; one reader per name; nobody reads another session's mail; the skill's hidden-file rule, or the skill changes too; delivery never waits on the log; no peer bytes in the doorbell, the instructions or a steward's pane (section 3).
4. **Release:** the full program path, with John's word for every push, unit and switch. A new config key forces two nix units. A new MCP tool reopens MCP decision 7 and nix's tool pin. Hosts run mixed versions for most of a day, and sessions keep their old servers until they restart, so the log is a lower bound for days (section 4).
5. **Fleet:** ten hosts in a star through hera. The four Positron hosts share an NFS home but keep four separate local copies of the Mailbox. `maxConflicts` is 0, so a shared file must be single-writer or write-once. agent-mail allows 10 minutes of clock skew (`MARGIN`) (section 5).
6. **Prior art verdicts:** F1 fleet table, do not build yet; F6 handoff pointer, the convention now and the pointer later in a release of its own; 6A self-attestation, keep SENT.log and the census now, and add 6A later, on decision 5b's signal or with a release made for other reasons. The exploration's S3 copies were to come only if NOTE coverage fell below about 80% (section 6).

## 1. Decision 1, and the part a metadata log reverses

### 1.1 What decision 1 says

> 1. **Routing:** copy, not custody. Peers mail each other directly and post NOTEs; state changes are posts; the steward sees everything and never carries a message; nothing changes in agent-mail 2 for the pilot. (spec:568)

The same choice elsewhere in the design: "A steward sees a copy and never carries the mail ... Nothing waits for it; if it dies, delivery continues." (spec:12); "agent-mail 2 is not changed for the pilot" (spec:14); "visibility is the goal and custody was the proposed means; the exploration showed custody costs more than it gives, and John chose visibility (decision 1)" (spec:32); anti-goal "No change to agent-mail 2 for the pilot" (spec:59). John's request of 2026-10-31 had asked for a conductor per topic "which sees all traffic" (handoff-rollout:56). Decision 1 met that request with posts and NOTEs, not with all traffic.

### 1.2 What "the steward sees everything" means in the built system

- Every post on its channel, through a wake subscription, and every direct mail sent to it (spec:79-80, 158).
- Direct mail between two other sessions only through: the NOTE a worker posts after mailing a peer, `direct: <message id>` (WORKER.md:17, rule 4 at WORKER.md:28; "this is what makes peer-to-peer traffic visible", spec:137); the `mailed: <n>` field of UPDATE (WORKER.md:13); and the helper's census of file names (spec:373 "Names only, never contents"; signals:22; CLAUDE.md:28), joined against `NOTES.log` (CLAUDE.md:119, :362) as `transit-without-note`, "a lower bound" (CLAUDE.md:209; spec:463; signals:604-637), because "delivered mail leaves the shared folder within seconds" (spec:25, :79). A direct mail with no NOTE that is claimed between two census passes is invisible (spec:495).
- Posts on other channels and broadcasts are peers' notices: one log line each, never filed (decision 32, spec:609; spec:334; CLAUDE.md:108; rollout decision 72, dec:118).

### 1.3 Clause by clause

| Clause of spec:568 | Under a metadata log | Basis |
|---|---|---|
| "copy, not custody" | **kept**: a tool makes the copy and it is never a hop; no message waits for the log or for the steward | spec:12, :21-22, :48; CLAUDE.md:11; WORKER.md:31 |
| "Peers mail each other directly and post NOTEs" | direct mail **kept**; the NOTE's role as the **only** source of the steward's view of direct traffic is **reversed for metadata**. Whether WORKER.md rule 4 stays (the NOTE ties a mail to a slug, which a subject need not carry) is the design's to say. Decision 90: "The pilot runs on the NOTE convention meanwhile." | WORKER.md:17, :28; dec:176 |
| "state changes are posts" | **kept** | spec:568 |
| "the steward sees everything" | **widened**, from posts, NOTEs and names seen in transit to every send's time, sender, recipient, id, subject and replied-to id | spec:137, :373, :463; dec:176 |
| "never carries a message" | **kept** | spec:55, :110 |
| "nothing changes in agent-mail 2 for the pilot" | **reversed**: 2.4 changes agent-mail while the pilot runs. The stewards started on 2026-11-05 without day zero (dec:169), and the two-week review is Phase 3 (spec:562). The steward-native proposal already reads this clause as a freeze that any agent-mail release reverses (proposal:21, :250, :479) | spec:14, :59 |

Decision 90 says "it reverses design decision 1 for metadata when built" (dec:176), and decision 38 repeats it in the worktree spec: "The all-traffic overview John asked for is to be designed as a metadata send log in agent-mail (rollout decision 90), which would reverse decision 1 for metadata when built." (a3-spec:623). Neither text says which clause. The table above is my reading.

### 1.4 What else in the conductors design the log moves

- **Decision 9 and rule 3.** "census of file names" (spec:578); "Never read, move or delete another session's mail; listing file names is allowed to the stewards **[John 9]**" (spec:101); "Names only, never contents" (spec:373); "the helper lists file names and metadata for you, and you may too" (CLAUDE.md:28). Under these texts a subject is message text, not metadata: "never a message's text" (signals:22), "reads member records as metadata, never a message body" (spec:364). F1's reviewers made the same point: CLAUDE.md:28 "grants metadata only" (F1:1183). Reading subjects is therefore a **new grant**, not covered by decision 9.
- **Decision 32 and rollout decision 72.** A steward files only its own channel's posts and the mail sent to it (spec:609; dec:118). John's words, as the brief quotes them, ask both for "traffic patterns within a set of topics" and for "a summary of ALL mail traffic, posts to +fleet, posts to any channel, and direct mail between sessions and between hosts". The second form amends 32 and 72. The first can be met by filtering the log to the topic's members.
- **Anti-goal spec:58:** "No ledger, decision log or state in `~/Mailbox`, where any session on any host can write." The send log is agent-mail's, not the steward's, but its content is exactly that kind of forgeable shared state. The steward keeps its ledger in its own directory and treats a log line as asserted data.
- **Decision 12** (spec:584; section 8 item 4 at spec:525; revisit row at spec:641, a3-spec:645): the "cannot authorize" text change waits for evidence that relays are used and useful. The five pinned strings (spec:85) stay unchanged in 2.4 unless that row's signal has fired.
- **Section 8 item 1** (spec:522): the steward relies on six behaviours, all intact in 2.2.1 (map:150): `read_member` ignoring extra keys, readers printing headers as text, `read_purpose` folding extra marker lines, readers skipping posts in their own name, `send` making the id, and the body-only duplicate hash. A 2.4 must keep all six.
- **Section 12, row 1** (spec:629, a3-spec:633). Signal: "a conflict causes damage that holding the message for minutes would have prevented; or member mail seen in transit without a NOTE stays high for a week (a lower bound, so judged with the NOTE count)". Alternative: "gated claims on shared paths; tool-written Cc copies (the exploration's S3)". Neither signal has been measured, since no pilot week has passed. **The reversal comes from John's decision 90, not from the row.** The row needs amending to say so, and the log needs rows of its own, saying what would withdraw it. Row 32 ("`+fleet` traffic crowds the steward's turns or its day's log", a3-spec:649) and row 9 (nagging, a3-spec:643) are the rows an all-traffic feed is most likely to trip, against the self-caps of 6 sends a turn and 20 an hour (CLAUDE.md:35; spec:368).

### 1.5 The earlier answers to the same question

- 2026-11-04 21:03 PST, to the director: "no facility, none planned; WORKER.md's NOTE convention is the boundary; a sender-side record is a 2.3 design for John" (decision 82, dec:157).
- 2026-11-05 22:11Z, to conductor-ares-steward: "no such feature exists in any release, by design decision 1" (decision 84, dec:169).
- 2026-11-05 18:1x PST: decision 90, "design now, build after approval" (dec:176).
- John's two quoted sentences in the brief appear in none of the records I read; dec:169 and dec:176 record the requests by time only.

## 2. The privacy change, in plain terms

### 2.1 Today

- A direct mail is one file, `<to>_<UTC>_from-<from>_<unique>.txt` (README:21-28), at the top level (README:57). Its **name** gives the recipient, the sender (asserted, README:48), the sender's clock and the id. Its **subject** is a header line inside the file (README:32-39).
- Syncthing carries every top-level file to all ten hosts (README:183, :187). Until the mail is claimed, the whole file, subject and body, sits on every host's disk. By rule, only its one reader opens it (README:46; skill:38-41). Other sessions may at most list names, and only the stewards are told they may (spec:101; CLAUDE.md:28).
- The reader claims a mail by renaming it to a hidden name (README:166), which `.stignore` keeps local, so the name leaves the shared folder: within seconds of a Claude doorbell on the reader's host (spec:79), and about a minute later on the others (spec:65; mcp-spec:434). Unclaimed mail waits up to 14 days (README:71). The reader's copy stays 7 days in `.<you>-read/`, on its own host only (README:72, :189).
- So today **the subject is read by the recipient alone and kept only on the recipient's host for 7 days**. The file name (sender, recipient, time) is visible fleet-wide only while the mail waits. Even the sender keeps nothing: "you cannot re-read your own posts" (WORKER.md:36), and the sender's copy (candidate 3) is not built (gaps:14).
- agent-mail has kept senders and subjects out of every surface but the delivery:
  - `pending --format json` "holds no message id, file name, sender, subject or text" (README:239);
  - Pi's footer "shows counts only, never senders or subjects" (README:147; dec:168);
  - doctor's roster shows recipients, counts and the oldest stamp, "from names and lstat alone, opening no file" (am:3035-3039, :3085-3110; README:71);
  - `channels` shows counts and the last stamp per channel (map:267);
  - candidate 1's roster was drawn as "Names and stamps only, never ids or subjects" (gaps:37).
  The overview report: "The program has never parsed a body or kept a shared file about other sessions" (overview:938).

### 2.2 With the log

- Each send appends time, sender, recipient, id, subject and the replied-to id (dec:176). For direct mail this adds the subject and the reply link to what the name already shows. It turns all of it from transient into **durable** (for the log's retention), **fleet-wide** (the disks of all ten hosts) and **readable by design** (a sanctioned reader, where today only a forbidden open could see it).
- It records the who-mails-whom graph with times, which today exists only as the stewards' transient lower bound (spec:463).
- Posts and broadcasts are already readable by any subscriber or by every session (README:58-62). The log adds their subjects for sessions that are not subscribed.
- **Data domains.** Positron work subjects reach the personal-only hosts (Clio, Vulcan, athena, pallas; hera and nyx carry both), and personal subjects reach the four Positron hosts. F6's note describes those as "corporate machines holding the Mailbox under /var/lib/jwiegley/Mailbox" (F6:355). Decision 7b accepted mixed domains for the topic channels: "the v2 risk 'one folder mixing data domains' is accepted, not retired" (spec:482; spec:576). The exploration listed "Positron posts sit on all ten hosts for 30 days" as a pilot blocker (handoff-rollout:56). Decision 90 extends that acceptance to the subject of every direct mail and names it as such: "every direct mail's subject visible fleet-wide" (dec:176).

### 2.3 Who can read it

- **Any session on any host, and any process of John's user there.** "The trust boundary is the pair (host, OS user). Every session runs as John." That includes root on a Positron NFS client (mcp-spec:460). A file in `~/Mailbox` opens for any process of `johnw` or `jwiegley`.
- **Nothing can single out a steward.** A server's name comes from `--name`, `AGENT_MAIL_NAME` or identity files that any process of the user can set or read (README:97; gaps:144). The reader lock guards one name's mail on one host (README:101), not a file about every name. "Readable by the stewards only" can therefore be a rule in texts, never a permission.
- **The rule that keeps sessions out today is the skill's** (skill:38-39). If the log is a hidden file, that rule forbids every session, the stewards included, to open it. The read must then go through agent-mail itself, as `channels` shows peer-written hidden member records to everyone (map:142, :267), or the rule must change (section 3.4).
- **Leak paths to watch.** The digest is returned to Discord on demand (spec:413), and the director's pages go to Discord, a third-party service (overview:335, :934). The steward texts already keep subjects out: "never a peer's subject or body" in the pane (CLAUDE.md:23); question lines and digests carry "ids, counts and paths, never peer bodies" (spec:474); the summary post is one paragraph of at most 400 bytes of counts and names (CLAUDE.md:409-415). A subject taken from the log must never reach `DIGEST.md`, the summary, a question line, the pane or Discord.

### 2.4 Bounds the existing decisions put on the log's shape

- **Non-goals of agent-mail 2: "read receipts, and history beyond 30 days"** (mcp-spec:64). A log kept beyond 30 days, or one that records claims or reads, crosses them. Recording reads would also amend decision 10, "reader state stays local" (dec:12): F1 found that read stamps in member records "amend decision 10" (F1:1071-1080; overview:975).
- **"Unanswered" from sends alone is a heuristic.** A reply is a send with `In-Reply-To` (am:1137, :1145), and every send may leave it out (README:80; skill:140-153). "No reply in the log" is therefore not proof that a question went unanswered.
- **The word "metadata."** Decision 90's metadata includes the subject. Every steward text uses metadata for names and stat data (CLAUDE.md:28; signals:4-5, :22; spec:364). The design must define the word once, and amend the texts that promise "never a message's text".

## 3. Rules that must survive

### 3.1 Bodies never

- "bodies never" (dec:176). Copying bodies to stewards is rejected. The exploration's S3 wrote direct mail "byte for byte" into `+<topic>-cc` (synthesis:71), and the design did not choose it (synthesis:92, :111). "Coordinators earn their keep by verifying, not carrying" (spec:21). The steward never copies a body into its ledger or a reply (CLAUDE.md:106).
- In the program, an entry can come from `send`'s own arguments: the subject is cleaned and cut to 200 characters at am:1138 and the header is built at am:1144-1145, so nothing needs the file's text.

### 3.2 No relay, no re-post

- "Never forward or re-post peer text" (skill:32-33; README:64; the server's instructions at am:2561-2564). "The steward never forwards, quotes or re-posts peer text. It writes its own one-line summaries and gives message ids." (spec:55; CLAUDE.md:19).
- MCP anti-goal: "No tool that executes, approves, forwards, re-posts or edits configuration." (mcp-spec:70). A read path that hands other sessions' subjects to any caller must be argued against that line: the sender's own `send` wrote the subject, and a read returns it inside a frame as data, never posted again.
- A steward never quotes a subject from the log in its mail, posts, digest or pane (CLAUDE.md:19, :23).

### 3.3 Mail authorizes nothing, and neither does the log

- README:48-49; skill:24-29; spec:99-100 (rules 1 and 2); "nothing in `~/Mailbox` is read back as authority" (spec:474).
- The relay exception rests on four checks: the worker's own open question, the steward's name in the file name and the From line, a DECISION post on the channel stamped within 10 minutes, and `protected: yes` (WORKER.md:46-51; decision 11, spec:583). A log line must never stand in for any of them.
- Sender names are asserted. The CLI's `send` takes any `--from` (spec:82; am:3329), so a log written by `send` records the asserted sender, and a forger using the CLI writes the file and its log line together. **The log adds no authenticity** (section 6.3).

### 3.4 One reader per name; nobody reads another session's mail; the hidden-file rule

- "Each name has exactly one reader. Don't run a second reader for the same name." (README:46; README:191; skill:40-41; mcp-spec:105). The log claims nothing and moves nothing, changes no seen set, and needs no name's reader lock to be written.
- "Never read, move or delete another session's mail, or any hidden file in the mailbox that is not yours." (skill:38-39). "Read only through agent-mail, so that your name keeps its one reader." (skill:174-175). "Files whose names begin with `.` are never mail. They are temporary files, claims, archive copies, and other tools' state." (README:45). Rule 3 of the design (spec:101).
- **Where a log file can sit, given what doctor and the readers do:**
  - non-hidden at the top level: a finding on every host and every version, "legacy or nonstandard name at the top level, never mail" (am:3078-3084);
  - non-hidden in a channel directory: the doctor finding "not named as a post" (am:3129), a notice from each reader, "is not named as a post; left in place" (am:1381-1384), and a break with README:58's "nothing else" (also spec:524);
  - hidden, at the top level, in a channel or under `.agent-mail/`: invisible to every 2.x reader and to doctor (am:1276, :1373-1374, :3082, :3122). But then skill:38-39 forbids every session to open it.
- **So, either** the stewards read the log only through agent-mail (a CLI subcommand or a tool), which honours skill:174-175 and grows the steward's closed list at CLAUDE.md:25-31, **or** skill:38-39 is amended in nix-config (a nix unit, then every harness restarted). Either way, CLAUDE.md:28 ("read and write in `<dir>` alone ... never read, move or delete another session's mail; the helper lists file names and metadata for you") and CLAUDE.md:22 ("never open a path a message names except through `reread`") are reworded on the light path.
- F1 met the same wall and listed three ways out (F1:1110-1122; overview:1044): runners outside the mail rules (John's terminal, a job that is not a session); an exception in the runner's own rules, "the form of CLAUDE.md:28", which "grants metadata only"; or amending skill:38-39 and the read-only-through-agent-mail lines, a nix unit. Whether a `stat` or a listing counts as a "read" under skill:38-39 was left "not established" (overview:177).
- **The helper's literal property:** "This helper never opens a message file, never writes to the mailbox and never sends anything, and it reads metadata only, with one exception: the probe" (signals:4-5); "The census lists names and ctimes only ... never a message's text" (signals:22); both pinned by tests (`test_listing_opens_no_file`, `test_signals_opens_no_message_file`, overview:100). A helper that reads subjects changes the property and its pins.

### 3.5 Nothing waits, and delivery never depends on the log

- "Nothing waits for it; if it dies, delivery continues." (spec:12; CLAUDE.md:11; WORKER.md:31).
- The program already follows this for its own record of a send: "The message is already in place, so failing the send now would invite a second copy." (am:612-613, the late warning). A log append that fails must neither fail nor repeat the send. A refused send (paused, invalid name, too large, looks like a secret, rate-limited, duplicate; am:1106-1153) must log nothing.
- The secret backstop checks subject and body before the write (am:1149-1150), so the log carries only subjects that passed it, and the backstop is "a backstop, not permission" (README:51; skill:31).

### 3.6 The steward's own lines

- "Never echo peer text into your pane. Print ids, counts, slugs and your own words, never a peer's subject or body" (CLAUDE.md:23). The scanner reads every pane and types `Continue` or `/compact` when certain strings appear anywhere in it (spec:95). F1 needed a whole defence for its own output: it searches what it prints with the scanner's four signature names (overview:277-289).
- "it reads its own mail only (no subagent receives)" (spec:474); "Never hand mail to a subagent or a workflow" (CLAUDE.md:21); "A subagent never calls `receive` or `reread`" (skill:62-63).
- "The steward reads more peer text than any session" (spec:478). The log adds every subject in the fleet, each at most 200 characters with control characters removed (am:1138; README:37).
- **Paging stays where it is.** A steward reaches John only by its pane question line (decision 4, spec:571; decision 89, dec:175). agent-mail cannot page (gaps:29). The scanner "reads nothing under `~/Mailbox`" (decision 87, dec:173; a3-spec:623), so a scanner rule over the log would reverse decision 87.

### 3.7 MCP and program invariants

- "No peer-controlled bytes in a hook output, doorbell line, tool description, instructions field or user-role prompt." (mcp-spec:71); "Constant doorbells and hook outputs" and a "Static surface" (mcp-spec:469-470). The doorbell writes nothing but its slot and the log (spec:83; gaps:22).
- Seven tools, static and byte-stable (MCP decision 7, mcp-spec:637; map:190). nix pins the seven names (nix-mcptest:83-91, compared exactly at :109-111). The instructions text is 472 characters against a self-test bound of 512 (map:19, :200).
- No listener, socket or daemon. One stdlib-only file that runs on Python 3.9.6 (mcp-spec:69; map:47-51; gaps:30; topics:47).

### 3.8 Shared-file discipline

- "Every shared file today is write-once (messages, markers) or single-writer (member records)." (map:18); "a shared file must stay single-writer or write-once" (gaps:28); "a rewritable shared file loses writes under `maxConflicts = 0`" (overview:970). The Mailbox folder has `versioning = null` and `maxConflicts = 0` (nix-syncthing:298-301; decision 57, dec:88; MCP decision 10). "Of two concurrent edits of one file, one is lost without a trace" (map:18). The CLI can send as any name from any host (am:3329), so even a per-sender file has more than one possible writer unless it is also per host, or each entry is its own file.
- Anything new under `~/Mailbox` that no ignore line matches reaches all ten hosts (gaps:28; map:156). The live `.stignore` holds exactly the six lines (`.*-claimed-*`, `.*-read`, `.*-rejected-*`, `.*.tmp`, `(?d).nfs*`, `(?d).DS_Store`, read at 19:4x PST); nix writes them (nix-syncthing:82-84); the self-test pins them (am:6739-6742); doctor checks each one (am:3069-3077). A new ignore line is therefore a nix syncthing change as well. A name ending `.tmp` already stays local (map:156).
- agent-mail refuses a mailbox in which `.agent-mail`, `.agent-mail/members`, `.agent-mail-presence` or `.<you>-read` is a symlink or not a directory (README:172; am:642-699). A new log directory joins that list.
- **No room in existing records.** Every older writer drops a member record's extra key (gaps:21; overview:116-117). Older servers drop a send-window entry with a fourth element (gaps:27). Neither can carry log data.
- **Pruning.** Member records are never pruned (map:22), so a log needs its own prune rule. The precedent is direct mail: every reader prunes every name but its own, and the window is set only at the config's top level "because every reader prunes the other names' mail" (README:71, :106; am:270-273). A fast clock prunes early (mcp-spec:497).

## 4. Release constraints

### 4.1 The program path

- **Two release paths** (map:20): "A change to the program goes through a fresh audit, a nix-config pin unit through nix's queue, and a switch on each of ten hosts ... A change confined to `steward/` and `docs/superpowers/` with the program blob unchanged is pushed to GitHub and the five fleet checkouts under John's standing rule, decision 69, with no nix unit and no switch." A skill-text change is a third kind: a nix unit with activations but no pin (overview:155).
- **The program flow** (map:630-641): a worktree under `~/Products`, since peers run the live tree (topics:51); VERSION bumped, which moves the `serverInfo` pin (map:623); the self-test on hera's python3 3.14, on `/usr/bin/python3` 3.9.6 and on a Linux host from a copied file; a fresh reviewer, a sentinel probe and a fresh fess audit; an ff-only landing; pushes to GitHub and to the five checkouts; a nix-config pin unit (`sources/tools.json` rev, narHash and version; about 16 gate items; one signed commit through nix's queue); activations (Hera by pi or nix; Clio by John; Vulcan, athena, pallas and nyx through their `/etc/nixos` drivers; the four Positron hosts through andoria-08's home-manager, with the generation built on each host first); "Sessions see the new server only after their harness restarts."
- **Time:** "14.8 to 34.8 hours from landing to the last host"; "no first audit has come back clean"; 2 to 3 fix rounds (overview:156).
- **Approval:** "John approves every push, switch and activation" (decision 4, dec:6). Decision 69, as extended by 85, covers only fast-forwards that touch `steward/` and `docs/superpowers/` with the program blob unchanged; "anything touching the program or other paths comes back to John" (dec:112, :171).

### 4.2 A new config key forces two nix units

- **Mechanism.** `read_config` refuses an unknown key at any level (am:303-329, the refusal at am:318-319; README:106 "An unknown key is an error."). A refused file fails every CLI command, and every MCP server that loads it answers every tool with the error; a server loads its settings once, at start (am:3002, :2817-2823; gaps:26). nix writes one file for every home: "Every home gets the same file ... agent-mail refuses a key it does not know" (nix-amconfig:1-19).
- **Rule:** "A key is written to `config.json` only once every host runs the version that knows it" (map:189; topics:49; spec:528). So unit 1 pins the program and every host switches; then unit 2 writes the key and changes nix's exact config pin (nix-mcptest:158-165), and every host switches again.
- **Escapes.** "Defaults need no key" (topics:49; map:189). Candidate 4's constant form needed no two-phase rollout (gaps:114, :122). F6's pointer: "no config key, so no two-phase rollout" (overview:893).
- **Decision 26:** "the first channel is +fleet (fleet-wide notices), nothing pinned, subscriptions quiet by default" (dec:31). Per-session config (`sessions.<name>.*`) exists, but nothing is pinned, and pinning the stewards' channels was already noted as amending decision 26 "for the stewards only" (spec:531). A per-steward key for the log, naming its readers for example, would be both a new key (two units) and a departure from decision 26's empty config.
- **Decision 45**, the precedent for a retention rule: "readers prune unclaimed direct mail after a set age (default 14 days, configurable) ... A v2.1 unit, developed in a worktree, audited, landed on main and re-pinned." (dec:62). Its key, `retention.direct_days`, lives only at the top level (README:106).

### 4.3 A new tool, argument or command

- An eighth MCP tool "reopens decision 7 (MCP spec:637), changes nix's tool pin ..., and costs every session the tool's definition" (F6:404; the pin is now at nix-mcptest:83-91). The skill names seven tools (skill:13-15).
- A new argument sent to an older server is JSON-RPC error -32602 (map:148).
- A new CLI subcommand must appear in the docstring's Commands section, which `section_help` checks; README section 9 is kept by hand (map:622). A steward that runs it through Bash puts its output in the pane (section 3.6).

### 4.4 Mixed versions, for most of a day per host, and per session until restart

- **Hosts.** 2.2.1 took hera at 04:28 PST on 2026-11-04 and the last four hosts at 17:51, by John's own switches (memory note fleet-agent-mail-version-check; map:645). 2.2.2 took hera at 22:40 PST on 11-04 and Clio at 00:04 on 11-05, and left Vulcan and pallas on 2.2.1 (dec:163-167). 2.3.0 ran from 01:32 to 12:15 PST on 11-05 (memory notes fleet-agent-mail-version-check and pi-agent-mail-extension; gaps:31, written earlier, still lists Clio, Vulcan and pallas as John's to switch).
- **Sessions.** "A session sees new server behaviour only after its harness restarts" (gaps:30; map:190, :641); "Your MCP server keeps the version it started with" (skill:56-58). At 13:26 PST on 11-05 no running Pi session had yet loaded the 2.3.0 extension (memory note pi-agent-mail-extension). Which version the three stewards' servers run is "not established" (gaps:31, :203); hera ran 2.3.0 from 01:32 PST, so it is probably 2.3.0, which I did not verify.
- **Consequence for the log.** Until every host runs 2.4 and every session has restarted, a send from an older host, or through an older server, writes no entry. Like the census, the log is a lower bound: "no entry" must read as unknown, never as not sent or unanswered. gaps:71 used the same pattern for a member-record key: "absence must read as 'unknown', never 'dead'".
- **Older readers must ignore the log silently.** A hidden name is skipped by every 2.x reader and by doctor (am:1373, :3082, :3122); a non-hidden one is a finding (am:3084, :3129). A host rolled back to 2.3 stops writing and reading entries, and nothing it reads may break: both directions stay compatible, as for 2.1 and 2.2 state (map:191; overview:125).

### 4.5 Self-test and audit

- **Mechanics** (map:618-624): `t.ok` checks in the section where their neighbours live; each section gets a fresh HOME; time is injected with `t.at` and constants patched with `t.patch`; a VERSION change moves the `serverInfo` pin; a new subcommand or option must pass `section_help`. 578 checks at 2.2.2 (overview:158).
- **Where log checks land** (gaps:52, :79, :99, :167): `section_send` (written, refused, failed append), `section_sync` (FakeSync and its `.stignore` model, am:6558-6565, :6739-6742: what crosses, what stays local, a single writer), `section_retention`, `section_tools` and `section_help`, `section_doctor` (no new finding on any host).
- **Hosts.** Run on three interpreters, copying the program to a temporary directory on the Linux host (memory note agent-mail-self-test-needs-a-file). andoria-08 stamps a new file up to 5 ms before a clock read taken just before it, so frozen-clock checks flake there; take the clock from the files' ctimes (memory note andoria-08-coarse-file-times).
- **Audit** (map:651-656): a sentinel probe (`verify-history-isolation.py`) before every independence-sensitive dispatch; a fresh fess audit; every audit prompt states the runner mode and the probe's result; mutants run by hand before the audit (topics:142); signing on hera with GnuPG, on Clio with the YubiKey, unsigned elsewhere (decision 67, dec:108).

### 4.6 Texts outside the program

- **nix-config skill.** Any rule change (skill:38-39 for hidden files, :13-15 for the seven tools, :174-175 for reading only through agent-mail) is a nix unit through nix's queue and activations on ten hosts, read by a session only after it restarts or rebuilds its context (map:601, :643; overview:155, :897).
- **Steward texts, on the light path** (decision 85, dec:171): CLAUDE.md (the closed list at :25-31; :21-23; :28; the filing table at :104-133; the census columns and counts at :202-209; NOTES.log at :362; the metric at :379; the digest and summary at :392-415) and the helper (signals:4-5, :22; the census at :604-637; its docs pins, overview:135). A live steward needs its files pulled and a fresh context to read a changed CLAUDE.md (map:649; gaps:183). CLAUDE.md is 53,446 bytes in the worktree, near the size at which its README says to split it (map:19; overview:134).
- **WORKER.md** (rule 4 at :28, the NOTE row at :17, `mailed:` at :13): "a revision is one push he approves and a pull per host" (decision 14, spec:586; overview:162).
- **POLICY keys.** A new threshold (an unanswered-after time, say) needs every conductor's own POLICY.md edited at the moment of the helper's pull, or every signals call is refused (map:192; overview:136). Three topic conductors run (ingest, nix, ares), and test-steward runs for the weekly re-runs.
- **The conductors spec** takes its amendments with markers (decision 25, spec:598), in sections 8, 11 and 12.

### 4.7 The version name

The steward-native proposal already uses "agent-mail 2.4.0: the roster as `doctor --format json`, and the doorbell refreshing the member record" (proposal:479). Decision 90 calls the send log "agent-mail 2.4" (dec:176). Whether they ship as one release or two is open (gaps:202, open question 11).

## 5. Fleet facts

- **Ten hosts:** hera (the hub), Clio, Vulcan, nyx, athena, pallas, andoria-08, andoria-t2, delphi-3bd4 and gpu-server (README:7-11, :183).
- **Star through hera** (decision 1, dec:3; README:183): "Mail between two other hosts passes through Hera, and waits while Hera is down." Hera reaches the Positron hosts only over Tailscale, which stopped on 2026-11-04 at 11:10 PST and cut them off (decision 80a, dec:154; memory note fleet-hosts-and-networks). An entry written on a cut-off spoke reaches the others late, and hera's replica is the most complete (overview:312).
- **The Positron hosts share an NFS home but not the Mailbox.** "The Positron hosts share `/home/jwiegley` over NFS, but each keeps its own copy of the folder on local disk, at `/var/lib/jwiegley/Mailbox`, runs its own Syncthing, and pairs with Hera over the tailnet. `~/Mailbox` on the NFS home is one symlink to that path." (README:185; decision 19, dec:21; mcp-spec:84, :88, :445; MCP decision 19). **This corrects the brief's "one copy": there are four copies.** What the four do share through the home is `~/.local/state/agent-mail/` (reader state, send windows, the diagnostic log) and `~/.config/agent-mail/` (`config.json` and the pause file) (mcp-spec:90; F6:113, :243; overview:761). Since `dirkey` hashes the mailbox's real path (am:488-490), which is `/var/lib/jwiegley/Mailbox` on all four, a name's state file has the same name on all four. Stale or ambiguous wording elsewhere: skill:9-10 ("the shared-work hosts also see it through their NFS home"), decision 17 (dec:19, superseded by 19), and the memory note fleet-hosts-and-networks ("must run exactly one instance (andoria-08)").
- **Vulcan** runs a second Syncthing instance as johnw, for the Mailbox only (README:184; decision 2, dec:4).
- **Syncthing:** for the Mailbox, `maxConflicts = 0` and `versioning = null` (nix-syncthing:298-301); `fsWatcherDelayS = 1.0`, `rescanIntervalS = 3600` (map:157); deletions propagate, so a prune on any host deletes everywhere (README:70-71); its temporary files are hidden names (README:194; mcp-spec:89).
- **Lag:** hera to andoria-08 9.9 s; andoria-08 to hera 2.2 s for the file and 4.4 s to the doorbell (README:209-215); one hop 2.07 s, with deletions about a minute behind (spec:65).
- **Clock skew of 10 minutes:**
  - `MARGIN = 600.0  # since's allowance for clock skew` (am:1341). A post stamped earlier than `since - MARGIN` is skipped (am:1385); the self-test pins "since keeps a 10-minute margin" (am:4493-4496).
  - `FUTURE = 86400.0`: a stamp more than 24 hours ahead is invalid (am:1342; README:70).
  - Another host's claim stands for 10 minutes (`CLAIM_STALE`, am:99; README:166).
  - The duplicate-reader warning fires within 10 minutes (am:1345; README:191).
  - "Clock skew | Ordering changes. `since` has a 10-minute margin. Future stamps beyond 24 hours are invalid. Pruning by a fast clock removes history early; hosts are assumed to run NTP." (mcp-spec:497).
  - A stamp is the writer's word; arrival is the file's ctime (spec:332; map:159; README:165; CLAUDE.md:106). The steward's `topic_since` uses the same 10-minute allowance (spec:303), and F1 added `MARGIN` to a claim's `until:` (overview:275).
- **Trust:** one OS user per host, and every session runs as John (mcp-spec:460).
- **Volumes:** about 35 live sessions on hera and about 11 on five remotes; `+fleet` gets about 3 posts a day and `+nix-config` about 5 a week (overview:83, as of 2026-11-04). Six channels exist: `+ares-fleet`, `+fleet`, `+ingest`, `+nix-config`, `+steward-test` and `+vulcan` (the listing). At 19:4x PST on 11-05, 101 message-named entries waited at hera's top level, direct mail and broadcasts together (the listing; only the count was kept).
- **Interpreters:** hera's python3 3.14, macOS `/usr/bin/python3` 3.9.6, and Linux (map:50).

### 5.1 The code map's twelve facts, read against a send log (map:9-22)

| # | Fact (map line) | Bearing on the log |
|---|---|---|
| 1 | Seen sets never leave their host (:11) | the log cannot say who read what; "unread" stays the roster's question, not the log's |
| 2 | Nothing parses a post's subject or body (:12) | agent-mail may store a subject, never interpret it; the model reads it (overview:106) |
| 3 | No steward has run (:13) | stale: three stewards run since 2026-11-05 (dec:169), and the patches are retired (a3-spec:623; dec:177) |
| 4 | The one push path to John is the director's scanner (:14) | the log reaches John only through a steward's question line; the scanner reads nothing in the Mailbox (decision 87) |
| 5 | Waking differs by harness (:15) | Codex sessions read only at a turn's start; their sends are logged only once their server runs 2.4 |
| 6 | `checked` means different things per harness (:16) | never infer reading or answering from member records |
| 7 | Old and new versions must coexist (:17) | the config-key rule; MCP schemas refuse unknown arguments; sessions keep their servers |
| 8 | No conflict copies in the Mailbox (:18) | the log must be single-writer or write-once |
| 9 | Text surfaces are small and pinned (:19) | instructions 472 of 512 characters; `channels` 8 KB; CLAUDE.md near its split |
| 10 | Two release paths (:20) | the program path for 2.4; the light path for the steward follow-ups |
| 11 | Steward cost is not measured (:21) | no token or dollar figure for reading the log; counts only (decision 20) |
| 12 | Member records are never pruned (:22) | the log needs its own prune |

## 6. Prior art in the two earlier design reports

### 6.1 F1, the fleet table (overview report, 2026-11-04, section 3)

F1 is a read-only `steward/steward-fleet` on hera. It joins agent-deck's rows from ten hosts, member-record presence, unread direct mail by name and ctime, each steward's STATUS and, in the full build, items parsed from posts on channels that have no steward (overview:28, :168-224). It bears on the log in four ways.

1. **The same skill wall.** Even its small form, S1, "lists other names' direct mail by name and ctime and stats their hidden member records: metadata reads that the conductors spec allows the stewards alone ... while the agent-mail skill forbids a session to read 'any hidden file in the mailbox that is not yours'" (overview:177). It offered three ways out: runners outside the mail rules, an exception in the runner's own rules, or a skill amendment (F1:1110-1122).
2. **An output rule.** "It never prints a peer's subject, body ..." (overview:224), plus a defence that searches its own output with the scanner's signatures (overview:277-289).
3. **An agent-mail `overview` command, costed and rejected:** "Nothing in agent-mail parses bodies"; such a command would couple the mail standard to WORKER.md (overview:106, :350, :970).
4. **Privacy, named as a drawback.** It would be "the first all-channels reader of post contents" (overview:335; F1:892-905). It also rejected read stamps in member records as amending decision 10 (F1:1071-1080).

**Verdict:** "Do not build yet. Build the item half when the pilot records a parked or held item that the digest and the director's pages both missed. If John wants a Discord answer sooner, the director convention of §3.10 (no lines)." (overview:28; open decisions 11 to 13 at overview:1035-1049).

### 6.2 F6, the handoff pointer (overview report, section 8)

F6 is an optional `handoff` object in the member record, set by `agent-mail handoff NAME PATH`, published at once, shown in the bound session's `channels` and from any host (overview:726-757). It bears on the log in four ways.

1. **Its exposure statement is the model for the log's.** The record "reaches the disks of all ten Mailbox hosts, among them the four Positron hosts, corporate machines ... every process of John's user there", and "A file name can say more than its owner means it to" (F6:355).
2. **Its rollout avoided both costly forms.** An eighth MCP tool "reopens decision 7 ..., changes nix's tool pin ..., and costs every session the tool's definition" (F6:404). A config key is "refused by every older version; a two-phase rollout and a nix-config unit per pointer" (F6:406). Hence "No config key, tool, argument or `v` changes, so no two-phase rollout is needed" (overview:761, :893).
3. **The member-record rule:** one optional key, validated on every read, which older writers drop (overview:123, :761). That makes the member record no home for log data.
4. **Forgery.** Records are peer-writable, so a pointer is a lead that a successor reads only on John's word (overview:774; F6:358).

**Verdict:** "Convention now, in the next nix-config unit that touches skills; the pointer only after F1 is built, in a 2.3 release of its own with the `member_at` catch." (overview:33, :710; open decisions 25 and 26 at overview:1104-1113).

### 6.3 6A, self-attestation (steward-native design, 2026-11-05; gaps section 7)

6A builds on candidate 3's host-local store of the sender's own copies (`.<name>-read/sent/`, kept 31 days, written only by the process that holds the name's reader lock), which then doubles as a ledger: agent-mail lists "files in your name that the server reading your mail on this host did not write" (gaps:146). It bears on the log in three ways.

1. **It puts sender-side records on the host.** They are host-local and never synced: "Host-local; nothing crosses; old servers show no notice" (gaps:93, :95, :165). Decision 90's log is the opposite: synced to every host.
2. **Its strength comes from its writer.** Only the reader-lock holder writes the store; the CLI's `send --from` keeps no copy (gaps:91). Even so, a forger on hera that starts its own server under the name gets through (gaps:149). A synced log that `send` writes for any asserted sender (am:3329; spec:82) therefore adds no authenticity, and cannot retire SENT.log or the forged-file census (CLAUDE.md:149; spec:347).
3. **It needs a start stamp,** or every earlier file in a name looks forged for 30 days (gaps:151). A log has the same cold start: entries exist only from 2.4 senders onward.

**Verdict:** "Retiring SENT.log and the forged census. Keep both now. A later release with 31-day copies of what the sender sent and self-attestation (candidates 3 and 6A; ...) has today's strength and only simplifies. When decision 5b's revisit signal fires (spec:634), or when a release is going out anyway" (proposal:480). "6A would retire them at the same strength, not a stronger one." (proposal:260). Signed mail (6B) is out of scope (gaps:154-163).

### 6.4 Also bearing: the exploration's S3, tool-written copies (2026-10-31)

S3 was S1 plus "**Copies on send.** A boolean `copy`, on by default for joined members, writes their direct mail byte for byte, with `Cc:`, into `+<topic>-cc`, which only the steward subscribes to", and "**Stubs.** A copy dropped for size, cap or opt-out leaves a bodiless stub post on its own budget". Its cost was "about 60 lines of v2.1 ... plus a stated exception to v2 §6.4: member records decide what is copied" (synthesis:68-74). **Verdict:** "S3 only if NOTE coverage is below about 80%" (synthesis:111), and it is decision 1's named alternative in revisit row 1 (spec:629). The metadata log is S3's bodiless stub, kept for every sender (not joined members only) and written where every host can read it (not into a channel only the steward reads).

### 6.5 Candidates 1 and 3 of the gaps note, for completeness

- **Candidate 1, the roster** of waiting direct mail: half built in doctor since 2.1.0 (gaps:24); "from file names only, writing nothing"; "Names and stamps only, never ids or subjects" (gaps:35-37). It writes nothing, so nothing crosses hosts. An older CLI refuses `doctor --format json` with status 2, so a consumer falls back (gaps:48). It answers "who has unread mail", not "who answered", which is the log's question.
- **Candidate 3, the sender's own copy:** "Host-local only: nothing new crosses hosts, no mail name or channel changes, no key. One reader per name holds, since the copy is the reader's own." (gaps:95). Only the reader-lock holder keeps copies, for 7 days, or 31 days in `sent/` (gaps:91, :93).

## 7. Where the requested search terms are

| Term | Anchors |
|---|---|
| "copy, not custody" | spec:568 (decision 1); spec:629 / a3-spec:633 (revisit row 1); map:53-55; spec:12 ("sees a copy and never carries") |
| "listing file names" | spec:101 (rule 3, John 9); spec:373 (census, "Names only, never contents"); CLAUDE.md:28; signals:22; overview:177 |
| "never read" | skill:38-39; spec:101; CLAUDE.md:28; map:36 |
| "hidden" | skill:38-39; README:44-45; README:172; am:1276, :1373, :3082, :3122 (readers and doctor skip hidden names); F1:1116 ("despite the agent-mail skill's rule on hidden files"); decision 10 (dec:12) |
| "doctor" | README:30, :58, :71, :172, :177, :194, :234; am:3035-3149 (what it flags: no mailbox or marker, unsafe meta directories, missing `.stignore` lines, conflict copies, non-mail names at the top level, a `+` entry that is no channel, a non-post, misfiled or invalid post in a channel, another reader active on another host within 10 minutes, the pause file; plus the roster of waiting mail as plain lines) |
| "config" | README:106-117; am:263-329; nix-amconfig:1-19; nix-mcptest:158-165; map:145, :189; dec:31 (decision 26) |
| "unknown key" | README:106; am:303-319; am:3935, :3972-3975 (self-test); gaps:26; map:17, :145; spec:528 |
| "MARGIN" | am:1341, :1385, :4493-4496; map:173; overview:275; F1:405, :661 (a claim's `until:` and clock skew), :1121 (decision 2(a)) |
| "10 minutes" | README:63 (same body refused for 10 minutes), :166 (another host's claim), :191 (duplicate reader); spec:81, :303, :583 (relay check 3); WORKER.md:7, :50; mcp-spec:497 |

## Open questions

1. **Positron copies.** The brief says the four Positron hosts "share one NFS home and so one copy". README:185 and decision 19 say each keeps its own copy of the Mailbox and shares only the home: state, config and the pause file. The design should assume README:185, and should say where any host-local part of the log lives, given that `~/.local/state` is shared on those four.
2. **Scope of the steward's view.** "Traffic patterns within a set of topics" (filter by each topic's members), or "ALL mail traffic ... posts to any channel" (amends decision 32 and rollout decision 72)? Only John can choose.
3. **Who reads, and by which path.** A rule in the steward texts only (unenforceable), or every session through agent-mail? A hidden file with an agent-mail read path (skill unchanged; the steward's closed list grows), or an amendment of skill:38-39 (a nix unit)? Is a `stat` or a listing a "read" under skill:38-39 (overview:177 left it open)?
4. **Is a subject "metadata"?** Decision 90 says yes. The steward texts and the helper say metadata is names and stat data and promise "never a message's text". Which texts change, and do the helper's pins move?
5. **Retention and pruning.** Within the 30-day non-goal (mcp-spec:64)? Who prunes, by which clock, given that a fast clock prunes early (mcp-spec:497)?
6. **Configuration.** Any key at all (on or off, retention, readers) means two nix units and departs from decision 26's empty config. Are constants enough?
7. **Writer shape.** One file per sender and host, one file per entry, or daily files? The constraint is single-writer or write-once, with the CLI able to send as any name from any host.
8. **One release or two.** Does 2.4 also carry the roster and doorbell items the steward-native proposal called 2.4.0 (proposal:479; gaps:202)?
9. **The NOTE convention after the log.** Does WORKER.md rule 4 stay for its slug link, change, or go? Any revision is John's push under decision 14.
10. **The revisit table.** Amend row 1 to record decision 90 (the reversal is by John's word; the row's signal was never measured), and add rows for the log itself: what would withdraw it, and what would show it crowds a steward's turns.
11. **Decision 12's text change.** Does it stay out of 2.4 unless its evidence exists?
12. **The stewards' own servers.** Which agent-mail version do the three stewards run (gaps:203)? Their own sends are not logged until they restart on 2.4.
13. **Data domains.** Does John, or Positron's policy, accept Positron subjects held durably on personal hosts and personal subjects on the Positron hosts? Decision 7b covered posts on the topic channels; decision 90 names "every direct mail's subject visible fleet-wide" but not the domain split.
14. **The day-one backlog.** 101 message-named entries waited at hera's top level at 19:4x PST, including mail for readerless names (decision 45 counted 61 such files for eight names, dec:62). "Unanswered" and "misdirection" signals would flag them at once, though a log started at 2.4 sees none of them as sends. How should the design treat mail sent before the log began?
15. **Reads versus sends.** Does any part of the design need to know that mail was read (not only sent)? If so, it crosses the "read receipts" non-goal (mcp-spec:64) and decision 10 (dec:12).
