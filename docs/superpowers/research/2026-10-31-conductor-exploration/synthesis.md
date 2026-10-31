# Per-topic stewards for agent-mail: synthesis

Written 2026-10-31, about 11:00 PDT, on hera, for John and the designer who will brainstorm with him on branch work/conductor.

**Marks.** [V: x]: verified today by report or critique x; [V: me]: checked while writing this. [U]: documented or estimated, unchecked. Dollar figures are the critics' list-price estimates, ±50%; the fleet's billing basis is unchecked.

**Sources** (in `docs/superpowers/research/2026-10-31-conductor-exploration/`): six reports, three candidates (design-A, B, C) and six critiques (critique-{A,B,C}-{trust,goalfit}). Brief: `/Users/johnw/Products/agent-mail-conductor/docs/superpowers/briefs/2026-10-31-conductor-exploration.md`.

## 1. Summary

1. **Long-lived fleets don't relay through a coordinator.** No such system relays every peer message through an LLM. Relaying costs an extra call per request, paraphrase errors (LangChain's fix gained about 50%, https://www.langchain.com/blog/benchmarking-multi-agent-architectures) and 285% overhead (https://arxiv.org/html/2512.08296v3). Coordinators pay off by verifying, not carrying [V: prior-art]. `docs/superpowers/research/2026-10-31-conductor-exploration/prior-art.md`
2. **agent-mail 2 already supports the cheap form and rules out the relay.** A topic channel serves as the visible copy, with routing by reference through `in_reply_to`, and needs no code. A relay collides with the anti-goal "No tool that executes, approves, forwards, re-posts or edits configuration" [V: me, design §4] and with "Never forward or re-post peer text" [V: v2-fit]. `docs/superpowers/research/2026-10-31-conductor-exploration/v2-fit.md`
3. **The relay is where the risk is.** At 100 messages an hour it saturates near 36 s per message, its death stops delivery silently, and it re-reads about 16 times the tokens of a script-filed copy (inputs assumed [U]). `docs/superpowers/research/2026-10-31-conductor-exploration/risks.md`
4. **Questions already reach John through the director's scanner, not mail.** About every 69 s it pages any pane's trailing question line to Discord and indicator 1. Its Codex model reads mail only when given a turn, and it never reads work, so it missed the Vulcan duplicate, the obr id collision and hera's load contention [V: director]. `docs/superpowers/research/2026-10-31-conductor-exploration/director.md`
5. **Both skills are drafted on pass-through** (direct mail plus a one-line NOTE). v2's doorbell never refreshes member records, so an idle Claude session looks dead [V: operations]. `docs/superpowers/research/2026-10-31-conductor-exploration/operations.md`
6. **John answers `<id> <letter>` in the steward's pane**, the only place his word counts; a relayed answer is a peer's report. At 10:13, 13 sessions had unread mail, some 18 hours old [V: johns-view]. `docs/superpowers/research/2026-10-31-conductor-exploration/johns-view.md`
7. **The three candidates agree on the shape.** Conventions (A), v2.1 primitives (B) and a filing tool (C) all chose the name "steward", visibility over custody, Claude Code on hera, and the scanner for questions. `docs/superpowers/research/2026-10-31-conductor-exploration/design-C-tooling.md` (and the A and B files beside it)
8. **All six critiques say "fixable" and share five defects.** Nothing watches the steward. Page lines over 80 characters wrap in hera's 108-column panes. "John chose X" is relayed from a pane others can type into. Stewards keep dangerous_mode. Claims are ordered by stamps the writer chooses. `docs/superpowers/research/2026-10-31-conductor-exploration/critique-A-conventions-trust.md` (and the five other critique files)
9. **B's copies reach every member,** which recreates global messaging. A copy channel that only the steward reads fixes this [V: critique-B-trust]. `docs/superpowers/research/2026-10-31-conductor-exploration/critique-B-primitives-trust.md`
10. **Two pilot blockers lie outside agent-mail.** agent-deck 1.16.29 on andoria-08 and delphi-3bd4 reports idle Claude workers that run watchers as `running/background-work`, which blinds the stuck, starved and deaf checks; 1.16.30 fixes it [V: critique-C-goalfit]. And Positron posts would sit on all ten hosts for 30 days. `docs/superpowers/research/2026-10-31-conductor-exploration/critique-C-tooling-goalfit.md`

