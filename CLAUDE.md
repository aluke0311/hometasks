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

The base task list lives in the `TASKS` array (182 entries). Users can add custom tasks and hide base tasks without touching code. Some entries with `custom_…`/`oneoff_…` ids are personal customizations baked into the base array.

### Scoring (`scoreTask`, `scoreTaskParts`)

Lower score = higher priority. `scoreTaskParts` returns the labeled component breakdown (used by the "why?" modal); `scoreTask` sums them and adds jitter. Components:
- Base score = `task.freq` (lower frequency → lower base → higher priority)
- Overdue penalty: reduces score based on how overdue the task is; weight is `0.5` normally, `0.8` in Catch Up. `daysOverdue()` is whole calendar days for every frequency now (fixed 2026-08-22) — the `freq > 1` tail used to measure from the completion instant, so two tasks last done on the same date could show different overdue badges depending on the hour they were ticked. **The cap is mode-dependent (`overdueCap`) and that is the whole difference between the two modes:**
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

**Who gets dealt what** is `dealtByMe(task)` — the single gate, replacing eight longhand owner filters. True for `alina`/`either`; true for a `bob` task only when **`bobAway`** is on or **that task is flagged**. Flagging one of his tasks means "bring me this one" — before that, the flag set a badge and changed nothing, because he has no hand.

**The budget being filled** is `budgetWeekday`/`budgetWeekend` (non-daily time only) *unless* a `todayBudget` is set for today, in which case it is `todayBudget − dailyTime` — the slider is a whole-day total, so the daily load comes off the top. `dailyTime` covers the dailies being forced into the hand plus any already completed today. Only tasks the pool could contain (`alina`/`either`) count as time already spent: letting a Bob-only completion consume the budget silently shrank her hand, since she logs his chores.

**Mode-dependent pool filling:**

| Mode | Pool sort | Fill rule |
|------|-----------|-----------|
| **Keep Up** | Score (overdue weight 0.5, cap `freq*0.8`) | Fill budget in score order; heavy tasks (>15 min) capped at 2 per hand |
| **Catch Up** | Score (overdue weight 0.8, cap `freq-3`) | Same fill, but the absolute floor lets long-neglected tier C tasks reach the top |
| **By Freq** | Raw frequency, shuffle within same-freq | Fill budget in freq order |

If `_dealPreferTier` is set (via redeal with tier button), tasks of that tier float to the top of the pool for that deal only, then the field is deleted.

**Flagged tasks:** enter the pool even when not yet due (flag bypasses `isDue`) and fill **first in every mode** — critical for By Freq, where the raw-frequency sort would otherwise ignore the -50 flag bonus entirely. They count against the budget and dealing stops at the limit, except the first flagged task is always dealt (so flagging guarantees surfacing even when the budget is spent). Flagged tasks that don't fit, and `getOverflowTasks()` generally, surface via "+ Give me more", which includes flagged not-yet-due tasks.

**Cycle slots** (replaced the hardcoded laundry slot and `ALWAYS_ASSIGN_IDS` on 2026-08-17):
- Each entry in `CYCLES` gets at most `dealMax` (currently 1) running at a time. Candidates are the tasks that own that cycle, picked by score.
- **Done today, pinned, or in progress** all mean that cycle has had its turn — the rule the wash slot always had, now applied per definition.
- A cycle the dealer opened **bypasses the budget**, as the wash slot always did and as `c_fountain` used to via its own special case. That case is gone.
- Cycle owners never enter the ordinary pool; they arrive through their slot.
- **Today's pick is dated and sticks** (`cycleChoice` / `cycleChoiceDate`). `redealHand()` nulls the hand, so without this a redeal re-sorted six near-identical wash tasks and today's laundry silently became a different load.

### Cycles (`CYCLES`, `state.cycles`)

A cycle is a multi-step job you start once and come back to. **One task owns the cycle and carries the frequency** ("Wash whites", every 7 days); the steps are entries in `CYCLES` and are **not tasks**. They were tasks in the first cut, which meant four `freq: 1` dailies that `dealHand` had to suppress by hand; those four (`l_start`, `l_dryer`, `l_fold`, `l_put_away`) were **deleted on 2026-08-17** along with their rows on six presets.

Three definitions ship: **laundry** (into the washer → into the dryer → fold → put away), **dishwasher** (load → start → unload) and **fountain** (into the dishwasher → reassemble). The dishwasher and the fountain both wait **129 min**, because the fountain is cleaned in the dishwasher; a selftest case pins the two together.

