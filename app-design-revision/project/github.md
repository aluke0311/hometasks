repo: aluke0311/hometasks
branch: main
path: index.html

## Last sync
date: 2026-08-09T18:40:00Z

### Updated in this project
- Chose Editorial calm (2a) as the direction and carried it across the whole app (turn 3).
- Built the anti-clutter row system: pristine default row → swipe reveals Snooze/More/Remove → ⋯ sheet holds pin, start, snooze-with-date, edit last-done, remove.
- New screens in editorial calm: My Hand (clean / mid-swipe / sheet), All Tasks, Presets, Stats, Settings, plus one dark My Hand.
- Task states expressed on the row: pinned (hairline pin), overdue (red note), in-progress (Started rule). Line icons only, no emoji.

## Sync history
- 2026-08-09T12:26:00Z — Recreated current My Hand (1a); explored three whole-app directions (Editorial calm, Pro utility, Warm confident).

## Screen map
| Project screen | Repo files |
|---|---|
| Home Tasks redesign.dc.html — My Hand (1a baseline) | index.html (CSS :root + `.header`/`.view-hero`/`.stats-row`/`.task-card`, `renderAlina`/`renderCard`/`renderDayBudget`) |
| Home Tasks redesign.dc.html — directions 1b/1c/1d | New designs derived from the 1a recreation |
| Home Tasks redesign.dc.html — turn 3 (3a–3h, full app in Editorial calm) | index.html (task features: pinnedIds, removedToday, snoozed, inProgress, dealHand, completeTask; ZONE_ROOMS; budgetWeekday/Weekend) |
