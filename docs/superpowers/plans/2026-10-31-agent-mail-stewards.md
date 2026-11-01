# Per-topic stewards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build everything the approved steward design puts in the agent-mail repository: the steward's instruction files, the worker's section, the read-only signals helper with its self-test, the two patches for John's director runtime with their tests, and the day-zero runbook, so that John can create the two steward conductors the day the preconditions land.

**Architecture:** All deliverables live under `steward/` on branch `work/conductor`. The model-facing files are Markdown that `agent-deck conductor setup` links into each conductor's directory; the one program, `steward/steward-signals`, is a single stdlib-only Python 3 file with a `self-test` subcommand, like `agent-mail` itself, and it reads only metadata (file names, ctimes, agent-deck JSON), never a message body. The patches are produced against copies of John's unversioned `monitor_sessions.py` and `bridge.py` and tested there; John applies them.

**Tech Stack:** Python 3.12+ standard library only (no third-party imports); Markdown; `agent-deck` 1.16.34 CLI (read-only subcommands); unittest for the patch tests, run in John's conductor venv.

**Status:** approved by John on 2026-10-31 (decision 19), to be executed subagent-driven on `work/conductor`; amended the same day for decision 20 (no steward cost measurement: Task 1 has 19 policy keys, Task 5 is the metrics file alone); `mail-dev` confirmed the six v2 behaviours Task 1 of the design's requests relies on (main b2b912c; program unchanged at 5be5acd).

**Spec:** `docs/superpowers/specs/2026-10-31-agent-mail-conductors-design.md` (approved 2026-10-31). The plan argues from it; executors read both. The drafts the Markdown starts from are `docs/superpowers/research/2026-10-31-conductor-exploration/operations.md` sections 4 and 5; where a draft and the spec differ, the spec is right (spec 7.1 lists the differences).

## Global Constraints

