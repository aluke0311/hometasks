# Work Queue

Living document. Not a plan and not a record — it holds only what is **still open**.
Delete a line when it is done; do not tick it. Git history is the record.

Last touched: 2026-08-16 · app at `2026-08-16 v7` · selftest 66/66 green.

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

### C-7 — reduce the inline styles · S4 · large, mechanical
**211** inline `style="` attributes remain (was 219). The design rule is that the count
must not grow; new UI uses classes. Convert opportunistically, in small commits.

### C-8 — the last 60 spacing literals · S4 · needs judgment
The token sweep converted 123 spacing sites. The remaining 60 sit inside `calc()`,
`env()` and mixed-unit shorthands where a blind substitution would be wrong. These need
reading one at a time, so they were left.

### C-9 — one-off display type · S4 · tiny
`22px`, `26px`, `30px`, `38px` are each used once, for display numerals. Either give them
tokens or accept them as deliberate exceptions and note it. Currently neither.

**Not in the queue, deliberately:** the `16px` on inputs. iOS Safari zooms the page on
focus below it. It looks like an inconsistency and must stay.

---

## Audit queue

Run order is roughly judgment-density. `AUDITS.md` holds the full spec for each.

| # | Audit | State | Needs from her |
|---|-------|-------|----------------|
| 1 | Flow | **Not run** | Optional: which journeys currently annoy her |
| 2 | Behaviour | **Partly run** | — (simulator exists) |
| 3 | State | **Not run** | — |
| 4 | Surface | **Not run** | — |
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

### 3. State — not run, cheapest real win
The probe that matters: **tabulate every writer of `state.completions`** against the
satellite state a completion implies — `inProgress`, `snoozed`, `starvation`, `flaggedIds`,
`completionHistory`, hand membership, pins. Any blank cell is a finding. `setLastDone` sat
in that table with five blanks for months and it cost her a task that could never leave the
hand.

### 4. Surface — not run
Trend numbers to carry forward: **211** inline styles, **60** spacing literals, radius and
small type now tokenised. Needs paired light/dark screenshots of all six tabs, contrast
against AA, and hit targets — `.icon-btn` is 26–28px against a 44px target, which is a
known systemic gap and should be fixed once, not per-component.

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
