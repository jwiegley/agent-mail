# The metadata send log: what it touches in agent-mail, and the shapes costed

- **Written:** 2026-11-05, about 19:45 to 20:30 PST (2026-11-06 03:45 to 04:30 UTC), by a read-only subagent of the traffic-log design workflow, for the designer of rollout decision 90's metadata send log (decisions:176).
- **Read:** the program `/Users/johnw/src/sys/agent-mail/agent-mail` as it is (VERSION 2.3.0 at am:2559; 6,819 lines, measured) and `README.md` beside it; nix-config's `config/syncthing.nix`, `config/agent-mail.nix`, the agent-mail skill and the Pi extension's `tools.ts`; the steward files and the conductors spec in the worktree `/Users/johnw/Products/agent-mail-a3/`; the earlier gaps note and code map; the 2.2.0, 2.2.1 and 2.2.2 store copies (grep only). In `~/Mailbox`: the top-level listing only (`ls -la`, and `ls -1` filtered by name), and `.stignore`. Nothing below the top level was listed or opened; entry counts of subdirectories come from their link counts in the top-level listing (section 11).
- **Not done:** no git, no agent-mail run, no agent-mail MCP tool, no message file opened, no agent-deck command, nothing written outside this file. This subagent saw no compaction.
- **Anchors:** `am:N` is a line of the program; `README:N` its README; `st.nix:N` `/Users/johnw/src/nix/config/syncthing.nix`; `am.nix:N` `/Users/johnw/src/nix/config/agent-mail.nix`; `skill:N` `/Users/johnw/src/nix/config/ai/skills/agent-mail/SKILL.md` (the installed copy in `~/.config/claude/positron/skills/agent-mail/` has the same lines); `tools.ts:N` `/Users/johnw/src/nix/config/ai/extensions/pi-agent-mail/tools.ts`; `spec:N` `/Users/johnw/Products/agent-mail-a3/docs/superpowers/specs/2026-10-31-agent-mail-conductors-design.md`; `WORKER.md:N`, `CLAUDE.md:N`, `steward-README:N`, `signals:N` the files of `/Users/johnw/Products/agent-mail-a3/steward/`; `gaps:N` `/Users/johnw/Products/steward-native-design/notes/agent-mail-gaps.md`; `code-map:N` `/Users/johnw/Documents/Obsidian/agent-mail-overview-design-20261104/00-code-map.md`; `decisions:N` `/Users/johnw/Products/mailbox-rollout/decisions.md`.
- **Labels:** every number is an estimate, marked **(est.)**, unless marked **(measured)**. Statements about Syncthing's internals are its documented behaviour, not measured on this fleet, and are marked **(Syncthing, unmeasured)**.

## 0. The findings in brief

1. **One hook point.** Every message file is written by `send` (am:1106-1169); the CLI and the MCP tool both reach it, and a send without a recipient calls it once per write target (`send_default`, am:1185-1204). The log line belongs after the warnings are computed (am:1159-1167) and before the return (am:1168-1169): the write has succeeded, the final id is known, and the misdirection warnings exist. A failed append must become a warning, never an error, as the send window's late record already does (am:606-613, tested at am:6200-6211).
2. **Safe shapes under maxConflicts 0.** The Mailbox folder keeps no conflict copy (st.nix:292-302), so of two concurrent edits of one file one is lost without a trace (code-map:18). A synced file must be single-writer or write-once. Per-host append files are the only safe *append* shape; write-once records (one file per send, as mail itself is) are the other safe shape, so "the only safe shape" holds for appends only. Per-sender files, one shared file and the member record are unsafe (section 12).
3. **Where.** Under `.agent-mail/traffic/`: hidden, inside agent-mail's own metadata tree (`open_meta`, am:642-675), never listed by any 2.x version, skipped by every 2.x doctor, matched by no `.stignore` line, and outside the directories the doorbell and the claiming watch observe (am:2131-2138). A file directly at the top level would wake every watcher on every host at every sync (section 10).
4. **Day files remove the rewrite hazard.** One append-only file per host per UTC day, `<host>_<YYYYMMDD>.jsonl`, makes retention the deletion of whole past-day files, which any host may do because a past day's file never changes again; nothing is ever rewritten in place.
5. **Who can record what.** The sender records "sent". The reader can record "read" in its own host's file at `commit` (am:1770-1805); it cannot write the sender's host file. "Answered" exists only as a later send's In-Reply-To, which the replier writes. "Claimed" is already inferable without any write: a direct mail's name leaves the top level before the prune window (section 2).
6. **"Unanswered" is computable only as "no later line names this id".** In-Reply-To is optional and never set automatically (am:1136-1137, 1145, 2578, 3387); only the steward protocol requires it (WORKER.md:18, 44; CLAUDE.md:231, 284-293). The share of ordinary replies that carry it is unmeasured, and the log would measure it from its first day.
7. **No config key.** A new key would be refused by every 2.2.x and 2.3.0 host and force two nix units (am:303-329; gaps:26). A constant, as for posts' 30 days (am:104-108, 1494), needs one.
8. **Traffic today (measured from names):** 183 posts in six channels and 3 broadcasts, all still on disk since the folder began on 2026-10-30; 98 unclaimed direct mails waiting at the top level; 2,236 entries in hera's 41 reader archives (7-day window). **About 100 to 300 sends a day fleet-wide (est.)**, about 400 bytes a line (measured on synthetic lines), so **40 to 120 KB a day and 1.2 to 3.7 MB over 31 days (est.)**.
9. **A discrepancy to settle.** The task context says the four Positron hosts share one copy; README:185 says each keeps its own copy on local disk at `/var/lib/jwiegley/Mailbox` and runs its own Syncthing, with `~/Mailbox` on the NFS home a symlink to that path. The design holds either way if the append lock is host-local (the runtime directory, am:820-835) and never in `~/.local/state/agent-mail` (am:479), which on those hosts is on the shared NFS home.

## 1. The send path

### 1.1 Entry points

