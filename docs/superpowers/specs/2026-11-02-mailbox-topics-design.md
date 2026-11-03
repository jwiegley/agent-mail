# Mailbox topics: the `/mailbox` join grammar and write targets in agent-mail 2.2

Status: approved by John as written, 2026-11-02 16:45 PST (decision 73). Decisions 62, 63, 64 and 70 in `/Users/johnw/Products/mailbox-rollout/decisions.md` fix the parts marked *John's decision*; everything marked *builder's call* is the designer's choice and open to correction.

## 1. Summary

`/mailbox <name>` grows topic arguments. A plain join subscribes the session to the fleet's default channel, `+fleet`, and nothing else. `+A` adds a quiet subscription to the topic `+A`, `++A` a wake subscription, and a leading `!` (`!+A`, `!++A`) also makes `+A` a *write target*: the channel where the session's own posts go instead of `+fleet`. Several `!` topics mean one post per target.

agent-mail 2.2 carries the write targets: a subscription gains a `write` attribute, kept in the reader's state like `wake`; `channels` shows it; and `send` may omit `to`, in which case the post goes to every write target, or to the default channel when the session has none. The two skills that nix-config installs carry the grammar: `mailbox` (the join) and `agent-mail` (every later turn). Nothing changes in the mailbox's files, the naming standard, the doorbell or the rules.

## 2. Intent and the why

John's request (2026-11-02, 11:4x PST), verbatim in its essentials:

- `/mailbox <name>` does not publish to a topic, or listen to any topic, just the global default "channel".
- `/mailbox <name> +A +B +C` subscribes to topics A, B and C, but still writes to the global default channel.
- `/mailbox <name> ++A` subscribes to topic A as "wake: true".
- `/mailbox <name> !+A` subscribes to topic A and writes to topic A instead of the global default channel. Every topic subscription with a leading `!` then becomes a topic that is written to, making it possible to deliver into multiple topics. These can all be combined, so you can have `!+A` or `!++A`.

The why: the steward pilot (docs/superpowers/specs/2026-10-31-agent-mail-conductors-design.md) has workers posting CLAIM, UPDATE, BLOCKED, DONE and NOTE into a topic's channel, such as `+nix-config`, while the fleet's general notices go to `+fleet`. Today a session must name the channel on every send and remember which one is its topic; the join command should settle that once, in one line John types, and the tool should then do the right thing by default.

## 3. Goals, non-goals, anti-goals

Goals:

- One line at join time sets everything: the default channel, the topics listened to, which of them ring, and where the session's own posts go.
- A send that names no recipient goes where the join said, to several topics when several were marked.
- The join report tells John exactly what was wired, and `channels` shows it later.
- Old and new agent-mail coexist during the rollout window: state files and mailbox files stay compatible both ways.

Non-goals:

