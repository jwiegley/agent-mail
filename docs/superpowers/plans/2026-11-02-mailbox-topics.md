# Mailbox topics (agent-mail 2.2 and the join grammar) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** agent-mail 2.2 keeps write targets on a session's subscriptions and lets `send` omit `to`, and nix-config's `mailbox` and `agent-mail` skills carry the `/mailbox <name> [+A | ++A | !+A | !++A ...]` grammar.

**Architecture:** Everything in the program is one more attribute, `write`, on the runtime subscription that `effective_subs` already carries, plus a routing step in front of the unchanged single-target `send`: a send without `to` becomes one `send` per write target, or one to the default channel. The MCP tools, the CLI and `channels` expose it; the state schema stays version 2 and both directions of file compatibility hold. The skills are text: the join parses the specs, replaces the session's runtime subscriptions, and reports.

**Tech Stack:** one stdlib-only Python file (`agent-mail`, Python 3.9 or newer; argparse's `BooleanOptionalAction` is available), its own self-test as the test suite; nix-config (Nix, the source catalog pin, Markdown skills) published through nix's queue.

**Spec:** `docs/superpowers/specs/2026-11-02-mailbox-topics-design.md` (approved, decision 73). Decisions 62, 63, 64, 70, 71, 72 in `/Users/johnw/Products/mailbox-rollout/decisions.md`.

## Global Constraints

- The program stays a single stdlib-only Python file; the CLI and the MCP server share one core; the self-test is the suite and every section must pass: `python3 -I ./agent-mail self-test` (Hera's Python 3.14 and 3.9.6, Vulcan's 3.13 before the audit, as for 2.1).
- The reader state schema version stays `2`; a 2.1 state file loads in 2.2 (`write` absent means false) and a 2.2 file loads in 2.1 (the key is ignored).
- No key is written to `~/.config/agent-mail/config.json` by this work; `default_channel` has a default and nix need not set it.
- The mailbox layout, the naming standard, claims, archives, retention, the doorbell, the reader lock, the identity chain, broadcasts, `channel create`, `doctor`, member records and the mail rules do not change (spec 5.6).
- Work happens in a worktree under `~/Products` (`~/Products/agent-mail-v22`, branch `work/v2.2` from main); the live checkout `~/src/sys/agent-mail` is run by peers and takes docs commits by explicit path only.
- Every commit on Hera is signed (`git commit -S`, decision 67) and ends with the two attribution trailers used in this session.
- Ponytail: the smallest change that fully works; no option, wrapper or abstraction the spec does not name.

## Review Focus

- A `send` whose `to` is the empty string: treated as omitted (the targets, or the default channel), never as a session named ""; pinned to Task 2's tests.
- A write target whose channel directory is gone at send time: that target is refused with `no_such_channel`, the others are still written, and the call succeeds when any was; pinned to Task 2's tests.
- A 2.1 state file, and a state whose `write` is not a boolean: both load, `write` false unless the value is exactly true; pinned to Task 1's tests.
- A default channel given as `+fleet` (with the plus) or naming a channel that does not exist: the plus is stripped and validated, and a send to a missing default channel fails with `no_such_channel` naming it; pinned to Task 3's tests.
- The same body sent to two targets, then sent again within ten minutes: the duplicate window is per target, so the second round is refused on both and the first round was not a duplicate across targets; pinned to Task 2's tests.

---

### Task 1: `write` on subscriptions

**Files:**
- Modify: `agent-mail` (`effective_subs` ~986, `subscribe` ~997, `subscription_text` ~3010, `tool_subscribe` ~2696, `cmd_subscribe` ~3016)
- Test: `agent-mail`, `section_channels` (~3875) and `section_tools` (~4848)

**Interfaces:**
- Consumes: nothing new.
- Produces: `effective_subs(state, settings, now) -> {ch: {"wake": bool, "write": bool, "pinned": bool, "since": float}}`; `subscribe(d, name, state, settings, ch, wake=None, backlog=0, write=None, now=None)` where `None` leaves an existing subscription's attribute as it is and means false for a new one; `subscription_text` ends the level with `, write` when set (`+a: pi subscribes, quiet, write; subscribers listed: ...`).

- [ ] **Step 1: Write the failing tests** in `section_channels`, after the `backlog 2` check (use the section's `d`, `s`, `now`, `st`):

```python
subscribe(d, "pi", st, s, "fleet", write=True, now=now)
t.ok(effective_subs(st, s, now)["fleet"]["write"] is True, "subscribe with write marks the channel a write target")
subscribe(d, "pi", st, s, "fleet", wake=True, now=now)
t.ok(effective_subs(st, s, now)["fleet"] == {**effective_subs(st, s, now)["fleet"], "wake": True, "write": True}, "subscribing again changes only the attributes named")
subscribe(d, "pi", st, s, "fleet", write=False, now=now)
t.ok(effective_subs(st, s, now)["fleet"]["write"] is False and effective_subs(st, s, now)["fleet"]["wake"] is True, "write false clears the target and leaves wake alone")
pinned_ops = load_settings({"channels": ["ops:wake"]}, {}, "pi")
create_channel(d, "ops", "ops notices")
subscribe(d, "pi", st, pinned_ops, "ops", write=True, now=now)
t.ok(effective_subs(st, pinned_ops, now)["ops"] == {"wake": True, "write": True, "pinned": True, "since": now - 86400}, "a pinned channel takes write without touching its cursor")
st["subs"]["fleet"] = {"since": now, "wake": False}
t.ok(effective_subs(st, s, now)["fleet"]["write"] is False, "a 2.1 subscription without write loads as not a target")
st["subs"]["fleet"] = {"since": now, "wake": False, "write": "yes"}
t.ok(effective_subs(st, s, now)["fleet"]["write"] is False, "write is a target only when it is exactly true")
```

and in `section_tools`, after the `subscribe answers with the subscription` check:

```python
t.ok("+fleet: b subscribes, quiet, write; subscribers listed: b" in text(tool(b, "subscribe", channel="fleet", write=True)), "the subscribe tool takes write and says so")
t.ok("+fleet: b subscribes, wake, write;" in text(tool(b, "subscribe", channel="fleet", wake=True)), "the tool leaves write as it was when the call does not name it")
```

Keep the existing `pinned_ops` and `fresh` checks working: the `fresh` expectation gains `"write": False`.

- [ ] **Step 2: Run the two sections to see them fail**

Run: `python3 -I ./agent-mail self-test channels tools`
Expected: FAIL at "subscribe with write marks the channel a write target" (TypeError on the unknown keyword, or the check).

- [ ] **Step 3: Implement.** `effective_subs` reads `info.get("write") is True` into `"write"` and gives pinned entries `"write": False`. `subscribe` takes `wake=None, write=None`: for an existing or pinned channel, each attribute `None` keeps the current value, else `bool(value)`; for a new channel `None` means false; the stored record is `{"since": ..., "wake": ..., "write": ...}`. `subscription_text` appends `, write` after `wake`/`quiet` and before `, pinned`. The `subscribe` tool schema gains `"write": {"type": "boolean"}`; `tool_subscribe` passes `args.get("wake")` and `args.get("write")`. `cmd_subscribe` gains a `write` parameter passed through (the CLI flags come in Task 5).

- [ ] **Step 4: Run the sections and the whole suite**

Run: `python3 -I ./agent-mail self-test channels tools && python3 -I ./agent-mail self-test`
Expected: both PASS; the full run prints `agent-mail self-test: N checks passed` with N above 456.

- [ ] **Step 5: Commit**

```bash
git add agent-mail
git commit -S -m "Keep a write flag on each subscription"
```

### Task 2: `send` without `to`

**Files:**
- Modify: `agent-mail` (`send` ~1083; new `write_targets` and `send_default` beside it; `tool_send` ~2664; `cmd_send` ~3089)
- Test: `agent-mail`, `section_send` (~5207) and `section_tools`

**Interfaces:**
- Consumes: Task 1's `effective_subs` with `write`.
- Produces: `write_targets(state, settings, now) -> list[str]`: the channel names with `write` true, sorted, else `[settings.default_channel]`; `send_default(d, sender, subject, body, settings, state, in_reply_to=None, ident=None) -> list[tuple[str, SendResult | MailError]]`: one entry per target in order, a `SendResult` where the post was written and the `MailError` where that target refused it; it raises instead when no target took the post (the first refusal), or when the refusal is one the message itself earns (every code but `no_such_channel`, `rate_limited`, `duplicate`, `send_failed`), which `send` raises before any write.
- `settings.default_channel` exists from Task 3; until then the tests set it on a settings object by `dataclasses.replace(s, default_channel="fleet")` only if Task 3 is not yet in; do Task 3 first when working in order (it is independent and small).

- [ ] **Step 1: Write the failing tests** in `section_send`, at its end (the section's `d` has no channels; create them):

```python
s = load_settings({}, {}, "pi")
create_channel(d, "fleet", "fleet-wide notices")
create_channel(d, "a", "topic a")
create_channel(d, "b", "topic b")
st = load_state(d, "pi")
t.ok(write_targets(st, s, time.time()) == ["fleet"], "with no targets the default channel is the only target")
r = send_default(d, "pi", "s", "TO-THE-DEFAULT", s, st)
t.ok(len(r) == 1 and r[0][0] == "fleet" and os.path.exists(r[0][1].path) and "+fleet" in r[0][1].path, "a send without to posts to +fleet")
subscribe(d, "pi", st, s, "b", write=True, now=time.time())
subscribe(d, "pi", st, s, "a", write=True, now=time.time())
t.ok(write_targets(st, s, time.time()) == ["a", "b"], "targets are the write subscriptions in name order")
r = send_default(d, "pi", "s", "TO-BOTH", s, st)
t.ok([x[0] for x in r] == ["a", "b"] and all(isinstance(x[1], SendResult) and os.path.exists(x[1].path) for x in r), "one post per target")
e = t.raises(send_default, d, "pi", "s", "TO-BOTH", s, st)
t.ok(getattr(e, "code", None) == "duplicate", "the same body again within ten minutes is refused on every target, so the call fails")
r = send_default(d, "pi", "s", "TO-BOTH, LATER", s, st)
t.ok(all(isinstance(x[1], SendResult) for x in r), "a different body is not a duplicate across targets")
os.rename(os.path.join(d, "+b"), os.path.join(root, "gone-b"))
r = send_default(d, "pi", "s", "A-STAYS", s, st)
t.ok(isinstance(r[0][1], SendResult) and isinstance(r[1][1], MailError) and r[1][1].code == "no_such_channel", "a missing target is refused and the other target still takes the post")
e = t.raises(send_default, d, "pi", "s", "-----BEGIN PRIVATE KEY-----\nx\n-----END PRIVATE KEY-----", s, st)
t.ok(getattr(e, "code", None) == "looks_like_secret" and not [f for f in os.listdir(os.path.join(d, "+a")) if "PRIVATE" in open(os.path.join(d, "+a", f)).read()], "a message-level refusal writes nothing to any target")
```

In `section_tools`, after the subscribe checks:

```python
r = tool(b, "send", subject="no to", body="DEFAULT-POST")
t.ok(not r["isError"] and "sent " in text(r) and "to +fleet" in text(r), "the send tool without to posts to the write target")
r = tool(b, "send", to="", subject="empty to", body="EMPTY-TO-POST")
t.ok(not r["isError"] and "to +fleet" in text(r), "an empty to counts as omitted")
```

(`b` subscribed to `fleet` with `write` in Task 1's test, so `+fleet` is its target.)

- [ ] **Step 2: Run to see them fail**

Run: `python3 -I ./agent-mail self-test send tools`
Expected: FAIL with `NameError: write_targets`.

- [ ] **Step 3: Implement.** `write_targets` as in Interfaces. `send_default` loops over the targets calling `send(d, sender, f"+{ch}", subject, body, settings, in_reply_to=in_reply_to, ident=ident)`, catching `MailError` whose code is in `PER_TARGET_CODES = frozenset({"no_such_channel", "rate_limited", "duplicate", "send_failed"})` into the result and re-raising any other; at the end, with no `SendResult` in the list, it raises the first `MailError`. `tool_send` treats `to` missing or `""` as omitted: it loads the server's state (`load_state(d, name)`), calls `send_default`, and answers one line per target, `sent <id> to +<ch>` or `agent-mail: warning: not sent to +<ch>: <code>: <message>`, after the binding head; with `to` given, nothing changes. The `send` tool schema: `required` becomes `["subject", "body"]` and `to`'s description becomes `a session name, +channel, or all; omitted: your write targets, or the default channel when you have none`. `cmd_send` takes `to=None`: it reads `load_state(d, sender)` (no lock), calls `send_default`, prints each written path on its own line, each refusal as `agent-mail: warning: not sent to +<ch>: <message>` on stderr, and returns 0 (a raise from `send_default` reaches `main` as any `MailError` does).

- [ ] **Step 4: Run the sections and the suite**

Run: `python3 -I ./agent-mail self-test send tools && python3 -I ./agent-mail self-test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add agent-mail
git commit -S -m "Let send omit to: one post per write target, or the default channel"
```

### Task 3: The `default_channel` setting

**Files:**
- Modify: `agent-mail` (`SETTINGS` ~272, `FILE_KEYS` ~264, `Settings` ~286, `setting_value` ~322, `cli_settings` ~3225, the `send` parser ~3129)
- Test: `agent-mail`, `section_settings` (~3657)

**Interfaces:**
- Produces: `Settings.default_channel: str` (a channel name without `+`), from `--default-channel` (the `send` command), `AGENT_MAIL_DEFAULT_CHANNEL`, `default_channel` in `sessions.<name>`, `hosts.<host>` or the top level, default `"fleet"`; `source["default_channel"]` as for every setting.

- [ ] **Step 1: Write the failing tests** in `section_settings`, before the final `config({})`:

```python
config({})
t.ok(load_settings({}, {}, "pi").default_channel == "fleet" and load_settings({}, {}, "pi").source["default_channel"] == "default", "the default channel is fleet by default")
config({"default_channel": "ops", "sessions": {"pi": {"default_channel": "+team"}}})
t.ok(load_settings({}, {}, "pi").default_channel == "team" and load_settings({}, {}, "nix").default_channel == "ops", "default_channel comes from the session, then the file, with a leading + stripped")
t.ok(load_settings({}, {"AGENT_MAIL_DEFAULT_CHANNEL": "fleet"}, "pi").default_channel == "fleet" and load_settings({"default_channel": "a"}, {"AGENT_MAIL_DEFAULT_CHANNEL": "fleet"}, "pi").default_channel == "a", "the environment and then the flag beat the file")
for bad in ("", "+", "no spaces", "all", "john-x"):
    config({"default_channel": bad})
    t.ok(getattr(t.raises(load_settings, {}, {}, "pi"), "code", None) == "bad_setting", f"refuses default_channel {bad!r}")
config({})
```

and in `section_send`, after Task 2's checks:

```python
e = t.raises(send_default, d, "pi", "s", "NOWHERE", dataclasses.replace(s, default_channel="nowhere"), load_state(d, "fresh"))
t.ok(getattr(e, "code", None) == "no_such_channel" and "nowhere" in e.message, "a default channel that does not exist is named in the refusal")
```

- [ ] **Step 2: Run to see them fail**

Run: `python3 -I ./agent-mail self-test settings send`
Expected: FAIL with `AttributeError: default_channel` (or the unknown-key refusal).

- [ ] **Step 3: Implement.** Add `"default_channel"` to `FILE_KEYS`; the `SETTINGS` row `"default_channel": ("default_channel", "default_channel", "AGENT_MAIL_DEFAULT_CHANNEL", "fleet")`; the `Settings` field; in `setting_value`, for this key take a string, strip one leading `+`, and validate with `check_name(value, "channel")`, turning its `MailError` into `bad("...")` as the channels branch does. `cli_settings` passes `getattr(a, "default_channel", None)`; the `send` parser gains `--default-channel NAME` (help: `where a send without --to goes when NAME has no write targets (default fleet)`).

- [ ] **Step 4: Run the sections and the suite**

Run: `python3 -I ./agent-mail self-test settings send && python3 -I ./agent-mail self-test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add agent-mail
git commit -S -m "Add the default_channel setting, fleet by default"
```

### Task 4: `channels` shows the targets; version 2.2.0

**Files:**
- Modify: `agent-mail` (`channels_text` ~2500, `VERSION` ~2404)
- Test: `agent-mail`, `section_channels`, `section_mcp` (~4778), `section_cli` (~5459)

**Interfaces:**
- Produces: in `channels_text`, each subscribed channel's mark is `(you, quiet)`, `(you, wake)`, `(you, quiet, write)` or `(you, wake, write)`, then `(pinned)` as today; one line before the binding line: `posts go to: +a, +b` or `posts go to: +fleet (the default channel; no write targets)`; a `channels` call without a name prints neither marks nor the line.

- [ ] **Step 1: Write the failing tests** in `section_channels`, after the `unsubscribe drops the channel` check (fresh state for `pi2`):

```python
st2 = load_state(d, "pi2")
subscribe(d, "pi2", st2, s, "fleet", write=True, now=now)
subscribe(d, "pi2", st2, s, "c1", wake=True, now=now)
save_state(d, "pi2", st2)
listing = channels_text(d, "pi2", s, now)
t.ok("+fleet (you, quiet, write):" in listing and "+c1 (you, wake):" in listing and "\nposts go to: +fleet\n" in listing, "channels marks write targets and says where posts go")
t.ok("\nposts go to: +fleet (the default channel; no write targets)\n" in channels_text(d, "pi", s, now), "with no targets channels names the default channel")
t.ok("posts go to" not in channels_text(d, None, s, now), "without a name channels says nothing about posts")
```

In `section_mcp`, change the serverInfo expectation to `"2.2.0"` in both the dict and the message. In `section_cli`, extend the `channels NAME marks` check: after `t.run(["--dir", d, "subscribe", "pi", "+ops", "--wake", "--write"])`, assert `"you, wake, write" in listing and "posts go to: +ops" in listing` (Task 5 adds the flag; write this check in Task 5 instead if working strictly in order).

- [ ] **Step 2: Run to see them fail**

Run: `python3 -I ./agent-mail self-test channels mcp`
Expected: FAIL at "channels marks write targets" and at the serverInfo check.

- [ ] **Step 3: Implement.** In `channels_text`, build the mark from `sub["wake"]` and `sub["write"]`; after the channel lines and before `known sessions`, when `name` is set, append the `posts go to` line from `write_targets(state, settings, now)` (the state loaded once at the top of the function) and `settings.default_channel`. Set `VERSION = "2.2.0"`.

- [ ] **Step 4: Run the sections and the suite**

Run: `python3 -I ./agent-mail self-test channels mcp && python3 -I ./agent-mail self-test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add agent-mail
git commit -S -m "Show write targets in channels; version 2.2.0"
```

### Task 5: The CLI and the usage text

**Files:**
- Modify: `agent-mail` (the module docstring lines ~30-55, `build_parser` ~3120-3190, `dispatch` ~3243 and ~3272)
- Test: `agent-mail`, `section_cli` (~5434-5462) and `section_help` (~5197)

**Interfaces:**
- Produces: `send --from NAME [--to NAME|+CHANNEL|all] [--default-channel NAME] ...` (no `--to`: the write targets, else the default channel); `subscribe NAME CHANNEL [--wake | --no-wake] [--write | --no-write] [--backlog N]` with `argparse.BooleanOptionalAction` and `default=None` for both; `channels NAME` prints the marks and the `posts go to` line (Task 4).

- [ ] **Step 1: Write the failing tests** in `section_cli`, after the `unsubscribe drops a subscription` check (the section's `d` has `+fleet` and `+ops`):

```python
t.run(["--dir", d, "subscribe", "pi", "ops", "--write"])
t.ok(load_state(d, "pi")["subs"]["ops"]["write"] is True and "you, quiet, write" in t.run(["--dir", d, "channels", "pi"])[0], "subscribe --write marks a target and channels shows it")
t.run(["--dir", d, "subscribe", "pi", "ops", "--wake"])
t.ok(load_state(d, "pi")["subs"]["ops"] == {**load_state(d, "pi")["subs"]["ops"], "wake": True, "write": True}, "a later --wake leaves --write in place")
t.run(["--dir", d, "subscribe", "pi", "ops", "--no-write"])
t.ok(load_state(d, "pi")["subs"]["ops"]["write"] is False, "--no-write clears the target")
body = t.mk(os.path.join(root, "cli-body.txt"), "CLI-DEFAULT-SEND\n")
out, code, err = t.run(["--dir", d, "send", "--from", "pi", "--subject", "s", body])
t.ok(code == 0 and out.count("\n") == 1 and "/+fleet/" in out, "send without --to posts to the default channel")
t.run(["--dir", d, "subscribe", "pi", "ops", "--write"])
t.run(["--dir", d, "subscribe", "pi", "fleet", "--write"])
body2 = t.mk(os.path.join(root, "cli-body2.txt"), "CLI-TWO-TARGETS\n")
out, code, err = t.run(["--dir", d, "send", "--from", "pi", "--subject", "s", body2])
t.ok(code == 0 and out.count("\n") == 2 and "/+fleet/" in out and "/+ops/" in out, "send without --to posts to every write target, one path per line")
body3 = t.mk(os.path.join(root, "cli-body3.txt"), "CLI-THIRD-BODY\n")
out, code, err = t.run(["--dir", d, "send", "--from", "pi", "--subject", "s", "--default-channel", "nowhere", body3])
t.ok(code == 0 and out.count("\n") == 2, "with targets the default channel is not used")
```

`section_help` checks that every option string of every subparser appears on its command's usage line in the module docstring, so the docstring's `send` line must show `[--to NAME|+CHANNEL|all] [--default-channel NAME]` and the `subscribe` line `[--wake|--no-wake] [--write|--no-write] [--backlog N]` (BooleanOptionalAction registers both spellings).

- [ ] **Step 2: Run to see them fail**

Run: `python3 -I ./agent-mail self-test cli help`
Expected: FAIL at "subscribe --write marks a target" (unrecognized argument).

- [ ] **Step 3: Implement.** In `build_parser`: `--to` loses `required=True` and its help says `a session, a channel, or all; without it, NAME's write targets or the default channel`; `send` gains `--default-channel`; `subscribe` replaces `--wake` with `add_argument("--wake", action=argparse.BooleanOptionalAction, default=None, help=...)` and adds `--write` the same way. In `dispatch`, `cmd_subscribe(name, d, bare(a.channel), a.wake, a.backlog, s, write_target=a.write)` and `cmd_send(..., to=a.to ...)` with `a.to` possibly `None`. The module docstring's `send` and `subscribe` lines match the Interfaces block; add one line under `send`: `Without --to, posts go to NAME's write targets, or to the default channel.`

- [ ] **Step 4: Run the sections and the suite**

Run: `python3 -I ./agent-mail self-test cli help send && python3 -I ./agent-mail self-test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add agent-mail
git commit -S -m "CLI: send without --to, subscribe --write, the usage text"
```

### Task 6: README

**Files:**
- Modify: `README.md` (section 3 ~55-66, section 5 ~77-83, the settings table ~107-116, section 9 ~213-223)

**Interfaces:** none; text only.

- [ ] **Step 1: Section 3.** After the **Pinned channels** bullet add **Write targets:** a subscription may also be a write target (`--write`, or `write: true` in the tool); a send that names no recipient is a post to each of your write targets, one post per target, or to the default channel, `+fleet`, when you have none. Direct mail, replies and broadcasts always name their recipient. Subscribing again changes only the attributes the call names; unsubscribing drops the target with the subscription. Where posts go is shown by `channels`.

- [ ] **Step 2: Section 5.** `send {to?, subject, body, in_reply_to?}`: `to` is a session, `+channel` or `all`; omitted, your write targets, or the default channel when you have none, and the result names each post. `subscribe {channel, wake?, backlog?, write?}`. `channels {}` adds "and where your posts go".

- [ ] **Step 3: The settings table.** Add the row `| Default channel | `--default-channel` (send) | `AGENT_MAIL_DEFAULT_CHANNEL` | `default_channel` | `fleet` |`.

- [ ] **Step 4: Section 9.** `send --from NAME [--to NAME\|+CHANNEL\|all] [--default-channel NAME] --subject TEXT ...` with "without `--to`, NAME's write targets or the default channel"; `subscribe NAME CHANNEL [--wake\|--no-wake] [--write\|--no-write] [--backlog N]`.

- [ ] **Step 5: Check and commit.** Run `python3 -I ./agent-mail self-test help` (the usage text is the program's, not the README's; this guards nothing in the README but confirms the tree still passes), then:

```bash
git add README.md
git commit -S -m "README: write targets, the default channel, send without to"
```

### Task 7: Verification before the audit

**Files:** none changed; this task produces evidence.

- [ ] **Step 1: Full suite on three interpreters.** On Hera: `python3 -I ./agent-mail self-test` (3.14) and `/usr/bin/python3 -I ./agent-mail self-test` (3.9.6); on Vulcan, with the file copied to a temp dir (the self-test needs a file): `python3 -I /tmp/agent-mail-v22/agent-mail self-test` (3.13). Expected: three `checks passed` lines with the same count.

- [ ] **Step 2: Mutants, each run with the suite, each must fail at least one check.** (a) `effective_subs` drops the `write` carry; (b) `send_default` writes only the first target; (c) `send_default` swallows a per-target refusal without recording it; (d) `write_targets` returns `[]` instead of the default channel. Restore the file after each (`git checkout -- agent-mail`), and record the failing check's text for each in `~/Products/mailbox-rollout/scratch/v22/mutants.txt`.

- [ ] **Step 3: Audit.** A fresh sentinel probe (the parallelize skill's parent-history probe; the 18:58 one of 11-01 is spent), then a fresh `fess-auditor` in no-history mode over the branch's commits; fold real findings into the branch (amend or follow-up commits), re-audit a fix only when it is more than a fix.

- [ ] **Step 4: Land.** Rebase `work/v2.2` onto main first with re-signing (`git rebase --gpg-sign main`; main moves under docs commits while the branch is built, and `git range-diff` must show every patch identical), then merge it into main in the live checkout by fast-forward (`git merge --ff-only work/v2.2` after checking `git diff --cached --stat` is empty and the tree holds only known docs edits), verify `git ls-tree HEAD agent-mail` shows the new blob and `python3 -I ./agent-mail self-test` passes there, push to GitHub (decision 27), and record the new revision in the handoff; push main to the five fleet checkouts (decision 58's pattern; they run the installed nix package, not the checkout).

### Task 8: nix-config: the skills and the 2.2.0 pin

**Files:**
- Modify (in a nix-config worktree under `~/Products`, branch `work/agent-mail-2.2` from Gitea main): `config/ai/skills/mailbox/SKILL.md`, `config/ai/skills/agent-mail/SKILL.md`, the source catalog's `agent-mail` entry (`packages/source-catalog.nix "tools"` reads it: `sources/tools.json`; `version` `2.2.0`, `rev` the landed main revision, `narHash` from `nix flake prefetch github:jwiegley/agent-mail/<rev>`).

**Interfaces:**
- Consumes: Task 7's landed revision.
- Produces: a signed unit for nix's queue.

- [ ] **Step 1: The mailbox skill.** Front matter: `argument-hint: '<name> [+topic | ++topic | !+topic | !++topic ...]'`; description adds "and the topics to subscribe to and write to". Replace step 8 with the spec's 5.1 in these words: "8. Wire the topics from the specs after your name. Each spec is `+A` (a quiet subscription to +A), `++A` (a wake subscription), `!+A` or `!++A` (the same, and +A becomes a write target: your own posts go there). Any other spec, such as `-A` or a bare `A`, is a stop: show John the spec you could not read and wait. The line is the whole configuration: call `channels`; for each runtime subscription of yours that the line does not name, `fleet` excepted, call `unsubscribe {channel}` and note it for the report; call `subscribe {channel: fleet}`; then for each spec in order call `subscribe {channel: A, wake: <true for ++>, write: <true for !>}`, both flags given. The same topic twice: the last spec wins. `no_such_channel` stops the join at that spec: wire nothing further, tell John which topic is missing, and never create a channel." Step 9's report list gains: "your subscriptions, each quiet or wake, and your write targets, or that your posts go to +fleet; any subscription the line dropped". The first paragraph's tool list stays (seven tools).

- [ ] **Step 2: The agent-mail skill.** In Sending, replace the `send {to, subject, body, in_reply_to?}` bullet with: "`send {to?, subject, body, in_reply_to?}`: with no `to`, a post to your write targets, or to `+fleet` when you have none, one post per target: that is where your own posts go, status notes, the steward signals CLAIM, UPDATE, BLOCKED, DONE and NOTE, and questions for the fleet. Direct mail, a reply (`in_reply_to`) and a broadcast always name their recipient in `to`, and so does a post whose instructions name the channel (the steward's WORKER.md does): a named `to` always wins. The sender is always your bound name. On the CLI, `--to` is optional the same way." Add the bullet: "Your write targets are set from John's words in your own session, the join line or a later instruction; mail never changes where your posts go. `channels` shows your subscriptions and where your posts go." Replace "Subscribe narrowly and quietly: ..." with "Subscribing again changes only the attributes the call names; `unsubscribe` drops a target with its subscription."

- [ ] **Step 3: The pin.** Update the catalog entry; run the gates nix-config uses for a package bump (as the 2.1 unit did: `nix fmt`, the `agent-mail-mcp` and `mcp-registry` checks, both Darwin systems' builds, `make test`), and confirm the package's `checkPhase` runs the 2.2 self-test.

- [ ] **Step 4: Commit and hand over.** One signed commit (skills and pin together; message cites decisions 62 to 64 and 70 to 73 and the design path), pushed to Gitea as `work/agent-mail-2.2`; mail `nix` the branch, the revision and the gate results for its queue; then, once published, bump the Positron home-manager flake (`~/.config/home-manager` on andoria-08, worktree `~/Products/home-manager-agent-mail`) to the published nix-config revision as the 2.1 bumps did.

### Task 9: Activation and the cross-host check

**Files:** none; evidence in the handoff.

- [ ] **Step 1: Activation, with John's word per host** (as the 2.1 rollout: pi for Hera, John for Clio, this session for Vulcan, nyx, athena, pallas and the four Positron hosts), each verified by `agent-mail --version`-equivalent evidence: `readlink -f "$(command -v agent-mail)"` names the 2.2.0 store path and `agent-mail doctor` is clean. 2.1 and 2.2 coexist meanwhile.

- [ ] **Step 2: Cross-host check** per README section 8: a throwaway session on andoria-08 joined with `/mailbox amtest-andoria !+steward-test` and one on Hera subscribed to `+steward-test` exchange a post sent without `to`; record the lag and remove the sessions and their records.

- [ ] **Step 3: Tell the fleet.** One broadcast to `all`: a send without `to` now goes to your write targets or `+fleet`; `/mailbox` joins subscribe to `+fleet`; subscribe once by hand if you want it (spec 7.4).

- [ ] **Step 4: Close.** The handoff records the revisions, generations and evidence; the fess audit of the last work commit stands (Task 7); decision log updated.
