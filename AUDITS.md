# Audit Types

Seven audit lenses for the Home Tasks app. Each one looks for a *different class* of defect,
uses different tooling, and accepts different evidence. They are designed not to overlap: a
finding that shows up in two of them means one of the specs is drifting.

Six of them are diagnostic — they find the gap between what the app is and what it should
already be. The seventh is generative: it looks for what is not there at all. It runs last, and
on the other six's output.

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
| 6 | **Coherence** | Did fixing things in layers leave it misaligned? | Whole file + git history | Both sites side by side, dated |
| 7 | **Opportunity** | What is missing that would make it better? | Her real data + audits 1–6 | Evidence of the need, not the idea |

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

**Handoff to audit 6.** This audit *measures against external standards* — WCAG, 44px, 16px —
and produces the inventories. It does not judge whether eleven type sizes should be eleven. That
question is internal self-consistency and belongs to Coherence, which consumes these inventories
rather than recomputing them. Rule of thumb: if a lone designer on a lone screen could still get
it wrong, it is Surface; if it is only wrong *relative to the rest of the app*, it is Coherence.

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

## 6. Coherence Audit — accretion and alignment

**Question.** The app was built in layers, over many sessions, mostly by solving one problem at
a time. Where a problem was solved by *adding* rather than *changing*, what did that leave
behind — and does the app still hang together as one thing?

**Scope.** Internal self-consistency, everywhere: visual constants, interaction grammar, naming,
and the shape of the code. This is the audit where **everything it finds already works.** Nothing
here is a bug; it is all tax — paid on every future edit, and eventually paid by her when two
paths that were supposed to agree quietly stop agreeing.

**The distinguishing question**, applied to every divergence found: *was this decided, or did it
accumulate?* A decided difference is fine and gets one line of documentation. An accumulated one
gets converged. That question is the whole audit; everything below is a way of generating
candidates for it.

**Method.** Four sweeps.

**(a) Constants that forked.** Inventory every visual constant and sort by how many uses escape
the token. *Run at the time of writing:*

| Constant | Via token | Literal uses | Distinct literal values |
|---|---|---|---|
| `font-size` | 151 | 19 | 10 (+ `16px`, which is deliberate) |
| `border-radius` | 22 | 58 | 11 real corner sizes |

The font scale is mostly holding — seven tokens carry 151 uses — but `13.5px`, `13px` and
`12.5px` all exist, which is three sizes nobody chose and no eye can tell apart. Corners are the
opposite: the tokens are barely used, and `8px` / `9px` / `10px` between them account for 26 uses
of what is obviously meant to be one corner. Neither is a rendering fault, so audit 4 will not
raise them; both mean the next screen has no single right answer to copy.

**(b) Behaviours that forked.** For each verb the user can perform, list every code path that
implements it and diff their semantics. The preset system is the worked example, and it is the
clearest accretion scar in the app — the function names admit it out loud:

- `clearLegacyPreset` **/** `clearNewPreset`
- `renderPresetChecklist` **/** `renderNewPresetChecklist`
- `loadPresetIntoHand`, `loadPresetDueOnly` **/** `loadNewPreset`, `loadNewPresetDueOnly`
- state: `guestHand`, `resetHand`, `goingOutHand` **/** the generic `presetHands` map

A general mechanism was built; the three originals were never migrated onto it; so every preset
operation now exists twice and every future preset change costs double. The same shape, milder,
in the six near-identical section collapsers (`toggleOverflow`, `toggleDailies`,
`toggleOtherTasks`, `toggleCompleted`, `toggleSprintCompleted`, `toggleChangelog`) — one
parameterised function wearing six coats.

Then the interaction grammar, which is the user-visible half of the same sweep: does a tick mean
the same thing on My Hand, Sprint, a preset checklist and All Tasks? Does "remove" mean removed
from today, or removed from the pool — and does it mean that consistently? Do the same three
icons appear in the same order on every card? Where two surfaces show the same task, do they
agree on what its buttons do?

**(c) Fixes that patched the symptom.** Read the watch-out list in `CLAUDE.md` and treat every
entry as a finding, because **a rule you have to remember is a design that failed to enforce
itself.** Three of them are load-bearing right now:

- *"`saveTask` rebuilds a custom task from an explicit field list, so any new flag must be added
  there too"* — the data model is defined in two places that must be kept in sync by memory.
- *"Every path that ends a task must clear `state.inProgress[id]`"* — a lifecycle invariant
  enforced by four separate call sites remembering to.
- *"`renderCard` has a local `isSnoozed` that shadows the global helper"* — a name collision
  documented instead of renamed.

For each, the audit asks what structural change would make the rule unnecessary, and what that
would cost. Some will be worth leaving — but leaving it should be a decision, not the default.

