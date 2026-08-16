# Work Queue

Living document. Not a plan and not a record — it holds only what is **still open**.
Delete a line when it is done; do not tick it. Git history is the record.

Last touched: 2026-08-16 · app at `2026-08-16 v13` · selftest 73/73 green.

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

### C-7 — reduce the inline styles · S4 · **at a stable floor**
**92** inline `style="` attributes remain (was 219 → 211 → 184 → 145 → 104 → 92). Every
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
| 1 | Flow | **Not run** | Optional: which journeys currently annoy her |
| 2 | Behaviour | **Partly run** | — (simulator exists) |
| 3 | State | **Done** (2026-08-16) — 5 found, 5 fixed | — |
| 4 | Surface | **Run 2026-08-16** — 7 findings, unfixed | — |
| 5 | Content | **Half run** | Copy sweep outstanding |
| 6 | Coherence | **Done** | — |
| 7 | Opportunity | **Not run** | A fresh export, and the other audits' findings |
| 8 | Housekeeping | **Done** (2026-08-16) | — |

### 1. Flow — not run
Ten journeys, each ending by getting back *out*. The exit leg is not optional: the
2026-08-16 sheet bug lived entirely on the way out and no one-way journey could reach it.

### 2. Behaviour — simulator built, needs a re-run
`simulate.html` works. **The numbers in the last report are stale** — they were measured
before the Catch Up split, so Keep Up's figures no longer apply. Re-run and record:

- Keep Up coverage is expected to return to ~54% with tier C at 0%. That is now by design;
  tier C progress depends on using Catch Up.
- Catch Up should show a large tier C improvement.
- **Run a second seed before trusting any percentage.** By Freq shuffles within equal
  frequency and only one seed has ever been run.

Known gaps in the simulator: it uses the default task pool rather than her customised one
(hidden tasks, edited frequencies), and it models three completion policies, none of which
is actually her.

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

### 4. Surface — **run 2026-08-16.** 7 findings, none fixed

Trend numbers: **92** inline styles (219 → 211 → 184 → 145 → 104 → 92) · **13** colour
literals outside token definitions, of which 4 are shadows and 4 are documented
swipe-rail fills · **0** colour literals in JS-built markup · **34** distinct type triples.

**S2 · Hit targets, systemic. 539 instances under 44×44, 76 distinct shapes.** Height is
almost always the failing dimension — plenty are wide and 23–34px tall. By volume:
`.task-more-btn` 30×30 (**216** instances, Hand/Sprint/All Tasks). By severity:
`.settings-save-btn.quiet` 98×**23**, `.manage-hand-btn.sm` 61×**26**, `.stepper-btn` 28×28,
and `.cad-name` — a tappable text link **15px** tall. Fix once as a policy (a minimum
block-size plus transparent padding), not per-component.

**S2 · `importTextarea` computes to 13px.** Every other input clears the 16px floor; this
one does not, so iOS Safari zooms the page on focus and does not zoom back — on the
restore-from-backup field, which is a bad place to be fighting the viewport.

**S3 · Dark `--text3` misses AA by 0.03.** `#8b8574` on `--card` = **4.47:1** against 4.5.
Sixteen of the 19 dark failures are this single token: `.caption`, `.muted`, `.cad-sub`,
`.task-notdue`, `.sheet-group-label`, `.tab-label` (inactive), `.settings-summary`. One
nudge clears the lot. On `--bg2` it is 4.36 (`.settings-stat-label`, `.vac-legend-label`).

**S3 · The swipe rail's literal text colour is wrong for one of its four buttons.**
`.task-swipe button { color: #fffdf8 }` is deliberate — the comment says those fills stay
dark in both themes. True for `.sw-more`/`.sw-pin`/`.sw-remove`, but `.sw-today` was added
later using `var(--green)`, which lightens to sage in dark mode: **2.04:1**. Separately
`.sw-pin` is 3.11:1 and `.sw-more` 3.62:1 in *both* themes.

**S3 · A collapsible card's title is dimmer than a fixed card's, for no reason she can
read.** `.ed-details > summary { color: var(--text3) }` and `.settings-card-title` inherits
it, so "How Scoring Works" and "Recent Activity" sit at 4.47:1 while "System Overview" and
"Vacation Mode" are ~11:1. Plainly visible side by side in Settings.

**S4 · Light-mode near-misses:** `.settings-stat-label` / `.vac-legend-label` 4.32:1,
`.rp-zone` 4.17:1.

**S4 · Dead CSS.** `.owner-badge`, `.owner-alina`, `.owner-bob` are defined and never
emitted by any code path.

**Type scale — the near-duplicates are on the line-height axis, not size.** 13px/400 renders
at five different line-heights (19.5, normal, 23.4, 18.85, 22.1); 12px/400 at four. Whether
34 triples should be 34 is Coherence's question; that one size/weight resolves five ways is
this one's.

**Passed:** no horizontal overflow at 375px on any tab. Zero colour literals in JS-built
markup — the only two matches are `&#10003;` tick entities. Every other input ≥16px.

**Method trap, cost me four false findings twice.** Changing the emulated colour scheme
*without reloading* leaves elements resolving old foreground colours against new
backgrounds, which reads as catastrophic contrast failures: the active tab measured 2.00:1
and the active mode button 1.87:1. After a reload they are **7.96:1** and **7.43:1**.
Reload after every theme switch, then measure.

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
