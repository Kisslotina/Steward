@AGENTS.md

# Claude-specific notes

The shared operating instructions are in `AGENTS.md` (imported above). This file adds only
Claude-specific guidance, per Claude Code conventions
(https://code.claude.com/docs/en/memory).

## Skills

Domain procedures live in `.claude/skills/` and load on demand:

- `bootstrap-notion` — one-time: create all Notion bases, relations, views (first-time setup).
- `roll-day` — the daily routine: close-out → sweep → sort → carry-forward → refresh (calls `sort-inbox`).
- `sort-inbox` — classify + route Inbox records (invoked by `roll-day`, or run standalone).

## Rules

Path-agnostic rules in `.claude/rules/` load every session:

- `taxonomy.md` — record types + routing map (keep in sync with Notion).
- `conventions.md` — database / field / status names.

## Keep this file thin

Per the docs, CLAUDE.md should stay under ~200 lines and hold only stable facts. Deep or
procedural content belongs in skills (`.claude/skills/`) or rules (`.claude/rules/`), and the
full design in `docs/architecture.md`.
