---
name: app-update
description: Workflow for changing the Home Tasks app (index.html) and for closing out a work session. Use when making any edit to the app — fixing a bug, adding a task or preset, changing scoring/dealing/UI — and especially when the user says "close out", "wrap up", "close out the session", or otherwise signals the session is done and the docs should be reconciled. On close-out it updates CLAUDE.md, DOCUMENTATION.md, and IMPROVEMENT_PLAN.md from that session's changes and learnings.
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

3. **Bump `APP_VERSION`** to today's date (`const APP_VERSION = 'YYYY-MM-DD';`)
   whenever you touch `index.html`, so a Reload in Settings can be confirmed to
   have taken. (Standing user instruction.)

4. **Verify in the preview** before committing — never ask the user to check
   manually:
   - `preview_start` with the `hometasks` config (python http.server on 7821).
   - Navigate to `/index.html`, then `preview_eval` to exercise the change and
     `preview_console_logs level:error` to confirm no errors.
   - For data/preset changes, a quick Python scan of `index.html` to confirm
     referenced task ids still resolve (see the dead-ref check below).

5. **Commit + push automatically** — no confirmation needed (standing user
   instruction). Branch first only if not on `main` would be unusual here; this
   repo works directly on `main`.

6. **State the export status** at the end of the change description, always one
   of: `✅ No export needed` or `⚠️ Export needed before deploying`.
   - **No export needed:** UI/logic changes, and adding new state fields (give
     them a default in `defaultState()` AND in the `loadState()` migration block
     AND in `applyImport`).
   - **Export needed:** bumping `STORAGE_KEY`, a breaking field-format change, or
     intentionally clearing/restructuring state.

7. **Commit message convention:** a title plus a body, both in a single fenced
   code block, no manual line breaks in the body (let it wrap). End the message
   with the `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` trailer.

**Watch-outs when editing** (these have bitten before):
- Adding state? Update all three of `defaultState()`, `loadState()`, and
  `applyImport` or import will silently drop the field.
- Referencing a task id in a preset that renders via `getAllTasks()`
  (`ROOM_PRESETS`, `EXPRESS_RESET_SECTIONS`, `RETURN_HOME_SECTIONS`,
  `BEFORE_CLEANERS_SECTIONS`, `RECOVERY_SECTIONS`, `POST_ILLNESS_SECTIONS`,
  `EVENING_*`)? The id must exist in `TASKS` or it's silently dropped.
  `completeTask` returns early for unknown ids, so a checklist item with a
  phantom id can never be checked off. Run the dead-ref scan after preset edits.
- Don't break: laundry slot logic, `_dealPreferTier`, the starvation cap
  (`freq * 0.15`), pinned/in-progress carry-over.

Dead-reference scan (run from the repo root):

```bash
python3 - <<'PY'
import re
src=open("index.html").read(); lines=src.split("\n")
taskIds={m.group(1) for i in range(482,696) for m in [re.search(r"id:'([^']+)'",lines[i])] if m}
def block(n):
    s=src.find("const "+n); i=s; d=0; st=False
    while i<len(src):
        c=src[i]
        if c=='[': d+=1; st=True
        elif c==']':
            d-=1
            if st and d==0: return src[s:i+1]
        i+=1
rk={"quick","deep","kitchen","dsBath","usBath","bedroom","livingRoom","diningRoom","sunroom","office","backRoom","laundry","hall","mudRoom","backPorch","cats"}
for nm in ["ROOM_PRESETS","EXPRESS_RESET_SECTIONS","RETURN_HOME_SECTIONS","BEFORE_CLEANERS_SECTIONS","RECOVERY_SECTIONS","POST_ILLNESS_SECTIONS","EVENING_SHUTDOWN_FIXED","EVENING_TIDY_POOL"]:
    b=block(nm) or ""
    ids=[m.group(1) for m in re.finditer(r"'([a-z][a-z0-9_]+)'",b) if m.group(1) not in rk]
    miss=sorted({x for x in ids if x not in taskIds})
    print(nm,"->", miss or "OK")
PY
```

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

5. **Don't touch `index.html`** in a pure close-out (no `APP_VERSION` bump),
   unless code changes are still pending — finish those via Mode A first.

6. **Commit + push the docs** in one commit. Docs-only → `✅ No export needed`.
   Use the standard commit convention and the Co-Authored-By trailer.

7. **Report** a short summary of what the docs now reflect.

The goal: after close-out, CLAUDE.md and DOCUMENTATION.md describe the app
exactly as it now behaves, with the Known Issues list matching reality, so the
next session starts from accurate docs.
