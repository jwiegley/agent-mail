# Critique of candidate B (small primitives in agent-mail 2): goal fit and cost

Critic for the goal-fit and cost lens. Written 2026-10-31, about 10:55 PDT, on hera. Everything was read-only. Candidate: `design-B-primitives.md`, written about 10:45.

Evidence marks:

- **Verified**: I read or ran it today, at the place named.
- **(report.md)**: a sibling exploration report verified it, and I did not re-check it.
- **Unverified**: documented or assumed, not checked.

Dollar figures are list prices for Claude Opus 5.5 (section 2.1). I did not check how the fleet is billed. Two things moved while B was being written:

- v2's HEAD is now 9de4b43, with an uncommitted diff of +88/−4 lines in `agent-mail`. B cites a30f782.
- The director's scanner was edited at 10:34, and a Discord reply module appeared at 10:36 to 10:44 (Verified: `git log`, `git diff --stat`, `ls -la`).

## Verdict: fixable

B's shape fits John's goals:

- The steward never carries mail, so it cannot become a bottleneck or a silent single point of failure.
- Questions take the one path that already reaches John's phone in about a minute.
- The steward stays a peer whose mail authorizes nothing.
- agent-mail 2.0 lands unchanged.

Four things keep it from being sound as written:

1. **Every member reads every copy.** By default each worker subscribes to the channel that receives the copies. So each member-to-member message is delivered to about 7 models instead of 1, and each worker gains about 10k tokens of context an hour. One bad message reaches seven contexts, a reach that v2 §6.4 rejected for broadcasts. And a request copied to bystanders can be misread as addressed to them.
2. **The cost section counts only message bodies.** B's own per-turn steps grow the steward's context by about 40k tokens an hour, not 13k, at about 10 requests a tick. If the cache lifetime is 5 minutes, every tick rewrites the whole context. Nothing bounds the context. That is about $1 a tick and $75 a day at a 127k mean context, rising with uptime.
3. **The new code is not what catches duplicates and conflicts.** The checks for duplication, conflict, drift, stuck and starved workers read CLAIM and UPDATE posts. Those are etiquette, exactly as in a zero-code design. Yet the pilot waits for v2.0, a v2.1 build and two fleet rollouts.
4. **Some goals have gaps:**
   - conflicts over shared resources and id spaces;
   - workers that are busy but stuck;
   - questions asked only by mail, which can take two to three hours to reach John.

Each issue has a small fix (section 5), and none of them requires giving up B's design.

## 1. Goal by goal

| # | John's goal | How B delivers it | Delay | What it misses | Rating |
|---|---|---|---|---|---|
| 1 | "see the overall pattern of all the work" | Copies of members' direct mail, and status posts, in `+<topic>`. A ledger on hera. A status line each turn, in the pane and agent-deck's preview. A digest at 07:30, on request, and as a quiet post at most every 2 h. The `channels` roster, from any host. | Ledger at most one tick behind (15 min busy, 25 idle) plus a 1–3 min turn (Unverified) | No view across topics. No measure of the traffic it cannot see. Work never announced by CLAIM. Everything but the Q line must be pulled. | Partly |
| 2 | "see if there's any duplication" | Each new CLAIM's distinctive tokens are matched against open and done items. One mail to the later claimant, citing ids. | Mean ≈10 min, worst ≈18 min, plus the worker's current turn if it is busy | Work begun without a CLAIM. Duplicates with sessions outside the topic. Paraphrased duplicates. Duplicates that the copies themselves invite. | Partly |
| 3 | "anything ... that might conflict" | Overlapping `writes:` between open claims, or two copies asking one worker for an exclusive resource. B mails both owners; the earlier claim stands; John after 30 more minutes. | First mail ≈10–18 min. Page to John: mean ≈50 min, worst ≈70 min. | Holds on shared resources (GPUs, host load), id spaces, non-members. Seen only after both claims exist. | Partly |
| 4 | "all of the work is harmonious to the purpose" | John's purpose, in his words, heads the ledger. A claim it does not cover gets one question and a digest line. Scope questions become Q lines. | Next tick. John sees drift in the next digest. | Drift inside an item. Workers see only the 200-character channel marker. No means to correct, by design. | Partly (the weakest) |
| 5 | "no one is getting stuck" | Stuck: an active item with no UPDATE for 45 min while agent-deck shows the owner waiting or idle earns up to two NUDGEs, then a digest line. Deaf: the roster shows direct mail waiting 15 min. John hears only if someone is blocked. | First NUDGE at 45–60 min, the second at 75–105. John only through a blocked dependency. | Workers that are busy but stuck. Blocks nobody posted. A stuck worker who blocks no one waits for the digest. | Partly |
| 6 | "no one is being starved for work" | No open item and idle for 20 min earns an OFFER of up to 3 backlog items, at most every 2 h. An empty backlog with 2 or more starved workers becomes a Q line. | 20–45 min | Works only while John keeps the backlog filled. One starved worker with an empty backlog waits for the digest. | Partly |
| 7 | "questions ... proactively surfaced to the Agent Deck conductor" | The steward ends its turn with `Q <topic>-<n>: ...?`. The director's scanner pages Discord with a mention and turns on indicator 1. John answers in the steward's pane. Protected questions stay in the worker's own pane. | Needs-John items: mean ≈12 min, worst ≈21. Unsettled topic choices: about 30 min more. Questions asked only in mail: ≈2–3 h on the four-rung ladder. | The director's own session never hears. One Q visible per steward at a time. No answer from the phone. Rests on heuristics edited today. | Partly |

