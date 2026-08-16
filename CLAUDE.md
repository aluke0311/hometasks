# Home Tasks App

A personal home task manager for Alina and Bob, deployed as a GitHub Pages site.

## Writing

Write to Alina in Simplified Technical English. Every rule here is a thing to do.

- Lead with the result. Give the reason after it.
- Put one idea in each sentence. Keep sentences to about 20 words.
- Use the active voice. Name who does what.
- Choose the shortest word that carries the meaning.
- Cut every word that does no work.
- Describe things plainly, in place of figures of speech.
- Use the same word for the same thing, every time.
- Use everyday words for everyday ideas. Keep the codebase's real names — `dealHand`,
  Tier C, `NEGLECT_FLOOR`, starvation. A plain-English substitute for those reads as
  less clear, not more.
- Use a table to compare two or more things.
- Say what you measured and what you assumed. Mark an estimate as an estimate.
- Say plainly when something got worse, or when you were wrong, and then say what you
  did about it.
- Break any rule above when following it would make the sentence worse.

She is a statistical coder, not a web developer. Explain web and browser concepts.
Keep the maths at full strength.

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

The base task list lives in the `TASKS` array (186 entries). Users can add custom tasks and hide base tasks without touching code. Some entries with `custom_…`/`oneoff_…` ids are personal customizations baked into the base array.

### Scoring (`scoreTask`, `scoreTaskParts`)

Lower score = higher priority. `scoreTaskParts` returns the labeled component breakdown (used by the "why?" modal); `scoreTask` sums them and adds jitter. Components:
- Base score = `task.freq` (lower frequency → lower base → higher priority)
- Overdue penalty: reduces score based on how overdue the task is; weight is `0.5` normally, `0.8` in Catch Up. **The cap is mode-dependent (`overdueCap`) and that is the whole difference between the two modes:**
  - **Catch Up** caps at `freq − NEGLECT_FLOOR` (`NEGLECT_FLOOR = 3`) — an *absolute* floor, so every task reaches 3 at ~1.25× its own interval whatever the interval is
  - **Keep Up / By Freq** keep the original `freq * 0.8`, which leaves a floor of `freq * 0.05` that scales with the interval
  - Before this split, both modes used `freq * 0.8` and so scored any well-overdue task identically — Catch Up was a second copy of Keep Up. It also meant a 365-day task could never score below 18.25 while routine work sits between 0.5 and 6, so tier C could never be dealt **in any mode**: 68 of 182 tasks had never been completed once, and the simulator measured 0% tier C coverage everywhere
  - **Keep Up deliberately did not get the floor.** Two attempts were measured against real state and both made the everyday hand worse (tier A fell from 6 tasks to 2), because badly neglected tier B/C tasks are *heavy* and a few of them consume a budget that previously held six small routine ones. Raising the floor to 6 changed nothing, which is what proved the cause was task weight rather than the floor value. Frequent tasks really are more important; backlog work belongs in Catch Up
- Starvation: the `starvation` counter ticks `+1` per calendar day a task is due but not dealt; the score reduction is `-0.3 * count`, **capped at `freq * 0.15`** (this cap is critical — without it, long-interval tasks accumulate infinite priority and dominate the hand)
- Flagged tasks: `-50` (always surfaces first)
- Zone bonus: `-2` if task is in the active zone
- Cat tasks: `-0.5` tiebreaker
- Jitter: tasks with `freq > 60` get `±1` random added in `scoreTask` (not `scoreTaskParts`) for deep-clean variety

### `getTaskTier(task)`

```javascript
function getTaskTier(task) {
  if (task.freq <= 7)  return 'A';  // Operational
  if (task.freq <= 60) return 'B';  // Maintenance
  return 'C';                        // Deep clean
}
```

Tiers are used only for the `_dealPreferTier` feature (redeal with a tier preference) — they do **not** drive separate quota or bypass logic inside `dealHand`.

### Hand Dealing (`dealHand`)

**Always-assigned (bypass budget):**
- `c_fountain` — always appears when due, regardless of mode
- One laundry load task (see Laundry below) — always assigned when due

**The budget being filled** is `budgetWeekday`/`budgetWeekend` (non-daily time only) *unless* a `todayBudget` is set for today, in which case it is `todayBudget − dailyTime` — the slider is a whole-day total, so the daily load comes off the top. `dailyTime` covers the dailies being forced into the hand plus any already completed today. Only tasks the pool could contain (`alina`/`either`) count as time already spent: letting a Bob-only completion consume the budget silently shrank her hand, since she logs his chores.

