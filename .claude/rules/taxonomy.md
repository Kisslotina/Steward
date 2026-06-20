# Taxonomy & routing map

Single taxonomy for `Inbox.Type` and where each type goes. Destinations are **base names**; their
per-user **data-source IDs** live in the local base registry `bases.local.json` (written once by
`bootstrap-notion`, git-ignored). The sort routine reads names here + IDs from the registry — it does
NOT query Notion for the base list each run.

## Types (Inbox.Type)

`task · reminder · reference · goal · idea · event · review · expense · unsure`

## Routing map

| Type | Destination base | Notes |
|---|---|---|
| task | **Tasks** | Type=Work/Personal, Priority, Do date, Assignee; Tag=From Daily if from Today; Executor=Me/Auto. "Buy X" shopping items are tasks (Tag=Shopping). |
| reminder | **Tasks** | Tag=Reminder + date |
| reference | **Knowledge** | a fact/note/reference (e.g. health note), NOT a task |
| goal | **Goals** | Horizon, Area, Target date |
| idea | **Ideas** | any idea/thought (content, startup, or other). Set `Ideas.Type` = Content / Startup / Other (the subtype). Status optional — mainly for Content. |
| event | **Outbox** (Type=calendar, Handler=Steward (MCP)) | enqueue; the executor routine creates the calendar event via MCP |
| review | **Reviews** | reflection on a period; find/create the current period row and append the text to the matching area column (Health/Sport/Career/Work/Money/Family) |
| expense | (no destination yet) | set Type, leave New, note it in the report — Finance not connected |
| unsure | **Not Recognized** | Note + Reason + Source; do not guess — leave for the user |

## Rules

- Route only to bases present in `bases.local.json`. If a type's base is missing → Not Recognized
  (reason: "no destination base: <type>"). **Never invent, rename, or abbreviate a base.**
- Do not split "partially recognized": file it confidently or send the whole thing to Not Recognized.
- `reference` ≠ action: knowledge is filed separately from tasks.
- `goal` ≠ `idea`: a **goal** is a committed, measurable outcome with a horizon/target date
  (e.g. "run a marathon this year"). An **idea** is an unstarted seed — a startup, product, or
  content idea. A startup/business/product concept is an `idea` (`Ideas.Type=Startup`), NOT a goal.
  When committed to execution it becomes a **Projects** row, not a Goal.
- Always set `Area` on bases that have it (Goals, Ideas, Knowledge, Projects): match by meaning, and
  if there is no clear, explicit match fall back to **Other** — never leave `Area` empty.
- The sort routine never executes outbound work — it enqueues an **Outbox** row with a `Handler`:
  `notify` → Concierge Gateway; `calendar/doc/sheet/other` → Steward (MCP, executor routine).

## Completion / update notes (not a new record)

Some notes mark an EXISTING item done (e.g. "closed task X", "finished goal Y"). These do NOT create
a new record — `sort-inbox` matches the existing item and updates its status:
- one confident match → set Done / Status=Done; write to the audit log; enqueue an Outbox `notify`
  row (Handler=Concierge Gateway, so it logs to Telegram); Inbox.Target = "closed: <base>/<title>".
- no/ambiguous match → Not Recognized ("completion note, no confident match"). Never guess.
