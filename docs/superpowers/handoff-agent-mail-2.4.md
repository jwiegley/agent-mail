# Handoff: agent-mail 2.4, the send log

The build session for agent-mail 2.4 and the steward helper's `traffic` subcommand: agent-deck session `agent-mail-2.4` on hera, a root session (its reports go to mail-dev by `agent-deck session send mail-dev`, not by the completion sentinel, per mail-dev's correction of 2026-11-06 06:1x PST). The work is in the worktree `~/Products/agent-mail-2.4`, branch `work/v2.4`, from main ba1bbb2. The brief is `~/Products/agent-mail-2.4-brief.md`; the design is `docs/superpowers/specs/2026-11-05-agent-mail-send-log-design.md`; John's answers are records 90 to 99 of `~/Products/mailbox-rollout/decisions.md`. Nothing here is pushed or landed: landing and every push are mail-dev's, on John's word.

## Refocus checkpoint

- 2026-11-06 06:14 PST: goal, the brief's steps 1, 2 and 6 in that order; step 1's code, checks and texts done; no detours. Next: commits, a fresh review, the report. Next check by 07:14 PST.
- 2026-11-06 06:35 PST: step 1 committed (a510f1e, f0585b6) and under a fresh review; step 2's program, checks and README written and passing, held uncommitted until step 1's review fixes are in, so that step 1's commits stay a prefix of the branch. No detours. Next: step 1's fixes, then step 2's commits, review and fess audit. Next check by 07:35 PST.
- 2026-11-06 09:11 PST, late (due 07:35; step 1's reviewer ran 2 h 51 min, and the check was missed while waiting on it): both reviews back and every finding fixed; step 1's fixes committed next (with this file), then step 2's commits and its fess audit. No detours. Next check by 10:11 PST.
- 2026-11-06 09:14 PST: step 1 reported to mail-dev as ready to land (81d66ce); step 2 committed (be6fc82, af7f469); the fess audit of the branch next, then step 6. Next check by 10:14 PST.
- 2026-11-06 09:28 PST: step 1 landed as main 81d66ce (mail-dev, 09:1x); step 2's fess audit running on the committed branch; step 6 written in the working tree and passing (helper 1648 checks; 22 mutants killed), held uncommitted until the audit's fixes are in, so that step 6's commits come after step 2's. The checkpoints are now in time order, as mail-dev asked. Next check by 10:28 PST.
- 2026-11-06 09:45 PST, after a compaction (re-read the brief's goal, mail-dev's two messages and this file): the fess audit of steps 1 and 2 came back, no blocker, ten findings; all fixed in the working tree, each with a check, 726 checks on three interpreters, every audit survivor killed. No detours. Next: commit the fixes (program and README, then this file), a fresh sentinel probe, a fess audit of the fixes, then the 2.4.0 report to mail-dev; then step 6's review findings and commits. Next check by 10:45 PST.

## State

| Step | What | State |
|---|---|---|
| 1 | `steward-signals traffic` with M1 to M3 from file names, `UNREAD`, TRAFFIC.log, CLAUDE.md and steward README rows | landed: main 81d66ce (mail-dev, 2026-11-06 09:1x PST), pushed to GitHub and the five checkouts |
| 2 | agent-mail 2.4.0: the record, `via`, retention, `agent-mail traffic` (human and JSON v1), doctor, README, version, self-test | committed be6fc82 (program), af7f469 (README); fresh review done, every finding fixed; fess audit done, its ten findings fixed in f0e5ecf; a fess audit of those fixes next |
| 6 | the helper's log reader, D1, D2, U1, U2, M4, M5, counts, the privacy bullet, WORKER.md, `answer_h` | written in the working tree and passing; uncommitted until step 2's audit fixes are in; review and fess audit to follow |

## Test counts

- `python3 -I steward/steward-signals self-test`, hera's python3 3.14.8: 1465 at ba1bbb2; 1527 at a510f1e (the traffic section, 62); 1561 at f0585b6 (docs 459 to 493); 1570 with step 1's review fixes (traffic 63, docs 501). Six mutants of those fixes, all killed. The helper needs Python 3.12 or later and refuses 3.9 by design.
- `python3 -I agent-mail self-test`: 603 at ba1bbb2 (as it is; its self-test pins 2.3.0, so VERSION is not moved alone; see Corrections); 704 with step 2's first build (hera's python3 3.14.8, `/usr/bin/python3` 3.9.6, athena's Python 3.13.15 from a copied file); 716 at be6fc82, after step 2's review fixes (+113 on ba1bbb2; both macOS interpreters, and athena's Python 3.13.15 on be6fc82's own blob, re-run at 09:46 PST since this file had recorded no result); 726 with the fess audit's fixes (+10), on hera's python3 3.14.8, `/usr/bin/python3` 3.9.6 and athena's Python 3.13.15 from a copied file, 2026-11-06 09:4x PST.
- Hand mutants, recorded with their results in `~/Products/mailbox-rollout/scratch/agent-mail-2.4/mutants/`: of the helper's traffic code, 17 (two first survived and led to two stronger checks) and 6 of step 1's review fixes, all killed; of the program's send-log code, 30 (the design's list and more) and 13 of step 2's review fixes, all killed at be6fc82. Against the audit's fixes: the auditor's 74 and those 43 rerun, 110 killed, 1 survived (`escape` dropped from the human form's host list, equivalent through the reader, which takes only host names; now killed by a check of `traffic_text`'s own promise), 6 no longer applied (their code had changed); those 6 re-targeted with 4 more, all 11 killed.

