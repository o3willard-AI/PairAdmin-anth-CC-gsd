---
phase: 03-tmux-terminal-capture
plan: 02
subsystem: ui
tags: [zustand, immer, xterm, wails, react, typescript, hooks]

# Dependency graph
requires:
  - phase: 01-application-shell
    provides: terminalStore foundation with setActiveTab/setTermRef/getTermRef, xterm.js integration
  - phase: 03-tmux-terminal-capture
    plan: 01
    provides: Go TerminalService emitting terminal:tabs and terminal:update Wails events
provides:
  - terminalStore addTab/removeTab/clearTabs lifecycle actions with empty initial state
  - useTerminalCapture hook subscribing to terminal:tabs and terminal:update Wails events
  - Tab reconciliation logic (diff-based add/remove from event payload)
  - xterm content write via term.clear() + term.write() with null guard
affects: [03-tmux-terminal-capture plan 03 UI wiring, any component consuming terminalStore tabs]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Zustand + Immer store actions for tab lifecycle with external Map for xterm refs"
    - "Dynamic Wails runtime import with @vite-ignore comment (matches useLLMStream pattern)"
    - "Tab reconciliation via Set diff (currentIds vs newIds) rather than full replace"

key-files:
  created:
    - frontend/src/hooks/useTerminalCapture.ts
  modified:
    - frontend/src/stores/terminalStore.ts
    - frontend/src/stores/__tests__/terminalStore.test.ts
    - frontend/src/components/__tests__/ThreeColumnLayout.test.tsx

key-decisions:
  - "Empty initial store state: tabs start empty, populated dynamically via terminal:tabs Wails events"
  - "addTab first-tab-becomes-active: when tabs was empty, first addTab call sets activeTabId"
  - "removeTab active-tab auto-switch: switches to first remaining tab or empty string per D-06"
  - "term.clear() + term.write() (not writeln): full pane content replace on each terminal:update"
  - "Null guard on getTermRef in terminal:update: discard if tab already removed"

patterns-established:
  - "Tab lifecycle: addTab/removeTab/clearTabs with termRefsMap cleanup co-located in store"
  - "Wails event hook: useEffect + dynamic import + unsub cleanup (same as useLLMStream)"

requirements-completed: [TMUX-06]

# Metrics
duration: 4min
completed: 2026-03-28
---

# Phase 3 Plan 02: Terminal Store Actions and Capture Hook Summary

**terminalStore extended with addTab/removeTab/clearTabs actions plus useTerminalCapture hook subscribing to Wails terminal:tabs and terminal:update events**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-28T17:08:28Z
- **Completed:** 2026-03-28T17:12:18Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Extended terminalStore with addTab (duplicate guard, first-tab-becomes-active), removeTab (auto-switch active tab, termRefsMap cleanup), and clearTabs (full reset)
- Changed initial store state from mock data (bash-1/bash-2) to empty — tabs now populated dynamically via Wails events
- Created useTerminalCapture hook following useLLMStream pattern: dynamic import of wailsjs/runtime, EventsOn subscriptions, cleanup on unmount
- terminal:tabs handler diffs current store tabs against event payload using Set operations
- terminal:update handler writes to xterm via term.clear()+term.write() with null guard

## Task Commits

Each task was committed atomically:

1. **Task 1: Extend terminalStore with addTab, removeTab, clearTabs** - `07729d2` (feat)
2. **Task 2: Create useTerminalCapture hook** - `8b3c908` (feat)

## Files Created/Modified
- `frontend/src/stores/terminalStore.ts` - Added addTab/removeTab/clearTabs; changed initial state to empty
- `frontend/src/stores/__tests__/terminalStore.test.ts` - Replaced 3 tests with 15 tests covering all new actions
- `frontend/src/hooks/useTerminalCapture.ts` - New hook for Wails terminal event subscriptions
- `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx` - Updated tab test to reflect empty initial state

## Decisions Made
- Empty initial store state is correct: tabs are added dynamically via the terminal:tabs event from Go TerminalService, not hardcoded
- First addTab call sets activeTabId so first real tmux pane is immediately active
- removeTab on active tab auto-switches to first remaining tab (empty string if last tab removed) — per D-06 decision

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated ThreeColumnLayout test to reflect empty initial store state**
- **Found during:** Task 2 (full vitest run verification)
- **Issue:** ThreeColumnLayout.test.tsx had a test asserting `bash:1` and `bash:2` tabs existed, which relied on the old mock initial state we intentionally removed in Task 1
- **Fix:** Updated test to assert the negative (queryByText returns null) with a comment explaining that tabs are now added dynamically via Wails events
- **Files modified:** frontend/src/components/__tests__/ThreeColumnLayout.test.tsx
- **Verification:** Full test suite passes 62/62 tests
- **Committed in:** 8b3c908 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug - test relied on removed mock state)
**Impact on plan:** Auto-fix necessary for correctness. No scope creep.

## Issues Encountered
- npm dependencies not installed in worktree — ran `npm install` before tests could run (Rule 3 blocking fix, transparent)

## Known Stubs
None - store actions are fully implemented; useTerminalCapture subscribes to real Wails runtime events.

## Next Phase Readiness
- terminalStore has full tab lifecycle: Plan 03 (UI components) can call addTab/removeTab/clearTabs
- useTerminalCapture hook is ready to mount in App.tsx or root component (Plan 03)
- xterm write path is wired: terminal:update events will write to visible xterm instances once Plan 03 mounts TerminalPreview with refs

## Self-Check: PASSED

- frontend/src/stores/terminalStore.ts — FOUND
- frontend/src/stores/__tests__/terminalStore.test.ts — FOUND
- frontend/src/hooks/useTerminalCapture.ts — FOUND
- .planning/phases/03-tmux-terminal-capture/03-02-SUMMARY.md — FOUND
- Commit 07729d2 — FOUND
- Commit 8b3c908 — FOUND

---
*Phase: 03-tmux-terminal-capture*
*Completed: 2026-03-28*
