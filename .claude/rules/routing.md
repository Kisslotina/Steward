# Routing table — deterministic first pass

A lookup table the sort phase applies **before** any model reasoning. Most captures match a rule
here and are classified by table lookup — no deliberation, no extra tokens, identical result every
run. Only a note that matches **no** rule falls through to model judgement (then `unsure` → **Not
Recognized** if still unclear).

This is the fast path for `sort-inbox`. It does not replace `taxonomy.md` (the semantic source of
truth for what each type means and where it goes); it operationalises it. If the two ever disagree,
`taxonomy.md` wins and this table must be corrected to match.

How to use it (sort phase):
1. Lowercase the note. Test the cue groups **top to bottom**; the **first** group that matches sets
   the `Type`. Order matters — completion and event cues are tested before generic task cues.
2. Apply the Area cues (independent of Type) for bases that carry an `Area`.
3. If no Type group matches, do **not** guess from this table — hand the note to model judgement per
   `taxonomy.md`; unresolved → `unsure`.

Cues are substrings/stems, case-insensitive, matched in English and Russian (the capture language).

## Type cues (first match wins, tested in this order)

| # | Type | Cue stems (EN / RU) | Then set |
|---|---|---|---|
| 1 | completion | `done`, `closed`, `finished`, `completed` / `сделал`, `закрыл`, `завершил`, `выполнил`, `готово` | not a new record — route to **Not Recognized** ("completion note — close manually"); no auto-match/close (taxonomy.md) |
| 2 | event | a meeting/call cue **with an explicit time** — `meeting`, `call`, `appointment` **+ a clock time** (`at 3pm`, `15:00`), `завтра в 10` / `встреча`, `созвон`, `записан` **+ время** (`в 15:00`), `завтра в` | -> **Outbox** (Type=calendar, Handler=Steward (MCP)). **No time present → not an event; falls to row 10 (task).** |
| 3 | review | `review`, `reflection`, `weekly recap`, `looking back` / `итоги`, `ретро`, `рефлексия`, `обзор недели` | -> **Reviews** (ONE new row: `Type` = the area per the Review-type cues below, `Note` = the text, `Date` = today) |
| 4 | reminder | `remind`, `remember to`, `don't forget` / `напомни`, `напоминание`, `не забыть` | -> **Tasks** (Tag=Reminder + date) |
| 5 | goal | `goal`, `by end of year`, `this quarter`, measurable target + horizon / `цель`, `к концу года`, `за месяц`, `похудеть до`, `накопить` | -> **Goals** (Horizon, Area, Target date) |
| 6 | idea | `idea:`, `what if`, `concept`, `startup`, `video about`, `post about`, `blog` / `идея`, `а что если`, `стартап`, `снять видео`, `пост про`, `проект про` | -> **Ideas** (set subtype, see below) |
| 7 | reference | `note:`, `fyi`, `fact`, `allergic to`, `password`, a stored fact with no action / `факт`, `на заметку`, `аллергия на`, `номер`, `пароль` | -> **Knowledge** (no action) |
| 8 | expense | `spent`, `paid`, `$`, currency amount / `потратил`, `оплатил`, `купил за`, `руб`, `грн` | no Finance base yet — set Type=expense, then **move out to Inbox Archive** marked `Target=expense → pending Finance` (sort-inbox Step 4); note in report. Do NOT leave it `New`. |
| 9 | task (shopping) | `buy`, `pick up`, `order`, `groceries` / `купить`, `закупить`, `заказать`, `нужны` (a thing to acquire) | -> **Tasks** (Tag=Shopping) |
| 10 | task (default) | any remaining imperative/action — `do`, `send`, `fix`, `write`, `call X`, `book` / `сделать`, `отправить`, `написать`, `починить`, `записаться` | -> **Tasks** (Tag=From Daily if swept from Today) |

Notes:
- Rows 1-9 are **specific**; row 10 is the **catch-all action**. A note reaches row 10 only if no
  earlier group matched, so "buy milk" is caught at row 9, not 10.
- `goal` vs `idea` (rows 5-6): a **goal** is a committed, measurable outcome with a horizon
  ("save 5k this year"). A **startup/product/content seed** is an **idea**, never a goal — even
  phrased ambitiously. When in doubt between the two, prefer `idea`.
- A note matching **no** row (1-10) is **not** a task by default — send it to model judgement, then
  `unsure` -> **Not Recognized**. Never let the catch-all swallow genuinely unclear input.
