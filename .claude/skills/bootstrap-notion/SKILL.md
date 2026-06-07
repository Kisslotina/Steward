---
name: bootstrap-notion
description: Creates the full Steward Notion structure from scratch — all bases with fields, relations, Area rows, filtered views, emojis — and writes the per-user base registry. Use once on first-time setup / fresh install, or when the user asks to bootstrap, initialize, or recreate the Notion bases. Requires the Notion connector with permission to create databases and pages in a chosen parent page.
---

# Bootstrap Notion

One-time setup. Creates every base the system needs, wired with relations, then saves a registry of
data-source IDs. Schema reference: `docs/data-model.md`; names/casing: `.claude/rules/conventions.md`.

## Prerequisites
- Notion connector connected, able to create databases/pages.
- Ask the user which **parent page or teamspace** to create everything under. Offer to create a
  page named **Steward** as the container.

## Order (relations need their targets to exist first)
Create each as a database under the parent; put the emoji in the title (e.g. `📥 Inbox`).

1. **📥 Inbox** — `Note` (title), `Status` (select: New, Sorted), `Type` (select: task, reminder,
   reference, goal, thought, event, review, expense, unsure), `Target` (text). Date = created time.
2. **🔁 Areas DB** — `Area` (title), `Notes` (text). Add rows:
   Career, Health, Sport, Money, Family, Content, Other.
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
6. **💡 Content Ideas** — `Idea` (title), `Platform` (select: Telegram, X, LinkedIn),
   `Status` (select: New, In progress, Drafted, Posted), `Notes` (text),
   `Area` RELATION → Areas DB (two-way "Content Ideas"; optional).
7. **📚 Knowledge** — `Title` (title), `Notes` (text),
   `Area` RELATION → Areas DB (two-way "Knowledge"; optional). Home for `reference` notes/facts.
8. **🗒️ Reviews** — `Period` (title), `Date` (date), text columns:
   Health, Sport, Career, Work, Money, Family. (Wide format: one row = one period.)
9. **⚠️ Not Recognized** — `Note` (title), `Reason` (text),
   `Status` (select: Open, Resolved), `Source` (select: Inbox voice, Daily notes).
10. **📡 Outbox** — `Item` (title), `Type` (select: notify, calendar, doc, sheet, other),
    `Handler` (select: Steward (MCP), Concierge Gateway), `Payload` (text),
    `Status` (select: Queued, Done, Failed), `Source` (text).

## After — write the registry (required)
- Save **`bases.local.json`** in the project root: a JSON map of base name → data-source ID for every
  base created, e.g. `{"Inbox":"<id>","Tasks":"<id>", ...}`. `sort-inbox` reads this instead of
  querying Notion. (This file is git-ignored — it holds the user's own IDs.)
- Report each base with its ID. Remind the user to connect their capture integration to **Inbox**
  (••• → Connections) so the iOS shortcut can POST to it.
- Leave all bases empty except the Area rows. Never insert personal data.

## Rules
- Relations are two-way (DUAL); the back-relation appears automatically on the target base.
- Use exact names/casing from `conventions.md`. Do not invent extra fields.
- If a base already exists, update it to match rather than creating a duplicate.
