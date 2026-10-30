# Plan: land the Mailbox rollout, then the agent-mail MCP server (2026-10-31)

Frozen at 2026-10-31 00:16 PDT on John's instruction of 00:13 PDT: "commit, push, link this into ~/src/nix, commit and push there, and activate this on all machines so that we can enable this agent-mail MCP server on all hosts, all harnesses and begin using it for topic-based cross-communication as soon as possible."

Preparation (scope A, plan 2026-10-30) is complete: every branch holds its final, audited commit (handoff T1 to T10). This plan continues from there. Decisions: `/Users/johnw/Products/mailbox-rollout/decisions.md`. Design for the MCP server and channels: `docs/superpowers/specs/2026-10-30-agent-mail-mcp-channels-design.md`, treated as approved by the instruction above (stated to John in the reply of 00:1x PDT).

## Phase B: land and activate the Syncthing Mailbox (v1)

B1. Publish every branch: nyx (Gitea main fast-forwarded to b95b394), pallas-athena (main and production to 04519e0), andoria (main to 30e1066), nix-config (work/syncthing-mailbox 038a0737 published, main fast-forwarded through `bin/publish --publish` from `~/src/sys/nix`, which is the "link into ~/src/nix"), vulcan (rebased onto Vulcan's live /etc/nixos, nix-config input bumped, pushed to Gitea main). agent-mail main is landed last, at B5.
B2. Activate the hub and the Macs: Hera (`./build system`, then the switch; pi informed by mail before and after, since John's 16:02 decision made pi the Hera switcher and John's 00:13 instruction now covers it), then Clio.
B3. Activate the leaves: nyx (runbook chain), athena then pallas (pallas-athena docs; native base-policy check first), andoria-08 (home-manager switch) then andoria-t2, delphi-3bd4 and gpu-server (per-host start, XDG_RUNTIME_DIR on andoria-t2), vulcan (native build, switch).
B4. Pairing: every peer connected to Hera, the mailbox folder in sync, `~/Mailbox` present with `.stignore` on every host.
B5. Cutover, host by host: move waiting mail from `~/Products/mailbox` into `~/Mailbox`, replace `~/Products/mailbox` with a symlink to `~/Mailbox` (one for the Positron NFS home), land agent-mail work/mailbox-default on main (`~/src/sys/agent-mail`, pushed to the andoria and vulcan checkouts; clio, nyx, athena and pallas checkouts updated), tell every peer by mail that HEAD changed and watchers must restart.

Done when: B1 to B5 hold with evidence (each host's generation, Syncthing's connection list on Hera, a test message delivered across hosts), each repository's landed commit passed its fess audit, and nothing remains that only John can do.

## Phase C: agent-mail 2, the MCP server with channels

C1. Implement the design: `+name` channel directories one level below the mailbox (decision 6), the MCP server (stdio) with the settings John named (mailbox directory, poll interval, inotify), identity by the existing resolution chain with the reader lock and `me` check (decision 16), 30-day retention of channel posts and broadcasts and 7-day archive copies (decision 20), tests in the self-test, README. TDD, on a branch of agent-mail, audited by fess.
C2. Package agent-mail in nix-config (decision 7: store paths for the MCP and hook configuration, config from nix; checkouts stay the development source), register the MCP server for every harness in nix-config's AI configuration and in the Positron home-manager flake, publish, activate everywhere (Hera through the flow of B2).
C3. Verify a topic exchange between two hosts through the MCP server from at least two harnesses, and record the runbook in the README.

Done when: C1 to C3 hold with evidence, the final commits passed fess audits, and John has the list of anything that needed his word.

Stop-and-escalate: an activation that fails its gate (report and leave the host on its old generation); a conflict on Vulcan's /etc/nixos that needs intent; any step that would delete mail.
