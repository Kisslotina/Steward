---
name: sort-inbox
description: Classifies and files raw Inbox records into the EXISTING typed Notion bases, applies completion notes that close existing items, and enqueues outbound work to the Outbox. Use after capturing notes, or whenever the user asks to sort/process the Inbox. Invoked by roll-day after its sweep phase, or run standalone over the Inbox. Reads Inbox rows with Status=New, decides new-vs-completion, routes to a real base, sets Status=Sorted and Target, then moves filed rows to Inbox Archive.
---

# Sort Inbox

Processes every Inbox record with `Status=New`. Invoked by `roll-day` after its sweep phase, or run standalone over the Inbox.

**Built to run fast on a small model.** Classification is a table lookup (`routing.md`), not free
reasoning; schemas are read once and reused; the live Inbox is kept small so reads stay cheap. Follow
the steps in order. Do not call the paid query tools (see Step 1).

**One run drains the whole `New` backlog in batches.** `notion-search` shows at most 25 rows with no
pagination (Step 1), so a backlog larger than 25 cannot be read in a single pass. Instead of trying
to page past the cap, this skill processes **one batch of ≤25 at a time** (Steps 1–3), **moves that
batch out to Inbox Archive** (Step 4), then **re-reads the now-smaller live Inbox and repeats**
(Step 5). Because each filed batch leaves the live Inbox, the next read surfaces the next ≤25 — so the
loop drains any backlog reliably, on a free plan, with no pagination and no anchor-union guessing.

**⛔ NEVER output the final report or stop while a fresh read (Step 1.2) returned exactly 25 rows.**
Exactly 25 = the 25-row cap was hit, so there is almost certainly a remainder behind it — silently
launch the next pass (Steps 1.2 → 4). A read of fewer than 25 only means nothing is hidden behind the
cap; it is **not** itself a stop signal. The real exit is a single condition: **a fresh read finds
zero `Status=New` rows** (Step 5.2). Keep looping — process any `New` rows you see, archive, re-read —
until that holds, then write the one final report.

**⚠ HARD RULE — never stop after a single batch.** After every Step 4 archive, you MUST go back to
Step 1.2 and re-read the Inbox before declaring done. A batch of exactly 25 almost certainly has more
behind the cap. Even fewer than 25 is not a stopping signal on its own — re-read and confirm zero
`Status=New` rows remain. The only valid exit is a fresh read that finds no `New` rows.

## Step 0 — load the registry, rules, and schemas (no live Notion list query)
1. Read the local registry **`bases.local.json`** (base name → data-source ID), written by
   `bootstrap-notion`. This is the source of truth for which bases exist and their IDs.
2. Read `.claude/rules/routing.md` (the deterministic routing table — apply it **first** when
   classifying, before any reasoning), then `.claude/rules/taxonomy.md` (types + routing semantics)
   and `conventions.md` (names).
3. Route only to bases present in the registry. **Never invent, rename, or abbreviate a base**
   (no "Reflections DB", "Shopping DB", "Content Ideas"). If a type's destination base is not in the
   registry → that item goes to **Not Recognized** (reason "no destination base: <type>").
4. If `bases.local.json` is missing, stop and tell the user to run `bootstrap-notion` first
   (do a single discovery query only as a last resort, then offer to save the registry).
5. Resolve every ID used downstream from this registry only — **no hardcoded IDs anywhere**:
   `Inbox` (the source), `Inbox Archive` (where filed rows are moved, Step 4), and the targets
   `Tasks`, `Goals`, `Ideas`, `Knowledge`, `Reviews`, `Not Recognized`, `Outbox`. A base whose key is
   absent from the registry is treated as "does not exist" → items for it go to **Not Recognized**.
