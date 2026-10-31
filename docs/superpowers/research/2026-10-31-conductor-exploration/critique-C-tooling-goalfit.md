# Critique of candidate C (steward tooling): goal fit and cost

Critic for the GOAL FIT AND COST lens. Written 2026-10-31 at about 10:55 PDT on hera, read-only.

I read the candidate (`design-C-tooling.md`) in full. Then I read:

- the brief;
- the six exploration reports;
- the v2 design §§4, 6.8 and 6.10;
- the v2 plan: scope, Global Constraints, Exact copy, Tasks 19 to 25 and Needs John;
- decisions 1 to 27;
- the cost sections of candidates A and B.

**Marks.**

- **[V]**: I checked it today, at the place named.
- **[V: report]**: a sibling explorer checked it today and I did not repeat the check.
- **[U]**: documented or reported, but not checked.
- "Assumed": an input to an estimate.

**Line numbers.** v2 line numbers refer to `git show HEAD:agent-mail` at 9de4b43. The candidate cites a30f782, one commit earlier. Every mechanism it cites is at, or within two lines of, the number it gives. The worktree also holds an uncommitted change, which I did not read.

## Verdict: fixable

C has the right shape for John's goals, at the lowest model cost of the three candidates:

- The steward stays off the delivery path.
- A script, reading under a second name, files every post and keeps the ledger, so the record survives compaction.
- The model wakes only to judge.
- Questions reach John through the director's scanner, a path that works today.
- It needs no change to agent-mail before v2 lands.

But as written it does not deliver "nobody stuck" or "nobody starved" on its own pilot:

- **Idle workers look busy on the pilot hosts.** The ingest workers' hosts, andoria-08 and delphi-3bd4, run agent-deck 1.16.29. There, a Claude worker whose background shell is its mail watcher reports `running/background-work` [V: the catalog at 17:47Z, and the fork's source]. So C's "waiting or idle" tests never fire on those hosts.
- **No latency bound.** The candidate does not say which detections wake the model. Either drift, stuck and starved judgements have no latency bound, or the cost estimate leaves out one turn per claim.
- **A fragile cost claim.** "About 2 turns and 0.8M tokens an hour" holds only with few exceptions and a young context. The strict grammar and a keyword list that matches routine ingestion posts will push exceptions up.
- **A costly home.** Placing the tool in the agent-mail repository and package puts every steward release on agent-mail's shared nix pin, and on John's approval queue.

Each gap has a small fix (§5), and none needs a different architecture.

## 1. Goal by goal

