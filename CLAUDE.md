# Home Tasks App

A personal home task manager for Alina and Bob, deployed as a GitHub Pages site.

## What This Is

A single-file (`index.html`) web app — all HTML, CSS, and JavaScript in one file with no build step, no dependencies, and no server. State is stored entirely in `localStorage`. Changes are deployed by committing and pushing `index.html`.

## How It Works

The app deals a daily "hand" of tasks based on a time budget, pulling from a weighted task pool. It's inspired by FlyLady's zone cleaning system.

### Task Data Structure

Each task has:
- `id` — unique string key
- `name` — display name
- `room` — location (used for zone filtering)
- `freq` — how often it should be done, in days (1 = daily, 7 = weekly, 180 = twice a year)
- `time` — estimated minutes
- `owner` — `'alina'`, `'bob'`, or `'either'`
- `cat` — boolean, true for cat-care tasks (minor score boost)

The base task list lives in the `TASKS` array (~line 483, 177 entries). Users can add custom tasks and hide base tasks without touching code. Some entries with `custom_…`/`oneoff_…` ids are personal customizations baked into the base array.

### Scoring (`scoreTask` ~line 1015, `scoreTaskParts` ~line 993)

Lower score = higher priority. `scoreTaskParts` returns the labeled component breakdown (used by the "why?" modal); `scoreTask` sums them and adds jitter. Components:
- Base score = `task.freq` (lower frequency → lower base → higher priority)
- Overdue penalty: reduces score based on how overdue the task is, capped at `freq * 0.8`; weight is `0.5` normally, `0.8` in Catch Up mode
- Starvation: the `starvation` counter ticks `+1` per calendar day a task is due but not dealt; the score reduction is `-0.3 * count`, **capped at `freq * 0.15`** (this cap is critical — without it, long-interval tasks accumulate infinite priority and dominate the hand)
- Flagged tasks: `-50` (always surfaces first)
- Zone bonus: `-2` if task is in the active zone
- Cat tasks: `-0.5` tiebreaker
- Jitter: tasks with `freq > 60` get `±1` random added in `scoreTask` (not `scoreTaskParts`) for deep-clean variety

### `getTaskTier(task)` (~line 1043)

```javascript
function getTaskTier(task) {
  if (task.freq <= 7)  return 'A';  // Operational
  if (task.freq <= 60) return 'B';  // Maintenance
  return 'C';                        // Deep clean
}
```

Tiers are used only for the `_dealPreferTier` feature (redeal with a tier preference) — they do **not** drive separate quota or bypass logic inside `dealHand`.

### Hand Dealing (`dealHand`, ~line 1049)

**Always-assigned (bypass budget):**
- `c_fountain` — always appears when due, regardless of mode
- One laundry load task (see Laundry below) — always assigned when due

**Mode-dependent pool filling:**

| Mode | Pool sort | Fill rule |
|------|-----------|-----------|
| **Keep Up** | Score (overdue weight 0.5) | Fill budget in score order; heavy tasks (>15 min) capped at 2 per hand |
| **Catch Up** | Score (overdue weight 0.8) | Same as Keep Up; overdue tasks score higher so they surface faster |
| **By Freq** | Raw frequency, shuffle within same-freq | Fill budget in freq order |

If `_dealPreferTier` is set (via redeal with tier button), tasks of that tier float to the top of the pool for that deal only, then the field is deleted.

**Flagged tasks:** enter the pool even when not yet due (flag bypasses `isDue`) and fill **first in every mode** — critical for By Freq, where the raw-frequency sort would otherwise ignore the -50 flag bonus entirely. They count against the budget and dealing stops at the limit, except the first flagged task is always dealt (so flagging guarantees surfacing even when the budget is spent). Flagged tasks that don't fit, and `getOverflowTasks()` generally, surface via "+ Give me more", which includes flagged not-yet-due tasks.

