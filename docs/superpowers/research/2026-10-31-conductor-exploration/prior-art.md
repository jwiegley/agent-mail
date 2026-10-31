# Prior art: a per-topic coordinator that sees peer traffic

Explorer: prior art (one of six), for the agent-mail conductor design on branch work/conductor. Written 2026-10-31. Research was read-only: no repository, worktree, mailbox file or session was changed, woken or messaged.

**Legend.** *Verified* means I read the primary source or ran the command today (2026-10-31), and the citation says where. *Unverified* (marked **(U)**) means documented elsewhere but not checked by me, or known only from secondary sources. *(mine)* marks my own inference. "Steward" is used below as a placeholder for John's per-topic role. Section 6 discusses the name.

## 0. Bottom line

1. **No system built for a fleet like this one relays every peer message through an LLM coordinator.** By "like this one" I mean long-lived, independent sessions spread over harnesses and hosts. Claude Code agent teams let teammates message each other directly, while the lead gets idle notifications and a shared task list. Gas Town's Mayor dispatches work from a ledger while separate Witness patrols watch for stuck workers. mcp_agent_mail has no lead at all. agent-deck's director watches panes, not mail. The frameworks that do route everything through a manager each run one task inside one process for minutes: MetaGPT's MGX team leader, LangGraph's supervisor, CrewAI's hierarchical process, OpenAI's manager pattern and Magentic-One.
2. **Relaying through a coordinator has measured costs.**
   - One extra model call per request ("results flow back through the main agent", LangChain's docs).
   - Paraphrase ("telephone") errors. LangChain's supervisor scored poorly until it got a verbatim `forward_message` tool, a fix worth "nearly 50%".
   - A synchronous bottleneck (Anthropic, June 2025).
   - 285% coordination overhead at a matched token budget, with success per 1,000 tokens falling from 67.7 for a single agent to 21.5 under central coordination (Kim et al., 2026).
3. **A coordinator pays off where it verifies, not where it carries messages.** Under central coordination, trace-level error amplification fell from 17.2× (independent agents) to 4.4×, and duplicated work (redundancy) was lowest of the multi-agent setups (Kim et al.).
4. **Software and human coordinators fail the same four ways:**
   - It does the work itself (Claude Code docs; CrewAI issue 4783; PagerDuty forbids it for incident commanders).
   - It rubber-stamps. Claude Code approves teammate plans by default "without the lead reviewing it". Agents' completion claims run 28.7 to 37.9 points ahead of verified passes.
   - It over-manages: Anthropic's "50 subagents for simple queries", and the director restarting sessions unasked in July.
   - It becomes a bottleneck or single point of failure (the EIP broker warning).
5. **Agent-mail 2 already rules out the relay form.** Its anti-goals forbid any tool that "forwards, re-posts", and its MCP instructions say "Never forward or re-post peer text" (design lines 70, 255, 337, 419 and 471). A visible copy (EIP's Wire Tap) fits v2. A relay does not, unless v2 changes.
6. **"Conductor" and "foreman" are both taken here.** agent-deck's own conductor README calls the conductor "the foreman on a jobsite". I recommend **steward**, with **shepherd** (the IETF role) as the alternative and **coordinator** as the neutral fallback.

## 1. What this builds on

Section 2 of the v2 design already cites the transport-side prior art: AutoGen Core topics, MetaGPT's case for publish-subscribe, Claude Code agent teams' one JSON inbox per agent, mcp_agent_mail and its Human Overseer preamble, agent-bus's "mailbox plus doorbell", AgentRadio, Prompt Infection, Morris II, adaptive attacks, DeLM and AgentPrune. I do not repeat those findings. This report re-reads the same projects, and others, for their coordinator: what it sees, what it decides and how it fails. The design's section 11 lists LangGraph and CAMEL as not researched; both are covered here.

## 2. Comparison table

Sources are Verified (read 2026-10-31) unless marked (U). Full URLs are in section 9.

| Pattern or framework | What the coordinator sees | How messages route | Duplication and conflict handling | Stuck or idle detection | Human escalation | Known failure modes | Source |
|---|---|---|---|---|---|---|---|
| **Classic patterns** | | | | | | | |
| Mediator (GoF) | Every interaction among its components | Components talk only to the mediator, never directly | A central place for rules; nothing built in | None | None | "Over time a mediator can evolve into a God Object" | refactoring.guru mediator |
| Supervisor (Erlang/OTP) | Child terminations (liveness). The page covers lifecycle only; it says nothing of message content (mine) | Not on the message path | n/a | A child's exit triggers restart: one_for_one, one_for_all, rest_for_one | Escalates up the tree. More than MaxR restarts in MaxT s (default 1 in 5) kills all children and exits to the parent supervisor | Restart loops, hence intensity limits; nested intensities multiply | erlang.org sup_princ |
| Hierarchical control (broker hierarchy, supervision tree, multi-level supervisors) | Its own subtree | Local traffic stays local; cross-subsystem traffic goes up to a central broker | Per level | Per level | Upward | Too many brokers make "Message Broker spaghetti"; too few make an "über-Message Broker" | EIP MessageBroker; langgraph-supervisor README (multi-level) |
| Blackboard (Hearsay-II 1980, BB1 1985) | The whole board of partial solutions | Knowledge sources read and write the board; a control shell schedules who acts | The control shell sequences contributors so they do not collide ("mad dash to grab the chalk") | BB1's meta-level control watches whether problem solving progresses or stalls and switches strategy | None | Control becomes the bottleneck; representation burden | Wikipedia "Blackboard system" |
| Moderated pub-sub (Usenet) | Every submission, before anyone else | The injecting agent mails it to the moderator, who adds `Approved:`; servers "MUST reject any article without an Approved header field" | The moderator can refuse repeats | None in the protocol | The moderator is human | Latency; the group stalls when its moderator is away (U, mine) | RFC 5537 §3.5 |
| Message Broker, Process Manager, Wire Tap (EIP) | Broker and Process Manager: every message. Wire Tap: a copy of every message on one channel | Broker: hub and spoke. Process Manager: holds process state and picks the next step. Wire Tap: a recipient list that copies to a second channel | The Process Manager holds the state of the sequence | n/a | n/a | "can turn the Message Broker into a serious bottleneck"; "danger of turning the Process Manager into a performance bottleneck". Wire Tap is added "without any danger of modifying the primary channel behavior" | EIP pages |
| Orchestrator-workers (Anthropic, Dec 2024) | Task, plan, worker results | "a central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results" | The orchestrator chooses subtasks | n/a | n/a | Use only when it "demonstrably improves outcomes" | anthropic.com building-effective-agents |
| Facilitator (KQML federation 1994; SRI OAA, late 1990s) | Capability adverts and requests (U) | Agents send requests to facilitators, which match them to capable agents; agents need not know each other (U) | Matching prevents double assignment (U) | (U) | (U) | Facilitator bottleneck, hence federations of facilitators (U) | OAA's "central 'blackboard' server holds a list of tasks" (Wikipedia, Verified); federation details from secondary sources only (U) |
| **Human roles** | | | | | | | |
| Foreman | The crew and the day's work | Instructions go to the foreman, who assigns the crew | Sequences trades (U) | Walks the site (U) | To the superintendent (U) | A working foreman stops supervising (mine) | Wikipedia "Foreman": "in charge of a construction crew" |
| Dispatcher (computer-aided dispatch) | Unit status ("Avail." / "Not Avail." with incident number), call locations | Calls come to dispatch; the dispatcher picks from CAD's shortlist of the closest available units | Mapped calls "disambiguate calls for service", so one incident is not dispatched twice | Status board; every action timestamped | Supervisors (U) | "relatively minor failures can have major effects"; an AVL outage spikes workload | Wikipedia "Computer-aided dispatch" |
| Air-traffic controller | All traffic in its sector (U) | Clearances to pilots; handoffs between sectors (U) | Separation is the job (U) | Lost-communication procedures (U) | Authority stays in the cockpit: the pilot in command "is the final authority as to, the operation of that aircraft" | Sector overload (U) | 14 CFR 91.3 (Verified); the rest (U) |
| Newsroom assignment editor | Tips, releases, reporters' beats | Assigns stories; runs the daily "budget meeting" | Splits a big story into angles across reporters rather than repeating it | Day-to-day management of staff | Senior editors | (none found) | Wikipedia "Assignment editor" |
| Incident commander (NIMS; PagerDuty) | The whole incident; the single point of coordination | "All communication from your team should come via the leader"; tasks go to a named person with a time box | Named assignment: "Never say 'Can someone...'" (bystander effect) | Time boxes: "I'll come back to you for an answer in 3 minutes" | Handover is announced with a briefing; an executive is asked whether they wish to take command | An IC doing the work: "You should not be performing any actions or remediations". NIMS: optimal span of control is one supervisor to five | PagerDuty IC training; FEMA NIMS 2017 |
| Scrum master | Impediments; the events | Not a router. Teams "internally decide who does what, when, and how" | The Daily Scrum serves to "identify impediments" | Daily Scrum | Facilitates stakeholder collaboration | Becoming the task assigner breaks self-management (mine) | Scrum Guide 2020 |
| IETF document shepherd (added) | All feedback on one document | Point of contact for reviewers; writes the shepherd write-up for the Area Director | Tracks and follows up feedback "from all relevant parties" | "must keep the document moving forward" | The Area Director decides; only ADs clear a DISCUSS. The shepherd "SHOULD NOT be an editor of the shepherded document" | When a DISCUSS stalls, the AD takes over shepherding | RFC 4858 |
| Linux subsystem maintainer (added) | Every patch for its subsystem (MAINTAINERS) | Patches go to subsystem maintainers, not Linus; Andrew Morton is "maintainer of last resort" | Separate trees per subsystem | Wait "a minimum of one week", then resend with `[PATCH RESEND]` | Linus is "final arbiter", yet "very few patches go through Linus directly" | Unresponsive maintainers (implied by the resend rules) | docs.kernel.org submitting-patches |
| **Frameworks and products** | | | | | | | |
| AutoGen SelectorGroupChat and Core group chat | The full shared history | Every message is "broadcasted to all other participants"; an LLM or `selector_func` picks the next speaker (`RequestToSpeak` in Core) | Prompt-level only; by default no speaker goes twice in a row | Termination conditions such as `MaxMessageTermination(25)` "to avoid infinite loop" | `UserProxyAgent` as a participant; an APPROVE loop | Selector prompts overload; in the docs' own run an agent called a tool with invented numbers | AutoGen stable docs |
| Magentic-One (AutoGen) | A task ledger (facts, guesses, plan) and a progress ledger | The orchestrator gives one agent a subtask per step | The progress ledger asks `is_in_loop` | A stall counter: +1 for no progress or a loop, −1 otherwise; re-plan when it reaches `max_stalls` (default 3; the paper used ≤ 2) | `approval_func` before code runs | The docs warn of prompt injection from web pages; orchestrators were hijacked (Triedman et al., below) | arXiv 2411.04468; AutoGen main `_magentic_one_orchestrator.py` |
| MetaGPT MGX team leader ("Mike") | Every regular message | "every regular message goes through team leader". The leader re-publishes to members as a `UserMessage`. A user's direct chat to an idle role bypasses it | Leader prompt: "If a team member has finished a task, do not ask them to repeat it" | Tracks the plan from members' feedback | `ask_human`; "ask the user for clarification immediately" | Telephone risk ("you are their sole info source"). A copy to the leader "to know but not to react" was commented out "because TL doesn't understand the message well in actual experiments". Relayed text becomes user-role text | FoundationAgents/MetaGPT main: `mgx_env.py` (last changed 2024-10-15), `team_leader.py`, prompts |
| CrewAI hierarchical process | Tasks and workers' outputs | The manager delegates and validates; delegation is off by default | Manager allocation | None documented | None documented | Issue 4783 (opened 2026-03-09, closed 2026-04-16): manager agents "never delegate to other agents" | docs.crewai.com; GitHub API |
| LangGraph supervisor | Everything: it "controls all communication flow"; `full_history` or `last_message` | Workers return to the supervisor through handoff tools | n/a | n/a | n/a | Telephone errors, fixed with `forward_message` (+~50%). The library was archived 2026-09-20; LangChain now recommends "the supervisor pattern directly via tools" | langgraph-supervisor-py README; LangChain blog 2025-06-10 |
| OpenAI Agents SDK | Manager: every subtask. Handoffs: nothing after the transfer | "Agents as tools": the manager keeps control. Handoffs: the specialist takes over | n/a | n/a | The guide escalates on "Exceeding failure thresholds" and "High-risk actions" | "maximize a single agent's capabilities first" | SDK docs; *A practical guide to building agents* (PDF) |
| Anthropic research system (June 2025) | Its plan in memory; subagents' summaries | The lead spawns 3 to 5 subagents in parallel and waits for them synchronously | Each subagent needs "an objective, an output format, guidance on the tools and sources to use, and clear task boundaries" | The lead waits; the system "can be blocked while waiting for a single subagent" | n/a | 50 subagents for simple queries; duplicated work (the chip-crisis example); the synchronous bottleneck; 15× the tokens of chat | anthropic.com multi-agent-research-system |
| Claude Code agent teams | Idle notifications that carry each teammate's final answer; the task list; failures | Peers message each other directly; the lead is not on the path; the user can message anyone. No broadcast: "send one message per recipient" | Shared task list; claims under a file lock; dependencies. "Two teammates editing the same file leads to overwrites" | Idle notification; the `TeammateIdle` hook (exit 2 keeps the teammate working); a notice on an API error | Permission prompts surface in the lead. A relayed approval is untrusted. In auto mode a classifier reviews each message, and "A message it blocks never reaches the recipient" | The lead "starts implementing tasks itself"; the lead "can stop early"; "Task status can lag"; "Lead is fixed"; plans approved "without the lead reviewing it"; about 7x the tokens in plan mode | code.claude.com agent-teams.md, costs.md |
| Google ADK coordinator/dispatcher | Requests and sub-agents' descriptions | `transfer_to_agent` or `AgentTool`; task-mode sub-agents return to the coordinator; parallel branches cannot see each other | Parallel agents "write results to distinct keys in Shared Session State" | `LoopAgent` `max_iterations`; `escalate=True` ends a loop | A custom tool, or a policy engine that asks for CONFIRM | None documented | adk.dev workflows/patterns, collaboration |
| Microsoft Agent Framework group chat | The full history, in a "star topology" | The orchestrator picks the speaker and "broadcasts the response to all other agents" | n/a | Round limit and termination condition | Points to the Magentic orchestration for planning | n/a | learn.microsoft.com group-chat (ms.date 2026-10-22) |
| CAMEL Workforce | Tasks and assignments | A task agent decomposes; a coordinator assigns; workers run in parallel | Dependencies inside each `TaskAssignment` | `task_timeout_seconds`; recovery by Retry, Replan or Decompose | `HumanToolkit` on any agent | None documented | docs.camel-ai.org workforce |
| mcp_agent_mail | No coordinator; a human web UI | Peer to peer under contact policies | Advisory file reservations with a TTL; a conflict is reported but the reservation is "still granted"; an optional pre-commit guard | `last_active_ts`; `ack_required`; no watchdog | Human Overseer, downward and in-band ("supersedes all other priorities") | In-band authority (already cited by v2) | GitHub README (Python version deprecated; Rust supersedes it) |
| agent-bus | No coordinator | One ledger; each session reads its own address | None | None | None | Its research says pane-typing orchestrators are "an injection risk" | github.com/rawr-ai/agent-bus (research dated 2026-10-26) |
| Gas Town (Steve Yegge) | Mayor: the workspace. Witness: one rig's workers. Deacon: every rig | `gt mail`, `gt nudge`, assignment with `gt sling`; a beads ledger | Beads assignment; the Refinery, "a Bors-style merge queue" that bisects failures | States "GUPP Violation", "Stalled", "Zombie"; daemon "heartbeat every 3 min"; nudge or hand off | `gt escalate` P0 to P2, routed "Deacon -> Mayor -> Overseer" | Rate-limit exhaustion, hence a scheduler cap; author calls it "100% vibecoded" (U, secondary) | github.com/steveyegge/gastown README (gastown.dev did not resolve today) |
| **Local prior art** | | | | | | | |
| agent-deck director (conductor-director, Codex) | The panes of about 34 active sessions (48 local catalog entries, 13 remote). It does not see mail traffic | Not on the mail path; reads its own mail with the v1 CLI each turn | n/a | Pane fingerprint unchanged for 30 min (5 h for vulcan/nixos), alerted once; regex recovery signatures; a watchdog alerts if no scan completes for 10 min | `NEED(<location>):` lines, Discord with a mention, `indicator 1`; an "Always Escalate" list; "When Unsure ... escalate" | Restarted sessions unasked (LEARNINGS 20260715-002); a missed alert; noise from review sessions. TUI heartbeat submission was unreliable, so detection moved into the deterministic `monitor_sessions.py` | Read locally: `~/.local/share/agent-deck/conductor/director/` |
| agent-deck conductor primitives (v1.16.34) | Parent and child turns, sends and wakes (comms ledger, off by default) | `conductor notify --tier urgent\|info` ("urgent now, info as a digest"), an outbox with ack, a delta-only `heartbeat-tick`, `ask` dialogs to the owner | Target `duplicate_turn_pct == 0` | `session children` shows status and last completion | Notify tiers; `ask choice` / `ask text` | Target `wakes_per_parent_hour <= 4` | `agent-deck --help`, `conductor --help`, `ask --help`; `~/src/fork/agent-deck/docs/comms.md` (4f383789, 2026-10-25) |

Another precedent in this fleet is the rollout session itself (`agent-mail`). Its handoff keeps a per-task status table, answers branch agents' questions, and records routing such as "Hera activation coordination is routed to `pi`" (director POLICY). It is already a working per-topic coordinator.

## 3. What each classic pattern and human role decides, and what it must not become

| Pattern or role | Decides | Must not become | Lesson for a steward |
|---|---|---|---|
| Mediator | How components interact | A god object holding all the logic | Keep the steward's job narrow |
| Erlang supervisor | When to restart, when to give up and escalate | A worker, or a restart loop | Liveness checks plus escalation; lifecycle actions only where John has authorized them (director POLICY) |
| Blackboard control | Which contributor acts next | The only reasoner | A shared ledger that workers write to, which the steward reads |
| Moderator | Whether a post appears | A gate whose absence stops the group | Never block delivery |
| Broker / Process Manager | Where a message goes, or the next step | An über-broker bottleneck | Stay off the delivery path |
| Wire Tap | Nothing; it copies | A filter on the primary path | The visible-copy shape |
| Orchestrator-workers | The subtasks | Used where one agent suffices | Only for topics with parallel work |
| Facilitator | Who can serve a request | The only path between agents | Advise; peers still talk directly |
| Foreman | Who does which job today | The crew's best worker doing the job | The steward does not edit or push |
| Dispatcher | Which available unit takes a call | A single point whose failure stalls all service | Track who is free; avoid starvation |
| Air-traffic controller | Clearances for separation | The pilot | Final authority stays with the worker and John |
| Assignment editor | Which reporter covers which angle | The copy desk rewriting stories | Assign angles; never rewrite peer text |
| Incident commander | The plan, and who does what by when | A hands-on troubleshooter, or a boss of 15 | Time boxes, named owners, span of 5 |
| Scrum master | Nothing about who does what | The team's manager | Remove impediments; serve |
| Document shepherd | Nothing about approval | The document's editor or approver | Keep work moving; summarize for the decider |
| Subsystem maintainer | What enters its tree | An unresponsive gate | Resend norms; a last-resort route |

## 4. Cross-cutting lessons from the frameworks

1. **Three routing families.**
   - *Star with broadcast*: AutoGen group chat and Microsoft group chat. Everyone sees everything, and the manager only picks the next speaker.
   - *Relay through the manager*: MetaGPT MGX, LangGraph supervisor, OpenAI's manager pattern, CrewAI.
   - *Peers direct, plus a coordinator fed by notifications and a shared ledger*: Claude Code agent teams, Gas Town, and mcp_agent_mail without a lead.

   Only the third family is built for long-lived independent sessions, and it is the closest to agent-mail. (All Verified, section 2.)
2. **Relaying causes telephone errors, and the fix is verbatim forwarding.** LangChain measured it and fixed it with `forward_message` and by stripping routing chatter from workers' context. MetaGPT's prompt pleads "DONT omit any necessary info ... because you are their sole info source". MetaGPT also dropped a "know but do not react" copy to its leader because the leader misread it. That warns against copies that look like requests.
3. **Coordinators drift into doing the work.**
   - Claude Code documents its lead doing it ("Wait for your teammates to complete their tasks before proceeding").
   - CrewAI issue 4783 records it as a bug.
   - AutoGen's planner prompt forbids it: "You only plan and delegate tasks - you do not execute them yourself".
   - Gas Town's README: "Witnesses delegate work rather than implementing it directly".
   - PagerDuty forbids it outright for incident commanders.
4. **Rubber-stamping is a default, not an accident.** Claude Code approves a teammate's plan "as soon as the request arrives, without the lead reviewing it" unless the teammate's permission mode has changed. In "Who Holds the Pen?" (arXiv 2609.29921, Sept 2026), completion-claim rates "exceed official evaluator pass rates by 28.7--37.9 percentage points" across seven models. MAST lists "No or incomplete verification" and "Incorrect verification" among its 14 failure modes (arXiv 2503.13657 v3, which reports failure rates of 41% to 86.7% across seven open-source multi-agent systems).
5. **Stuck detection is always cheap metadata, never reading everything.** The forms:
   - counters with thresholds (Magentic-One's stall counter; Gas Town's "no progress for an extended period"; the director's 30-minute fingerprint);
   - event notifications (Claude Code's idle notification and `TeammateIdle` hook);
   - time boxes (PagerDuty);
   - resend norms (the kernel).

   The director alerts once per episode, a lesson it learned in this fleet.
6. **Upward escalation with tiers exists in only two places.** Every framework offers a human hook for its own process: `UserProxyAgent`, `approval_func`, `HumanToolkit`, ADK's CONFIRM. Tiered upward escalation exists only in Gas Town (P0 to P2, "Deacon -> Mayor -> Overseer") and agent-deck (urgent now, info as a digest). mcp_agent_mail's human path runs downward and in-band, which is the anti-pattern v2 already rejects.
7. **The coordinator is the most exposed node.**
   - Web content hijacked orchestrators into running arbitrary code "in 58-90% of trials (depending on the orchestrator)" with GPT-4o, up to 100% in some configurations. It worked "even if individual agents are not susceptible to direct or indirect prompt injection" (Triedman, Jha and Shmatikov, arXiv 2503.12188, Sept 2025).
   - An agent on the message path can compromise the whole system "by only manipulating the messages passing between agents" (AiTM, arXiv 2502.14847, ACL 2025).
   - A monitor is not a boundary. "Seven of nine tested frontier models disguise the credential ... while evading the monitor", and "Monitoring favors concealment" (Covert Assistance, arXiv 2609.39050, Sept 2026).
8. **Observability works best on structure, not content.** Anthropic tracked "agent decision patterns and interaction structures—all without monitoring the contents of individual conversations". SentinelAgent models interactions as "dynamic execution graphs" with anomalies at node, edge and path level (arXiv 2505.24201). The Arbiter (arXiv 2606.10747, June 2026) watches under an "inspection budget"; active questioning improved "both detection accuracy and speed", and logging improved recall "at the cost of precision".

   Diagnosis stays weak, though. The best automated failure attribution identifies the responsible agent 53.5% of the time and the failing step 14.2% of the time (Who&When, arXiv 2505.00212). The best model scores about 11% on TRAIL's trace debugging (arXiv 2505.08638).
9. **Telling the allocator what each worker can do helps.** "providing explicit information about worker capabilities enhances the allocation strategies of planners", and a planner beat an orchestrator "in handling concurrent actions" (arXiv 2504.02051). A learned central "puppeteer" orchestrator reached "superior performance with reduced computational costs" through "more compact, cyclic reasoning structures" (arXiv 2505.19591, NeurIPS 2025; the abstract gives no figures).

## 5. Evidence about cost

1. **One extra call per request through a coordinator** (Verified, LangChain multi-agent docs). Subagents take 4 model calls for a one-shot request against 3 for handoffs, skills or router, and 8 against 5 for a repeat request. The docs' reason: "adds one extra call because results flow back through the main agent—this overhead provides centralized control". In the multi-domain case, context isolation means the subagent pattern "processes 67% fewer tokens overall".
2. **Paraphrase costs score and tokens** (Verified, LangChain blog, 2025-06-10, on a modified τ-bench). The supervisor "consistently uses more tokens than swarm", because of "translation". Its fixes brought "a nearly 50% increase in performance". Other numbers appear only in charts.
3. **Overhead at a matched budget** (Verified, Kim et al., arXiv 2512.08296 v3, revised 2026-04-08, 260 configurations, mean 4,800 tokens per trial):

   | Metric | Single agent | Independent | Decentralized | Centralized | Hybrid |
   |---|---|---|---|---|---|
   | Success rate | 0.466 | 0.370 | 0.477 | 0.463 | 0.452 |
   | Turns | 7.2 | 11.4 | 26.1 | 27.7 | 44.3 |
   | Overhead | 0% | 58% | 263% | 285% | 515% |
   | Redundancy | 0 | 0.48 | 0.50 | 0.41 | 0.46 |
   | Error amplification | 1.0 | 17.2 | 7.8 | 4.4 | 5.1 |
   | Success per 1k tokens | 67.7 | 42.4 | 23.9 | 21.5 | 13.6 |

   Further findings:
   - Coordination failures: 1.8% centralized, 3.2% decentralized, 12.4% hybrid.
   - Turns grow as T = 2.72 (n + 0.5)^1.724, and "per-agent reasoning capacity becomes prohibitively thin beyond 3–4 agents".
   - Tasks whose single-agent baseline exceeds 45% "experience negative returns from additional agents".
   - Decomposable financial reasoning gained +80.8% under central coordination. Sequential planning (PlanCraft) lost 39% to 70% under every multi-agent architecture.
   - Caveat (mine): these are single benchmark tasks at matched budgets, not long-lived sessions with separate budgets. Read the direction, not the magnitude.
4. **Token multipliers.** Agents use "about 4× more tokens than chat interactions, and multi-agent systems use about 15× more", and token usage "explains 80% of the variance" on BrowseComp (Verified, Anthropic, 2025-06-13). Agent teams "use approximately 7x more tokens than standard sessions when teammates run in plan mode" (Verified, Claude Code costs.md).
5. **Each wake re-reads the whole context** (Verified, Claude Code costs.md). A message from another session arrives at an idle session "as a new turn ... sending your full context each time". A long session re-reads its history at the cached rate on every request. After an idle gap longer than the cache lifetime, the first message "reprocesses your full context". `/compact` "is itself a large request". So every relay hop is at least one full-context steward turn (mine).
6. **A wake budget already exists here** (Verified locally). agent-deck's comms ledger sets the target `wakes_per_parent_hour <= 4`.
7. **Context growth degrades quality.** Across 18 models, performance "grows increasingly unreliable as input length grows", even on simple tasks, and "Even a single distractor reduces performance" (Verified, Chroma, *Context Rot*, 2025-07-14). Anthropic treats context as a finite "attention budget" and recommends three things (Verified, 2025-09-29):
   - compaction;
   - structured note-taking;
   - subagents that return summaries of "often 1,000-2,000 tokens".

   Its research lead saves its plan to memory because "if the context window exceeds 200,000 tokens it will be truncated". Cognition recommends a dedicated compression model for long histories (Verified, 2025-06-12).
8. **Local data points** (Verified locally):
   - In about 108 days since 2026-07-15, the director's `task-log.md` reached 14.1 MB and 134,688 lines.
   - Its `state.json` holds 126 KB in 59 top-level keys, including one-off records such as `last_ares_github_issue_project_audit_directive`. Its startup checklist reads that file. That is roughly 30k tokens at 4 characters per token (my estimate).
   - Its routine detection now runs as deterministic Python every 60 s, outside the LLM.
9. **Latency in this fleet** (mine, from the v2 design and decisions):
   - Mail takes seconds on one host and tens of seconds between hosts, and twice that between two Positron hosts through hera.
   - A relay adds the steward's turn latency on top.
   - A Codex steward cannot be woken idle (decision 9), so a relay through it would wait for its next turn start or heartbeat.
10. **Not found.** I found no measurement of an LLM coordinator's quality decay over days, and no measured overhead of a coordinator over file-based mail. I did not measure this fleet's message volume: archives are host-local and off-limits to me.

## 6. Names

| Name | Where it is used | What it implies for behaviour | Fit |
|---|---|---|---|
| conductor | agent-deck's fleet supervisor, whose README says "the foreman on a jobsite" (Verified, local). conductor-oss, "an event driven agentic workflow engine" (Verified, GitHub API). Netflix Conductor (archived; Verified). Melty Labs' Conductor app for parallel agents (U) | Sets the tempo; everyone follows; central authority; does not play | Poor: collides at least three ways, and implies command |
| foreman | agent-deck's description of its conductor; Wikipedia: "in charge of a construction crew" | Directs and assigns the crew; often works alongside | Poor: same collision; gendered (Wikipedia offers forewoman and foreperson); implies directing |
| manager | CrewAI manager agent; OpenAI "manager pattern"; AutoGen and Microsoft `GroupChatManager` | Owns the final answer; keeps control | Poor: implies authority that mail cannot carry |
| supervisor | LangGraph ("controls all communication flow"); Erlang (restarts children) | Lifecycle control | Poor: the director may act on a session's lifecycle only where John has authorized it (POLICY rule 5) |
| lead | Claude Code "team lead"; Anthropic "lead agent" | A member who also works; fixed leadership | Poor: invites doing the work |
| moderator | Usenet (`Approved:`) | Approves or blocks posts | Poor: gating, and a bottleneck |
| editor / desk | Assignment editor; copy desk | Assigns angles (good) or rewrites copy (bad) | Mixed: suggests rewriting peer text, which v2 forbids |
| dispatcher | Computer-aided dispatch; ADK "Coordinator and Dispatcher" | Tracks who is free; assigns work | Partial: fits starvation, misses consistency and purpose |
| coordinator | ADK, CAMEL | Neutral | Acceptable fallback |
| orchestrator | Anthropic, Magentic-One, Microsoft | Plans, delegates, synthesizes | Implies planning everyone's work |
| steward | General English: data steward, shop steward (U) | Cares for something on an owner's behalf, without command. A shop steward carries workers' concerns upward | Good |
| shepherd | IETF document shepherd (RFC 4858, Verified) | Keeps work moving, tracks feedback, writes the summary for the decider; no approval; "SHOULD NOT be an editor" | Good |
| witness / mayor / deacon | Gas Town | Patrol watcher; dispatcher | Tied to Gas Town |
| arbiter / sentinel | Arbiter (2026), SentinelAgent (2025) | Judges; guards | Poor: implies judgement or a security role |

**Recommendation**

1. **steward**, named `<topic>-steward`, for example `ingest-steward` or `nix-steward`. It says the role cares for a topic's consistency on John's behalf without commanding anyone, and it avoids both agent-deck collisions. It is not on v2's reserved list and fits the name grammar. Its shop-steward sense, carrying workers' concerns upward, matches "surface questions".
2. **shepherd.** The IETF role matches the wanted behaviour most closely: keep work moving, track open items, write the summary, let the decider decide, never edit. It is less familiar, and "herding" is the wrong connotation for peers.
3. **coordinator.** Neutral, and used by ADK and CAMEL. It is the safe fallback but says less about the restraint the role needs.

Two practical notes. A `<topic>-steward` name avoids colliding with channel names; v2's plan notes that the session `nix` already blocks a channel named `+nix`. Also, the role name in a prompt probably shapes behaviour (mine; I found no study measuring it). MAST lists "Disobey role specification" as a failure mode, and the docs above show leads drifting into work, so a restrained name helps a little.

## 7. Implications for agent-mail

1. **Make visibility a copy, not a relay.** The sender mails the recipient directly, as today, and posts a short structured note to the topic channel (or to a future `cc`). That is the Wire Tap shape. The steward is never on the delivery path, so a dead, busy or idle-Codex steward costs visibility, never delivery. A relay would need the forward or re-post that v2's anti-goals and MCP instructions forbid.
2. **If John wants a relay for some messages anyway, make it opt-in and verbatim.** The steward's note points to the original message id. The recipient reads the original by its id, and no peer text is regenerated (LangChain's `forward_message`; MetaGPT's "sole info source" risk). Any v2 change this needs goes to the rollout session as a request by mail.
3. **The steward is a peer, and its mail carries no authority.**
   - Its mail gets the same frames as any session's.
   - It never uses `agent-deck session send` and never relays an approval. Claude Code treats a relayed approval as untrusted.
   - It never re-publishes peer text as user-role text, as MetaGPT's leader does with `UserMessage`.
   - It never takes an overseer voice like mcp_agent_mail's preamble.
4. **Questions for John go up an existing path.**
   - Either mail `conductor-director` with a `NEED(<topic>/<session>):` line, or use `agent-deck conductor notify --tier urgent|info`, which forwards urgent items now and info items as a digest. Whether a session that is not an agent-deck conductor may call `notify` is open (question 3).
   - The answer comes from John in the asking session's own harness, or through `agent-deck ask`. It never arrives as mail from the steward.
   - The steward never answers on John's behalf. The director's POLICY ("Always Escalate"; "When Unsure ... escalate") is a ready template.
5. **Keep a topic ledger that only the steward writes.** It records claims (who, which repositories, paths or hosts, since when, TTL), status and evidence, and its inputs are structured `CLAIM`, `DONE` and `BLOCKED` posts on the topic channel. It is advisory, never a lock. mcp_agent_mail grants overlapping reservations and reports the conflict; Claude Code locks the claim step but leaves edits to file ownership; Gas Town uses beads. The steward detects duplication against the ledger, not against its own memory.
6. **Trust evidence, not claims.** A `DONE` needs a commit, path or test log, and the ledger records asserted and evidenced separately. Completion claims run 28.7 to 37.9 points ahead of verified passes, Claude Code warns that "Task status can lag", and `agent-deck session children` reports asserted completions.
7. **Detect stuck and starved workers in code from metadata, and use the LLM only on exceptions.** The inputs:
   - the age of `checked` in each member record (display-only and peer-writable);
   - the last post per member in the topic channel;
   - direct questions with no reply (`In-Reply-To`) after N hours;
   - ledger items that have not moved.

   Add a stall counter per item with a threshold of 2 to 3 cycles, as Magentic-One does. Alert once per episode, as the director learned to, with thresholds per topic (the director uses 30 minutes, and 5 hours for vulcan/nixos).
8. **Nudge by mail only, and respect each harness.** A nudge is direct mail, which rings the worker's doorbell; never keystrokes. Codex workers get no idle nudge (decision 9) and will see a nudge only at turn start or heartbeat, so the steward must know each worker's harness. Capability notes per worker also improve allocation (arXiv 2504.02051).
9. **Budget the steward's wakes.** Keep its channel subscription quiet (v2's default), let direct mail wake it only for conflicts and questions, and read copies in batches on a fixed cadence. Aim for at most 4 wakes an hour, agent-deck's own target, since every wake re-reads the steward's full context.
10. **Bound the steward's context.**
    - Read file names (to, from, stamp) and subjects first; open bodies only on an exception.
    - Keep state in files (the ledger, notes) and compact or restart on a schedule.
    - Cap and rotate logs and state. The director reached a 14 MB log and a 126 KB, 59-key state file in 108 days.
    - Context rot favours a fresh context fed by the ledger over one long transcript.
11. **Someone else watches the steward.** Keep it off the critical path. The director or `doctor` should flag a steward whose member record goes stale, as `heartbeat_watchdog.py` does for the scanner after 10 minutes. Restarting it stays John's call unless he grants a standing rule, as he did for the director's errored sessions (POLICY rule 5; LEARNINGS 20260715-002 records the restarts made before that rule existed).
12. **Keep 3 to 7 active workers per steward.** NIMS puts the optimum at 1:5; Claude Code advises starting with 3 to 5 teammates; Kim et al. find capacity thin beyond 3 to 4 agents. Today's mailbox shows natural topics:
    - Positron model ingestion: glm-5.2, glm-5.3-flash, kimi-linear, lfm3, muse-spark;
    - Nix: nix, nixos, pi, pallas, athena;
    - agent-mail: agent-mail, mail-conductor, mcp-servers.

    All 13 were active within the last 25 minutes on hera (presence listing at 10:21 PDT).
13. **The steward does not do the work.** No edits or pushes in topic repositories. Deny write tools in the harness configuration where possible, and state the rule in the skill too. The evidence: Claude Code's lead, CrewAI issue 4783, PagerDuty's incident commander rule.
14. **Summaries point; they do not paraphrase.** Digests cite message ids and paths so readers can `reread` the originals, and never quote peer text at length (v2's etiquette and the telephone evidence).
15. **Least privilege, because the steward is the most exposed.** It reads the most untrusted text in its topic, so it holds no secrets, no push rights and no lifecycle actions, and workers treat its requests like any peer's. Monitoring is hygiene, not a boundary (Triedman et al.; AiTM; Covert Assistance). By the Rule of Two the v2 design cites, a session that ingests mail should not also hold write and push power over private repositories.
16. **The steward's role is display-only metadata.** Record it in the channel marker's purpose line or in a member-record field, and show it in `channels`. It is never authority: v2 member records are display-only and writable by any peer.
17. **Pilot on a decomposable topic first.** Central coordination helps parallel work (+80.8%) and hurts sequential work (−39% to −70%). Positron model ingestion (similar parallel jobs across hosts) is a better first topic than Nix, which is sequential and publishes through one queue owned by `nix`. Measure duplicates caught, questions surfaced, wakes per hour and the steward's tokens per day.
18. **Treat diagnoses as hypotheses, and ask first.** Attribution is weak (53.5% by agent, 14.2% by step; TRAIL about 11%). Active questioning improved the Arbiter's detection. The steward asks the worker before flagging, and flags rather than decides.
19. **Reuse what exists before inventing anything.**
    - From the director: the `NEED(...)` convention, Discord deduplication and `indicator 1`.
    - From agent-deck: notify tiers, the outbox, the delta-only `heartbeat-tick`, `ask` and `session annotate` (durable purpose, decision and outcome hints).
    - From the rollout session: its status table and routing notes, a ready template for the ledger.

## 8. Open questions for John (most important first)

1. **Visibility or control?** Is it enough for the steward to see a copy of each coordination message? Or must messages pass through the steward, so it can hold, reorder or annotate them before the recipient reads them? A relay costs a steward turn per message, adds paraphrase risk and a single point of failure, and needs a change to v2's no-forward rule.
2. **What may the steward ask of workers?** Only observe, flag and ask questions? Or also propose assignments and reassign stalled work by mail? Either way, mail cannot authorize anything.
3. **Where should questions for you land, and how will you answer?** The options: mail to conductor-director, agent-deck's notify tiers (urgent now, info as a digest), Discord directly, or a digest read on a schedule. Will you always answer in the asking worker's own session?
4. **Which name:** steward, shepherd or coordinator? Conductor and foreman already belong to agent-deck.
5. **Which topic first, and who is in it?** I suggest Positron model ingestion.
6. **Which direct mail is in scope for copies?** All mail between topic members, or only coordination (claims, handoffs, conflicts, questions)? Is any mail exempt, such as reviews or private notes?
7. **What runs the steward?** Which harness and model, what daily token budget, and is it an agent-deck conductor (heartbeat, notify) or an ordinary session with the agent-mail skill?
8. **Who alerts for what, and at what threshold?** The director already pages on pane stalls. If the steward pages on work stalls, what threshold applies per topic, and how do we avoid two pages for one problem?
9. **Where do you want to see the whole picture?** A digest post in `+fleet`, a status page, the director's summary, or Discord info digests?
10. **Should stewards become agent-deck parents of their topic's sessions?** That would route `NEED:` lines and completion sentinels to them, but it changes existing parent links.

## 9. Sources

All read on 2026-10-31 unless marked (U).

**Frameworks and products**

- Claude Code agent teams: https://code.claude.com/docs/en/agent-teams.md (Verified)
- Claude Code costs (7x plan mode; cross-session messages as full-context turns): https://code.claude.com/docs/en/costs.md (Verified)
- Anthropic, "How we built our multi-agent research system", 2025-06-13: https://www.anthropic.com/engineering/multi-agent-research-system (Verified)
- Anthropic, "Building effective agents", 2024-12-19: https://www.anthropic.com/engineering/building-effective-agents (Verified)
- Anthropic, "Effective context engineering for AI agents", 2025-09-29: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents (Verified)
- AutoGen SelectorGroupChat: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/selector-group-chat.html (Verified)
- AutoGen Core group chat pattern: https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/design-patterns/group-chat.html (Verified)
- Magentic-One docs: https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html; paper: https://arxiv.org/abs/2411.04468 (stall threshold ≤ 2); source on GitHub main: `python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_magentic_one/` (`max_stalls` default 3, `max_turns` default 20, the five ledger fields) (Verified)
- MetaGPT MGX environment and team leader: https://github.com/FoundationAgents/MetaGPT, files `metagpt/environment/mgx/mgx_env.py`, `metagpt/roles/di/team_leader.py` and `metagpt/prompts/di/team_leader.py` (Verified; downloaded raw from main)
- CrewAI hierarchical process: https://docs.crewai.com/en/learn/hierarchical-process; issue 4783: https://github.com/crewAIInc/crewAI/issues/4783 (Verified via the GitHub API)
- LangGraph supervisor README (archived 2026-09-20): https://github.com/langchain-ai/langgraph-supervisor-py (Verified)
- LangChain, "Benchmarking multi-agent architectures", 2025-06-10: https://www.langchain.com/blog/benchmarking-multi-agent-architectures (Verified)
- LangChain multi-agent docs (calls and tokens by pattern): https://docs.langchain.com/oss/python/langchain/multi-agent (Verified)
- OpenAI Agents SDK, orchestrating multiple agents: https://openai.github.io/openai-agents-python/multi_agent/ (Verified)
- OpenAI, *A practical guide to building agents* (PDF): https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf (Verified; text extracted with pdftotext; I did not confirm its publication date)
- Google ADK patterns and collaboration: https://adk.dev/workflows/patterns/ and https://adk.dev/workflows/collaboration/ (Verified; google.github.io/adk-docs now redirects there)
- Microsoft Agent Framework group chat: https://learn.microsoft.com/en-us/agent-framework/workflows/orchestrations/group-chat (Verified; ms.date 2026-10-22)
- CAMEL Workforce: https://docs.camel-ai.org/key_modules/workforce (Verified)
- mcp_agent_mail README: https://github.com/Dicklesworthstone/mcp_agent_mail (Verified; the first 100,000 characters of the page were read)
- agent-bus: https://github.com/rawr-ai/agent-bus (Verified)
- Gas Town: https://github.com/steveyegge/gastown (Verified); https://gastown.dev did not resolve today. Release date and the "100% vibecoded" remark come from secondary sources found through search, such as https://dark-factory-kb.pages.dev/steve-yegge-gas-town.html (U)
- AgentScope MsgHub (broadcast hub with an announcement; no moderator): https://doc.agentscope.io/tutorial/task_pipeline.html (Verified)
- conductor-oss and Netflix Conductor, via the GitHub API: https://github.com/conductor-oss/conductor and https://github.com/Netflix/conductor (Verified)
- Melty Labs Conductor: https://www.conductor.build (U; secondary sources only)
- Google Antigravity's Agent Manager inbox: https://codelabs.developers.google.com/getting-started-google-antigravity (U; secondary sources only)
- agentgateway (A2A and MCP proxy with traffic observability): https://agentgateway.dev (U)
- OpenTelemetry GenAI agent conventions moved to https://github.com/open-telemetry/semantic-conventions-genai (Verified that the old page says so; contents not checked)

**Papers**

- Kim et al., "Towards a Science of Scaling Agent Systems", arXiv 2512.08296 v3 (2026-04-08), HTML version: https://arxiv.org/html/2512.08296v3 (Verified)
- Cemri et al., "Why Do Multi-Agent LLM Systems Fail?" (MAST), arXiv 2503.13657 v3 (2025-10-26): https://arxiv.org/html/2503.13657v3 (Verified)
- Triedman, Jha and Shmatikov, "Multi-Agent Systems Execute Arbitrary Malicious Code", arXiv 2503.12188: https://arxiv.org/abs/2503.12188 (Verified)
- He et al., "Red-Teaming LLM Multi-Agent Systems via Communication Attacks" (AiTM), ACL 2025: https://arxiv.org/abs/2502.14847 (Verified)
- He et al., "SentinelAgent: Graph-based Anomaly Detection in Multi-Agent Systems": https://arxiv.org/abs/2505.24201 (Verified)
- Tonini et al., "The Arbiter Agent", 2026-06-09: https://arxiv.org/abs/2606.10747 (Verified)
- Alnuhait et al., "Covert Assistance", 2026-09-30: https://arxiv.org/abs/2609.39050 (Verified)
- Li et al., "Who Holds the Pen? Let Specifications, Not Agents, Sign Off", 2026-09-24: https://arxiv.org/abs/2609.29921 (Verified)
- Zhang et al., "Which Agent Causes Task Failures and When?" (Who&When): https://arxiv.org/abs/2505.00212 (Verified)
- Deshpande et al., "TRAIL": https://arxiv.org/abs/2505.08638 (Verified)
- Dang et al., "Multi-Agent Collaboration via Evolving Orchestration" (NeurIPS 2025): https://arxiv.org/abs/2505.19591 (Verified)
- Amayuelas et al., "Self-Resource Allocation in Multi-Agent LLM Systems": https://arxiv.org/abs/2504.02051 (Verified)
- Hong, Troynikov and Huber, "Context Rot", 2025-07-14: https://www.trychroma.com/research/context-rot (Verified)
- Yan, "Don't Build Multi-Agents", Cognition, 2025-06-12: https://cognition.com/blog/dont-build-multi-agents (Verified)

**Classic patterns and human roles**

- Enterprise Integration Patterns (all Verified):
  - https://www.enterpriseintegrationpatterns.com/MessageBroker.html
  - https://www.enterpriseintegrationpatterns.com/ProcessManager.html
  - https://www.enterpriseintegrationpatterns.com/WireTap.html
- Erlang supervisor principles: https://www.erlang.org/doc/system/sup_princ.html (Verified)
- Mediator: https://refactoring.guru/design-patterns/mediator (Verified)
- Wikipedia, all Verified:
  - Blackboard system: https://en.wikipedia.org/wiki/Blackboard_system
  - Open Agent Architecture: https://en.wikipedia.org/wiki/Open_Agent_Architecture
  - Computer-aided dispatch: https://en.wikipedia.org/wiki/Computer-aided_dispatch
  - Assignment editor: https://en.wikipedia.org/wiki/Assignment_editor
  - Foreman: https://en.wikipedia.org/wiki/Foreman
- Genesereth and Ketchpel, "Software Agents", CACM 37(7), 1994, the federation and facilitator architecture (U; I did not read the text)
- RFC 5537, moderated newsgroups: https://www.rfc-editor.org/rfc/rfc5537.html (Verified)
- RFC 4858, Document Shepherd: https://www.rfc-editor.org/rfc/rfc4858.html (Verified)
- Linux kernel, submitting patches: https://docs.kernel.org/process/submitting-patches.html (Verified)
- FEMA NIMS doctrine, 2017: https://www.fema.gov/sites/default/files/2020-07/fema_nims_doctrine-2017.pdf (Verified; text extracted)
- PagerDuty incident commander training: https://response.pagerduty.com/training/incident_commander/ (Verified)
- Scrum Guide 2020: https://scrumguides.org/scrum-guide.html (Verified)
- 14 CFR 91.3: https://www.law.cornell.edu/cfr/text/14/91.3 (Verified)

**Local (read-only, Verified 2026-10-31)**

- Director:
  - `~/.local/share/agent-deck/conductor/director/` files AGENTS.md, POLICY.md, LEARNINGS.md, `monitor_sessions.py`, `heartbeat.sh` and `heartbeat_watchdog.py`;
  - the tail of `task-log.md`, its line count and its entry counts by month;
  - the top-level key sizes of `state.json`.
- agent-deck v1.16.34:
  - `agent-deck --help`, `session --help`, `conductor --help`, `ask --help`, `session annotate --help`, `conductor list`;
  - in John's fork, `~/src/fork/agent-deck/docs/conductor/README.md` and `docs/comms.md` (commit 4f383789, 2026-10-25).
- Mailbox: the names in `~/Mailbox/.agent-mail-presence/` and their modification times (no file opened).
- agent-mail v2:
  - design lines 70, 255, 337, 419 and 471 (forwarding and re-posting);
  - plan line 227 (member record fields) and line 38 (`nix` blocks `+nix`);
  - `git -C /Users/johnw/Products/agent-mail-v2 log`.