| Path | Where | Sender name | Notes |
|---|---|---|---|
| MCP `send` tool | schema am:2574-2578; `McpServer.tool_send` am:2855-2874 | the server's bound name (am:2859) | agent-deck id from `AGENTDECK_INSTANCE_ID` when it matches `IDENT` (am:2860-2861, `IDENT` am:128). With `to`, one `send` (am:2862-2863); without, `send_default` (am:2864-2866). Needs no reader lock: a second server for a name is send-only (README:101; test am:5692-5694). |
| CLI `send` | the `send` branch of `dispatch` (am:3500-3504); `cmd_send` am:3329-3360 | `--from`, checked by grammar only (am:3383, 3501) | body from FILE or stdin (am:3339-3346); without `--to`, `send_default` with the sender's state read without the lock (am:3349-3350). |
| `send_default` | am:1185-1204 | as its caller | one `send` per write target (am:1194-1196): a post to N targets is N files and would be N log lines, with the same subject and In-Reply-To (test am:6274-6279). |

### 1.2 `send`, step by step (am:1106-1169)

| Step | Lines | What |
|---|---|---|
| Refusal order | am:1110-1113 | paused; invalid or reserved name; no_such_channel; broadcast_not_allowed; too_large; looks_like_secret; rate_limited or duplicate; a failed write is send_failed. |
| Paused host | am:1115-1116 | the pause file (`PAUSE_PATH` am:264) stops every send, so nothing would be logged. |
| Kind | am:1118-1123 | `all` is a broadcast (4 KiB, am:1073), `+ch` a post (8 KiB, am:1072), any other name direct mail (1 MiB, `LIMIT` am:100). |
| Directory | am:1124-1130 | a post opens its channel (`open_channel` am:904-920); others the mailbox. |
| In-Reply-To | am:1136-1137 | must parse as a message id (`parse_msg` am:210-224); never checked to exist. |
| Subject | am:1138-1140 | `clean_line` (am:3325-3326) turns every control or format character into a space and collapses whitespace, then the subject is cut to 200 characters (`SUBJECT_MAX` am:1074); empty is no_subject. |
| The id | am:1141-1143, 1169 | `now = clock()`; `stamp = utc(now, micro=True)` (am:452-454); file `<to>_<stamp>_from-<sender>_<8 hex>.txt` with `secrets.token_hex(4)`; the id is the file name without `.txt`. `MSG_RE` (am:116) and `parse_msg` recover recipient, stamp, sender and the unique part from an id. |
| Header | am:1144-1145 | `From: <sender>@<HOST>` plus ` (agent-deck <id>)` when known; `To: <to>` exactly as given (`name`, `+channel` or `all`); `UTC: <stamp>`; `Subject: <line>`; `In-Reply-To: <id>` only when given; `X-Agent-Mail: 2`. No program version is written. `HOST` is the writing process's short host name (am:126). |
| Size and secrets | am:1146-1150 | the whole file is measured; secrets are looked for in subject and body together (am:1077-1084). |
| Write | am:1151-1156 | inside `send_window` (section 1.4), `write_new` (section 1.3); an `OSError` is send_failed. |
| Warnings | am:1159-1167 | direct mail: no reader has checked the recipient, or not for an hour (`last_checked` am:804-817, `PRESENCE_STALE` am:113); a post: nobody lists the channel (`members` am:784-801). These are the send-time misdirection signals. |
| Result | am:1168-1169 | `SendResult(id, path, warnings)` (am:1075). |

### 1.3 The hidden-name write, then the rename (`write_new`, am:1087-1103)

