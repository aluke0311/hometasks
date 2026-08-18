# Work Queue

Living document. Not a plan and not a record — it holds only what is **still open**.
Delete a line when it is done; do not tick it. Git history is the record.

Last touched: 2026-08-17 · app at `2026-08-17 v9` · selftest 111/111 green.

---

## How to pick up work here

1. Read `CLAUDE.md` first — architecture and the watch-out list.
2. Read `AUDITS.md` section for whichever audit you are running. Each has its own
   method and evidence bar. They are deliberately different.
3. The `app-update` skill carries the workflow: Mode A for changes, **Mode B for
   audits**, Mode C for close-out.
4. **Line numbers rot within weeks.** Grep for the function name, never trust an anchor.

Three tools, three jobs:

| Tool | Answers | Run it over |
|---|---|---|
| `selftest.html` | Did I break a rule? | `http://localhost:7821/selftest.html?cb=N` |
| `simulate.html` | What happens over a year? | `…/simulate.html?days=180` |
| The app itself | Does it look and feel right? | `…/index.html?v=N` at 375px, both themes |

`preview_start` with the `hometasks` config serves all three. **Never open them over
`file://`** — the query string is stripped and cache-busting silently fails.

---

## Code queue

### C-9 — cycles: what is left open
All three cycles shipped 2026-08-17 (laundry, dishwasher, fountain). Stages are no
longer tasks — the four `l_*` steps are deleted, and `ALWAYS_ASSIGN_IDS` is gone with
them. What remains open:

- **Stage times no longer learn.** `actualTimes` fed `taskTime()` for the four step
  tasks; a stage's `mins` is now a constant in `CYCLES`. The timer still works on the
  owning task, so a load can be timed end to end, but "fold" specifically cannot.
  Deliberate — the alternative was keeping four scheduled tasks nobody wanted.
- **Stats lost the per-step rows.** "Fold laundry" was a real completion history and
  is not recorded any more. Only whole loads are. This is the same trade as above and
  it is the reason her washing cadence now reads truthfully.
- **The hand's minute totals still count the owner's `time`, not the current step's.**
  Unchanged from the first cut: `renderCard` and the budget use the step, the nine
  totalling sites in `renderAlina`/`renderSprint`/`copySprint` do not. **Do not fix it
  by making `taskTime()` cycle-aware** — it feeds the editor field and the learned-time
  hint.
- **`readyIn` still never blocks a tick.** It now picks which of phase/action the card
  shows, which is the useful half. A hard gate would still be the app's first sub-day
  due rule.
- **Presets lost their laundry rows.** Six presets no longer mention laundry at all. If a
  reset should start a load, that needs a checklist row that opens a cycle — a new
  interaction, not a restored id.
- **`dealMax` is 1 everywhere.** "Start another" covers the real case by hand.

### C-11 — the task page, with a keyboard up
Verified at 390px in both themes; **not** verified with a soft keyboard raised. The `--kb`
visual-viewport handling was written for a bottom sheet whose margin moved, and the page is
~2000px tall with a sticky header. One pass on her phone with the name field focused.

