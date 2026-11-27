# Personal Assistant — Project State

> **Next chat name: `Personal Assistant ver 3`**
> Porting rule: when moved to a new chat, name it "Personal Assistant ver X" (X = this number + 1) and read this file first. Cost rule: cheapest capable model, compact often, ask the user to move chats when context grows.

**State version:** 5 (2026-11-27)

## Goal
An agent that tells the user their outstanding tasks (pay X bill, 30 comics unread, 4 shows unwatched, chores due, low stock, trips to prepare), pings them through the day, and lets them mark items done or snoozed with one tap.

## Scope (user decision)
In: ComicsTracker, TVWorld, FinancialPlanner, VacationGuru, ChoreWheel, MCC-Inventory (repos under `marvyeo/`, data in Google Drive).
LifePlan = landing page / gateway; aggregates all apps' `upcoming[]`.
Out (side projects): NovelReader, FoodGuide.

## Notification behaviour (user decision, 2026-11-27)
- Push notification at the start of the day with outstanding items.
- Reminder pings around lunch, dinner and just before bed, asking whether outstanding items are done.
- Easy way to indicate done / not done / snooze.
- If an item is snoozed past today, it is NOT re-pinged until its snooze window ends.
- Proposed times (confirm; assume Asia/Singapore): 08:00 morning, 12:30 lunch, 19:00 dinner, 22:00 bed.

## Data sources (Drive file IDs)
| App | File | ID | Notes |
|---|---|---|---|
| ComicsTracker | comicstracker-data.json | 115DodCFpgAg0Ho1JlNdl-eKnBbPmRQp- | 324KB, modified 2026-11-27 |
| TVWorld | tvworld-data.json | 1o-G1i3htPYlrHQfklfNE2DQWxVFrJt9J | 20MB, modified 2026-11-26. Code only, never in a prompt |
| FinancialPlanner | financialplanner-data.json | 1i4IhCcDsL2emA6qQkvuMDZpNG8ZL9hFE | 2MB, modified 2026-11-22 |
| VacationGuru | vacationguru-data.json | 1xBNjBZr7-kMV1pGgOvLYpseWrIjRJMmL | 67KB, modified 2026-11-19 |
| ChoreWheel | chorewheel-data.json | 10x_YAVLwLhwE-CxeMcg2Zk3fLx-LteDi | 24KB, **last modified 2026-10-24 (may be stale)** |
| MCC-Inventory | mcc-inventory-items.json | 1qTGUshdGVFfJ3cpcYYLthDSaUyPiyBZ- | 160KB, **last modified 2026-10-18 (may be stale)**; app moved to a new OAuth client on 10-23, so a newer copy may exist (verify) |

## Schemas / "outstanding" definitions
- **Shared contract:** each app writes `upcoming[]` at its JSON root: `{label, date, type, app, priority, url}`. Types: episode, chore, trip, due-date, bill-due, low-stock, cap-alert, rules-update. Only refreshes when the app saves, so can be stale (LifePlan warns after 7 days).
- **Comics:** `issues[]` with `read=false`; `purchased=true` means owned and unread, else "want". Released = `releaseDate <= today`.
- **TV:** `series[].seasons[].episodes[]` with `watched=false` and `air_date <= today`, for status Watching.
- **Chores:** `chores[]`; next due = `snoozedUntil`, else `lastDone` + `freq {unit, every}`, else `firstDue`. Overdue if before today. Assignee marvin / caroline / either. (Note: the app has its own snooze field, `snoozedUntil`.)
- **FinancialPlanner bills:** `profiles.{marvin,wife,joint,cobae}.bills[]` and `.subscriptions[]`, each with `name, amount, dueDate, status` (`paid` = done). Bill is outstanding if `dueDate` set and `status != 'paid'`; overdue if before today. App itself emits type `bill-due`, `cap-alert` (card caps near limit), `rules-update`. Also non-bill chores: statements to upload (Month-end check coverage grid) and owed-to-me balances. `billHasAction(b)` in `index.html` filters which bills count.
- **MCC:** file stores `{items, upcoming}`; low-stock and expiry items (batch `{qty, expiry}`). Locations file `1idvjvAyK7ow7hxZ9oZdB3F77y7qCGijF`.
- **VacationGuru:** undone checklist items with due dates and upcoming trips feed `upcoming[]` (`trip`, `due-date`).

