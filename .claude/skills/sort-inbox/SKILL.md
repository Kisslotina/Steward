---
name: sort-inbox
description: Classifies and files raw Inbox records into the EXISTING typed Notion bases, applies completion notes that close existing items, and enqueues outbound work to the Outbox. Use after capturing notes, or whenever the user asks to sort/process the Inbox. Runs only after sweep-daily-notes. Reads Inbox rows with Status=New, decides new-vs-completion, routes to a real base, sets Status=Sorted and Target.
---

# Sort Inbox

Processes every Inbox record with `Status=New`. Runs ONLY after `sweep-daily-notes` completes.

## Step 0 — load the base registry (no live Notion list query)
1. Read the local registry **`bases.local.json`** (base name → data-source ID), written by
   `bootstrap-notion`. This is the source of truth for which bases exist and their IDs.
2. Read `.claude/rules/taxonomy.md` (types + routing) and `conventions.md` (names).
3. Route only to bases present in the registry. **Never invent, rename, or abbreviate a base**
   (no "Reflections DB", "Shopping DB", "Content Ideas"). If a type's destination base is not in the
   registry → that item goes to **Not Recognized** (reason "no destination base: <type>").
4. If `bases.local.json` is missing, stop and tell the user to run `bootstrap-notion` first
   (do a single discovery query only as a last resort, then offer to save the registry).
5. Resolve every ID used downstream from this registry only — **no hardcoded IDs anywhere**:
   `Inbox` (the source), and the targets `Tasks`, `Goals`, `Ideas`, `Knowledge`,
   `Reviews`, `Not Recognized`, `Outbox`. A base whose key is absent from the registry is treated
   as "does not exist" → items for it go to **Not Recognized**.

## Step 1 — fetch ALL Inbox rows with Status=New (content-independent)
Goal: load EVERY Inbox row whose `Status=New`, regardless of whether the page body is blank or the
title is in a non-English language. **Never rely on a search query string matching page content.**

This Notion MCP has **no native property-filter query**. `notion-search` over a `data_source_url`
performs a *semantic / text search over page content*, not a property filter. A query like
`query:"New"` can return **0 results even when New rows exist**, because `Status=New` is a property
value (not indexed as page text) and the pages are blank, so the word "New" is attached to nothing.
Do the following instead.

1. **Resolve the Inbox data source.** Take the `"Inbox"` data-source ID from `bases.local.json`
   (Step 0) and use it as `collection://<inbox_id>`. Never hardcode it.

2. **Confirm the schema once.** `notion-fetch` with `id: "collection://<inbox_id>"` to read the data
   source schema. Record the exact name of the Status property and the exact spelling/casing of its
   `New` option — the client-side filter in 1.4 must match it exactly.

3. **Enumerate ALL rows (broad-query fallback, no content semantics).** Because there is no property
   filter, force the search to return the whole data source rather than ranking on content:
   - Call `notion-search` with:
     `data_source_url: "collection://<inbox_id>"`, `page_size: 25` (the max),
     `max_highlight_length: 0`, and a **broad anchor query that the data source itself matches**,
     i.e. the base name: `query: "Inbox"`. The broad anchor (not a content word) is what makes the
     tool return the full datasource — this is the call that recovered all rows when `query:"New"`
     returned nothing.
   - **Guard against a truncated ranked return:** repeat the call with a few more broad anchors
     (`query:"note"`, `query:"task"`, `query:"status"`) and take the **UNION of all returned
     `page_id`s, de-duplicated**. Stop once two consecutive anchors add no new IDs.
   - **Guard against large inboxes:** if any single search returns the full 25 results, keep varying
     the anchor until no new IDs appear, so rows beyond the first page are not silently dropped.

4. **Filter `Status=New` client-side (the reliable filter).** For each unique `page_id`,
   `notion-fetch` the page and read its properties (`Note`, `Status`, `Type`, `Target`). Keep only
   rows whose `Status` equals exactly the `New` option from 1.2; drop `Sorted` and everything else.
   This per-page property check — not the search query — is what makes detection independent of
   search semantics, blank pages, and title language.