**Laundry slot logic:**
- "Load" tasks (`lroom_myclothes`, `lroom_towels`, `lroom_microfiber`, `lroom_whites`, `k_towels`) compete for a single daily slot — at most one is assigned per day, chosen by score (most overdue wins)
- If a load task was already completed today or is pinned, no new load is assigned on redeal
- "Process" steps (`l_start`, `l_dryer`, `l_fold`, `l_put_away`) are daily tasks suppressed unless a load is assigned today, pinned, or already completed today

### Modes

Selected via three pill buttons in the Today tab header. Switching a mode clears and re-deals the hand.

- **🏠 Keep up** (`maintenance`, default): score-sorted pool, overdue weight 0.5
- **🔄 Catch up** (`catchup`): same structure, overdue weight 0.8 — overdue tasks score higher so they surface faster
- **📋 By freq** (`byfreq`): pool sorted by raw frequency (shortest interval first), shuffle within same-freq

### State

Stored in `localStorage` as JSON under `STORAGE_KEY = 'hometasks_v8'`. Key fields:
- `completions` — `{taskId: timestamp}` of last completion. `loadState` and `applyImport` backfill any base task missing an entry to `now − freq` days, so a newly added base task reads as due-now instead of ~20,000 days overdue
- `completionHistory` — `{taskId: timestamp[]}` last 100 completion timestamps per task
- `starvation` — `{taskId: count}` of consecutive days due but not dealt
- `hand` / `handDate` — today's task list
- `budgetWeekday` / `budgetWeekend` — time budget in minutes for non-daily tasks (default 45/90)
- `zone` — current zone focus (0 = none, 1–5 = active zone)
- `mode` — `'maintenance'` | `'catchup'` | `'byfreq'`
- `tierCLastDate` — vestigial field kept for migration safety; no longer used in dealing logic
- `_dealPreferTier` — transient; set before redeal to float a tier, deleted immediately after
- `pinnedIds`, `flaggedIds`, `snoozed`, `deletedIds`, `customTasks`
- `guestHand`, `resetHand`, `goingOutHand` — legacy preset checklist state (Presets tab); all other presets live in `presetHands`
- `presetHands` — `{presetType | room_<id>_<depth>: id[] | null}` generated preset checklists (Express Reset, Weekly Reset, Return Home, Before Cleaners, Recovery, Post-Illness, Evening Shutdown, Seasonal Deep Clean, room presets)
- `removedToday` / `removedTodayDate` — task ids removed from the hand today; excluded from redeals and "give me more" until the next calendar day (unless completed today)
- `lastBackupDate` — ISO date of last backup download (auto-download fires weekly on load)
- `zoneMode` — `'auto'` (default; zone follows day-of-month: 1–7→Z1 … 29+→Z5 via `autoZone()`/`effectiveZone()`) | `'manual'`
- `paused` / `pausedAt` — vacation mode; while paused no dealing or starvation ticks, on resume pre-pause completion timestamps shift forward by the pause duration
- `taskMonths` — `{taskId: [1..12]}` seasonal windows; out-of-season tasks are excluded from dealing pools and starvation. `seasonalDefaultsApplied` guards the one-time defaults migration (porch tasks Mar–Nov, window-cleaning Apr–Oct)
- `actualTimes` — `{taskId: minutes[]}` (last 10); fed by the timer (stop or check-off opens a confirm/adjust modal) or manual entry (⏲ icon in Manage); `taskTime()` returns the median of 3+ samples (else static `time`) and drives budget filling and time displays
- `inProgress` — `{taskId: timestamp}` started-but-unfinished tasks (e.g. cat fountain in the dishwasher); carried into every new hand by `dealHand` until completed; cleared on complete/snooze/remove, restored by undo

### UI Tabs

Six tabs in the bottom nav (internal `currentTab` id in parentheses):

