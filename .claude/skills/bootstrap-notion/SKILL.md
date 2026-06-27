---
name: bootstrap-notion
description: Creates the full Steward Notion structure from scratch — all bases with fields, relations, Area rows, filtered views, emojis — and writes the per-user base registry. Use once on first-time setup / fresh install, or when the user asks to bootstrap, initialize, or recreate the Notion bases. Requires the Notion connector with permission to create databases and pages in a chosen parent page.
---

# Bootstrap Notion

One-time setup. Creates every base the system needs, wired with relations, then saves a registry of
data-source IDs.

> **Source of truth.** The field/value list below is an *executable copy* for running the bootstrap.
> Canon for *structure* is `docs/data-model.md`; canon for *exact names & select values* is
> `.claude/rules/conventions.md` (`Inbox.Type` in `taxonomy.md`). On any disagreement those win and
> this list is corrected to match — keep them in sync.

## Prerequisites
- Notion connector connected, able to create databases/pages.
- Ask the user **where** to create everything, and create the **Steward** page directly there:
  - their **Private** space (cleanest — pass no parent / a private page), or
  - an **existing** teamspace/page they name.
- **Do NOT create a new teamspace.** Notion auto-adds a default "Teamspace Home" page to every new
  teamspace, which we don't want. Always place Steward under an *existing* location.
- Create the **Steward** container page, the 10 databases, and the **Today** page inside it. Do NOT
  create any other pages, sections, or sub-pages.

## Order (relations need their targets to exist first)
Create each as a database under the parent; put the emoji in the title (e.g. `📥 Inbox`).

1. **📥 Inbox** — `Note` (title), `Status` (select: New, Sorted), `Type` (select: task, reminder,
   reference, goal, idea, event, review, expense, unsure), `Target` (text). Date = created time.
2. **🔁 Areas DB** — `Area` (title), `Notes` (text). Add rows, each with an emoji in the title text
   (emoji + space + name):
   💼 Career, 🩺 Health, 🏃 Sport, 💰 Money, 👨‍👩‍👧 Family, ✍️ Content, 📦 Other.
3. **🎯 Goals** — `Goal` (title), `Horizon` (select: Yearly, Short-term), `Target date` (date),
   `Status` (select: Not started, In progress, Done), `Notes` (text),
   `Area` RELATION → Areas DB (two-way "Goals").
4. **🚀 Projects** — `Project` (title), `Status` (select: Backlog, Active, On hold, Done),
   `Deadline` (date), `Summary` (text), `Area` RELATION → Areas DB (two-way "Projects"),
   `Goal` RELATION → Goals (two-way "Projects").
5. **✅ Tasks** — `Task` (title), `Done` (checkbox), `Do date` (date),
   `Priority` (select: Top, Secondary), `Type` (select: Work, Personal),
   `Tag` (select: Triage, From Daily, Reminder, Shopping), `Assignee` (text),
   `Executor` (select: Me, Auto (Steward)), `Deferred` (number, default 0 — how many times the
   user pushed this task to a later day; bumped by the defer buttons below),
   `Project` RELATION → Projects (two-way "Tasks").
   Add two table views: **Work** (filter Type=Work), **Personal** (filter Type=Personal).
   **Defer buttons (manual — API cannot create Button properties).** After the base exists, the user
   adds two **Button** properties in the Notion UI so a task can be pushed off today in one tap.
   The bootstrap routine cannot create these (the API has no button type), so it must **instruct the
   user** to add them (see "After — defer buttons" below) and confirm they exist:
   - **→ Tomorrow** — edits this page: set `Do date` = formula `dateAdd(now(), 1, "days")`, and
     `Deferred` = `Deferred + 1`.
   - **→ Next Week** — edits this page: set `Do date` = formula `dateAdd(now(), 7, "days")`, and
     `Deferred` = `Deferred + 1`.
6. **💡 Ideas** — `Idea` (title), `Type` (select: Content, Startup, Other),
   `Status` (select: New, In progress, Drafted, Posted),
   `Notes` (text), `Area` RELATION → Areas DB (two-way "Ideas"; optional).
   (`Type` is the idea subtype; `Drafted` / `Posted` are mainly for Type=Content.)
7. **📚 Knowledge** — `Title` (title), `Notes` (text),
   `Area` RELATION → Areas DB (two-way "Knowledge"; optional). Home for `reference` notes/facts.
8. **🗒️ Reviews** — `Period` (title), `Date` (date), text columns:
   Health, Sport, Career, Work, Money, Family. (Wide format: one row = one period.)
9. **⚠️ Not Recognized** — `Note` (title), `Reason` (text),
   `Status` (select: Open, Resolved), `Source` (select: Inbox voice, Daily notes).
10. **📡 Outbox** — `Item` (title), `Type` (select: notify, calendar, doc, sheet, other),
    `Handler` (select: Steward (MCP), Concierge Gateway), `Payload` (text),
    `Status` (select: Queued, Done, Failed), `Source` (text).
