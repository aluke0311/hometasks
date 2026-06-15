# Home Tasks — Full Documentation

A personal home-task manager for Alina and Bob. It deals a daily "hand" of
cleaning/household tasks sized to a time budget, drawing from a weighted pool and
inspired by FlyLady's rotating-zone system.

This document is the complete reference for how the app works. For the condensed
working-instructions version, see [CLAUDE.md](CLAUDE.md).

---

## Table of contents

1. [Overview & architecture](#1-overview--architecture)
2. [Task data model](#2-task-data-model)
3. [Core concepts](#3-core-concepts)
4. [The scoring algorithm](#4-the-scoring-algorithm)
5. [Hand dealing](#5-hand-dealing)
6. [The laundry system](#6-the-laundry-system)
7. [Tabs in detail](#7-tabs-in-detail)
8. [Presets catalog](#8-presets-catalog)
9. [State schema](#9-state-schema)
10. [Persistence, migration & backup](#10-persistence-migration--backup)
11. [Special modes & features](#11-special-modes--features)
12. [Edge cases & gotchas](#12-edge-cases--gotchas)
13. [Deployment](#13-deployment)
14. [Glossary](#14-glossary)

---

## 1. Overview & architecture

- **Single file.** The entire app is `index.html` (~3,960 lines): HTML, CSS, and
  vanilla JavaScript with **no build step, no dependencies, no server**.
- **Client-only state.** Everything the user does lives in browser
  `localStorage` under the key `hometasks_v8`. There is no account and no sync.
- **Deploy = push.** The site is GitHub Pages; deploying is just committing and
  pushing `index.html`. Because state is in the browser, deploying never touches
  user data.
- **Mobile-first.** Designed for an iOS home-screen shortcut: safe-area insets,
  a fixed bottom tab bar, dark-mode support via `prefers-color-scheme`, and
  `APP_VERSION` shown in Settings so a reload can be confirmed to have taken.

### Code map (approximate line anchors — re-grep after edits)

| Area | Lines |
|------|-------|
| CSS | 9–265 |
| Body markup (views, nav, modals) | 266–480 |
| `TASKS` array (177 entries) | 483–703 |
| `DEFAULT_MONTHS`, `ZONE_ROOMS` | 711–745 |
| Storage layer (`loadState`/`saveState`/`defaultState`) | 746–840 |
| Core logic (due, season, zone, scoring) | 842–1047 |
| `dealHand` | 1049–1214 |
| Hand/overflow/not-due selectors | 1216–1311 |
| Completion + task actions | 1313–1575 |
| Render: My Hand | 1579–1760 |
| `renderCard` | 1762–1862 |
| "Why this task?" modal | 1866–1902 |
| Sprint tab | 1905–2150 |
| Presets (definitions + render; incl. Weekly Reset & Seasonal Deep Clean) | 2152–3000 |
| Stats tab | 3096–3240 |
| Settings tab | 3242–3360 |
| Vacation / backup / import (`applyImport` ~3475) | 3360–3548 |
| Manage tab + modals | 3549–3860 |
| Misc UI helpers + `init()` | 3862–3962 |

---

## 2. Task data model

Each base task is an object in the `TASKS` array:

```javascript
{ id:'k_stove', name:'Clean stovetop', room:'Kitchen', freq:7, time:8, owner:'alina', cat:false }
```

| Field | Type | Meaning |
|-------|------|---------|
| `id` | string | Unique key. Base ids are mnemonic (`k_` kitchen, `c_` cat, `usb_` upstairs bath, `l_`/`lroom_` laundry, etc.). Custom tasks use `custom_<timestamp>`; one-offs use `oneoff_<timestamp>`. |
| `name` | string | Display name. |
| `room` | string | Location; drives zone matching and sprint grouping. |
| `freq` | number | Target interval **in days** (1 = daily, 7 = weekly, 30 = monthly, 180 = twice a year, 365 = yearly). |
| `time` | number | Estimated minutes. Used for budget filling unless a learned time exists. |
| `owner` | `'alina' \| 'bob' \| 'either'` | Who does it. Alina's hand shows `alina` + `either`; Bob's tasks are tracked but never dealt into Alina's hand. |
| `cat` | boolean (optional) | Cat-care task; gets a small `-0.5` scoring boost. |
| `oneOff` | boolean (optional) | Set only on one-off custom tasks; always "due", removed after completion. |

> Some entries in `TASKS` have `custom_…` ids — these are personal
> customizations that were baked directly into the base array (see commit
> `56ffca8`), so they ship with the app rather than living only in a user's
> `customTasks`.

### Rooms

`Cats`, `Kitchen`, `Laundry Room`, `Bedroom`, `Downstairs`, `Upstairs`,
`Robot`, `Living Room`, `Dining Room`, `DS Bathroom`, `US Bathroom`, `Sunroom`,
`Office`, `Back Room`, `Hall & Stairs`, `Mud Room`, `Back Porch`,
`Whole House`. (Custom tasks may introduce others.) `Whole House`, `Robot`,
`Downstairs`, and `Upstairs` are zone-less — tasks in them don't receive zone bonuses.

### Custom tasks & overrides

- `getAllTasks()` returns base tasks minus `deletedIds`, plus everything in
  `customTasks`. A custom task **overrides** a base task with the same id (the
  base entry is suppressed). This is how editing a base task works: it gets
  copied into `customTasks` and its id is added to `deletedIds`.
- Hiding a base task = adding its id to `deletedIds`.
- Deleting a custom task = removing it from `customTasks`.

---

## 3. Core concepts

### Due / overdue (`daysOverdue`, `isDue`)

- **Daily tasks (`freq === 1`)** are calendar-day based: due whenever the last
  completion was on a previous calendar day. Done today → not due; done
  yesterday → 0 days overdue; done 2 days ago → 1 day overdue.
- **Non-daily tasks** are due when `now ≥ lastCompletion + freq days`.
  `daysOverdue` is the (possibly negative) number of days past the due date.
- **Snoozed** tasks return a negative `daysOverdue` until the snooze expires, so
  they read as not-due.
- `isDue(task)` is `daysOverdue(task) >= 0` (one-offs are always due).

### Seasons (`inSeason`)

- `state.taskMonths[id]` is an array of months (1–12) when a task applies.
- Absent, empty, or all-12 → year-round.
- Out-of-season tasks are **excluded from dealing pools and from starvation
  ticks**, and show a "❄ off-season" badge in Manage/Stats. They can still be
  found via quick-log (you might do them anyway).
- One-time defaults (`DEFAULT_MONTHS`) seed seasonal windows on first run:
  porch tasks Mar–Nov, all window-cleaning tasks Apr–Oct. Guarded by
  `seasonalDefaultsApplied`; any later user edit in Manage wins.

### Zones (FlyLady-style rotating focus)

Five zones, each a set of rooms (`ZONE_ROOMS`):

| Zone | Rooms |
|------|-------|
| 1 | Dining Room, Mud Room, Back Porch |
| 2 | Kitchen |
| 3 | Sunroom, Office, DS Bathroom, Laundry Room, Hall & Stairs |
| 4 | Bedroom, US Bathroom |
| 5 | Living Room, Back Room |

Tasks in the active zone get a `-2` scoring boost.

- **Auto mode** (`zoneMode: 'auto'`, default): the zone follows the day of the
  month via `autoZone()` — `Math.min(ceil(day/7), 5)` → 1st–7th = Zone 1,
  8th–14th = Zone 2, …, 29th+ = Zone 5.
- **Manual mode**: an explicit zone (or "No zone focus" = 0) chosen from the
  My Hand zone selector. `effectiveZone()` returns the auto zone or the manual
  `state.zone` accordingly.

### Tiers (`getTaskTier`)

```javascript
freq <= 7  → 'A'  // Operational (daily/weekly)
freq <= 60 → 'B'  // Maintenance (every 2–8 weeks)
else       → 'C'  // Deep clean (2+ months)
```

Tiers drive only optional UI features: tier-filtered "Give me more", tier-favoring
redeal (`_dealPreferTier`), and the Random Task filter. They do **not** create
separate quotas in `dealHand`.

### Modes

Three pills on My Hand. Switching mode clears and re-deals.

| Mode | Internal | Pool order | Overdue weight |
|------|----------|-----------|----------------|
| 🏠 Keep up (default) | `maintenance` | score | 0.5 |
| 🔄 Catch up | `catchup` | score | 0.8 (overdue surfaces faster) |
| 📋 By freq | `byfreq` | raw frequency, shuffled within equal freq | 0.5 |

---

## 4. The scoring algorithm

`scoreTaskParts(task, zone)` returns the labeled breakdown; `scoreTask` sums it
and adds jitter. **Lower score = higher priority.**

| Component | Formula | Notes |
|-----------|---------|-------|
| Base | `+task.freq` | Lower frequency → lower base → dealt sooner |
| Cat | `-0.5` if `cat` | Tiebreaker |
| Zone | `-2` if room in active zone | |
| Overdue | `-min(overdueRatio · freq · weight, freq · 0.8)` | `weight` = 0.5 (or 0.8 in catch-up); capped at 80% of freq |
| Starvation | `-min(0.3 · count, freq · 0.15)` | `count` ticks +1 per day the task is due but undealt; **cap is critical** — prevents long-interval tasks from dominating |
| Flagged | `-50` | Always floats to the top |
| Jitter | `+(random−0.5)·2` for `freq > 60` | Added only in `scoreTask`; gives deep-clean tasks variety between deals |

The "why this task?" modal renders each non-zero component with a one-line
explanation and the total, computed from the same `scoreTaskParts` so the
explanation can never drift from the real score.

---

## 5. Hand dealing

`dealHand()` builds `state.hand` (an array of task ids) once per calendar day and
caches it. It returns early if `state.handDate === today` and a hand exists.
Re-dealing (`redealHand`) clears `hand`/`handDate` to force a rebuild.

Order of operations:

1. **Daily reset of `removedToday`** if the date rolled over.
2. **Vacation guard.** If `paused`, keep the existing hand and skip dealing +
   starvation entirely.
3. **Tier preference.** Read and clear `_dealPreferTier` (one-shot).
4. **Laundry slot.** Pick at most one "load" task for the day (see §6).
5. **Dailies.** `getDailies()` = due, in-season, non-one-off, `freq === 1`,
   owner alina/either tasks, minus pinned, minus suppressed laundry-process steps.
6. **Always-assigned.** `c_fountain` (when due) and the chosen laundry load
   bypass the budget.
7. **Non-daily pool.** All due (or flagged) in-season alina/either non-daily
   tasks, excluding pinned/always-assigned/laundry-load, sorted by score
   (or floated by tier if `_dealPreferTier`).
8. **Budget fill.** `budget` = weekday/weekend setting. Time already spent today
   on non-dailies plus always-assigned time is pre-counted. Then:
   - **Flagged tasks fill first** in every mode. They count against the budget,
     but the **first flagged task is always dealt** even if the budget is spent
     (so flagging guarantees surfacing). Flagged tasks that don't fit fall to
     "Give me more".
   - **Keep up / Catch up:** fill remaining budget in score order.
   - **By freq:** fill in raw-frequency order, shuffling within equal frequency.
   - **Heavy cap:** at most 2 tasks over 15 minutes per hand (`HEAVY_MAX`).
     Always-assigned and flagged heavy tasks count toward this cap.
9. **Carry-overs.** Tasks completed today are kept in the hand (so they stay
   visible as "Completed"); in-progress tasks are always carried; tasks removed
   today are filtered out unless completed today.
10. **Starvation tick.** Once per calendar day: due in-season tasks not in the
    hand get `+1`; tasks in the hand reset to `0`.

### Selectors

- `getHandTasks()` — the hand plus any tasks completed today not already in it.
- `getOverflowTasks()` — due (and flagged-not-due) non-daily tasks not in the
  hand and not removed today; ordered by frequency in By-freq mode, else by
  score. Powers "+ Give me more" and the "Due / overdue" backup list.
- `getNotDueTasks()` — not-due, in-season alina/either tasks not in the hand,
  sorted by next-due date. Powers the "Not due yet" backup list.

---

## 6. The laundry system

Laundry is split into **load** tasks and **process** steps, coordinated so only
one wash happens per day.

- **Load tasks** (`lroom_myclothes`, `lroom_towels`, `lroom_microfiber`,
  `lroom_whites`, `k_towels`) compete for a **single daily slot**. The most
  overdue (lowest score) wins. No new load is chosen if one was already
  completed today, is pinned, or is in progress.
- **Process steps** (`l_start`, `l_dryer`, `l_fold`, `l_put_away`) are daily
  tasks that are **suppressed** unless there's a load today (assigned, pinned,
  in progress, or already completed).
- In the **Sprint** view, the day's chosen load rides next to "Start a load of
  laundry" in the Launch block so you know what to wash.

---

## 7. Tabs in detail

### My Hand (`alina`)

- **Header:** title "Today's Hand", date, Redeal (with A/B/C tier-redeal pills),
  the zone selector, the three mode pills, and "+ one-off task".
- **Stats row:** done/total count, time-left estimate, progress bar.
- **Sections:** Daily Tasks, Other Tasks (the budgeted non-dailies), Completed
  (collapsible), then "+ Give me more (N due)" with A/B/C filters and a Backup
  list (Due/overdue + Not-due-yet).
- **Quick-log:** "+ Log something else I did" expands a search over all tasks;
  selecting one records a completion **without** adding it to the hand or
  counting against budget.
- **Per-card actions** (`renderCard`): tap to complete (undo toast); swap; pin;
  snooze 1/2/3d; "did earlier" (yesterday/2d/3d ago backdate); timer; remove;
  why?; flag; edit last-done date. Badges show overdue/snoozed/in-progress,
  real cadence (`~Nd real`), learned time, owner.

### Sprint (`sprint`)

A guided, room-grouped walkthrough of today's hand:

- Blocks in order: **Launch** (passive/cycle-starting tasks + today's load) →
  **Kitchen** (dishwasher → tidy → counters → rest) → **Cat care** → **Laundry**
  → grouped rooms (Bathrooms, Bedroom, Living spaces, Office, Hall & entries,
  Around the house) → Other.
- Per-block remaining time, "Hide done" toggle, and per-task
  timer / in-progress / remove controls. Tapping a card completes it.

### All Tasks (`manage`)

- Search + owner/room/status filters; sorted by frequency then name.
- Add / edit / delete custom tasks; hide base tasks; pin; flag; +Today;
  edit last-done date inline.
- The editor includes 12 month chips for seasonal windows (all-off = year-round).
- Cards show learned-time hints (`~12m (est 10m, n=5)`) and real-cadence color.

### Presets (`presets`)

See §8.

### Stats (`stats`)

- **This Week:** tasks done (vs previous week), estimated effort, day streak,
  done today.
- **Budget Insight:** compares your real non-daily throughput (median
  minutes/day over the last 28 active days, today and zero-days excluded)
  against the configured budgets and suggests a change you can apply.
- **Activity:** 13-week GitHub-style heatmap (darker = more done that day).
- **Completion Cadence:** every task with 3+ completions, worst-behind first,
  showing `every ~Nd (target Md)` color-coded (red ≥1.5× behind, amber behind,
  green on pace, blue more often than needed). Each row has due badge, +Today,
  flag, "set target Nd" adoption, and tap-name-to-edit.

### Settings (`settings`)

Budget steppers, active-zone reference table, system overview counts, a scoring
explainer, recent-activity log, vacation mode toggle, a reload-app card (shows
`APP_VERSION`), the weekly-backup card, and export/import (copy to clipboard,
restore from a backup file, or paste).

---

## 8. Presets catalog

Presets build a checklist you tick through (completions count app-wide). Each has
**Load all** and **Load due only** buttons that replace today's hand;
`mergeStickyHand()` keeps pinned + in-progress tasks attached.

| Category | Preset | What it is |
|----------|--------|-----------|
| (top) | **Random Task** | Pick a random due task, optionally filtered to tier A/B/C; add to today |
| Routines | **Express Reset** | Quick whole-house pass (kitchen, baths, bedroom, living spaces, cat care) |
| Routines | **Weekly Reset** | The classic weekly/Sunday reset (~2h): sheets, bathroom, kitchen, vacuum + dust living spaces, towels, cat care (`WEEKLY_RESET_SECTIONS`) |
| Routines | **Full Reset** | Complete whole-house clean, ~2–3h (`resetHand`) |
| Guests | **Guest Prep** | Emergency (~15m) / Day / Overnight variants (`guestHand`) |
| Travel | **Going Out of Town** | Leave the house in great shape (`goingOutHand`) |
| Travel | **Return Home** | Back to baseline after travel (fridge, cats, laundry, bedroom) |
| Special | **Before Cleaners** | Declutter + cat care the cleaners won't touch |
| Special | **Recovery Mode** | Phased plan for overwhelm; Phase 1 alone is enough |
| Special | **Post-Illness** | Phase 1 sanitize high-touch surfaces → Phase 2 restore |
| Daily Rituals | **Evening Shutdown** | Dishes, counters, cat water + the single most-overdue tidy room |
| Seasonal | **Seasonal Deep Clean** | Spring-cleaning blitz: snapshots every in-season Tier C task into one room-grouped checklist |
| Rooms | **Per-room Quick / Deep** | Quick or deep preset for each of the 14 rooms (`ROOM_PRESETS`) |

Generated checklists are stored in `presetHands` keyed by type or
`room_<id>_<depth>`, except the three legacy presets with dedicated fields
(`guestHand`, `resetHand`, `goingOutHand`).

**Fixed-section presets** (Express Reset, Weekly Reset, Return Home, Before
Cleaners, Recovery, Post-Illness) define a `*_SECTIONS` constant and register in
`getNewPresetSections()`, so the generic `generateNewPreset` / `loadNewPreset` /
`loadNewPresetDueOnly` / `clearNewPreset` drive them. **Seasonal Deep Clean** is
the one dynamic preset: it has **no** section constant — `generateSeasonalDeepClean()`
computes its id list from `getAllTasks()` (every in-season Tier C task) at generate
time, stores it flat in `presetHands.seasonalDeep`, and `seasonalDeepSections()`
regroups it by room for rendering. It reuses the generic load/clear functions.

> Preset id lists that render via `getAllTasks()` must reference ids that exist
> in `TASKS` — unknown ids silently drop, and `completeTask` returns early for
> them so they can't be checked off. (The obsolete deep-room ids were pruned on
> 2026-06-15.)

---

## 9. State schema

Stored as JSON under `localStorage['hometasks_v8']`. Defaults come from
`defaultState()`; `loadState()` migrates older saves and fills missing fields.

| Field | Type | Purpose |
|-------|------|---------|
| `completions` | `{id: ts}` | Last completion timestamp per task. `loadState`/`applyImport` backfill any base task missing an entry to `now − freq` days (newly added tasks read as due-now, not ~20,000 days overdue) |
| `completionHistory` | `{id: ts[]}` | Up to 100 completion timestamps per task (drives stats/cadence) |
| `starvation` | `{id: count}` | Consecutive days due but not dealt |
| `starvationDate` | string | Last date starvation ticked |
| `hand` / `handDate` | `id[]` / string | Today's dealt hand and its date |
| `budgetWeekday` / `budgetWeekend` | number | Non-daily time budget (default 45 / 90) |
| `zone` / `zoneMode` | number / `'auto'\|'manual'` | Manual zone + whether auto-rotation is on |
| `mode` | string | `maintenance` \| `catchup` \| `byfreq` |
| `pinnedIds` | `id[]` | Pinned tasks (always in hand, survive redeal) |
| `flaggedIds` | `id[]` | Flagged tasks (`-50`, surface first, bypass not-due) |
| `snoozed` | `{id: untilTs}` | Snoozed-until timestamps |
| `inProgress` | `{id: ts}` | Started-but-unfinished tasks; carried into every hand |
| `deletedIds` | `id[]` | Hidden base tasks |
| `customTasks` | `{id: task}` | User tasks + base-task overrides + one-offs |
| `taskMonths` | `{id: [1..12]}` | Seasonal windows |
| `seasonalDefaultsApplied` | bool | Guards one-time `DEFAULT_MONTHS` migration |
| `actualTimes` | `{id: min[]}` | Last 10 logged durations; `taskTime()` uses median of 3+ |
| `guestHand` / `resetHand` / `goingOutHand` | `id[]` \| null | Legacy preset checklists |
| `presetHands` | `{key: id[] \| null}` | All other preset checklists |
| `removedToday` / `removedTodayDate` | `id[]` / string | Tasks removed from the hand today |
| `lastBackupDate` | string | Date of last backup (weekly auto-download) |
| `paused` / `pausedAt` | bool / ts | Vacation mode |
| `tierCLastDate` | string | **Vestigial**, kept only for migration safety |
| `_dealPreferTier` | string | **Transient**, set before a tier-favoring redeal, deleted in `dealHand` |

---

## 10. Persistence, migration & backup

### Saving

- `saveState()` debounces writes 300ms. `saveNow()` writes immediately (used for
  completions and other actions that must persist before a possible reload).

### Migration

`loadState()` reads `hometasks_v8`, falling back to `hometasks_v7`, and fills any
missing field with a safe default. **Adding a new state field never requires an
export** — just give it a default here. `mode: 'survival'` (a removed mode) is
migrated to `maintenance`.

`loadState()` (and `applyImport()`) also **backfill `completions`**: any task in
`TASKS` without a completion timestamp is seeded to `now − freq` days. This means
**adding a new base task never requires an export either** — without the backfill,
`daysOverdue()` would treat the missing timestamp as epoch and report ~20,000 days
overdue; with it, the new task simply reads as due-now and phases into the hand
naturally.

### When an export IS required before deploying

- Bumping `STORAGE_KEY` to a new version (abandons the old key).
- Changing an existing field's format in a breaking way.
- Intentionally clearing/restructuring state.

Otherwise: **no export needed.** Every code-change description should end with
`⚠️ Export needed before deploying` or `✅ No export needed`.

### Backup & import

- A backup `.json` auto-downloads on load at most weekly (`autoBackupCheck`,
  tracked by `lastBackupDate`), with a toast so it isn't mysterious.
- "Back up now" forces a download. Export copies the full state JSON to the
  clipboard.
- Import accepts a pasted blob or a restored `.json` file; it validates that
  `completions` exists, confirms, then replaces all state. **Import overwrites
  everything.**

> Each iOS home-screen shortcut has isolated storage, so export/import is the
> only way to move history between shortcuts or devices.

---

## 11. Special modes & features

### Vacation mode

- `pauseApp()` sets `paused`/`pausedAt`. While paused, `dealHand` keeps the
  current hand and skips dealing + starvation; a banner shows on My Hand.
- `resumeApp()` shifts every **pre-pause** completion timestamp forward by the
  pause duration (so nothing aged while away — you don't come home to a wall of
  fake-overdue tasks), then deals a fresh hand. `completionHistory` is **not**
  shifted (it records what actually happened). Completing tasks while paused is
  allowed and records normally.

### Per-task timer & learned times

- One timer at a time. Tap to start; starting another stops the first without
  recording. Elapsed time shows live on the card.
- Stopping the timer, or checking the task off while it runs, opens a
  confirm/adjust modal (you can edit or discard). Manual entry is also available
  via the ⏲ path. Entries are clamped to 1–600 minutes.
- `actualTimes[id]` keeps the last 10 durations. `taskTime(task)` returns the
  median once there are 3+ samples, else the static `time`. Learned times drive
  budget filling and all time displays.

### One-off tasks

"+ one-off task" creates an `oneOff` custom task, pins it, and adds it to today.
It's always "due", and is removed entirely after completion.

---

## 12. Edge cases & gotchas

- **Daily early-return in `dealHand`.** The hand is built once per day and
  cached; pin/flag/snooze/remove actions mutate `state.hand` directly rather
  than re-dealing, except where they explicitly null `hand`/`handDate` to force
  a rebuild (redeal, mode change, budget change, editing tasks).
- **Removed-today tasks** stay out of redeals and "Give me more" until the next
  calendar day — unless completed today, or re-pinned (pinning clears the
  removal).
- **Completed tasks remain visible** in the hand (Completed section) for the day,
  because `dealHand` re-adds anything completed today.
- **Flagged + budget.** The first flagged task always surfaces; subsequent
  flagged tasks respect the budget and overflow to "Give me more".
- **Heavy cap interaction.** As of 2026-06-15 every always-assigned task
  (`c_fountain` and the laundry loads) is ≤5 min, so none are "heavy" — only
  flagged tasks over 15 min consume the 2-heavy cap. Re-check this if a heavy
  task is ever added to the always-assigned set.
- **Jitter re-randomizes overflow ordering** for `freq > 60` tasks on each
  render (cosmetic — the cached hand is unaffected).
- **Preset ids must exist in `TASKS`** for lists that render via
  `getAllTasks()`, or they silently drop; `completeTask` returns early for
  unknown ids, so a phantom checklist item can't be checked off.
- **New base tasks are backfilled, not absurdly overdue.** `loadState`/`applyImport`
  seed a missing `completions[id]` to `now − freq` days, so a task added in a deploy
  shows as due-now rather than ~20,000 days overdue on the user's existing data.
- **Seasonal Deep Clean keys off Tier C (`freq > 60`).** It therefore includes a
  few light upkeep items (soap refills, descales) alongside true deep tasks, and it
  still lists DS-bath deep tasks even though that bath is rarely used. By design —
  skip them or use "Load due only".

---

## 13. Deployment

GitHub Pages. To ship a change:

1. Edit `index.html`.
2. Bump `APP_VERSION` to `YYYY-MM-DD vN` — start at `v1` for a new date, or increment `vN` if today's date is already in the current version (e.g. `v1` → `v2`).
3. Verify in the browser preview.
4. Commit + push. The live site updates; user `localStorage` is untouched.
5. Note `✅ No export needed` or `⚠️ Export needed before deploying`.

---

## 14. Glossary

- **Hand** — the set of tasks dealt for today.
- **Deal / redeal** — build (or rebuild) the hand.
- **Budget** — minutes allotted to non-daily tasks; dailies are always included.
- **Tier** — A/B/C bucket by frequency, for UI filtering only.
- **Mode** — Keep up / Catch up / By freq, controlling pool order and overdue weight.
- **Zone** — rotating room focus giving a scoring boost.
- **Starvation** — counter raising the priority of long-undealt due tasks.
- **Cadence** — the real median interval a task actually gets done, vs its target `freq`.
- **Sticky tasks** — pinned + in-progress tasks that ride along when a preset loads.
- **Overflow / backup list** — due/not-due tasks beyond the dealt hand.