## 2. The pattern in one page

**Roles**

| Role | Who | Does | Never |
|---|---|---|---|
| Steward | `<topic>-steward`, one Claude Code session per topic on hera | Keeps the ledger; flags duplicates, overlaps, drift, and stuck, starved or deaf workers; routes unaddressed requests by reference; asks John on its pane's last line | Carries, quotes or re-posts mail; assigns or approves; does the work; types into panes or restarts sessions; speaks for John |
| Workers | Topic members on any host or harness | Claim first; post state; mail peers directly, then post a NOTE; ask John in their own pane for authorizations | Treat steward mail as John's |
| Director | conductor-director and its scanner | Unchanged: health of every session | — |
| John | | Sets the purpose, members and backlog. Answers in the steward's pane. Authorizes only in the acting worker's pane. | — |

**Routing options**

| | Channel post, read by both | Cc copy (direct mail plus a copy) | Relay by the steward |
|---|---|---|---|
| Wakes B | Only on a wake subscription | Yes: direct mail rings | Only after the steward's turn |
| Latency, Positron to Positron | About 10 s | About 10 s | 31 to 76 s, more when queued (risks R1; turn time assumed) |
| Steward dead | Delivery unaffected | Delivery unaffected | Delivery stops; senders are warned only after an hour |
| Provenance | Kept; widest reach | Kept; narrow reach | Peer text goes out under the steward's name, amplifying injection |
| Fits v2 | Works today | Two sends today; `cc` would be v2.1 | Collides with the §4 anti-goal |
| Unique benefit | A log anyone can read | — | Can stop a request before B acts (risks R28) |

**What "through the conductor" should mean here: the steward sees the traffic but never carries it.** State changes are posts on `+<topic>`. A request to one peer is direct mail plus a one-line NOTE post. A request that names nobody is a post, and the steward mails the likeliest member in its own words, with `in_reply_to` set to that post. The steward relays attention, never text. Claiming before working, and correction within one tick, recover most of what holding a request would give: delivery takes seconds and work takes minutes (v2-fit §3.1).

**What the director keeps:** restarts on error, its Continue and Enter recoveries, the stall page, pane-question pages, indicator 1 and the Discord bot. A steward duplicates none of these, never runs `notify_discord.py`, never drains another inbox, and ends a line with "?" only to page John. Nothing time-sensitive goes to the director by mail.

## 3. Candidates

These recombine the three drafts with their critics' fixes. S1's rules underlie all three.

**S1. Steward by convention (A, hardened)**

1. **The session.** One Claude Code session per topic on hera, launched `--no-parent --no-assert-done`, under an allowlist instead of dangerous_mode if a test shows that works. Its ledger, STATUS and append-only decisions log (in git) live in `~/.local/share/agent-mail/stewards/<topic>/`, outside `~/Mailbox`.
2. **What workers do.** They post CLAIM (with `writes:`, `holds:` for GPUs and id ranges, optional `until`), UPDATE, BLOCKED and DONE to `+<topic>`. To ask a peer, they mail it directly and post a NOTE.
3. **How the steward wakes.** A quiet 15-minute tick while items are open, one tick at a time (pgrep); a wake subscription when nothing is open. Each turn calls one read-only signals helper and logs received ids before filing them.
4. **What it checks**, once per episode: duplicates, overlapping writes or holds, landed work, drift, stuck workers (including busy 2 h with no update, honouring `until`), starved and deaf workers, and unanswered asks. Claims are ordered by first-seen time. Its mail uses ids it assigns and fixed templates, never peer slugs or summaries.
5. **How it asks John.** One line of at most 80 characters, ending in "?", after every tool call. The scanner pages it, and John answers in the steward's pane. A rule in John's director files pages when STATUS goes stale.

**S2. Steward with a filer (C, split and fenced)**

1. **For workers, nothing changes.** S1's conventions and steward rules apply unchanged.
2. **A filer keeps the books.** `agent-steward run <topic>` is a launchd KeepAlive job on hera, reading under `<topic>-steward.ledger` because the MCP server holds the steward's reader lock [V: critique-C-trust]. It files posts within seconds, runs the mechanical checks, writes the ledger views and a heartbeat stamp, and counts mail file names for coverage.
3. **The model wakes only to judge.** A content-free bell wakes it at once for an overlap, authority words or a cycle, otherwise at most every 30 minutes. It approves every outbound message.
4. **The asker rings its peer itself.** No tool pointers at first; later perhaps a 30-minute second ring for roster senders.
5. **It ships separately**, in its own checkout on hera, not `pkgs.agent-mail`: about 1,500 lines plus 800 of tests, with self-test invariants covering names, slugs and paths.