## Key constraint: where "done/snoozed" is stored
The app JSON files are read-only to the Drive connector (`canEdit:false`), and the agent cannot mark a bill paid in FinancialPlanner. So the assistant keeps its OWN state (item key, status done / snoozed-until, last pinged), separate from the apps. Rules:
- "Done" in the assistant hides the item for that occurrence; it clears when the source data confirms (bill paid, episode watched). If the source still shows it unresolved the next day, it re-appears (flagged "marked done, not updated in app").
- Backlog counts (30 comics, 4 shows) are standing items: use snooze or a daily target instead of done.
- Snooze stores `snoozeUntil`; no pings for that item before then.

## Proposed design
1. Adapters (plain code, no LLM) per app read Drive JSON and return standard items `{id, app, kind, label, count, due, priority, url}`. TVWorld's 20MB stays out of prompts.
2. Assistant state store with a one-tap UI: an Artifact checklist page (Done / Snooze 1d / 3d / 1w buttons per item) backed by the shared artifact database. The PN links to it. The scheduled runs read and write that database.
3. Four scheduled tasks (morning, lunch, dinner, bed). Each run: load state, compute items, drop done and snoozed ones, ping only if something remains. If nothing remains it sends nothing and costs almost nothing. Cheap model for wording.
4. Item IDs are stable (app + record ID + due date) so state survives across days.

## Decisions (2026-11-27)
- Times confirmed: 08:00 / 12:30 / 19:00 / 22:00 Asia/Singapore.
- Input method: tap checklist page. Snooze buttons per item: Later today (returns at next ping slot), Tomorrow, 3 days, 1 week, date picker. Day-or-longer snoozes return at the morning ping of the target day. Snoozed items sit in a collapsed "Snoozed" section with an Unsnooze button. Snoozing a bill past its due date asks for confirmation.
- ChoreWheel and MCC Drive files are stale because the user stopped updating them; the assistant is meant to fix that habit. Treat their `upcoming[]` and dates as unreliable and warn when source data is older than 7 days.
- Repo `marvyeo/PersonalAssistant` approved by user, but `gh repo create` was blocked by the permission check. User must create it (empty, public) or allow it; then attach it with add_repo access "push".

## Next steps
2. Spot-check remaining 5 data files (ChoreWheel done).
3. Build adapters + tests, then the checklist page, then the four scheduled tasks.

## Spot-check findings
- Drive `download_file_content` saves large files to disk (not into context), so inspect with Python, zero token cost. Use this pattern for all adapters.
- ChoreWheel verified: root `{app, version, chores[63], log[32], settings, upcoming[47]}`. Chore = `{id, name, area, freq{unit,every}, assignee(either/marvin), lastDone, firstDue, notes, createdAt}`. Matches schema. All dates stop at 2026-10-24 (stale).
- Still to spot-check: Comics, TV (20MB), FinancialPlanner, VacationGuru, MCC.

## Change log
- 2026-11-27 ver 1: Created. Scope fixed, repos read, `upcoming[]` contract found.
- 2026-11-27 ver 2: Drive IDs for all six apps, FinancialPlanner bill schema, notification rules, state-storage design.
- 2026-11-27 ver 3: Times, input method, snooze behaviour and stale-file handling decided. Repo creation blocked, awaiting user.
- 2026-11-27 ver 4: Chat ver 2 started. Repo now attached (push branch claude/blissful-cannon-5serhn). Starting data spot-checks.
- 2026-11-27 ver 5: ChoreWheel schema verified; download-to-disk pattern noted.
