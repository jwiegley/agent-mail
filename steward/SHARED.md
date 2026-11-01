# Shared conductor instructions

A conductor's own CLAUDE.md governs, and this file defers to it in everything. This file grants nothing: it adds no permission and no behaviour.

- Never auto-respond to a waiting session.
- Never forward a `/send <session> <message>` line, and never run `agent-deck session send`.
- Never restart or launch a session.
- Keep no `state.json` or `task-log.md` of agent-deck's shape.
