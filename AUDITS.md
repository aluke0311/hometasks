# Audit Types

Five audit lenses for the Home Tasks app. Each one looks for a *different class* of defect,
uses different tooling, and accepts different evidence. They are designed not to overlap: a
finding that shows up in two of them means one of the specs is drifting.

The unifying rule, inherited from the 2026-08-08 round: **a green selftest proves nothing.**
Every audit below states its own evidence bar, and no finding is reported until it has been
reproduced by the means that audit specifies. Reading the code and inferring a bug is a
*hypothesis*, not a finding.

| # | Audit | Asks | Runs against | Evidence that counts |
|---|-------|------|--------------|----------------------|
| 1 | **Flow** | Can she actually get the thing done? | Rendered app, 375px, real taps | Screenshots + tap transcript |
| 2 | **Behaviour** | Does the algorithm do what it claims over time? | Headless sim, 90–365 simulated days | Metric tables, distributions |
| 3 | **State** | Can the data be lost, corrupted, or silently rewritten? | localStorage, import/export, clock | Before/after state diffs |
| 4 | **Surface** | Does it look like one app in both themes? | Computed styles, both colour schemes | Computed values + paired screenshots |
| 5 | **Content** | Is the task pool itself sound? | `TASKS`, presets, `actualTimes`, copy | Arithmetic over the pool |

---

## Shared severity rubric

Used by all five so findings are comparable across audits.

- **S1 — Data loss or lies.** Destroys real history, or shows a number that is wrong in a way
  she would act on. Fix before anything else ships.
- **S2 — Broken path.** A thing she wants to do cannot be done, or takes a workaround she
  has to remember.
- **S3 — Friction / drift.** Works, but costs extra taps, extra thought, or degrades slowly.
- **S4 — Blemish.** Cosmetic, or correct-but-inelegant. Batch these; never ship one alone.

Each finding carries: severity, the reproduction, the file:line, and — for anything above S4 —
what a fix would cost. A finding with no reproduction is filed as a *suspicion* and does not
count toward the audit's total.

---

## 1. Flow Audit — the journeys

**Question.** For each thing Alina actually opens the app to do, can she do it — first try,
one-handed, without reading anything twice?

**Scope.** The full width of the app as *behaviour*, not appearance. Navigation, affordance
discovery, state that survives a tab switch, recoverability of mis-taps, the cost in taps of
each journey. Explicitly *not* colour, spacing, or type (that is audit 4).

**Method.** Drive the real app in the browser pane at the mobile preset (375×812), one journey
per pass, tapping only what a person could see. No `javascript_tool` shortcuts to reach a state —
if you cannot get there by tapping, that *is* the finding. Journeys to walk:

1. **The morning.** Open cold → read the hand → do one task → check it off → notice it moved.
2. **The over-budget morning.** Hand is bigger than the day. Shrink it with the time slider.
   Does the redeal respect what she just said?
3. **The mis-tap.** Check off the wrong task; recover after the toast has expired.
4. **Logging Bob.** He did a chore; record it without it eating her budget.
5. **The return.** Resume from a 9-day vacation; first hand after resume.
6. **The half-done task.** Start the fountain, leave it in the dishwasher, come back tomorrow.
7. **The sprint.** Run a room-by-room walkthrough end to end, then copy it out.
8. **The unusual task.** Add a one-off, do it, and confirm it does not haunt the pool.
9. **Curiosity.** "Why is *this* on my list today?" — reach the score breakdown from a cold start.

**Evidence bar.** A screenshot at the moment of failure plus the exact tap sequence that got
there. Tap counts are recorded for every journey even when nothing fails — the count *is* the
metric, and it is how S3 friction gets caught before it is felt.

**Output.** One row per journey: taps taken, taps needed, where it stalled, severity.

**Traps specific to this app.** Handlers that re-render invalidate the node you were about to
tap — a journey that works when driven slowly may fail under a real thumb. Sheets must be
exercised via their real open/close path. And the hand is dealt once per day: a journey that
redeals to reach its start state is testing a different code path than the morning one.

---

## 2. Behaviour Audit — the algorithm over time

**Question.** Over a year, does the dealer keep the promises the design makes — every task
serviced near its `freq`, budget respected, nothing starving, nothing dominating?

**Scope.** `scoreTask` / `dealHand` / starvation / vacation shift / the laundry slot, measured
as a *dynamical system*. A single day's hand tells you almost nothing; the failure modes here
are all about accumulation.

**Method.** Extend the selftest's isolated-iframe harness into a simulator: inject a controllable
clock, run 90–365 simulated days, complete tasks under a stated policy, and log every deal.
Three policies, because the interesting failures are policy-dependent:

- **Compliant** — she does everything dealt. Tests the ceiling.
- **Realistic** — she does ~75%, skipping the longest task each day. Tests starvation.
- **Sporadic** — 3 days on, 4 off, plus one 10-day vacation. Tests recovery.

Metrics per run, per task:

- **Service interval** — empirical mean and P90 days between completions, as a ratio to `freq`.
  A ratio > 1.5 sustained is the task quietly falling off the map.