| # | John's goal | How C delivers it | Delay | What it misses | Rating |
|---|---|---|---|---|---|
| 1 | See the overall pattern | The tool files every post into `state.json` and regenerates STATUS, LEDGER.md, DECISIONS.md and QUESTIONS.md. `agent-steward status` prints one line per topic. A digest comes at 07:30 and on request. | About 6 s after a post reaches hera: hop 2 s, settle 2 s, detection ≤ 1 s, tool 1 s. Catalog fields refresh every 2 min (local) or 5 min (remote). John sees it when he looks, or at 07:30. | Work that nobody claimed. The content of unnoted direct mail, which appears only as a coverage count. Real repository state: DONE is only asserted. Cross-topic dependencies. Nothing reaches John's phone except questions. The director's model learns nothing unless John adds the AGENTS.md pointer. | Partly |
| 2 | See duplication | A new CLAIM is matched against open items and items done in the last 30 days, by rarity-weighted tokens (score ≥ 0.5), "of the same kind of work". A match is an exception: the model judges it and sends DUPLICATE or ALREADY-DONE. | About 30 to 80 s after the second claim: 5 s to hera, 1 s in the tool, 3 s to wake, a 15 to 60 s turn, and about 8 s delivery (wake and turn time assumed) | A duplicate where one side never claimed: the handoff's Vulcan case, where nixos built its own module while a prepared branch waited. Duplicates across topics. Paraphrases with no shared rare token. The grammar has no field for "kind of work". | Partly |
| 3 | See conflict | Overlapping `writes:`: a host path inside another, or one repo@branch with overlapping paths. Wait cycles from `on:` and `needs:`. Two DECISION posts on one key. Words near John's authority. | As for 2 | Resources and id spaces. GPUs and host load are ingestion's likely contention; the worked example's purpose is "one model per GPU host". The fleet's one recorded cross-session conflict was in an id space: obr comment ids 812 and 813 [V: director report, from the handoff]. Writes outside the claimed paths. DECISIONs under different slugs. | Partly |
| 4 | Keep work harmonious to the purpose | TOPIC.md holds John's words verbatim, and the model judges each new CLAIM against them | No bound as written. New claims appear in the next turn sheet, and the model wakes only for exceptions and the 07:30 digest (§1; the §6 Drift row does not raise an exception). On a quiet day a drifting claim waits for the next exception, or until 07:30. | Drift after a valid claim. Unclaimed work. A 160-character claim gives the model little to judge. | Partly |
| 5 | Nobody stuck | NUDGE after 45 min of silence (2 h if busy). An unanswered ASK gets a second pointer at 30 min and becomes an exception at 60. Wait cycles. "Deaf" after 15 min. BLOCKED `on: john` is left to the director's pane scan. | 45 min to 2 h, plus a turn, if NUDGE drafts wake the model; the candidate does not say | Idle workers read as busy on andoria-08 and delphi-3bd4. Both run agent-deck 1.16.29 [V]. glm-5.3-flash and kimi-linear, whose background shell is their mail watcher, read `running/background-work` [V, 17:47Z]. Such a worker counts as busy, so it waits for the 2 h threshold, and "deaf", which needs `waiting`, never fires. Under v2 every Claude worker runs a background doorbell (plan Task 23). | Partly |
| 6 | Nobody starved | An OFFER of up to 3 items after 20 min with no open item while `waiting` or idle. A question to John if the backlog is empty and two workers are starved. | 20 min plus a turn, if drafts wake the model | The same misclassification: on the pilot hosts an idle Claude worker with a watcher never reads `waiting` or idle. No command creates backlog items: `add` and `drop` are not defined, and `init` records only the purpose. The worker section never mentions OFFER. | Not met (pilot as written) |
| 7 | Surface questions proactively to the agent-deck conductor | `ask` writes a page line of at most 120 characters, ending in "?", as the turn's last line. The director's scanner pages Discord with a mention and lights indicator 1. The text stays identical while the question is open, and changes once to "STILL BLOCKED 4h". Quiet hours are 23:00 to 07:30. John answers in the steward's pane. Authorizations stay in the worker's own pane. | 1.5 to 4 min from the steward's decision: a 15 to 60 s turn, plus one scanner pass (median 69 s, longest observed 160 s [V: director report]). A peer question asked only by mail takes at least 60 min by design. Quiet hours can add up to 8.5 h. | It reaches the director's scanner, not the director's model, while John said "the Agent Deck conductor". John should confirm that this meets his intent. After 60 min the ladder has no further rung unless the matter is John's alone. Answering needs a terminal in the steward's pane. | Met, with one gap |

Goals 4 to 6 depend on the wake rule. If the candidate means every new claim and every NUDGE or OFFER draft to wake the model, those delays fall to minutes. The turn count then becomes claims plus drafts plus exceptions per hour, not "about 2" (§2.2).

## 2. Cost

### 2.1 Messages per worker message

| Worker action | Synced files under C | Synced files today (direct mail) | Deliveries under C (7 subscribers and the tool) |
|---|---|---|---|
| State post (JOIN, CLAIM, UPDATE, BLOCKED, DONE, DECISION, LEAVE) | 1 | 0 (not sent today) | 8 |
| `ASK @B` | 2 (the post and a pointer); 3 if unanswered at 30 min | 1 | 8, plus the pointer |
| ANSWER | 1 | 1 | 8 |
| A private, long or urgent exchange | 2 (direct mail and a NOTE) | 1 | 1 + 8 |
| Each exception | 1 or 2 steward notices (OVERLAP goes to both owners) | 0 | 1 to 2 |

**What it adds up to.**

- A compliant worker adds 3 to 5 posts per work item that do not exist today: a CLAIM, 1 to 3 UPDATEs and a DONE.
- Synced files per useful worker message rise by about 1.3 to 2 times, and deliveries by 7 to 8 times.
- Every direct file also syncs a deletion when its reader claims it.

### 2.2 The steward's tokens

**Inputs**

1. **Fixed overhead:** 47.3k tokens per request. That is agent-deck's figure for this session [V: risks E7], assumed similar for a steward.
2. **Growth per turn:** about 3k tokens (assumed). It comes from:
   - a turn sheet of 1k to 3k tokens (the cap is 12 KB);
   - tool calls and their results;
   - visible output;
   - thinking blocks, which current Opus models keep in context [V: claude-api skill, `shared/prompt-caching.md` line 240].
