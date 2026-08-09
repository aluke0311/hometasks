---
name: app-update
description: Workflow for changing the Home Tasks app (index.html) and for closing out a work session. Use when making any edit to the app — fixing a bug, adding a task or preset, changing scoring/dealing/UI — and especially when the user says "close out", "wrap up", "close out the session", or otherwise signals the session is done and the docs should be reconciled. On close-out it updates CLAUDE.md, DOCUMENTATION.md, and IMPROVEMENT_PLAN.md, plus the skills and the project memory files, from that session's changes and learnings.
---

# Home Tasks — Update & Close-Out Workflow

This skill governs how to make changes to the Home Tasks app and how to "close
out" a session by syncing the documentation. The whole app is the single file
`index.html` (HTML + CSS + vanilla JS, no build step). State lives in the
browser's `localStorage`; deploying never touches it.

There are two modes. Pick based on what the user asked for.

---

## Mode A — Making a change

Use for any edit to `index.html` (bug fix, new task/preset, scoring/dealing/UI
change).

1. **Understand before editing.** Grep/read the relevant function. Line numbers
   shift constantly — re-grep, don't trust remembered anchors. Key functions:
   `TASKS`, `loadState`/`defaultState`, `scoreTaskParts`/`scoreTask`,
   `dealHand`, `getAllTasks`, `completeTask`, `renderCard`, the preset
   definitions block. See [CLAUDE.md](../../../CLAUDE.md) for the architecture.

2. **Make the edit** matching the surrounding code style.

3. **Bump `APP_VERSION`** whenever you touch `index.html`, so a Reload in
   Settings can be confirmed to have taken. (Standing user instruction.)
   - Format is `'YYYY-MM-DD vN'` — e.g. `const APP_VERSION = '2026-08-03 v2';`
   - **Read the committed value, not the working tree:**
     `git show HEAD:index.html | grep APP_VERSION`. An earlier session may have
     already bumped it today; reading the working tree lets two different builds
     ship the same `vN` and the same-day scheme collides silently.
   - Same date already committed → increment `vN`. New date → start at `v1`.

4. **Verify in the preview** before committing — never ask the user to check
   manually:
   - `preview_start` with the `hometasks` config (python http.server on 7821).
   - Navigate to `/index.html?v=<something new>`, exercise the change, and check
     the console for errors. **The query string is not optional:** the pane
     caches, so a plain `/index.html` can keep serving the previous build and you
     will verify a change that isn't there. `selftest.html` is immune (it fetches
     `no-store`), which makes the failure mode "tests green, page stale" —
     confirm `APP_VERSION` in the page matches what you just wrote.
   - For data/preset changes, run the dead-ref check below.
   - **For any UI change, render the screen at 390px and look at it** — in both
     light and dark mode. A clean console is not evidence the layout is right;
     reading CSS ranks findings by grep count, and grep count is not visual
     weight. Screenshot it and read the screenshot.
   - Once `selftest.html` exists: any change to scoring, dealing, state shape,
     or presets requires it green before commit. Pure CSS/copy/task-text changes
     are exempt.

5. **Commit + push automatically** — no confirmation needed (standing user
   instruction). Branch first only if not on `main` would be unusual here; this
   repo works directly on `main`.

6. **State the export status** at the end of the change description, always one
   of: `✅ No export needed` or `⚠️ Export needed before deploying`.
   - **No export needed:** UI/logic changes, and adding new state fields (a
     default in `defaultState()` is enough — see the retired watch-out below).
   - **Export needed:** bumping `STORAGE_KEY`, a breaking field-format change, or
     intentionally clearing/restructuring state.

7. **Commit message convention:** a title plus a body, both in a single fenced
   code block, no manual line breaks in the body (let it wrap). End the message
   with a `Co-Authored-By:` trailer naming the model that made the change.

**Mobile defaults — apply without asking** (phone-first app; these are settled,
not proposals): viewport meta locks zoom (`maximum-scale=1, user-scalable=no`);
any bottom sheet with a text input sits above the keyboard via a `visualViewport`
listener setting a `--kb` variable; never auto-fill a field the user hasn't
entered; toggles look like toggles; derived or proxy values never masquerade as
real data.

**Watch-outs when editing** (these have bitten before):
- Don't break: laundry slot logic, `_dealPreferTier`, the starvation cap
  (`freq * 0.15`), pinned/in-progress carry-over. All four are pinned by
  `selftest.html` — run it.
- Preset task ids must exist in `TASKS`, or the item silently vanishes and
  `completeTask` returns early so it can never be checked off. `selftest.html`
  case 3 covers every preset constant; no manual scan needed, and case 4 fails if
  a new preset type is wired into the tab without joining the sweep.
- **A new per-task setting goes in its own `state.*` map keyed by id**
  (`taskMonths`, `taskVac`), NOT on the task object. `saveTask` rebuilds a custom
  task from an explicit field list, so anything stored on the task is silently
  stripped the first time she edits it. `load: true` is the one flag that lives
  on the task, and it only survives because it has its own editor checkbox.
- **Re-query a DOM node after calling anything that can re-render.** The
  date-picker toggles save-and-close the open row first; that save re-renders, so
  a node captured before the loop is detached and opening it does nothing at all.
- **Check whether a "does nothing" bug is a short-circuit upstream.** `markTaskDue`
  wrote the right timestamp for a year, but `daysOverdue()` returns early on an
  active snooze before it ever reads it, so the button toasted success and changed
  nothing.