11. **📅 Today** (a PAGE, not a database) — the daily driver. Create it as a page under Steward with:
    - a heading `## 📝 Daily notes` followed by an EMPTY capture area (no list items) — free text
      typed here is swept into Inbox by the daily routine (`roll-day`);
    - a divider;
    - an inline **linked view of Tasks** (the day board). Configure it with the view DSL:
      `GROUP BY "Type"; FILTER "Done" = "false" AND "Do date" <= "today"; SORT BY "Do date" ASC,
      "Priority" ASC; SHOW "Task", "Do date", "Priority", "Tag", "Type"` (relative dates are quoted,
      e.g. `"today"`).
      Work / Personal come from the grouping; `Triage` and overdue/carried items are surfaced by the
      `Tag` colour and the date sort (overdue rises to the top). Leave the page otherwise EMPTY of
      personal data — `roll-day` fills and handles it daily.

## After — write the registry (REQUIRED, do not skip)
- Using your file tools, **write `bases.local.json` in the project root** — a JSON map of base name →
  data-source ID for EVERY base created. Use the exact name keys from `bases.local.example.json`
  (Inbox, Tasks, Projects, Goals, Ideas, Knowledge, Areas DB, Reviews, Not Recognized,
  Outbox). Example: `{"Inbox":"<data-source-id>", "Tasks":"<data-source-id>", ...}`.
- Also record the **Today page** under key `"Today"` — this value is a **page ID**, not a
  data-source ID, so `roll-day` can locate the daily page without searching.
- This file is the single source of IDs for `roll-day` and `sort-inbox` — they read it
  instead of querying Notion. It is git-ignored (holds the user's own IDs). Without it the routine
  cannot file anything, so writing it is mandatory.
- Report each base with its ID. Remind the user to connect their capture integration to **Inbox**
  (••• → Connections) so the iOS shortcut can POST to it.
- Leave all bases empty except the Area rows. Never insert personal data.

## After — write the schema cache (REQUIRED, do not skip)
You just created every base, so you already know its exact title field, date fields, select options,
and the URL of each Area row. Persist that as the **write-contract** so `sort-inbox` never has to
re-discover it through failed writes. **Write `schema.cache.json` in the project root** (git-ignored,
same place as `bases.local.json`); shape mirrors `schema.cache.example.json`.

Per **destination base** that `sort-inbox` writes (Tasks, Goals, Ideas, Knowledge, Reviews,
Not Recognized, Outbox) record:
- `title` — the exact title-property name to write the row name into (Tasks→`Task`, Goals→`Goal`,
  Ideas→`Idea`, Knowledge→`Title`, Reviews→`Period`, Not Recognized→`Note`, Outbox→`Item`).
- `dateFields` — every date property, by exact name (Tasks→`["Do date"]`, Goals→`["Target date"]`,
  others `[]`). These are written with the expanded form `date:<Field>:start`, never bare.
- `selects` — every select property → its full allowed-option list, copied verbatim from the create
  step (e.g. Goals `Horizon: ["Yearly","Short-term"]`, `Status: ["Not started","In progress","Done"]`;
  Ideas `Type: ["Content","Startup","Other"]`, `Status: ["New","In progress","Drafted","Posted"]`;
  Tasks `Type/Priority/Tag/Executor`; Outbox `Type/Handler/Status`; Not Recognized `Status/Source`).
  Writing any value outside this list errors ("Invalid select value").

Then record the top-level maps:
- `areas` — Area **name** (without the emoji prefix) → the **full Notion URL** of that Area row you
  just created (Career, Health, Sport, Money, Family, Content, Other). Relations require the full URL,
  not a bare id. Areas DB is canonical and effectively never changes, so this map is cached
  indefinitely.
- `reviews.currentRow` — leave `{"id":"", "periodLabel":""}` at bootstrap; `sort-inbox` fills it the
  first time it touches Reviews and refreshes it when the period rolls over.

This file is what makes the sort phase cheap: with it present, `sort-inbox` reads field names,
formats, select values, and Area URLs from disk instead of fetching base schemas or learning them
through API errors. Writing it here is mandatory.

## After — defer buttons (REQUIRED, manual UI step)
The Notion API has **no Button property type**, so neither this routine nor `roll-day` can create the
defer buttons programmatically — they must be added by hand once, in the Notion UI. Walk the user
through it and confirm both exist before finishing:

1. Open the **✅ Tasks** database → **+** (add property) → type **Button** → name it **→ Tomorrow**.
2. Action **Edit pages** → *this page* → add two edits:
   - `Do date` → **Set to** → switch to a formula and enter `dateAdd(now(), 1, "days")`.
   - `Deferred` → **Set to** → formula `Deferred + 1`.
3. Repeat for a second button named **→ Next Week**, identical but `dateAdd(now(), 7, "days")`.
4. (Optional) On the **Today** linked Tasks view, show the two button columns and the `Deferred`
   column so a task can be pushed straight from the morning board in one tap.

Once present, a task the user does not want today leaves the Today board the moment they tap a button
(its `Do date` moves into the future, so the `Do date <= today` filter drops it), and `Deferred`
records how often it has been pushed — `roll-day` surfaces chronically-deferred tasks from that count.

## Rules
- Relations are two-way (DUAL); the back-relation appears automatically on the target base.
- Use exact names/casing from `conventions.md`. Do not invent extra fields.
- If a base already exists, update it to match rather than creating a duplicate.
