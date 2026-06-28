---
name: sort-inbox
description: Classifies and files raw Inbox records into the EXISTING typed Notion bases, applies completion notes that close existing items, and enqueues outbound work to the Outbox. Use after capturing notes, or whenever the user asks to sort/process the Inbox. Invoked by roll-day after its sweep phase, or run standalone over the Inbox. Reads Inbox rows with Status=New, decides new-vs-completion, routes to a real base, sets Status=Sorted and Target, then moves filed rows to Inbox Archive.
---

# Sort Inbox

Processes every Inbox record with `Status=New`. Invoked by `roll-day` after its sweep phase,
or run standalone over the Inbox.

**Built to run fast on a small model.** Classification is a table lookup (`routing.md`), not
free reasoning; schemas are read once and reused; the live Inbox is kept small so reads stay
cheap. Follow the steps in order.

## Notion free-plan constraints (why this skill is shaped this way)

Two facts about this Notion plan dictate the whole design:

- **No server-side property filter.** `query_data_sources` (SQL) needs Enterprise;
  `query_database_view` needs Business. **Never call them — they error.** Read via
  `notion-search` only.
- **`notion-search` returns id, title, and timestamp only — at most 25 results, no
  pagination.** The Inbox title field IS `Note`, so the full capture text needed to classify
  is already in the search result; no per-row `notion-fetch` to read or confirm it.

Because of the 25-row cap, one read can't see a backlog larger than 25. So the skill drains
in batches: read ≤25 → classify & file → **move that batch out to Inbox Archive** → re-read
the now-smaller live Inbox → repeat. Each filed batch leaves the live Inbox, so the next read
surfaces the next ≤25. The loop and its exit condition are defined once, in **Step 5**.

## Step 0 — load the registry, rules, and schemas (no live Notion list query)

1. Read the local registry **`bases.local.json`** (base name → data-source ID), written by
   `bootstrap-notion`. This is the source of truth for which bases exist and their IDs.
   If it is missing, stop and tell the user to run `bootstrap-notion` first (a single
   discovery query only as a last resort, then offer to save the registry).
2. Read `.claude/rules/routing.md` (the deterministic routing table — apply it **first**,
   before any reasoning), then `.claude/rules/taxonomy.md` (types + routing semantics) and
   `conventions.md` (names).