- **My Hand** (`alina`) — the main dealt hand titled "Today's Hand"; header has Redeal (+ A/B/C tier redeal), the zone selector, the three mode pills, and "+ one-off task". Cards: check off (undo toast), flag, snooze, "did earlier" backdating, pin, remove, started…/in-progress, per-task timer, "why?" score-breakdown modal (`scoreTaskParts`), and a quick-log search to record completions outside the hand. Sections: Daily Tasks, Other Tasks, Completed, "+ Give me more" / Backup list (due + not-due)
- **Sprint** (`sprint`) — room-grouped, ordered walkthrough of today's hand (Launch → Kitchen → Cat care → Laundry → room groups), per-block time, hide-done toggle, per-task timer/in-progress/remove buttons
- **All Tasks** (`manage`) — full task list with search + owner/room/status filters; add/edit/delete custom tasks, hide base tasks, pin/flag, +Today, edit last-done; month chips in the editor set seasonal windows; shows learned-time and real-cadence hints
- **Presets** (`presets`) — see the Presets section below
- **Stats** (`stats`) — This Week box (count/effort/streak/today), Budget Insight (suggests budget changes from median non-daily throughput, apply-only), 13-week activity heatmap, and full Completion Cadence list with due badges, +Today/flag, "set target Nd" adoption, and tap-name-to-edit. Real cadence (`cadenceInfo()`, colored `~Nd real`) also shows inline on My Hand and All Tasks cards
- **Settings** (`settings`) — budget steppers, active-zone reference table, system overview, scoring explainer, recent-activity log, vacation mode toggle, reload-app card (shows `APP_VERSION`), weekly backup card, and export/import (paste or restore-from-file)

### Presets Tab

Presets build a checklist you tick through (completions count app-wide), then optionally "Load all" or "Load due only" into today's hand. `mergeStickyHand()` ensures pinned + in-progress tasks ride along when a preset is loaded. Categories:

- **Random Task** — pick a random due task (optionally filtered to tier A/B/C), then add it to today
- **Routines** — **Express Reset** (quick whole-house pass), **Weekly Reset** (the classic weekly/Sunday reset: sheets, bathroom, kitchen, vacuum + dust the living spaces, towels, cat care; `WEEKLY_RESET_SECTIONS`, ~2h), and **Full Reset** (`state.resetHand`, complete 2–3h clean)
- **Guests** — **Guest Prep** with Emergency (~15 min) / Day / Overnight variants (`state.guestHand`)
- **Travel** — **Going Out of Town** (`state.goingOutHand`) and **Return Home**
- **Special** — **Before Cleaners**, **Recovery Mode** (phased), **Post-Illness** (phased: Sanitize → Restore)
- **Daily Rituals** — **Evening Shutdown** (fixed core + the single most-overdue tidy room)
- **Seasonal** — **Seasonal Deep Clean**: a spring-cleaning blitz that snapshots every in-season Tier C task into one room-grouped checklist. Built dynamically by `generateSeasonalDeepClean()` (no fixed section constant), stored flat in `presetHands.seasonalDeep`, regrouped for render by `seasonalDeepSections()`. Load all → or Load due only →
- **Rooms** — per-room **Quick** / **Deep** presets for all 14 rooms (`ROOM_PRESETS`)

Generated preset state lives in `presetHands` (keyed by type or `room_<id>_<depth>`), except the three legacy ones that have dedicated fields (`guestHand`, `resetHand`, `goingOutHand`). Preset task definitions live in module constants near the top of the Presets section (`FULL_RESET_TASKS`/`FULL_RESET_SECTIONS`, `GOING_OUT_TASKS`/`…SECTIONS`, `EXPRESS_RESET_SECTIONS`, `WEEKLY_RESET_SECTIONS`, `RETURN_HOME_SECTIONS`, `BEFORE_CLEANERS_SECTIONS`, `RECOVERY_SECTIONS`, `POST_ILLNESS_SECTIONS`, `EVENING_SHUTDOWN_FIXED`/`EVENING_TIDY_POOL`, `ROOM_PRESETS`). Seasonal Deep Clean has **no** constant — its id list is computed at generate time. Fixed-section presets register in `getNewPresetSections()`; Weekly Reset is wired there as `weeklyReset` so the generic `generateNewPreset`/`loadNewPreset`/`loadNewPresetDueOnly`/`clearNewPreset` handle it.

