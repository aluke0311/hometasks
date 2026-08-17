# Work Queue

Living document. Not a plan and not a record — it holds only what is **still open**.
Delete a line when it is done; do not tick it. Git history is the record.

Last touched: 2026-08-17 · app at `2026-08-17 v2` · selftest 86/86 green.

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

### C-9 — cycles beyond laundry · shipped for laundry only, on purpose
`CYCLES` (2026-08-17) holds one definition. The shape was proven against one real
process before generalising, which was the whole plan. Adding the next two is data,
not code — but each has one open question:

| Cycle | Stages | The question |
|---|---|---|
| **Dishwasher** | load → run → unload | `k_dishwasher` ("Unload dishwasher") is a freq-1 daily and would become a stage, so it stops being scheduled on its own. It is also on **seven** preset lists. Ticking it there advances a running cycle, but with no cycle open it stays a plain task — check that reads sanely before committing. |
| **Cat fountain** | clean → reassemble | `c_fountain` is the one task in `ALWAYS_ASSIGN_IDS`, a hack that exists *because* it is a two-stage job. Converting it should delete that special case — the point of the model. Its stages have no tasks yet; they must be added to `TASKS` first or the sweep fails. |

**Blocked on nothing.** The reason to wait is evidence: run the laundry cycle for a
week first and see whether the one-card form actually helps before spending the
change twice more.

**What v1 does not do, all deliberate:**
- **`maxOpen: 1`.** The instance model supports concurrent loads; one washer does
  not. Raising it is one line, and the card layout has never been seen with two.
- **`readyAt` is inert.** Stored and displayed ("ready ~1:36 PM"), never a gate.
  Making it block the tick would be the app's first sub-day due rule — everything
  else resolves at calendar-day granularity — so it needs a render-time re-check,
  not a deal-time one.
- **The hand's minute totals count the load's own `time`, not the current step's.**
  `renderCard` and the budget use the step; `renderAlina`/`renderSprint`/`copySprint`
  total `taskTime(task)` across nine call sites. Worth ≤7 minutes on a laundry day.
  Fixing it means a cycle-aware total helper at all nine, which was not worth the
  blast radius on the first cut. **Do not make `taskTime()` itself cycle-aware** —
  it feeds the editor field and the learned-time hint.
- **A stage ticked with no cycle open is just a task.** Presets still list the four
  steps; `toggleComplete` advances a cycle waiting on exactly that step and
  otherwise completes normally. Two representations, reconciled at the tick.

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
| 1 | Flow | **Blocked** — see note | Optional: which journeys annoy her |
| 2 | Behaviour | **Re-run 2026-08-16** — 3 findings, unfixed | Her call on B-1 (By Freq) |
| 3 | State | **Done** (2026-08-16) — 5 found, 5 fixed | — |
| 4 | Surface | **Done** (2026-08-16) — 7 found, 7 fixed | — |
| 5 | Content | **Half run** | Copy sweep outstanding |
| 6 | Coherence | **Done** | — |
| 7 | Opportunity | **Not run** | A fresh export, and the other audits' findings |
| 8 | Housekeeping | **Done** (2026-08-16) | — |

### 1. Flow — **attempted 2026-08-16, blocked by the harness.** Still not run.

Not a finding about the app. The Browser pane renders and screenshots fine but times out on
synthetic input — three consecutive `left_click` / `left_click_drag` calls each hung for 30s,
and one of them was held long enough that it registered as a **long-press and opened the task
actions sheet instead of ticking the task**. Flow's method is explicitly "tapping only what a
person could see" and its evidence bar is "a screenshot at the moment of failure plus the
exact tap sequence". With input that unreliable, every finding would be indistinguishable
from a driver artefact — and this session already produced two of those (four false contrast
readings from an un-reloaded theme switch). Abandoned rather than reported.

**Before retrying:** confirm the pane accepts a single click inside 30s on something
harmless. If it does not, Flow cannot be run from here and needs either a fixed harness or
her walking the ten journeys herself. Journey 10 (the escape) is the one to protect if time
is short — it is the only journey that would have caught the 2026-08-16 sheet bug.

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

### 3. State — **run and fixed 2026-08-16.** Nothing open.

All five findings fixed in `2026-08-16 v13`, each with a mutation-checked case. Kept here
only as things that must not regress, because all five were invisible to a green suite:

- **Every path writing `state.completions` to TODAY must call `stashPrevCompletion(id)`.**
  The stash lived inside `completeTask` alone; `endTaskAsOf` lacked it, and that cost a
  180-day task its real date. Shared helper now — do not inline it again.
- **`dealHand`'s daily starvation tick hides missing resets.** It zeroes the counter for
  anything in the hand, including a task completed today, so a path that forgets to clear
  starvation looks correct all day. Any test of this must pin `starvationDate` to today.
- **Vacation mode must survive midnight.** `refreshIfDayChanged` returns early when paused.
- **`applyImport` validates shape, not truthiness**, and reports counts rather than success.
- **A completion whose task was deleted still counts.** `ORPHAN_ROOM` keeps `byRoom` summing
  to `count`; it is excluded from the most/least-attention superlatives on purpose.

Passes worth not re-deriving: export → import round trip is clean across all 41 fields;
storage settles at **≈62 KB, 83× under a 5 MB quota**, so `pruneHistory` genuinely bounds it.

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

### 5. Content — numbers done, copy not
Done: pool economics, calibration, tier demand. Outstanding: read every user-visible string
in one pass for voice, terms used two ways, and empty states that do not say what to do next.

### 7. Opportunity — run last
Needs a fresh export and the other audits' output. Standing kill list: **weather**
(declined — costs the zero-external-connections property) and **anything multi-user**
(declined — Bob does not use the app). P1 candidate already identified: surface the
pool-pressure figure, since the app already has a Room Pressure concept to hang it on.

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
- **A test whose subject is a hardcoded id list can stop testing without failing.** Case
  42 deleted the four laundry steps to prove dailies are charged against the day budget.
  They became cycle stages, stopped being dailies, and the probe quietly compared a hand
  against itself — it still passed. It now derives its subject from `getDailies()`.
  When a change moves tasks between categories, grep the suite for those ids before
  trusting the green.
