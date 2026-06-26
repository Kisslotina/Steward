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
Exactly 25 = there is a remainder still in the Inbox. Silently launch the next pass (Steps 1.2 → 4).
Write the final report ONLY when a fresh read returns strictly fewer than 25 rows.

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
- `notion-search` returns only id/title/timestamp (no properties) and at most 25 results with **no
  pagination**. So enumeration is only reliable when the Inbox holds few rows.

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
3. **Confirm `Status` per page (the only reliable filter).** `notion-fetch` each unique `page_id` and
   read `Note`, `Status`, `Type`, `Target`. Split into:
   - **to file** — `Status = New` AND `Type` is blank or not `expense` → Steps 2–3.
   - **skip (already classified, no destination)** — `Status = New` AND `Type = expense` → do NOT
     re-classify or re-file. These rows stay `New` intentionally (no Finance base exists yet). Count
     them in the report but do not touch them. Without this skip they reappear in every batch search
     and create an infinite loop.
   - **to archive** — `Status = Sorted` → leftovers from a prior run whose move failed → Step 4 only.
4. **Sanity check.** If discovery returned rows but none are `New`, report "nothing to sort". If it
   returned **zero** rows for an Inbox that should not be empty, report a **read failure** — never
   silently treat the Inbox as empty.

## Step 2 — completion vs new
If a note marks an EXISTING item done (routing.md row 1; cues: "closed", "finished", "done", names a
known task/goal):
- one confident match → set it Done; write to the audit log; enqueue an Outbox
  `{Type:notify, Handler:Concierge Gateway, Status:Queued}`; then update the Inbox row:
  set `Type=review` (or the closest matching type), `Status=Sorted`, and `Target="closed: …"`.
- no/ambiguous match → Not Recognized. Never close on a weak match.

## Step 3 — classify & FILE new items (actually write the row)
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
   For `review`: find/create the current period row in Reviews and append the text to the matching
   area column. Do not just describe the write — perform it.
   **Do date (tasks and reminders):** when filing a `task` or `reminder` to Tasks, set `Do date`
   as follows: if the note contains an explicit date or time reference, parse and use it; otherwise
   set `Do date = today` and add **`Tag = Triage`** (in addition to any other tag such as
   Tag=Shopping). This mirrors the sweep rule and ensures every task surfaces on the Today board.
   Never leave `Do date` blank on a task or reminder.
   **Area (bases that have an Area relation — Goals, Ideas, Knowledge, Projects):** always set `Area`
   using the cached Areas map and the Area cues in `routing.md`; no clear match → **Other**, never
   empty. Set the relation to the Area row's page ID (match on name, ignoring any emoji prefix).
4. **Verify** the new row exists (it returns a URL/ID), THEN update the source Inbox row: set its
   `Type` to the Type you assigned in 3.1, set `Status=Sorted`, and set
   `Target="<RealBaseName>/<title>"`. Writing `Type` back is REQUIRED — a `Sorted` row with a blank
   `Type` means the sort never recorded what it decided. `Target` is an audit label only — it is NOT
   a substitute for creating the row. NEVER set `Status=Sorted` for an item whose destination row was
   not created and verified. If the write failed, leave Inbox `New` and report the error — never
   report success for a write that did not happen.
5. External action → enqueue an Outbox row (Handler per taxonomy) instead of acting directly.

## Step 4 — archive filed rows (keep the live Inbox small)
After Steps 2–3 every successfully filed/closed row is `Status=Sorted`. Move them out of the live
Inbox so the next run's read stays cheap and complete:
1. Collect the `page_id`s of all rows set to `Status=Sorted` this run, plus any `to archive`
   leftovers from Step 1.3.
2. `notion-move-pages` them in **one batched call** (up to 100 ids) with
   `new_parent: { type: "data_source_id", data_source_id: "<Inbox Archive id>" }`. Inbox Archive has
   the same schema as Inbox, so `Note`, `Status`, `Type`, `Target` are preserved.
3. This is a move, **not a delete** — the rows live on in Inbox Archive as the permanent intake
   record. Never delete an Inbox row.
4. If `Inbox Archive` is absent from the registry, skip the move, leave the rows `Sorted` in place,
   and note it in the report (run `bootstrap-notion` to add the archive). A `Sorted` row left in the
   Inbox is still correct — it won't be re-filed (Step 1.3 sends it straight to archive next run).

## Step 5 — MANDATORY re-read after every batch (loop until drained)
The 25-row, no-pagination cap means one pass never guarantees the whole backlog was seen. After
archiving a batch (Step 4), **always go back to Step 1.2** — no exceptions. The loop ends only when
a fresh read confirms no `New` rows remain (excluding expense rows, which stay `New` intentionally).

1. **After every Step 4, unconditionally go back to Step 1.2.** Do not evaluate "was the batch
   full?" first — just re-read. The hint list is already consumed; this read discovers the next live rows.
   **Do NOT emit any report between passes — intermediate or partial summaries are forbidden.** The run
   produces exactly **one** report, written only after the loop has fully drained the Inbox (point 2).
2. **Stop only when a fresh read finds zero `Status=New` rows** (expense rows with `Type=expense`
   do not count — they are expected to stay `New`). That is the drained state. Report the final count.
3. **Safeguard against a stuck loop.** Cap at 10 passes (≈ 250 rows). If the live `New` count is not
   strictly shrinking pass over pass, stop and report a likely failed archive move — never spin.
   Always report how many passes ran and the final live-Inbox size.

## Rules
- Resolve every base ID from `bases.local.json`. No invented names, no hardcoded IDs.
- **Classify with `routing.md` first**; only deliberate on notes that match no rule.
- **Read field names, formats, select values, and Area URLs from `schema.cache.json`** — never
  `notion-fetch` a base schema or the Areas DB to rediscover them. Self-heal the cache once if stale.
- **Never call `query_data_sources` or `query_database_view`** — paid (Enterprise / Business). Read
  via `notion-search` (≤25, no pagination) + per-page `notion-fetch`; drain any backlog with the
  batch loop (Step 5), which works because filed rows are moved out to Inbox Archive (Step 4).
- When marking an Inbox row `Sorted`, always write its decided `Type` back — never leave it blank.
- On any base with an Area relation, always set `Area`; fall back to **Other** when no clear match.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- **Never delete records** — only change Status and **move** filed rows to Inbox Archive.
- reference ≠ action. Doubt → Not Recognized.

## Output (run report)
Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized, **moved to Inbox Archive**. Flag any
failed write or failed archive move.