The temporary file is `.<file>.tmp` in the target directory (am:1089), created `O_EXCL | O_NOFOLLOW` with mode 0600 (am:1090); an existing target is refused (am:1094-1098); the rename puts it in place (am:1099); any failure unlinks the temporary file (am:1100-1103). The temporary name matches `.*.tmp`, so Syncthing never carries it (am:3032). **No `fsync` appears anywhere in the program** (measured by grep; the only append is `log()`'s `open(path, "a")` at am:2296), so message files themselves are not forced to disk.

### 1.4 The duplicate and rate caps (`send_window`, am:573-615)

- Per sender per host: the lock `~/.local/state/agent-mail/<dirkey>-<sender>.send.lock` (am:587) and the window `<dirkey>-<sender>.send.json` (am:584), under `STATE_DIR` (am:479), keyed by `dirkey` (am:488-490). The flock is held through the write, because the CLI and MCP senders are separate processes (am:579-582, 589, 606-607).
- The window keeps `[time, target, sha256(body)[:16]]` for an hour (am:586, 608; `SEND_WINDOW` am:482) and drops anything that is not such a triple (am:590-591), so new data must not ride in it (gaps:27).
- `duplicate`: the same body to the same target within 600 s (am:592-595; `DUPLICATE_WINDOW` am:485). `rate_limited`: a 21st post to one channel within an hour (am:596-600; `POSTS_PER_HOUR` am:483), or a second broadcast within 900 s (am:601-605; `BROADCAST_EVERY` am:484). Direct mail has no count cap (am:580-581).
- A window that cannot be saved after the write becomes a warning, not a failure, because failing now would invite a second copy (am:609-613; test am:6200-6211). The log append needs the same rule.
- The window is per host, so a name sending from two hosts has two windows. On the Positron hosts `~` is the NFS home (README:185), so the four share one window file per sender name (inference from am:479 and am:488-495).

### 1.5 Where the append goes

- **H1 (preferred):** after am:1167, before am:1168. The message is in place, the warnings exist (a cheap misdirection field), and the append runs under a per-host log lock of its own. On failure, `warnings.append("this send was not logged (...)")`.
- **H2:** inside the `with send_window` block right after `write_new` (am:1153). It is still serialized only per sender, so it needs the log lock anyway, and it cannot carry the warnings.
- **Never before the write.** A refused or failed send is never logged, and the append never fails a send. Because the subject and body are checked for secrets before the write (am:1149-1150), a secret-bearing subject is refused before it could reach the log.
- **The caller.** `send` does not know whether the CLI or the MCP tool called it; a keyword argument (`via="mcp"` with `Identity.source`, am:2143, 2217-2256, or `via="cli"`) costs two call sites (am:2863-2866, 3350-3352) and `send_default`'s passthrough (am:1185, 1196).

## 2. The receive and claim path: what a reader knows, and who can record what

- **Finding mail.** `candidates` (am:1257-1308) lists the top level for the name's standard-named files and abandoned claims (`parse_claim` am:432-449). A file is read only after 2 s without change by ctime (am:1302-1305, `SETTLE` am:98); another host's claim stands for 600 s (am:1296-1301, `CLAIM_STALE` am:99).
- **The claim.** `collect` (am:1631-1741) refuses unsafe metadata directories first (am:1639-1641), renames each file to `.<name>-claimed-<pid>-<host tag>-<file>` (am:1669, 1676; `claim_name` am:428-429) and reads it whole (`read_file` am:1685, am:1215-1251). At that point the reader knows the id, the asserted sender (from the name), itself, the stamp, the size, the claim time, its host, and every header line; it parses only `From` (`header_from` am:1330-1338), which `render` compares with the name (am:1759-1762). Posts and broadcasts come through `scan_shared` and `read_shared` (am:1365-1401, 1439-1451).
- **Delivery, then commit.** The CLI writes the text, then commits (am:1865-1877); the MCP receive commits in its `done` action (am:2890-2895), which `serve` runs only after the answer line is written and flushed (am:2972-2986). A failed output puts every claim back (am:1620-1628, 1872-1876).
- **`commit`** (am:1770-1805) renames claims into `.<name>-read/` (am:1782), writes post copies (am:1784), updates the seen sets (am:1790-1799) and `last_batch` (am:1800-1801), saves the state (am:1803-1804) and touches the member record (am:1805). Archives never leave their host (`.*-read`, am:3032).
- **Can the reader record "read"?** Yes, in its own host's log file, appended inside `commit` after the archive step, which keeps one writing host per file. It cannot append to the sender's host file without making it two-writer. Volume: one line per direct claim, but one per post per subscriber (each `+ares-fleet` post reaches about 13 readers on hera alone, est. from the archive counts in section 11). It also publishes reading times; the member record already does at 5-minute granularity (am:737-781, `MEMBER_EVERY` am:480).
- **"Answered"** is known only to the replier, at its send, through `in_reply_to`: it is a property of a later "sent" line, never something the original's reader can record separately.
- **"Claimed" needs no new write.** A direct mail whose name has left the top level before `direct_days`, with no prune recorded, was claimed (or removed by hand). doctor's roster already reads waiting mail from names and `lstat` alone (am:3085-3110).
- **"Pruned unread"** is logged today only on the pruning host, in `~/.local/state/agent-mail/log` (am:1602; `log` am:2290-2307), which does not sync. A "pruned" line in the pruning host's traffic file would make "never read" visible fleet-wide; two hosts may prune the same file before the deletion syncs, so readers would dedupe by id.

## 3. Member records

- **Where and what.** `.agent-mail/members/<name>.json` (am:624-625), exactly five keys: `{"v": 2, "name", "host", "checked", "subs"}` (am:757), written to `.<name>.json.<pid>.tmp` and renamed (am:764-768), with the v1 presence file `.agent-mail-presence/<name>` touched alongside (am:770-774).
- **Who writes.** The name's reader at every `commit` (am:1805), the claiming watch's quiet pass (am:1856-1861), subscribe and unsubscribe (forced, am:1061-1067), and the CLI `pending` (am:3266); at most every 300 s unless the subscriptions change, and another host's fresh record is left alone (am:748-756).
- **Limits on reading.** A regular file of ours of at most 65,536 bytes, `v` 2, `host` a string, `subs` a list of strings, `checked` a stamp; extra keys pass (`member_at` am:702-722, size at am:712, validator am:719-721); `members` also needs the name to match (am:795).
- **The 8,000-byte cut.** `channels` caps its text at 8,000 bytes (`CHANNELS_MAX` am:2603), cutting the channel lines and the known sessions built from records first (am:2690-2695, 2704-2713), with per-peer bounds (am:2604).
- **Why it cannot carry a send log.** The record is written by the reader, not by senders, so CLI sends from other processes or hosts would be a second writer; every 2.2.x and 2.3.0 writer rewrites the five keys and drops any other (am:757; gaps:21); the size cap; whole-file rewrites with no lock between `touch_member`'s read and write (am:747, 757-768).

## 4. Retention and pruning

| What | Window | Who prunes | Where | Anchor |
|---|---|---|---|---|
| Archive copies | 7 days by ctime | the owner only | its own host (never synced) | `prune_archive` am:1523-1539; `KEEP_DAYS` am:103 |
| Posts and broadcasts | 30 days by name stamp (by ctime when stamped over 24 h ahead) | each reader, for its subscribed channels and for broadcasts | its host; deletions sync | `prune_posts` am:1542-1572; `RETENTION` am:1494; `expired` am:1511-1515; README:70 |
| Unclaimed direct mail | `retention.direct_days`, default 14, 1 to 365, top level of the config only | every reader, for every name but its own; regular files of ours only | its host; deletions sync; each logged locally | `prune_direct` am:1575-1605; `DIRECT_DAYS` am:109; `prunable` am:1518-1520; README:71 |

- **Schedule.** `prune_due` runs the three at most once an hour per name per host (am:1608-1617; `PRUNE_EVERY` am:1495), called only from `receive` (am:1847) and the receive tool (am:2883). Nothing prunes from `send`, the doorbell, `pending` (am:2487-2488) or doctor (am:3036-3037).
- **Selection is by mail name.** The prunes take only names that `parse_msg` accepts (am:1562-1563, 1591-1592) or the owner's archive, so no existing prune would ever touch a file under `.agent-mail/traffic/`. A 2.4 prune of traffic files would be a fourth step in `prune_due`.

## 5. doctor, and where a new file may live

- **What doctor reports** (am:3035-3149; it writes nothing, am:3036-3037, tested at am:5917-5921). Findings: a missing marker (am:3053-3056); unsafe metadata directories through `mailbox_problems` (am:3057-3058), which checks only `.agent-mail/members`, `.agent-mail-presence` and the named archive (am:685-699, the tuple at am:689); missing `.stignore` lines on `~/Mailbox` (am:3068-3077); conflict copies at the top level, hidden or not (am:3080-3081); non-hidden names at the top level that are not mail (am:3082-3084); `+` entries that are not channels (am:3111-3118); non-hidden bad names in a channel (am:3120-3136); another reader of the name (am:3139-3142); the pause file (am:3143-3144). The roster of waiting direct mail is information, not a finding (am:3085-3110).
- **Hidden names are skipped** at the top level (am:3082) and inside channels (am:3122-3123). The same skips are lines 2961 and 3001 of the 2.2.0 store copy, 2983 and 3023 of 2.2.1, and 3028 and 3068 of 2.2.2 (measured by grep).
- **Nothing lists inside `.agent-mail/` but `members/`.** Every directory listing in the 2.3.0 product code (measured): am:791 (members), am:930, 1372 and 3120 (channels), am:976, 1269, 1372, 1482, 1561, 1590 and 3078 (top level), am:1531 and 2726 (the reader's own archive), am:2345 (the runtime directory). The 2.2.x store copies touch the same metadata paths (their lines 687, 726, 758, 785, 807).
- **So a file under `.agent-mail/traffic/`, or under any hidden top-level directory, draws no finding and no prune on any host running 2.2.0 to 2.3.0.** A channel directory is the wrong place even though a hidden file there also passes silently (am:1373, 3122): README:58 promises it holds posts and the marker only, and the task context's "doctor reports anything else" is true only of non-hidden names.
- **2.4's doctor** can add information lines for the log (per host, the newest line and the day's count) and findings for an unsafe or foreign entry under `.agent-mail/traffic/` (section 16).

## 6. The config and its unknown-key refusal

- `read_config` refuses an unknown key at the top level, in `hosts.*`, in `sessions.*` and in `retention` (am:303-329, refusal at am:317-319, groups at am:321-328), against `TOP_KEYS` (am:274), `FILE_KEYS` (am:269), `SESSION_KEYS` (am:275) and `RETENTION_KEYS` (am:273). A refused file fails every CLI command and every MCP tool on that host: `run_mcp` keeps the problem (am:3000-3004) and `ready` raises it at each call (am:2822-2823). README:106 says so.
- nix writes one file to every home, today with `poll_interval`, `notify`, `max_bytes` and `sessions.conductor-director.max_bytes` (am.nix:12-17).
- **Any new key** (a `retention.traffic_days`, an on-off switch) therefore means the release on all ten hosts first, then a second nix unit (gaps:26). Posts' 30 days and archives' 7 are constants "fixed by the standard" (am:104-108); a `TRAFFIC_DAYS` constant needs no unit.
- **Switch without a key, if wanted:** a file flag like the pause file (am:264, read at am:425) or an environment variable (unknown variables are ignored by old versions, but nix would have to set it in every harness).

## 7. The MCP tool table and the CLI: what adding `traffic` touches

- **The tool table** is static text, `TOOLS_JSON` (am:2573-2597), built with `_tool` (am:2567-2569). `check_args` accepts only `string`, `boolean` and `integer` with optional bounds (am:2611-2635), so filters are strings (a comma list, as the `channels` setting is parsed at am:381-398) or integers. `handle` dispatches with `getattr(self, f"tool_{name}")` (am:2800) to methods returning `(text, done, failed)` (am:2855-2939); results are text only (am:2805).
- **A `tool_traffic`** would call `ready(need_name=False)` as `tool_channels` does (am:2920-2921), since the data is fleet-wide, and bound its text as `channels` (am:2603, 2704-2713) or `receive` (`MAX_BYTES_RANGE` am:266) do.
- **Text that says "seven":** the module docstring (am:65), the section comment (am:2812), `mcp`'s help, which also names them (am:3450-3453), README:78, and the skill (skill:13-15). Pi's extension lists the seven operations (tools.ts:5-11); an unknown tool classifies as `null` (`classifyCall`, tools.ts:32-40), which is right for a call that changes no waiting mail.
- **Pins:** `tools/list` is byte-stable and holds nothing from the mailbox (am:5613-5617); the instructions stay at most 512 characters (am:5618).
- **The CLI:** a subparser in `build_parser` (am:3371-3463) and a branch in `dispatch` (am:3493-3532). `section_help` requires the subcommand on the docstring's Commands lines (am:6074), help and a description (am:6075), every option on its usage line (am:6076-6077), and `--help` exiting 0 for each subcommand (am:6104-6105). README section 9's table (README:219-237) is not pinned by a check.
- **CLI only is an option:** no tool-list change, no "seven" churn, no prompt cost for the many sessions that will never call it. The stewards already run the CLI through Bash (steward-README:342-363).

## 8. The self-test

- **Counting.** `T.ok` raises on a false condition and otherwise adds one check (am:3551-3554); `run_section` returns the section's count (am:6794); `self_test` sums the sections, prints `agent-mail self-test: N checks passed` and stops at the first failure (am:6801-6815).
- **Structure.** 19 sections in `SECTIONS` (am:6750-6770), each in a fresh temporary `HOME` and runtime directory with every `AGENT_MAIL_*`, `AGENTDECK_*` and `CLAUDE_JOB_DIR` variable removed (am:6772-6798).
- **Size.** 510 `t.ok(` call sites (measured by grep); the runtime total is higher because some checks run in loops (am:6104-6105, 6171-6172). Not run here.
- **Tools a log's tests would use:** the injected clock (`t.at`, am:3630-3633) for day boundaries and retention; `t.patch` for constants (am:3635-3638); `FakeSync` (am:6568-6644), whose replicas act with their own `HOST`, host tag and a 7 s clock offset (`on`, am:6592-6601; offsets at am:6578) and carry only names the ignore model passes (`ignored`, am:6561-6565), writing received files as `.syncthing.<name>.tmp` then a rename (am:6639-6642). The pin of the six ignore lines (am:6741-6742) would not change.
- **Where checks would go:** a new `traffic` section; `section_send` (am:6108), `section_tools` (am:5671), `section_mcp` (am:5595), `section_doctor` (am:5883), `section_retention` (am:4657), `section_sync` (am:6647); `section_help` picks up a new subcommand by itself.
- The program must still run on macOS's `/usr/bin/python3` 3.9.6 (gaps:30).

## 9. The identity binding: what proves a sender on a host

- **MCP.** The server resolves its name once (`resolve_identity` am:2225-2256): `--name`, `AGENT_MAIL_NAME`, `AGENTDECK_TITLE`, the `- title:` line of `AGENTDECK_IDENTITY_FILE`, or agent-deck's identity file for `AGENTDECK_INSTANCE_ID` under `identity_roots` (am:2164-2170), each file checked by `identity_title` (am:2196-2214). Every MCP send uses that name (am:2859). `check_me` guards only `receive` and `reread` (am:2274-2277, 2879, 2928).
- **CLI.** Any grammatical `--from` (am:3383, 3501). The `@<host>` in `From` is the writing process's host name (am:1144, 126).
- **Files.** Every file is the same user's on every host (Vulcan runs a second Syncthing as johnw for that reason, README:184); readers refuse other users' files (am:1231-1232).
- **Nothing proves a sender** (README:48; `BANNER` am:132-133; spec:82). A log line inherits that: its `from` is as asserted as the file name's. What a line can add is evidence, not proof: `via` (mcp or cli), the identity source, and the agent-deck id (am:2860-2861). Any process can write a line into any host's file; doing so from another host makes that file two-writer, and Syncthing may then drop the owner's own lines (st.nix:299-301).

## 10. The Mailbox, Syncthing and the watchers

- **The listing (measured, 2026-11-06 03:46 to 03:58 UTC):** six channels (`+ares-fleet`, `+fleet`, `+ingest`, `+nix-config`, `+steward-test`, `+vulcan`); `.agent-mail/` with one entry (`members`); `.agent-mail-presence/` with 60 entries; 41 `.<name>-read/` archives on hera's disk; `.stfolder`; `.stignore`; 98 direct mails and 3 broadcasts at the top level.
- **`.stignore` (measured)** holds a comment and `.*-claimed-*`, `.*-read`, `.*-rejected-*`, `.*.tmp`, `(?d).nfs*`, `(?d).DS_Store`, matching `STIGNORE` (am:3032) and nix's list (st.nix:80-88). A pattern matches any path component at any depth (am:6561-6565). `.agent-mail/traffic/<host>_<day>.jsonl` matches none, so it syncs; a `.tmp` beside it does not. README:187 and README:189 list what crosses and what stays, and would gain the log.
- **The folder (st.nix:292-302):** no versioning, `maxConflicts = 0`; watcher on with a 1 s delay and a 5 s timeout, full rescan hourly (st.nix:133-136); fsync on receivers (st.nix:145); compression of metadata only (st.nix:270); Syncthing 2.1.5 asserted (st.nix:238). These are the home-manager settings for hera and Clio; the NixOS hosts' modules were not read (code-map:18 cites decision 57 for them).
- **Topology.** A star through hera (README:7-11, 181-185): a Positron host's file reaches the other eight through hera.
- **An appended file under Syncthing (Syncthing, unmeasured).** The watcher notices the change within about 1 to 5 s; the file is rehashed whole; peers fetch the blocks whose hashes changed and copy the rest from their old copy; each receiver writes a temporary file and renames it over the old one (README:194; modelled at am:6639-6642), so a reader there always sees a whole version, possibly ending in a partial line if the scan caught a write in progress. Blocks are at least 128 KiB, so **while a file is smaller than 128 KiB, every change re-sends the whole file to every peer** (cost in section 12).
- **The watchers.** The claiming watch and the doorbell observe only the top level and the subscribed channels (`watch_dirs` am:2131-2138), non-recursively: kqueue's `NOTE_WRITE` on each directory (am:1931-1935) and inotify with `IN_CREATE`, `IN_MOVED_TO` and `IN_CLOSE_WRITE` (am:1888, 1891-1892). A log file directly at the top level would wake every watcher on every host at each sync (the receiver's create and rename), and on Linux at each local append too (`IN_CLOSE_WRITE` on a file in a watched directory). A file in a subdirectory wakes none. Each wake costs a scan, at most one every 0.5 s (`MIN_GAP` am:1886).