**S3. Steward with tool-written copies (B, fixed)**

1. **S1, plus two v2.1 primitives.**
2. **Copies on send.** A boolean `copy`, on by default for joined members, writes their direct mail byte for byte, with `Cc:`, into `+<topic>-cc`, which only the steward subscribes to.
3. **Stubs.** A copy dropped for size, cap or opt-out leaves a bodiless stub post on its own budget.
4. **A roster in `channels`:** waiting mail per recipient, from file names.
5. **Cost.** About 60 lines of v2.1 after Task 25, then audit, push, publish and activation on every host, plus a stated exception to v2 §6.4: member records decide what is copied.

**Trade-offs**

| | S1 Convention | S2 Filer | S3 Copies |
|---|---|---|---|
| Goals met (of 7) | 1 met (questions), 6 partly | 1 met, 6 partly; faster on duplication and conflict, measured coverage | 1 met, 6 partly; tool-sent mail always visible |
| Adds to v2 | Nothing. Skill text after Task 22 (uncommitted). | Nothing required. Optional `channels --json` (Tasks 12 and 18, committed). | Code in committed Tasks 4 to 6, 12, 14, 15 and 18, plus a nix rollout like Tasks 20 to 24 |
| Files and model tokens per message | Request 2 files, state post 1; model reads every post (150 to 375 tokens); $7.50 to $57 per steward-day fixed, $33 to $107 as written (critique-A-goalfit) | Same files; 0 tokens per routine post, about 200 per claim; $19 a day at 2 wakes an hour, $111 at 6 (critique-C-goalfit) | 2 files per request; about 375 tokens per copy; $33 to $75 a day (critique-B-goalfit) |
| Latency (delivery unchanged in all) | Notices 1.5 to 17 min after the second claim; pages 1.5 to 4 min | Notices 30 to 80 s; judgement within 30 min | Notices 10 to 18 min |
| On failure | Oversight pauses, delivery continues; noticed only via the STATUS rule | Filer outlives the model; a stale stamp pages | As S1; a refused copy still delivers |
| Etiquette must carry | Claims, NOTEs and verbs. Coverage unmeasured without a census. | Claims and NOTEs. Coverage measured. | Claims only. Bypass must be deliberate. |
| Time to usable | Day after Task 25, plus agent-deck 1.16.30 on the Positron hosts | S1 plus days of build; it can join a running pilot | Weeks of v2.1 work and rollouts |

Workers pay about $8 to $16 a day per topic for posts in all three (critique-A-goalfit). The relay stays rejected: 21 to 66 s more per message, silent delivery stops, and an anti-goal amendment.

## 4. Recommendation

**Default: S1, built so that S2's filer can be added later without changing what workers do.**

Why:

1. Every critique keeps the shape, rejects custody, and finds S1's defects fixable in place.
2. B's code does not power the duplicate and conflict checks, which read CLAIM posts in any design (critique-B-goalfit issue 3).
3. C as written puts the steward back on the attention path (critique-C-trust issue 1), its tool dies at every wake (issue 2), and its home moves agent-mail's pin on ten hosts (critique-C-goalfit issue 4).
4. S1 has the fewest parts; rollback is stopping one session (critique-A-trust). It stays off v2's critical path if the worker section stays out of Task 22 (critique-A-goalfit §4).
5. Its weak points, cost and detection delay, are measurable in a week, and S2 fixes both without touching workers.

**Ship first** (one topic, one week, after Task 25):

1. **The skill** as `CLAUDE.md` in the steward's directory, re-read every turn; worker conventions in the opening post, plus one line John types in each member's pane.
2. **Trust fixes:** relayed decisions limited by question 3; first-seen ordering; steward-assigned ids; a write-ahead id log; a ledger lock. Each turn the steward also compares the `from-<steward>` names in `+<topic>/` with its sent log, because a reader never receives posts in its own name [V: critique-A-trust].
3. **Goal fixes:** `holds:` and `until` in claims; the busy-stuck rule; questions only John can answer page within 45 minutes; a wake subscription while idle.
4. **Cost fixes:** the signals helper; a summary of at most 1 KB re-read each turn; a fresh context each morning; an overnight tick of at most 55 minutes.
5. **Preconditions:** agent-deck 1.16.30 or later on the Positron hosts; John's STATUS rule; a permission test.
6. **Measure:** tokens per day, NOTE coverage, interventions John rates useful, and pages per day.