- **Starvation tail** — max and P95 days-overdue. The `freq * 0.15` cap on the starvation bonus
  is load-bearing; this measures whether it is set right, which the unit tests cannot.
- **Budget adherence** — dealt minutes vs. the day's budget, as a distribution. Mean overshoot
  is the wrong statistic; the tail is the one she feels.
- **Coverage** — share of the pool dealt at least once in 365 days. Anything never dealt is
  dead weight paying rent in the scorer.
- **Repeat rate** — probability a task appears in tomorrow's hand given it appeared today and
  was not completed. Too high reads as nagging; too low reads as forgetful.
- **Laundry slot** — utilisation and mean lateness per load task. Known to be near-saturated;
  this quantifies the slack.
- **Zone effect size** — does the −2 zone bonus actually shift the composition of the hand, or
  is it lost in the noise of the score distribution?

**Evidence bar.** A metric table from a seeded, reproducible run, plus the seed. Any finding
must survive a second seed — a claim that rests on one RNG draw is not a finding. Where a
threshold is proposed (change a weight, change a cap), the audit reports the metric *before and
after* the proposed value, not just the complaint.

**Output.** The metric tables, a ranked list of tasks whose service interval is worst, and for
each proposed constant change, the before/after.

**Traps.** Jitter on `freq > 60` means results must be averaged, never eyeballed from one run.
Vacation shift interacts with `taskVac`, so a sim that leaves everything at the `freeze` default
is not testing the feature. And a simulator that deals by calling internals rather than the real
`dealHand` will happily certify a dealer that does not exist.

---

## 3. State Audit — durability and truth of the data

**Question.** Is there any sequence of ordinary use that loses history, corrupts it, or writes a
date that is not the date the thing happened?

**Scope.** `localStorage` as the single copy of record. Migration, coercion, import/export
fidelity, the two undo paths, history pruning, and every place a timestamp is written or
inferred. This is the audit that protects years of real completion data.

**Method.** Adversarial, state-first. For each probe: snapshot state → act → snapshot → diff,
and assert on the *diff*, not on what the UI says happened.

- **Round trip.** Export → wipe → import → deep-diff. Any field that does not survive is S1.
- **Migration.** Load state saved by each shipped version still plausibly on her phone. Assert
  no field is dropped and no default silently overwrites a real value.
- **Malformed input.** Import truncated JSON, wrong types per field, unknown task ids, a
  `completions` entry in the future, negative counters. Nothing may throw past the boundary;
  nothing may half-apply.
- **The two undos.** `undoComplete` (toast) and `uncompleteTask` (card / All Tasks / Sprint)
  restore *different* things. Probe both on a task with real history and on one with none —
  neither may ever fabricate `now − freq`. On a 180-day task that quietly destroys half a year.
- **Double-completion.** Same task twice in a day, then undo once. Then undo again.
- **Clock.** Cross midnight mid-session. Cross a DST boundary. Change the device timezone
  between two completions. Anything comparing calendar days is suspect here.
- **Growth.** Project `localStorage` size at 3 and 10 years of history against the quota, and
  confirm `pruneHistory` actually bounds it rather than deferring the cliff.
- **Orphans.** Ids present in `completions` / `taskVac` / `actualTimes` / `pinnedIds` that no
  longer exist in `TASKS`, and preset ids that point at nothing.

**Evidence bar.** The state diff, printed. A UI that shows the right thing over wrong stored
data is an S1 finding, not a pass — this audit never accepts the screen as evidence.

**Output.** A probe table (probe / expected diff / actual diff / verdict), plus the storage
growth projection.

**Traps.** `coerceState` spreading over `defaultState()` means a missing field is invisible
rather than loud; probes must assert positively that a value survived, never merely that nothing
threw. Fields defaulting to `null` cannot have their type inferred and need explicit shape
checks. And anything that "makes a task due" by writing `completions` is a no-op while the task
is snoozed.

---

## 4. Surface Audit — the visual system

**Question.** Does every screen belong to the same app, in both themes, at her actual font size?

**Scope.** Colour, type, spacing, hit targets, motion, contrast, and the discipline of the token
system. Appearance only — if the finding is "this is hard to *do*", it belongs to audit 1.

**Method.** Render every screen and *look at it*, then confirm with computed values.

- **Token discipline.** Sweep for hardcoded colour literals in styles and in JS-built markup.
  Every one is a finding; count them as a trend line across audits. Same for the inline-style
  count (**219** at the time of writing) — the rule is that it must not grow.
- **Theme pairs.** Screenshot all six tabs plus every sheet and modal in light and dark, side by
  side. Text on a filled accent uses `var(--on-accent)`; the toast inverts and needs its own
  token. Anything defined only inside a media query fails on one of the three theme states.
- **Contrast.** Computed foreground/background for every text style against WCAG AA. Secondary
  and disabled text is where this fails.
- **Hit targets.** Measure every interactive box. Below 44×44 is a finding; the swipe rail and
  the icon rows on task cards are the repeat offenders.
- **Input zoom.** Every input's computed `font-size` ≥ 16px, or iOS Safari zooms the page on
  focus and does not zoom back.