**(d) Things that stopped being true.** Vestigial state (`tierCLastDate` is flagged as such in
the docs and still ships), names that outlived their meaning (`openRoomPressurePreset` —
"pressure" appears in no current concept), comments describing older behaviour, and stale
anchors in the docs themselves. Currently `CLAUDE.md` says `index.html` is ~3,960 lines when it
is **5,719**, and puts `TASKS` at line 667 when it is at **945** — the docs have accreted too.

**Evidence bar.** Every finding shows all the diverging sites *side by side in one block* — the
divergence is invisible unless the reader sees them together, which is exactly why it survived.
Then, uniquely to this audit, **`git log` for each site**: two things introduced in the same
commit were probably designed that way; two introduced eight months apart accumulated. That date
gap is the difference between a finding and a matter of taste, and no other audit uses history
as evidence.

**Output.** A convergence list ranked by **cost of divergence** — how many future edits pay the
tax, and how likely the paths are to drift apart — never by how ugly it looks. Each entry: the
sites, the dates, decided-or-accumulated, and the cost to converge. Plus the running trend
numbers (literal-vs-token ratios, duplicate path count), so the next audit can see the direction
of travel.

**Severity mapping.** Most of this is S3, correctly — it is tax, not damage. Two exceptions
promote to S1: two paths that can write *different things* to the same user data (the two undo
paths are the standing example, which is why they are pinned by tests), and any invariant held
together only by a rule in a document, where forgetting it corrupts history rather than throwing.

**Traps.** The temptation is to converge everything at once, in one heroic pass; that produces a
diff nobody can review against an app with one user and no staging. Convergence work ships in
small pieces, each with the selftest green. And divergence is not automatically wrong — the
legacy preset fields hold *live user state*, so converging them needs a migration, which may
well cost more than the duplication does. Say so when that is the answer.

---

## 7. Opportunity Audit — what is missing

**Question.** Given how the app is actually used and what its data already knows, what does not
exist yet that should — and which of those is worth building first?

**Scope.** New features and experience improvements. This is the only generative audit, and the
only one where the output is a proposal rather than a defect. It runs **last**, because its best
raw material is the other six's findings: a tap count from Flow, a starving task from Behaviour,
a calibration ratio from Content. An opportunity audit run cold produces plausible-sounding
product ideas; run on that evidence it produces things she will actually use.

**The discipline.** Ideas are free, which is the problem. Three gates, and an idea that fails any
one is recorded on the kill list rather than the slate:

1. **Evidence gate.** Every idea names the observation that produced it — a friction point, a
   metric, a thing she said, a moment where the app was open and unhelpful. *"Users like
   streaks"* is not evidence. *"This task was dealt 40 times and completed 3"* is. An idea whose
   only support is that it sounds good goes on the kill list with that as the reason.
2. **Character gate.** The app has a settled character: one file, no build step, no accounts, no
   external connections, one user. Those are not accidents — weather was declined specifically
   because zero external connections was worth more than the feature, and multi-user work is dead
   because Bob does not use the app. An idea that violates the character is not forbidden, but it
   must be argued *against the property it costs*, explicitly, in the proposal.
3. **Frequency gate.** How often does she meet the situation this improves? A brilliant fix for
   an annual moment loses to a small fix for a daily one. Rank on frequency × relief, not on how
   interesting the idea is to build.

**Method — six generative angles.** Each is a different way of finding absence, because
"brainstorm improvements" reliably finds only the same three.

- **Data she owns but never sees.** `completionHistory`, `actualTimes` and the starvation
  counters record years of household truth that the app spends on scoring and then discards.
  What is derivable and never shown? This vein is the richest one, and it suits a user who thinks
  in distributions rather than checkboxes.
- **Invert the direction.** Everywhere she has to *ask*, consider the app *telling*. Everywhere
  she has to *decide*, consider it proposing a default she can override. The existing Budget
  Insight is this shape already and is the proof the shape works here.
- **The twenty-three hours.** The app only exists when opened. What happens in the moments it is
  closed — walking past the laundry, standing in a room with nothing to do, the end of a day?
- **The worst day, not the average one.** Illness, overwhelm, a week away, a house guest in an
  hour. The app already has Recovery, Post-Illness and Return Home presets, which proves this
  vein produces things that get used. What other bad days have no shape yet?
- **Removal as a feature.** Which existing thing, deleted, would make the app better? Rarely-used
  surfaces cost attention on every screen they sit on. This angle is mandatory, not optional —
  an opportunity audit that only ever adds is how a good app becomes a cluttered one.
- **The hands-and-body reality.** She is holding a phone while cleaning: wet hands, gloves, a
  pocket, a room away from the counter she set it on. Which interactions assume a desk?

**Evidence bar.** For each idea on the slate: the observation that produced it, an honest cost in
the app's own terms (does it touch `dealHand`? does it add state? does it need a migration?), the
character property it spends if any, and the smallest version that would still be worth having.
That last one does most of the work — most good ideas here have a version that is a tenth of the
size and eight-tenths of the value.

