---
name: sort-inbox
description: Classifies and files raw Inbox records into the EXISTING typed Notion bases, applies completion notes that close existing items, and enqueues outbound work to the Outbox. Use after capturing notes, or whenever the user asks to sort/process the Inbox. Invoked by roll-day after its sweep phase, or run standalone over the Inbox. Reads Inbox rows with Status=New, decides new-vs-completion, routes to a real base, sets Status=Sorted and Target, then moves filed rows to Inbox Archive.
---

# Sort Inbox

Processes every Inbox record with `Status=New`. Invoked by `roll-day` after its sweep phase, or run standalone over the Inbox.

**Built to run fast on a small model.** Classification is a table lookup (`routing.md`), not free
reasoning; schemas are read once and reused; the live Inbox is kept small so reads stay cheap. Follow
the steps in order. Do not call the paid query tools (see Step 1).

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
6. **Cache schemas once.** `notion-fetch` the schema of each destination you will actually write this
   run, plus `Areas DB` once (build a map: Area name → page ID, ignoring emoji prefixes). Reuse these
   for every item — never re-fetch a base schema or the Areas map per item. This, with the routing
   table, is what keeps the sort phase cheap.

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
2. **Enumerate the live Inbox.** `notion-search` with `data_source_url: "collection://<Inbox id>"`,
   `query: "Inbox"` (a broad anchor, not a content word), `page_size: 25`, `max_highlight_length: 0`.
   Union the returned `page_id`s with the hint list. If exactly 25 come back, the Inbox is larger than
   expected (an archive move may have failed) — repeat with anchors `"note"` then `"task"`, union, and
   flag it in the report so the archive can be repaired.
3. **Confirm `Status` per page (the only reliable filter).** `notion-fetch` each unique `page_id` and
   read `Note`, `Status`, `Type`, `Target`. Split into:
   - **to file** — `Status = New` → Steps 2–3.
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

## Rules
- Resolve every base ID from `bases.local.json`. No invented names, no hardcoded IDs.
- **Classify with `routing.md` first**; only deliberate on notes that match no rule.
- **Read each base schema and the Areas map once per run**, never per item.
- **Never call `query_data_sources` or `query_database_view`** — paid (Enterprise / Business). Read
  via `notion-search` + per-page `notion-fetch`; that is why the Inbox is kept small (Step 4).
- When marking an Inbox row `Sorted`, always write its decided `Type` back — never leave it blank.
- On any base with an Area relation, always set `Area`; fall back to **Other** when no clear match.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- **Never delete records** — only change Status and **move** filed rows to Inbox Archive.
- reference ≠ action. Doubt → Not Recognized.

## Output (run report)
Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized, **moved to Inbox Archive**. Flag any
failed write or failed archive move.