- **Overflow.** Longest real task name, longest room name, biggest number Stats can produce —
  in every card, at 375px. Nothing may scroll the body horizontally.
- **Type scale.** Every distinct computed `font-size` / `font-weight` / `line-height` triple in
  the app, listed. Near-duplicates are the finding.
- **Density.** Screenshot at the system's largest accessibility text size and confirm the
  primary journey is still operable.

**Evidence bar.** Paired light/dark screenshots plus the computed value. "Looks fine" is not a
result; neither is a grep count standing in for visual weight.

**Output.** The trend numbers (colour literals, inline styles, distinct type triples), the
contrast and hit-target failure lists, and the theme-pair contact sheet.

**Traps.** `input { appearance: none }` is a blanket reset — any control not drawn by hand is
invisible, and has been before. The three theme states (explicit light, explicit dark, unset)
are not two; a page that only handles `prefers-color-scheme` is broken for one of them.

---

## 5. Content Audit — the pool and the words

**Question.** Is the task list itself well-formed, correctly estimated, and honestly described —
before any algorithm touches it?

**Scope.** The 186 entries in `TASKS`, the preset id lists, the learned-time data, and the app's
copy. This audit needs no running app at all; it is arithmetic and reading.

**Method.**

- **Pool economics.** Sum `time / freq` over the pool she can actually be dealt (`alina` +
  `either`) to get demanded minutes per day, and compare to budgeted capacity. *Run at the time
  of writing:*

  | Quantity | Value |
  |---|---|
  | Whole-pool demand | 113.8 min/day |
  | Alina + either | 102.5 min/day |
  | — of which daily (`freq ≤ 1`) | 48.0 min/day |
  | — non-daily | 54.5 min/day → **382 min/week** |
  | Budgeted non-daily capacity (45×5 + 90×2) | **405 min/week** |

  That is **94% utilisation**. The pool is not over-subscribed, but at 94% with day-to-day
  variance in what actually gets done, queue lengths are governed by the tail, not the mean —
  chronic overdue tasks are the *expected* behaviour of this configuration, not a scoring bug.
  This number is the single most useful thing the content audit produces, and it should be
  recomputed every time tasks are added. Adding one more weekly 30-minute task pushes it past 99%.

- **Estimate calibration.** For every task with ≥3 samples in `actualTimes`, the ratio of learned
  median to static `time`. Systematic optimism inflates every hand; the ratio distribution says
  whether the static estimates need a global correction or a handful of individual ones.
- **Frequency plausibility.** Declared `freq` vs. observed real cadence from `completionHistory`.
  Where the gap is large and persistent, either the frequency is aspirational or the task is
  mis-scoped — the audit says which by looking at whether it is *always* late or *never* dealt.
- **Duplication and overlap.** Task pairs whose scope plausibly intersects (two tasks that both
  clean the same surface), and tasks whose name does not distinguish them at a glance in a hand.
- **Structural integrity.** Every preset id exists in `TASKS`. Every room maps to a zone or is
  deliberately zone-less. Owner distribution (currently 138 `either` / 30 `alina` / 18 `bob`) —
  `either` this dominant means the owner field is barely doing work, which is worth a decision.
- **Tier balance.** Currently A 46 / B 73 / C 67. Tier C is a third of the pool and the least
  likely to be dealt; the audit checks that C tasks are reachable at all in a year (cross-check
  against audit 2's coverage metric).
- **Copy.** Every user-visible string, read in one pass: labels, empty states, toasts, the
  score explainer. Looking for inconsistent voice, terms used two ways (mode names vs. their
  descriptions), and empty states that say nothing is here without saying what to do next.

**Evidence bar.** Computed from the actual arrays and the actual history — never from reading a
few entries and generalising. Copy findings quote the string and its location.

**Output.** The economics table with the utilisation figure, the calibration ratio distribution,
the duplication candidate list, and the copy diff.

**Traps.** Demand must exclude `bob`-owned tasks from *her* capacity comparison but include them
in the whole-pool figure — they are real work, just not hers. Seasonal tasks (`taskMonths`) are
not demanded year-round and should be prorated, or the utilisation figure runs high.

---

## What these five do not cover

Deliberate omissions, so the gap is a decision rather than an oversight:

- **Performance.** A 5,700-line single file with no network calls, on one user's phone. If it
  ever feels slow, that is a bug report, not an audit.
- **Security.** No server, no accounts, no external connections. The threat model is "she loses
  her phone", which audit 3 covers as data durability.
- **Accessibility** beyond contrast, hit targets and text scaling — those live in audit 4. Full
  screen-reader semantics is a real audit, just not one with a user waiting for it. Promote it if
  that ever changes.

## Suggested rotation

Not all five, every time. **Content** is cheap and should run whenever tasks change. **Behaviour**
runs after any change to scoring or dealing. **State** runs before any change to the storage
schema and after any change to import/export. **Flow** and **Surface** run together after visual
work, because a redesign moves both. A full sweep of all five is a session of its own, and the
order above is roughly the order of judgment density — do Behaviour and State while fresh.