Score: none met, seven partly met, none missed. The parts each goal misses are listed in the notes below.

### Notes, with evidence

**Goal 1, the pattern.**
- The ledger is the picture John asked for: items, owners, states, open questions, and decisions marked by who decided (B §7). It lives on hera. Workers on andoria-08 and delphi-3bd4 cannot read it; they see only the quiet digest post.
- To compare topics, John must open each steward's pane or ledger. johns-view §2.2 proposed one STATUS line per topic in a shared directory (`cat topics/*/STATUS`). B has the line but writes it only to the pane.
- No digest says how much the steward did not see:
  - `cc: []`;
  - sessions that never joined;
  - v1 CLI senders (the director still reads and sends with the v1 CLI; Verified: its `AGENTS.md:32`);
  - git and shared files.

  B §8 admits the bypass. A digest that cannot say "I saw N of M" lets John assume full coverage (risks R23).

**Goal 2, duplication.**
- B §6's Duplicate row reads CLAIM posts, not copies. So detection is as good as CLAIM compliance, which is etiquette.
- Delay: one hop of 2–5 s, plus the 2 s settle, plus a wait for the next 15-minute tick (mean 7.5 min), plus a 1–3 min turn. That gives a mean of ≈10 min and a worst case of ≈18.
- The fleet's one recorded duplicate crossed topics: the rollout session's Vulcan branch against nixos's module (director §1.9). B would not have caught it.
- A request mailed to one worker reaches five bystanders as a byte-identical copy that still says `To: muse-spark`. MetaGPT commented out a "know but do not react" copy to its team leader because the leader misread it (prior-art §2 and §4.2). B's worker skill never says not to act on, or answer, a copy addressed to someone else.

**Goal 3, conflict.**
- In ingest, the main conflict is over hardware: johns-view's sample purpose says "one model per GPU host at a time". A `writes:` path cannot express a GPU hold. Two other cases would also pass unseen (director §1.9):
  - hera's load of 70 to 90 from another session's build, which timed out a gate twice;
  - the obr comment-id collision (812 and 813), which is an id space, not a path.
- Correction comes only after both claims exist. B's decision 1 argues that delivery takes seconds and work takes minutes. Whether a ring reaches a busy worker before its turn ends is Unverified (operations, unverified item 5).

**Goal 4, purpose.**
- B rightly never lets mail stop work: pointing a drift out and recording it is the most a peer can legitimately do.
- But the check fires only on new CLAIMs.
- The purpose workers can read is at most 200 characters, folded to one line (v2 §6.4; v2-fit §3.3). v2 has no command to change a channel's marker (johns-view §5).

**Goal 5, stuck.**
- Spotting deaf workers is B's most concrete win.
  - At 10:44, 55 files waited for 11 recipients (Verified: names only). Five of them were for dsv41f, an ingest worker that agent-deck shows waiting (Verified: `agent-deck remote sessions andoria-08 --json`).
  - B's roster (P2) would name it in the next digest.
- The Stuck row requires the owner to be waiting or idle. A worker running in circles for hours is never nudged unless its traffic trips the Loop row. operations §7 had a rule for this (two hours running with no update); B dropped it.
- The director's 30-minute stall page still covers a frozen pane, once per session lifetime (Verified: `record_progress`). B adds stalls at the level of work items, not panes.

**Goal 6, starved.**
- The director does not treat idling at an empty prompt as needing attention (director §1.10), so B is the only cover here.
- It works only while John's backlog has items. B never invents work, which is right (prior-art §4.3; risks R19).