## 11. Today's traffic and the growth estimate

**Measured from the top-level listing (names and link counts only).** On APFS, which holds hera's `~/Mailbox` (`/dev/disk3s5`, measured with `mount` and `stat`), a directory's link count is its number of entries plus 2 (measured on a scratch directory: 7 entries gave 9 links). So:

| Directory | Entries | Meaning |
|---|---|---|
| `+nix-config` | 63 | 62 posts and the marker |
| `+ares-fleet` | 53 | 52 posts and the marker |
| `+ingest` | 38 | 37 posts and the marker |
| `+fleet` | 29 | 28 posts and the marker |
| `+vulcan` | 5 | 4 posts and the marker |
| `+steward-test` | 1 | the marker |
| top level, mail names | 101 | 98 unclaimed direct mails and 3 broadcasts |
| 41 archives on hera | 2,236 | direct claims plus copies of posts and broadcasts delivered to hera's readers in 7 days |

- **No post has expired yet:** the folder began on 2026-10-30, under the 30-day window (am:1494), so the 183 posts and 3 broadcasts are everything ever posted or broadcast. The three steward channels hold 151 of the posts, most of them recent (est.).
- **Direct mail waiting, by UTC day of stamp:** 8 on 10-30, 38 on 10-31, 12 on 11-01, 3 on 11-02, 8 on 11-03, 13 on 11-04, 16 on 11-05. The oldest groups (`catalog` 10, `nyx-ai` 10, `clio-pi` 8, `keyboard-maestro` 7, `agent-catalog` 6, since 10-30) are mail to names nobody reads, which doctor's roster already shows from names alone.
- **Claimed direct mail is invisible** once claimed. Hera's archives suggest several hundred direct deliveries a week on hera alone after the post copies are subtracted (est.; for example `.nix-read` holds 324 entries and `.pi-read` 322).
- **Rate (est.):** 40 to 80 posts and 60 to 220 direct mails a day now, so **about 100 to 300 sends a day fleet-wide**, most of them from hera's sessions (41 archives there).
- **Line size (measured on synthetic lines with the keys of section 13):** 280 bytes (short id, short subject), 402 bytes (a 72-character id, a 70-character subject and an In-Reply-To), 556 bytes (the longest names and a 200-character subject); 200 four-byte characters add about 600 bytes. A 2 KiB cap per line holds every legal line.
- **Growth (est.):** at 400 bytes a line, 40 to 120 KB a day fleet-wide, 1.2 to 3.7 MB over 31 days; at a stress rate of 1,000 sends a day, 400 KB a day and 12.4 MB over 31 days.