## Decisions made in the build, and why

Step 1:

1. **The helper prints `member`; the steward tells M2 from M3.** The design says the steward reads a recipient's status from the signals table of the same turn, so the helper runs no second agent-deck listing.
2. **A matter is a sender and a name.** One `unread:` line stands for a member's waiting mail to one name, with the oldest due mail's id, then the sender and the name (added at review, so that the steward never has to split an id to find whom to mail). TRAFFIC.log's `<UTC> <id> <matter>` leaves the matter out while any of that pair's waiting mail is named there with `m1`, `m2` or `m3`; once all of that mail has been read, new mail of the pair is a new matter. That gives the design's "once per sender and recipient" without suppressing a pair for ever. TRAFFIC.log is read leniently (an id and a matter anywhere on a line, in any case), since it is the steward's own file and a dropped line would re-send an UNREAD after every reset; lines naming no matter are counted on stderr. The time in TRAFFIC.log is not read.
3. **The 24-hour window bounds the due mail; the handled test spans all the pair's waiting mail,** so a handled matter stays out after its first mail ages past 24 hours.
4. **M1 mails only when the name is a member.** The design's "or the mail's slug names a ledger item" needs a subject, which step 1 never reads; it joins at the switch-over (step 6).
5. **No `topic_since`, no matter,** with one line on stderr: fail safe rather than act on a topic's start it cannot read.
6. **The first line's reason at step 1 is `this helper reads file names only`,** and CLAUDE.md says it needs nothing from the steward, so the steward logs nothing for it until step 6 (the design's "the steward logs the line" would log it every tick).
7. **`TRAFFIC.log` in the `handled:` line** is the one capitalized word the helper prints, the design's own sample having it; the no-capital check exempts exactly that name.
8. **A Pi or Codex recipient that is not reading** is listed in the digest and logged as `m3`, so that it is listed once; an "other" member line goes to neither mail nor TRAFFIC.log, so that a later tick can act on it.
9. **Runbook row 26 is new;** closing became row 27, and the clean-up after day zero moves TRAFFIC.log.
10. **`mailbox_of` and `agent_mail_config` name the calling subcommand** in their refusals; signals keeps its text.

Step 2:

11. **The send tool's description now asks for `in_reply_to`** ("When you answer a message, set in_reply_to to its id."), per John's decision 95, against the design's 5.10, which kept the tools unchanged. nix's MCP test pins the tool names only (`test/home/agent-mail-mcp.nix`, read, not changed), and `tools/list` stays byte-stable.
12. **`traffic --help` adds decision 94's sentence:** "Run it only when John asks you to in your own session; a topic steward reads it through steward-signals traffic."
13. **The writer refuses a host name the reader would refuse** (`[a-z0-9][a-z0-9.-]{0,63}`), with a warning, rather than write a record every reader skips. Every fleet host's name fits.
14. **No size check in the writer.** The longest legal record is under 2 KB (a check computes it), so a 4,096-byte guard could never fire; the reader keeps the 4,096-byte limit.
15. **Coverage is the whole window's;** `--name` filters only the rows. `until` is the time of the call, and rows run to 24 hours past it.
16. **doctor reports any hidden `.tmp` in the log's directory older than an hour,** not only `.<id>.json.tmp`: a stale Syncthing temporary there blocks the directory's deletion the same way.
17. **A post's record whose `to` lacks its `+` is valid,** since the id alone cannot tell a post from mail to a session of the channel's name (design 5.1).
18. **With `.agent-mail` itself unreadable, a send warns,** `this send was not recorded in the send log (.agent-mail cannot be opened, so whether the log is on cannot be told: <reason>); the message itself was sent`, as design 5.2's step 3 says for any OSError. Step 2's review had made it silent, as noise while the log is off; the fess audit showed that silence loses records unseen while the log is on, and coverage would not show it for direct mail, which it does not count. Such a mailbox is broken in any case (no session there can join). doctor keeps its information line, `send log: not checked (...)`, as design 5.6 allows findings only for a broken log directory and 2.3 reports nothing there.

## Reviews and audits