- Python: `#!/usr/bin/env -S python3 -I`, standard library only, one file, `self-test [SECTION ...]` runs in a temporary directory with `HOME` redirected and never touches `~/Mailbox` or agent-deck (spec 5.2 conventions; agent-mail's own self-test is the model).
- The helper never opens a message file. It reads directory listings (`os.scandir`), `st_ctime`, member records (`.agent-mail/members/<name>.json`, metadata), `SENT.log`, `NOTES.log`, `TOPIC.md`, `POLICY.md` and agent-deck JSON (spec 6.5).
- Names follow agent-mail's grammar: `[a-z0-9](?:[a-z0-9.]|-(?!-))*`, at most 64 characters, not ending in `-` (spec 5.2). Message ids are file names without `.txt`: `<to>_<UTC>_from-<from>_<hex8>` with `<UTC>` as `YYYYMMDDTHHMMSS.ffffffZ`.
- Exact copy, pinned by the self-test's `docs` section: the question-line template `Q <prefix>-<n>: <text>: A <x>, B <y>?` of at most 80 characters, using only letters, digits, spaces and `- . , : ( )`; the STATUS line `<UTC> tick=<min> open=<n> active=<n> waiting-john=<n> last-post=<UTC> doorbell=<ok|missing>`; the page text `NEED(steward): <steward> STATUS unchanged since <UTC> (tick <m>m, <n> open)`; the ledger states `unclaimed active blocked waiting-john done dropped orphaned`; the message-template subjects in spec 6.6; the summary form `SUMMARY +<channel> <date>: …` of at most 400 bytes with no `?`.
- Thresholds live only in `steward/POLICY.md` (spec 6.5, 7.1); the starting values are the spec's: stale 45 min, idle 20 min, question 30 min, nudge gap 30 min, 2 nudges per item per day, busy-stuck 2 h, deaf 15 min, offer 3 items every 2 h, 6 sends per turn, 20 per hour, ledger 8 KB, logs 14 days, back-after 30 min, doorbell delay 5 min, tick 15 min active and 60 otherwise, summary at 07:30 America/Los_Angeles.
- No `?` appears in any text the steward prints except the question line; no `?` in `SUMMARY`, `STATUS` or template subjects (spec 6.4 step 7, 6.7).
- The steward never uses `agent-deck session send`, `launch`, `stop`, `restart`, `set-parent`, `notify_discord.py`, or `inbox drain` (spec 4, 6.6); the instruction files must say so and the `docs` check must find those words only in a "never" sentence.
- Nothing here modifies `agent-mail`, its README's rules, or any file outside `steward/` and `docs/`, except one pointer paragraph in the top-level `README.md` (Task 12). John's `monitor_sessions.py` and `bridge.py` are read-only; the patches are made against copies (Tasks 10, 11).
- Every task ends with a commit whose message ends with two attribution lines: a `Co-Authored-By:` line naming the model that wrote the commit (the implementer's own session reminder gives it, for example `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`), and this session's `Claude-Session: https://claude.ai/code/session_018B4HUohdwotbPJprp7dr7q`.

## Review Focus

1. A channel directory with thousands of 30-day posts: the census must stay under a second and read no file. Pinned in Task 3 (`test_census_scales`).
2. Member titles with dots and digits (`glm-5.2`, `glm-5.3-flash`, `dsv41f`) in every name regex, and a member named as a prefix of another (`pi` vs `pipeline`): exact matching only. Pinned in Task 3 (`test_names_are_exact`).
3. agent-deck JSON that is an object with a `sessions` key instead of a bare array, or rows missing `substate`, `background_work` or `transcript_path`: parse what is there, mark the rest unknown, never crash. Pinned in Task 2 (`test_list_shapes`).
4. A `metrics/<date>.json` left corrupt or hand-edited: the merge must never crash the steward's turn or lose the counters it was given; the bad file is moved aside to `<date>.json.bad` and a fresh one written. Pinned in Task 5 (`test_metrics_corrupt_file`).
5. A `POLICY.md` John edited with a misspelt key or a non-numeric value: refuse with the key named, never fall back to a default silently. Pinned in Task 1 (`test_policy_refuses_bad_values`).

---

## File structure

```
steward/
  README.md               what the pieces are, John's commands (channels, conductor setup, pane lines), the day-zero runbook
  CLAUDE.md               the steward's instructions (conductor setup -instructions-md)
  SHARED.md               steward-safe shared conductor instructions (-shared-instructions-md)
  HEARTBEAT_RULES.md      (-heartbeat-rules-md)
  POLICY.md               thresholds and caps: one Markdown table the helper parses; John edits the conductor's copy
  POLICY-test.md          the same keys with short values for day zero
  WORKER.md               the stewarded-topic section workers follow; names steward-questions.md
  steward-signals         Python 3, stdlib: signals, probe, metrics, self-test (incl. docs checks)
  patches/
    README.md             how John applies and tests the two patches
    monitor_sessions-steward.patch
    bridge-steward-routing.patch
    test_steward_patches.py   unittest for both, run against patched copies in John's venv
README.md                 one new paragraph pointing at steward/ (Task 12)
```

`steward-signals` internal layout (one file, top to bottom): constants and regexes; `load_policy`; `load_roster`; agent-deck readers; census; probe; metrics; the `signals`, `probe`, `metrics` commands and their text/JSON rendering; `self_test` with sections `policy deck census probe metrics signals docs`; `main`.

---

### Task 1: POLICY.md and its parser

**Files:**
- Create: `steward/POLICY.md`, `steward/POLICY-test.md`
- Create: `steward/steward-signals` (skeleton: `main`, `self_test` harness, `load_policy`)

**Interfaces:**
- Produces: `load_policy(path: str) -> dict[str, int | float | str]` with keys exactly: `stale_min idle_min question_min nudge_gap_min nudges_per_item_day busy_stuck_h deaf_min offer_items offer_every_h sends_per_turn sends_per_hour ledger_kb log_days back_after_min doorbell_delay_min tick_active_min tick_idle_min summary_local_time summary_tz` (19 keys). Raises `PolicyError(key, reason)` on a missing or unknown key or a non-numeric value where a number is required. `POLICY.md` is a Markdown table with columns `key | value | meaning`.
- Produces: the self-test harness `ok(cond, what)` and `capture(fn, *args)` and `run_section(name)` in the style of `agent-mail self_test`.

- [ ] **Step 1: Write the failing tests** in `self_test`, section `policy`:

```python
def test_policy_parses_spec_values(d):
    p = load_policy(os.path.join(REPO, "steward", "POLICY.md"))
    ok(p["stale_min"] == 45 and p["idle_min"] == 20 and p["question_min"] == 30, "policy: spec thresholds")
    ok(p["nudges_per_item_day"] == 2 and p["busy_stuck_h"] == 2 and p["deaf_min"] == 15, "policy: nudge and deaf values")
    ok(p["tick_active_min"] == 15 and p["tick_idle_min"] == 60 and p["doorbell_delay_min"] == 5, "policy: ticks and delay")
    ok(p["summary_local_time"] == "07:30" and p["summary_tz"] == "America/Los_Angeles", "policy: summary time")

def test_policy_refuses_bad_values(d):
    bad = write(d, "POLICY.md", TABLE.replace("| stale_min | 45 |", "| stale_min | forty-five |"))
    ok(raises(PolicyError, load_policy, bad).key == "stale_min", "policy: non-numeric value names its key")
    missing = write(d, "POLICY.md", TABLE.replace("| idle_min | 20 | …|\n", ""))
    ok(raises(PolicyError, load_policy, missing).key == "idle_min", "policy: missing key named")
    unknown = write(d, "POLICY.md", TABLE + "| idel_min | 20 | typo |\n")
    ok(raises(PolicyError, load_policy, unknown).key == "idel_min", "policy: unknown key named, no silent default")
    test = load_policy(os.path.join(REPO, "steward", "POLICY-test.md"))
    ok(set(test) == set(p := load_policy(os.path.join(REPO, "steward", "POLICY.md"))) and test["stale_min"] < p["stale_min"], "policy: test policy has the same keys and shorter values")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test policy`
Expected: exit 1, `steward-signals self-test FAILED: policy: …` (or ImportError-style failure for the missing file).

- [ ] **Step 3: Write `steward/POLICY.md`** as a Markdown table `| key | value | meaning |` with the 19 keys above and the spec's starting values (Global Constraints); `steward/POLICY-test.md` with the same keys and values suited to day zero (`stale_min 2`, `idle_min 1`, `question_min 1`, `nudge_gap_min 1`, `busy_stuck_h 0.1`, `deaf_min 1`, `tick_active_min 1`, `tick_idle_min 2`, the rest unchanged). Implement `load_policy` and `PolicyError` in `steward/steward-signals`, plus `main` with subcommand `self-test [SECTION ...]`.

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test policy`
Expected: `steward-signals self-test: N checks passed`

- [ ] **Step 5: Commit**

```bash
git add steward/POLICY.md steward/POLICY-test.md steward/steward-signals
git commit -m "Add the steward policy table and the signals helper's skeleton"
```

### Task 2: The roster and the agent-deck readers

**Files:**
- Modify: `steward/steward-signals`

**Interfaces:**
- Consumes: `load_policy` (Task 1).
- Produces: `load_roster(topic_md: str) -> list[Member]` where `Member = (name, host, harness)` parsed from the `## Members` table in `TOPIC.md` (columns `name | host | harness`), refusing a name that fails the agent-mail grammar with `RosterError(name)`. `deck_rows(run: Callable, members, hera_host="hera") -> dict[name, Row]` where `Row = dict(status, substate, bg, last_activity, archived: bool | None, host_ok: bool)`; `run(argv) -> (stdout, returncode)` is injected so tests need no agent-deck. Local members come from `agent-deck list --json`; remote members from `agent-deck remote sessions <host> --json --with-errors`, one call per host. A failed host sets `host_ok=False` on every member of that host and leaves the other fields `None`. `bg` is `"shell"`, `"tool"`, `None` or `"unknown"`: shell-only background work (agent-deck 1.16.29's misreport) is `"shell"`, and the steward's instructions treat it as unknown (spec 6.5).

- [ ] **Step 1: Write the failing tests**, section `deck`:

```python
def test_roster_parses_and_refuses(d):
    members = load_roster(write(d, "TOPIC.md", TOPIC_WITH_TABLE))   # pipeline/andoria-08/claude, omlx/hera/pi, glm-5.2/andoria-08/claude
    ok([m.name for m in members] == ["pipeline", "omlx", "glm-5.2"], "roster: names in table order")
    ok(raises(RosterError, load_roster, write(d, "T2.md", TOPIC_WITH_TABLE.replace("glm-5.2", "Glm_5.2"))).name == "Glm_5.2", "roster: refuses a bad name")

def test_list_shapes(d):
    rows = deck_rows(fake_run({"list": '[{"title":"omlx","status":"running","substate":"busy","background_work":{"task":"sleep"}}]',
                               "remote andoria-08": '{"sessions":[{"title":"pipeline","status":"waiting"}]}'}), members)
    ok(rows["omlx"]["status"] == "running" and rows["omlx"]["bg"] == "shell", "deck: local row read, shell background flagged")
    ok(rows["pipeline"]["status"] == "waiting" and rows["pipeline"]["substate"] is None and rows["pipeline"]["host_ok"], "deck: object-with-sessions shape and missing fields tolerated")
    ok(rows["glm-5.2"]["status"] is None and rows["glm-5.2"]["host_ok"], "deck: absent from a successful listing is None, not an error")

def test_failed_host_marks_unknown(d):
    rows = deck_rows(fake_run({"list": "[]", "remote andoria-08": ("", 1)}), members)
    ok(rows["pipeline"]["host_ok"] is False and rows["glm-5.2"]["host_ok"] is False and rows["pipeline"]["status"] is None, "deck: a failed host marks all its members unknown")
    ok(rows["omlx"]["host_ok"] is True, "deck: hera unaffected")
```

Before writing `deck_rows`, capture real shapes read-only: `agent-deck list --json | python3 -c 'import json,sys; r=json.load(sys.stdin); r=r if isinstance(r,list) else r["sessions"]; print(sorted(r[0]))'` and `agent-deck remote sessions andoria-08 --json --with-errors | head -c 600`. Adapt the field names in `Row` to what they print (the operations report saw `status`, `substate`, `last_activity_at`, `background_work.task`and put a trimmed real row into the fixture.

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test deck`
Expected: FAIL at `roster: names in table order` (NameError for `load_roster`).

- [ ] **Step 3: Implement `load_roster`, `RosterError`, `deck_rows` and `fake_run`** (a test helper that maps a joined argv prefix to stdout or `(stdout, code)`), in `steward/steward-signals`.

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test deck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/steward-signals
git commit -m "Read a topic's roster and its members' agent-deck rows, marking failed hosts unknown"
```

### Task 3: The census

**Files:**
- Modify: `steward/steward-signals`

**Interfaces:**
- Consumes: `Member` (Task 2).
- Produces: `MSG_RE` (the id grammar); `list_entries(mailbox: str, channel: str) -> list[Entry]` with `Entry = (dir: "top" | "channel", name: str, to: str, sender: str, stamp: str, ctime: float)` from `os.scandir` of the top level and `+<channel>/`, skipping hidden names and anything that fails `MSG_RE`, never opening a file; `read_sent_log(path) -> (intents: list[Intent], sent_ids: set[str])` for the two-phase format `intent <seq> <UTC> <to> <subject> <body-sha>` / `sent <seq> <id>`; `read_notes_log(path) -> set[str]` of direct ids that NOTEs referenced (lines `<UTC> <id>`); `census(entries, members, steward, sent, notes, nudge_ids, now) -> Census` with fields: `unread: dict[name, (count, oldest_age_s)]` for direct mail to each member; `nudges_unread: list[(id, age_s)]` for the steward's own NUDGE ids still at the top level; `unmatched_own: list[id]` for files whose sender is the steward and whose id is in neither `sent_ids` nor matched by an open intent (one with no `sent` line) with the same `to` and a stamp within 10 s; `transit_without_note: int` for top-level member-to-member files whose id is not in `notes`; `posts_since(ts) -> int`.

- [ ] **Step 1: Write the failing tests**, section `census`:

```python
def test_names_are_exact(d):
    box = mailbox(d, top=["pipeline_<s1>_from-glm-5.2_aaaaaaaa.txt", "pi_<s2>_from-nix_bbbbbbbb.txt", ".pipeline-claimed-1-x-y.txt", "pipeline.md"], channel=("ingest", ["ingest_<s3>_from-glm-5.3-flash_cccccccc.txt"]))
    e = list_entries(box, "ingest")
    ok([x.name for x in e if x.dir == "top"] == ["pipeline_<s1>_from-glm-5.2_aaaaaaaa.txt", "pi_<s2>_from-nix_bbbbbbbb.txt"], "census: hidden and malformed names skipped")
    c = census(e, members_named("pipeline", "glm-5.2", "glm-5.3-flash"), "conductor-ingest-steward", ([], set()), set(), set(), now)
    ok(c.unread["pipeline"][0] == 1 and "pi" not in c.unread, "census: pi's mail is not pipeline's")

def test_sent_log_two_phase(d):
    intents, ids = read_sent_log(write(d, "SENT.log", "intent 7 20261102T204044Z kimi-linear DECISION_ingest-3 ab12\nsent 7 kimi-linear_20261102T204047.551200Z_from-conductor-ingest-steward_d17f5c20\nintent 8 20261102T204100Z lfm3 NUDGE_glm53-fp8 cd34\n"))
    ok(len(intents) == 2 and ids == {"kimi-linear_20261102T204047.551200Z_from-conductor-ingest-steward_d17f5c20"}, "census: two-phase log parsed")

def test_unmatched_own_files(d):
    e = entries_named("lfm3_20261102T204103.000000Z_from-conductor-ingest-steward_e1e1e1e1.txt",   # matches intent 8 (lfm3, within 10 s)
                      "muse-spark_20261102T210000.000000Z_from-conductor-ingest-steward_f2f2f2f2.txt")  # no intent, no sent line
    c = census(e, members, "conductor-ingest-steward", read_sent_log(SENT), set(), set(), now)
    ok(c.unmatched_own == ["muse-spark_20261102T210000.000000Z_from-conductor-ingest-steward_f2f2f2f2"], "census: an intent within 10 s matches; the rest is disputed")

def test_nudges_and_notes(d):
    c = census(entries_named("lfm3_<s>_from-conductor-ingest-steward_11111111.txt", "kimi-linear_<s>_from-glm-5.2_22222222.txt", "lfm3_<s>_from-glm-5.2_33333333.txt"), members, steward, sent, notes={"kimi-linear_<s>_from-glm-5.2_22222222"}, nudge_ids={"lfm3_<s>_from-conductor-ingest-steward_11111111"}, now=now)
    ok(len(c.nudges_unread) == 1 and c.transit_without_note == 1, "census: unread NUDGE found; one member mail in transit lacks a NOTE")

def test_census_scales(d):
    box = mailbox(d, top=[f"glm-5.2_{stamp(i)}_from-lfm3_{i:08x}.txt" for i in range(500)], channel=("ingest", [f"ingest_{stamp(i)}_from-lfm3_{i:08x}.txt" for i in range(5000)]))
    t = time.monotonic(); e = list_entries(box, "ingest"); ok(time.monotonic() - t < 1.0 and len(e) == 5500, "census: 5,500 names listed under a second without opening a file")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test census`
Expected: FAIL (NameError `list_entries`).

- [ ] **Step 3: Implement** `MSG_RE`, `list_entries`, `read_sent_log`, `read_notes_log`, `census` and the test helpers `mailbox`, `entries_named`, `members_named`, `stamp`. Match a sender or recipient by exact string equality on the parsed name, never by prefix. Age in seconds is `now - stamp` parsed from the file name, falling back to `ctime`.

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test census`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/steward-signals
git commit -m "Take a names-only census of a topic: unread mail, unread nudges, unmatched steward files, mail without a NOTE"
```

### Task 4: The waiting-on-John probe

**Files:**
- Modify: `steward/steward-signals`

**Interfaces:**
- Consumes: `deck_rows` (Task 2), `run`.
- Produces: `probe(run, name: str, host: str | None) -> bool | None`: for a hera member, runs `agent-deck session output <name> --max-tokens 200` (text form, never `-q`), returns `True` when the last non-empty line ends with `?`, `False` otherwise; returns `None` (unknown) for a remote member or when the command fails. The `probe` subcommand prints `yes`, `no` or `unknown` and nothing else, so no peer text ever reaches the steward's context.

- [ ] **Step 1: Write the failing tests**, section `probe`:

```python
def test_probe_local(d):
    ok(probe(fake_run({"session output lfm3": "⏺ Done.\n\nShould I push to main?\n"}), "lfm3", "hera") is True, "probe: trailing question line is yes")
    ok(probe(fake_run({"session output lfm3": "⏺ Done. Waiting for mail.\n"}), "lfm3", "hera") is False, "probe: no question is no")
    ok(probe(fake_run({"session output lfm3": ("", 1)}), "lfm3", "hera") is None, "probe: a failed command is unknown")

def test_probe_remote_is_unknown(d):
    ok(probe(fake_run({}), "pipeline", "andoria-08") is None, "probe: remote members are unknown (no remote output path verified)")

def test_probe_prints_one_word(d):
    out, code, _ = capture(main, ["probe", "lfm3", "--run-fixture", fixture_path])
    ok(out.strip() in ("yes", "no", "unknown") and "\n" not in out.strip(), "probe: prints one word only")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test probe`
Expected: FAIL (NameError `probe`).

- [ ] **Step 3: Implement `probe`** and the `probe <name>` subcommand (host looked up from `TOPIC.md` via `--dir`), with a hidden `--run-fixture FILE` option used only by the self-test to inject `fake_run`.

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test probe`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/steward-signals
git commit -m "Probe one local member for a trailing question without printing its text"
```

### Task 5: The metrics file

**Files:**
- Modify: `steward/steward-signals`

**Interfaces:**
- Consumes: the harness (Task 1).
- Produces: `write_metrics(conductor_dir: str, day: str, data: dict) -> str`, merging `data` into `<conductor_dir>/metrics/<day>.json` and returning its path: for integers keep the larger, for dicts merge recursively, for other types take the new value; a file that is not valid JSON is moved aside to `<day>.json.bad` and a fresh file written, with one notice on stderr. The `metrics --dir DIR [--date YYYY-MM-DD] --set key=value ...` subcommand merges the steward's counters: a value is an int when it parses as one and a string otherwise, and a dotted key (`interventions.nudge=1`) addresses a nested dict. The counters the steward records are the spec's section 6.10 without cost: notice delay (p50 and max, seconds), posts and direct mails seen, NOTEs, member mail seen in transit without a NOTE, interventions by type and by the worker's response, questions opened, answered, dropped and defaulted, pages, `rate` records, relays sent, disputed and failed, forged files found, STATUS staleness episodes, doorbell and tick restarts, context resets, nudges, offers, defaults applied alone, nag reports. No token, dollar or usage-window figure is collected (spec decision 20).

- [ ] **Step 1: Write the failing tests**, section `metrics`:

```python
def test_write_metrics_merges(d):
    write_metrics(d, "2026-11-03", {"turns": 3, "interventions": {"nudge": 1}})
    path = write_metrics(d, "2026-11-03", {"turns": 4, "interventions": {"overlap": 1}, "last_post": "20261103T0700Z"})
    m = json.load(open(path))
    ok(m["turns"] == 4 and m["interventions"] == {"nudge": 1, "overlap": 1} and m["last_post"] == "20261103T0700Z", "metrics: merge keeps the larger counter, merges dicts, takes new strings")

def test_metrics_subcommand(d):
    out, code, _ = capture(main, ["metrics", "--dir", d, "--date", "2026-11-03", "--set", "turns=5", "--set", "interventions.nudge=2", "--set", "last_post=20261103T0700Z"])
    m = json.load(open(os.path.join(d, "metrics", "2026-11-03.json")))
    ok(code == 0 and m["turns"] == 5 and m["interventions"]["nudge"] == 2 and m["last_post"] == "20261103T0700Z", "metrics: --set writes ints, nested keys and strings")
    ok("?" not in out, "metrics: no question mark")

def test_metrics_corrupt_file(d):
    os.makedirs(os.path.join(d, "metrics"), exist_ok=True); write(d, "metrics/2026-11-03.json", "{not json")
    _, _, err = capture(write_metrics, d, "2026-11-03", {"turns": 1})
    path = os.path.join(d, "metrics", "2026-11-03.json")
    ok(os.path.exists(path + ".bad") and json.load(open(path))["turns"] == 1 and "moved aside" in err, "metrics: a corrupt file is moved aside and a fresh one written, with a notice")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test metrics`
Expected: FAIL (NameError `write_metrics`).

- [ ] **Step 3: Implement `write_metrics` and the `metrics` subcommand** in `steward/steward-signals`. The merge rule: for integers keep the larger, for dicts merge recursively, for other types take the new value. The file is written atomically (temporary file in `metrics/`, then `os.replace`).

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test metrics`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/steward-signals
git commit -m "Keep a steward's daily metrics file, merging counters without losing any"
```

### Task 6: The signals command

**Files:**
- Modify: `steward/steward-signals`

**Interfaces:**
- Consumes: everything from Tasks 1 to 5.
- Produces: the `signals <channel> --dir CONDUCTOR_DIR [--mailbox DIR] [--json] [--probe NAME]` subcommand. It loads `TOPIC.md`, `POLICY.md`, `SENT.log`, `NOTES.log` from the conductor directory, lists the mailbox, calls agent-deck once per host, and prints one table with a row per member: `name host harness status substate bg last-activity unread(oldest) checked-age`; then `exceptions:` lines for each `unmatched_own` id and each `nudges_unread` entry; then `counts: posts-since-last=<n> transit-without-note=<n>`; and, when `--probe NAME` is given, `probe <NAME>: yes|no|unknown`. `--json` prints the same as one JSON object. Member records' `checked` age comes from the mtime of `.agent-mail/members/<name>.json` (metadata only). The last call's time is kept in `<dir>/steward-state.json` under `signals_last` so `posts-since-last` is right; the command writes nothing else. Output never contains `?`.

- [ ] **Step 1: Write the failing tests**, section `signals`:

```python
def test_signals_end_to_end(d):
    cdir = conductor_dir(d, topic=TOPIC_WITH_TABLE, policy=SPEC_POLICY, sent=SENT, notes=NOTES)
    box = mailbox(d, top=["pipeline_<s1>_from-glm-5.2_aaaaaaaa.txt", "lfm3_<s2>_from-conductor-ingest-steward_11111111.txt"], channel=("ingest", ["ingest_<s3>_from-pipeline_cccccccc.txt"]), members_checked={"pipeline": now - 7200})
    out, code, _ = capture(main, ["signals", "ingest", "--dir", cdir, "--mailbox", box, "--run-fixture", DECK_FIXTURE])
    ok(code == 0 and re.search(r"^pipeline +andoria-08 +claude +waiting ", out, re.M), "signals: a member row")
    ok("exceptions:" in out and "lfm3_<s2>_from-conductor-ingest-steward_11111111" in out, "signals: the unmatched steward file is listed")
    ok("transit-without-note=1" in out and "posts-since-last=1" in out, "signals: counts")
    ok("?" not in out, "signals: no question mark")
    out2, _, _ = capture(main, ["signals", "ingest", "--dir", cdir, "--mailbox", box, "--run-fixture", DECK_FIXTURE, "--json"])
    j = json.loads(out2); ok(j["members"]["pipeline"]["unread"]["count"] == 1 and j["counts"]["posts_since_last"] == 0, "signals: JSON form, and posts-since-last advanced")

def test_signals_writes_only_state(d):
    before = tree(d); capture(main, ["signals", "ingest", "--dir", cdir, "--mailbox", box, "--run-fixture", DECK_FIXTURE])
    ok(tree(d) - before <= {os.path.join(cdir, "steward-state.json")}, "signals: writes nothing but steward-state.json")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test signals`
Expected: FAIL (unknown subcommand).

- [ ] **Step 3: Implement the `signals` subcommand** and the renderers `render_text(report) -> str`, `render_json(report) -> str`.

- [ ] **Step 4: Run the whole self-test**

Run: `python3 -I steward/steward-signals self-test`
Expected: all sections pass; `steward-signals self-test: N checks passed`.

- [ ] **Step 5: Commit**

```bash
git add steward/steward-signals
git commit -m "Print a topic's signals in one call: member rows, exceptions and counts, text or JSON"
```

### Task 7: The steward's instructions (`CLAUDE.md`) and the docs check

**Files:**
- Create: `steward/CLAUDE.md`
- Modify: `steward/steward-signals` (self-test section `docs`)

**Interfaces:**
- Consumes: spec 6.4 to 6.9 and 7.1; the draft `operations.md` section 4.
- Produces: `steward/CLAUDE.md` with these headings, in order: `# Topic steward`; `## 1. Your standing`; `## 2. Precedence` (over the shared conductor instructions, naming the parts that do not apply: auto-respond, /send, session control, the heartbeat NEED: format, state.json and task-log.md); `## 3. Starting a topic`; `## 4. Every turn`; `## 5. Each delivered item`; `## 6. Exception checks`; `## 7. Waking`; `## 8. Time checks`; `## 9. Questions to John`; `## 10. Relaying John's answers`; `## 11. The ledger and the logs`; `## 12. Digest, summary and STATUS`; `## 13. Restarts and resets`; `## 14. Ending a topic`. It quotes the exact-copy items from Global Constraints and every message template from spec 6.6, and cites `POLICY.md` keys by name instead of numbers.
- Produces: `docs_checks(repo_dir) -> list[str]` in the helper, run by self-test section `docs`, returning the failures below.

- [ ] **Step 1: Write the failing tests**, section `docs`:

```python
def test_claude_md(d):
    t = read(os.path.join(REPO, "steward", "CLAUDE.md"))
    for h in CLAUDE_HEADINGS: ok(h in t, f"docs: CLAUDE.md has '{h}'")
    ok("Q <prefix>-<n>: <text>: A <x>, B <y>?" in t and "80" in t, "docs: question template and limit")
    ok(all(s in t for s in ("unclaimed", "active", "blocked", "waiting-john", "done", "dropped", "orphaned")) and "| open |" not in t, "docs: ledger states, no 'open' state")
    for subj in ("OVERLAP", "DUPLICATE", "ALREADY-DONE", "NUDGE", "OFFER", "ROUTE", "NEEDS-JOHN", "DECISION", "DEFAULT", "DISPUTE", "STEWARD back"): ok(subj in t, f"docs: template {subj}")
    ok("NEED:" not in t.replace("NEED(steward)", ""), "docs: no NEED: lines (the scanner pages Q lines)")
    ok(re.search(r"never[^.\n]*agent-deck session send", t, re.I) and not re.search(r"^(?!.*never).*agent-deck session send", t, re.M | re.I), "docs: session send appears only in a never-sentence")
    ok("steward-tick <channel>" in t and "echo steward-tick" in t, "docs: the tick is named by channel")
    ok("intent " in t and "sent " in t and "RECEIVED.log" in t and "NOTES.log" in t, "docs: two-phase SENT.log, RECEIVED.log and NOTES.log")
    ok("DECISION" in t.split("## 10.")[1].split("## 11.")[0] and "post first" in t.split("## 10.")[1].split("## 11.")[0].lower(), "docs: DECISION post before the relay mail")
    ok("steward-signals signals" in t and "steward-signals metrics" in t, "docs: the helper is called by name")
    for num in ("45 min", "20 min", "2 h"): ok(num not in t, f"docs: threshold {num} is cited by POLICY key, not value")
    ok("?" not in strip_code_and_template(t), "docs: no stray question marks outside code blocks and the template")
```

- [ ] **Step 2: Run to verify it fails**

Run: `python3 -I steward/steward-signals self-test docs`
Expected: FAIL (file missing).

- [ ] **Step 3: Write `steward/CLAUDE.md`** from the draft, applying every difference in spec 7.1 and the sections' contents from spec 6.4 to 6.9 (turn steps 1 to 7; the three wake paths with the doorbell delay and the named tick scheduled to the next 07:30; the exception checks and time classes as tables; the question rules; relaying with the DECISION-then-mail order and the labelled `by:` line; the ledger format; `DIGEST.md` on ticks, the 07:30 one-paragraph `SUMMARY`, `STATUS` every turn; `steward-state.json`, `SENT.log`, `RECEIVED.log`, `NOTES.log`, `metrics/`; reset handling; ending without an agent-deck completion line). Implement `docs_checks` and `strip_code_and_template`.

- [ ] **Step 4: Run to verify it passes**

Run: `python3 -I steward/steward-signals self-test docs`
Expected: PASS for the CLAUDE.md checks.

- [ ] **Step 5: Commit**

```bash
git add steward/CLAUDE.md steward/steward-signals
git commit -m "Write the steward's instructions and check them in the self-test"
```

### Task 8: `SHARED.md` and `HEARTBEAT_RULES.md`

**Files:**
- Create: `steward/SHARED.md`, `steward/HEARTBEAT_RULES.md`
- Modify: `steward/steward-signals` (section `docs`)

**Interfaces:**
- Produces: `SHARED.md`, the shared conductor instructions for Claude conductors on hera, saying only: a conductor's own `CLAUDE.md` governs; this file grants nothing; it never auto-responds, forwards, restarts or launches; it keeps no `state.json` or `task-log.md` of agent-deck's shape. `HEARTBEAT_RULES.md`: a heartbeat is a turn like any other; run the turn in `CLAUDE.md` section 4; print the status line and nothing else unless a check fires; never answer in a `[STATUS]`/`NEED:` form.

- [ ] **Step 1: Write the failing tests** (append to section `docs`):

```python
def test_shared_and_heartbeat(d):
    s = read(os.path.join(REPO, "steward", "SHARED.md")); h = read(os.path.join(REPO, "steward", "HEARTBEAT_RULES.md"))
    ok("own CLAUDE.md governs" in s and "auto-respond" in s and "never" in s.lower(), "docs: SHARED.md defers and forbids")
    ok("/send" in s and "session send" in s and re.search(r"never[^.\n]*session send", s, re.I), "docs: SHARED.md names the forwarding commands it forbids")
    ok("a turn like any other" in h and "NEED:" in h and "never" in h.lower(), "docs: heartbeat rules")
    ok(len(s) < 1500 and len(h) < 800, "docs: both files short")
```

- [ ] **Step 2: Run to verify it fails** — `python3 -I steward/steward-signals self-test docs` → FAIL (files missing).

- [ ] **Step 3: Write both files.**

- [ ] **Step 4: Run to verify it passes** — same command → PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/SHARED.md steward/HEARTBEAT_RULES.md steward/steward-signals
git commit -m "Add steward-safe shared conductor instructions and heartbeat rules"
```

### Task 9: The worker's section (`WORKER.md`)

**Files:**
- Create: `steward/WORKER.md`
- Modify: `steward/steward-signals` (section `docs`)

**Interfaces:**
- Produces: `WORKER.md` = spec 6.2's rules 1 to 8 and the post table, the `steward-questions.md` format (one line per open protected question: `<id> <slug> <options>`; remove the line when the question closes), the relay checks of spec 6.7 numbered 1 to 4 with the re-receive before disputing and the recovery after a disputed relay, the override sentence of spec 6.2 rule 6, `holds:`, `until:`, `needs:`, `mailed:`, `protected: yes`, `slug:` in every body, `DISPUTE`, and no silence rule.

- [ ] **Step 1: Write the failing tests** (append to section `docs`):

```python
def test_worker_md(d):
    w = read(os.path.join(REPO, "steward", "WORKER.md"))
    for verb in ("JOIN", "CLAIM", "UPDATE", "BLOCKED", "DONE", "DROP", "NOTE", "QUESTION", "DECISION", "DISPUTE", "LEAVE"): ok(f"`{verb}" in w, f"docs: WORKER.md has {verb}")
    ok("slug:" in w and "protected: yes" in w and "steward-questions.md" in w and "mailed:" in w, "docs: worker fields")
    checks = w.split("protected action")[1] if "protected action" in w else ""
    ok(all(f"\n{i}." in checks for i in (1, 2, 3, 4)) and "within 10 minutes" in checks and "receive" in checks and "DISPUTE" in checks, "docs: the four checks, the window, the re-receive and the dispute")
    ok("overrides" in w and "banner" in w and "doorbell" in w, "docs: the override sentence for a checked relay")
    ok("silent" not in w.lower() or "nothing waits for the steward" in w.lower(), "docs: no silence rule; nothing waits for the steward")
    ok("every 2 hours" in w and "every turn" not in w, "docs: UPDATE on change or every 2 hours, never every turn")
```

- [ ] **Step 2: Run to verify it fails** → FAIL (file missing).

- [ ] **Step 3: Write `steward/WORKER.md`** from `operations.md` section 5 with spec 7.2's changes.

- [ ] **Step 4: Run to verify it passes** → PASS.

- [ ] **Step 5: Commit**

```bash
git add steward/WORKER.md steward/steward-signals
git commit -m "Write the stewarded-topic section for workers, with the relay checks"
```

### Task 10: The scanner patch (STATUS rule and MCP tool-call bullets)

**Files:**
- Create: `steward/patches/monitor_sessions-steward.patch`, `steward/patches/test_steward_patches.py`, `steward/patches/README.md`

**Interfaces:**
- Consumes: John's `~/.local/share/agent-deck/conductor/director/monitor_sessions.py` (read-only; copy it to a scratch directory and patch the copy), its tests `test_monitor_sessions.py` and the venv `~/.local/share/agent-deck/conductor/venv/bin/python`.
- Produces: in the patch, `steward_status_alerts(now: float, conductor_root: str) -> list[dict]`: for each `<conductor_root>/*-steward/STATUS`, parse the line (Global Constraints), and when `now - mtime > 2 * tick * 60` and `open > 0`, or when the channel named in `<conductor dir>/TOPIC.md` holds a post stamped after `last-post=` while the file is stale, return an alert whose `detail` is exactly `<steward> STATUS unchanged since <the file's UTC> (tick <m>m, <n> open)` and whose `where` is `steward`, fed to the existing `deliver_attention` so it is paged once per text; and the tool-call test extended so a bullet of the form `⏺ <server> - <tool> (MCP)(` counts as a tool call. `README.md` in `patches/` gives the apply and test commands.

- [ ] **Step 1: Write the failing tests** in `steward/patches/test_steward_patches.py` (unittest; it imports the patched copy by path given in `STEWARD_PATCHED_MONITOR`):

```python
class StatusRule(unittest.TestCase):
    def test_stale_with_open_items_pages_once_per_episode(self):
        write_status(self.dir, "20261102T171500Z tick=15 open=4 active=3 waiting-john=1 last-post=20261102T171000Z doorbell=ok", mtime=now - 40*60)
        a = m.steward_status_alerts(now, self.root)
        self.assertEqual(a[0]["detail"], "conductor-ingest-steward STATUS unchanged since 20261102T171500Z (tick 15m, 4 open)")
        self.assertEqual(m.steward_status_alerts(now + 600, self.root)[0]["detail"], a[0]["detail"])   # same text within the episode
    def test_fresh_or_nothing_open_is_quiet(self):
        write_status(self.dir, "… tick=15 open=0 …", mtime=now - 3*3600); self.assertEqual(m.steward_status_alerts(now, self.root), [])
        write_status(self.dir, "… tick=15 open=2 …", mtime=now - 20*60); self.assertEqual(m.steward_status_alerts(now, self.root), [])
    def test_new_posts_while_stale_pages(self):
        write_status(self.dir, "… tick=60 open=0 last-post=20261102T171000Z …", mtime=now - 3*3600)
        touch_post(self.mailbox, "ingest", "ingest_20261102T190000.000000Z_from-lfm3_abcdef01.txt")
        self.assertEqual(len(m.steward_status_alerts(now, self.root, mailbox=self.mailbox)), 1)
class McpBullet(unittest.TestCase):
    def test_mcp_tool_call_is_a_tool_call(self):
        self.assertTrue(m.is_tool_call_bullet("⏺ agent-mail - receive (MCP)(me: \"conductor-ingest-steward\")"))
        self.assertTrue(m.is_tool_call_bullet("⏺ Bash(pgrep -f steward-tick)"))
        self.assertFalse(m.is_tool_call_bullet("⏺ Q ingest-3: HF org for glm-5.3-fp8: A positron-internal, B positron-ai?"))
    def test_peer_question_inside_mcp_block_is_not_paged(self):
        pane = MCP_RECEIVE_BLOCK_ENDING_IN_PEER_QUESTION
        self.assertIsNone(m.visible_prompt_or_question(pane))
```

Read `monitor_sessions.py` first for the names of the real tool-call test and of `visible_prompt_or_question`'s signature, and use those names; `is_tool_call_bullet` above stands for whatever the file calls it.

- [ ] **Step 2: Run to verify it fails**

Run: `cp ~/.local/share/agent-deck/conductor/director/monitor_sessions.py /tmp/steward-patch/ && STEWARD_PATCHED_MONITOR=/tmp/steward-patch/monitor_sessions.py ~/.local/share/agent-deck/conductor/venv/bin/python -m unittest steward/patches/test_steward_patches.py`
Expected: FAIL (`steward_status_alerts` missing; the MCP bullet not recognised).

- [ ] **Step 3: Edit the copy** to add `steward_status_alerts`, call it from the pass that gathers attention items (alongside the question and stall items), extend the tool-call bullet test, and write the diff: `diff -u <original> <copy> > steward/patches/monitor_sessions-steward.patch`. Never edit the original.

- [ ] **Step 4: Run to verify it passes**, and that John's own tests still pass on the copy

Run: the Step 2 command → PASS; then `cd /tmp/steward-patch && cp ~/.local/share/agent-deck/conductor/director/test_monitor_sessions.py . && ~/.local/share/agent-deck/conductor/venv/bin/python -m unittest test_monitor_sessions` → PASS.

- [ ] **Step 5: Write `steward/patches/README.md`** (apply with `patch -p0 -d ~/.local/share/agent-deck/conductor/director < …`, run the unit tests in the venv, confirm the next scanner pass logs normally in `heartbeat.log`) and commit

```bash
git add steward/patches/monitor_sessions-steward.patch steward/patches/test_steward_patches.py steward/patches/README.md
git commit -m "Patch the director's scanner to page a stale steward STATUS once and to see MCP tool calls"
```

### Task 11: The bridge patch (case-insensitive prefix, reply routing)

**Files:**
- Create: `steward/patches/bridge-steward-routing.patch`
- Modify: `steward/patches/test_steward_patches.py`, `steward/patches/README.md`

**Interfaces:**
- Consumes: John's `~/.local/share/agent-deck/conductor/bridge.py` (read-only; patch a copy), `parse_conductor_prefix(text, names)`, the Discord `on_message` handler and the text the scanner posts, `NEED(local/<session title>): …`.
- Produces: `parse_conductor_prefix` matching the prefix case-insensitively and tolerating a space before the colon; `conductor_for_reply(replied_text: str, names: list[str]) -> str | None` returning `<name>` when `replied_text` starts with `NEED(local/conductor-<name>)` for a known conductor; `on_message` routing a reply to such a page to that conductor before falling back to the default.

- [ ] **Step 1: Write the failing tests** (append to `test_steward_patches.py`, importing the patched copy from `STEWARD_PATCHED_BRIDGE`):

```python
class BridgeRouting(unittest.TestCase):
    names = ["director", "ingest-steward", "nix-steward"]
    def test_prefix_case_insensitive(self):
        self.assertEqual(b.parse_conductor_prefix("Ingest-steward: ingest-3 A", self.names), ("ingest-steward", "ingest-3 A"))
        self.assertEqual(b.parse_conductor_prefix("nix-steward : nix-4 B", self.names), ("nix-steward", "nix-4 B"))
        self.assertEqual(b.parse_conductor_prefix("ingest-3 A", self.names), (None, "ingest-3 A"))
    def test_reply_to_page_routes_to_that_steward(self):
        self.assertEqual(b.conductor_for_reply("@johnw NEED(local/conductor-ingest-steward): Q ingest-3: HF org …?", self.names), "ingest-steward")
        self.assertIsNone(b.conductor_for_reply("NEED(local/nix): some question?", self.names))
```

- [ ] **Step 2: Run to verify it fails** — the Step 2 command of Task 10 with `STEWARD_PATCHED_BRIDGE=/tmp/steward-patch/bridge.py` added → FAIL.

- [ ] **Step 3: Edit the copy** (`parse_conductor_prefix`; add `conductor_for_reply`; in `on_message`, when the Discord message is a reply to a bot message, call `conductor_for_reply` on the replied message's content before `parse_conductor_prefix`), write the diff to `steward/patches/bridge-steward-routing.patch`, and extend `patches/README.md` (apply, then `launchctl kickstart -k gui/$(id -u)/com.agentdeck.conductor-bridge`, then send `ingest-steward: digest` from Discord and expect the digest back).

- [ ] **Step 4: Run to verify it passes** → PASS, and the bridge's own tests if any exist beside it.

- [ ] **Step 5: Commit**

```bash
git add steward/patches/bridge-steward-routing.patch steward/patches/test_steward_patches.py steward/patches/README.md
git commit -m "Patch the Discord bridge to route case-insensitive prefixes and replies to a steward's page"
```

### Task 12: `steward/README.md`, the day-zero runbook, and the top-level pointer

**Files:**
- Create: `steward/README.md`
- Modify: `README.md` (one paragraph after section 3's etiquette bullet)
- Modify: `steward/steward-signals` (section `docs`)

**Interfaces:**
- Produces: `steward/README.md` with: what each file is; the preconditions from spec 10 as a checklist; the three channels as they exist since 2026-10-31 14:59 PDT with their purposes verbatim (created by mail-dev under John's decision 36 in that session; a purpose cannot be edited by a command); John's commands verbatim (the two `agent-deck conductor setup` lines from spec 6.4 with `-shared-instructions-md`, `-instructions-md`, `-policy-md`, `-heartbeat-rules-md` and `-env AGENTDECK_CONTEXT_WINDOW=`; the post-setup `cp --remove-destination` that turns the conductor's `POLICY.md` link into a copy; the one line typed in each member's pane naming the topic, the steward and that host's `WORKER.md` path); the Discord answer form and its two traps; and the day-zero runbook: the Phase 0 checks of spec 10 as a numbered table `check | how | expected`, run on `+steward-test` with `POLICY-test.md` and planted posts from John's shell (`agent-mail send --from steward-test-a --to +steward-test --subject "CLAIM t1: test" …`), each row's expected output exact, and the rollback (`agent-deck session stop <steward>`).
- Produces: in the top-level `README.md`, one paragraph under section 3: "Some topics have a steward …" pointing at `steward/WORKER.md`, with no other change to the README's rules.

- [ ] **Step 1: Write the failing tests** (append to section `docs`):

```python
def test_steward_readme_and_pointer(d):
    r = read(os.path.join(REPO, "steward", "README.md"))
    ok(all(c in r for c in ("+nix-config", "+ingest", "+steward-test")) and "channel create" not in r.split("day-zero")[0].lower().replace("cannot be edited", "") and r.count("agent-deck conductor setup") == 2, "docs: the three existing channels and the two setup commands")
    ok("-shared-instructions-md" in r and "-env AGENTDECK_CONTEXT_WINDOW=" in r and "cp --remove-destination" in r, "docs: setup flags and the POLICY copy step")
    ok("POLICY-test.md" in r and "steward-test-a" in r and "| check | how | expected |" in r, "docs: the day-zero runbook table")
    for check in ("doorbell", "steward-tick", "07:30", "session context", "DISPUTE", "session stop", "bridge.py", "Q&A"): ok(check in r, f"docs: day-zero check {check}")
    top = read(os.path.join(REPO, "README.md"))
    ok("steward/WORKER.md" in top and top.count("steward") >= 2, "docs: top-level README points at the worker section")
    ok("cannot authorize" in top, "docs: the README's rules are unchanged")
```

- [ ] **Step 2: Run to verify it fails** → FAIL (file missing).

- [ ] **Step 3: Write `steward/README.md`** and add the paragraph to `README.md`.

- [ ] **Step 4: Run the whole self-test and agent-mail's**

Run: `python3 -I steward/steward-signals self-test && python3 -I ./agent-mail self-test`
Expected: both pass (agent-mail's 401 checks are untouched by this plan).

- [ ] **Step 5: Commit**

```bash
git add steward/README.md README.md steward/steward-signals
git commit -m "Document the steward files, John's setup commands and the day-zero runbook"
```

---

## Not in this plan

- **The fork fix (spec decision 13).** In John's agent-deck fork: `conductor setup` must not overwrite an existing `conductor/bridge.py` and must not `launchctl unload` or rewrite plists it did not create (the nix-managed `com.agentdeck.conductor-bridge` and `com.agentdeck.transition-notifier`); the Discord Q&A hook (`route_question`, `discord_qna.py`) folded into the embedded bridge. Acceptance: running `conductor setup steward-probe -agent claude -no-heartbeat` on a copy of the conductor directory leaves `bridge.py` byte-identical and both agents loaded; then `conductor teardown steward-probe --remove`. Then the nix-config bump and hera's activation (pi). A separate plan in the fork's repository if John asks.
- agent-deck 1.16.30 or later on andoria-08 and delphi-3bd4 (John's manual step; the director's policy forbids `remote update`).
- The channels' creation and the two `conductor setup` runs: John runs the commands in `steward/README.md` when the preconditions hold.
- The v2 requests in spec section 8 (mailed to `mail-dev` on 2026-10-31) and the v2.0.x text change after the pilot.
- nix packaging of `steward/` (after the pilot, like v2's Task 22).
- The filer (spec section 9), only if the pilot's metrics call for it.

## Self-review notes

Spec coverage: 6.2 (Task 9), 6.4 (Tasks 7, 12), 6.5 (Tasks 2 to 6, 7), 6.6 and 6.7 (Tasks 7, 9), 6.8 (Task 7), 6.9 (Tasks 7, 10), 6.10 (Tasks 5, 6, 7), 7.1 (Tasks 7, 8), 7.2 (Task 9), 8 "of John's own files" patches (Tasks 10, 11), 10 Phase 0 (Task 12). The Discord traps (spec 6.7) are both in Task 11's patch and in Task 12's runbook. Names used across tasks: `load_policy`, `PolicyError`, `load_roster`, `RosterError`, `Member`, `deck_rows`, `Row`, `fake_run`, `MSG_RE`, `list_entries`, `Entry`, `read_sent_log`, `read_notes_log`, `census`, `Census`, `probe`, `write_metrics`, `docs_checks`, `strip_code_and_template`, `steward_status_alerts`, `conductor_for_reply`.