6. **Load the write-contract from `schema.cache.json` — do NOT fetch base schemas.** This
   git-ignored cache (written by `bootstrap-notion`; shape per `schema.cache.example.json`) holds, per
   destination base: `title` (the title-field name), `dateFields`, and `selects` (allowed select
   values), plus the top-level `areas` map (Area name → **full Notion URL**) and `reviews.currentRow`.
   Read it once and reuse it for every write this run — **never `notion-fetch` a base schema or the
   Areas DB to rediscover field names, date formats, select values, or Area URLs.** This, with the
   routing table, is what keeps the sort phase cheap and is what prevents the failed-write retry loops
   (wrong title field, bare date, invalid select value, bare-id relation).
   - **The Areas map is permanent.** Areas DB is canonical and effectively never changes, so the
     `areas` URLs are reused indefinitely across runs — never re-fetch them.
   - **Self-heal a missing or stale cache, once.** If `schema.cache.json` is absent, or a write later
     errors because the cache disagrees with Notion ("Property … not found", "Date type must be
     expanded", "Invalid select value", "Invalid agent URL"), discover the affected base's schema
     **once** via `notion-fetch`, **write the corrected contract back to `schema.cache.json`**, then
     keep reading from the file. Discover at most once per base per run — never per item.
   - **Reviews period shortcut (skip the weekly search).** Before searching Reviews for the current
     period row, check `reviews.currentRow`: if its `periodLabel` equals the current period label,
     **reuse its `id` with no search and no fetch**. Only when the label does not match (the period
     rolled over) do you find/create the period row and then update `reviews.currentRow` (`id` +
     `periodLabel`) in `schema.cache.json`.

## Step 1 — load the work list: every Inbox row with Status=New
Goal: get EVERY `Status=New` row, regardless of capture language or blank body, at a cost that does
**not** grow with Inbox history. Two facts about this Notion plan shape how:

- **No server-side property filter** on free/standard plans. `query_data_sources` (SQL) needs
  Enterprise; `query_database_view` needs Business. **Do not call them — they error.**
- `notion-search` returns id, **title, and timestamp** (no other properties) and at most 25 results
  with **no pagination**. So enumeration is only reliable when the Inbox holds few rows. **The Inbox
  title field IS `Note`** — so the full capture text needed to classify is already in the search
  result; you do **not** need a per-page `notion-fetch` to read it.

The system keeps the Inbox small on purpose: filed rows are moved to **Inbox Archive** in Step 4, so
the live Inbox holds only un-filed `New` rows (today's captures plus any the phone shortcut added).
That is what makes this read both cheap and complete.

1. **Take the hint list, if given.** When `roll-day` invokes this skill it passes the `page_id`s it
   just swept. Seed the work list from those — they need no discovery.
2. **Read one batch (≤25) of the live Inbox.** `notion-search` with
   `data_source_url: "collection://<Inbox id>"`, `query: "Inbox"` (a broad anchor, not a content
   word), `page_size: 25`, `max_highlight_length: 0`. Union the returned `page_id`s with the hint
   list. **Do not try to page past 25** — there is no cursor. If exactly 25 come back there is a
   backlog; you will reach the rest in Step 5 after this batch is archived out (do not guess at extra
   rows with more anchors). Remember whether this read returned a full 25 — Step 5 uses it.
3. **Classify straight from the search results — do NOT `notion-fetch` each page to confirm
   `Status`.** The title returned by Step 1.2 IS the `Note`, which is everything routing.md needs, and
   the live-Inbox invariant (filed rows are moved out to Inbox Archive in Step 4) means every row
   surfaced here is an unfiled `New` capture. So:
   - **Classify each row from its `Note` title** via routing.md (Steps 2–3). No confirmation fetch —
     this removes one `notion-fetch` per row, the single largest cost of a run.
   - **Note blocks (title starts with "🗒 ") → one task, never re-split.** `roll-day` sweeps a
     multi-line Daily-notes block as ONE Inbox row whose `Note` begins with "🗒 " (a handle) and whose
     **full text is in the row body**. Detect it by that prefix in the search title and file it via
     Step 3's **note-block short-circuit** as a single item — do **NOT** run the per-line routing table
     over its contents. To carry the full notes into the task, `notion-fetch` **this one row** to read
     its body (an allowed, scoped exception to the no-fetch rule, like the leftover guard). The full
     text is also preserved verbatim when the row is moved to Inbox Archive (Step 4).
   - **Detect expenses by cue, not by a pre-set `Type`.** routing.md row 8 (spent/paid/$/потратил/
     оплатил/купил за/руб/грн/…) identifies expense captures from the `Note` text itself; the old
     reliance on a pre-set `Type=expense` is dropped (it is re-derived from the cue and so is
     redundant). Collect these `page_id`s for Step 4, which **moves them out of the live Inbox** (with
     a pending-Finance marker) exactly like filed rows. They must LEAVE the live Inbox, otherwise they
     resurface in every `notion-search`, eat slots in the ≤25 cap, and spin the loop. Count them in
     the report.
   - **Leftover guard (rare, cheap).** A `Status=Sorted` row only lingers in the live Inbox if a prior
     run's archive move failed — which Step 4 records as `inbox.lastArchiveFailed = true` in
     `schema.cache.json`. **Only when that flag is set**, run a one-time per-page `notion-fetch` over
     this batch and drop rows already `Status=Sorted` (they go to Step 4 archive, not re-filed), then
     clear the flag. When the flag is unset (the normal case), trust the invariant and do not fetch.
4. **Sanity check.** If Step 1.2 returned **zero** rows for an Inbox that should not be empty, report a
   **read failure** — never silently treat the Inbox as empty.

## Step 2 — completion vs new
A note matching routing.md row 1 (cues: "closed", "finished", "done", "completed", "сделал",
"закрыл", "завершил", "выполнил", "готово") marks an EXISTING item done — it is NOT a new record.

**Do NOT search the typed bases to auto-close it.** Matching a completion note to an existing
Task/Project/Goal would cost an extra `notion-search` + `notion-fetch` per note and risks closing the
wrong item on a weak match. Instead, route every completion note straight to **Not Recognized**
(reason: "completion note — close the matching item manually") and let the user close the real item by
hand. No search, no fetch, no auto-close.

## Step 3 — classify & FILE new items (actually write the row)

**Note-block short-circuit — do this BEFORE the routing table.** If the row's `Note` starts with
"🗒 " it is a swept multi-line note block (a meeting note, a list). **File it as ONE task and skip the
routing table entirely** — never re-classify it line-by-line. Strip the "🗒 " marker, then
`notion-create-pages` ONE row in **Tasks** (`parent` = Tasks `data_source_id` from the registry):
- title = the note block's **first non-empty line** with the "🗒 " marker stripped (a concise handle);
- **`content` (task body) = the full block text** read from the Inbox row body, so the captured notes
  travel into the task in context (the same full text also remains on the archived Inbox row);
- `date:Do date:start` = **tomorrow** (today + 1 day) — the block is queued to be processed tomorrow;
- `Tag = Triage`;
- `Priority` = the **highest-ranked value** in the cached Tasks `Priority` `selects` (the top option —
  e.g. `Top`); if Tasks has no `Priority` field in the cache, omit it;
- `Type` = `Work` if the text has a clear work cue, else `Personal`.
Verify the row, then set the Inbox row `Type=task`, `Status=Sorted`, `Target="Tasks/<handle>"` and
archive it (Step 4). This realises "a meeting/notes block becomes **one** high-priority task to process
tomorrow," never a pile of mis-parsed fragments. For every other row (no "🗒 " prefix) continue below.

1. **Assign a `Type` with `routing.md` first** — apply the cue table top-to-bottom, first match wins.
   Only if a note matches **no** row, fall back to model judgement per `taxonomy.md`; still unclear →
   `unsure`. Distinguish `goal` (a committed, measurable outcome with a horizon) from `idea` (an
   unstarted seed — startup, product, or content); a startup/business/product concept is an `idea`,
   never a `goal`.
2. Map it to a base in the registry (Step 0). If none → Not Recognized.
   For `idea` → **Ideas**: also set the `Ideas.Type` subtype (routing.md) —
   `Content` (post/social/blog idea), `Startup` (business/venture/product concept), or `Other`.
   The `Drafted/Posted` statuses apply mainly to `Type=Content`; leave blank otherwise.
3. **Create the actual page/row** in that base's data source. Pass the destination base's
   **`data_source_id` from `bases.local.json`** as the parent, i.e.
   `parent: { type: "data_source_id", data_source_id: "<id-from-registry>" }`
   (equivalently the base's `collection://<id>`). **Do NOT use a database page URL or a
   `database_id`** — these bases are addressed by data-source ID, keyed by name in the registry.
   Use the property names from the cached schema (Step 0.6).
   For `review`: find/create the current period row in Reviews (reuse `reviews.currentRow` per
   Step 0.6 when the period label matches), then **append by read-modify-write** — Reviews has NO
   `append` command: `notion-fetch` the period row, read the current value of the matching area
   column, and `notion-update-page` with the FULL new value (old text + newline + the new line). Do
   not just describe the write — perform it.
   **`Type` = Work / Personal (tasks and reminders) — REQUIRED, never blank.** When filing a `task`
   or `reminder` to Tasks, ALWAYS set the Tasks `Type` select: `Work` if the note has a clear work
   cue (job, team, client, project/code, meeting, report, a named colleague), else `Personal`. This
   is the **same rule the note-block short-circuit uses** — apply it to every task, not just note
   blocks. The Today board is **grouped by `Type`**, so a task with a blank `Type` lands in an
   unlabeled "(No Type)" group and is invisible under Work / Personal — that is a filing failure, not
   a cosmetic gap. `Type` must be one of the cached Tasks `selects` values (`Work` / `Personal`);
   never leave it empty and never invent a third value.
   **`Priority` (tasks and reminders) — never blank, default to normal.** If the note carries an
   explicit urgency cue (e.g. "urgent", "asap", "important", "срочно", "важно"), set `Priority` to the
   **highest-ranked** value in the cached Tasks `Priority` `selects` (the top option — e.g. `Top`).
   Otherwise set `Priority` to the **normal default = the lowest-ranked value** in that `selects` list
   (e.g. `Secondary`) — never leave it empty. If Tasks has no `Priority` field in the cache, omit it.
   (Note blocks are the one exception — they always take the top `Priority`, per the short-circuit.)
   **Do date (tasks and reminders):** when filing a `task` or `reminder` to Tasks, set `Do date`
   as follows: if the note contains an explicit date or time reference, parse and use it; otherwise
   set `Do date = today` and add **`Tag = Triage`** (in addition to any other tag such as
   Tag=Shopping). This mirrors the sweep rule and ensures every task surfaces on the Today board.
   Never leave `Do date` blank on a task or reminder.
   **Area (bases that have an Area relation — Goals, Ideas, Knowledge, Projects):** always set `Area`
   using the cached Areas map and the Area cues in `routing.md`; no clear match → **Other**, never
   empty. Set the relation to the Area's **full Notion URL** from the cached `areas` map (Step 0.6) —
   never a bare page id (a bare id errors "Invalid agent URL"). Match on name, ignoring emoji prefix.
4. **Verify** the new row exists (it returns a URL/ID), THEN update the source Inbox row: set its
   `Type` to the Type you assigned in 3.1, set `Status=Sorted`, and set
   `Target="<RealBaseName>/<title>"`. Writing `Type` back is REQUIRED — a `Sorted` row with a blank
   `Type` means the sort never recorded what it decided. `Target` is an audit label only — it is NOT
   a substitute for creating the row. NEVER set `Status=Sorted` for an item whose destination row was
   not created and verified. If the write failed, leave Inbox `New` and report the error — never
   report success for a write that did not happen.
5. External action → enqueue an Outbox row (Handler per taxonomy) instead of acting directly.

### Exact call forms (copy verbatim — never improvise these)
These are the parameter shapes that previously failed and cost retries. Use them exactly.

1. **`notion-fetch` — always by `id`, never `url`:** `notion-fetch { id: "<page_id>" }`. Passing
   `url:` makes every fetch in the batch fail.
2. **`notion-move-pages` — param is `page_or_database_ids`, ONE batched call (≤100 ids):**
   `notion-move-pages { page_or_database_ids: [<ids…>], new_parent: { type: "data_source_id",
   data_source_id: "<Inbox Archive id>" } }`. Not `page_ids`.
3. **`notion-create-pages` — `parent` at the TOP level, not inside `pages[]`; dates expanded:**
   `notion-create-pages { parent: { type: "data_source_id", data_source_id: "<dest id>" },
   pages: [ { properties: { "<title-field>": "…", "date:<Field>:start": "YYYY-MM-DD", … } } ] }`.
   A `parent` key inside a `pages[]` object errors "Unrecognized key: parent". Write dates as
   `date:<Field>:start` (e.g. `date:Do date:start`) — a bare `Do date` errors "Date type must be
   expanded".
4. **Area relation — always the full Notion URL** from the cached `areas` map (Step 0.6), never a
   bare page id (a bare id errors "Invalid agent URL").
5. **Select properties — only values from the write-contract** (`selects` in `schema.cache.json`).
   Anything else errors "Invalid select value" (e.g. free-text "December 2026" into Goals.`Horizon`,
   whose only values are `Yearly` / `Short-term`).
6. **Reviews has no append** — read-modify-write: `notion-fetch` the period row → read the area
   column → `notion-update-page` with the full new value. There is no `append` command.
7. **Transient server errors → retry once, then report.** A 5xx / timeout / "service unavailable"
   on a single write is usually transient: **retry that one call exactly once.** If it fails again,
   leave that Inbox row `New` and report it — do not abort the batch and do not retry more than once.
   This applies ONLY to transient server errors, never to the validation errors above (those are
   contract bugs — fix the call shape or self-heal the cache, do not retry blindly).

## Step 4 — archive filed rows (keep the live Inbox small)
After Steps 2–3 every successfully filed/closed row is `Status=Sorted`. Move them out of the live
Inbox so the next run's read stays cheap and complete:
1. Collect the `page_id`s to move out this run:
   a. every row set `Status=Sorted` by Steps 2–3 (filed / closed items);
   b. the **expense** rows from Step 1.3 — first update each: set `Status=Sorted` and
      `Target="expense → pending Finance"` (`Type=expense` is preserved). This keeps Inbox Archive's
      invariant ("everything here is Sorted") and lets a future Finance consumer find them by
      `Type=expense` + that `Target`. If the registry later gains a dedicated holding base
      (e.g. `Expenses Pending`), move expense rows there instead — destination is registry-driven;
   c. any already-`Status=Sorted` leftovers surfaced by the Step 1.3 leftover-guard reconciliation
      (prior-run rows whose archive move had failed) — archive them now, never re-file.
2. `notion-move-pages` them all in **one batched call** (up to 100 ids) with
   `new_parent: { type: "data_source_id", data_source_id: "<Inbox Archive id>" }`. Inbox Archive has
   the same schema as Inbox, so `Note`, `Status`, `Type`, `Target` are preserved. **Expense rows are
   moved out too** — they no longer stay `New` in the live Inbox.
   - **Record the move outcome for the next run's leftover guard (Step 1.3).** On success, set
     `inbox.lastArchiveFailed = false` in `schema.cache.json`. If the move still fails after the
     one allowed retry, set `inbox.lastArchiveFailed = true` (so the next run runs the one-time
     per-page reconciliation that skips already-`Sorted` rows) and report the failure.
3. This is a move, **not a delete** — the rows live on in Inbox Archive as the permanent intake
   record. Never delete an Inbox row.
4. If `Inbox Archive` is absent from the registry, skip the move, leave the rows `Sorted` in place,
   and note it in the report (run `bootstrap-notion` to add the archive). A `Sorted` row left in the
   Inbox is still correct — it won't be re-filed (Step 1.3 sends it straight to archive next run).

## Step 5 — MANDATORY re-read after every batch (loop until drained)
The 25-row, no-pagination cap means one pass never guarantees the whole backlog was seen. After
archiving a batch (Step 4), **always go back to Step 1.2** — no exceptions. The loop ends only when
a fresh read confirms no `Status=New` rows remain. (Expense rows are now moved out in Step 4, so they
no longer linger as `New` — there is no expense exclusion to carry anymore.)

1. **After every Step 4, unconditionally go back to Step 1.2.** Do not evaluate "was the batch
   full?" first — just re-read. The hint list is already consumed; this read discovers the next live rows.
   **Do NOT emit any report between passes — intermediate or partial summaries are forbidden.** The run
   produces exactly **one** report, written only after the loop has fully drained the Inbox (point 2).
2. **Stop only when a fresh read finds zero `Status=New` rows.** That is the drained state — and it
   is now reachable for an all-expense remainder too, because Step 4 moves expense rows out instead of
   leaving them `New`. Report the final count.
3. **Safeguard against a stuck loop.** Cap at 10 passes (≈ 250 rows). If the live `New` count is not
   strictly shrinking pass over pass, stop and report a likely failed archive move — never spin.
   Always report how many passes ran and the final live-Inbox size.

## Rules
- Resolve every base ID from `bases.local.json`. No invented names, no hardcoded IDs.
- **Classify with `routing.md` first**; only deliberate on notes that match no rule.
- **Read field names, formats, select values, and Area URLs from `schema.cache.json`** — never
  `notion-fetch` a base schema or the Areas DB to rediscover them. Self-heal the cache once if stale.
- **Never call `query_data_sources` or `query_database_view`** — paid (Enterprise / Business). Read
  via `notion-search` (≤25, no pagination) and **classify from the returned `Note` title — no per-page
  confirmation fetch** (Step 1.3); drain any backlog with the batch loop (Step 5), which works because
  filed rows are moved out to Inbox Archive (Step 4). Per-page `notion-fetch` is now used only for the
  rare leftover reconciliation (Step 1.3 guard).
- When marking an Inbox row `Sorted`, always write its decided `Type` back — never leave it blank.
- **Every Tasks row gets a `Type` (`Work` / `Personal`)** — required on regular tasks and reminders,
  not just note blocks. The Today board groups by `Type`; a blank-`Type` task is invisible there.
- **Every Tasks row gets a `Priority`** — never blank. Use the top value on an explicit urgency cue,
  otherwise the normal default (the lowest-ranked `Priority` value, e.g. `Secondary`). Note blocks
  always take the top value.
- On any base with an Area relation, always set `Area`; fall back to **Other** when no clear match.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- **Never delete records** — only change Status and **move** filed rows to Inbox Archive.
- reference ≠ action. Doubt → Not Recognized.

## Output (run report)
Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized, **moved to Inbox Archive**. Flag any
failed write or failed archive move.
