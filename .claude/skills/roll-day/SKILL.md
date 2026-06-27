---
name: roll-day
description: The single daily routine — run once per day to roll Steward forward. Use when the user says "daily run", "start my day", "roll the day", "process today", "run Steward for today", or wants the day kicked off. Orchestrates five phases in order — close-out, sweep, sort, carry-forward, refresh — sweeping Today's free-text Daily notes into the Inbox and calling sort-inbox to classify them. Supersedes the retired sweep-daily-notes step. Registry-driven (bases.local.json); never deletes records.
---

# Roll Day

The single daily routine. Run once per day to roll Steward forward: record what got done, sweep new
Daily notes into the Inbox, sort them, carry undone work forward, and leave the `Today` page clean
for tomorrow. This routine **orchestrates** the day and **calls `sort-inbox`** for classification —
it never re-implements sorting. It supersedes the retired `sweep-daily-notes` step.

**Built to run on a small model.** Every phase except **sort** is mechanical — plain reads and
writes, no judgement. Only the **sort** phase needs reasoning, and that is delegated to `sort-inbox`.
Follow the numbered steps **literally and in order**; do not improvise, batch, or skip. When a step
says STOP, stop. If anything is ambiguous, do the safe thing named in the step and record it in the
run report rather than guessing.

`Today` is **one rolling page**, not a page per day. Its linked Tasks view is filtered
`Done = false AND Do date <= "today"` and grouped by Type, so undone past tasks resurface on their
own each day — carry-forward therefore needs **no date mutation**.

## Step 0 — load the base registry (no live Notion list query)
1. With your **file tools**, read the local registry **`bases.local.json`** (base name → ID),
   written by `bootstrap-notion`. This is the source of truth for which bases exist and their IDs.
2. Read these three keys — **no hardcoded IDs anywhere**:
   - **`Inbox`** — data-source ID; where swept lines are appended (the sort phase reads it).
   - **`Tasks`** — data-source ID; the day board behind `Today`, and the close-out source.
   - **`Today`** — a **page ID** (NOT a data-source ID); the rolling daily page to sweep and refresh.
3. If `bases.local.json` is missing, or the **`Today`** key is absent or empty, **STOP** and tell the
   user to run `bootstrap-notion` first. Never guess, search for, or hardcode the Today page or any ID.
4. `sort-inbox` reads `routing.md`, `taxonomy.md`, and `conventions.md` itself in Phase 3 — `roll-day`
   does not classify, so it does not need them here.

## Tools (Notion MCP) — use these exact tools
- `notion-fetch` — read a page or a data-source schema by ID (the `Today` page; confirm a base's
  property names if unsure). Read, never assume property spellings.
- `notion-create-pages` — create a new row. For an Inbox row, set the parent to the Inbox
  **data-source ID** from Step 0: `parent: { data_source_id: "<Inbox id>" }`. Never use a database
  URL or a `database_id`.
- `notion-update-page` — edit existing page content: prepend "✅ " to the first line of a swept
  Daily-notes block (Phase 2), and remove the "✅"-marked blocks at the end of the day (Phase 5).
This routine does not call `notion-search` — every ID comes from the registry in Step 0, and the sort
phase's reads belong to `sort-inbox`.

## Phases (run in order)
The day rolls through five phases. Do them strictly in sequence; each later phase assumes the earlier
one finished. Nothing here executes outbound work — see Rules.

### 1. close-out — record only (read-only, no writes)
The `Today` board is a **live linked view of Tasks**, so a task ticked there has **already** updated
its Tasks row to `Done` — there is nothing to write back. This phase only **records** today's
completed tasks for the run report / audit log.
1. `notion-fetch` the `Today` page once and keep its content in memory for Phases 2 and 5 (do not
   re-fetch it each phase). Read the tasks shown as done today: `Done = true` with a `Do date` of today.
2. List their titles and count them for the report.
3. **Write nothing.** Do not re-close, re-check, or edit these rows — no double-close.

### 2. sweep — Daily-notes **blocks** → Inbox, then mark the source block "✅"
Move the free-text Daily notes into the Inbox so the sort phase can process them. **The capture unit
is a block, not a line.** Blocks are separated by a divider line that is exactly `---`. Sweep each
**block as a whole** into ONE Inbox row — **never split a block line-by-line** (splitting is what used
to fragment one meeting note into dozens of mis-parsed captures). A block may be a single line (a
quick capture) or many lines (a meeting note, a list). Work **one block at a time, in order**, and
**keep a running list of the Inbox `page_id`s you create** — Phase 3 hands that list to `sort-inbox`:
1. From the `Today` content read in Phase 1, look only under the `## 📝 Daily notes` heading. Split
   that area into **blocks** on lines that are exactly `---`. Drop empty blocks. Skip any block whose
   **first non-empty line already starts with "✅"** (it was swept on an earlier run).
2. For the next unmarked block, `notion-create-pages` **one** Inbox row,
   `parent: { data_source_id: "<Inbox id>" }`, `Status` = `New`, and:
   - **single-line block** → `Note` = that line (a normal capture; sort classifies it cheaply).
   - **multi-line block** → `Note` = `"🗒 "` + the block's **first non-empty line** (a short handle),
     and put the **full block text in the Inbox row's body/content** (the `content` field). A Notion
     **title does not reliably keep internal line breaks**, so the full multi-line text must live in
     the body, never crammed into the title. The leading `"🗒 "` on the title marks it as a note block
     so `sort-inbox` files it as **one** item (one high-priority task to process tomorrow), never
     re-split into fragments.
   Record the returned `page_id` in the swept-ids list.
3. Confirm the create returned a page ID. Then **immediately** `notion-update-page` the source block
   on `Today` to prepend "✅ " to its **first line**.