**Output.** A ranked slate, capped at **twelve live ideas**. A backlog longer than that is a wish
list, and wish lists are how the real ones get lost. Plus a **kill list**, which matters just as
much: every declined idea recorded with its reason, permanently, so it stops being re-proposed
every audit. Weather (declined: costs the zero-external-connections property) and anything
multi-user (declined: Bob does not use the app) are standing entries.

**Priority scale.** The S1–S4 severity rubric does not apply — none of this is damage. Its own
scale:

- **P1 — build next.** Daily or near-daily relief, evidence is strong, cost is understood.
- **P2 — build this round.** Clear value, but either less frequent or costlier than P1.
- **P3 — worth having.** Real, but waits for a session where it is adjacent to other work.
- **P4 — parked.** Good idea, wrong time or wrong cost. Reviewed each audit, not carried silently.

**Worked examples**, to show the gates doing real work rather than describing them:

- **Pool pressure, made visible** *(P1)*. Evidence: audit 5 computes 382 min/week of demand
  against 405 of capacity — 94% utilisation — and the app never says so. She currently
  experiences that as a permanent, unexplained sense of being behind. A single figure in Stats,
  recomputed when tasks change, converts a feeling into a number with three levers (raise the
  budget, cut tasks, lengthen a frequency). Cost: arithmetic over `TASKS`, no new state, no
  algorithm change. Character: free. This is the strongest idea currently available and it came
  entirely from another audit's output.
- **Close the estimate loop** *(P2)*. Evidence: `actualTimes` has been collecting real durations
  for months and `taskTime()` already uses the median for budgeting — but the static `time` is
  never corrected and she is never asked. One prompt: *"Your estimate for this is 8 min; the
  last six runs averaged 14. Adopt?"* Cost: low, and the apply-only pattern already exists in
  Budget Insight. Smallest version: show the discrepancy, no adoption button at all.
- **Notify her when the app is closed** *(P4, and instructive)*. Evidence is real — the app is
  useless during the twenty-three hours it is shut. Local notifications need no server, so the
  no-external-connections property survives. But they need a service worker and an install step,
  which spends *one file, no build* — the property that has kept this app alive and editable for
  a year. Parked, with the cost named, so the next audit argues about the right thing instead of
  rediscovering the idea.

**Traps.** Generative sessions drift toward what is fun to build; the frequency gate exists to
pull them back. Beware ideas that are really a second app wearing this one's clothes — the test
is whether it changes what she does in the morning. And an idea that requires her to be
disciplined in a new way is not an improvement; the app's whole premise is that it does the
remembering.

**Operational note.** This audit, and parts of 2 and 5, are only as good as the data behind them.
Running it properly needs a **fresh state export from her phone** — the repo holds the code, but
every observation about what actually happens lives in `localStorage`. Run it against the code
alone and it degrades into exactly the generic product brainstorm the gates exist to prevent.

---

## What these seven do not cover

Deliberate omissions, so the gap is a decision rather than an oversight:

- **Performance.** A 5,700-line single file with no network calls, on one user's phone. If it
  ever feels slow, that is a bug report, not an audit.
- **Security.** No server, no accounts, no external connections. The threat model is "she loses
  her phone", which audit 3 covers as data durability.
- **Accessibility** beyond contrast, hit targets and text scaling — those live in audit 4. Full
  screen-reader semantics is a real audit, just not one with a user waiting for it. Promote it if
  that ever changes.

## Suggested rotation

Not all seven, every time. **Content** is cheap and should run whenever tasks change. **Behaviour**
runs after any change to scoring or dealing. **State** runs before any change to the storage
schema and after any change to import/export. **Flow** and **Surface** run together after visual
work, because a redesign moves both.

**Coherence is the odd one out** and should run on a *time* cadence rather than a change
cadence — roughly every 8–10 sessions, or after any stretch of consecutive bug-fixing. Accretion
is invisible from inside the session that creates it: each individual fix was the right call at
the time, and only the accumulation is wrong. That is precisely why it needs its own scheduled
pass instead of vigilance during the work. Run it *before* a redesign, never after — it tells you
which divergences are load-bearing, and a redesign that starts without knowing that just adds a
seventh layer.

**Opportunity runs last, and never alone.** It is the only audit that consumes the others'
output, so running it first wastes it — the evidence gate has nothing to bite on and the slate
fills with generic product ideas. Pair it with whichever diagnostic audits just ran, and let
their findings feed it. Twice a year for a full generative pass is plenty; the slate is capped at
twelve regardless.

A full sweep of all seven is a session of its own, and the order above is roughly the order of
judgment density — do Behaviour, State and Coherence while fresh, and Opportunity while the
findings are still on the table.