## 12. The shapes, costed

| Shape | Where | Writers of one file | Safe with maxConflicts 0 | Retention | Syncthing transfer a day (est.) | Files kept (est.) | Verdict |
|---|---|---|---|---|---|---|---|
| **A. Per-host day files** | `.agent-mail/traffic/<host>_<YYYYMMDD>.jsonl` | processes of one host | yes | unlink past days beyond the window; any host may, since a past day never changes | a file under 128 KiB is re-sent whole at each sync: about 200·n² bytes per peer for n lines a day, so 4.5 MB per peer for hera at n = 150, and about 45 MB a day of transfers fleet-wide through the star; the 1 to 5 s watcher batching lowers it | at most 10 × 32 = 320 | **the baseline** |
| **A'. Per-host hourly files** | `<host>_<YYYYMMDDTHH>.jsonl` | one host | yes | as A | about 1/24 of A's, about 2 MB a day fleet-wide | 1,000 to 3,000 (hours with sends), 7,440 at most | if A's transfer matters |
| **B. One write-once file per send** | `.agent-mail/traffic/<YYYYMMDD>/<id>.json` | one write, never changed | yes, as mail is | any host unlinks by the id's stamp, as posts expire (am:1511-1515) | the payload only, about 1 MB a day fleet-wide, plus a small index entry per file | 3,000 to 9,000, about 10 times today's synced files | simplest concurrency; most files |
| **C. Per-host single file, trimmed** | `.agent-mail/traffic/<host>.jsonl` or `.traffic/<host>.log` | one host, plus its trim | yes only if the owner alone trims | rewrite the kept tail (as `log()` does, am:2298-2305) | as A while small; after each trim the content shifts, so the whole file goes again | 10 | **not recommended** |
| **D. Per-sender file** | `.agent-mail/traffic/<sender>.jsonl` | every host the name sends from | **no**: `--from` works on any host (am:3383, 3501), a session can be re-created elsewhere (README:60), two readers of a name are only warned about (am:1655-1658) | | | | **unsafe** |
| **E. In the member record** | `.agent-mail/members/<name>.json` | the name's reader, and every old writer | **no** for history: old writers keep five keys (am:757; gaps:21); 64 KiB cap (am:712); written by the reader, not the sender | | | | **unfit** |
| **F. A hidden file at the top level** | `~/Mailbox/.traffic-<host>.log` | one host | yes | as A or C | as A, plus a wake of every top-level watcher on every host at each sync, and on Linux at each local append (section 10) | | **avoid; use a subdirectory** |
| **G. One shared file** | `.agent-mail/traffic.log` | every host | **no** | | | | **unsafe** |

