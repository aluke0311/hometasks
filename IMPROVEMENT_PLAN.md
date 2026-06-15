# Improvement Plan — 9 Features (June 2026)

Agreed with Alina on 2026-06-09. Execute in order, one commit + push per feature
(per standing instructions). All features are additive to state — no STORAGE_KEY
bump, all defaults handled in the `loadState()` migration block, so every step
is **✅ No export needed**. Verify each feature with the preview tools before
committing.

## Decisions made

- **Backup:** weekly silent auto-download (no Gist sync).
- **PWA:** skipped entirely — the existing home-screen shortcut already
  auto-updates and behaves like an app.
- **Stats:** new 6th tab.
- **Zone rotation:** FlyLady calendar (day-of-month 1–7 → Zone 1, … 22–28 →
  Zone 4, 29+ → Zone 5).
- **Time tracking:** per-task manual timer (tap to start, check-off stops it).
- **Vacation mode:** manual toggle in Settings only, no preset integration.
- **Seasonal tasks:** add `months` capability AND assign sensible default
  windows to obviously seasonal base tasks.

## Feature specs (in build order)

### 1. Auto-backup (weekly download)
- New state field `lastBackupDate` (ISO date string, default null).
- On app load, after state is loaded: if `lastBackupDate` is null or ≥7 days
  ago, generate the same JSON the manual export produces, trigger a download
  named `hometasks-backup-YYYY-MM-DD.json`, set `lastBackupDate`, save.
- Show a small toast "Weekly backup saved to Downloads" so it isn't mysterious.
- Settings: show "Last backup: <date>" plus a "Back up now" button (reuses the
  same code path; the manual Export button stays as-is).

### 2. Undo + backdating
- First check what exists: there is already uncomplete logic (~line 1105,
  history-filtering in what looks like an uncomplete function). If tapping a
  completed task already un-completes it, the Undo half is done — just add a
  5-second toast with an Undo button after `completeTask` as a convenience.
- Backdating: long-press (and right-click on desktop) on a task row opens the
  existing task action sheet/menu — add "Done earlier…" with options
  Yesterday / 2 days ago / 3 days ago. Sets `completions[id]` to that
  timestamp (noon of that day) and pushes it into `completionHistory` in
  sorted position. Clears starvation for the task.

### 3. Quick-log ("I also did this")
- On the Today tab below the hand: a collapsed "+ Log something else" row that
  expands to a search input with type-ahead over all visible tasks (base minus
  deleted, plus custom).
- Selecting a result logs a completion now via the same path as
  `completeTask` (history, starvation clear, pin clear) but does NOT add the
  task to the hand and does NOT count against budget. Toast confirms, with
  Undo.

### 4. Stats tab
- New 6th tab "Stats" (pick a short label/emoji to keep the tab bar fitting at
  ~390px width; verify with preview_resize).
- Contents, all computed from `completionHistory`:
  - **Heatmap:** last ~13 weeks, GitHub-style grid of completions per day.
  - **This week vs last week:** completion counts + total estimated minutes.
  - **Streak:** consecutive days with ≥1 completion.
  - **Chronically overdue:** tasks whose median actual interval (from history)
    exceeds 1.5× their `freq`, sorted worst-first, showing "every ~Nd
    (target Md)". Only tasks with ≥3 history entries.
- No new state. Pure derived rendering.

### 5. Score breakdown ("Why this task?")
- Refactor `scoreTask` (~line 818) to optionally return its components: base
  freq, overdue penalty (and its cap), starvation reduction (and its cap),
  flagged −50, zone −2, cat boost. Keep the existing call signature working
  (e.g. `scoreTask(task, zone)` returns number; `scoreTaskParts(task, zone)`
  returns the object — implement one in terms of the other so they can never
  drift).
- Add an ⓘ affordance on task rows in Today (and the same long-press menu as
  backdating) opening a small overlay listing each component with its value
  and one-line meaning, plus the total.

### 6. Zone auto-rotation
- New state field `zoneMode: 'auto' | 'manual'`. Migration: default existing
  and new users to `'auto'` (this is the point of the feature), preserving the
  old `zone` value for manual mode.
- Auto zone from day of month: 1–7→1, 8–14→2, 15–21→3, 22–28→4, 29–31→5.
- The zone `<select>` (~line 294) gains a first option "Auto (Zone N this
  week)"; choosing any explicit zone or "No zone focus" sets
  `zoneMode:'manual'`. A helper `effectiveZone()` replaces direct `state.zone`
  reads in scoring/dealing.