3. **Requests per turn:** 5, the candidate's own figure: the wake, `pgrep`, `turn`, `act` or `skip`, and the restart of `wait`, which carries the final message.
4. **Cache.** The API's default cache lifetime (TTL) is 5 minutes [V: claude-api skill, `prompt-caching.md` lines 123 to 124]. Turns about 30 minutes apart therefore start cold, and the first request of each turn writes the whole context again. Whether Claude Code asks for the 1-hour TTL: I did not check.
5. **Prices, for scale only.** Claude Opus 5.5 costs $4 per million input tokens and $20 per million output tokens. A cache read costs 0.05 times the input price ($0.20 per million). A cache write costs 1.25 times (5-minute TTL) or 2 times (1-hour TTL). [V: claude-api skill, model table cached 2026-10-06, and `prompt-caching.md` "Economics".] Whether John's fleet pays per token or by plan usage: I did not check.
6. **Day shape:** 16 active hours a day, with no compaction during the day. I did not check Claude Code's autocompact threshold.

**Formulas**

- Input-equivalent tokens per turn under the 5-minute TTL: E = 1.25·C + 4 × 0.05·C + 1.25 × 4k ≈ 1.45·C + 5k, plus about 1.5k output tokens.
- Mean context over the day at T turns an hour: C̄ = 47.3k + (16T ÷ 2) × 3k = 47.3k + 24k·T.

**Per event**

| Event | Model requests | Input-equivalent tokens | About $ at Opus 5.5 list prices |
|---|---|---|---|
| Routine post | 0 | 0 | 0 |
| New claim, shown in the next turn sheet | 0 more | about 200 × 1.45 in each later turn: about 4.6k over a day at 2 turns an hour | 0.02 |
| New claim, if each claim wakes the model | 5 | about 143k (C̄ = 95k) | 0.60 |
| Exception turn (C̄ = 95k) | 5 | 1.45 × 95k + 5k ≈ 143k, plus 1.5k output | 0.60 |
| The same turn, if Claude Code uses the 1-hour TTL | 5 | 5 × 0.05 × 95k + 2 × 7k ≈ 38k | 0.18 |
| 07:30 digest, fresh context (C = 50k) | 4 to 5 | 1.45 × 50k + 3k ≈ 76k, plus 1k output | 0.32 |
| 07:30 digest, yesterday's context kept (C = 143k) | 4 to 5 | 1.45 × 143k + 3k ≈ 210k | 0.86 |

**How the exception rate moves the total** (5-minute TTL)