Each stage carries `action` (what she does), `phase` (what is true *while* that action is pending), `mins` and `readyIn` (machine minutes after the previous step). **The card shows `phase` + a ready time until the wait passes, then switches to `action`** — she will not tap "into the dryer" while it is still washing. `readyIn` never blocks a tick; it only picks which of the two is shown. Overridable per step in the task editor via `state.cycleTimings`, keyed `def.stageKey`.

- **A task declares its cycle with `cycle: '<key>'`.** `load: true` is the retired wash-slot flag and survives only as a read fallback in `cycleDefFor` for stored custom tasks; the editor checkbox is gone, and `coerceState` folds `load` into `cycle` on import.
- **Before a cycle starts**, the card still shows step 0's action and the dots, so ticking "Wash cools" means "into the washer", not "the washing is finished".
- **The owning task completes only when the last step does**, so `freq: 7` measures whole loads. `finishCycle` deletes the cycle then calls `completeTask`.
- **`dealMax` governs the dealer, not a cap** — `startExtraCycle` runs a second load by hand.
- A cycle untouched for `CYCLE_STALE_DAYS` (3) shows "still going?" with **Drop it** and **Finished it**.

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
- `cycles` — `{cid: {def, taskId, stage, startedAt, stageAt, readyAt}}` running cycle instances. `coerceState` **drops any entry whose `def` is unknown or whose `stage` is out of range** — such an entry used to reach `renderCard`, which dereferences `CYCLES[def].stages`, and threw, taking My Hand (the default tab) down with no way back
- `cycleTimings` — `{'<def>.<stageKey>': minutes}` her overrides for machine time
- `cycleChoice` / `cycleChoiceDate` — which task took each cycle's slot today. Deal output, not a preference: dated, expires on its own, and excluded from export round-trip comparisons alongside `hand`/`starvation`
- `bobAway` / `bobAwaySince` — hands Bob's 18 tasks to her until switched back. Their due dates are untouched, so anything overdue arrives overdue
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
- `tasksView` — `'rooms'` (default) | `'flat'`. Which All Tasks render path is shown; flipped in Settings, in one tap, with no deploy
- `tasksRoomSort` — `'pressure'` (default) | `'az'`. Sort order of the room list in `'rooms'` view

### UI Tabs

Six tabs in the bottom nav (internal `currentTab` id in parentheses):

- **My Hand** (`alina`) — the main dealt hand titled "Today's Hand"; header has Redeal (+ A/B/C tier redeal), the zone selector, the three mode pills, the **Today's time slider** (`renderDayBudget`; drag repaints the labels only, the deal happens on release, and "back to usual" clears the override), and "+ one-off task". Cards: check off (undo toast), swipe rail (More / Pin / Remove), long-press, and the ⋯ button — all three open **the task page** (see below). Plus a quick-log search to record completions outside the hand. Sections: Daily Tasks, Other Tasks, Completed, "+ Give me more" / Backup list (due + not-due)
- **Sprint** (`sprint`) — room-grouped, ordered walkthrough of today's hand (Launch → Kitchen → Cat care → Laundry → room groups), per-block time, hide-done toggle, per-task timer/in-progress/remove buttons, and a 📋 Copy button (`copySprint()`) that writes the sprint to the clipboard as markdown (title + done/total/time-left summary, numbered blocks with labels and time estimates, tasks as `- [ ]`/`- [x]` checkboxes with per-task time); respects the hide-done toggle and renumbers visible blocks
- **All Tasks** (`manage`) — search + owner/room/status filters apply at every level; tapping a row opens the task page; shows learned-time and real-cadence hints. `state.tasksView` (`'rooms'` default | `'flat'`) picks the render path, flipped in Settings → "All Tasks opens on" — an experiment (2026-08-22), reversible in one tap:
  - **`'rooms'`** (`renderManageRoomsView`): level 1 is one row per room (`.room-row`) — count, due count, and the room's pressure figure from `roomPressure()` — sorted A–Z or by pressure (`state.tasksRoomSort`). Tapping a room (`openManageRoom`) drills to level 2: that room's tasks in the same row markup as before, with a sticky back header (`backToRoomList`). **The sticky room head's `top` is set in JS to the app header's real `offsetHeight`**, not `0` — both are `position:sticky`, so a room head stuck at the same offset as the app's own sticky header renders half-covered underneath it, and the header's height isn't a fixed number (`env(safe-area-inset-top)`). Search is scoped to the open room, with a "Search everywhere" escape hatch (`searchEverywhere`, leaves the room but keeps the term) shown whenever the term also matches outside it — never a dead end, even when the room match count is zero. The back control (not "search everywhere") clears the search term, so a room's query never leaks back out to the house; it clears the DOM input directly too, since the render re-syncs `manageSearch` from the live box (see below). At level 1 with a search active there is no room scope yet, so results render flat-grouped-by-room, same as `'flat'`.
  - **`'flat'`** (`renderManageFlat`) — the original one-long-list screen, room dropdown and all, kept working unchanged and unforked; only the per-row markup was pulled out into a shared `taskRowHtml(t)` so both paths render one row exactly one way.
  - `manageRoom` (which room, if any, is drilled into) is a plain module variable, not `state` — view position, not a preference. Cleared in `switchTab` on leaving the tab, and by the back control.
  - `syncManageSearchFromDom()` reads the live search box's value into the `manageSearch` module variable before every render. Normal typing already keeps them in step via the input's `oninput`; this covers a render triggered any other way (a filter chip, a room tap, or — the reason it exists — a test setting `.value` directly without dispatching an event) so it honours what is actually in the box.

