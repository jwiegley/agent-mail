# Critique of candidate A (conventions only): goal fit and cost

Critic for the goal-fit-and-cost lens. Written on hera, 2026-10-31, about 10:55 PDT, read-only. I read the candidate in full, then the brief, all six exploration reports, and the summaries of candidates B and C. I also checked the candidate's load-bearing claims at their sources.

Evidence marks:

- **[V]**: I verified it today at the place given.
- **[V: name]**: a sibling report or candidate verified it today, and I did not re-check it.
- **[U]**: documented or assumed, not checked.

Dollar figures use API list prices from the claude-api skill's reference, a table cached on 2026-10-06 [V]. I did not check how John's Claude Code sessions are billed. This session's footer shows 5-hour and 7-day usage meters [V], which suggests plan limits, not dollars, are what a steward actually consumes.

## Verdict: fixable

Candidate A makes the central decision correctly: the steward sees the work but never carries a message. The risks, v2-fit and prior-art reports all reach the same conclusion. A stays off agent-mail 2's critical path with no agent-mail code. It delivers part of each of John's seven goals and none in full.

Three defects would show up in the first week of a pilot.

1. **Its only push path to John is mis-specified.**
   - The page line in the worked example is 107 characters. hera's panes are 108 columns, and Claude indents continuation lines by 2, so the line wraps. The scanner then pages a fragment.
   - A question only John can answer, posted to the channel, waits 30 minutes plus up to 15 before anyone pages him. Today a question in a worker's own pane reaches him in about 69 s.
2. **Its cost section is not a budget.**
   - It quotes no prices and counts no cost on the workers' side.
   - It runs a model turn on a blind tick: every 15 minutes by day and every 60 minutes overnight, which land where prompt caches expire.
   - My estimate is about $33 to $47 a day per steward on Opus 5.5, or $69 to $107 on Fable 5.1 (the model this session runs). Workers' extra requests add about $8 to $16 a day per topic.
3. **Nothing notices a steward that has gone deaf.**

Each defect has a small fix inside A's own boundary, with no agent-mail code and at most one host-local script or one rule in John's director files. So the design is fixable, not unsound.

## 1. Goal by goal

Delays assume v2's expected latencies (design §6.10), the candidate's 15-minute active tick and 60-minute quiet tick (§5.1 rule 5), and the scanner's measured pass interval: median 69 s, 160 s at most [V: director report §1.3].

| # | John's goal | How A delivers it | Delay as written | What it misses | Rating |
|---|---|---|---|---|---|
| 1 | "see the overall pattern of all the work" | A ledger of items, owners, `writes:` and states. One STATUS line per topic (`cat …/stewards/*/STATUS`). A DIGEST post at most every 2 h, and a morning digest in the steward's pane. | A post is filed at the next tick: 0 to 15 min (mean 7.5), or 0 to 60 min (mean 30) in quiet hours. | Direct mail sent without a NOTE. Coordination through files, git and obr. Sessions in no topic, and any view across topics. Coverage is unknown, and A says it "cannot be measured" (§2.4 item 1). John must pull everything except pages. | Partly |
| 2 | "see if there's any duplication" | The worker's own check before claiming (rule 2). The steward's DUPLICATE and ALREADY-DONE checks on a shared distinctive token, looking back 7 days. | From the second claim to mail in that claimant's session: about 1.5 to 17 min. Overnight, up to about 62 min. | Work nobody claims. Duplicates across topics: the fleet's one recorded duplicate, the Vulcan branch, crossed topics [V: director report §1.9]. A compacted worker cannot find "the latest DIGEST", because `reread` takes only an id or `last=true` [V: `tool_reread` at 9de4b43]. | Partly |
| 3 | "anything taking place that might conflict" | OVERLAP on `writes:` and `uses:` (paths, branches, GPUs, ports, id ranges, host load). Conflicting DECISIONs. Wait cycles. | Same as goal 2. An overlap still unsettled goes to John at 60 min. | Anything not declared up front: obr ids are taken at write time, which is how the 812/813 collision happened [V: risks E15]. Contention across topics, such as GPU work during a Positron activation, or `pi` serving two topics. | Partly |
| 4 | "harmonious to the purpose", "the goal is remaining focused" | Each claim is checked against the ledger's purpose. The steward asks the owner once, and after two drifting claims asks John whether to widen the purpose. | At filing: 15 min at most. | Drift after the claim, because the steward never sees the work itself. `purpose` changes only the ledger: v2 cannot edit a channel marker, so the purpose workers see in `channels` falls behind the ledger. | Partly |
| 5 | "no one is getting stuck" | **Stuck:** 45 min with no state post while the catalog shows the owner idle, then a NUDGE. **Deaf:** mail older than 15 min while idle, reported in the digest, and to John if someone waits. **Gone:** the member's items are listed as orphaned. An unanswered REQUEST gets a ROUTE. | 45 to 60 min until a NUDGE. 15 to 30 min until deafness is noted. | Codex workers, which A lists for John after 90 min. False NUDGEs during long background jobs (issue 7). The steward itself, which has no liveness alarm (issue 4). | Partly |
| 6 | "no one is being starved for work" | **Starved:** 30 min idle with no open item, then an OFFER of up to 3 backlog items that fit the host. An empty backlog with at least 2 starved members goes to John. | 30 to 45 min. | John has to seed the backlog. One starved member facing an empty backlog waits for a second to starve. A member idle behind a BLOCKED item never counts as starved. | Partly |
| 7 | questions "proactively surfaced to the Agent Deck conductor rather than waiting for me to notice them" | The steward's last pane line, which the director's scanner pages to Discord and indicator 1. | About 69 s after the line is printed. A QUESTION posted to the channel takes 30 min, plus 0 to 15 min until the tick, plus a 1 to 2 min turn, plus up to 160 s: about 32 to 50 min. | The 107-character example wraps and pages a fragment (issue 1). Questions only John can answer still wait 30 min. One upload pages John twice (issue 2). A text question cannot be answered from Discord (issue 10). | Partly |