*(The two-writers rule this created now lives in `CLAUDE.md`'s watch-out list.)*

### C-10 — Bob's away: the numbers are unmeasured
Shipped 2026-08-17. The mechanism is pinned by four mutation-checked cases; what is
**not** measured is what his list does to her hand over time. The card states
**58 min/week** of non-daily demand from his 18 tasks. Against her real 30/60 budget
that moves utilisation from **141% to 163%** — arithmetic, not simulation.

`simulate.html` has no `bobAway` scenario. Adding one is the honest way to answer
"what falls off while he is gone", and it is the same question the standing capacity
constraint below already poses. Until then, treat 163% as a projection.

Also unresolved, and small: the Settings **system overview** still counts
`alinaCount`/`dailyMin` from static ownership, so those two numbers do not move when
his list is hers. Defensible — they describe the task list, not today's pool — but it
is the only place `dealtByMe` is deliberately not used.

### C-7 — reduce the inline styles · S4 · **at a stable floor**
**90** inline `style="` attributes remain (219 → 211 → 184 → 145 → 104 → 92 → 90). Every
shared component has been named. What is left does not want a class:

| Count | What | Why it stays |
|---|---|---|
| 18 | dynamic — `'…' + value + '…'` | Carries data into the markup: a bar's height, a meter's width, the colour that says how far off pace a task is. A stylesheet cannot hold a per-row value. |
| 6 | `display:none` | Toggled by JS. |
| ~52 | one-off static | A class named after its single use is renaming, not structure. |

The rule stands: **the count must not grow, and new UI uses classes.** Converting more is
only worth it when a *second* site wants the same rule.

### C-8 — the residual off-scale spacing · S4 · needs a decision, not a sweep
The exact-match sweep is **finished**: every spacing declaration whose values sit on
`--s-1..--s-10` now uses the token, including inside `calc()` and `env()`. **44
declarations remain**, all genuinely off the scale:

| Value | Sites | Verdict |
|---|---|---|
| 13px | 6 | **The open question.** Six sites is a de-facto step between `--s-6` (12) and `--s-7` (14), not an accident. Either add a token or snap them — both are a real pixel change. |
| 18px | 3 | Card padding (`.settings-card`, `.ed-card`, `.sprint-block`). Consistent with itself. |
| 5, 9, 11, 15, 22, 24, 36, 40, 60px | 1–3 each | One-offs. |
| −1, −6, −10px | 4 | Not spacing: a hairline overlap, two hit-area bleeds, and the slider thumb offset, which is derived from the thumb/track geometry. Leave as literals. |

C-2's note sanctions snapping 3/7/11px to a neighbour. Two such sites survive
(`.sheet-group-label` 7px, `.sheet-chips > button` 11px); they were left out of the
sweep only to keep it a provable visual no-op.

**A second open spacing question, same shape as the 13px one.** The Generate button's
bottom gap is `14px` at eleven call sites and `10px` at three (`.add-btn.wide.gap-b` vs
`.gap-b-sm`). It reads as drift, but both are preserved and named rather than resolved.
Deciding it is one line.

**Not drift, despite looking like it:** the Rooms preset cards sit on an 8px rhythm
against the 4px of a card that opens a category group. That call site is inside
`ROOM_PRESETS.forEach`, so it is fourteen cards in a uniform list, not one card in the
group-opening position — `.settings-card.room-card` exists to say so. It was unified to
4px once by mistake and reverted.

**Not in the queue, deliberately:** the `16px` on inputs. iOS Safari zooms the page on
focus below it. It looks like an inconsistency and must stay.

---

## Audit queue

Run order is roughly judgment-density. `AUDITS.md` holds the full spec for each.

| # | Audit | State | Needs from her |
|---|-------|-------|----------------|
| 1 | Flow | **Blocked again** (2026-08-17) — harness | A working input path, or her walking it |
| 2 | Behaviour | **Re-run 2026-08-16** — 3 findings, unfixed | Her call on B-1 (By Freq) |
| 3 | State | **Re-run + fixed** (2026-08-17) — 2 found, 2 fixed | — |
| 4 | Surface | **Done** (2026-08-16) — 7 found, 7 fixed | — |
| 5 | Content | **Fully run** (2026-08-17) — 6 found, 5 fixed | An export for calibration |
| 6 | Coherence | **Re-run** (2026-08-17) — 3 found, 2 fixed | — |
| 7 | Opportunity | **Run 2026-08-18** — 10 live, 5 killed | Her ranking |
| 8 | Housekeeping | **Done** (2026-08-16) | — |

### 1. Flow — **cannot be run from here.** Three input paths tried, all closed.

Not a finding about the app. Retried 2026-08-17 with the pane reopened and confirmed
**displayed**, which disproves the earlier hidden-pane theory.

| Path | Result |
|---|---|
| `left_click`, by coordinate and by `ref` | 30s timeout; tap verified never to land |
| `key` — **Tab** | **works.** Focus walks real controls in a sensible order |
| `key` — Return / Enter / space on a focused `<button>` | dispatched, but **never activates** it |
| `hover`, `screenshot`, `read_page`, `javascript_tool` | all work |
| Real Chrome (`claude-in-chrome`) | **no browser connected** — extension not attached to the account |

So focus can be moved but nothing can be *activated*, by pointer or by keyboard. The page is
responsive throughout and the console is clean.

**Do not retry these three.** Four attempts across two sessions. A fifth costs minutes and
yields nothing new.

**Two ways it becomes runnable:**
1. **She connects the Chrome extension.** `claude-in-chrome` is separate tooling with its own
   input implementation and is the most likely to work. `list_connected_browsers` returning
   `[]` is the only thing stopping it.
2. She walks the ten journeys herself and reports where she stalled. Journey 10 (the escape)
   first.

**One incidental positive.** Tab order was checked while testing and walks the header, the
mode pills and the cards in visual order without traps. That is not a Flow pass — Flow is
about thumbs — but it is worth knowing the app is keyboard-navigable.

### 1b. Reachability — **new lens, run 2026-08-17.** 2 findings

Proposed as its own small lens rather than smuggled under Flow. Method: render each tab and
overlay, enumerate every on-screen control and its handler, and ask whether each destination
has a route. It would have caught this session's real defect (no route from the hand to the
task editor), which Flow's absence let through for months.

**R-1 · S3 · A task on a preset checklist has no route to its own page.** Preset rows are
`.sprint-card.preset-row`; the swipe and long-press delegation binds to `.task-card`
(`index.html`, the two `closest('.task-card')` sites in `initSwipe`). So a preset row has no
long-press, no swipe rail and no ⋯. It carries three inline buttons — add to today, flag, pin
— out of the thirteen the task page offers. Snooze, "did it earlier", the timer, "why this
task", edit last done and the vacation setting are all unreachable from a preset. Reproduced
by generating Weekly Reset: 26 rows, none a `.task-card`. Same shape as the bug fixed today.

**R-2 · S3 · Three controls are under the 44×44 tap target**, against Surface's recorded
"0 controls short". All three are the Guest Prep radio labels — `label.opt-label` at
**155×20, 91×20 and 84×20**. Measured across all six tabs including a rendered preset; every
other control clears 44×44. This is exactly the case Surface's fix warned about: a new small
control must join the two `::before` selector lists or it silently keeps its drawn size.

**Checked and cleared — do not re-report these:**

| Suspicion | Why it is not a finding |
|---|---|
| Bottom-nav buttons unnamed | `read_page`'s tree does not resolve the label span; they have visible text and an accessible name |
| Guest Prep radios unlabelled | They sit inside `<label class="opt-label">`, so the text is clickable and names them |
| Preset action buttons drawn at 26px | Effective hit area measures **44×44** via `::before`; the CSS size is not the target |
| Stats has no route to the task page | The route exists (`cad-name` → `openModal`); the cadence list needs ≥3 completions and the preview fixture has none. **Fixture limit, not a defect** |
| Task page has no backdrop-tap exit | Correct for a full page; it keeps a ✕ in a sticky header |

