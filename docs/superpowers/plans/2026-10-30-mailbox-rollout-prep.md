# Frozen plan: Mailbox rollout preparation (wiggum scope A)

Written 2026-10-30 14:55 PDT, before work. Read-only for the purpose of lowering the bar.

## Goal

Prepare, on branches only, every change needed for the Syncthing share "Mailbox" at `~/Mailbox` on ten hosts (hera, clio, vulcan, nyx, athena, pallas, andoria-08, andoria-t2, delphi-3bd4, gpu-server), as decided in `/Users/johnw/Products/mailbox-rollout/decisions.md`, with every non-activating gate passing and evidence shown, so that only John's pushes, activations and the cutover remain.

Reference runbook: `/Users/johnw/Products/mailbox-rollout/plan.md`, amended by decisions 10 to 20 (hidden state excluded from sync; hera listens on its tailnet address; syncthing-next 2.1.5 everywhere; pallas home-LAN only; June state deleted; generated keys only; no agent-deck change; channels later; Positron local directories per host; 30-day post retention is a design item, not rollout).

## Out of scope

Pushes, switches, activations, launchctl or systemctl changes, the mailbox cutover itself, deleting anything on any host, agent-mail v2 (the design document is not approved for building).

## Tasks and done-criteria

| # | Task | Done when |
|---|---|---|
| T1 | Seed identities on nyx, athena, pallas, vulcan (johnw), andoria-08, andoria-t2, delphi-3bd4, gpu-server | All eight device IDs recorded in `device-ids.txt`. Done 14:52 PDT. |
| T2 | nix-config branch `work/syncthing-mailbox` (worktree `~/Products/nix-syncthing-mailbox`) | Registry `syncthing.mailbox` pins all eight peers with real IDs (Positron peers by tailnet address); hera listens on `tcp://100.120.206.121:22000` and the test that forbids it is replaced; folder `mailbox` at `~/Mailbox`, no versioning; preflight writes the `.stignore` of decision 10; tests updated; `nix fmt`, `test/bin/quality --tier pre-commit`, `checks.aarch64-darwin.syncthing`, `./build system` and the clio system build pass with output shown; Linux checks pass or are recorded "not run" with the reason; signed commit. Separate branch `work/setup-mailbox-dir` holds the `/setup-mailbox` follow-up. |
| T3 | nyx branch `work/syncthing-mailbox` (worktree `~/Products/nyx-syncthing-mailbox`) | Module runs syncthing-next 2.1.5 as johnw, hera-only peer, `.stignore` of decision 10 as `ignorePatterns`, socket GUI, seeded gate, firewall rule; policy check evaluates; signed commit. |
| T4 | pallas-athena branch `work/syncthing-mailbox` (worktree `~/Products/pallas-athena-syncthing-mailbox`) | Same as T3 for both hosts with the repo's conventions satisfied (journal, PLAN.org, REPLICATE.md, HANDOFF); both base-policy checks evaluate; signed commit. |
| T5 | andoria home-manager branch `work/syncthing-mailbox` (worktree `~/Products/home-manager-syncthing-mailbox` on andoria-08) | Local-disk model: one unit for all four hosts, no `ConditionHost`, `--home /var/lib/jwiegley/syncthing-mailbox`, folder `/var/lib/jwiegley/Mailbox`, `~/Mailbox` symlink created by activation, syncthing-next 2.1.5, `ignorePatterns` of decision 10, listen `tcp://0.0.0.0:22000`, hera at its tailnet address; `nix flake check` and `home-manager build` pass on andoria-08; commit per the repo's signing convention. |
| T6 | vulcan nixos-config branch `work/syncthing-mailbox` (worktree `~/Products/nixos-config-syncthing-mailbox`) | johnw instance module as in plan 3.3 with syncthing-next from the system package, `ignorePatterns` of decision 10; evaluated with `--override-input nix-config` against T2's worktree; signed commit. |
| T7 | agent-mail branch `work/mailbox-default` (worktree `~/Products/agent-mail-mailbox`) | Default directory `~/Mailbox`; README reflects decisions (star through hera, vulcan member, Positron local directories with the `~/Mailbox` symlink, install paths, never create `~/Mailbox`); `self-test` passes (95 checks) with output shown; this plan and the handoff committed on the branch; signed commit. |
| T8 | fess audit | The last work commit on each of the six branches passes a fess audit by a separate evaluator; real findings fixed and folded in. |
| T9 | Branch currency | Each branch rebases cleanly onto its base, locally. |
| T10 | Handoff | `docs/superpowers/handoff-mailbox-rollout.md` lists, in order, every push, activation and cutover step John must approve, with the exact commands. |

## Stop and escalate

The wiggum conditions, plus: any step that would need a push, an activation, a second seed, a deletion, or an edit inside `~/src/nix`, `/etc/nixos` on vulcan, or any main/production branch.