4. Only after the block's first line is marked "✅", move to the next block. Repeat 2–4 until no
   unmarked blocks remain.

The "✅" mark (on the block's first line) is the **only** dedup guard. If an Inbox row was created but
its source block **cannot be marked** "✅", **STOP immediately and report** which block — do not
continue. (Otherwise the next run re-imports it and duplicates the Inbox row.)

### 3. sort — invoke sort-inbox (process the WHOLE `New` backlog)
Invoke **`sort-inbox`**, passing the **swept `page_id`s from Phase 2 as its hint list** so it need
not rediscover them. **Its scope is still fixed and unambiguous: every Inbox row whose `Status = New`**
— the lines just swept **plus every earlier unsorted `New` capture** (e.g. notes the phone shortcut
added on previous days). Process **all** `New` rows and nothing else (a `Sorted` row is already done).

**Do not ask the user which items, or which subset, to sort, and never narrow to "just today's
swept lines."** `Status = New` already defines the scope completely.

`sort-inbox` owns classification, routing, completion-vs-new, and archiving filed rows to **Inbox
Archive** — **do not re-implement any of it here**. It also handles the read limit itself: Notion
`notion-search` shows at most 25 rows with no pagination, so `sort-inbox` drains the backlog in
**batches** — read ≤25, file them, move them out to Inbox Archive, re-read, repeat until no `New`
rows remain. `roll-day` does not loop or batch here; it invokes `sort-inbox` once and lets it drain.
The default-date rule also lives there: a swept task with no date gets `Do date = today`,
`Tag = Triage`, so it lands on today's board. **Note blocks** (rows whose `Note` starts with "🗒 ")
are filed there as a single high-priority task dated **tomorrow** (`Tag = Triage`), never split. See
`.claude/skills/sort-inbox/SKILL.md`. Wait for it to finish before Phase 4.

### 4. carry-forward — do nothing to dates (verify + surface chronic deferrals)
Undone past tasks **resurface on their own**: the Today board filters `Done = false AND Do date <=
today`, so anything still open with a past `Do date` is already shown. **Do not touch dates** —
**preserve each task's original `Do date`** as a slippage signal (how long it has been overdue).
`notion-fetch` the board only to **count** the overdue (carried) tasks for the report. Make no writes.

The user pushes tasks they do not want today off the board with the **defer buttons** (`→ Tomorrow` /
`→ Next Week`), which move `Do date` into the future and bump the task's **`Deferred`** counter. That is
a deliberate, healthy action — **roll-day never undoes it and never auto-defers anything itself.** But
a task pushed again and again is a signal it should be dropped, not done. So while counting the board,
also read each shown task's `Deferred` value and **flag any task with `Deferred >= 3`** for the report
(title + count) as "chronically deferred — consider dropping or rescoping." **Flag only — make no
writes**; the user decides what to do with it.

### 5. refresh — empty the Daily-notes area for the new day
Only **after** Phase 2 confirmed every swept block reached the Inbox, `notion-update-page` the `Today`
page to remove **every block whose first line starts with "✅"** (i.e. everything successfully swept)
from the `## 📝 Daily notes` area, leaving that area **empty under its heading** for the new day. In a
clean run that is the whole area — Today reads as freshly reset each morning. Do **not** remove an
unmarked block (it was never swept; a halt left it for the next run). `Today` is **one rolling page** —
never recreate it or create a dated / per-day page; only its notes area is cleared. Leave the linked
Tasks view untouched; it refreshes itself from the relative-date filter.

## Rules
- **Registry-driven.** Resolve `Inbox`, `Tasks`, and the `Today` page ID from `bases.local.json` —
  no invented names, no hardcoded IDs. Missing registry or absent `Today` key → STOP and point the
  user to `bootstrap-notion`.
- **Calls `sort-inbox`** for classification — never re-implements sorting, routing, completion, or
  archiving logic. Passes the swept `page_id`s as a hint; `New` remains the authoritative scope.
- **Scope of the sort phase is exactly `Status = New`.** Process every `New` Inbox row — the whole
  backlog, not only the lines swept this run. Never pause to ask which records to sort.
- **Never delete database records.** The only thing ever removed is transient "✅"-marked free-text
  blocks on the `Today` page, and only **after** they are safely in the Inbox (Phase 5 follows a
  verified Phase 2). Filed Inbox rows are **moved** to Inbox Archive by `sort-inbox`, never deleted.
- **Never perform outbound work.** Anything outbound is enqueued as an **Outbox** row with a
  `Handler` (`Concierge Gateway` for notifications, `Steward (MCP)` for connector actions); the
  executor routine or the Gateway runs it. `roll-day` does not act directly.
- **One rolling page.** Sweep and refresh operate on the single `Today` page; never create per-day pages.
- **One block at a time.** In sweep, create the Inbox row, record its id, verify it, mark the block's
  first line "✅", then move on. A block that cannot be marked halts the run (Phase 2). Blocks are split
  on `---` dividers; a multi-line block is swept whole (prefixed "🗒 "), never line-by-line.

## Output (run report)
Report, for this run:
- **close-out** — today's completed/closed tasks recorded (count + titles).
- **sweep → sort** — blocks swept into Inbox and handed to `sort-inbox` (count; note how many were
  multi-line note blocks); per-item outcomes (filed / closed / enqueued to Outbox / Not Recognized /
  moved to Inbox Archive) come from `sort-inbox`'s own report.
- **carry-forward** — overdue tasks carried (count); **chronically deferred** tasks flagged
  (`Deferred >= 3`): title + defer count, listed as "consider dropping" (no writes made).
- **errors / skips** — any block that could not be marked "✅" (and where the run stopped), a missing
  registry / `Today` key, or any failed write.
