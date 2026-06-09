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

The base task list lives in the `TASKS` array (~line 402). Users can add custom tasks and hide base tasks without touching code.

### Scoring (`scoreTask`, ~line 786)

Lower score = higher priority. Key components:
- Base score = `task.freq` (lower frequency → lower base → higher priority)
- Overdue penalty: reduces score based on how overdue the task is, capped at `freq * 0.8`; weight is `0.5` normally, `0.8` in Catch Up mode
- Starvation: `+1/day` for each day a task is due but not dealt; reduces score, **capped at `freq * 0.15`** (this cap is critical — without it, long-interval tasks accumulate infinite priority and dominate the hand)
- Flagged tasks: `-50` (always surfaces first)
- Zone bonus: `-2` if task is in the active zone

### `getTaskTier(task)` (~line 824)

```javascript
function getTaskTier(task) {
  if (task.freq <= 7)  return 'A';  // Operational
  if (task.freq <= 60) return 'B';  // Maintenance
  return 'C';                        // Deep clean
}
```

Tiers are used only for the `_dealPreferTier` feature (redeal with a tier preference) — they do **not** drive separate quota or bypass logic inside `dealHand`.

### Hand Dealing (`dealHand`, ~line 830)

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

**Laundry slot logic:**
- "Load" tasks (`lroom_myclothes`, `lroom_towels`, `lroom_microfiber`, `k_towels`) compete for a single daily slot — at most one is assigned per day, chosen by score (most overdue wins)
- If a load task was already completed today or is pinned, no new load is assigned on redeal
- "Process" steps (`l_start`, `l_dryer`, `l_fold`, `l_put_away`) are daily tasks suppressed unless a load is assigned today, pinned, or already completed today

### Modes

Selected via three pill buttons in the Today tab header. Switching a mode clears and re-deals the hand.

- **🏠 Keep up** (`maintenance`, default): score-sorted pool, overdue weight 0.5
- **🔄 Catch up** (`catchup`): same structure, overdue weight 0.8 — overdue tasks score higher so they surface faster
- **📋 By freq** (`byfreq`): pool sorted by raw frequency (shortest interval first), shuffle within same-freq

### State

Stored in `localStorage` as JSON under `STORAGE_KEY = 'hometasks_v8'`. Key fields:
- `completions` — `{taskId: timestamp}` of last completion
- `completionHistory` — `{taskId: timestamp[]}` last 100 completion timestamps per task
- `starvation` — `{taskId: count}` of consecutive days due but not dealt
- `hand` / `handDate` — today's task list
- `budgetWeekday` / `budgetWeekend` — time budget in minutes for non-daily tasks (default 45/90)
- `zone` — current zone focus (0 = none, 1–5 = active zone)
- `mode` — `'maintenance'` | `'catchup'` | `'byfreq'`
- `tierCLastDate` — vestigial field kept for migration safety; no longer used in dealing logic
- `_dealPreferTier` — transient; set before redeal to float a tier, deleted immediately after
- `pinnedIds`, `flaggedIds`, `snoozed`, `deletedIds`, `customTasks`
- `guestHand`, `resetHand`, `goingOutHand` — preset checklist state (Presets tab)
- `lastBackupDate` — ISO date of last backup download (auto-download fires weekly on load)
- `zoneMode` — `'auto'` (default; zone follows day-of-month: 1–7→Z1 … 29+→Z5 via `autoZone()`/`effectiveZone()`) | `'manual'`
- `paused` / `pausedAt` — vacation mode; while paused no dealing or starvation ticks, on resume pre-pause completion timestamps shift forward by the pause duration
- `taskMonths` — `{taskId: [1..12]}` seasonal windows; out-of-season tasks are excluded from dealing pools and starvation. `seasonalDefaultsApplied` guards the one-time defaults migration (porch tasks Mar–Nov, window-cleaning Apr–Oct)
- `actualTimes` — `{taskId: minutes[]}` (last 10) recorded by the per-task timer; `taskTime()` returns the median of 3+ samples (else static `time`) and drives budget filling and time displays

### UI Tabs

- **Today** — the main dealt hand; check off (undo toast), flag, snooze, "did earlier" backdating, "why?" score-breakdown modal (`scoreTaskParts`), per-task timer, and a quick-log search to record completions outside the hand
- **Sprints** — timed sprint view of today's hand (per-task timer buttons here too)
- **Presets** — generate and work through preset checklists: Guest Prep (day/overnight variants), Full Reset, Going Out of Town
- **Stats** — 13-week heatmap, week-vs-week counts, streak, chronically-overdue list (all derived from `completionHistory`, no own state)
- **Manage** — full task list with filters; add/edit/delete custom tasks, hide base tasks; month chips in the editor set seasonal windows
- **Settings** — budget sliders, vacation mode toggle, backup card, import/export state (the zone selector lives on the Today tab)

### Zone System

Five cleaning zones (FlyLady-style), each covering specific rooms:
- Zone 1: Dining Room, Mud Room, Back Porch
- Zone 2: Kitchen
- Zone 3: Sunroom, Office, DS Bathroom, Laundry Room, Hall & Stairs
- Zone 4: Bedroom, US Bathroom
- Zone 5: Living Room, Back Room

Selecting a zone gives a `-2` score bonus to tasks in that zone.

## Deployment

GitHub Pages — push `index.html` to deploy. No build step needed.

## State & Export/Import

All app data (completions, starvation, custom tasks, settings) lives in `localStorage` in the browser. Deploying a new `index.html` to GitHub Pages **never touches localStorage** — data persists across deploys automatically.

**Export/import is NOT needed for:**
- UI or logic changes (scoring, dealing, modes)
- Adding new state fields — `loadState()` has a migration block that sets safe defaults for any missing fields

**Export first IS needed when:**
- The `STORAGE_KEY` constant is bumped to a new version — the old key is abandoned and state starts fresh
- An existing field's format changes in a breaking way (e.g., `completions` value type changes)
- State is intentionally being cleared or restructured

**Claude must always note** at the end of each code change description: `⚠️ Export needed before deploying` or `✅ No export needed`.

## Commit Message Convention

Provide a commit title and body after every code change. No line breaks in the body — let it wrap naturally. Both in a single fenced code block.