**Method note.** Three of those five started as findings and died on contact with a
measurement. Two came from reading CSS or an accessibility tree instead of measuring the
rendered element. Measure, then report.

### 2. Behaviour — **re-run 2026-08-16.** Numbers refreshed; 3 findings, unfixed

180 simulated days × 7 scenarios, against the **default pool**, **two independent draws**.
Every aggregate below is draw 1 with draw 2 in brackets where it differs. Read the caveats.

**Coverage — share of her pool dealt at least once**

| Mode | Policy | Budget | All | A | B | C | Never dealt |
|---|---|---|---|---|---|---|---|
| byfreq | realistic | **30/60** | **27%** (27) | 100% | 5% | 0% | **123** (123) |
| maintenance | realistic | 30/60 | 54% (55) | 100% | 77% (80) | 0% | 77 (75) |
| catchup | realistic | 30/60 | **81%** (81) | 100% | 100% | **48%** (48) | 32 (32) |
| byfreq | realistic | 45/90 | 38% (38) | 100% | 33% | 0% | 105 (105) |
| maintenance | realistic | 45/90 | 71% (71) | 100% | 100% | 23% (21) | 48 (49) |
| maintenance | compliant | 90/120 | **100%** (100) | 100% | 100% | 100% | **0** (0) |
| maintenance | sporadic | 45/90 | 65% (67) | 100% | 98% | 6% (13) | 59 (55) |

**Starvation (days due but not dealt, end of run) · Budget · Repeat rate**

| Mode | Policy | Budget | Median starv. | ≥100 days | vs budget | Repeat |
|---|---|---|---|---|---|---|
| byfreq | realistic | 30/60 | **180** (180) | 123 | −1.5 | **74%** (73) |
| maintenance | realistic | 30/60 | 27 (27) | 77 | −1 | 54% (54) |
| catchup | realistic | 30/60 | 14.5 (13.5) | 46 | −2 | 42% (42) |
| byfreq | realistic | 45/90 | **180** (180) | 105 | −1 | **78%** (76) |
| maintenance | realistic | 45/90 | 8 (9.5) | 48 | −1 | 64% (64) |
| maintenance | compliant | 90/120 | 0 (0) | 0 | −35 | 0% (0) |
| maintenance | sporadic | 45/90 | 11 (10) | 59 | −1 | 76% (75) |

**B-1 · S2 · By Freq services almost nothing over time, and this is in tension with a
recorded decision.** At her real budget it deals 27% of the pool in 180 days and leaves
**123 tasks never dealt once**; its median task's starvation counter ends at 180, meaning
the median task was never dealt at all. Repeat rate 74–78% — it re-offers the same short
list. Worst service intervals, all against a 7-day target: *Tidy back porch* **12.0×**
(~84 days), *Dust surfaces (living room)* 5.0×, *Sweep stairs* 3.9×. *Kitchen tidy* is a
**daily** task running at 2.0×. The recorded decision "By Freq stays — 14 routine tasks
against Keep Up's 6" is not wrong, but it measured **one morning's hand on her real state**;
this measures **service over 180 days on the default pool**. Both can be true: By Freq fills
today's hand best and starves everything outside its front rank. Worth putting to her as a
trade-off rather than treating either number as the answer.

**The aggregate survives a second draw; the per-task ranking does not.** 27%, 123 never
dealt and a median starvation of 180 came back identical. But *Tidy back porch* was 12.0× in
draw 1 and **3.9× in draw 2** — the worst-offender ordering is unstable and no single task's
ratio should be quoted. What repeats across both draws is the shape: several 7-day tasks run
at **3–5×** target (*Dust surfaces (living room)* 5.0× / 5.1×, *Clean stovetop* 3.0× / 3.1×,
*Tidy up (bedroom)* 2.5× / 2.0×), and *Kitchen tidy* — a **daily** — runs at exactly **2.0×
in both**. That last one is the durable finding in this table.

**B-2 · S3 · Tier C is 0% in every mode except Catch Up.** Confirmed as designed — but at
her real 30/60 budget, Keep Up also reaches 0% and only Catch Up gets to 48%. Raising the
budget to 45/90 moves Keep Up's tier C to 23%, so the budget is the binding constraint, not
the scoring. Consistent with the standing 141% utilisation note.

**B-3 · S4 · The dealer never overshoots the budget.** −1 to −2 minutes against target in
every realistic scenario, so budget adherence is not a problem anywhere. Under
compliant/90–120 it undershoots by 35 because the pool runs out of due work — and that same
scenario reaches **100% coverage with zero starvation**, which is the useful result: the
algorithm is sound, and everything above is a capacity problem.

**Caveats — do not quote these percentages without them.**
- **`simulate.html` has no seed.** `?seed=` is ignored; the `freq > 60` jitter is unseeded
  `Math.random()`. The audit's evidence bar asks for "a seeded, reproducible run, plus the
  seed", so **the harness cannot currently meet its own bar**, and the standing instruction
  to "run a second seed" was never satisfiable. Adding a seeded PRNG is the first fix here.
- **Two draws, unseeded.** Both completed and agree on every aggregate; the per-task service
  ranking does not repeat (see B-1). Because there is no seed, neither draw is reproducible —
  "it agreed twice" is the strongest claim available until a PRNG is added.
- **Default pool, not hers.** Her hidden tasks and edited frequencies are absent, which is
  exactly what makes B-1's tension with the recorded decision unresolvable from here.
