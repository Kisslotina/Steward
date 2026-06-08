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
   (no "Reflections DB", "Shopping DB", "Ideas DB"). If a type's destination base is not in the
   registry → that item goes to **Not Recognized** (reason "no destination base: <type>").
4. If `bases.local.json` is missing, stop and tell the user to run `bootstrap-notion` first
   (do a single discovery query only as a last resort, then offer to save the registry).

## Step 1 — fetch work
Fetch only Inbox records with `Status=New`.

## Step 2 — completion vs new
If a note marks an EXISTING item done (cues: "closed", "finished", "done", names a known task/goal):
- one confident match → set it Done; write to the audit log; enqueue an Outbox
  `{Type:notify, Handler:Concierge Gateway, Status:Queued}`; set Inbox `Target="closed: …"`.
- no/ambiguous match → Not Recognized. Never close on a weak match.

## Step 3 — classify & FILE new items (actually write the row)
1. Assign a `Type` from the taxonomy.
2. Map it to a base in the registry (Step 0). If none → Not Recognized.
3. **Create the actual page/row** in that base's data source (use its ID from the registry) with the
   proper fields. For `review`: find/create the current period row in Reviews and append the text to
   the matching area column. Do not just describe the write — perform it.
4. **Verify** the new row exists (it returns a URL/ID), THEN set Inbox `Status=Sorted` and
   `Target="<RealBaseName>/<title>"`. `Target` is an audit label only — it is NOT a substitute for
   creating the row. NEVER set `Status=Sorted` for an item whose destination row was not created and
   verified. If the write failed, leave Inbox `New` and report the error — never report success for a
   write that did not happen.
5. External action → enqueue an Outbox row (Handler per taxonomy) instead of acting directly.

## Rules
- Only route to bases in `bases.local.json`. No invented names.
- Don't split partially recognized: file whole OR Not Recognized with a reason.
- Never delete records — only change Status. The sort routine never executes outbound work.
- reference ≠ action. Doubt → Not Recognized.

## Output (run report)
Per item: input → Type → REAL base written to (with confirmation) OR Not Recognized (reason).
Totals: filed, closed, enqueued to Outbox, Not Recognized. Flag any failed write.