**The task page** (`#taskModal`, `.modal-backdrop.as-page`) is the one surface for a task, merged from the ⋯ sheet and the editor on 2026-08-17. Before that they were two screens with **no route between them from the hand**, which is why the vacation setting was unfindable in practice while rendering perfectly in the form. Opened by `openModal(id, canSwap)`; `openTaskActions` is an alias. Layout: Mark done, then the actions (cycle steps, **add/remove today** — whichever applies, pin, in-progress, timer, flag, snooze, mark due, swap, **While away: <mode>**, edit last done, did-it-earlier, why this task, remove), then every setting. Add to today (2026-08-22) and Remove from today are mutually exclusive, gated on `state.hand.includes(id)` — the page used to offer removal for every task, including one that was never dealt, and had no way to add a task to the hand at all from its own page.
- **It saves as you type.** No Save/Cancel — that matches the rest of the app, where pins, flags and chips all commit on touch. `writeTaskForm` is the single writer and **refuses a half-typed form**, since it runs on every keystroke. The hand is redealt **once on close**, not per keystroke, or the list moves while she types.
- **Creating keeps an explicit Add button** — a page that saves as you type would otherwise leave a half-named task behind.
- Delete confirms once and says whether the task is hidden (base) or deleted (custom).
- Shows every timed run under the estimate: each figure, the range, the median, and the blended number `taskTime()` actually uses.
- **Presets** (`presets`) — see the Presets section below
- **Stats** (`stats`) — This Week box (count/effort/streak/today), Budget Insight (suggests budget changes from median non-daily throughput, apply-only), Room Pressure (median of days-since-done ÷ target per room, worst first; tap opens that room's Deep preset), 13-week activity heatmap, and full Completion Cadence list with due badges, +Today/flag, "set target Nd" adoption, and tap-name-to-edit. Real cadence (`cadenceInfo()`, colored `~Nd real`) also shows inline on My Hand and All Tasks cards. **Room Pressure's numbers come from the top-level `roomPressure()`** (extracted 2026-08-22), shared with the All Tasks room list — one definition of "which room is worst," so the two screens can't disagree
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

- `index.html` — the entire app (HTML + CSS + JS in one file, ~7000 lines). **Line-number anchors rot within weeks — name the function and grep for it, never cite a line**
- `CLAUDE.md` — this file: working instructions + architecture summary for Claude
- `DOCUMENTATION.md` — full reference documentation of the app (data model, algorithms, every tab, state schema, edge cases)
- `QUEUE.md` — **the living work queue.** What is still open in the code and audit queues, the decisions already made so they are not re-argued, and the standing 141% capacity constraint. Read it before starting anything. It holds only open work — finished items are deleted, not ticked, because git is the record
- `simulate.html` — the day simulator the Behaviour audit needs. Runs the real `dealHand` forward N days against a shimmed clock and reports coverage by tier, the starvation tail, service intervals and budget adherence. Serve over http like the selftest: `…/simulate.html?days=180`
- `AUDITS.md` — the eight audit lenses (Flow, Behaviour, State, Surface, Content, Coherence, Opportunity, Housekeeping), each with its own method, evidence bar, severity scale and traps. Read the relevant section **before** starting any audit or review; the `app-update` skill's Mode B carries the operational side (what to request from her, reproduce-before-reporting, report-don't-fix). Behaviour, Content and Opportunity need a fresh state export to be worth running
- `selftest.html` — the test harness, 111 cases. Serve the folder and open it over **http** (`preview_start` with the `hometasks` launch config, then `http://localhost:7821/selftest.html?cb=N`); it loads `index.html` into an isolated iframe (memory-backed storage, so it can never touch live data) and reports `N/N passed`. **Required green before committing** any change to scoring, dealing, state shape, or presets (pure CSS/copy/task-text changes are exempt). Every case has been mutation-checked — mutate by copying a broken `index.html` into the repo folder and loading `?src=<that file>`. **Don't run it over `file://`**: the preview pane serves a stale snapshot and strips the query string, so both `?cb=` and `?src=` are silently ignored and a mutation check will report a false green