- `event` (row 2) requires a **time/clock reference**. A bare `call`/`meeting`/`созвон` with **no
  time** ("Call Dominik about ZUS") is **not** an event — it falls through to a **task** (row 10).
  Only enqueue to Outbox (calendar) when a time is present.

## Note blocks (multi-line captures) — handled **before** the cue table

A capture whose `Note` starts with **`🗒 `** is a multi-line note block swept whole from Today's Daily
notes (a meeting note, a list — blocks are separated by `---` dividers on the Today page). This is a
**structural marker** set at sweep time, not a content cue, so it **takes precedence over rows 1–10**:
`sort-inbox` files it as **one** Tasks row (`Do date = tomorrow`, `Tag = Triage`, highest `Priority`,
`Type` = Work/Personal by cue) and does **not** run the cue table over its contents — so a long note
is never fragmented into many mis-classified rows. The full block text is preserved on the archived
Inbox row. See `sort-inbox` Step 3 (note-block short-circuit).

## Ideas subtype (`Ideas.Type`) — when Type=idea

| Subtype | Cues |
|---|---|
| Content | `video`, `post`, `blog`, `youtube`, `reel`, `article` / `видео`, `пост`, `ролик`, `статья` |
| Startup | `startup`, `business`, `product`, `app`, `saas`, `venture` / `стартап`, `бизнес`, `продукт`, `приложение`, `проект про` |
| Other | anything else |

## Review type (`Reviews.Type`) — when Type=review

A review note becomes **ONE new Reviews row** (`Note` = the text, `Date` = today). Pick `Type` —
the area the reflection is about — by the cues below, **first match wins**, tested top to bottom.
These options match the live Reviews `Type` select exactly; no clear match -> **Other** (never
empty). Many notes about the same area over a month are just many rows with the same `Type` —
there is no per-period row and no merging.

| Type | Cues (EN / RU) |
|---|---|
| Health | `health`, `doctor`, `sleep`, `diet`, `fasting`, `allergy` / `здоровье`, `врач`, `сон`, `питание`, `голодание`, `самочувствие` |
| Sport | `run`, `gym`, `training`, `marathon`, `workout` / `бег`, `зал`, `тренировка`, `спорт`, `пробежка` |
| Money | `money`, `budget`, `invest`, `savings`, `salary`, `spending` / `деньги`, `бюджет`, `инвест`, `накоплен`, `зарплата`, `траты` |
| Family | `family`, `wife`, `kids`, `parents`, `home` / `семья`, `жена`, `дети`, `родители`, `дом` |
| Career | `career`, `interview`, `promotion`, `cv`, `resume`, `job search` / `карьера`, `собеседование`, `повышение`, `резюме`, `поиск работы` |
| Work | `work`, `project`, `task`, `deadline`, `client`, `meeting`, `standup`, `content`, `blog`, `channel` / `работа`, `проект`, `задача`, `дедлайн`, `клиент`, `совещание`, `контент`, `блог`, `канал` |
| Other | no clear match above |

> `Career` vs `Work`: **Career** is professional growth — interviews, promotions, job hunting,
> the resume. **Work** is day-to-day execution — projects, tasks, deadlines, meetings, clients,
> content output. Career is tested first; a generic work cue falls to **Work**.

## Area cues (set `Area` on Goals / Ideas / Knowledge / Projects)

Canonical Areas: `Career · Health · Sport · Money · Family · Content · Other`. First match wins;
no match -> **Other** (never empty).

| Area | Cues (EN / RU) |
|---|---|
| Health | `health`, `doctor`, `sleep`, `diet`, `fasting`, `allergy` / `здоровье`, `врач`, `сон`, `питание`, `голодание`, `аллергия` |
| Sport | `run`, `gym`, `training`, `marathon`, `workout` / `бег`, `зал`, `тренировка`, `кроссовки`, `спорт` |
| Career | `job`, `career`, `interview`, `promotion`, `cv`, `resume` / `работа`, `карьера`, `собеседование`, `резюме` |
| Money | `money`, `budget`, `invest`, `savings`, `salary` / `деньги`, `бюджет`, `инвест`, `накоплен`, `зарплата` |
| Family | `family`, `wife`, `kids`, `parents`, `home` / `семья`, `жена`, `дети`, `родители`, `дом` |
| Content | `content`, `youtube`, `blog`, `audience`, `post`, `channel` / `контент`, `блог`, `канал`, `аудитория` |
| Other | no clear match above |