- **The `sporadic` scenario is pathologically slow** — minutes, and on the second run it had
  not finished after ~15. Under 3-days-on/4-off nearly everything ends up overdue, so the
  candidate list `dealHand` sorts grows every day. Budget for it, or cut the scenario when
  iterating.

### 3. State — **re-run 2026-08-17** after the cycle work. 2 findings, unfixed

Re-run because the session added five state fields (`cycles`, `cycleTimings`, `cycleChoice`,
`cycleChoiceDate`, `bobAway`/`bobAwaySince`) and a save-as-you-type path. The method's core
probe is the writer table: enumerate every writer of `state.completions` and compare what each
does to the satellite state a completion implies. **`cycles` is a new satellite and two writers
do not touch it.**

| Writer | inProgress | snoozed | starvation | flagged | history | pins | hand | **cycles** |
|---|---|---|---|---|---|---|---|---|
| `completeTask` | yes | yes | yes | yes | yes | yes | yes | **— S-1** |
| `endTaskAsOf` | yes | yes | yes | yes | yes | yes | yes | **— S-1** |
| `finishCycle` | via completeTask | | | | | | | yes |
| `undoComplete` | yes | — | — | — | yes | — | — | yes |
| `uncompleteTask` | — | — | — | — | yes | — | — | — |

**S-1 · S2 · Completing a cycling task by any route except its last step orphans the cycle.**
`finishCycle` deletes the cycle and then calls `completeTask`, so the normal path is clean. Every
other route — the task page's **Mark done**, a preset tick, a Sprint tick, quick-log, and
"did it earlier" via `endTaskAsOf` — completes the task and leaves the cycle open. Reproduced,
with the consequence, which is what makes it worth fixing:

```
start a whites cycle        → stage 1, card reads "In the washer"
tap Mark done on the page   → completions written, cycle STILL OPEN
roll to tomorrow            → card reads "In the washer" again, for a load already finished
                            → it is in the hand, and it BLOCKS a new load being dealt
```

The laundry slot stays jammed until she finds "Drop it", and nothing connects yesterday's
Mark done to today's phantom. Fix: clear `cyclesForTask(id)` in the same place `completeTask`
clears `inProgress`, and in `endTaskAsOf`. **The rule to write down is the one that already
exists for `inProgress`: every path that ends a task must end its cycle too.**

**S-2 · S2 · A stored cycle naming an unknown definition crashes the hand.** `renderCard` does
`cycDef.stages.map(...)` without checking that `CYCLES[cyc.def]` resolved, so a cycle whose
`def` no longer exists throws and takes `renderAlina` down with it. My Hand is the default tab,
so the app opens broken with no in-app way back.

```
state.cycles = { cy_old: { def:'laundry_v1', … } }
dealHand()    → ok
renderCard()  → THREW: Cannot read properties of undefined (reading 'stages')
renderAlina() → THREW (same)
```

Reachable from any export written before a cycle key is renamed or removed — three keys were
created today and more are queued in C-9. Fix is either a guard in `renderCard` or, better,
`coerceState` dropping cycles whose `def` is not in `CYCLES`, which also cleans the stored blob.
A stage index past the end of a real definition is already handled safely.

**Passes worth not re-deriving.** All four new fields survive an export → import round trip;
malformed types are rejected (`cycles:'nope'` → object, `bobAwaySince:'soon'` → null). Zero
orphans across `completions`, `taskVac`, `actualTimes`, `taskMonths`, `starvation`,
`cycleTimings`, `pinnedIds`, and no cycle points at a missing task. `cycleChoiceDate` expires
on its own and re-dates. A cycle started yesterday survives midnight byte-for-byte, stays in
the hand, and accrues no starvation.

**Still true from 2026-08-16, and must not regress:**

- **Every path writing `state.completions` to TODAY must call `stashPrevCompletion(id)`.**
- **`dealHand`'s daily starvation tick hides missing resets** — pin `starvationDate` to today
  in any test of it.
- **Vacation mode must survive midnight** (`refreshIfDayChanged` returns early when paused).
- **`applyImport` validates shape, not truthiness**, and reports counts.
- **A completion whose task was deleted still counts** (`ORPHAN_ROOM`).

### 4. Surface — **run and fixed 2026-08-16.** Nothing open.

Trend numbers: **90** inline styles (219 → 211 → 184 → 145 → 104 → 92 → 90) · **34**
distinct type triples · **0** contrast failures in either theme (was 5 light, 19 dark) ·
**0** controls short of a 44×44 tap target (was 539 instances / 76 shapes).

Invariants the fixes leave behind:

- **Tap targets are a transparent `::before` box, not a resize.** Controls keep their drawn
  size; the pseudo-element gives them 44×44. Use `::before` — `.toggle` and `.field-check`
  already own `::after`. A new small control must join the two selector lists or it silently
  keeps a 26px target.
- **`select` and `input` cannot use it** — pseudo-elements on replaced elements are
  unreliable, so `.zone-select`, `.room-select` and `.search-input` take `min-height`
  directly. This is the one part of the fix with a visible consequence: the mode strip grew
  from 55px to 69px.
- **`.task-swipe button` is (0,1,1).** A bare `.sw-today` override is (0,1,0) and loses
  silently. Rail overrides must be written `.task-swipe .sw-*`.
- **Three rail fills stay dark in both themes** (`--slate-strong`, `--amber-strong`,
  `--red-strong`) so their literal `#fffdf8` label keeps working. `.sw-today` is the
  exception — its fill is `var(--green)`, which lightens, so it pairs with `--on-accent`.
- **`--text3` is the lighter of the two quiet greys and must stay that way.** It cannot be
  darkened far enough to clear AA on `--bg3` without inverting the hierarchy against
  `--text2`; it is not used on `--bg3`, and that is why.
