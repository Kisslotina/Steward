# Conventions — bases, fields, statuses

**Role of this file — the exact strings:** the canonical, case-sensitive base / field / status names
and select-option values the sorter reads **at runtime** (this is an always-loaded rule). This file
wins on exact spelling. The *structure and rationale* (which fields exist, relations, hierarchy, why)
live in `docs/data-model.md`; `Inbox.Type` values live in `taxonomy.md`.

Names must match Notion exactly (case-sensitive). Per-user data-source IDs live in
`bases.local.json` (git-ignored), written by `bootstrap-notion`.

## Bases (PARA + operational)
- **Inbox** — capture intake (Note, Status, Type, Target). Filtered view **🆕 New (unsorted)** (Status=New) is the work surface; rows drop off it when sorted.
- **Inbox Archive** — same schema as Inbox. `sort-inbox` **moves** filed rows here so the live Inbox stays small (the daily read stays cheap). Permanent record — never deleted.
- **Tasks**, **Projects**, **Goals**, **Ideas** — typed bases.
- **Knowledge** — facts / references / notes (Title, Notes, Area).
- **Areas DB** — life-domain lookup (relation target). Canon: `Career · Health · Sport · Money · Family · Content · Other`.
- **Reviews** — wide format: one row = period, text columns per area.
- **Not Recognized** — dead-letter (Note, Reason, Status, Source).
- **Outbox** — outbound queue, two consumers (Item, Type, Handler, Payload, Status, Source).

## Relations
Areas DB ← Goals / Projects / Ideas / Knowledge · Goals → Area · Projects → Area, Goal · Tasks → Project.

## Statuses
- Inbox.Status: `New → Sorted` (then the row is moved to **Inbox Archive**, not deleted)
- Tasks: `Done` (checkbox), Executor `Me / Auto (Steward)`, Tag `Triage / From Daily / Reminder / Shopping`
- Projects.Status: `Backlog · Active · On hold · Done`
- Goals.Status: `Not started · In progress · Done`
- Ideas.Type: `Content · Startup · Other` (subtype set by the sorter) · Ideas.Status: `New · In progress · Drafted · Posted`
- Not Recognized.Status: `Open → Resolved`
- Outbox.Status: `Queued → Done` (or `Failed`) · Outbox.Type: `notify · calendar · doc · sheet · other`
- Outbox.Handler: `Steward (MCP)` (connector actions) · `Concierge Gateway` (notifications)

## Audit
Every sort run writes a report: how many processed, what went where, what was closed, what landed in
Not Recognized, and what was enqueued to Outbox.

## Reading the Inbox (plan limits)
Server-side property/SQL queries are paid: `query_data_sources` needs Enterprise, `query_database_view`
needs Business. On free/standard plans skills read the Inbox with `notion-search` + per-page
`notion-fetch`, which is reliable only while the Inbox is small — hence **Inbox Archive** (filed rows
are moved out). Classification uses the deterministic table in `.claude/rules/routing.md` first, and
falls back to model judgement only on misses.
