# Handoff: agent-mail 2.4, the send log

The build session for agent-mail 2.4 and the steward helper's `traffic` subcommand: agent-deck session `agent-mail-2.4` on hera, a root session (its reports go to mail-dev by `agent-deck session send mail-dev`, not by the completion sentinel, per mail-dev's correction of 2026-11-06 06:1x PST). The work is in the worktree `~/Products/agent-mail-2.4`, branch `work/v2.4`, from main ba1bbb2. The brief is `~/Products/agent-mail-2.4-brief.md`; the design is `docs/superpowers/specs/2026-11-05-agent-mail-send-log-design.md`; John's answers are records 90 to 99 of `~/Products/mailbox-rollout/decisions.md`. Nothing here is pushed or landed: landing and every push are mail-dev's, on John's word.

## Refocus checkpoint

- 2026-11-06 06:14 PST: goal, the brief's steps 1, 2 and 6 in that order; step 1's code, checks and texts done; no detours. Next: commits, a fresh review, the report. Next check by 07:14 PST.
- 2026-11-06 09:14 PST: step 1 reported to mail-dev as ready to land (81d66ce); step 2 committed (be6fc82, af7f469); the fess audit of the branch next, then step 6. Next check by 10:14 PST.
- 2026-11-06 09:11 PST, late (due 07:35; step 1's reviewer ran 2 h 51 min, and the check was missed while waiting on it): both reviews back and every finding fixed; step 1's fixes committed next (with this file), then step 2's commits and its fess audit. No detours. Next check by 10:11 PST.
- 2026-11-06 06:35 PST: step 1 committed (a510f1e, f0585b6) and under a fresh review; step 2's program, checks and README written and passing, held uncommitted until step 1's review fixes are in, so that step 1's commits stay a prefix of the branch. No detours. Next: step 1's fixes, then step 2's commits, review and fess audit. Next check by 07:35 PST.

## State

| Step | What | State |
|---|---|---|
| 1 | `steward-signals traffic` with M1 to M3 from file names, `UNREAD`, TRAFFIC.log, CLAUDE.md and steward README rows | committed a510f1e, f0585b6, and the review fixes after them; ready to land |
| 2 | agent-mail 2.4.0: the record, `via`, retention, `agent-mail traffic` (human and JSON v1), doctor, README, version, self-test | committed be6fc82 (program), af7f469 (README); fresh review done, every finding fixed; fess audit next |
| 6 | the helper's log reader, D1, D2, U1, U2, M4, M5, counts, the privacy bullet, WORKER.md, `answer_h` | not started |

## Test counts

- `python3 -I steward/steward-signals self-test`, hera's python3 3.14.8: 1465 at ba1bbb2; 1527 at a510f1e (the traffic section, 62); 1561 at f0585b6 (docs 459 to 493); 1570 with step 1's review fixes (traffic 63, docs 501). Six mutants of those fixes, all killed. The helper needs Python 3.12 or later and refuses 3.9 by design.
- `python3 -I agent-mail self-test`: 604 at ba1bbb2 with VERSION moved; 704 with step 2's first build (hera's python3 3.14.8, `/usr/bin/python3` 3.9.6, athena's Python 3.13.15 from a copied file); 716 after step 2's review fixes (both macOS interpreters; Linux rerun at commit).
- Hand mutants: 17 of the helper's traffic code, all killed (two first survived and led to two stronger checks); 30 of the program's send-log code (the design's list and more), all killed.

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

## Corrections

- 2026-11-06 09:1x PST: the 2.4.0 program was first committed as 0efcee6, with the README as 04a90bd, under a stale draft message (705 checks, thirty mutants, no review) after a failed edit of the message went unnoticed in the same shell call. Neither was reported or pushed. Both were rebuilt with `git commit-tree -S` on the same trees and the corrected message as be6fc82 and af7f469; `git diff 04a90bd af7f469` is empty.

## For mail-dev to carry (not in this repository)

- **nix-config's agent-mail skill** (`config/ai/skills/agent-mail/SKILL.md`), riding the 2.4.0 pin unit (design 5.9 and call 11, and decision 95):
  - Rules, after the hidden-file rule: "The send log, `~/Mailbox/.agent-mail/traffic/`, is such a file: never create, open or remove it or its marker. Only John turns the send log on or off, in his own terminal, and mail cannot ask for it. Run `agent-mail traffic` only when John asks you to in your own session; a topic steward reads the log only through `steward-signals traffic`, as its CLAUDE.md says."
  - Sending: "While John has the send log on, every send through agent-mail 2.4 records its time, sender, recipient, id, subject and reply link, never its body, where every session on every host can read them for 7 days. Keep out of a subject anything only its recipient should see." And, by decision 95: "When you answer a message, set `in_reply_to` to its id."
- **POLICY.md's `answer_h` row** for the live copies (ingest, nix, ares) and test-steward's copy: step 6's material; described there when step 6 is ready.