### Zone System

Five cleaning zones (FlyLady-style), each covering specific rooms:
- Zone 1: Dining Room, Mud Room, Back Porch
- Zone 2: Kitchen
- Zone 3: Sunroom, Office, DS Bathroom, Laundry Room, Hall & Stairs
- Zone 4: Bedroom, US Bathroom
- Zone 5: Living Room, Back Room

Selecting a zone gives a `-2` score bonus to tasks in that zone.

`Whole House`, `Robot`, `Downstairs`, and `Upstairs` are zone-less rooms — tasks in them don't compete for zone bonuses.

## Repository Files

- `index.html` — the entire app (HTML + CSS + JS in one file, ~3960 lines)
- `CLAUDE.md` — this file: working instructions + architecture summary for Claude
- `DOCUMENTATION.md` — full reference documentation of the app (data model, algorithms, every tab, state schema, edge cases)
- `IMPROVEMENT_PLAN.md` — historical plan for the 9 features shipped in June 2026; **all 9 are now implemented** (auto-backup, undo/backdating, quick-log, Stats tab, "why?" breakdown, zone auto-rotation, vacation mode, seasonal months, per-task timer). Its line-number anchors are stale; keep only as a record of intent

## Known Issues / Watch-outs

The June 2026 review bugs — dead preset task ids (`k_backsplash`, `dsb_exhaust`, `dsb_mop`, `bed_vacuum`), `completeEarlier` double-logging, `applyImport` field gaps, preset name/room drift, and the phantom "Front Porch" zone entry — were all **fixed on 2026-06-15**.

Editing pitfalls that remain true (also encoded in the `app-update` skill):
- **Adding state** requires a default in all three of `defaultState()`, `loadState()`, and `applyImport`, or import silently drops the field.
- **Preset task ids** in lists that render via `getAllTasks()` (`ROOM_PRESETS`, `EXPRESS_RESET_SECTIONS`, `RETURN_HOME_SECTIONS`, `BEFORE_CLEANERS_SECTIONS`, `RECOVERY_SECTIONS`, `POST_ILLNESS_SECTIONS`, `EVENING_*`) must exist in `TASKS`, or they silently vanish; `completeTask` returns early for unknown ids, so a phantom checklist item can never be checked off.

Minor known behaviors (by design / low priority):
- Random Task ignores owner, so it can surface a Bob-only task to add to the hand.
- Stats "est. effort" uses static `time` while Budget Insight uses learned `taskTime()`.
- The `freq > 60` jitter reshuffles the overflow list on each render (cosmetic; the cached hand is unaffected).
- Seasonal Deep Clean keys off Tier C (`freq > 60`), so it includes a few light upkeep items (soap refills, descales) alongside true deep-clean tasks, and it still lists DS-bath deep tasks even though that bath is rarely used. Both are by design — skip them or use "Load due only".

## Deployment

GitHub Pages — push `index.html` to deploy. No build step needed.

## State & Export/Import

All app data (completions, starvation, custom tasks, settings) lives in `localStorage` in the browser. Deploying a new `index.html` to GitHub Pages **never touches localStorage** — data persists across deploys automatically.

**Export/import is NOT needed for:**
- UI or logic changes (scoring, dealing, modes)
- Adding new state fields — `loadState()` has a migration block that sets safe defaults for any missing fields
- Adding new base tasks to `TASKS` — `loadState`/`applyImport` backfill missing `completions` so they read as due-now, not absurdly overdue

**Export first IS needed when:**
- The `STORAGE_KEY` constant is bumped to a new version — the old key is abandoned and state starts fresh
- An existing field's format changes in a breaking way (e.g., `completions` value type changes)
- State is intentionally being cleared or restructured

**Claude must always note** at the end of each code change description: `⚠️ Export needed before deploying` or `✅ No export needed`.

## Commit Message Convention

Provide a commit title and body after every code change. No line breaks in the body — let it wrap naturally. Both in a single fenced code block.