**Goal 7, questions.**
- **The path is proven.** The scanner paged `local/nix` and `andoria-08/glm-5.2` today, with a median of 69 s between passes and at most 160 s seen (director §1.3, §1.5).
- **"To the Agent Deck conductor."** The scanner is the director's deterministic half. Its Codex session hears nothing unless John adds the optional line to `AGENTS.md` (B §4). John's intent, that he need not notice questions himself, is met. His wording is met only through the scanner.
- **Mail to the director.** It now moves faster than it did this morning: at 10:44 `conductor-director`'s presence file was two minutes old and no mail waited for it (Verified). It still has no bound, and B is right not to depend on it.
- **Questions that live only in mail.** B's Unanswered row climbs four rungs: the recipient, the likely answerer, one post, then John. If each rung waits the 30 minutes that operations §8 uses, John is paged after about 2 hours, plus up to four 15-minute tick waits. B does not say how long each rung waits.
- **No answer from the phone.** The director gained a Discord reply path today, but only for "boxed, numbered Claude single-choice menus"; "custom text" is unsupported (Verified: `DISCORD-QNA.md` at 10:36, `discord_qna.py` at 10:43). B's free-text Q line can be paged, but John cannot answer it from Discord.
- **The page channel is noisy.** The 10:43:39 pass paged a shell fragment from `local/pi` that contained a `?` (Verified: the tail of `task-log.md`). Steward pages compete with false alarms in the same channel.
- **No contract.** B's path rests on heuristics in `monitor_sessions.py`, a file B does not own, which was edited at 10:34 today and has no tested rule for steward lines (Verified: `ls -la`).

## 2. Cost

### 2.1 Inputs