**Tally: 0 met, 7 partly, 0 not.**

With the fixes in section 5:

- Goal 7 becomes **met**.
- Goals 1 to 3 become partly met with a measured coverage figure, instead of partly met with an unknown one.

### Notes behind the ratings

- **Goal 1.** The worked example's 2-minute catch (claim at 17:19, tick at 17:21) is the lucky case. With a uniform arrival time, the mean wait is 7.5 minutes and the worst is 15.
  - Ingest runs GPU jobs overnight, and the quiet-hours tick is 60 minutes. A duplicate 3-hour job started at 01:05 can run until about 02:05 before the steward sees it.
- **Goals 2 and 3.** The worker's pre-claim check is the first line of defence and the steward only backs it up. That first line fails exactly when a worker has been compacted and the last DIGEST is no longer in its context [V: `reread` takes `id` or `last`].
- **Goal 5.** For remote workers, all seven ingest sessions included, the catalog fields A needs do exist.
  - `remote sessions <host> --json` returns `status`, `substate`, `last_activity_at` and `tool`, but no `background_work` [V: design C, lines 194 to 195; design B, line 64, run at 10:29 on delphi-3bd4 and andoria-08].
  - All seven ingest workers run Claude Code [V: designs B and C]. A's "[U: its fields]" can be closed.
  - The director's scanner tries a direct `ssh <host> agent-deck list --json` first, and uses `remote sessions` only when that returns nothing.
    - Its docstring says the direct path exists for "remotes whose configured wrapper is stale" [V: `monitor_sessions.py` lines 335 and 374 to 398].
    - Both Positron hosts run agent-deck 1.16.29 behind `agent-deck-remote-env` [V: `agent-deck remote list --json`].
- **Goal 7.** The mechanism is the right one. It is mis-specified in two places, the line width and the 30-minute floor, and it doubles pages for protected follow-ups.
  - The scanner it depends on changed three times this morning: `monitor_sessions.py` at 10:34, `notify_discord.py` at 10:37, and a new `discord_qna.py` at 10:44 [V: file times]. A's formatting rules are coupled to a parser that is moving.

## 2. Cost

### 2.1 Messages per worker message

