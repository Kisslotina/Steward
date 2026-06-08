---
name: sweep-daily-notes
description: Pulls free-text Daily notes from the Today page into the Inbox before sorting. Use at the start of an inbox sort run, or when the user asks to collect/ingest Today's notes. Appends each new line to Inbox as Status=New and marks the source line done so it is not re-imported. Always runs before sort-inbox.
---

# Sweep Daily Notes

Moves free text from the **Today** page into the **Inbox** so `sort-inbox` can process it. Runs
FIRST in the sort routine.

## When to use
- At the start of an inbox sort run (before sort-inbox).
- The user asks to collect/ingest Today's notes.

## Step 0 — load the base registry
Read `bases.local.json` (base name → data-source ID) from the project root to get the **Inbox** ID.
If it is missing, stop and tell the user to run `bootstrap-notion` first — do not guess or hardcode
the Inbox ID.

## Steps
1. Read the Today page; take Daily-notes lines WITHOUT the "✅ done" mark.
2. For each new line, create an Inbox record with `Status=New` (the line text → Note).
3. Immediately mark the source line on Today as "✅ done".

## Idempotency (critical)
The "✅" mark is the ONLY defence against duplicates. Never move a line without marking it. On a
re-run, lines with "✅" are skipped. If an Inbox record was created but the mark could not be set,
do NOT continue to the next line — report the error (otherwise the next run duplicates it).

## After
Hand off to `sort-inbox` (it starts only after sweep fully completes).

## Output
How many lines moved to Inbox; how many skipped as already done.