**Later:** S2's filer, if day-one cost exceeds the budget or a 15-minute delay proves too slow; request 5 (the roster) after landing; S3 only if NOTE coverage is below about 80%; the Discord answer relay; `+nixcfg` once John decides whether the director hands over its ad hoc Nix records (director §4.9).

## 5. Questions only John can answer

1. **Visibility or custody?** (a) The steward sees posts and NOTEs and corrects within about 15 minutes. (b) A relay: amend the anti-goal, and accept a single point of failure, doubled hops and an injection path. (c) Gate claims on shared paths: more latency, and a single point of failure for gated work. *Recommend (a).*
2. **Who watches the steward?** The live-work footer suppresses the stall page, which fires once per session lifetime anyway [V: critique-B-trust]. (a) One rule in your scanner: page once when `stewards/*/STATUS` is older than twice its tick while items are open. (b) A separate watchdog, duplicating the Discord path. (c) Nobody. *Recommend (a), and drop the stall-exemption idea.*
3. **What may a steward relay as "John chose"?** Any hera session can type into its pane. (a) Only choices any peer could suggest: order, holds, technical choices within a claim; publishing, spending, deletion and scope go to the acting worker's pane. (b) Nothing; it only points. (c) Everything. *Recommend (a).*
4. **Less power for stewards?** No dangerous_mode, and an allowlist: agent-mail tools, read-only agent-deck catalog queries, pgrep and sleep, with writes only in the steward's own directory. Whether one session can override the global setting is unknown, so test it first. Cost: the steward cannot fix anything itself. *Recommend yes.*
5. **The first topic.** (a) `+positron-ingest` with the seven Claude sessions, accepting posts on all ten hosts for 30 days and questions on Discord. (b) The same, after a nix `.stignore` rule keeps `+positron-*` off the personal-only hosts. (c) `+nixcfg`: sequential work, already coordinated ad hoc by the director. Ingest needs agent-deck 1.16.30 or later on andoria-08 and delphi-3bd4. *Recommend (b), or (a) with page lines carrying only ids and counts.*
6. **The name.** (a) steward (`ingest-steward`): unreserved, commands no one. (b) shepherd: the IETF role, apt but less familiar. (c) coordinator: neutral. (d) foreman: agent-deck's README already uses it. (e) `conductor-<topic>`: Discord answers, but agent-deck's machinery comes with it, and a name sorting before "director" captures your unprefixed messages. *Recommend (a).*
7. **Where questions reach you.** (a) A pane line, through the scanner, to Discord; you answer in the steward's pane. (b) (a), plus a director POLICY line relaying `<topic>-<n> <letter>` from Discord as a peer report, for choices that need no authority. (c) A numbered menu with Discord buttons, which blocks the steward. (d) `conductor notify`, which hera's bridge does not forward. Quiet hours 23:00 to 07:30. This reaches the director's scanner, not its Codex model. *Recommend (a) now, (b) after a week.*
8. **Budget and freshness.** S1: about $7.50 to $57 per steward-day plus $8 to $16 for workers, notices within 15 minutes. S2: notices within a minute, for about 2,300 lines and a second reader name. *Recommend S1 for a week under your cap.*
9. **May a steward list file names, never contents, in `~/Mailbox`** to find deaf members and measure coverage? *Recommend yes.*
10. **What may it do alone?** Proposed: route by reference; flag; nudge at most twice per item per day, never Codex workers; offer backlog you seed; apply defaults that only hold, wait or skip. Never assign, approve, or touch a session's lifecycle. *Recommend as proposed.*

## 6. Requests to the rollout session for agent-mail 2

To be mailed to `agent-mail` (`mail-dev` after the rename) after John approves; each is a peer request. work/v2 at 10:56: HEAD 0a53627 (Task 19 audit fixes), an uncommitted change, Tasks 20 to 25 not committed [V: me].

