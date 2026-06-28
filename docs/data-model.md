# Data model (Notion)

**Role of this file — the *what* and *why*:** which bases exist, what fields and relations they
carry, the PARA hierarchy, and the rationale behind the structure. It is the canonical source for
*structure*.

It is **not** the canon for exact strings. The precise base/field/status names and select-option
values the sorter relies on at runtime live in **`.claude/rules/conventions.md`** (and `Inbox.Type`
in `taxonomy.md`). Where a table below lists option values they are illustrative — if they ever
disagree with `conventions.md`, **conventions.md wins** and this file is corrected to match.

Full rationale also in `architecture.md` (sections 4–5a).

## Capture

### Inbox
| Field | Type | Values |
|---|---|---|
| Note | Title | captured text |
| Date | Created time | auto |
| Type | Select | task · reminder · reference · goal · idea · event · review · expense · unsure (set by Steward) |
| Status | Select | New → Sorted |
| Target | Text | where it was filed (audit) |

Second capture surface: free-text **Daily notes** on the `Today` page (swept into Inbox by the daily
routine).

### Today (page, not a database)

The daily driver — created EMPTY by `bootstrap-notion`, filled and handled daily by `roll-day`. Its
page ID is stored in `bases.local.json` under key `Today` (a page ID, not a data-source ID). Structure:

- `## 📝 Daily notes` — free-text capture area (swept into Inbox each day).
- a divider, then an inline **linked view of Tasks** (the day board): `GROUP BY Type`,
  `FILTER Done = false AND Do date ≤ today`, `SORT BY Do date ASC, Priority ASC`. Work/Personal come
  from the grouping; `Triage` and overdue/carried items surface via the `Tag` colour and the date sort.

One rolling page (not one page per day): `roll-day` refreshes its content daily; task history lives in
Tasks, swept notes live in Inbox.

## Typed bases (PARA)

| Base | Key fields | Relations |
|---|---|---|
| Areas DB | Area (title), Notes | ← Goals, Projects, Ideas |
| Goals | Goal, Horizon (Yearly/Short-term), Target date, Status, Notes | → Area ; ← Projects |
| Projects | Project, Status (Backlog/Active/On hold/Done), Deadline, Summary | → Area, → Goal ; ← Tasks |
| Tasks | Task, Done, Do date, Priority, Type (Work/Personal), Tag, Assignee, Executor (Me/Auto) | → Project |
| Ideas | Idea, Type (Content/Startup/Other), Status (New/In progress/Drafted/Posted), Notes | → Area (optional) |
| Knowledge | Title, Notes | → Area (optional) — home for `reference` notes/facts |
| Reviews | Note (title), Type (Health/Sport/Career/Work/Money/Family/Other), Date | long format; each `review` note is one new row — add many per area over time |

> **Ideas** is the seedbed for any idea — its `Type` subtype splits Content / Startup / Other.
> The `Drafted/Posted` statuses are mainly for `Type=Content`; leave them blank for
> Startup/Other. A startup or product idea is an `idea` here, NOT a `goal`; once committed to
> execution it graduates to a **Projects** row.
>
> "Buy X" shopping items are `task`s (Tag=Shopping), not a separate base. Per-user data-source IDs
> live in `bases.local.json` (git-ignored), written by `bootstrap-notion`; the sort routine reads it
> instead of querying Notion each run.

**Canonical Area vocabulary:** Career · Health · Sport · Money · Family · Content · Other.

**Hierarchy:** Area ⊃ Goal ⊃ Project ⊃ Task (every level above Task is optional).

## Dead-letter

### Not Recognized
| Field | Type | Values |
|---|---|---|
| Note | Title | unrecognized text |
| Reason | Text | why Steward was unsure |
| Status | Select | Open → Resolved |
| Source | Select | Inbox voice / Daily notes |
| Date | Created time | auto |

## Outbox — outbound queue (two consumers)

Single queue of outbound work the sort routine enqueues. **Two** consumers read it, each picking
the rows assigned to it via `Handler`:

- **Steward (MCP)** — a Claude **executor routine** that runs connector actions (Google Calendar,
  Docs, Sheets, other) natively via MCP. The LLM + native connectors handle these better than a
  plain microservice.
- **Concierge Gateway** — the always-on Node.js service, scoped to Telegram (posting
  logs/notifications, and later receiving incoming taps) — the part Claude can't do natively/24-7.

| Field | Type | Values |
|---|---|---|
| Item | Title | short description of the message/action |
| Type | Select | notify · calendar · doc · sheet · other |
| Handler | Select | Steward (MCP) · Concierge Gateway |
| Payload | Text | params/details (text to send, event fields, …) |
| Status | Select | Queued → Done (→ Failed) |
| Source | Text | which Inbox/record spawned it (audit) |
| Date | Created time | auto |

Flow: sort routine writes `Status=Queued` with a `Handler` → the matching consumer reads its Queued
rows → performs the action → sets `Done` (or `Failed`). Default routing: `notify` →
Concierge Gateway; `calendar/doc/sheet/other` → Steward (MCP). This replaces the earlier "Actions"
table. Two-way approvals, if needed later, extend a row with an approval status.