1. **Volume:** B's own figure. Seven workers each send 3 direct messages and 2 status posts an hour, so 21 copies and 14 posts, or 35 items an hour, reach `+ingest`. This is unmeasured (B §9).
2. **Size:** waiting messages average 1,074 bytes (risks E6). With the frame that is about 1,375 to 1,625 bytes, or about 375 tokens at 4 bytes a token. The tokenizer ratio is Unverified.
3. **Fixed overhead:** each request in a Claude Code session here re-reads about 47.3k tokens of fixed overhead (risks E7; agent-deck's figure for this session).
4. **Prices, Opus 5.5 list (Verified):**
   - input $4/MTok, output $20/MTok;
   - cache reads $0.20/MTok (0.05×);
   - cache writes $5/MTok with the 5-minute lifetime (1.25×), or $8/MTok with the 1-hour lifetime (2×);
   - a read refreshes the lifetime.

   Source: the claude-api reference bundled with Claude Code 2.1.312, cached 2026-10-06, `shared/prompt-caching.md` lines 123 to 153 and its model table. Which lifetime Claude Code uses for these sessions, I did not check.
5. **Turn length:** a steward turn takes 1 to 3 minutes. Assumed.

### 2.2 Messages per worker message

| Per direct message between two joined members | v2.0, no steward | B as written | B with a copy channel only the steward reads (issue 1) | A relay, for comparison |
|---|---|---|---|---|
| Synced files | 1 | 2 | 2 | 2 |
| Deliveries to a model | 1 | 7: the recipient, the steward and 5 other members | 2 | 2, plus the steward re-typing the text |
| Steward model requests | 0 | 0 at send; about 1.1 when amortized (10 a tick ÷ 9 items) | the same | at least 1 per message, plus receives |
| Delivery latency added | 0 | 0 | 0 | 16 s plus a steward turn: 31–76 s between Positron hosts with fast hops (risks R1) |

Other traffic:

- **Status post:** 1 file and 7 deliveries (six members and the steward), the same in any channel design.
- **Steward mail:** 2 files and 7 deliveries each, only by exception, capped at 6 a turn and 20 an hour.
- **Synced files:** B counts 56 an hour against 21 without a steward. That is cheap for Syncthing.

### 2.3 The steward's tokens: per tick, per message, per digest

**One tick, every 15 minutes while any item is open (about 9 items), following B §5's turn steps:**

| Step | Requests | Context added |
|---|---|---|
| pgrep both background tasks | 1 | 0.05k |
| read the ledger (under 8 KB) | 1 | up to 2k |
| receive until nothing waits (9 × ≈1.5 KB exceeds the 12,000-byte budget) | 2 | 3.4k of items, 0.2k of list lines |
| `channels` roster | 1 | ≈1.3k |
| `agent-deck remote sessions` for two hosts, filtered with jq | 2 | ≈0.8k |
| save the ledger | 1 | ≈1.5k of output |
| restart the tick | 1 | 0.05k |
| final status line, and any Q line | 1 | ≈1k, with the visible reasoning |
| **Total** | **≈10** | **≈10k** |

**Growth.**
- B's "about 13k tokens of context" an hour is 35 items × 375 tokens: bodies only.
- With B's own steps, the steward grows about 40k tokens an hour (4 ticks × 10k), three times B's figure.
- Without compaction its context is C(h) ≈ 47k + 40k·h: 87k after 1 hour, 207k after 4, and 527k after 12.
- Every request re-reads C. I did not check the autocompact threshold that would bound C for these sessions.

**The cost of one tick:**

| Context C | 5-minute lifetime: each 15-minute tick starts cold | 1-hour lifetime: warm |
|---|---|---|
| 127k (about 2 h of uptime) | First request rewrites C: 127k × $5/M = $0.64. Nine reads: 1.14M × $0.20/M = $0.23. New content: 10k × $5/M = $0.05. Output: ≈5k × $20/M = $0.10. **≈ $1.02.** | Ten reads: 1.27M × $0.20/M = $0.25. New content: 10k × $8/M = $0.08. Output: $0.10. **≈ $0.43.** |
| 500k | $2.50 + $0.90 + $0.05 + $0.10 = **≈ $3.55** | $1.00 + $0.08 + $0.10 = **≈ $1.18** |

**Per message (9 a tick):**
- About 1.1 steward requests and about 141k tokens processed (1.27M ÷ 9). About 1.1k of that is new context.
- At C = 127k that is ≈ $0.11 with the 5-minute lifetime, or ≈ $0.05 with the 1-hour lifetime.
- B's table says the steward makes "0" requests per message "at send". That is true at the moment of sending, but amortized it is about one request per message at B's volume.
- A tick costs nearly the same for 1 item as for 9. At 5 items an hour the cost per message rises to ≈ $0.70 (4 ticks × ≈ $0.90 ÷ 5).

**Per digest:**
- Inside a tick, a digest costs about 3 extra requests (print it, write `digest.md`, send the quiet post): 3 × 127k × $0.20/M + 1.2k × $20/M ≈ $0.10.
- Run as its own cold turn, it costs ≈ $0.75.
- Each quiet post also adds about 450 tokens to each of seven workers' contexts.

**Per day:**
- 12 busy hours (48 ticks) plus 12 quiet hours at 25 minutes (about 29 ticks) make about 77 ticks and 770 requests.
- At a mean context of 127k, that is ≈ $75 a day with the 5-minute lifetime, or ≈ $33 with the 1-hour lifetime.
- At a mean of 300k, it is ≈ $170 a day with the 5-minute lifetime.
- The overnight ticks alone (23:00 to 07:30, about 20 ticks at ≈ $0.87) cost ≈ $17 a night with nothing to do.

### 2.4 What the copies cost the workers

**What each worker reads, per hour:**
- copies not addressed to it: 21 − 3 of its own − 3 addressed to it (skipped as siblings) = 15;
- 12 status posts from others;
- about 1 steward intervention.

That is about 28 items, or ≈10.5k tokens an hour (B says 9k).

**Re-reading.** These tokens stay in the worker's context, and every later request re-reads them.
- At R requests an hour (60 assumed, not measured), after h hours the copies cost 10.5k × h × R tokens an hour.
- At h = 4 that is ≈2.5M cached tokens an hour per worker, or ≈17.6M for all seven.
- At $0.20/MTok that is ≈ $3.5 an hour, about as much as the steward.
- They also add about 4 receives an hour per worker (28 × 1.5 KB ÷ 12,000 bytes).

**With the fix.** If only the steward reads the copies (issue 1), each worker reads 12 posts and the interventions, about 5k tokens an hour, and the copies cost the workers nothing.

### 2.5 Latency per hop, under v2's expected latencies

| Hop | v2.0 (design §6.10, unmeasured) | Under B | Note |
|---|---|---|---|
| Worker to worker, same host | under 1 s, plus 2 s settle | unchanged | the same `send` writes the copy |
| Worker to worker, hera to a Positron host | a few seconds | unchanged | one hop measured once at 2.07 s (risks E1) |
| Worker to worker, between two Positron hosts | twice that, through hera | unchanged | |
| A copy reaching the steward's attention | — | a few seconds, plus 0–15 min until the tick (mean 7.5; 0–25 when idle), plus a 1–3 min turn | the steward subscribes quietly, so posts never ring |
| The steward's correction reaching an idle worker | — | a few seconds, plus the 2 s settle, plus a wake (≈3 s, assumed in risks) | a busy worker sees it when its turn ends (Unverified) |
| The steward's Q line reaching John's phone | — | one scanner pass (median 69 s, at most 160 s seen; director §1.3), plus Discord | |
| John's answer reaching the worker | — | the steward's next turn (1–3 min), plus a few seconds | a protected confirmation needs a second page, from the worker's pane |

### 2.6 Sessions, approvals and burden, against doing nothing

**Sessions.** One new Claude Code session per topic, on hera: two, for ingest and the Nix configuration. The director stays as it is.

**John's approvals, in order:**

1. **The design**, including:
   - copies instead of a relay;
   - the exception to v2 §6.4, whose subscription list "is display only and is never read back as authority" (Verified). Under B, a recipient's peer-writable member record decides whether a copy is made.
2. **The v2.1 plan.** mail-dev builds about 110 lines and 15 checks, then a fess audit.
3. **The pushes:** the agent-mail repositories and the Gitea pin (decisions 4 and 25).
4. **The nix-config publish and a rolling activation:**
   - hera, through pi;
   - the Positron flake, then its four hosts in turn;
   - vulcan, pallas, athena and clio.

   Decision 27's standing approval covers "Phase C" only (Verified), so v2.1 needs a new one.
5. **If `:cc` pins are used:** an amendment to decision 26 ("nothing pinned"), and a second publish and activation once every host runs v2.1.
6. **Optional:** the stall exemption and the `AGENTS.md` line in the director's files.

**Per topic:** John runs `channel create` and the launch line, gives the purpose and backlog, and types a join instruction in each worker's pane (seven for ingest).

| | Doing nothing | B |
|---|---|---|
| New sessions | 0 | 1 per topic (2 for ingest and Nix) |
| Steward turns | 0 | about 77 ticks a day each, plus rings |
| One-time work | 0 | v2.1 build and audit by mail-dev; a push, a publish and a rolling activation on every Mailbox host; perhaps a second rollout for pins |
| John's approvals | 0 | 2 design decisions (copies, and the member-record exception), 3 or 4 publish and activation approvals, optional director edits |
| Setup per topic | 0 | the channel, the launch line, the purpose and backlog, 7 pane instructions for ingest |
| Pages | today's (14 between 09:44 and about 10:20; director §1.5) | the same, plus steward Q lines; B removes none |
| Reading | Discord | plus the ledger, `digest.md` and the roster, when he wants |
| Upkeep | none | refill backlogs; revive a steward after its one-time stall page; re-join workers re-created on another personal host |
| What John gets | late discovery: the Vulcan duplicate, the obr ids, hera's load; 55 unread files for 11 recipients today | announced duplicates and path conflicts within ≈10–18 min; deaf members named daily; questions asked only by mail are paged |

## 3. Simplicity

### 3.1 What can go without losing a goal

1. **P1c, the sibling skip** (about 12 lines and 2 checks). It is not needed once the copies go to a channel only the steward reads.
2. **Explicit `cc` names and mentions on posts, in P1a.** No goal uses them.
   - A boolean opt-out, `copy: false`, can replace `cc: []`, and the tool schema stays scalar.
   - Today `check_args` accepts only strings, booleans and integers (Verified). B's list would need a new argument type in the static schema and in the checker.
3. **The `NAME:cc` pins, in P1b.** A runtime `subscribe` is enough for a pilot. Pins cost an amendment to decision 26 and a second rollout.
4. **Reading the ledger every turn.** The ledger is already in context. Read it only after a restart or compaction, a case B §5 already lists. That saves about 2k tokens a tick.
5. **The 25-minute idle tick.** While nothing is open, the steward can re-subscribe with `wake` and stop ticking: posts are rare then, and subscribing again changes only `wake` (v2-fit §1). This needs John to exempt `*-steward` titles from the stall page, one line in his file. It saves about 29 ticks a day.
6. **The quiet digest post every two hours,** if workers do not need it. John reads `digest.md`, and each post costs seven readers.

Together, items 1 and 2 shrink P1 from about 82 lines to roughly 40 (my estimate), and the schema stays scalar.

### 3.2 What is missing that a goal needs

1. **Goal 1:** a STATUS line per topic, kept in a file John can read across topics.
2. **Goal 1:** a coverage line in every digest, naming members without `cc`, topic sessions that never joined, and copies the caps dropped (risks R23).
3. **Goal 3:** `holds:` and `ids:` fields in a CLAIM, checked like `writes:`.
4. **Goal 5:** a rule for running without updates: one NUDGE after two hours of running with no UPDATE.
5. **Goal 7:** one bounded clock for questions asked only by mail. John is paged within 45 minutes of the first missed answer if anyone is blocked.
6. **Cost:** a context budget for the steward.
7. **Goal 7:** a tested rule in the scanner for steward Q lines.
8. **Goal 2:** a line in the worker skill saying that a copy addressed to someone else is information only, never something to act on or answer. This is needed only if workers keep reading copies.
9. **Who owns Nix coordination.** The director already keeps ad hoc Nix records, `mcp_plan_coordination` and `pending_triggers` (director §1.6). John named Nix as the second topic, and its steward would overlap those records until he decides (director, question 4).

## 4. The critical path of agent-mail 2

**v2.0 itself: off the path.** B needs nothing from v2.0 before Task 25, and its compatibility claims hold (Verified in v2 HEAD 9de4b43):

- `check_args` refuses an unknown argument, which the server answers with −32602.
- `member_at` validates only `v`, `host`, `subs` and `checked`, so extra keys pass through.
- `setting_value` refuses any pin but `NAME:wake` with `bad_setting`.

**Caveat 1: Task 22.** B §1 and §4 list uncommitted Task 22, part of C2 on v2.0's own path, for skill text. That text describes `cc`, which a v2.0 server refuses. Task 22 should change nothing.

**Caveat 2: the pilot's own path is long.** Task 19 has not landed: v2's HEAD is 9de4b43, with an uncommitted diff (Verified). The pilot then waits for all of these:

1. C2 (Tasks 20 to 24), with John's approvals host by host;
2. Task 25;
3. a v2.1 plan, build and audit;
4. a push and a nix publish;
5. a rolling activation;
6. perhaps the pins.

All of it sits in mail-dev's queue.

**Caveat 3: pins and the shared NFS home.** config.json lands once in the Positron NFS home and serves four hosts that switch in turn (plan Task 24, Verified). A `:cc` pin there would make the settings fail on every host not yet switched. B already holds pins back; keep that.

**Result:** B stays off v2.0's critical path but sits on the pilot's.

## 5. Issues

1. **[High] Every member reads every copy.**
   - *Scenario:* lfm3 mails muse-spark a request. Under B's defaults every worker subscribes to `+ingest` with `cc`, so the byte-identical copy reaches glm-5.2, dsv41f, pipeline, glm-5.3-flash and kimi-linear at their next receive. That makes:
     - 7 deliveries to a model instead of 1;
     - about 10k tokens of new context an hour per worker (section 2.4);
     - one poisoned or mistaken message in seven contexts that all run in dangerous_mode (risks E13). v2 §6.4 refused to make top-level mail a broadcast because "it would make every message the worst case for injection" (Verified).

     A bystander may also answer or act on a copy that says `To: muse-spark`, the misreading that led MetaGPT to drop its leader's "know but do not react" copy (prior-art §4.2). The mechanism meant to expose duplication then creates it.
   - *Smallest fix:* send copies to a channel only the steward subscribes to, for example `+ingest-cc`, created by John and named in the P1b flag. Workers subscribe to `+ingest` for CLAIM, DONE and digests. P1c then goes away.
2. **[High] The cost is understated about threefold, and it grows with uptime.**
   - *Scenario:* with B's own turn steps, the steward adds about 10k tokens each tick, not 13k each hour. It makes about 10 requests a tick. If the cache lifetime is 5 minutes, it rewrites its whole context every tick: ≈ $1 a tick and ≈ $75 a day at a 127k mean context, and ≈ $3.55 a tick at 500k. Nothing bounds the context, and the 25-minute tick runs all night (≈ $17 a night with nothing to do).
   - *Smallest fix:*
     - drop the ledger read from every turn, and filter agent-deck's JSON;
     - stop the idle tick, using a wake subscription while idle plus a one-line `*-steward` stall exemption in John's file;
     - set a context budget: John clears or restarts each steward after the 07:30 digest, unless Claude Code's autocompact keeps it near 150k (I did not check).
3. **[Medium] The pilot waits for code that does not power the duplicate and conflict checks.**
   - *Scenario:* before the first topic starts, v2.0 lands, then C2's rollout, Task 25, and the v2.1 plan, build, audit, push, publish and rolling activation. Yet B §6's Duplicate, Drift, Stuck and Starved rows, and most of the Conflict row, read CLAIM, UPDATE and DONE posts. Workers must remember those exactly as in a zero-code design. P1, which is about 82 of the 110 lines and 12 of the 15 checks, feeds the pattern view, the Loop row and unanswered requests.
   - *Smallest fix:*
     - pilot `+ingest` on v2.0 with B's own fallback (its decision 7);
     - ask mail-dev for P2 first (about 25 lines and 3 checks);
     - decide on P1 after a week of measuring how often workers post CLAIMs and NOTE lines.
4. **[Medium] Conflict detection misses holds on resources and id spaces.**
   - *Scenario:* muse-spark claims an evaluation on andoria-08 GPUs 0 to 3 while pipeline's conversion holds them. Both write different paths, so no overlap fires. The same goes for hera's load of 70 to 90 from another build and for the obr comment ids 812 and 813 (director §1.9).
   - *Smallest fix:* a CLAIM gains `holds: <host>:<resource>` and `ids: <space> <range>`, checked like `writes:`. This is skill text only.
5. **[Medium] A question asked only by mail can take two to three hours to reach John.**
   - *Scenario:* muse-spark posts a QUESTION nobody answers. If each of the four rungs waits the 30 minutes that operations §8 uses (B does not say), John is paged after about 120 minutes, plus up to four 15-minute tick waits.
   - *Smallest fix:* one clock. Ask the likely answerer at 30 minutes. Page John at 45 if anyone is blocked; otherwise list the question in the digest.
6. **[Medium] The question path has no contract and no answer from the phone.**
   - *Scenario:* pages depend on the heuristics in `monitor_sessions.py`:
     - the last 15 lines after the last bullet;
     - the lowest line with a `?`;
     - about 60 exclusion rules.

     The file was edited at 10:34 today. The pass that would page a steward paged a shell fragment from `local/pi` at 10:43 (Verified). John can answer only by typing in the steward's pane, because the director's new Discord replies cover numbered Claude menus only (Verified: `DISCORD-QNA.md`).
   - *Smallest fix:* ask John to add a tested scanner rule for `*-steward` titles, `^Q [a-z0-9.-]+-\d+: .*\?$` (johns-view §3.3). Optionally, let a steward ask through a numbered menu when John is away from his desk, accepting that the menu holds the steward's turn open until he answers. I expect it does, but I did not check.
7. **[Medium] Copies spend the sender's post budget.**
   - *Scenario:* v2 allows each sender 20 posts per channel per hour, counted on each host (Verified: `send_window`). Suppose a worker mails members 18 times in an hour during a handoff; the rollout session sent 21 messages within 15 seconds this morning (risks E11). Its 21st post, perhaps its own DONE, is refused as `rate_limited` until the window clears, and later copies vanish with only a warning.
   - *Smallest fix:* count copies in a window of their own, and never let them block a deliberate post.
8. **[Low] Task 22 is listed among the tasks B touches.**
   - *Scenario:* B §1 and §4 name uncommitted Task 22 (in C2) for skill text. Its worker text describes `cc`, which a v2.0 server refuses with −32602 (Verified: `check_args`).
   - *Smallest fix:* state that Task 22 changes nothing, and ship both skills in the v2.1 plan's own nix task.
9. **[Low] A list argument where a flag would do.**
   - *Scenario:* `cc`, taking up to 3 names or channels, and mentions on posts both need a list type. `check_args` checks only strings, booleans and integers (Verified), and no goal needs either feature.
   - *Smallest fix:* one boolean `copy`, on by default for members. Drop explicit names and mentions on posts.
10. **[Low] Pins add a decision change and a second rollout.**
    - *Scenario:* `NAME:cc` pins amend decision 26, and they must wait until every host runs v2.1. v2.0 refuses them with `bad_setting`, and config.json is shared on the Positron NFS home while hosts switch in turn (Verified: `setting_value`; plan Task 24).
    - *Smallest fix:* leave pins out of v2.1. A runtime `subscribe` is enough until the pilot proves a topic is worth keeping.
11. **[Low] Workers that are busy but stuck go unseen.**
    - *Scenario:* kimi-linear loops for three hours while agent-deck shows it running. B's Stuck row requires the owner to be waiting or idle, so no NUDGE goes out.
    - *Smallest fix:* restore operations §7's rule. A worker running with no UPDATE for two hours gets one NUDGE asking it to post an UPDATE when it next stops.
12. **[Low] Replies that wake the steward, and answers it cannot match.**
    - *Scenario:* a NUDGE answered by direct mail rings the steward, costing a full turn of about 8 requests over its whole context to read one line. A reply without `in_reply_to` (from the CLI, from v1, or from a forgetful model) leaves the request looking unanswered and triggers more nudges.
    - *Smallest fix:* workers answer a NUDGE with an UPDATE post. The steward also counts as an answer any reply from the recipient to the sender within the window.
13. **[Low] Sessions in two topics get two copies.**
    - *Scenario:* pi belongs to both ingest and Nix (johns-view §7). When it mails a session that is also in both, the copy goes to both channels and both stewards weigh it.
    - *Smallest fix:* copy only to the sender's first-listed topic, and to others only by explicit choice.
14. **[Low] Mixed versions deliver twice.**
    - *Scenario:* a v2.1 sender on hera copies mail to a recipient on a Positron host still running v2.0, which delivers both the direct file and the post. B calls the duplicate harmless, but a model handed the same request twice may act on it twice.
    - *Smallest fix:* turn on copying for a topic only after every host its members use has switched to v2.1.

## 6. What B does better than A and C would

1. **Visibility that does not depend on memory.** The sender's own `send` writes the copy, and leaving it out takes a deliberate `cc: []`. A depends on a NOTE line for every exchange, and C on ASK and NOTE posts (A §1; C §1 and §3).
2. **Same-host pairs become visible.** Four ingest workers share delphi-3bd4 (Verified). A direct file claimed there within seconds may never appear on hera (risks R23). B's copy stays in `+ingest` for 30 days.
3. **A roster of waiting mail, inside agent-mail (P2).** It shows names only, and anyone can run it on any host without a session name.
   - Two explorers asked for it independently (johns-view §8, item 1; director §6, item 6).
   - It serves the whole fleet: 55 files wait for 11 recipients today (Verified).
4. **No second script and no second reader identity.** C needs `agent-steward`, reading as `ingest-steward.ledger` (C §1 and §2).

## 7. The three questions John must answer for B to work

1. **Copies.** Is "through the conductor" satisfied by a copy that the sender's own tool writes into the topic channel while delivery stays direct? And do you accept that a peer-writable member record decides whether that copy is made? That is an exception to v2 §6.4's rule that the subscription list "is display only and is never read back as authority".
2. **Questions.** Is a Discord page from the director's scanner what you meant by "surfaced to the Agent Deck conductor"? Under B you answer in the steward's pane, and protected confirmations stay in the worker's pane. Or must the director's own session know about each topic question?
3. **Build order.** Do you approve building the v2.1 primitives after Task 25? That means about 110 lines, two fleet rollouts, and roughly $33 to $75 a day per steward at list price. Or should the first topic pilot on v2.0 conventions now, with only the roster requested, and copies built only if workers prove poor at posting their claims and notes?

## Evidence log

**Verified today, by me:**

- **v2, in `/Users/johnw/Products/agent-mail-v2`:**
  - HEAD is 9de4b43 (10:29 PDT), with an uncommitted diff of +88/−4 lines in `agent-mail`.
  - `check_args` checks scalar types only.
  - `member_at` validates only `v`, `host`, `subs` and `checked`; its docstring says member records are "only ever displayed or warned about".
  - `setting_value` accepts only `NAME` or `NAME:wake` pins.
  - `send_window` keeps a per-sender window in host-local state: 20 posts per channel an hour, and a duplicate refused when both target and body hash match within 10 minutes.
  - `POST_LIMIT` is 8192 and `POSTS_PER_SENDER` is 10.
  - The send header set contains no `Cc:`.
  - `render` prints each item's text inside its frame. That the text is the whole file, headers included, comes from v2-fit §1.
- **v2 design:** §4's anti-goals; §6.4, including "display only and is never read back as authority" and the injection reason for not making top-level mail a broadcast; §6.7, including the 8 KB cap on `channels`.
- **v2 plan:** Tasks 19 to 25; Task 24's note that config.json is shared on the Positron NFS home.
- **Decisions:** 4, 9, 25, 26 and 27; decision 27 covers "Phase C".
- **The director's files:**
  - modification times today: `monitor_sessions.py` 10:34, `discord_qna.py` 10:43, `DISCORD-QNA.md` 10:36;
  - `DISCORD-QNA.md`: numbered menus only, custom text unsupported, a trial on hera;
  - the question scan in `monitor_sessions.py` at 745 to 925;
  - `record_progress`: one stall page per session lifetime, after 30 minutes;
  - `pane_has_live_work`;
  - the stall exemption by the `-review` or `-conv` suffix, now at lines 680 to 681;
  - the `task-log.md` pass at 10:43:39, which paged `local/pi` with a shell fragment.
- **The mailbox, names only:**
  - `~/Mailbox` at 10:44: 55 files waiting for 11 recipients, 5 for dsv41f;
  - `.agent-mail-presence`: `conductor-director` touched at 10:42.
- **agent-deck:** `agent-deck remote sessions delphi-3bd4|andoria-08 --json`: all seven ingest workers run Claude Code; pipeline and dsv41f are waiting.
- **Prices and cache lifetimes:** the claude-api reference bundled with Claude Code 2.1.312, cached 2026-10-06, as cited in section 2.1.

**From sibling reports, not re-checked:**

- risks: E1 (one hop of 2.07 s), E6 (message sizes), E7 (47.3k tokens of fixed overhead), E11 (bursts), E13 (dangerous_mode);
- director: §1.3 (scanner cadence), §1.5 (pages today), §1.9 (the Vulcan duplicate, the obr ids, hera's load), §1.10;
- johns-view §3.3 and §7; operations §7 and §8; prior-art §4.2 (MetaGPT's dropped copy); v2-fit §1 and §3.3.

**Unverified:**

- which cache lifetime Claude Code uses, and its autocompact threshold;
- requests per hour for a worker (I assumed 60);
- steward turn length;
- whether a ring reaches a busy worker before its turn ends;
- that a `sleep` exit wakes Claude, which B also marks Unverified;
- whether a pending numbered menu blocks a steward's background wakes;
- the fleet's billing basis;
- message volume, which nobody has measured.
