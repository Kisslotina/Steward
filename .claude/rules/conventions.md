# Conventions — bases, fields, statuses

Names must match Notion exactly (case-sensitive). Per-user data-source IDs live in
`bases.local.json` (git-ignored), written by `bootstrap-notion`.

## Bases (PARA + operational)
- **Inbox** — capture intake (Note, Status, Type, Target).
- **Tasks**, **Projects**, **Goals**, **Content Ideas** — typed bases.
- **Knowledge** — facts / references / notes (Title, Notes, Area).
- **Areas DB** — life-domain lookup (relation target). Canon: `Career · Health · Sport · Money · Family · Content · Other`.
- **Reviews** — wide format: one row = period, text columns per area.
- **Not Recognized** — dead-letter (Note, Reason, Status, Source).
- **Outbox** — outbound queue, two consumers (Item, Type, Handler, Payload, Status, Source).

## Relations
Areas DB ← Goals / Projects / Content Ideas / Knowledge · Goals → Area · Projects → Area, Goal · Tasks → Project.

## Statuses
- Inbox.Status: `New → Sorted`
- Tasks: `Done` (checkbox), Executor `Me / Auto (Steward)`, Tag `Triage / From Daily / Reminder / Shopping`
- Projects.Status: `Backlog · Active · On hold · Done`
- Goals.Status: `Not started · In progress · Done`
- Content Ideas.Status: `New · In progress · Drafted · Posted`
- Not Recognized.Status: `Open → Resolved`
- Outbox.Status: `Queued → Done` (or `Failed`) · Outbox.Type: `notify · calendar · doc · sheet · other`
- Outbox.Handler: `Steward (MCP)` (connector actions) · `Concierge Gateway` (notifications)

## Audit
Every sort run writes a report: how many processed, what went where, what was closed, what landed in
Not Recognized, and what was enqueued to Outbox.