### 7. Vacation mode
- New state fields `paused: false`, `pausedAt: null` (timestamp).
- Settings toggle "⏸ Pause (vacation)". While paused:
  - Banner on Today: "Paused since <date> — Resume".
  - No new hand dealt; starvation accrual skipped (find the daily starvation
    update guarded by `starvationDate` and gate it).
- On resume: compute days paused; shift every `completions[id]` timestamp
  forward by that duration (so nothing aged while away), clear
  `paused/pausedAt`, deal a fresh hand. Do NOT shift `completionHistory`
  (it records what actually happened).
- Edge case: completing a task while paused is allowed and records normally.

### 8. Seasonal tasks (months windows)
- Optional task field `months: [1..12]` (absent = year-round). For custom
  tasks it lives in `customTasks`; for base tasks, overrides live in a new
  state field `taskMonths: {taskId: [..]}` so base-task windows survive
  deploys and remain user-editable.
- Dealing: tasks out of season are excluded from the pool entirely (also from
  quick-log? No — quick-log still finds them; you might do them anyway).
  Out-of-season tasks don't accrue starvation. Manage shows a "❄ out of
  season" badge instead of overdue status.
- Manage task editor: a row of 12 toggleable month chips (all-off = year-round).
- Defaults: I assign windows in code (as a DEFAULT_MONTHS map applied via
  migration into `taskMonths` only for tasks the user hasn't customized) for
  obviously seasonal base tasks — read the full TASKS array and pick
  conservatively (gutters, screens, porch/outdoor furniture, AC/heating
  filters if seasonal, etc.). When in doubt, leave year-round. List the chosen
  defaults in the commit body so Alina can review in Manage.

### 9. Per-task timer + learned estimates
- New state field `actualTimes: {taskId: number[]}` (minutes, keep last 10).
- Task rows in Today and Sprints get a small ▶ timer button; tapping starts a
  timer for that task (one active at a time — starting another stops the
  first without recording). A compact running indicator shows elapsed time on
  the row. Checking the task off while its timer runs stops it and records
  rounded minutes (ignore <1 min or >4 h as accidents).
- Learned estimate = median of recorded times. Where `task.time` is used for
  budget filling and display, use a `taskTime(task)` helper: median actual if
  ≥3 samples, else `task.time`. Display in Manage as "~12m (est 10m, n=5)".
- Timer state itself is transient (in-memory only — not persisted; an
  abandoned timer just disappears on reload).

## Execution notes

- The app is one file, [index.html](index.html) (~2950 lines). Key anchors:
  `STORAGE_KEY` 682, `loadState` 696, `scoreTask` 818, `dealHand` 863,
  `completeTask` 1057, `render` 1243, zone select markup ~294. Re-grep after
  each feature since lines shift.
- After each feature: verify in browser preview (load with existing-ish state,
  exercise the feature, check console for errors), then commit + push with the
  conventional title/body, ending the change description with
  "✅ No export needed".
- Don't break: laundry slot logic, preset hands, `_dealPreferTier`, the
  starvation cap. Touch `dealHand` minimally (features 6–8 only filter the
  pool or change the zone input).

---

## Follow-on work shipped — cleaning-schedule research pass (2026-06-15)

Separate from the 9 features above. Driven by a review of current cleaning advice
(frequency guides, weekly-reset routines, spring-cleaning checklists, "commonly
forgotten" lists) versus the existing task model. All **✅ No export needed**.

- **Frequencies tightened** toward expert consensus / pet-ownership norms: bed
  sheets 14→7, towels 14→7, microwave 60→30, washing machine 90→60, living-room
  dusting 14→7, dining dusting 21→10. (Downstairs-bath changes were skipped — that
  bath is rarely used.)
- **12 new tasks** filling common gaps: refrigerator coils; US-bath showerhead
  descale + shower-curtain/liner wash; living-room throw blankets; cat beds &
  blankets; whole-house baseboards + rug shampoo; and a new **Home Maintenance**
  room (furnace filter relocated here, dryer vent & duct, smoke/CO detector test,
  water-heater flush, gutters) intentionally in no zone.
- **Two new presets:** **Weekly Reset** (fixed-section, the classic Sunday reset)
  and **Seasonal Deep Clean** (dynamic — snapshots every in-season Tier C task into
  one room-grouped spring-cleaning checklist).
- **Completion backfill** added to `loadState`/`applyImport` so newly added base
  tasks read as due-now instead of ~20,000 days overdue. This removes the last
  reason adding a base task might need an export.