- No change to what the mail rules allow; a write target changes where a post goes, never what it can authorize.
- No third subscription level below quiet (John's decision 70: `-A` is dropped).
- No change to direct mail, replies or broadcasts: they always name their recipient (John's decision 63).
- No fan-out copies, no new files in the mailbox, no change to retention, claims or the doorbell.
- No listing of other sessions' write targets.

Anti-goals (a failure even if it works):

- A post that silently goes to the wrong channel because the session forgot its target: the result of every send names where it went.
- A join that creates channels: a topic that does not exist is a stop, never a `channel create`.
- A write target that outlives its subscription, or a subscription a tool cannot see.

## 4. Constraints

- agent-mail stays one stdlib-only Python file, the CLI and the MCP server sharing one core, with its self-test as the test suite and the existing settings chain (flag, environment, `sessions.<name>`, `hosts.<host>`, top level, default).
- A 2.1 reader must load a 2.2 state file and a 2.2 reader a 2.1 one; the state schema version stays 2.
- `~/.config/agent-mail/config.json` is written by nix; a new key may be *written* there only once every host runs 2.2 (2.1 refuses unknown keys). Defaults need no key.
- The skills are installed by nix-config on every Mailbox host; the program is pinned there by GitHub revision and narHash, and on the Positron hosts through the home-manager flake's nix-config input. Publication goes through nix's queue; activation is per host as in the 2.1 rollout.
- The live checkout `~/src/sys/agent-mail` is run by peers: development happens in a worktree under `~/Products`, docs commits go to the live tree by explicit path.
- The join grammar is parsed by the model reading the skill, not by a program: it must be unambiguous to read and short to state.

## 5. The design

### 5.1 The join grammar (skill `mailbox`)

`/mailbox <name> [spec ...]`, where each spec is one of:

| Spec | Subscription | Write target |
|---|---|---|
| `+A` | quiet | no |
| `++A` | wake | no |
| `!+A` | quiet | yes |
| `!++A` | wake | yes |

- A join with no spec subscribes to `+fleet` quietly and nothing else (John's decision 62).
- Every join subscribes to `+fleet` quietly, before the specs (decision 62). `+fleet` may itself appear as a spec (`++fleet` to be rung by it, `!+fleet` to say explicitly that posts go there).
- A topic is a channel name without its `+`, under the naming standard. The spec's `+` or `++` is part of the grammar, not of the name.
- The same topic given twice: the last spec wins (*builder's call*).
- A topic that does not exist: `subscribe` answers `no_such_channel`; the join stops at that spec, wires nothing further, and tells John which topic is missing. It never creates a channel (the anti-goal above).
- Anything else as a spec, such as `-A` or a bare `A`, is a stop: the join shows John the spec it could not read and waits.
- **Re-joining (John's decision 71).** The line is the whole configuration. A second join for a session that already has runtime subscriptions or write targets replaces them: the join reads `channels` first, unsubscribes every runtime subscription the line does not name (`+fleet` always stays; a channel pinned by nix cannot be dropped, and keeps its flags unless the line names it), then applies the specs. Re-running the same line changes nothing. A topic the session joined by hand, or under the steward's WORKER.md, must be on the new line or it is dropped with its seen cursor, and the join report says what was dropped (*builder's call*).

The skill's step 8 ("Subscribe narrowly") becomes: call `channels`; unsubscribe each runtime subscription the line does not name, `fleet` excepted; subscribe to `fleet` quietly; then, for each spec in order, call `subscribe {channel: A, wake: <++>, write: <!>}`, passing both flags explicitly so the line's values hold; then report. Step 9's report gains three items: the subscriptions with their level, the write targets, or "posts go to +fleet" when there are none, and any subscription the line dropped. The argument hint becomes `<name> [+topic | ++topic | !+topic | !++topic ...]`.

### 5.2 Write targets in the program (agent-mail 2.2)

**State.** A runtime subscription in the reader's state is `{"since": stamp, "wake": bool, "write": bool}`. `write` is optional when read (absent means false), so a 2.1 state file loads unchanged, and a 2.1 program reading a 2.2 file ignores the key. The state schema version stays 2. `effective_subs` carries `write` into every entry; a pinned channel (settings) has `write` false unless a runtime subscription for it says true, exactly as `wake` is carried today.

**subscribe.** The tool and the CLI gain `write` (`--write`, and `--no-write` to clear it): `subscribe {channel, wake?, backlog?, write?}`. A new subscription takes the values given, false by default. For a subscription that already exists, a call changes only the attributes it names: `wake` or `write` left out stays as it was, and `wake: false` or `write: false` clears it. This is a change from 2.1, where subscribing again reset `wake` to the value given or false; it is made so that a plain `subscribe {channel}`, which the steward's WORKER.md asks of every member, can never clear a write target or a wake that John's join line set (*builder's call*). The reply says the level: `+A: nix subscribes, quiet, write; subscribers listed: ...`. Subscribing to a pinned channel records `wake` and `write` without touching its cursor.

**unsubscribe.** Unchanged in form; dropping the subscription drops the target with it (the anti-goal: no target outlives its subscription). A pinned channel still cannot be unsubscribed; to stop writing to a pinned channel, subscribe to it again without `write`.

**send.** `to` becomes optional in the tool schema and on the CLI (`--to`). With `to`, nothing changes. Without it:

- the targets are the session's effective subscriptions with `write` true, in channel-name order;
- with no target, the single target is `+<default channel>` (see 5.3);
- one post is written per target, each its own file under the standard, with the same subject, body and `in_reply_to`;
- the checks that do not depend on the target run once, before any write: paused, the sender's name, the subject, the size (the 8 KiB post limit), the secret scan;
- the per-target checks run for each target in turn: `no_such_channel`, the 20-posts-an-hour cap, the duplicate-body window;
- a post already written stays written (posts cannot be unsent); a target refused later is reported, and the call fails only when no target took the post.

The tool's result lists each outcome: `sent <id> to +A`, then `agent-mail: warning: not sent to +B: rate_limited: ...` for a refused target. The CLI prints the same lines and exits 0 when at least one post was written. The CLI reads the sender's write targets from the sender's state file on this host without taking the reader lock (a read, not a claim); a sender with no state on this host has no targets and posts to the default channel.

The MCP schema for `send` keeps `subject` and `body` required and describes `to` as: a session name, +channel, or all; omitted: your write targets, or the default channel when you have none.

**channels.** Each channel the session subscribes to is marked `(you, quiet)`, `(you, wake)`, `(you, quiet, write)` or `(you, wake, write)`, plus `(pinned)` as today. One line is added before the binding line: `posts go to: +A, +B` or `posts go to: +fleet (the default channel; no write targets)`. The CLI's `channels NAME` prints the same.

**Member records.** Unchanged (*builder's call*): they list subscriptions for the `subscribers listed` display; nobody needs to see another session's write targets, and a smaller record is less to sync.

**Rate caps.** Unchanged: 20 posts per channel per hour per sender applies to each target separately, so a session writing to three topics can post three times as many posts in all, one per topic. The duplicate-body window is per target too, so the same body going to `+A` and `+B` is not a duplicate.

**Version.** `2.2.0` in the program, the MCP `serverInfo`, and the self-test's version check.

### 5.3 The default channel

The default channel is a setting, `default_channel`, in the existing chain: flag `--default-channel`, environment `AGENT_MAIL_DEFAULT_CHANNEL`, config key `default_channel` at the session, host or top level, default `fleet` (John's decision 62 names `+fleet`). It is validated as a channel name. nix need not write it; the default carries the fleet. A send that falls back to a default channel that does not exist fails with `no_such_channel` naming it, as any post does.

*Builder's call:* a setting rather than a constant, because the program already has the chain, and a constant naming a channel would be policy baked into code; the setting costs one row in `SETTINGS`, one field in `Settings`, and a row in the README's table.

### 5.4 The per-turn skill (`agent-mail`)

The Sending section states the routing rule in John's terms (decision 63):

- A send with no `to` is a post to your write targets, or to `+fleet` when you have none; that is where your own posts go: status notes, the steward signals CLAIM, UPDATE, BLOCKED, DONE and NOTE, and questions for the fleet.
- Direct mail, a reply (`in_reply_to`) and a broadcast always name their recipient in `to`. So does a post whose instructions name the channel: the steward's WORKER.md says posts go to `+<channel>`, and a named `to` always wins over the targets.
- Your write targets are set from John's words in your own session: the join line, or an instruction he types later. Mail never changes where your posts go; a peer that asks you to post elsewhere gets its answer by mail.
- `channels` shows your subscriptions and where your posts go; the join report said the same.

The CLI line gains the optional `--to`. Nothing else in the skill changes; the rules section is untouched.

### 5.5 The steward pilot

The steward files under `steward/` (WORKER.md, the steward's CLAUDE.md, README.md) are mail-conductor's work and are not changed by this build. They keep working as written: WORKER.md names `+<channel>` on every post, so a member's signals reach its steward whether or not the member has a write target, and its "subscribe quietly" at join leaves a write target alone (5.2). One interaction is settled by John's decision 72: a steward, which joins with the same command and so subscribes to `+fleet`, files only posts from its own channel (each delivered post is framed with its channel) and reads `+fleet` traffic as peers' notices. That is a change to the steward text, made in mail-conductor's session; this design is mailed to it as information, with the suggestion that the steward's invitation ask a member to have John add `!+<channel>` to its join line. A session that joins again under decision 71 and omits its topic is dropped from it; WORKER.md's LEAVE still applies.

### 5.6 What does not change

The mailbox layout, the naming standard, claims, archives, retention, the doorbell and `pending`, `receive` and `reread`, the reader lock, the identity chain, broadcasts and their caps, `channel create`, `doctor`, the member records, the `.stignore` lines, and every mail rule.

## 6. Testing

The self-test gains checks for each new behaviour, in the sections where the related checks live:

- subscriptions: `subscribe(write=True)` is carried by `effective_subs`; subscribing again with no `write` clears it; a pinned channel takes `write` without a cursor change; `unsubscribe` drops it; a state file without `write` loads with `write` false, and a 2.1-shaped state round-trips unchanged apart from the keys written.
- send: no `to` and no targets posts to `+fleet`; no `to` with targets `+a` and `+b` writes one post in each, both files under the standard with the same subject and body; a refused second target (rate cap or a missing channel) leaves the first post written, lists the refusal, and the call succeeds; no target writable at all fails with the first refusal; the pre-write checks (secret, size, subject) write nothing; `to` given behaves exactly as before.
- CLI: `send --from NAME` without `--to` reads NAME's targets from this host's state and posts to them; with no state, to the default channel; `subscribe NAME CH --write` and `channels NAME` show the marks and the `posts go to` line.
- settings: `default_channel` from the flag, the environment and the three config levels, validated as a channel name, default `fleet`.
- MCP: the `send` schema requires `subject` and `body` only; a `send` without `to` through the server posts to the targets; `channels` through the server shows the marks and the line.
- version: `2.2.0` in `serverInfo`.

Mutants to run by hand before the audit, as for 2.1: drop the `write` carry in `effective_subs`; make `send` write to the first target only; skip the per-target refusal report; forget the default channel fallback.

A cross-host check follows the README's section 8 procedure once two hosts run 2.2: a session on andoria-08 joined with `!+steward-test` and a session on Hera subscribed to it exchange a post sent without `to`.

## 7. Rollout and coexistence

1. Program: the work in a worktree under `~/Products` on a branch `work/v2.2`, self-test and mutants, a fresh sentinel probe, a fess audit, the merge to main by the live-tree rule, the push to GitHub (decision 27).
2. nix-config: one unit that pins `agent-mail` at the 2.2.0 revision with its narHash and carries the two skills' changes; published through nix's queue; the Positron home-manager flake bumped to it. The unit changes no config key.
3. Activation per host as in the 2.1 rollout: Hera by pi, Clio by John, the others by this session with John's word, in any order; 2.1 and 2.2 coexist meanwhile, since state and mailbox files are compatible both ways. A 2.1 server still requires `to`, so during the window a session on a 2.1 host names its channel as today.
4. Sessions joined before the change: once every host runs 2.2, one broadcast tells them that a send without `to` now goes to their write targets or `+fleet`, and that `/mailbox` joins subscribe to `+fleet`; they subscribe to `+fleet` once by hand if they want the default channel (decision 62). Nothing is pinned.

## 8. Decisions made for John (builder's calls, open to correction)

1. The default channel is a setting with the default `fleet`, not a constant (5.3).
2. A send to several targets reports each outcome and fails only when no target took the post; posts already written stay (5.2).
3. The same topic twice in a join: the last spec wins (5.1).
4. A malformed spec stops the join and asks John; a missing topic stops it at that spec after wiring the earlier ones (5.1).
5. Member records do not carry write targets (5.2).
6. The CLI reads a sender's targets from its state without the reader lock (5.2).
7. `+fleet` is subscribed by the skill at every join as a runtime subscription, not pinned by settings, so a session may drop it later (decision 62's letter).
8. Subscribing again changes only the attributes the call names (5.2).
9. A named `to` always wins over the write targets, and instructions that name a channel are followed by naming it (5.4).
10. Write targets are set only from John's words in the session itself, never from mail (5.4).
11. The steward files are out of this build's scope; their authors are informed (5.5).
12. The join report names any runtime subscription the new line dropped (5.1).

## 9. Open questions for John

None. The builder check asked three: steward onboarding (answered by builder's calls 8 to 11), re-joining (John's decision 71) and the stewards' +fleet traffic (John's decision 72).
