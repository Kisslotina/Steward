# Steward

**An AI second brain with an action layer — Notion + Claude (+ code).**

Voice notes captured on the phone land in a Notion **Inbox**; a routine sorts them into a typed
Notion structure (PARA); outbound work (calendar, notifications) is queued in an Outbox — connector
actions (Calendar/Docs/Sheets) run in a Claude executor routine via MCP, and a separate gateway
service handles Telegram.

> Status: research/prototype. Built and documented in the open so others can fork and follow.

## How it works (high level)

![Architecture](docs/diagrams/architecture.svg)


```
iPhone (Hey Siri "Inbox" / Action Button / Back Tap)
        │  dictate → text on-device
        ▼
Notion Inbox [Status: New]  ──►  Steward routine  ──►  typed Notion bases (PARA)
                                  (roll-day)            + Not Recognized (dead-letter)
                                        │  enqueue outbound work (with Handler)
                                        ▼
                                  Outbox queue (Notion)
                                   ├─ Handler=Steward ─► executor routine (MCP) ─► Google Calendar/Docs
                                   └─ Handler=Gateway ─► Concierge Gateway (Node) ─► Telegram
```

Full design, diagrams and data model: [`docs/architecture.md`](docs/architecture.md).

## Repository layout

```
AGENTS.md            # vendor-neutral agent instructions (the operating contract)
CLAUDE.md            # imports AGENTS.md + Claude-specific notes
.claude/
  rules/             # always-loaded rules (taxonomy, conventions)
  skills/            # on-demand procedures (bootstrap-notion, roll-day, sort-inbox)
docs/                # architecture, data model, diagrams, iOS shortcut, profile template
.env.example         # required secrets (names only)
```

> **Concierge Gateway** — the Node.js service scoped to **Telegram** (consumes Outbox rows with
> `Handler=Gateway`, posts logs/notifications; receives incoming taps later) — lives in a **separate
> repository**, not here. Connector actions (Calendar/Docs/Sheets) run in the Claude executor
> routine via MCP, not in this service. This repo is the agent's brain (instructions + skills + docs).

> Before first run: copy `docs/profile.template.md` → `docs/profile.md` and fill in your personal
> context. `profile.md` is git-ignored (stays local).

## Two runtimes

This project runs in two phases with different "where the brain lives". Both read this folder
directly — clone the repo, then point the tool at the folder:

1. **Cowork / Claude Desktop (prototype).** Connect the cloned folder as the project. `CLAUDE.md`
   (with its `@AGENTS.md` import) and `.claude/rules/` load automatically at session start.
   Notion is connected via the connector (OAuth) in the UI.
2. **Claude Code / Agent SDK (production, e.g. on a VPS/Raspberry Pi).** Same filesystem
   conventions; `CLAUDE.md`, `.claude/rules/` and `.claude/skills/` are picked up automatically when
   Claude Code runs in this directory.

> **Skills caveat (important).** Claude Code auto-discovers `.claude/skills/` from the project
> folder. **Cowork / claude.ai does *not*** — folder skills are visible as files but are not
> registered as runnable skills, so `bootstrap-notion` etc. won't appear in the skills list. Two ways
> to use them in Cowork:
> 1. **Just ask** — the assistant can read and follow a skill file directly:
>    *"Read `.claude/skills/bootstrap-notion/SKILL.md` and follow it."* (It has the folder + connectors.)
> 2. **Install it** — zip the skill and upload via Settings → Features → Skills:
>    `cd .claude/skills && zip -r bootstrap-notion.zip bootstrap-notion` (repeat per skill).
>
> Verify what's registered by asking *"what skills are available?"*.

## Reuse / quick start

Set the whole project up from scratch by following these five steps in order. There are **two
separate Notion connections** and it's important not to confuse them:

| Connection | Who uses it | Auth | Purpose |
|---|---|---|---|
| **Notion connector** | Claude (Cowork / Claude Code) | OAuth, in the Claude UI | Lets Claude create and manage the bases, run `bootstrap-notion`, and run the daily `roll-day` routine (sweep + sort the Inbox). |
| **Internal integration** (`Steward Capture`) | the iOS shortcut on your phone | a secret token (`ntn_...`) | Lets the phone write captured notes straight into the Inbox via the Notion API. |

You need **both**: the connector is how Claude reaches Notion; the integration token is how your
phone reaches Notion. Setting up one does not set up the other.