5. **Result + sanity check.** The kept `Status=New` pages (with `page_id`s and properties) are the
   work list for Steps 2–3. If the broad searches returned rows but none were `New`, report
   "nothing to sort". If the broad searches returned **zero rows** for an Inbox that should not be
   empty, report a **read failure** — do not silently treat the Inbox as empty.

## Step 2 — completion vs new
If a note marks an EXISTING item done (cues: "closed", "finished", "done", names a known task/goal):
- one confident match → set it Done; write to the audit log; enqueue an Outbox
  `{Type:notify, Handler:Concierge Gateway, Status:Queued}`; then update the Inbox row:
  set `Type=review` (or the closest matching type), `Status=Sorted`, and `Target="closed: …"`.
- no/ambiguous match → Not Recognized. Never close on a weak match.

## Step 3 — classify & FILE new items (actually write the row)
1. Assign a `Type` from the taxonomy. Distinguish `goal` (a committed, measurable outcome with a
   horizon) from `idea` (an unstarted seed — startup, product, or content). A startup/business/
   product concept is an `idea`, never a `goal`.
2. Map it to a base in the registry (Step 0). If none → Not Recognized.
   For `idea` → **Ideas**: also set the `Ideas.Type` subtype on the new row —
   `Content` (post/social/blog idea), `Startup` (business/venture/product concept), or `Other`.
   The `Drafted/Posted` statuses apply mainly to `Type=Content`; leave blank otherwise.
3. **Create the actual page/row** in that base's data source. Pass the destination base's
   **`data_source_id` from `bases.local.json`** as the parent, i.e.
   `parent: { type: "data_source_id", data_source_id: "<id-from-registry>" }`
   (equivalently the base's `collection://<id>`). **Do NOT use a database page URL or a
   `database_id`** — these bases are addressed by data-source ID, keyed by name in the registry.
   Use the property names from the data source schema (fetch the collection if unsure).
   For `review`: find/create the current period row in Reviews and append the text to the matching
   area column. Do not just describe the write — perform it.
   **Area (bases that have an Area relation — Goals, Ideas, Knowledge, Projects):** always set `Area`.
   Match the note to a canonical Area (Career · Health · Sport · Money · Family · Content · Other) by
   meaning, using only an exact, obvious match; if there is no clear match, fall back to **Other**.
   Never leave `Area` empty. Resolve the Area row's page ID by fetching the Areas DB
   (`collection://<Areas DB id>` from the registry) once and mapping its row titles → page IDs, then
   set the relation to that page (match on the Area name, ignoring any emoji prefix in the title).
4. **Verify** the new row exists (it returns a URL/ID), THEN update the source Inbox row: set its
   `Type` to the Type you assigned in 3.1, set `Status=Sorted`, and set
   `Target="<RealBaseName>/<title>"`. Writing `Type` back is REQUIRED — an Inbox row left with a
   blank `Type` means the sort never recorded what it decided. `Target` is an audit label only — it is
   NOT a substitute for creating the row. NEVER set `Status=Sorted` for an item whose destination row
   was not created and verified. If the write failed, leave Inbox `New` and report the error — never
   report success for a write that did not happen.
5. External action → enqueue an Outbox row (Handler per taxonomy) instead of acting directly.

## Rules
- Resolve every base ID from `bases.local.json`. No invented names, no hardcoded IDs.
- Filter `Status=New` client-side via per-page `notion-fetch`; never trust a search query string to
  filter by property.
- When marking an Inbox row `Sorted`, always write its decided `Type` back — never leave it blank.
- On any base with an Area relation, always set `Area`; fall back to **Other** when no clear match.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- Never delete records — only change Status. The sort routine never executes outbound work.
- reference ≠ action. Doubt → Not Recognized.

## Output (run report)
Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized. Flag any failed write.
