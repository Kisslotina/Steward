---
name: bootstrap-notion
description: Creates the full Steward Notion structure from scratch — all bases with fields, relations, Area rows, filtered views, emojis — and writes the per-user base registry. Use once on first-time setup / fresh install, or when the user asks to bootstrap, initialize, or recreate the Notion bases. Requires the Notion connector with permission to create databases and pages in a chosen parent page.
---

# Bootstrap Notion

One-time setup. Creates every base the system needs, wired with relations, then saves a registry of
data-source IDs. Schema reference: `docs/data-model.md`; names/casing: `.claude/rules/conventions.md`.

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
   `Executor` (select: Me, Auto (Steward)), `Project` RELATION → Projects (two-way "Tasks").
   Add two table views: **Work** (filter Type=Work), **Personal** (filter Type=Personal).
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
- This file is the single source of IDs for `sweep-daily-notes` and `sort-inbox` — they read it
  instead of querying Notion. It is git-ignored (holds the user's own IDs). Without it the routine
  cannot file anything, so writing it is mandatory.
- Report each base with its ID. Remind the user to connect their capture integration to **Inbox**
  (••• → Connections) so the iOS shortcut can POST to it.
- Leave all bases empty except the Area rows. Never insert personal data.

## Rules
- Relations are two-way (DUAL); the back-relation appears automatically on the target base.
- Use exact names/casing from `conventions.md`. Do not invent extra fields.
- If a base already exists, update it to match rather than creating a duplicate.