- **Why per-host appends are safe:** each file has one writing host, so Syncthing never sees two devices change it between syncs; every writer on that host serializes on a local lock; receivers only ever replace their copy whole.
- **Why write-once is safe too:** a unique name written once is never changed, which is why mail itself survives `maxConflicts = 0`. B is the shape that needs no lock, never shows a partial line, and can be pruned by anyone; it costs one file per send, a listing of a growing directory per read (est. a few thousand entries), and a file open per line read.
- **A host-local log** (in `~/.local/state`) is not a candidate: the steward on hera would see only hera's sends.
- **The directory:** `.agent-mail/traffic/` keeps agent-mail's synced metadata under one root that `open_meta` already guards and README:187 already names, and `.agent-mail` exists on every host already. A new top-level `.traffic/` works equally for doctor, the prunes and the watchers.
- **A flat directory** (`<host>_<day>.jsonl`; `_` never occurs in a name, am:114) is one listing per read; a directory per host (`<host>/<day>.jsonl`) is equally safe. Never remove a host's directory: another host may be writing into it.

## 13. What one line holds

JSON Lines, one object per line, at most 2 KiB, ending in `\n`, illustrated with invented values:

    {"v":1,"ev":"sent","utc":"20261105T214048.305329Z","host":"hera","from":"conductor-nix-steward","to":"searxng","id":"searxng_20261105T214048.305329Z_from-conductor-nix-steward_f4605e0a","subject":"...","re":null,"bytes":1083,"via":"mcp","warn":[]}

| Key | Source | Note |
|---|---|---|
| `v` | constant | a format version, as `pending --format json` carries (README:251) |
| `ev` | constant | `sent`; room for `read` and `pruned` (section 2) |
| `utc` | the id's stamp (am:1142) | the same text as the name and the `UTC:` header, so joins are exact |
| `host` | `HOST` (am:126) | must equal the file name's host; readers reject a mismatch |
| `from` | `sender` | as asserted (section 9) |
| `to` | `to` as the `To:` header writes it (am:1144) | keeps the `+` of a post and `all`; the id alone cannot tell a post from direct mail to a session of the channel's name |
| `id` | am:1143, 1169 | readers check that its recipient, stamp and sender agree with `to`, `utc` and `from` (`parse_msg` am:210-224) |
| `subject` | the cleaned subject, `line` (am:1138) | at most 200 characters, no control characters (am:3325-3326) |
| `re` | `in_reply_to` or null | grammar only (am:1136-1137) |
| `bytes` | `len(data)` (am:1146) | the file's size |
| `via` (optional) | a new `send` argument | `mcp` (with the identity source) or `cli` |
| `warn` (optional) | the warnings of am:1159-1167, as codes | `no_reader`, `stale_reader`, `no_subscribers`: misdirection at send time |