| Worker action | Today, without a steward | Under A | Synced files |
|---|---|---|---|
| Ask one peer to act | 1 direct | 1 direct, plus 1 NOTE post | 2 (twice today's) |
| Change state (CLAIM, UPDATE, BLOCKED, DONE, DROP) | none | 1 post | 1 (new) |
| A question nobody owns | 1 guess, or a pane question | 1 post, then after 30 min a ROUTE from the steward (1 direct) | 1 to 2 |
| Reply | 1 | 1 | 1 |
| Steward intervention | none | 1 direct per session concerned | 1 to 2 |
| Digest | none | at most 1 post every 2 h | at most 12 a day |

For ingest, assume 7 workers each making 2 state changes and 0.5 requests an hour:

- Today they produce about 3.5 synced files an hour.
- Under A they produce about 3.5 + 3.5 + 14 + 1.5 = 22 files an hour: about 6 times as many.
- Every file reaches all ten hosts through hera [V: design §6.10].
- Posts are kept 30 days, so `+ingest` holds about 14 × 16 × 30 = 6,720 post files at 16 working hours a day.

### 2.2 Tokens and money for a Claude steward

**Assumptions.** Each is labelled.

1. **Prices per million tokens** [V: claude-api skill].

   | Model | Input | Output | Cache read |
   |---|---|---|---|
   | Opus 5.5 | $4 | $20 | $0.20 |
   | Fable 5.1 | $10 | $50 | $0.25 |
   | Sonnet 5.5 | $2 | $10 | $0.20 |

   Cache writes cost 1.25 times input with a 5-minute lifetime and 2 times input with a 1-hour lifetime [V: skill's prompt-caching reference].
2. **The cache lifetime Claude Code 2.1.312 uses: I did not check.** Claude Code's costs page says the first message after an idle gap longer than the cache lifetime reprocesses the full context [V: prior-art §5 item 5].
3. **The steward's model: I did not check** agent-deck's default for `-c claude`. This session runs Fable 5.1 [V: its footer].
4. **Fixed overhead:** 47.3k tokens per request [V: risks E7].
5. **Requests per tick turn: 8.** The candidate says 3 to 5, but its own sequence (§5.1 rule 4) has seven tool calls and a final message:
   - pgrep;
   - Read the ledger;
   - receive;
   - the catalog checks: `list --json`, plus `remote sessions` for two hosts;
   - save the ledger, STATUS and log;
   - restart the tick;
   - the final message.
6. **Turns.** About 4.5 an hour while work is active: 4 ticks, plus about 0.5 rings. Workers almost never mail the steward under worker rules 5 and 8. One tick an hour overnight. For ingest, 8 active and 8 quiet hours a day, because overnight jobs keep items open.
7. **Context.** It grows about 5k tokens a turn:
   - reading the 8 KB ledger, about 2.3k;
   - rewriting it, about 2.3k, which stays in context as tool input;
   - receive output and signals, 0.5k to 1k.

   The mean is about 135k over the active day and about 230k overnight.
8. **Output.** About 0.6k tokens per request (tool calls and thinking), plus 2.3k per ledger rewrite.

**Tokens, as written.**

- **Per active hour.**
  - 4.5 turns × 8 requests = 36 requests.
  - 36 × 135k = **4.86M cached input tokens**.
  - **About 32k output tokens**: 36 × 0.6k + 4.5 × 2.3k.
  - If the cache does not survive the 15-minute gap (a 5-minute lifetime), add 4 full rewrites: 4 × 135k = 0.54M tokens at the write price.
- **Per overnight tick.** One full rewrite of about 230k, then 7 reads (1.6M) and about 7k output.
  - A 60-minute sleep outlives even a 1-hour cache. The cache's clock starts at the last request, which follows the tick's restart by under a second, so the next turn starts just after the hour.
- **Per day:** about 50M cached input and 0.3M output tokens.
- The candidate's range of 2M to 15M an hour contains my 4.9M. What it leaves out is cache misses, output, prices and the workers.

**Dollars per steward at API list prices.** Arithmetic for the first column: 4.86M × $0.20 + 22.5k × $8/M + 32k × $20/M = $0.97 + $0.18 + $0.64 = $1.79.

| As written | Opus 5.5, 1-h cache | Opus 5.5, 5-min cache | Fable 5.1, 1-h cache | Fable 5.1, 5-min cache |
|---|---|---|---|---|
| Active hour | $1.79 | $4.31 (includes 4 × 135k × $5/M = $2.70 in misses) | $3.27 | $9.71 |
| Overnight tick | $2.30 (230k × $8/M = $1.84 miss) | $1.61 | $5.36 | $3.63 |
| **Day: 8 active hours and 8 ticks** | **$33** | **$47** | **$69** | **$107** |

**Per worker message.**

- **Marginal cost:** about $0.01 to $0.02.
  - About 185 tokens are read once, then carried by about 150 later requests (about 4 hours): 28k cached tokens, or $0.006 to $0.007.
  - Filing output of about 150 tokens costs $0.003 to $0.008.
- **Fully loaded:** the hourly cost divided by ingest's 17 worker messages an hour.
  - $0.11 to $0.25 on Opus 5.5, $0.19 to $0.57 on Fable 5.1.
  - The blind tick is more than 90% of it.

**Per digest: about $0.25 to $0.32, at most 12 a day, so about $3 to $4.**

- **The steward:** 2 to 3 requests × 135k = 0.27M to 0.41M cached, plus about 1.4k output. That is about $0.10 on Opus 5.5 and $0.16 on Fable 5.1.
- **The readers:** 7 workers × about 550 tokens, each carried by about 150 requests: 0.58M cached tokens, about $0.15.

**The workers' side, which the candidate does not count.** The conventions add tool calls, not turns.

- **Extra requests:** about 3.5 per worker-hour:
  - each claim takes 3 (receive for the board, post, receive again), at 0.5 claims an hour;
  - each state post takes 1, at 1.5 an hour;
  - each NOTE takes 1, at 0.5 an hour.
- **Cost of those requests:** each re-reads the worker's context, assumed 200k [U]. That is 0.7M cached tokens, about $0.14 per worker-hour, or about $1.0 an hour for 7 workers.
- **Carrying the posts:** each worker carries about 2.6k tokens of posts per hour of work into every later request. At an assumed 60 requests an hour, that adds about $0.9 an hour by mid-day for 7 workers.
- **Total:** about $1 to $2 an hour, or **$8 to $16 per 8-hour day, per topic**.
- **Each NUDGE** costs its worker about 3 requests: 0.6M tokens, about $0.12.

**With the small fixes in issue 3.**

- A read-only helper prints one signal sheet, so a turn takes about 5 requests.
- The steward re-reads a summary of at most 1 KB and edits the ledger, so context grows about 2k a turn.
- A fresh context each morning keeps the mean at about 83k by day and 120k overnight.
- The overnight tick is at most 55 minutes.

The result is about **$7.50, $25, $14 and $57 a day** for the four columns above, a cut of 46% to 80%. All of these figures are estimates within ±50%. Day 1 of the pilot should replace them with the harness's own usage figures.

### 2.3 Latency per hop

The direct times use v2's expected latencies and one measured hop of 2.07 s [V: risks E1], with an assumed 3 s to wake an idle Claude session [U].

| Hop | Today | Under A, as written |
|---|---|---|
| A to B direct, same host | about 6 s (2 s settle + 1 s detection + 3 s wake) | the same |
| A to B direct, Positron to Positron through hera | about 10 s | the same. The NOTE costs A one tool call (5 to 15 s of A's time), off B's path. |
| A state post reaches the steward | never | 0 to 15 min (mean 7.5), plus about 4 s of sync. 0 to 60 min overnight. |
| From a second claim to a DUPLICATE or OVERLAP in that claimant's session | never | tick wait + a 1 to 2 min turn + about 10 s hop + about 3 s ring: 1.5 to 17 min. Overnight, up to about 62 min. |
| A question in a worker's pane reaches Discord | median 69 s, at most 160 s [V] | the same |
| A QUESTION posted to the channel reaches Discord | never (until someone notices) | 32 to 50 min |
| A stuck item gets a NUDGE | never | 45 to 60 min |
| A starved member gets an OFFER | never | 30 to 45 min |
| A dead steward comes to John's notice | not applicable | unbounded (issue 4) |

### 2.4 Sessions, approvals and burden

**Sessions John runs and watches.**

- One long-lived Claude steward per topic on hera, two as named (ingest, nixcfg).
- Discord pages, as today.
- `cat` of the STATUS files, the morning digest in each steward's pane, and the ledger on demand.
- The director is unchanged.

**One-time actions and approvals per topic.**

1. Approve the design and answer its load-bearing questions (section 7).
2. Create the channel with `agent-mail channel create ingest --purpose …`. Only the CLI creates channels, and A leaves the choice to John [V: `create_channel`].
3. Run the `agent-deck launch` line [V: `--create-dir`, `-g`, `--no-parent`, `--no-assert-done` and `-m` in `launch --help`].
4. Type `add <names>` in the steward's pane. In practice he also types one line in each member's pane (7 for ingest, 6 for nixcfg) unless he accepts INVITEs from the steward.
5. Seed the backlog, one `backlog` line per item.
6. Decide whether stewards run with dangerous_mode, and whether they may list mailbox file names (the candidate's Q9 and Q7).
7. Later, approve a nix-config publish and activations that carry the worker section and the steward skill:
   - Hera, through pi;
   - the four Positron home-manager switches, as in plan Task 24 [V].

   The alternative is to ride inside Tasks 22 and 24, which touches v2's critical path (section 4).
8. Optionally, edit his director files.

**Recurring actions.** Answer topic questions in the steward's pane. Authorizations stay in workers' panes, as today. Use `purpose`, `give` and `pause` when needed.

**Compared with doing nothing:**

| | Doing nothing | A as written |
|---|---|---|
| Long-lived sessions | 0 | 1 per topic |
| Pane visits to start | 0 | about 15 (1 launch, 7 + 6 members), or 2 if INVITEs are accepted |
| Pages | workers' pane questions (7 since 09:44 today [V: director report §1.5]) | the same, plus steward questions, plus a second page for protected follow-ups |
| Reading | nothing | STATUS, the morning digest, the ledger on demand |
| Recovery | the director handles error states | the same, plus noticing a deaf steward, which nothing alarms on |
| API-rate cost | 0 | $33 to $107 a day per steward, plus $8 to $16 a day per topic for workers |
| Upkeep | none | the worker section and the steward skill, kept in nix |
| What John gains | nothing | the duplicates, overlaps, stalls, starvation and mail-only questions the table in section 1 shows, within its delays |

## 3. Simplicity

### What could be removed without losing a goal

1. **The suggested exemption of `*-steward` sessions from the 30-minute stall page.**
   - It is unnecessary: a healthy steward's footer shows "N shell(s)", which the scanner already counts as live work [V: this session's footer reads "1 shell"; the pattern is in `pane_has_live_work`].
   - It would also remove the one alarm a dead steward trips (issue 4).
2. **The wait-cycle check.** It needs "waiting on" links between items. With 7 workers a cycle is rare, and the Stuck check catches its symptom within the hour.
3. **`digest.md`**, a third digest product beside STATUS, the ledger and the morning digest.
4. **The 4-hour "STILL OPEN" re-page.** The morning digest already brings open questions back.
5. **The full Read of the ledger on every turn (§5.1 rule 4.2).** A summary of at most 1 KB is enough; the full ledger is needed only after a restart or compaction. This is the largest cost lever in section 2.2.

### What is missing that a goal needs

1. **A coverage figure (goals 1 to 3).** For cross-host mail the gap can be measured, because every cross-host file passes through hera.
   - A host-local log of member-to-member file names (to, from, stamp; never content), compared with NOTEs, gives a percentage for each digest (risks R23, constraint 14).
   - A 15-minute `ls` misses most of it, because Claude workers claim within seconds.
2. **A liveness alarm for the steward itself** (all goals).
3. **A fast path for questions only John can answer, and one page per protected action** (goal 7).
4. **A duplicate check across topics** (goal 2).
5. **A way for a worker to see the board after compaction** (goal 2).
6. **Honouring a declared `until` or ETA** (goal 5, and cost).
7. **A named model and a budget line** (cost).
8. **The cost constraint is self-imposed.** A says "each name has one reader, so no script can file posts for the steward" (§2.4 item 6). Posts are never claimed, though: a helper can subscribe under its own name, as candidate C does with `ingest-steward.ledger` [V: design C §2]. Adopting that moves A toward C.

## 4. agent-mail 2's critical path

- **Code.** None in agent-mail. Every primitive A uses is committed at 9de4b43: `create_channel` with `PURPOSE_MAX` of 200, quiet subscriptions, `send_window`, `ringable` and `reread` [V].
- **The channel needs no plan change.** `+ingest` fits the plan's "Needs John" item 4, which allows `+fleet` plus at most one topic channel of John's choosing [V: plan].
- **The request to mail-dev is harmless as long as it is about wording.** It asks to keep two of Task 22's phrases verbatim and to leave room for a worker section.
- **The risk is the worker section.**
  - A defines it as "a section of the agent-mail skill" (§5.2). That skill is created by Task 22, which is uncommitted. Task 24 carries it to the Positron hosts through home-manager switches that need John's approval [V: plan].
  - Writing the section into Task 22 before landing adds content, review and a nix check to C2.
  - Writing it afterwards needs another publish and five activations.
  - The pilot needs neither. The steward's opening post (at most 8 KiB) can carry the conventions, and John's one line in each member's pane can point to it. The nix section can follow once the pilot has shown which rules survive.
- **Timing.**
  - Nothing of John's goals arrives before Task 24 activates the Positron hosts and Task 25 passes. A has no v1 path; the operations draft had one.
  - Task 25 exercises a Claude session on hera and a Codex session on andoria-08 [V: plan]. The ingest workers are Claude sessions on Positron hosts, so the pilot's first hour should ring one Claude worker on each Positron host before NUDGE and DUPLICATE mail is relied on.
- **Verdict: off the critical path**, provided the worker section stays out of Task 22's landing.

## 5. Issues

1. **High: the page line wraps on hera and pages a fragment.**
   - **Scenario.**
     - hera's agent-deck panes are 108 columns wide (18 panes, plus one at 90) [V: `tmux list-panes`]. Claude indents continuation lines by 2 columns [V: this session's pane].
     - The worked example's line is 107 characters [V: counted], so it wraps.
     - The scanner captures without `-J` and pages only the row holding "?", here something like "(public)?" [V: `capture_tmux`, `visible_prompt_or_question`].
     - The scanner already paged such a fragment today: audio8-tts's "that I want to generate in the synthesized voice? …" [V: `heartbeat.log` tail].
   - **Smallest fix.**
     - Cap the line at 80 characters with the "?" inside it, for example `Q ingest-3: which HF org gets glm-5.3-fp8? A positron-internal, B positron-ai` (77 characters). Put details on earlier lines, and never include a second "?".
     - Ask John for one fixture in his `test_monitor_sessions.py` asserting that a steward pane ending in `Q <topic>-<n>: …` pages the whole line. The scanner changed three times today, and the fixture would pin the contract.
2. **High: questions only John can answer wait at least 30 minutes, and one upload pages John twice.**
   - **Scenario.**
     - In the candidate's own example, a QUESTION posted at 19:52Z is paged at 20:22Z. The worst case is about 50 minutes; a question in a worker's pane pages in about 69 s.
     - The upload's authorization then pages again from kimi-linear's pane, because §7 sends "what gets published" to the steward's pane and steward rule 8 sends the authorization to the worker's pane. John answers twice, in two panes.
     - §7's "straight to John" contradicts the 30-minute row in §6.
   - **Smallest fix.**
     - Classify each question when it is filed: anything on §7's list pages at the first turn that sees it.
     - A choice that gates a protected action goes to the acting worker as NEEDS-JOHN with its options, so John settles both choice and authorization once, in the pane where his word counts.
3. **High: cost has no budget, its blind ticks defeat the cache, and the workers' side is omitted.**
   - **Scenario.** Section 2.2:
     - about $33 to $47 a day per steward on Opus 5.5, or $69 to $107 on Fable 5.1, plus $8 to $16 a day of workers' extra requests;
     - each overnight 60-minute tick rewrites the full cache;
     - the candidate gives 3 to 5 requests a turn against its own list of 8, and checks no price.
   - **Smallest fix.**
     - One read-only helper in the steward's directory that prints the turn's signals (pgrep, catalog, remote lists, mailbox names) in one call.
     - A summary of at most 1 KB re-read each turn, with Edit instead of a ledger rewrite.
     - A fresh context each morning; the state lives in files.
     - An overnight tick of at most 55 minutes, or one only when a deadline falls in the window.
     - Name the model, and state the budget with the workers' cost included.
     - The result is about $7.50 to $25 a day on Opus 5.5, or $14 to $57 on Fable 5.1. Measure it on day 1.
4. **Medium: a deaf steward goes unnoticed.**
   - **Scenario.**
     - A restart without error status (a hera reboot, an agent-deck restart) kills the doorbell and the tick (candidate §8).
     - The director restarts only sessions in status `error` [V: POLICY rule 5].
     - The scanner's stall page fires once per session and is remembered in `progress_watch` [V: `record_progress`], so a second death never pages.
     - Workers' "steward silent" notes go to a channel only the dead steward reads.
   - **Smallest fix.** One rule in John's scanner: page when a `stewards/*/STATUS` line is older than twice its tick while it reports open items.
   - **The alternative.** Tag stewards `continue-until-complete` so the scanner sends `Continue` when a steward waits with no live work. This works for local sessions only [V: director report §1.4].
     - Use it only after a day's check that a healthy steward always shows "N shell(s)".
     - Otherwise the scanner prompts it every pass, about 60 turns an hour.
   - **Drop the suggested stall exemption** either way.
5. **Medium: coverage is assumed, not measured.**
   - **Scenario.** A worker mails a peer and forgets the NOTE. The steward never knows, and John reads a ledger that looks complete.
   - **Smallest fix.** Add a host-local log of names only on hera, covering top-level files addressed to members, with a coverage line in each digest, for example "NOTEs for 9 of 14 member-to-member messages". It needs John's yes on listing names (the candidate's Q7).
6. **Medium: duplicates across topics are invisible.**
   - **Scenario.** The fleet's one recorded duplicate crossed topics: the rollout session's Vulcan branch against nixos's module. Each steward checks only its own ledger, and steward rule 10 forbids reading anything else.
   - **Smallest fix.** Let each steward read its siblings' `stewards/*/ledger.md` on hera (same user, same host) and run the duplicate check across them.
7. **Medium: long background jobs draw false NUDGEs.**
   - **Scenario.**
     - glm-5.2 starts a 3-hour conversion as a background task and waits at an empty prompt.
     - Remote listings carry no `background_work` [V: design C, lines 194 to 195], and agent-deck reports a Claude session with background shells as `idle-at-empty-prompt` [V: director report §4 item 4].
     - After 45 minutes without a post, the steward sends a NUDGE, and each NUDGE costs the worker about 0.6M tokens.
   - **Smallest fix.** Worker rule 4 gains `UPDATE <slug>: running until <time>`. The Stuck check skips an item until its declared time plus 25%.
8. **Medium: the board cannot be found after compaction.**
   - **Scenario.** Worker rule 2 requires "the latest DIGEST", but `reread` takes only an id or `last=true` [V]. A compacted worker has no id.
   - **Smallest fix.** Workers send `BOARD <topic>` by direct mail, and the steward answers from the ledger in its own words. That is at most one ring per claim.
   - **Later.** Ask mail-dev for a `history` tool, as a v2.1 request after landing.
9. **Low: starvation has gaps.**
   - **Scenario.** One starved member facing an empty backlog waits until a second starves. A member idle behind a BLOCKED item never counts as starved.
   - **Smallest fix.**
     - Ask John when any member has been starved for 60 minutes with an empty backlog, once a day per member.
     - Offer items to members idle behind a block.
10. **Low: John cannot answer from his phone.**
    - **Scenario.**
      - The page reaches Discord, but an answer works only in the steward's pane: the TUI over ssh, or Remote Control [U for this fleet].
      - The director's Discord replies, new at 10:44 today, cover only boxed, numbered Claude single-choice menus, and only on hera [V: `DISCORD-QNA.md`].
    - **Smallest fix: John chooses one.**
      - Short, urgent questions go out as a numbered menu he can answer with a Discord button. The steward's turn blocks until he does.
      - Or one director POLICY line relays `<topic>-<n> <answer>` from Discord to the steward as a peer's report, for choices that need no authority (johns-view §3.6).

## 6. What A does better than the alternatives

1. **Nothing to build, package or audit.** A is two texts and existing commands.
   - B needs a v2.1 change of about 110 lines and 15 checks. It touches committed Tasks 3 to 6, 8, 12 and 14 to 18, plus uncommitted Task 22 [V: design B §1].
   - C needs a second script [V: design C §1].
   - A can start the day Task 25 passes.
2. **Direct mail stays private.** Only a one-line NOTE goes to the channel. B's `cc` subscription lands members' direct mail in `+<topic>` by default [V: design B §1], which copies it to all ten hosts and changes what "direct" means.
3. **It is the easiest to abandon.** Dropping A means deleting a skill section. B leaves a primitive in agent-mail, and C leaves a second reader name and tool.

## 7. The three questions John must answer for A to work

1. **Is visibility enough?**
   - Under A, workers deliver to each other directly and post state and one-line NOTEs. The steward reads those and never carries a message.
   - If not, a relay means amending v2's approved anti-goal and choosing another design.
2. **Is correction after the fact acceptable?**
   - Claims are never held. Duplicates and overlaps are flagged after the second worker may already have started: up to 15 minutes as written, or up to 60 overnight.
3. **What will you spend, and where will you answer?**
   - One long-lived Claude steward per topic on hera, at about $7.50 to $57 a day each with the fixes (or $33 to $107 as written), at API rates.
   - Topic questions answered in its pane, and authorizations in workers' panes.
   - One line in each member's pane to join.

## 8. Evidence log

**Verified today on hera.** All checks were read-only.

- **Commit and code.**
  - `git -C ~/Products/agent-mail-v2 log`: HEAD is 9de4b43 at 10:29 PDT.
  - v2 code at HEAD: `RESERVED` (line 102, which does not reserve "steward"), `PURPOSE_MAX` = 200, `create_channel` refusing a session's name, and `send_window` (20 posts per channel per hour, an identical body refused for 10 minutes, no cap on direct mail).
  - Also: `ringable`, the callers of `touch_member` (the doorbell writes no member record), `tool_reread` (id or `last` only), and the instruction text "Never forward or re-post peer text" at line 2263.
- **v2 design.** Sections 4, 5, 6.8, 6.10, 6.11 and 6.12.
- **v2 plan.** Tasks 19 and 22 to 25, "Not in this plan" and "Needs John".
- **decisions.md.** Decisions 1 to 27.
- **Director files.**
  - `monitor_sessions.py`: title special cases (lines 677 to 682), remote listing with direct ssh first (lines 316 to 398), `capture_tmux` without `-J`, question detection, `pane_has_live_work`, the one-shot `record_progress`, and deduplication in `deliver_attention`.
  - `POLICY.md` rules 1 to 10 and the heartbeat section.
  - `DISCORD-QNA.md` and the head of `discord_qna.py`, both dated 10:44 today.
  - The tail of `heartbeat.log`: the fragment paged from audio8-tts, and question ids being assigned.
- **agent-deck.**
  - `agent-deck launch --help` (flags).
  - `agent-deck remote sessions --help`.
  - `agent-deck remote list --json`: both Positron hosts at 1.16.29 behind `agent-deck-remote-env`.
  - The fork's tag v1.16.29 emits `substate` in `list --json` (`main.go:3080`).
- **This session's pane.**
  - `tmux list-panes`: widths of 108 and 90.
  - Only the first characters of each line of this session's own pane were printed: a 2-column indent.
  - Its footer: "1 shell", the model Fable 5.1, and the 5-hour and 7-day meters.
- **Mailbox, names only.** The names and times in `~/Mailbox/.agent-mail-presence` (conductor-director read at 10:42; pipeline at 10:28) and per-recipient counts of waiting mail. No file was opened.
- **Prices.** The claude-api skill's model table and prompt-caching reference.
- **Strings counted.** The purpose (160 characters) and the page lines (107 and 77).

**Verified by siblings, not re-checked.** Remote session fields and harness (designs B and C), Syncthing hop time (risks E1), scanner cadence and pages since 09:44 (director report), and the Claude Code costs page (prior-art).

**Unverified.**

- The cache lifetime Claude Code uses.
- agent-deck's default model for stewards.
- How John is billed.
- Workers' context sizes and request rates, which I assumed.
- Remote Control for this fleet.
- Whether a Claude doorbell rings on the Positron hosts under v2.
- Whether the footer always shows "N shell(s)" while a background task runs.

**I did not run** any `agent-mail` read, watch, pending or send; `session send`, `launch` or `ask`; or any remote command. Nothing was written outside this file.