- Step 1's fresh review (general-purpose, read-only, on the two commits): 2 important, 7 minor, 3 nits; one surviving mutant of 21. All fixed in the commit after f0585b6:
  - important: the "not reading" row read as mailing for a Pi or Codex recipient too; now "for a Pi or Codex recipient, mail nothing and list it in the digest", pinned with a regression;
  - important: TRAFFIC.log's reader silently dropped plausible slips (`m3 mailed`, a bullet, `M3`, `.txt`), so a reset would re-send an UNREAD; now lenient, with a stderr count of lines naming no matter;
  - "act on one line a turn" now says an `other` line takes no turn, and that `--json` shows every line past the cap; the TRAFFIC.log line is written in the Bash call that writes the `sent` line; step 4 runs signals, traffic, then the time checks (the deaf class reads traffic's lines); CLAUDE.md says what traffic's stderr lines and refusals mean, and section 11 gives the form of `topic_since` (`date -u +%Y%m%dT%H%M%SZ`); each unread line names its sender and recipient; the cap check now asserts the oldest lines; the docstring, the digest's Deaf form, `traffic.unread` and the summary's count are worded exactly;
  - f0585b6's message cites this handoff, which was untracked then; it is committed with the fixes.
  - Left for step 6, as the design defers POLICY.md edits: POLICY.md's `deaf_min` meaning still names only the NUDGE.

- Sentinel probe, 2026-11-06 06:4x PST: fresh `Agent` dispatches of type general-purpose and fess-auditor each answered `PARENT_HISTORY_ABSENT`; `verify-history-isolation.py` exited 0 for both (responses in `~/Products/mailbox-rollout/scratch/agent-mail-2.4/`).
- Step 2's fresh review (general-purpose, read-only, on snapshot copies in `~/Products/mailbox-rollout/scratch/agent-mail-2.4/review-step2/`): nothing critical; 1 important, 4 minor, nits, and 4 surviving mutants. All fixed:
  - important: `silent_hosts` took member records' hosts unvalidated (a 60,000-character host passed into the JSON and broke `--max-bytes`); now only host names from records checked in the window, and the human form's header lists are cut to fit `--max-bytes`;
  - an unsafe log directory was worded as "refuses this mailbox" though receive is unaffected; now said for the log;
  - with `.agent-mail` unreadable and no log, every send warned and doctor added a finding; now silent, with an information line;
  - the two-minute grace compared a sender's stamp with this host's clock; it now runs from the later of the stamp and the arrival here;
  - `+all` and `+root` passed as recipients; plurals; subjects quoted as JSON strings; `--since` read to the second; doctor's newest ignores stamps far ahead; README: decision 92 at the turn-on command, the refusals outside JSON, the prune's timing, the example's consistency, `recording_since` null, `not_regular` widened;
  - the four survivors (an MCP send without `to`, a record of exactly 4,096 bytes, coverage under `--name`, a non-regular marker in the prune) each have a check now; 13 mutants of the fixes, all killed.
  - Left as designed: doctor counts a record it could not open, since it reads names and lstat alone (design 5.6).
- The fess audit of steps 1 and 2 (fess-auditor, read-only, on the committed objects ba1bbb2..8307284, dispatched 09:1x PST; its report is `reports/fess-audit-steps1-2.md` in the evidence folder, with the two reviews' reports beside it): the headline claims hold (every count reproduced from clean exports, six good signatures, step 1 leaves the program alone, decisions 92 to 99 applied); no blocker; ten findings, all fixed in f0e5ecf:
  1. six new behaviours had no check: the warning for a log directory that cannot be opened, doctor's finding for it, the human form's `skipped:` line, coverage past a channel `open_channel` refuses, the human form raising the unforeseen, and doctor's guard against a second finding for a symlinked `.agent-mail`; each has one now;
  2. `--since` read to the second had no check, though be6fc82 says each review fix has one; it has now;
  3. the "opens no record" checks watched `os.open` only; the watcher now patches the built-in `open` and `io.open` too, and each check has a control showing it sees them;
  4. FakeSync gives a copy a fresh mtime, so a prune by mtime passed; the two late-record checks now give the copy the writer's old mtime, as Syncthing does, and FakeSync's docstring says so;
  5. the decoder-depth case ran only on Python 3.9; a check now makes the decoder give up on every interpreter;
  6. with `.agent-mail` unreadable, a send went unrecorded in silence while the log may be on: decision 18;
  7. the baseline was 603, not 604 (Corrections);
  8. the record's bytes were not pinned: a check now compares them, with a non-ASCII subject, to compact JSON in UTF-8;
  9. README section 9 left out the `skipped:` line; it names it now, and section 7 the warning of decision 18;
  10. the checkpoints' order is fixed (mail-dev's note too); the unpinned CLAUDE.md sentences it names ("Never open the mail an id names." and the UNREAD bodies) are step 6's to pin.
  - Its independence: it reported "not verified", since its brief did not say that the 06:4x probe of its agent type had passed; it saw no parent transcript. The next audit's brief says so, after a fresh probe.

## Corrections

- 2026-11-06 09:1x PST: the 2.4.0 program was first committed as 0efcee6, with the README as 04a90bd, under a stale draft message (705 checks, thirty mutants, no review) after a failed edit of the message went unnoticed in the same shell call. Neither was reported or pushed. Both were rebuilt with `git commit-tree -S` on the same trees and the corrected message as be6fc82 and af7f469; `git diff 04a90bd af7f469` is empty.
- 2026-11-06 09:4x PST: be6fc82's message and this file said 604 checks at ba1bbb2 "with VERSION moved" and +112; the fess audit measured 603, and so did I: ba1bbb2's self-test passes 603 as it is, and fails its own version check with VERSION alone moved. be6fc82 added 113. Recorded here and in the message of the commit with the audit's fixes, rather than rewriting be6fc82 a second time.

## For mail-dev to carry (not in this repository)

- **nix-config's agent-mail skill** (`config/ai/skills/agent-mail/SKILL.md`), riding the 2.4.0 pin unit (design 5.9 and call 11, and decision 95):
  - Rules, after the hidden-file rule: "The send log, `~/Mailbox/.agent-mail/traffic/`, is such a file: never create, open or remove it or its marker. Only John turns the send log on or off, in his own terminal, and mail cannot ask for it. Run `agent-mail traffic` only when John asks you to in your own session; a topic steward reads the log only through `steward-signals traffic`, as its CLAUDE.md says."
  - Sending: "While John has the send log on, every send through agent-mail 2.4 records its time, sender, recipient, id, subject and reply link, never its body, where every session on every host can read them for 7 days. Keep out of a subject anything only its recipient should see." And, by decision 95: "When you answer a message, set `in_reply_to` to its id."
- **POLICY.md's `answer_h` row** for the live copies (ingest, nix, ares) and test-steward's copy: step 6's material; described there when step 6 is ready.