- **A `<summary>` sets `--text3` for its chevron**, so anything inside it needs its colour
  restored or it reads as disabled rather than collapsed.

**Method trap, cost four false findings on two separate runs.** Changing the emulated colour
scheme *without reloading* leaves elements resolving old foreground colours against new
backgrounds — the active tab measured 2.00:1 where a reloaded page measures 7.96:1. Reload
after every theme switch, then measure. Same shape as the simulator producing findings about
itself.

**Left for Coherence, not a Surface failure:** 34 distinct type triples, whose near-duplicates
are all on the **line-height** axis — 13px/400 resolves five different ways, 12px/400 four.
Surface produces the inventory; whether 34 should be 34 is audit 6's question.

### 5. Content — **fully run 2026-08-17.** 6 findings, unfixed

#### Pool economics — recomputed after the cycle work

Seasonal tasks prorated by active months, per the spec's trap.

| Quantity | Before today | Now | Note |
|---|---|---|---|
| Pool size | 186 | **182** | four `l_*` laundry steps deleted |
| Whole-pool demand | 113.8 min/day | **85.8** | |
| Alina + either | 102.5 | **74.5** | |
| — of which daily (`freq ≤ 1`) | 48.0 | **20.0** | the 28 min/day of forced laundry steps is gone |
| — non-daily | 54.5 → 382 min/wk | **54.5 → 382 min/wk** | **unchanged** |
| Utilisation @ 30/60 (her real budget) | 141% | **141%** | unchanged |
| Utilisation @ 45/90 (default) | 94% | **94%** | unchanged |

**The headline did not move, and that is the finding.** Every task deleted was a `freq: 1`
daily, and dailies do not compete for the non-daily budget. So the standing 141% constraint
survives the cycle work untouched — nothing was bought there.

**What did improve is the daily fixed cost: 48 → 20 min/day.** Under the Today's-time slider,
which is a whole-day total, that is 28 minutes a day returned. The cycles do not give it all
back: an open cycle charges only its *current* step (3–10 min) rather than all four at once,
and only while it is running. **This is the real win from the cycle work and it is invisible in
the utilisation figure.**

Tier balance **A 42 / B 73 / C 67** (was 46/73/67 — the four deleted steps were tier A).
Owners **26 alina / 138 either / 18 bob**. Eight tasks now own a cycle.

#### C-5 · S3 · "Dishwasher" no longer distinguishes itself from "Clean dishwasher"

Introduced 2026-08-17: `k_dishwasher` was renamed from *"Unload dishwasher"* to *"Dishwasher"*
to serve as the cycle owner. The Kitchen now holds **Dishwasher** (1d), **Clean dishwasher**
(90d), **Clean dishwasher filter** (30d) and **Wipe outside of dishwasher** (60d). In the hand
the card shows its step ("Load it") so it reads fine; in All Tasks it is a bare "Dishwasher"
next to "Clean dishwasher" and the two do not separate at a glance. A name like *"Run the
dishwasher"* would.

#### C-6 · S4 · Two overlap candidates worth her eye

- **"Clean trash can" and "Sweep under trash can"** — Kitchen, both **42d**, same object. Same
  frequency and same trip; plausibly one task.
- **"Deep clean robot"** (90d) against **"Clean robot filters"** (30d), **"Wipe robot sensors"**
  (30d) and **"Clean dock water filter"** (60d), per unit. A deep clean plausibly contains the
  other three; if it does, doing it should reset them.

**Method note: automated overlap detection does not work on this pool.** Name-similarity
scoring returned 65 then 47 candidates, almost all false — the naming convention puts the room
and the object into every name, so *"Tidy up (US bath)"* matches every other *"(US bath)"* task
on the word "bath". Both findings above came from reading the list. Do not re-run the detector
expecting signal; either read the 182 names or skip the check.

#### Not run — needs a fresh export

**Estimate calibration** (learned median vs static `time`, ≥3 samples) and **frequency
plausibility** (declared `freq` vs real cadence from `completionHistory`) both need her real
data. The preview fixture has neither. These are the two checks most likely to change a
frequency, so they are the reason to ask for an export next time.

#### Copy sweep — run 2026-08-17

Swept: 47 toasts, 14 action-sheet rows, 13 empty states, 18 Settings card titles and 29
descriptions, the nav labels and the mode labels. Numbers half was already done.

**C-1 · S3 · Delete asks twice, and the second question contradicts the first.** Reproduced
against the running app by tapping Delete on a base task: `confirmDeleteTask()` asks *Hide
"Clean stovetop"? It is a built-in task, so it is hidden rather than deleted. Its history is
kept.* — then `deleteTask()` immediately asks *Delete this task?*, and the toast says **Task
deleted**. The task is hidden and its history IS kept, so the second prompt and the toast are
both false. Introduced 2026-08-17 when the confirm was added without removing the old one.
Fix is two lines: drop the inner `confirm`, and make the toast say hidden or deleted to match.

**C-2 · S3 · Copy points at tabs that do not exist — three instances.** Nav reads
`Hand · Sprint · Tasks · Presets · Stats · Settings`. The Sprint empty state says *"check the
**Alina** tab first"*, and the Vacation Mode card says *"Change any task in its editor on the
**All Tasks** tab"*, and the Active Zone card says *"Override from the **My Hand** tab"*.
All three reproduced on screen. The second is also stale in substance: that setting is now on
the task page, reachable from anywhere.