### 1. Get the code into Claude
Clone this repo, then point a tool at the folder:
- **Cowork / Claude Desktop:** connect the cloned folder as the project (the folder picker in the UI).
- **Claude Code / Agent SDK:** run Claude Code from inside the cloned directory.

Either way, `CLAUDE.md` (with its `@AGENTS.md` import) and `.claude/rules/` load automatically at
session start. (Skills may need verification in Cowork — see the caveat above.)

### 2. Connect Notion to Claude and create the bases
1. In the Claude UI, add the **Notion connector** and authorize it (OAuth). Grant it access to the
   workspace (or page) where Steward will live. This is the connection Claude uses — **not** the
   token from step 4.
2. Run the **`bootstrap-notion`** skill to create all bases, relations and views automatically (or
   create them manually per [`docs/data-model.md`](docs/data-model.md)). Tell it to put the
   **Steward** page in your **Private** space or an **existing** teamspace — don't create a new
   teamspace (Notion auto-adds a "Teamspace Home" page to new teamspaces).
3. Bootstrap also writes `bases.local.json` (base name → data-source ID; git-ignored) — see
   [`bases.local.example.json`](bases.local.example.json). The sort routine reads this registry
   instead of querying Notion each run.

### 3. Fill in your profile
Copy `docs/profile.template.md` → `docs/profile.md` and fill in your personal context. It's
git-ignored and stays local.

### 4. Create the Notion integration token (for the phone)
This is the second, separate connection — the phone can't use Claude's OAuth connector, so it needs
its own token. Full walkthrough with screenshots-worth of detail is in
[`docs/ios-shortcut-setup.md`](docs/ios-shortcut-setup.md); the essentials:
1. Open https://www.notion.so/my-integrations → **New integration** → Type **Internal**, name it
   `Steward Capture`, capabilities **Read + Insert + Update**. Save and copy the **Internal
   Integration Secret** (starts with `ntn_...`) — this is the *token* you'll paste into the phone.
2. Open the **Inbox** database in Notion → **•••** → **Connections** → add `Steward Capture`.
   Without this the shortcut returns 404. Give the integration access to the **Inbox only**.
3. Open the Inbox database as a full page and copy its URL — the 32-char string between the last `/`
   and `?` is the **Database ID** (the *Inbox table token*). You'll paste this into the shortcut too.

### 5. Build the iOS capture shortcut
Follow [`docs/ios-shortcut-setup.md`](docs/ios-shortcut-setup.md) (Part 2). In short: create a
single-word shortcut named `Inbox` with **Dictate Text → Text (JSON body) → Get Contents of URL
(POST to `https://api.notion.com/v1/pages`)**, then where the two secrets go:
- **Database ID** → inside the JSON body, replace `YOUR_DATABASE_ID` in
  `"parent": { "database_id": "..." }`. This must be the ID of the **Inbox** database specifically
  (the one from step 4.3) — not any other base. Capture writes only to Inbox; the sort routine fans
  it out to the other bases later.
- **Integration token** → in the Get-Contents-of-URL **headers**, set
  `Authorization = Bearer ntn_YOUR_TOKEN` (mind the space after `Bearer`).

Wire the shortcut to **Hey Siri "Inbox"**, the **Action Button**, and **Back Tap**. The token lives
in the shortcut in plain text, so keep the integration scoped to the Inbox only and don't share the
shortcut.

> `.env` is optional — see [`.env.example`](.env.example). Most secrets live elsewhere (the phone
> shortcut and the Concierge Gateway repo), not here. Connectors, secrets and the Notion bases are
> per-user and set up by whoever forks the repo.

## Scheduling the daily run

A scheduled task should **reference the skills + `bases.local.json`**, never hardcode Notion IDs
(IDs drift between bootstraps and the run will fail). Recommended task prompt:

```
Run the Steward daily routine. Read bases.local.json for data-source IDs and the Today page ID
(never hardcode IDs; if missing, locate the bases under the "Steward" page and save
bases.local.json). Then follow .claude/skills/roll-day/SKILL.md and .claude/rules/* — it runs
close-out → sweep → sort → carry-forward → refresh, calling sort-inbox for classification. For
each New Inbox row: classify, CREATE the row in the destination base, VERIFY it exists, and ONLY
THEN set Status=Sorted + Target. Target is an audit label, not a substitute for creating the row —
never mark Sorted without a verified destination row. Report per entry and totals.
```

> Why this matters: setting `Status=Sorted` + a `Target` label in Inbox is **not** filing. The
> routine must create the actual row in the destination base and verify it before marking Sorted.

## License

MIT — see [`LICENSE`](LICENSE).