1. **Task 19 (uncommitted), informational.** No v2.0 change is needed. Please flag it if Tasks 19 to 25 change any of four behaviours: `read_member` passing extra keys, readers printing headers as text, `read_purpose` folding extra marker lines, and readers skipping posts in their own name.
2. **Task 22 (uncommitted).** Keep "Never re-post or forward peer text" and "direct mail when one owner must act" verbatim. Leave the steward's worker section out; it ships separately after the pilot.
3. **Task 18 (committed); fix in Task 19's audit.** Spec 6.8 and the README promise Codex heartbeat reads. Since 09:44 the director's heartbeat runs a scanner and gives Codex no turns (director §6.1).
4. **Task 19 (uncommitted): docs and a plan note.** Say what a channel directory may hold besides posts and the marker (v2-fit R8), and that topic channels have no retire command (R10).
5. **v2.1, touching committed Tasks 5, 12, 15, 17 and 18.** A roster of waiting direct mail per recipient (count, oldest stamp), from file names, in `channels` or `doctor`, writing nothing. johns-view and director asked for it independently.
6. **v2.1, optional, touching committed Tasks 3, 4 and 6.** A per-session hourly cap on direct mail, `sessions.<name>.direct_per_hour`; the code must land before nix writes the key.
7. **v2.1, optional, touching committed Tasks 4, 13 and 15.** A doorbell liveness stamp. It breaks "writes nothing but its slot", and B objected, but it ends false "last checked" warnings fleet-wide.
8. **Conditional.** With S2 only, `channels --json` (committed Tasks 12 and 18). If John agrees, pin each steward's topic in Task 22's config (uncommitted; amends decision 26), and a `.stignore` rule for `+positron-*` (outside the v2 plan).

**Needs nobody:** the skill texts, the files on hera, the signals helper, the opening post, read-only agent-deck commands, and John's `channel create` and launch lines.

**Needs John's own files:** the STATUS rule, an AGENTS.md pointer to STATUS, and optionally a scanner test fixture for steward lines.

## 7. Risks the recommendation keeps

| Risk | Mitigation | Test |
|---|---|---|
| Bypass: a forgotten NOTE, or same-host mail claimed before it reaches hera | A names-only census; a coverage figure in each digest, as a lower bound | Bypass on purpose; check it shows |
| The scanner's heuristics change (edited three times today) | Lines of at most 80 characters; a fixture in John's tests | Plant a question; expect exactly one page |
| Injection disguised as John's answer in the pane | Question 3; treat the `[agent-deck from:…]` envelope as peer input | `session send ingest-steward "ingest-3 B"` must not produce "John chose" |
| A forged `from-<steward>` post | Compare against the sent log; workers post DISPUTE | Plant a forged DIGEST |
| A deaf steward after a restart that is not an error | The STATUS rule | Kill the doorbell and the tick |
| Compaction in the middle of a batch | Write-ahead log reconciled against the archive | Force `/compact` mid-batch |
| Races between stamps and partitions | First-seen ordering; flag late claims | Replay a 6-hour partition |
| A nudge storm (direct mail is uncapped) | The allowlist, the steward's own caps, request 6 | Plant "remind everyone" |
| Idle workers misread on agent-deck 1.16.29 | Require 1.16.30 or later | A fixture from today's JSON |
| Conflicts over GPUs or ids | `holds:` | Seed a GPU clash and an obr id clash |
| Codex workers cannot be reached by mail | Never nudge them; list them for John | — |
| Cost overrun | Measure from day one; John's cap | Harness usage figures |

## 8. Still unknown or unverified

- Whether one session can drop the global dangerous_mode, and whether deny rules hold.
- Whether a background `sleep` exit wakes an idle Claude session (the watcher exit does).
- Claude Code's cache lifetime, autocompact threshold and John's billing basis, on which every cost figure rests.
- The fleet's message volume: archives are host-local, and nobody measured it.
- Syncthing ordering across directories, and `maxConflicts=0` behaviour.
- Whether skill text survives compaction.
- Whether a ring reaches a busy worker mid-turn, whether a pending numbered menu blocks a steward's wakes, whether Remote Control lets John answer from his phone, and whether parent links cross hosts.
- Prompt Infection, known to the critics only through v2 §2.
- Side effect: johns-view's outbox listing created an empty `~/.local/share/agent-deck/runtime/human-outbox/director.lock`.