- **Redundancy is deliberate:** the id encodes recipient, stamp and sender, and the explicit fields make every line checkable against its id.
- **TSV is the alternative:** `clean_line` leaves no tab in a subject and every other field is grammar-checked, so a tab-separated line with the subject last cannot be ambiguous, and it cannot nest. JSON matches the program's other formats (state, member records, `pending --format json`), and a reader must then catch `RecursionError` as `member_at` does (am:715).
- **Size per day file:** a reader cap (say 4 MiB, about 10,000 lines, est. 50 times hera's day) keeps a planted giant file from stalling a read, as `member_at`'s 64 KiB (am:712) and `read_file`'s 1 MiB (am:100) do.

## 14. How a line is written

1. Build the line after am:1167 (section 1.5).
2. Open `.agent-mail/traffic` through `open_meta(d, ".agent-mail/traffic", create=True)` (am:642-675): one component at a time, no symlink followed, mode 0700 when made.
3. Open the day file relative to that descriptor with `O_WRONLY | O_APPEND | O_CREAT | O_NOFOLLOW | O_NONBLOCK`, mode 0600; check by `fstat` that it is a regular file of ours with one link and under the cap.
4. Lock: an `flock` on the day file works, because every writer of it is a local process on local disk (APFS on hera, measured; the Positron copies are on local disk per README:185). A lock in the runtime directory (am:820-835, `/tmp/agent-mail-<uid>`, local and 0700) is equivalent and also holds if a mailbox were ever on NFS (`fs_type` detects that, am:2638-2654). Never in `STATE_DIR` (am:479), which the Positron hosts share over NFS.
5. If the file is not empty and its last byte is not `\n` (a torn earlier write), write `\n` first, so one torn line never swallows the next.
6. One `os.write` of the whole line; a short write is an error.
7. Close, unlock. Any `OSError` or `MailError` becomes the warning "this send was not logged (...)"; the send has already succeeded.

- **fsync: none,** at parity with message files (section 1.3). A crash can lose the last lines or leave a torn one; readers skip any line without its `\n` and any line that fails validation. Receivers fsync their copies (st.nix:145).
- **The README rule "never write a message in place"** (README:44) is about messages; the log is not one, and no mail path reads it.
- **Cost per send (est.):** one `open_meta` walk (two `openat` calls), one open, one `fstat`, one lock, one write: well under a millisecond on local disk.

## 15. Retention

- **Window:** a constant `TRAFFIC_DAYS`. 31 days outlives every post (30) and the default direct-mail window (14); `direct_days` may be set as high as 365 (am:267), so a long-waiting mail could outlive its line, which is harmless.
- **Who trims (shape A):** a fourth step in `prune_due` (am:1608-1617), once an hour per reader per host, unlinking `<host>_<day>.jsonl` files whose day is older than the window, for every host. A past day's file never changes, since a host writes only the day its own clock names, so a deletion never races an append; if a host with a clock 31 days slow did append to an expired day, Syncthing resolves delete-against-modify in favour of the modification (Syncthing, unmeasured), which resurrects the file harmlessly. The owner-only alternative never cleans up after a retired host; readers skip out-of-window days either way, and doctor could report a host whose newest line is old.
- **The rewrite-in-place hazard (shape C):** a trim is a second writer of the same path. Without the appenders' lock it loses lines appended between its read and its rename, as `log()` can today (am:2296-2305); with it, appends wait on the trim. A trimmed file's content shifts, so each trim re-sends the whole file (Syncthing, unmeasured). And if any host but the owner ever trimmed, the file would have two writers and one side would be dropped silently (st.nix:299-301). Shape A never trims.

## 16. How it is read

- **CLI (est. syntax):** `agent-mail traffic [--since STAMP | --hours N] [--member NAME[,NAME]] [--channel CHANNEL[,CHANNEL]] [--format human|json] [--max-bytes N]`. Read-only and writes nothing, like doctor and `pending`. `--member` matches `from` or `to`; `--channel` matches `to = +channel`.
- **MCP (optional):** `traffic {since?, member?, channel?, max_bytes?}`, the eighth tool (section 7).
- **Reading:** list `.agent-mail/traffic/` through `meta_fd` (am:677-682); take names of the pattern whose day is in range; open with `O_NOFOLLOW | O_NONBLOCK`; accept a regular file of ours with one link under the cap (as `read_file`, am:1215-1251); keep `\n`-terminated lines under 2 KiB that parse and validate; filter; sort by `(utc, id)`, the senders' clocks, as mail is sorted (`sort_key` am:457-460). Report per host: the day files present, the newest line, malformed lines.
- **Output:** subjects are peer text: shown through `sanitize` (am:254-258), names escaped (am:227-233) and clipped (am:2657-2666), under a banner like `BANNER` (am:132-133), bounded and cut at a line with a marker as `channels` is (am:2704-2713). A JSON form with a contract version, as `pending --format json` has (README:239-262), lets the steward helper read it by subprocess.
- **doctor (2.4):** information lines per host (newest line, lines today) and findings for an unsafe `.agent-mail/traffic` (through `open_meta`), and for entries there that are not regular files of ours, are hard-linked, oversized or misnamed. Whether `mailbox_problems` (am:685-699) should include the directory is a choice: if it does, a planted symlink there stops every 2.4 receive (`collect` refuses first, am:1639-1641), as one at `.agent-mail/members` does today (README:172).
- **The rule that blocks reading the files directly:** the skill says "Never read, move or delete another session's mail, or any hidden file in the mailbox that is not yours" (skill:38-39), and the steward rule allows only listing names (spec:101, John's decision 9 at spec:578). The command is the sanctioned reader; the skill and the steward texts need a line saying so.
- **A doctor-only reader is too little:** doctor's job is health (am:3035-3039, exit 1 on findings, am:3511-3516), and a per-send listing would bury its findings.

## 17. Mixed versions and coverage

- **Hosts on 2.2.x or 2.3.0:** Syncthing carries `.agent-mail/traffic/` to them (no ignore line matches, am:3032); their agent-mail never lists it (section 5): no finding, no prune, and none of their sends is logged.
- **A switched host:** CLI sends are logged from their next run. An MCP server keeps the code it started with until its harness restarts (skill:56-58), so long-lived sessions stay unlogged for hours or days after their host switches.
- **The order:** hera has switched first in earlier releases (gaps:48), so the stewards' own host would log first.
- **Measuring coverage:** posts stay on disk 30 days (am:1494), so for each channel the posts present since the log began, compared with the `sent` lines, give exact coverage per sender, and the unlogged senders name the old servers. For direct mail only names still in transit can be compared, a lower bound, as the helper's transit census is today (`census`, signals:613-637, the count at signals:630-631). Hand-written messages (README:44) are never logged and show the same way.
- **The reader must say it:** per host, whether a day file exists for each day in range; a host without one is "unknown", never "silent".

## 18. Duplicates, unanswered mail and misdirection: what the lines support

- **In-Reply-To today.** Set only when the sender passes it (am:2863, 2866, 3387), checked only for grammar (am:1136-1137), written only then (am:1145); the frames show each item's id (am:1762), so a replier has it. The skill describes `in_reply_to` but does not require it (skill:140-153), and a reply sent without `to` becomes a post (skill:152-153). The steward protocol requires it for `ANSWER` (WORKER.md:18), `DECISION` relays (WORKER.md:44; CLAUDE.md:284-293) and `ROUTE` (CLAUDE.md:231). **The share of ordinary replies that carry it is not measured here** (no message file was read). Two cheap levers: the send tool's `in_reply_to` description (am:2578) and the skill's sending section; sessions see the tool text only after a restart.
- **Unanswered:** a direct mail (or a `QUESTION` post, typed by its subject's first word, WORKER.md:18) with no later line whose `re` names it within T. It over-counts by every reply sent without In-Reply-To and every mail that needs none ("No reply needed" lives in bodies, skill:161-162, which the log never holds). A fallback: a later send from the recipient back to the sender within T counts as a probable answer. Whether it was read at all: its name still at the top level means unread (section 2).
- **Duplicates:** the same subject from several senders, or one sender's same subject to several recipients, within a window; today's cap stops only an identical body to one target from one sender on one host within 10 minutes (am:592-595). A body digest (the window's `sha256(body)[:16]`, am:586) would catch identical bodies across senders and targets, but it is derived from the body, which decision 90 excludes ("bodies never").
- **Misdirection:** the send-time warnings as codes (am:1159-1167); mail to a name with no member record or a stale one; posts to a channel no record lists; and, with `pruned` lines, mail deleted unread after `direct_days`.

## 19. The privacy change

- **Today:** a direct mail's subject is in its file only. The name (recipient, sender, stamp) is visible on every host while it waits (README:57; doctor's roster, am:3085-3110); the content is readable by any process of the user but forbidden by rule (skill:38-39; spec:101); once claimed it lives only in the reader's archive, on its host, for 7 days (am:1523-1539; README:72; `.*-read`, am:3032). `pending --format json` was designed to hold no id, sender or subject "so all of it may be shown" (README:239; am:2507-2508).
- **With the log:** every subject, on every host, for the whole window, readable by the command and by any process.
- **Narrower options, none needing a key:** omit direct mail's subjects and keep them for posts, which subscribers see anyway; keep only the first word (the steward verbs, WORKER.md:17-18); keep a hash (equality only); keep direct mail's subjects for a shorter window.
- **Secrets:** a subject with a key or token is refused before the write (am:1149-1150), so it never reaches the log.
- **Re-posting:** subjects are peer text; a steward's summaries should count and name ids and senders rather than quote subjects (the never-re-post rule, am:2561-2564).

## 20. What it costs (all est.)

Calibrated as the gaps note does: self-test 2.5 to 4 times the product lines, 0.5 to 0.9 checks per product line (gaps:6).

| Part | Product lines | Notes |
|---|---|---|
| Writer: format, `open_meta`, lock, append, warning, `via` passthrough | 45-70 | `send`, `send_default`, `tool_send`, `cmd_send` |
| Retention: the day-file prune in `prune_due` | 15-25 | |
| Reader: list, check, parse, validate, filter, sort, coverage | 50-80 | |
| Output: human and JSON, sanitized, bounded | 30-50 | |
| CLI subcommand and docstring lines | 12-20 | `section_help` pins |
| MCP tool (optional) | 15-25 | plus "seven" in five places |
| doctor lines and findings | 15-30 | |
| `read` and `pruned` events (optional) | 20-35 | `commit`, `prune_direct` |

- **Total:** about 180 to 300 product lines without the options, 200 to 335 with them; self-test 450 to 1,300 lines; 90 to 300 checks; README 20 to 40 lines (sections 3 to 5 and 7 to 9); the skill 3 to 6 lines (the tool count, the hidden-file rule). This is as large as the gaps note's six candidates together (gaps:182, 215 to 355 product lines), the largest release since 2.0.0.
- **Release:** one nix pin unit (no key), its gates and ten activations (gaps:30), then the sessions' restarts for the MCP side. The steward's use rides the light path afterwards (gaps:183).
- **Running cost:** about a millisecond per send; 1.2 to 3.7 MB on every host's disk for 31 days; Syncthing transfer as in section 12 (tens of MB a day through hera for shape A in the worst case, a few MB for A', about one for B).

## 21. Discrepancies with the task's context

1. **The Positron copy.** README:185 says each Positron host has its own copy on local disk and its own Syncthing; the context says one shared copy. Either way, per-host files stay single-writer; only the lock's place matters (section 14).
2. **"The only safe shape."** True of appends; write-once records (shape B) are safe too.
3. **"A channel directory may hold only posts and its marker (doctor reports anything else)."** doctor reports only non-hidden names there (am:3122-3123; README:58).
4. **"Append-only."** Shape A is append-only within each file; retention deletes whole files.

## Open questions

1. Shape A (day files), A' (hourly files) or B (one write-once file per send)?
2. `.agent-mail/traffic/` or a new top-level `.traffic/`; flat names or a directory per host?
3. Direct mail's subjects: whole, first word only, hashed, or omitted (posts keep theirs)?
4. A body digest for duplicates, against "bodies never"?
5. The window: 31 days, or 15 to cover only the default `direct_days`?
6. `read` lines from readers and `pruned` lines from pruners, or `sent` only?
7. Refused sends (`duplicate`, `rate_limited`, loops) as code-only lines, without subjects?
8. CLI only, or also an eighth MCP tool for every session?
9. doctor: information only, or findings (exit 1) for the log's problems; and should `mailbox_problems` cover `.agent-mail/traffic`, so that a planted symlink there stops every 2.4 receive?
10. Prune past days by any host, or by the owner only?
11. JSON Lines or TSV?
12. `via` and the identity source in each line?
13. Should the send tool's `in_reply_to` text and the skill ask every reply to carry the id, so that "unanswered" means something?
14. Is the layout of README:185 (a local copy per Positron host) current?
15. Who may read the log: may the stewards' helper call `agent-mail traffic --format json`, and what line in the skill and in the steward texts (skill:38-39; spec:101) allows it?