**C-3 · S4 · Three phrasings for the same act.** Adding to today says *"Added to today!"*,
*"Added to today's hand"*, and *"Added N more tasks"* from three call sites.

**C-4 · S4 · Two strings for an empty search.** *"No tasks match."* in quick-log versus
*"Nothing matches those filters."* in All Tasks.

**Checked and clean:** voice is consistent second-person plain English throughout; mode names
(`Keep up` / `Catch up` / `By freq`) are used identically everywhere they appear; most empty
states already say what to do next.

**A trap, not a finding.** `showToast` uses `textContent` when no undo action is passed and
`innerHTML` when one is. The what's-new toast contains a literal `&mdash;` and renders
correctly **only** because it passes an action. Verified both branches: without an action the
same string renders as `Updated &mdash; 3 changes`. Any new toast copying that phrasing without
an action will show the entity raw.

### 6. Coherence — **re-run 2026-08-17.** 3 findings, unfixed

**(a) Constants — both forks closed substantially since the last run.**

| Constant | Last run | Now | Direction |
|---|---|---|---|
| `font-size` via token / literal | 151 / 19 | **121 / 8** | distinct literal values **10 → 2** |
| — the literals that remain | `13.5px`, `13px`, `12.5px`… | `16px` ×7 (deliberate), `9px` ×1 | the three-sizes-nobody-chose problem is **gone** |
| `border-radius` via token / literal | 22 / 58 | **47 / 28** | distinct corner values **11 → 7** |
| Inline `style="` | 90 | **90** | holding at the floor |
| Lines | 5,719 | **7,009** | |

Nothing to raise here except a single stray `9px` font-size. Corners are still the weaker of the
two but token use has more than doubled.

**(b) Behaviours that forked.**

**CO-2 · S3 · `load` and `cycle` are two controls for one meaning, and they can contradict.**
Created 2026-08-17. `load: true` was the wash-slot flag; the cycle work made it a legacy alias
—`cycleDefFor` reads `task.cycle || (task.load ? 'laundry' : null)`. Both are now saved by
`writeTaskForm`, and **both appear on the task page as separate controls**: a Cycle picker
offering "Laundry (4 steps)" and a checkbox reading "Laundry load (competes for the one wash
slot)". Reproduced — a task with `cycle:'dishwasher'` **and** `load:true` resolves to
dishwasher, so the ticked checkbox is simply a lie on screen.

Decided or accumulated? **Decided at the time** — `load` was kept so existing data and the
existing checkbox kept working. But the reason has since evaporated: `cycle` is in the field
list and the picker exists. Converging means a one-line migration (`load → cycle:'laundry'`),
deleting the checkbox, and pointing `SPRINT_LOAD_IDS` at `cycleDefFor`. Cheap, and it removes
a user-visible contradiction.

**CO-3 · WITHDRAWN — the fork does not exist.** I reported it from a grep that counted a
pattern including function names that are already gone. Checked properly:
`clearLegacyPreset`, `loadPresetIntoHand`, `loadPresetDueOnly` and `generateLegacyPreset`
have **0 references**; `state.guestHand` / `resetHand` / `goingOutHand` are **never read**;
there is no `renderNewPresetChecklist` competing with `renderPresetChecklist`. Only the
generic path survives, exactly as `CLAUDE.md` said it did when the legacy presets were
retired on 2026-08-16. The three fields remain in `defaultState()` for migration safety and
are always `null` — that is documented and deliberate.

**The lesson, which is the same one this session already learned twice:** a grep count is not
evidence. `AUDITS.md` now says to measure rather than read for Reachability; it applies here
too — Coherence's own evidence bar asks for the diverging sites *side by side*, and had I
printed them I would have seen four of the six were empty.

**Converged since the last run, worth recording as progress:** the six near-identical section
collapsers are now **one** `toggleSection`. And today's work converged rather than added: eight
longhand owner filters became `dealtByMe`, the ⋯ sheet and the editor became one page, and
`writeTaskForm` is the single writer for the task form.

**(c) Fixes that patched the symptom.** The `CLAUDE.md` watch-out list has **grown** this
session, which is the wrong direction. Two new rules now held together by memory alone:

- *"Every path that ends a task must end its cycle too"* — not yet written down, and already
  violated twice (State's S-1).
- *"A field shown both in `writeTaskForm` and as an action row needs a sync in the row's
  handler"* — the vacation setting desynced the moment both existed.

Both are the same shape as the standing `inProgress` rule they sit beside. The structural fix
for all three is one lifecycle function that ends a task — clearing `inProgress`, snooze,
starvation and cycles together — instead of four call sites remembering.

**(d) Things that stopped being true.**

**CO-1 · S2 · `CLAUDE.md` describes an app that no longer exists.** It is the first thing the
workflow tells a new session to read, and after today it is wrong in ways that would cause real
mistakes:

| Claim | Reality |
|---|---|
| "`TASKS` array (186 entries)" | **182** |
| "~5960 lines" | **7,009** |
| "selftest.html — 63 cases" | **103** |
| A whole section on `LAUNDRY_LOAD_IDS` and the one-per-day wash slot | replaced by per-definition cycle slots |
| "Process steps (`l_start`, `l_dryer`, `l_fold`, `l_put_away`) are daily tasks suppressed unless…" | those four tasks **no longer exist** |
| Nothing about `CYCLES`, `dealtByMe`, `bobAway`, the merged task page, or save-as-you-type | five new state fields and three new subsystems undocumented |

A session that trusted it would look for functions that are gone and reason about a laundry
slot that has been replaced. **The fix is a Mode C close-out**, which is a separate job from
auditing and is the single highest-value thing left in this queue.

Not reported: `openRoomPressurePreset` was flagged as an outlived name in the last run, but
"Room Pressure" is a current Settings card, so the name is accurate now.

---

### 7. Opportunity — **run 2026-08-18** on her real export. 10 live ideas, 5 killed

Evidence base: 945 completions across 70 active days, 126 tasks with history, 11 tasks with
timed runs, her live pool of 177.

**The three numbers everything below leans on:**

| Measurement | Value |
|---|---|
| Real cadence vs declared target, 52 tasks with ≥3 completions | **median 1.60×** (worst 3.8×) |
| Timed runs vs static estimate, 11 tasks | **median 0.67×** — she is *faster* than her estimates |
| Live non-daily tasks with **no completion history at all** | **68 of 177** = 58 min/wk = 14% of demand |
| Her non-daily demand vs her actual 45/90 budget | 427 vs 405 min/wk = **106%** |

**Correction to the standing constraint:** the queue's "141% utilisation" is computed against a
30/60 budget. Her export says `budgetWeekday: 45, budgetWeekend: 90`. Against what she actually
runs, she is at **106%** — still over, but a fifth of the gap the queue has been quoting.

---

#### P1 — build next

**O-1 · Right-size the pool from her own data.** One screen: "38 of your tasks are consistently
done less often than their target. Adopt the real interval?" Modelled: adopting real cadence on
the 38 tasks running >1.25× target takes non-daily demand **427 → 342 min/wk, i.e. 106% → 84%
utilisation.** That is the standing capacity constraint solved with data she already owns, and no
task is lost — only re-targeted to the rhythm she actually keeps. *Evidence:* median cadence
ratio 1.60 across 52 measured tasks. *Cost:* no new state; Stats already has per-task "set target
Nd" and `cadenceInfo()`. This is a bulk wrapper over an existing action. *Smallest version:* the
list with a checkbox per row and one Apply.

**O-2 · Tell her the estimates are high.** Her timed runs come in at **0.67× the static
estimate** — counters estimated 5 min take 1, the dishwasher 5 takes 3. `taskTime()` only blends
once a task has samples, and just **11 of 177** do. So the budget is buying her fewer tasks than
she can actually do, every day. *Evidence:* the ratio table above. *Cost:* Budget Insight already
exists and is exactly this shape; add a second line. *Smallest version:* one sentence in Budget
Insight — "your timed tasks run about a third quicker than estimated" — with no automatic change.

**O-3 · An amnesty for the 68.** 68 live tasks have never been completed once and 36 have starved
past 150 days; they are 14% of demand and produce nothing but noise in every pool. Offer, once a
quarter: hide, halve the frequency, or keep. *Evidence:* the counts above. *Cost:* small; hiding
is `deletedIds` and already exists. *This is the removal angle the spec makes mandatory.*

#### P2 — build this round

**O-4 · Two sessions a day, not one.** Her completions cluster at **12:00–14:00 (29%)** and
**20:00–22:00 (25%)** — two distinct work windows, and the app deals one hand for the whole day.
An evening view of what is left, sized to the time she has left, fits her actual rhythm.
*Cost:* no new state — a filter over the existing hand. *Character:* none spent.

**O-5 · Kitchen is five times every other room.** 193 completions and ~964 min in 90 days against
Bedroom's 74 / 376 and Living Room's 18 / 144. Room Pressure shows *overdue*; it does not show
*where the effort goes*. A share-of-effort view would tell her something she cannot currently see
and probably does not know. *Cost:* derivable from history, no new state.

**O-6 · Surface the pool-pressure figure.** The standing P1 candidate from the last round, and it
now has a real number to show (106%, or 84% after O-1). Room Pressure is the place to hang it.

#### P3 — worth having

**O-7 · "What did I actually do this week?"** 945 completions and 70 active days are recorded and
the app shows a 13-week heatmap but no plain sentence. Median 13 completions on an active day,
busiest 40.

**O-8 · Let a preset row start a cycle.** Six presets lost their laundry rows in the rewrite
(C-9). A checklist row that opens a cycle restores them without restoring four dead tasks.

**O-9 · Stale-state sweep on load.** Her export carries `inProgress: {lroom_myclothes}` for a task
that no longer exists — stuck 9 days, unclearable from the UI because it has no card — plus 17
orphan history entries and a stored `returnHome` preset holding four deleted ids. *Cost:* a few
lines in `coerceState`, same shape as the cycle guard already added.

**O-10 · Cycle timings are being used — show they exist.** She changed `laundry.dry` from 55 to
47 unprompted. Nothing advertises that the other steps are editable.

#### Kill list — declined, with the reason, permanently

| Idea | Why not |
|---|---|
| **Weather** | Costs the zero-external-connections property. Standing. |
| **Anything multi-user** | Bob does not use the app. Standing. |
| **Push notifications / reminders** | Needs a service worker and a permission prompt, and the app's whole character is that it is a thing she opens. The two-session pattern (O-4) is the version of this that spends nothing. |
| **Auto-adopting real cadence without asking** | O-1 must stay a proposal. Frequencies are her judgement about her house; the app measuring them is help, the app overruling them is not. |
| **Gamification beyond the existing streak** | `bestStreak` is 9 and the milestone set already fires. No evidence she responds to more, and the character is a tool rather than a game. |

**A claim I withdrew before reporting it.** I nearly filed "she never uses *did it earlier* — remove
it" on the basis that 0 of 945 stamps were midnight-exact. `completeEarlier` writes `now − N days`,
which keeps the current hour, so that probe cannot detect it. The feature's usage is **unmeasured**,
not zero. To measure it, the write path would have to record its source.

---

## Decisions already made — do not re-litigate

| Decision | Why | Date |
|---|---|---|
| **Keep Up keeps the `freq * 0.8` cap** | Two measured attempts to give it the absolute floor made the everyday hand worse — tier A fell from 6 tasks to 2. The cause is that neglected tier B/C tasks are *heavy*, not the floor value; raising the floor to 6 changed nothing. | 2026-08-16 |
| **Frequency is importance** | Her rule, and the arithmetic supports it: a fresh 90-day task should lose to a due toilet. The base score stays `task.freq`. | 2026-08-16 |
| **By Freq stays** | It produces the best routine hand on real data — 14 routine tasks against Keep Up's 6. An earlier proposal to remove it was wrong. | 2026-08-16 |
| **No "Light" mode** | 83% of non-daily tasks already take ≤10 min, so the filter removed almost nothing and the hand was identical to Keep Up. | 2026-08-16 |
| **No tier reservations** | Reserving budget for deep cleaning takes time from the toilet to service the freezer. Rejected on her rule. | 2026-08-16 |
| **"Edit last done" writes history** | Setting a date means the task was done, so Stats and real-cadence must see it. | 2026-08-16 |
| **Removed: `IMPROVEMENT_PLAN.md`, `PLAN_2026-08.md`, `app-design-revision/`** | Work shipped; git holds them. The design bundle was committed once before deletion because it was untracked. | 2026-08-16 |
| **A cycle completes its load at the LAST stage** | Ticking "Wash whites" used to mean whites were washed while they were still in the machine, so `freq: 7` measured loads *started*. It now measures whole loads. Her washing reads slightly less up to date than before because the old count measured the wrong end. | 2026-08-17 |
| **Cycle membership keys off `load: true`, not a new task field** | `saveTask` rebuilds a custom task from an explicit field list, so a `cycle:'laundry'` property would be stripped the first time she edited the task. `load` survives only because it owns the `f_load` checkbox — and it already means "this is a wash". | 2026-08-17 |
| **Cycle stages are never scheduled alone** | "Move to dryer" is not due every day; it is due 55 minutes after the wash started. The four freq-1 dailies were a bad approximation of that, and `dealHand` needed a special case to hide them. Stages are excluded from `getDailies`, starvation, `getNotDueTasks` and the completed-today carry-in. | 2026-08-17 |
| **Bob's away is a toggle, not a dated pause** | Vacation mode shifts due dates on resume because the house genuinely stops. Bob's tasks still need doing while he is gone, so they age normally and arrive with their real dates. Nothing to shift back. | 2026-08-17 |
| **Flagging one of Bob's tasks delivers it** | He has no hand, so the flag set a badge and changed nothing — the button was lying. "Bring me this one" is the only thing it could usefully mean. | 2026-08-17 |

---

## The standing constraint

Her non-daily task list demands **380 min/week**. Her budget is **270** (30/60). That is
**141% utilisation**, and 68 of 182 tasks had never been completed once.

**No scoring change fixes this.** Something must fall off; the only question is what. The
levers are: raise the budget to 45/90, hide ~110 min/week of tasks, or accept that deep
cleaning happens through Catch Up and the Seasonal preset rather than the daily hand.

Any future proposal that claims to improve coverage without costing her something should be
measured against real state before it is believed. Two of them this session did not survive
that test.

---

## Traps that have bitten, in this file because they recur

- **Bumping `APP_VERSION` takes three edits**: the const, the `<meta name="app-version">`
  tag, and the newest `CHANGELOG` entry. The selftest pins the last two.
- **A test that pins old behaviour is not a failure to route around.** Four cases were
  rewritten this session; each kept its ability to fail and gained property assertions.
- **The simulator can produce findings about itself.** Two measurement bugs looked exactly
  like defects in the app: a clock reset in the wrong order, and hand minutes compared
  against a budget that excludes dailies.
- **Verify a CSS change by looking at it.** A green suite says nothing about a stylesheet.
  The Guest Prep radios had been invisible for as long as they have existed, and the suite
  was green the whole time. What found them was rendering the Presets tab and looking at
  the screenshot while doing unrelated tidy-up work.
- **A "hidden" element measures 0x0 for two completely different reasons.** Every tab's
  markup stays in the document, so an inactive tab's controls have no layout at all. Any
  probe that measures the DOM has to skip `offsetParent === null` first, or it reports the
  same control as broken from five tabs where it is merely not on screen.
- **A contrast reading against `rgba(0,0,0,0)` is a lie.** `.task-card` paints nothing;
  the colour comes from an ancestor. Measuring the card's own `backgroundColor` gave
  2.55:1 on text that actually sits at 7.43:1. Walk up to the first non-transparent
  ancestor before computing any ratio. Same family as the un-reloaded theme switch.
- **Never assert against `document.body.innerHTML`.** The app is one file with an
  inline `<script>`, so the body's HTML contains the entire source. Every string a
  case looks for is present as a literal whether or not it was ever rendered. A case
  written that way passed against a build where the feature never rendered at all.
  Assert on the specific container. The suite has no other use of it — keep it that way.
- **A test whose subject is a hardcoded id list can stop testing without failing.** Case
  42 deleted the four laundry steps to prove dailies are charged against the day budget.
  They became cycle stages, stopped being dailies, and the probe quietly compared a hand
  against itself — it still passed. It now derives its subject from `getDailies()`.
  When a change moves tasks between categories, grep the suite for those ids before
  trusting the green.