**Mode-dependent pool filling:**

| Mode | Pool sort | Fill rule |
|------|-----------|-----------|
| **Keep Up** | Score (overdue weight 0.5, cap `freq*0.8`) | Fill budget in score order; heavy tasks (>15 min) capped at 2 per hand |
| **Catch Up** | Score (overdue weight 0.8, cap `freq-3`) | Same fill, but the absolute floor lets long-neglected tier C tasks reach the top |
| **By Freq** | Raw frequency, shuffle within same-freq | Fill budget in freq order |

If `_dealPreferTier` is set (via redeal with tier button), tasks of that tier float to the top of the pool for that deal only, then the field is deleted.

**Flagged tasks:** enter the pool even when not yet due (flag bypasses `isDue`) and fill **first in every mode** — critical for By Freq, where the raw-frequency sort would otherwise ignore the -50 flag bonus entirely. They count against the budget and dealing stops at the limit, except the first flagged task is always dealt (so flagging guarantees surfacing even when the budget is spent). Flagged tasks that don't fit, and `getOverflowTasks()` generally, surface via "+ Give me more", which includes flagged not-yet-due tasks.

**Laundry slot logic:**
- "Load" tasks compete for a single daily slot — at most one is assigned per day, chosen by score (most overdue wins). Membership is declared per-task by **`load: true`**, not a hardcoded id list: `dealHand`'s `LAUNDRY_LOAD_IDS`, the sprint's `SPRINT_LOAD_IDS`, and the selftest's `loadIds()` all derive from the flag. Currently `lroom_whites` (7d), `lroom_cools` (7d), `lroom_warms` (10d), `lroom_towels` (7d), `lroom_microfiber` (14d), `k_towels` (7d) — combined demand ≈ 0.79 loads/day against a slot that serves 1/day, so the queue has little slack; adding another weekly load will make it run late
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
- `completionHistory` — `{taskId: timestamp[]}` completion timestamps, pruned by **age (3 years)** rather than count; `pruneHistory()` does this. The old 100-entry cap is what forced year totals to be accumulated instead of derived
- `starvation` — `{taskId: count}` of consecutive days due but not dealt
- `hand` / `handDate` — today's task list
- `budgetWeekday` / `budgetWeekend` — time budget in minutes for non-daily tasks (default 45/90)
- `todayBudget` / `todayBudgetDate` — the "Today's time" slider on My Hand. A **total** for the day, unlike the two fields above: `dealHand` charges the daily load against it and the remainder becomes the non-daily budget. Dated, so it expires on its own tomorrow, and it never writes to `budgetWeekday`/`budgetWeekend`. Replaced the old "Got 10/20/30 min" chips, which set only the non-daily budget and so promised 10 minutes while dealing an hour
- `prevCompletions` — `{taskId: timestamp}` of the date a *today* completion replaced, so `uncompleteTask` can restore the real one. Pruned to today's completions on every `completeTask`; a second same-day completion does not overwrite an existing entry
- `zone` — current zone focus (0 = none, 1–5 = active zone)
- `mode` — `'maintenance'` | `'catchup'` | `'byfreq'`
- `tierCLastDate` — vestigial field kept for migration safety; no longer used in dealing logic
- `_dealPreferTier` — transient; set before redeal to float a tier, deleted immediately after
- `pinnedIds`, `flaggedIds`, `snoozed`, `deletedIds`, `customTasks`
- `guestHand`, `resetHand`, `goingOutHand` — **retired 2026-08-16.** They now migrate into `presetHands` under the keys `guest`/`reset`/`goingout` via `migrateLegacyPresets()`, which runs on every load (not behind a flag) so an old export still converts. The fields stay in `defaultState()` and are always left `null`. Every preset runs on one code path now
- `presetHands` — `{presetType | room_<id>_<depth>: id[] | null}` generated preset checklists (Full Reset, Guest Prep, Going Out of Town, Express Reset, Weekly Reset, Return Home, Before Cleaners, Recovery, Post-Illness, Evening Shutdown, Seasonal Deep Clean, room presets)
- `removedToday` / `removedTodayDate` — task ids removed from the hand today; excluded from redeals and "give me more" until the next calendar day (unless completed today)
- `lastBackupDate` — ISO date of last backup download (auto-download fires weekly on load)
- `zoneMode` — `'auto'` (default; zone follows day-of-month: 1–7→Z1 … 29+→Z5 via `autoZone()`/`effectiveZone()`) | `'manual'`
- `paused` / `pausedAt` — vacation mode; while paused no dealing or starvation ticks. On resume each pre-pause completion shifts forward by `pauseDuration × VAC_SHIFT[vacMode(id)]` — see below. Completions logged *during* the pause are never shifted (a sitter's work, or something done the morning you left)
- `taskVac` — `{taskId: 'freeze'|'half'|'full'}` per-task vacation decay. `VAC_SHIFT` is the **share of the pause given back**: `freeze` 1 (didn't age at all), `half` 0.5, `full` 0 (aged exactly as if home). `vacMode()` returns `'freeze'` for anything unset, which is precisely how vacation mode behaved before this existed. `DEFAULT_VAC` (built from `VAC_HALF`/`VAC_FULL`) is applied once via the `vacDefaultsApplied` migration; edits in the task editor win after that. Currently 113 freeze / 61 half / 12 full. **Keyed by id, deliberately not stored on the task** — `saveTask` rebuilds a custom task from an explicit field list and would strip it (see the `load: true` watch-out below)
- `taskMonths` — `{taskId: [1..12]}` seasonal windows; out-of-season tasks are excluded from dealing pools and starvation. `seasonalDefaultsApplied` guards the one-time defaults migration (porch tasks Mar–Nov, window-cleaning Apr–Oct)
- `actualTimes` — `{taskId: minutes[]}` (last 10); fed by the timer (stop or check-off opens a confirm/adjust modal) or manual entry (⏲ icon in Manage); `taskTime()` shrinks the observed **median** toward the static `time` with weight `n/(n+2)`, so one sample moves the estimate a third of the way and it never lurches; it drives budget filling and every time display. The median (not the mean) is deliberate — a forgotten timer is one wild high sample
- `derivedSince` — timestamp stamped on first load after 2026-08-04. A year is only derived (`yearIsDerivable`) if this precedes its start; earlier years fall back to the stored `yearStats` bucket. **Don't infer derivability from the data** — a twice-yearly task's old entries would certify a year whose daily tasks were truncated
- `milestones` — `{key: timestamp}` seen-set; each milestone key fires once ever. `bestStreak` backs the streak rung. Undo un-spends both
- `seenVersion` — last `APP_VERSION` whose what's-new toast has been shown
- `activeTimer` — `{id, startedAt}` of the running per-task timer, or `null`. **Persisted deliberately**: it used to be an in-memory variable, so backgrounding the tab — what happens when the phone goes in a pocket mid-task — destroyed the timer and its elapsed time silently. A run over 3h is flagged as forgotten rather than logged; over 12h it is discarded on next load
- `inProgress` — `{taskId: timestamp}` started-but-unfinished tasks (e.g. cat fountain in the dishwasher); carried into every new hand by `dealHand` until completed; cleared on complete/snooze/remove, restored by undo

### UI Tabs

Six tabs in the bottom nav (internal `currentTab` id in parentheses):

- **My Hand** (`alina`) — the main dealt hand titled "Today's Hand"; header has Redeal (+ A/B/C tier redeal), the zone selector, the three mode pills, the **Today's time slider** (`renderDayBudget`; drag repaints the labels only, the deal happens on release, and "back to usual" clears the override), and "+ one-off task". Cards: check off (undo toast), flag, snooze, **un-snooze**, "did earlier" backdating, pin, remove, started…/in-progress, per-task timer, "why?" score-breakdown modal (`scoreTaskParts`), and a quick-log search to record completions outside the hand. Sections: Daily Tasks, Other Tasks, Completed, "+ Give me more" / Backup list (due + not-due)
- **Sprint** (`sprint`) — room-grouped, ordered walkthrough of today's hand (Launch → Kitchen → Cat care → Laundry → room groups), per-block time, hide-done toggle, per-task timer/in-progress/remove buttons, and a 📋 Copy button (`copySprint()`) that writes the sprint to the clipboard as markdown (title + done/total/time-left summary, numbered blocks with labels and time estimates, tasks as `- [ ]`/`- [x]` checkboxes with per-task time); respects the hide-done toggle and renumbers visible blocks
- **All Tasks** (`manage`) — full task list with search + owner/room/status filters; add/edit/delete custom tasks, hide base tasks, pin/flag, +Today, edit last-done; month chips in the editor set seasonal windows and a three-way chip row sets the task's vacation decay; shows learned-time and real-cadence hints
- **Presets** (`presets`) — see the Presets section below
- **Stats** (`stats`) — This Week box (count/effort/streak/today), Budget Insight (suggests budget changes from median non-daily throughput, apply-only), 13-week activity heatmap, and full Completion Cadence list with due badges, +Today/flag, "set target Nd" adoption, and tap-name-to-edit. Real cadence (`cadenceInfo()`, colored `~Nd real`) also shows inline on My Hand and All Tasks cards
- **Settings** (`settings`) — budget steppers, active-zone reference table, system overview, scoring explainer, recent-activity log, vacation mode card (pause/resume plus the freeze/half/full counts, with the task lists behind a `<details>` disclosure — the half-speed list alone is 60-odd tasks and swallows the tab if left open), reload-app card (shows `APP_VERSION`), weekly backup card, and export/import (paste or restore-from-file)

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

`Whole House`, `Robot`, `Downstairs`, `Upstairs`, and `Cats` are zone-less rooms — tasks in them don't compete for zone bonuses. (`Cats` tasks instead get their own `-0.5` tiebreaker via `cat: true`.)

## Repository Files

- `index.html` — the entire app (HTML + CSS + JS in one file, ~5960 lines). **Line-number anchors rot within weeks — name the function and grep for it, never cite a line**
- `CLAUDE.md` — this file: working instructions + architecture summary for Claude
- `DOCUMENTATION.md` — full reference documentation of the app (data model, algorithms, every tab, state schema, edge cases)
- `QUEUE.md` — **the living work queue.** What is still open in the code and audit queues, the decisions already made so they are not re-argued, and the standing 141% capacity constraint. Read it before starting anything. It holds only open work — finished items are deleted, not ticked, because git is the record
- `simulate.html` — the day simulator the Behaviour audit needs. Runs the real `dealHand` forward N days against a shimmed clock and reports coverage by tier, the starvation tail, service intervals and budget adherence. Serve over http like the selftest: `…/simulate.html?days=180`
- `AUDITS.md` — the eight audit lenses (Flow, Behaviour, State, Surface, Content, Coherence, Opportunity, Housekeeping), each with its own method, evidence bar, severity scale and traps. Read the relevant section **before** starting any audit or review; the `app-update` skill's Mode B carries the operational side (what to request from her, reproduce-before-reporting, report-don't-fix). Behaviour, Content and Opportunity need a fresh state export to be worth running
- `selftest.html` — the test harness, 63 cases. Serve the folder and open it over **http** (`preview_start` with the `hometasks` launch config, then `http://localhost:7821/selftest.html?cb=N`); it loads `index.html` into an isolated iframe (memory-backed storage, so it can never touch live data) and reports `N/N passed`. **Required green before committing** any change to scoring, dealing, state shape, or presets (pure CSS/copy/task-text changes are exempt). Every case has been mutation-checked — mutate by copying a broken `index.html` into the repo folder and loading `?src=<that file>`. **Don't run it over `file://`**: the preview pane serves a stale snapshot and strips the query string, so both `?cb=` and `?src=` are silently ignored and a mutation check will report a false green

## Known Issues / Watch-outs

The June 2026 review bugs — dead preset task ids (`k_backsplash`, `dsb_exhaust`, `dsb_mop`, `bed_vacuum`), `completeEarlier` double-logging, `applyImport` field gaps, preset name/room drift, and the phantom "Front Porch" zone entry — were all **fixed on 2026-06-15**.

Editing pitfalls that remain true (also encoded in the `app-update` skill):
- **Preset task ids** in lists that render via `getAllTasks()` (`ROOM_PRESETS`, `EXPRESS_RESET_SECTIONS`, `RETURN_HOME_SECTIONS`, `BEFORE_CLEANERS_SECTIONS`, `RECOVERY_SECTIONS`, `POST_ILLNESS_SECTIONS`, `EVENING_*`) must exist in `TASKS`, or they silently vanish; `completeTask` returns early for unknown ids, so a phantom checklist item can never be checked off. `selftest.html` case 3 now pins this.
- **`saveTask` rebuilds a custom task from an explicit field list**, so any new task-level flag must be added there *and* to the editor form, or editing a task silently strips it. `load: true` is the live example: without the `f_load` checkbox, changing a wash task's frequency would have dropped it out of the laundry slot with no visible cause. Case 12 pins the flag set being populated — note it also guards the two laundry cases below it, which go vacuously green when no task carries the flag.
- **`input { appearance: none }` is a blanket reset**, so a checkbox has no visual unless it is drawn by hand. `.field-check` does this; the cat checkbox had been invisible since it was added.
- **Every path that ends a task must clear `state.inProgress[id]`.** `dealHand` carries in-progress tasks into every hand *including when they are not due* (the fountain is in the dishwasher; the task was last completed four days ago), so a leftover flag is a task that returns forever with no way to shake it off. `completeTask`, `snoozeTask`, `removeTask` and `completeEarlier` all clear it; case 17 pins the backdating path, case 16 pins the not-due carry-over that stops you from "fixing" this by dropping stale flags in `dealHand`.

Patterns found by the 2026-08-08 audit, all fixed — but the shapes recur:
- **Two things called "undo".** `undoComplete()` (toast) restores the stored timestamp; `uncompleteTask()` (card button, All Tasks checkbox, Sprint untick) restores `prevCompletions[id]`. Neither may fall back to fabricating `now − freq`: for a 180-day task that moves last-done *backwards* half a year and destroys the real date. Cases 50–51 pin this.
- **A checklist tick must toggle.** Sprint and preset cards route through `toggleComplete`, not `completeTask`. Wired to `completeTask` a mis-tap is uncorrectable once the 5s toast expires, because tapping the ticked card just re-fires the completion. Case 56 pins it by asserting on the rendered HTML.
- **`daysOverdue()` short-circuits on an active snooze** before it reads the completion date, so anything that "makes a task due" by writing `completions` is a no-op while snoozed. `markTaskDue` clears the snooze for exactly this reason. Case 52.
- **Every path that puts a task into today's hand must call `unremoveToday(id)`**, or the next redeal takes it straight back out. `addToHand`, `pinTask`, `swapTask`, `addRandomPickToHand`. Case 53.
- **Re-query a DOM node after any handler that can re-render.** `toggleDatePicker`/`toggleManageDp` save-and-close the open row first; the save re-renders, so the element captured before the loop is detached and opening it silently does nothing. Pair this with `setLastDoneIfChanged` — saving an unchanged date redealt the hand and toasted a phantom edit.
- **`renderCard` has a local `const isSnoozed` boolean** that shadows the global `isSnoozed()` helper. Calling the helper inside `renderCard` throws.
- The selftest's preset-id sweep is now self-maintaining: case 4 scrapes `generateNewPreset('…')` out of the rendered Presets tab and fails if a type is missing from `REGISTERED_PRESET_TYPES`. Tidy Everywhere shipped unchecked because that list used to be hand-kept.

**Retired 2026-08-03 — do not reinstate.** "Adding state requires a default in all three of `defaultState()`, `loadState()`, and `applyImport`" is no longer true. Both paths route through `coerceState()`, which spreads over `defaultState()`, so **`defaultState()` is the only place a new field must be added.** The one addendum is a check, not a default: a field defaulting to `null` can't have its type inferred, so list it in `NULLABLE_SHAPE` if it should be shape-guarded. Forgetting that costs a missed type check, never a dropped field.

**Design rules adopted 2026-08-04 (from the plan's Part 3):**
- **No hardcoded colours in new UI.** Every literal routes through a token so both themes work. Use `var(--on-accent)` for text sitting on a filled accent — never literal white. `--amber-on-inverted` exists for the toast, whose surface is `var(--text)` and therefore flips the opposite way to every other surface.
- **New UI uses classes, not inline styles.** The existing inline styles can be retired opportunistically; what matters is that the count stops growing.
- **`taskTime()` is the single source for "how long will this take."** Raw `task.time` is correct only in the editor field and the learned-time hint.
- **16px on inputs is load-bearing** — iOS Safari zooms the page on focus below it. It is deliberately not part of the type scale.
- **Sheets open and close only via `showSheet()` / `hideSheet()`.** Toggling the `open` class directly loses the exit animation.

Minor known behaviors (by design / low priority):
- Random Task ignores owner, so it can surface a Bob-only task to add to the hand.
- Loading a preset shows a Sprint count whose denominator includes anything else completed earlier today, so a 14-task preset can read "6 / 20". Consistent with My Hand, which also counts the whole day.
- `getOverflowTasks()` filters to `freq > 1`, so a suppressed laundry-process step can't be surfaced through "+ Give me more".
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