3. **Route only to bases present in the registry. Never invent, rename, or abbreviate a base**
   (no "Reflections DB", "Shopping DB", "Content Ideas"). No hardcoded IDs anywhere — resolve
   every downstream ID from the registry: `Inbox`, `Inbox Archive`, and the targets `Tasks`,
   `Goals`, `Ideas`, `Knowledge`, `Reviews`, `Not Recognized`, `Outbox`. A type whose
   destination base is absent → that item goes to **Not Recognized** (reason "no destination
   base: <type>").
4. **Load the write-contract from `schema.cache.json` — do NOT fetch base schemas.** This
   cache (written by `bootstrap-notion`; shape per `schema.cache.example.json`) holds, per
   destination base: `title` (title-field name), `dateFields`, and `selects` (allowed select
   values), plus the top-level `areas` map (Area name → **full Notion URL**). Read it once and
   reuse it for every write this run — **never
   `notion-fetch` a base schema or the Areas DB** to rediscover field names, date formats,
   select values, or Area URLs. This is what prevents failed-write retry loops.
   - The Areas map is permanent (Areas DB is canonical) — reuse the URLs indefinitely.
   - **Reviews is a plain create-row.** A `review` note is filed as ONE new Reviews row
     (`Note` = the text, `Type` = the area, `Date` = today) — no period row, no search, no
     read-modify-write. Use the cached `Reviews.selects.Type` values for `Type`.
   - If the cache is missing or a write later disagrees with Notion → **self-heal once**, per
     `references/recovery.md` §1.

## Step 1 — load the work list: every Inbox row with Status=New

The live Inbox holds only un-filed `New` rows, because filed rows are moved to Inbox Archive
in Step 4. That invariant is what makes this read both cheap and complete.

1. **Take the hint list, if given.** When `roll-day` invokes this skill it passes the
   `page_id`s it just swept. Seed the work list from those — they need no discovery.
2. **Read one batch (≤25) of the live Inbox.** `notion-search` with
   `data_source_url: "collection://<Inbox id>"`, `query: "Inbox"` (a broad anchor, not a
   content word), `page_size: 25`, `max_highlight_length: 0`. Union the returned `page_id`s
   with the hint list. There is no cursor — do not try to page past 25; the batch loop
   (Step 5) reaches the rest. Remember whether this read returned a full 25 — Step 5 uses it.
3. **Classify straight from the search results — do NOT `notion-fetch` each page.** The title
   returned is the `Note` (everything routing.md needs), and the live-Inbox invariant means
   every surfaced row is an unfiled `New` capture. Removing the confirmation fetch eliminates
   the single largest cost per run. Two special cases:
   - **Note blocks (title starts with "🗒 ") → one task, never re-split.** `roll-day` sweeps a
     multi-line Daily-notes block as ONE Inbox row whose `Note` begins with "🗒 " (a handle);
     the full text is in the row body. File it via Step 3's note-block short-circuit as a
     single item — do **not** run the per-line routing table over its contents. To carry the
     full notes into the task, `notion-fetch` **this one row** to read its body (a scoped,
     allowed exception to the no-fetch rule). The full text is also preserved verbatim when
     the row is archived (Step 4).
   - **Detect expenses by cue, not by a pre-set `Type`.** routing.md row 8 (spent/paid/$/
     потратил/оплатил/купил за/руб/грн/…) identifies expense captures from the `Note` text.
     Collect these `page_id`s for Step 4, which **moves them out of the live Inbox** (with a
     pending-Finance marker) exactly like filed rows — otherwise they resurface in every
     search, eat slots in the ≤25 cap, and spin the loop. Count them in the report.
   - **Leftover guard:** only if `inbox.lastArchiveFailed = true` — see
     `references/recovery.md` §2. Normal case: trust the invariant, do not fetch.
4. **Sanity check.** If Step 1.2 returned **zero** rows for an Inbox that should not be empty,
   report a **read failure** — never silently treat the Inbox as empty.

## Step 2 — completion vs new

A note matching routing.md row 1 (cues: "closed", "finished", "done", "completed", "сделал",
"закрыл", "завершил", "выполнил", "готово") marks an EXISTING item done — it is NOT a new
record.

**Do NOT search the typed bases to auto-close it** — that would cost an extra search + fetch
per note and risks closing the wrong item on a weak match. Route every completion note to
**Not Recognized** (reason: "completion note — close the matching item manually") and let the
user close the real item by hand.

## Step 3 — classify & FILE new items (actually write the row)

Read `references/notion-call-forms.md` once before writing — it holds the exact, verbatim
call shapes (`notion-create-pages`, dates, Area URL, selects, Reviews create-row).

**Note-block short-circuit — do this BEFORE the routing table.** If the row's `Note` starts
with "🗒 " it is a swept multi-line note block. **File it as ONE task and skip the routing
table entirely.** Strip the "🗒 " marker, then `notion-create-pages` ONE row in **Tasks**:
- title = the block's **first non-empty line** (marker stripped) — a concise handle;
- **`content` (task body) = the full block text** from the Inbox row body;
- `date:Do date:start` = **tomorrow** (today + 1);
- `Tag = Triage`;
- `Priority` = the **highest-ranked** value in the cached Tasks `Priority` `selects`
  (omit if Tasks has no `Priority` field);
- `Type` = `Work` if the text has a clear work cue, else `Personal`.
Verify the row, set the Inbox row `Type=task`, `Status=Sorted`, `Target="Tasks/<handle>"`,
and archive it (Step 4). For every other row (no "🗒 " prefix) continue below.

1. **Assign a `Type` with `routing.md` first** — apply the cue table top-to-bottom, first
   match wins. Only if a note matches **no** row, fall back to `taxonomy.md` judgement; still
   unclear → `unsure`. Distinguish `goal` (a committed, measurable outcome with a horizon)
   from `idea` (an unstarted seed); a startup/business/product concept is an `idea`, never a
   `goal`.
2. **Map to a base** in the registry. If none → Not Recognized. For `idea` → **Ideas**: also
   set `Ideas.Type` — `Content` (post/blog), `Startup` (business/product), or `Other`. The
   `Drafted/Posted` statuses apply mainly to `Type=Content`; leave blank otherwise.
3. **Create the actual page/row** in that base's data source, with
   `parent: { type: "data_source_id", data_source_id: "<id-from-registry>" }`. Use property
   names from the cached schema (Step 0). Per-field rules — all REQUIRED, never blank:
   - **`Type` = Work / Personal** on every `task` and `reminder` filed to Tasks (work cue →
     Work, else Personal). The Today board is grouped by `Type`; a blank-`Type` task lands in
     an invisible "(No Type)" group — a filing failure, not a cosmetic gap. Must be a cached
     `selects` value; never invent a third.
   - **`Do date`** on every `task`/`reminder`: parse an explicit date/time if present;
     otherwise `Do date = today` + `Tag = Triage`. Never blank.
   - **`Area`** on bases that have it (Goals, Ideas, Knowledge, Projects): set from the cached
     `areas` map and the Area cues in `routing.md` (full Notion URL, match on name ignoring
     emoji). No clear match → **Other**, never empty.
   - **`review`:** create ONE new Reviews row — `Note` (title) = the text, `Type` = the area
     from the Review-type cues in `routing.md` (cached `Reviews.selects.Type`; no match →
     `Other`), `date:Date:start` = today. A plain `notion-create-pages`, no read-modify-write.
4. **Verify** the new row exists (it returns a URL/ID), THEN update the source Inbox row: set
   `Type` to what you assigned in 3.1, `Status=Sorted`, `Target="<RealBaseName>/<title>"`.
   Writing `Type` back is REQUIRED — a `Sorted` row with blank `Type` means the sort never
   recorded its decision. `Target` is an audit label only, NOT a substitute for creating the
   row. **Never set `Status=Sorted` for an item whose destination row was not created and
   verified.** If the write failed, leave the row `New` and report the error.
5. **External action → enqueue an Outbox row** (Handler per taxonomy) instead of acting
   directly.

On a transient server error during a write, see `references/recovery.md` §4 (retry once).

## Step 4 — archive filed rows (keep the live Inbox small)

Move every successfully filed/closed row out of the live Inbox so the next read stays cheap.

1. Collect the `page_id`s to move out this run:
   a. every row set `Status=Sorted` by Steps 2–3;
   b. the **expense** rows from Step 1.3 — first set each `Status=Sorted` and
      `Target="expense → pending Finance"` (`Type=expense` preserved), so a future Finance
      consumer can find them. If the registry later gains a dedicated holding base, move them
      there instead — destination is registry-driven;
   c. any already-`Sorted` leftovers surfaced by the recovery reconciliation — archive them
      now, never re-file.
2. `notion-move-pages` them all in **one batched call** (≤100 ids) into Inbox Archive
   (same schema as Inbox, so `Note`/`Status`/`Type`/`Target` are preserved). Record the move
   outcome for the next run's leftover guard — see `references/recovery.md` §3.
3. This is a **move, not a delete** — the rows live on in Inbox Archive as the permanent
   intake record. **Never delete an Inbox row.**

## Step 5 — re-read loop (drain until empty)

The 25-row / no-pagination cap means one pass never guarantees the whole backlog was seen.
So the run is a loop:

```
loop:
  read a batch (Step 1.2) → classify & file (Steps 2–3) → archive the batch (Step 4)
  re-read the live Inbox (Step 1.2)
  if the fresh read finds zero Status=New rows → exit
  else → repeat
```

- **After every Step 4, unconditionally re-read** (Step 1.2) before declaring done. Do not
  branch on "was the batch full?" — just re-read. A batch of exactly 25 means the cap was hit
  and there is almost certainly more behind it; a batch under 25 is also not, by itself, a
  stop signal.
- **The only exit is a fresh read with zero `Status=New` rows.** (Expense rows are moved out
  in Step 4, so an all-expense remainder also reaches zero.)
- **Emit no report between passes** — intermediate or partial summaries are forbidden. The run
  produces exactly **one** report, after the loop fully drains.
- **Safeguard:** cap at 10 passes (≈250 rows). If the live `New` count is not strictly
  shrinking pass over pass, stop and report a likely failed archive move — never spin.

## Rules

- Resolve every base ID from `bases.local.json`. No invented names, no hardcoded IDs.
- Classify with `routing.md` first; only deliberate on notes that match no rule.
- Read field names, formats, select values, and Area URLs from `schema.cache.json` — never
  `notion-fetch` a schema or the Areas DB to rediscover them. Self-heal once if stale
  (`references/recovery.md` §1).
- Never call `query_data_sources` or `query_database_view` (paid). Read via `notion-search`
  and classify from the returned `Note` title — no per-page confirmation fetch.
- When marking an Inbox row `Sorted`, always write its decided `Type` back — never blank.
- Every Tasks row gets a `Type` (`Work`/`Personal`); every `task`/`reminder` gets a `Do date`.
- On any base with an Area relation, always set `Area`; fall back to **Other**.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- reference ≠ action. Doubt → Not Recognized.
- **Never delete records** — only change Status and **move** filed rows to Inbox Archive.

## Output (run report)

Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized, moved to Inbox Archive. Flag any
failed write or failed archive move, and report how many passes ran and the final live-Inbox
size.