## Known Issues / Watch-outs

The June 2026 review bugs — dead preset task ids (`k_backsplash`, `dsb_exhaust`, `dsb_mop`, `bed_vacuum`), `completeEarlier` double-logging, `applyImport` field gaps, preset name/room drift, and the phantom "Front Porch" zone entry — were all **fixed on 2026-06-15**.

Editing pitfalls that remain true (also encoded in the `app-update` skill):
- **Preset task ids** in lists that render via `getAllTasks()` (`ROOM_PRESETS`, `EXPRESS_RESET_SECTIONS`, `RETURN_HOME_SECTIONS`, `BEFORE_CLEANERS_SECTIONS`, `RECOVERY_SECTIONS`, `POST_ILLNESS_SECTIONS`, `EVENING_*`) must exist in `TASKS`, or they silently vanish; `completeTask` returns early for unknown ids, so a phantom checklist item can never be checked off. `selftest.html` case 3 now pins this.
- **`writeTaskForm` rebuilds a custom task from an explicit field list**, so any new task-level flag must be added there *and* to the editor form, or editing a task silently strips it. `cycle` is the live example and is pinned by a case. (`saveTask` is now only the *create* path; `writeTaskForm` is the single writer.)
- **`input { appearance: none }` is a blanket reset**, so a checkbox has no visual unless it is drawn by hand. `.field-check` does this; the cat checkbox had been invisible since it was added.
- **Every path that ends a task calls `endTaskSideEffects(id)`** — snooze, in-progress, starvation, flags **and cycles**, in one place. Four call sites used to each remember this list, and when `cycles` joined it, `completeTask` and `endTaskAsOf` were not updated: completing a cycling task any way but its last step left the cycle running, so the next day the card read "In the washer" for a finished load and blocked a new one. **Add anything new to the helper, never to a call site.**
- **A field shown both in `writeTaskForm` and as an action row has two writers.** The vacation setting is both; the row moved it and the next keystroke wrote the form's stale `editingVac` back over it. Any second setting promoted to a row inherits this — sync the form's variable in the row's handler.
- **(retired) Every path that ends a task must clear `state.inProgress[id]`.** `dealHand` carries in-progress tasks into every hand *including when they are not due* (the fountain is in the dishwasher; the task was last completed four days ago), so a leftover flag is a task that returns forever with no way to shake it off. `completeTask`, `snoozeTask`, `removeTask` and `completeEarlier` all clear it; case 17 pins the backdating path, case 16 pins the not-due carry-over that stops you from "fixing" this by dropping stale flags in `dealHand`.
- **A duration in milliseconds is not a calendar-day count.** `state.completions[id]` is a wall-clock instant — `completeTask` writes `Date.now()` — so `(Date.now() - ts) / 86400000` floors a completion from yesterday evening down to "today" when read the next morning. Every day-count the user reads must go through `calDaysAgo(ts)`, which compares local midnights (`Math.round`, not `Math.floor`, so a 23/25-hour daylight-saving day still counts as one). Fixed 2026-08-22 in `relativeDay`, `daysOverdue`'s `freq > 1` tail, the week-bar labels, and the rescue-milestone toast. Windows over durations (`pruneHistory`, `budgetSuggestion`, the room-sweep cutoff, `medianRealFreq`) are correctly left as millisecond arithmetic — don't "fix" those.

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

**Fixed 2026-08-17 (do not re-report):** the orphaned-cycle bug on every non-last-step completion route; a stored cycle with an unknown `def` crashing My Hand; the double delete-confirm; copy naming an "Alina tab" / "All Tasks tab" / "My Hand tab"; the `load` + `cycle` double control; every control under 44×44 (Guest Prep radios, form inputs, selects, month chips, snooze chips); preset rows having no route to the task page.

**Still open — see `QUEUE.md`:** the hand's minute totals counting the owner's `time` rather than the current cycle step; Behaviour's B-1 (By Freq services almost nothing over time).

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
