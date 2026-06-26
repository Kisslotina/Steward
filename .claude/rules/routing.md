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
| 1 | completion | `done`, `closed`, `finished`, `completed` / `сделал`, `закрыл`, `завершил`, `выполнил`, `готово` | not a new record — run the completion-match path (taxonomy.md) |
| 2 | event | `meeting`, `call`, `appointment`, `at 3pm`, clock time / `встреча`, `созвон`, `записан`, `в 15:00`, `завтра в` | -> **Outbox** (Type=calendar, Handler=Steward (MCP)) |
| 3 | review | `review`, `reflection`, `weekly recap`, `looking back` / `итоги`, `ретро`, `рефлексия`, `обзор недели` | -> **Reviews** (append to matching area column) |
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

## Ideas subtype (`Ideas.Type`) — when Type=idea

| Subtype | Cues |
|---|---|
| Content | `video`, `post`, `blog`, `youtube`, `reel`, `article` / `видео`, `пост`, `ролик`, `статья` |
| Startup | `startup`, `business`, `product`, `app`, `saas`, `venture` / `стартап`, `бизнес`, `продукт`, `приложение`, `проект про` |
| Other | anything else |

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