| Turns an hour | C̄ | Per turn | Per hour | Per 16-hour day |
|---|---|---|---|---|
| 2 (the candidate's figure) | 95k | 143k, about $0.60 | $1.20 | about $19 |
| 4 | 143k | 213k, about $0.88 | $3.50 | about $56 |
| 6 | 191k | 282k, about $1.16 | $7.00 | about $111 |

**What this shows.**

- **At 2 turns an hour the figures agree.** My raw re-read figure, 10 requests × 95k ≈ 0.95M tokens an hour, matches the candidate's 0.8M.
- **The estimate leaves out two effects that move the total.** Every turn starts by writing the whole context to a cold cache. And when exceptions rise, the turn count and the context grow together, so cost rises faster than the turn count.
- **At 6 turns an hour C loses its edge.** C would re-read about 30 × 191k ≈ 5.7M tokens an hour, inside the 2M to 15M range candidate A gives for itself. C's cost advantage holds only while exceptions stay near 2 an hour.

**Why exceptions may exceed 2 an hour on `+ingest`.** I did not measure compliance; these are the reasons:

- **The grammar is strict.** The candidate's decision 6 makes every subject outside the grammar an exception, and the grammar has 11 verbs, slugs, @names, a 160-character limit and field syntax. At 10% of 25 posts an hour, failures alone give 2.5 exceptions an hour.
- **The keyword list matches routine posts.** The list near John's authority includes publish, upload, delete, release and token. Those words appear in routine ingestion posts, and the worked example's purpose says "convert, evaluate, publish".
- **Duplicate scoring is loose.** Rare-token scoring flags every convert/eval pair for one model unless "kind of work" is defined.

### 2.3 The workers' tokens

**The first read.**

- The candidate estimates "about 17k tokens an hour" for seven workers, at 100 tokens per post, counting first reads only.
- With its frame, a one-line post is about 150 tokens: 612 characters, by candidate A's arithmetic.
- So first reads come to 7 × 25 × 150 ≈ 26k tokens an hour.

**Re-reading costs more.** A post stays in each worker's context, and every later request reads it again.

- A worker making R requests an hour, h hours after its last compaction, carries 25 × 150 × h tokens of posts.
- At R = 40 (assumed) and h = 4, that is 40 × 15k = 600k cached tokens an hour. At 0.05 times the input price, that is about 30k input-equivalents.
- For seven workers that is about 210k input-equivalents an hour, about $0.84 an hour: roughly what the steward itself costs at 2 turns an hour.

This applies to every candidate built on a channel, not only to C, but C's estimate leaves it out.

**Waking an idle worker** costs about 1.25 × 150k + 2 × 0.05 × 150k ≈ 200k input-equivalents (about $0.80), assuming a 150k worker context.

- A pointer costs no more than the direct mail it replaces.
- A NUDGE, an OFFER or a notice that wakes an idle worker costs this much.

### 2.4 Latency per hop

**Terms.**

- *h*, one Syncthing hop: 2 s, from one measured sample [V: risks E1]. The v2 design expects "a few seconds" [V: design §6.10].
- *s*, the settle: 2 s [V: v2 `SETTLE`].
- *d*, detection: at most 1 s with kqueue or inotify, up to the 15 s poll.
- *w*, waking a session: about 3 s (assumed).
- *c*, a steward turn: 15 to 60 s (assumed).

| Path | Direct mail today | Candidate C | Added |
|---|---|---|---|
| ASK, both workers on delphi-3bd4 | s + d + w ≈ 6 s | (h+s+d) + 1 + (h+s+d) + w ≈ 14 s | 8 s more; up to 5 min more when pointers are batched (issue 11) |
| ASK, delphi-3bd4 to andoria-08 | 2h + s + d + w ≈ 10 s | about 14 s | 4 s more |
| ASK with slow hops (h = 20 s, polling) | about 20 s on one host, about 60 s across hosts | about 78 s | 58 s more on one host, 18 s more across |
| Direct mail with a NOTE | unchanged | unchanged, plus one send | 0 |
| State post | n/a | in the ledger about 6 s after it reaches hera; peers read it at their next receive | n/a |
| Overlap or duplicate notice | never sent | about 32 to 77 s after the second claim | n/a |
| A steward's question reaches John's phone | n/a | c plus one scanner pass: about 1.5 to 4 min | n/a |

### 2.5 Sessions John runs and watches

1. **Sessions.** One Claude Code session per topic, on hera: two for the topics he named, ingest and nixcfg. Each holds two background shells, the agent-mail doorbell and `agent-steward wait`. There is no daemon and no launchd job.
2. **What he watches:**
   - Discord, in the same channel as today;
   - each steward's pane, when he answers;
   - `agent-steward status`, when he wants the picture;
   - a 07:30 digest per topic, in two panes or two files.
3. **One-time work:** a join line typed in each worker's pane, 7 for ingest and 6 for nixcfg.
4. **A wedged steward gets no stall page.** A steward pane shows two background shells. The scanner's live-work patterns match an "N shells" footer [V: `monitor_sessions.py` lines 1043 to 1064], and those patterns suppress the 30-minute stall page [V: risks E16]. So a steward whose model is wedged, while its shells run, never draws that page. Only a stale `.ledger` member record in `channels` shows the problem.

### 2.6 Approvals John gives

| When | Approval | Why it falls to John |
|---|---|---|
| Design | The candidate's 10 questions: 12 decisions, counting the two-part ones | Policy |
| Build | Where the tool lives: mail-dev's johnw/agent-mail, or a new Gitea repository | Decision 25: a repository is created by push, or by John |
| Build | Each push of the tool | Decision 4 |
| Build | Nix packaging: a nix-config change, its publish, and a hera activation. Inside `pkgs.agent-mail`, the shared pin also moves for every host (§4). | Decisions 4, 7 and 25 |
| Build | The topic-steward skill and the worker section of the agent-mail skill, as a Task 22 follow-up: a nix-config publish, then activation wherever workers run (hera, and each of the four Positron hosts in turn) | Decision 4; plan Task 24's activation list |
| Build | The director's AGENTS.md pointer, and the `*-steward` stall exemption | Only John edits the director's files |
| Policy | The second reader name `.ledger`, and listing file names in `~/Mailbox` | New identity and privacy rules |
| Pilot | agent-deck 1.16.30 or later on andoria-08 and delphi-3bd4 (issue 1) | A change to those hosts |
| Per topic | `channel create`, `agent-deck launch` and the purpose, plus one join line per worker pane | Only John starts sessions and types in panes |
| Runtime | No routine approvals. Each page is a question, not an approval. | n/a |

### 2.7 Burden compared with doing nothing

| | Doing nothing | Candidate C |
|---|---|---|
| Sessions | None new | One more Claude session per topic, always on |
| Code | None | About 2,300 more lines (1,500 of code and 800 of tests): about half the size of agent-mail 2 itself, which is 4,805 lines at 9de4b43 [V: `wc`] |
| Worker protocol | Direct mail | 11 verbs with a slug and field grammar; claim before working; a NOTE after direct mail |
| John's attention | Discord pages for pane questions, errors and stalls | The same, plus steward pages, two morning digests, and answers typed in steward panes |
| Spend | None | About $19 to $111 a day per steward, plus $0.20 to $0.84 an hour for the workers' copies, at Opus 5.5 list prices (§§2.2 and 2.3) |
| Duplicates and conflicts | Found late or never: the Vulcan branch, the obr ids, hera's load (handoff) | Path overlaps and token-matched duplicates within 1 to 2 minutes. Resource and id-space conflicts are still missed (issue 6). |
| Failure handling | n/a | The director restarts a steward that errors, and the skill restarts dead shells at turn start. Nothing restarts a steward that is alive but never woken. |

## 3. Simplicity

### What could be removed without losing a goal

1. **Pi and Codex stewards**, with `due` and the second /loop. The candidate itself recommends Claude Code on hera. Dropping the other harnesses removes:
   - a command;
   - two wake paths that are unverified;
   - the need to renew Pi loops every 7 days.
2. **Pointers.** This is a trade, not a free cut. Suppose a worker asked by direct mail plus a one-line `ASK <slug> @B` post:
   - The tool would still track the ask: ANSWER closes it, and the 30- and 60-minute rungs are unchanged.
   - B would be rung directly, in 6 to 10 s, with no batching.
   - The tool would never send anything without the model, which removes the carve-out in the candidate's decision 7.

   The cost: two sends per ask for the worker instead of one, and the ask's full text no longer sits once on the channel.
3. **The landed-work detector.** It is the duplicate detector run over finished items, so one scorer suffices.
4. **Separate views.** QUESTIONS.md and DECISIONS.md can be sections of LEDGER.md, and `events/` and `batches/` can share one rotation.

### What is missing that a goal needs

1. Idle classification that is correct on agent-deck 1.16.29, or a precondition that every worker host runs 1.16.30 or later (goals 5 and 6).
2. Wake rules: which detections end `wait` at once and which coalesce, so drift, stuck and starved have a bound (goals 4 to 6).
3. A backlog: a command for John to add items, a verb for workers to post work they will not do, and a worker rule for OFFER (goal 6).
4. Claims on resources and id spaces, beyond file paths (goal 3).
5. A detector for active work that has no claim (goals 1 to 3).
6. A rung on the ladder after 60 minutes (goal 7).
7. A bound on the steward's own context, and control of exception noise (cost).
8. A liveness refresh for the ledger name while the topic is quiet (the workers' fallback rule, goal 5).

## 4. Does it stay off agent-mail 2's critical path?

**Yes, for the landing.**

- Every interface the tool uses is in the committed CLI: `read`, `send --in-reply-to`, `subscribe --wake`, `watch --doorbell --forever` and `channels` [V: usage text at 9de4b43, lines 30 to 48].
- CLI `read` takes the reader lock through `hold_reader`, with no background-job check, so the tool can read from a background shell [V: lines 2774 to 2790]. Refusing a Claude background job is the MCP server's rule alone [V: lines 2486 to 2491 and 2650 to 2653].
- The doorbell rings for posts on wake subscriptions, and takes no lock [V: `ringable`, lines 2084 to 2093; `doorbell`, lines 2161 to 2200].
- The pilot waits for Task 25, and the v2.1 requests are optional.

**Two costs land after the landing, on mail-dev and on John.**

1. **The home.**
   - Putting `agent-steward` in johnw/agent-mail and in `packages/agent-mail.nix` ties it to agent-mail's single source record. That record pins the whole repository by rev and narHash, sits in every host's `package-list`, and is "pinned by hand after the self-test passes on Hera and on a Linux host" [V: plan Task 20].
   - Each steward release would then move agent-mail's pin and store path on every host, and need a nix-config publish and an activation, each approved by John (decision 4).
   - It would also ship steward code to all ten hosts. That is the very cost the candidate held against making the tool "a mode of agent-mail".
2. **The `channels` text is not pinned.** An audit change in Task 19 could break the tool's parser. The tool's self-test, run against the real script, would catch it.

## 5. Issues

1. **High. Stuck, starved and deaf go blind on the pilot hosts.**
   - *Scenario.* At about 17:47Z today, `agent-deck remote sessions delphi-3bd4 --json` reported glm-5.3-flash and kimi-linear as `running/background-work`. Their `substate_detail` named the agent-mail watcher ("bash Restart the agent-mail watcher in the background"). lfm3 showed "bash 1 shell" [V].
     - Both Positron hosts run agent-deck 1.16.29 [V: `agent-deck --version` over ssh].
     - The change that stops a leftover background shell from pinning a session at running is commit 8813ec3b (#2733). It first ships in v1.16.30. From then on a shell counts as work only for 3 minutes after its launch [V: fork `internal/tmux/background_work.go`, `internal/session/userconfig.go` lines 4219 to 4232, `git tag --contains`].
     - On hera, which runs 1.16.34, `pi` with its watcher reads `waiting/idle-at-empty-prompt` [V].
     - Under v2 every Claude worker runs a background doorbell (plan Task 23).
     - So on `+ingest` an idle worker never reads `waiting` or idle. No OFFER is sent, "deaf" never fires, and a NUDGE waits 2 h instead of 45 min.
   - *Smallest fix.*
     - Make agent-deck 1.16.30 or later on every worker host a precondition of the pilot.
     - Until then, have the tool treat `running/background-work` whose detail is only a shell (no workflow or agent) as `unknown`.
     - Add today's 1.16.29 JSON to the self-test as a fixture.

2. **High. Nothing says which detections wake the model.**
   - *Scenario.* §1 says the model "wakes for exceptions and for a digest that is due". §6 marks these as exceptions: overlap, duplicate, landed work, cycle, decision conflict, words near John's authority, an ask unanswered for 60 minutes, and unparseable posts. New claims (drift), NUDGE drafts and OFFER drafts are not among them.
     - On a quiet day, kimi-linear claims at 09:10 a download the purpose forbids. No exception occurs until 15:00. So the drift check, and any NUDGE or OFFER drafted in between, wait six hours, or until 07:30.
     - If instead every claim and draft wakes the model, the turn count is claims plus drafts plus exceptions, not "about 2".
   - *Smallest fix.* State one rule. `wait` exits at once for overlap, for words near John's authority and for cycles. Everything else coalesces into at most one exit every 30 minutes: new claims, NUDGE and OFFER drafts, duplicates and unparseable posts. That bounds judgement at 30 minutes and adds at most 2 turns an hour.

3. **High. The cost advantage depends on few exceptions and a young context.**
   - *Scenario* (§2.2). The estimate fixes the context at 80k and the turns at 2 an hour, but:
     - The steward's conversation grows by about 3k tokens a turn, and nothing bounds it except autocompaction.
     - Under the default 5-minute cache TTL every turn starts cold.
     - Grammar failures, and keyword hits on routine ingestion posts, raise the exception count.

     At 6 turns an hour the steward costs about $111 a day at Opus 5.5 list prices, against about $19 at 2 turns. It also re-reads about 5.7M tokens an hour, which is within candidate A's range.
   - *Smallest fix.*
     1. Coalesce wakes as in issue 2.
     2. Make an unparseable post a digest line, plus one grammar reminder per worker per day written by the tool, rather than a wake.
     3. Keep the list of words near John's authority per topic, in TOPIC.md, and fire it only on CLAIM and ASK.
     4. Bound the context. For example, judge each turn sheet in a fresh subagent that returns at most 300 tokens of decisions. The tool, not the subagent, receives the mail, so the rule that only the main session receives still holds. A daily fresh start is the alternative.
     5. In the pilot's first week, record from the harness how many tokens each wake costs.

4. **Medium. The chosen home puts every steward release on agent-mail's shared pin.**
   - *Scenario.* In the pilot's first week the duplicate threshold is tuned three times.
     - If the tool ships inside `pkgs.agent-mail`, each release moves the shared pin and changes agent-mail's store path on all ten hosts, including every MCP entry that names it.
     - Each release reruns agent-mail's release gates, and needs a nix-config publish and at least a hera activation, approved by John (plan Task 20; decision 4).
   - *Smallest fix.*
     - Run the tool from a checkout on hera during the pilot; decision 7 keeps checkouts as the development source.
     - After the pilot, give it its own source record or Gitea repository, and a package for hera only.
     - Keep the worker section in the agent-mail skill.

5. **Medium. OFFER has no backlog to offer, and workers have no rule for it.**
   - *Scenario.* The command list has `add` and `drop` without saying what they add. `init` records only the purpose, and the worker section never mentions OFFER. When kimi-linear is starved, the tool has nothing to offer, so every starvation becomes "a question for John".
   - *Smallest fix.*
     - Add `agent-steward item add <slug> --host H` and `item drop`, typed by John in the steward's pane and recorded "by John".
     - Add a worker verb, `OPEN <slug>: …`, for work a worker finds but will not do.
     - Add one worker line: "An OFFER is a suggestion; claim an item only if it fits what John gave you."

6. **Medium. Conflict detection sees only file paths.**
   - *Scenario.* Ingestion's likely conflicts are over GPUs, host load and disk. The worked example's purpose is "one model per GPU host", and hera's load of 70 to 90 twice timed out a gate (handoff). The fleet's one recorded cross-session conflict was in an id space: obr comment ids 812 and 813 (handoff). `writes:` can express none of these.
   - *Smallest fix.* Generalize `writes:` to `holds: <namespace>:<key or range>`, for example `andoria-08:gpu0-3` or `obr:812-813`. Two holds overlap when they share a namespace and their keys or ranges intersect. Host paths keep their prefix rule.

7. **Medium. Work nobody claimed is invisible, and the coverage figure does not count it.**
   - *Scenario.* nixos builds an agent-mail module for vulcan without posting a CLAIM, as in the handoff's Vulcan duplicate. No detector fires. The census compares direct mail with NOTE and ASK posts, so it still reports full coverage.
   - *Smallest fix.* Add a detector: a roster worker running real work (a workflow or agent in `substate_detail`, not just a shell) for 30 minutes with no open item gets a digest line, and at most one reminder a day.

8. **Medium. The ledger name looks dead whenever the topic is quiet.**
   - *Scenario.*
     - v2 refreshes a member record on any CLI `read`, even an empty one, and on CLI `pending`, at most every 300 s. The doorbell never writes it [V: `touch_member` lines 679 to 690, `commit` line 1624, `cmd_pending` lines 2867 to 2873].
     - The tool reads only when its doorbell rings, that is, when a post arrives.
     - After 15 quiet minutes `ingest-steward.ledger` reads stale in `channels`. The workers' rule then declares the steward silent, and workers send their asks as direct mail. When traffic resumes, the steward never sees those asks.
   - *Smallest fix.* Have the tool run `agent-mail pending <topic>-steward.ledger --format human` every 5 minutes and discard the output.

9. **Low to medium. "The same kind of work" has no field.**
   - *Scenario.* glm53-fp8 (a conversion) and glm53-eval (an evaluation) share the rare tokens `glm-5.3` and `fp8`. The duplicate rule requires the same kind of work, which the grammar cannot carry. So the tool either flags the pair, costing an exception and a turn, or guesses from free text.
   - *Smallest fix.* Add an optional `kind:` from a short list (download, convert, eval, publish, fix, review), or adopt the slug convention `<model>-<kind>`. Without a kind, a match is listed in the turn sheet and wakes no one.

10. **Low. The ladder stops at 60 minutes.**
    - *Scenario.* §7 gives a second pointer at 30 minutes and a drafted ask to a related owner at 60, then nothing unless the matter is John's alone. A peer question nobody can answer stays open, and shows only in digests.
    - *Smallest fix.* At 120 minutes, the tool either offers the question to the model as an `ask` with "hold" as the default, or closes it in the digest as unanswered.

11. **Low. Pointer batching can add up to 5 minutes.**
    - *Scenario.* Under "at most 1 per recipient per 5 minutes":
      - muse-spark asks lfm3 at 17:20:44, and lfm3 receives at 17:21.
      - glm-5.3-flash asks lfm3 at 17:22, and its pointer waits until 17:25:50.
      - lfm3 is idle on a quiet subscription, so it sees nothing until then.
    - *Smallest fix.* Batch only within 30 seconds, or reset the window when the recipient claims its previous pointer.

12. **Low. The steward's own posts break its own grammar.**
    - *Scenario.* The ledger name receives the steward's digest and CLOSED posts. Neither verb is in the list, so by the stated rule each is an unparseable exception, which wakes the steward for its own output.
    - *Smallest fix.* File posts from `<topic>-steward` without parsing them, and add DIGEST and CLOSED to the verb list.

13. **Low. Answers need a terminal.**
    - *Scenario.* John is away from his desk and Discord shows `Q ingest-4 …?`. The only answer path that carries his authority is typing in ingest-steward's pane. The candidate rejects mail to the director, and does not adopt the johns-view report's relay from Discord through the director for choices that need no authority.
    - *Smallest fix.* For the pilot, accept panes only and say so in the page line. Revisit the one-line director relay (johns-view report §3.6) after a week of pages.

## 6. What C does better than the alternatives

1. **Faster detection.** It flags overlaps and duplicates in about 30 to 80 seconds. Candidate A's steward reads the channel on a 15-minute tick, so it can take up to 15 minutes plus a turn.
2. **Lower model spend.** A routine post costs the model nothing. At 2 exceptions an hour, C makes about 10 model requests an hour; A makes 18 to 50 (6 to 10 turns of 3 to 5 requests).
3. **An exact ledger that survives compaction.** Code files every post id, and a self-test pins that every id is either filed or shown to the model. In A and B the model writes the ledger.
4. **Asks ring the right peer.** One post rings the peer, without copying text and without a v2 change. A needs two sends per request, and B needs `cc` in v2.1.
5. **A second reader name.** A names its own limit: "Each name has one reader, so no script can file posts for the steward". C answers it with a second reader name, which is what makes points 1 to 3 possible.

## 7. Three questions John must answer for C to work

1. **The second reader name.** May a script read each topic under a second mailbox name that no session uses (`<topic>-steward.ledger`)? And may it send pointers and welcomes as `<topic>-steward` without the model's approval? If not, C becomes candidate A.
2. **Freshness and spend.** How fresh must the steward's judgement be, and what may each steward spend?
   - Should drift, stuck and starved be judged within 30 minutes (at most about 2 more turns an hour), or only at the next exception and the 07:30 digest?
   - What daily budget should each steward have? For scale: about $19 a day at 2 turns an hour and $111 a day at 6, at Opus 5.5 list prices with the 5-minute cache.
3. **Pilot preconditions.** Will you set these two?
   - agent-deck 1.16.30 or later on andoria-08 and delphi-3bd4. Without it, stuck, starved and deaf detection is blind on those hosts.
   - The tool run from its own checkout on hera, outside agent-mail's every-host nix package, until the pilot ends.

## 8. What I checked today, and what I did not

1. **agent-mail v2 at 9de4b43** (`git show HEAD:agent-mail`, 4,805 lines):
   - the name grammar, which allows dots (line 99);
   - the reserved names (lines 102 to 104);
   - CLI `read` takes the lock through `hold_reader` (lines 2774 to 2790);
   - the MCP server takes its lock at start, unless it runs in a Claude background job (lines 2650 to 2653), and refuses to receive inside one (lines 2486 to 2491);
   - `ringable` rings for direct mail and for posts on wake subscriptions (lines 2084 to 2093);
   - the doorbell writes only its slot (lines 2161 to 2200);
   - the send caps, and the sender's window on its host (lines 515 to 558);
   - subscribing again changes only wake (lines 944 to 949);
   - the member record is refreshed by `commit` and by CLI `pending` (lines 679 to 690, 1592 to 1624 and 2867 to 2873);
   - the usage text (lines 30 to 48).
2. **Documents:** the v2 design §§4, 6.8 and 6.10; the v2 plan's scope, Global Constraints, Exact copy, Tasks 19 to 25 and Needs John; decisions 1 to 27.
3. **agent-deck catalogs:**
   - `agent-deck remote sessions delphi-3bd4 --json` and `andoria-08 --json`, at about 17:47Z: their fields, all seven ingest workers on Claude, and the substates quoted above;
   - `agent-deck --version` on hera (1.16.34), and over ssh on both Positron hosts (1.16.29);
   - `agent-deck list --json` on hera;
   - `pgrep` of the local watchers: pi, nix, mail-conductor and agent-mail.
4. **agent-deck fork at 0953876e:**
   - `internal/tmux/substate.go`: Pi gets no substate; only Claude and Codex are classified;
   - `internal/tmux/background_work.go`;
   - `internal/session/userconfig.go`, lines 4219 to 4232;
   - `git tag --contains 8813ec3b`.
5. **The director's scanner:** `monitor_sessions.py`, lines 672 to 686, 768 to 812 and 1033 to 1066.
6. **The claude-api skill:** the model table (cached 2026-10-06), and `shared/prompt-caching.md` lines 123 to 124, 144, 148 and 240.
7. **Not checked:**
   - Claude Code's cache TTL and its autocompact threshold;
   - how John is billed;
   - how well workers will follow the grammar;
   - whether the exit of each of two background shells wakes an idle Claude session. That Claude Code shows two shells at once appears in agent-deck's fixture captured from a live pane ("· 2 shells, 1 monitor ·"), which I read today;
   - how Pi workers' status behaves. nixcfg has three Pi workers, and agent-deck gives Pi no substate, so the tool would have only running, waiting or idle for them.

I opened no file in `~/Mailbox`, ran no agent-mail read, watch, pending or send, and messaged, woke or spawned no session.