- **Two undos exist and they are not interchangeable.** `undoComplete` (toast)
  reverses the last completion from `_undoBuffer`; `uncompleteTask` (card button,
  All Tasks checkbox, Sprint untick) restores `prevCompletions[id]`. Never let
  either fabricate `now − freq` — on a twice-yearly task that moves last-done
  months *backwards* and destroys the real date.
- **A checklist tick must be `toggleComplete`, never `completeTask`.** Wired to
  the latter, tapping a ticked card re-fires the completion and a mis-tap is
  uncorrectable once the toast expires.
- **Every path that adds a task to today must call `unremoveToday(id)`,** or the
  next redeal takes it back out.
- `renderCard` has a local `const isSnoozed` boolean that shadows the global
  `isSnoozed()` helper. Calling the helper inside `renderCard` throws.

**When auditing rather than building:** the harness being green says nothing
about the paths it doesn't cover. The 2026-08-08 round found twelve real bugs
against a fully green suite. What actually surfaced them was driving the running
app with `javascript_tool` — set up a realistic state, take an action a user
would take, and assert on what came back. Reading the code alone would have
missed the ones that depend on state (Bob's completions eating the budget, the
detached date-picker node).

**Retired watch-outs — do not reinstate:**
- *"Adding state? Update all three of `defaultState()`, `loadState()` and
  `applyImport`."* Obsolete since 2026-08-03. Both paths route through
  `coerceState()`, which spreads over `defaultState()`, so **`defaultState()` is
  now the only place to add a field.** One exception, and it is a check rather
  than a default: a field whose default is `null` can't have its type inferred,
  so add it to `NULLABLE_SHAPE` if you want it shape-guarded. Forgetting that
  costs a missed type check, never a dropped field.
- *The Python dead-reference scan.* Superseded by `selftest.html` case 3, which
  covers more constants and can't drift out of sync with the line numbers it
  used to hardcode. One gate, not two.

---

## Mode B — Close out

Trigger phrases: **"close out"**, "wrap up", "close out the session", "we're
done", or any signal the working session is finished and docs should catch up.

Close-out reconciles the documentation with everything that changed this
session. Do this:

1. **Gather what changed.** Read the session: review the code edits made,
   `git log` for commits since the session began, and any decisions, new
   behaviors, fixed bugs, or newly discovered issues from the conversation.

2. **Update [CLAUDE.md](../../../CLAUDE.md)** (the condensed working reference):
   - Reflect any changed behavior (scoring, dealing, modes, presets, state).
   - Correct stale `~line` anchors for functions that moved.
   - In **Known Issues / Watch-outs**: remove anything fixed this session, add
     anything newly discovered.
   - Update **Repository Files** and the state/tab/preset sections if they
     changed.

3. **Update [DOCUMENTATION.md](../../../DOCUMENTATION.md)** (the full reference):
   - Apply the same factual changes in the relevant sections (data model,
     scoring, dealing, presets catalog, state schema, edge cases, code map).
   - Keep the code-map line ranges roughly current.

4. **Update [IMPROVEMENT_PLAN.md](../../../IMPROVEMENT_PLAN.md)** only if planned
   work shipped or new planned work was agreed — mark items done or add them.
   It's a historical record; don't rewrite finished history.

5. **Update this skill and any other skill the session outdated.** Anything
   learned about *how to work on this app* belongs here, not only in the prose
   docs. Look for:
   - A watch-out that a structural change has made obsolete — **delete it.** A
     warning that no longer describes a real hazard is worse than no warning,
     because it trains the reader to skim the list.
   - A new hazard, gate, or convention discovered this session — add it to the
     Mode A watch-outs or the verify step.
   - A procedure the session found to be wrong or stale (a changed file layout,
     a superseded manual scan, a drifted format string) — correct it against
     what the code actually does now, not what it used to.

6. **Update memory.** Session-level learnings die with the session unless
   written down. Check
   `~/.claude/projects/-Users-alinaluke-Documents-Claude-Code-hometasks/memory/`
   and:
   - Add a memory for any durable preference, correction, or project decision
     the session produced — especially decisions made so they need not be
     re-litigated, and the reasoning behind them.
   - **Update an existing file rather than creating a near-duplicate**, and
     delete any memory this session proved wrong.
   - Add the one-line pointer to `MEMORY.md` for anything new.
   - Don't record what the repo already captures (code structure, git history,
     anything in CLAUDE.md) — memory is for what isn't derivable from the files.

   The split: **skill = how to do the work; memory = how she wants it done and
   what's already been decided.** When something could go in either, prefer the
   skill — it loads with the task rather than by recall.

7. **Don't touch `index.html`** in a pure close-out (no `APP_VERSION` bump),
   unless code changes are still pending — finish those via Mode A first.

8. **Commit + push the docs** in one commit (the skill file goes in the same
   commit; memory lives outside the repo and isn't committed). Docs-only →
   `✅ No export needed`. Use the standard commit convention and trailer.

9. **Report** a short summary of what the docs now reflect, and state explicitly
   what was written to skills and to memory — she can't see either from the
   commit alone.

The goal: after close-out, CLAUDE.md and DOCUMENTATION.md describe the app
exactly as it now behaves, the skill describes how to change it safely, memory
holds the decisions that shouldn't be re-argued, and the Known Issues list
matches reality — so the next session starts from accurate ground and nothing
learned this session has to be rediscovered.
