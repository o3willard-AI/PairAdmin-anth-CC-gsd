---
phase: 03-tmux-terminal-capture
plan: "03"
subsystem: frontend-ui-wiring
tags: [terminal, xterm, wails-events, zustand, react]
dependency_graph:
  requires: [03-01, 03-02]
  provides: [live-terminal-ui, no-tmux-empty-state, terminal-event-subscription]
  affects: [frontend/src/components/terminal/TerminalPreview.tsx, frontend/src/components/layout/ThreeColumnLayout.tsx]
tech_stack:
  added: []
  patterns: [hook-mounted-at-layout-level, conditional-empty-state-render, event-subscription-lifecycle]
key_files:
  created: []
  modified:
    - frontend/src/components/terminal/TerminalPreview.tsx
    - frontend/src/components/layout/ThreeColumnLayout.tsx
    - frontend/src/components/layout/ThreeColumnLayout.test.tsx
decisions:
  - useTerminalCapture mounted in ThreeColumnLayout (not App.tsx) — ThreeColumnLayout already owns terminal state via useTerminalStore and renders both TerminalTabList and TerminalPreview; correct lifecycle scope
  - Empty state uses early return after all hooks — useEffect must be declared before any conditional return per React Rules of Hooks; tabId empty check placed after useEffect declaration
  - ThreeColumnLayout tests require wailsjs/runtime vi.mock — dynamic import of wailsjs/runtime/runtime fails in vitest without a stub module
key_decisions:
  - useTerminalCapture mounted in ThreeColumnLayout rather than App.tsx — ThreeColumnLayout is the true top-level owner of terminal UI
  - useEffect must precede early return in TerminalPreview — Rules of Hooks
  - ThreeColumnLayout test requires wailsjs/runtime mock when useTerminalCapture is mounted
metrics:
  duration: "~45 minutes"
  completed_date: "2026-03-29"
  tasks_completed: 3
  tasks_total: 3
  files_modified: 3
requirements: [TMUX-01, TMUX-02, TMUX-03, TMUX-04, TMUX-06]
---

# Phase 03 Plan 03: UI Wiring — Live Terminal Events Summary

**Live tmux content piped end-to-end: TerminalPreview mock data removed, no-tmux empty state added, useTerminalCapture mounted in ThreeColumnLayout, human-verified tab lifecycle and content updates**

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Remove mocks, add empty state in TerminalPreview | 38f3fe1 | frontend/src/components/terminal/TerminalPreview.tsx |
| 2 | Mount useTerminalCapture in ThreeColumnLayout | 71c0bea | frontend/src/components/layout/ThreeColumnLayout.tsx |
| Fix | Move useEffect before early return + wailsjs/runtime vitest mock | 6791dbc | TerminalPreview.tsx, ThreeColumnLayout.test.tsx |
| 3 | Human verify end-to-end live tmux capture | approved | checkpoint:human-verify passed |

## What Was Built

**Task 1 — TerminalPreview changes:**
- Removed 12 lines of mock `term.writeln()` content (ls -la output, Phase 1 mock string)
- Added no-tmux empty state: when `tabId` is empty string, renders a centered instruction panel with "No tmux session detected.", "Start a tmux session to begin.", and `$ tmux new-session` code block
- Preserved xterm initialization, FitAddon, CanvasAddon (loaded after open()), ResizeObserver, setTermRef registration, cleanup
- Live content now arrives exclusively via `terminal:update` Wails events handled by useTerminalCapture

**Task 2 — ThreeColumnLayout changes:**
- Added `import { useTerminalCapture } from "@/hooks/useTerminalCapture"`
- Added `useTerminalCapture()` call at top of component body (before return)
- Terminal event subscriptions (terminal:tabs, terminal:update) are now active for the entire app lifetime

## Deviations from Plan

**1. [Rule 3 - Blocking] No AppLayout.tsx exists — mounted in ThreeColumnLayout instead**
- **Found during:** Task 2
- **Issue:** Plan specified `frontend/src/components/layout/AppLayout.tsx` but this file does not exist in the codebase. The actual top-level layout is `ThreeColumnLayout.tsx`.
- **Fix:** Mounted `useTerminalCapture()` in `ThreeColumnLayout.tsx` — this component already owns the terminal state (reads activeTabId, renders TerminalTabList and TerminalPreview) making it the correct lifecycle scope.
- **Files modified:** `frontend/src/components/layout/ThreeColumnLayout.tsx`
- **Commit:** 71c0bea

**2. [Rule 3 - Blocking] node_modules not installed in git worktree**
- **Found during:** Task 1 verification
- **Issue:** Worktree has no node_modules; vitest failed with ERR_MODULE_NOT_FOUND. Main repo frontend has node_modules at `/home/sblanken/working/ppa2/frontend/node_modules`.
- **Fix:** Created symlink to parent node_modules during testing, removed after (gitignore already covers `frontend/node_modules/`). Tests confirmed passing (62/62) before removal.
- **Impact:** No code change; worktree vitest must be run with symlink or from parent repo.

**3. [Rule 1 - Bug] Rules of Hooks violation: useEffect after conditional early return**
- **Found during:** Task 2 verification — 6 tests failed with "Rendered more hooks than during previous render"
- **Issue:** TerminalPreview had the no-tmux early-return guard placed before the useEffect call; React requires all hooks called unconditionally before any conditional return
- **Fix:** Moved `if (!tabId) return ...` guard to after the useEffect declaration; xterm setup inside useEffect is a no-op when tabId is empty because effect guards on tabId internally
- **Files modified:** `frontend/src/components/terminal/TerminalPreview.tsx`
- **Commit:** 6791dbc

**4. [Rule 3 - Blocking] ThreeColumnLayout vitest failing: wailsjs/runtime unresolvable in test**
- **Found during:** Task 2 — mounting ThreeColumnLayout in tests pulled in useTerminalCapture which tries dynamic import of wailsjs/runtime
- **Issue:** wailsjs/runtime/runtime.js is Wails-generated at dev-time and not present during vitest; import fails with module not found
- **Fix:** Added `vi.mock("../../../wailsjs/runtime/runtime", () => ({ EventsOn: vi.fn(), EventsOff: vi.fn() }))` in ThreeColumnLayout.test.tsx
- **Files modified:** `frontend/src/components/layout/ThreeColumnLayout.test.tsx`
- **Commit:** 6791dbc

## Verification Results

- `npx vitest run`: 9 test files, 62 tests — all passed (0 errors after fixes)
- `go test ./services/... -count=1`: `ok pairadmin/services`, `ok pairadmin/services/llm`, `ok pairadmin/services/llm/filter` — all passed
- `go build -tags webkit2_41`: fails with "frontend/dist: no matching files found" — expected in dev without `wails dev` build step (not a code error)
- **Human verified (approved):** Empty state, tab auto-appear, content update, tab close/auto-switch, empty state reappears on tmux kill-server

## Known Stubs

None — live content now flows from Go service through Wails events to xterm.js. No mock data remains.

## Self-Check: PASSED

Files created/modified:
- FOUND: /home/sblanken/working/ppa2/.claude/worktrees/agent-a6a7e5ff/frontend/src/components/terminal/TerminalPreview.tsx
- FOUND: /home/sblanken/working/ppa2/.claude/worktrees/agent-a6a7e5ff/frontend/src/components/layout/ThreeColumnLayout.tsx
- FOUND: /home/sblanken/working/ppa2/.claude/worktrees/agent-a6a7e5ff/frontend/src/components/layout/ThreeColumnLayout.test.tsx

Commits:
- FOUND: 38f3fe1 (feat(03-03): remove mock content, add no-tmux empty state in TerminalPreview)
- FOUND: 71c0bea (feat(03-03): mount useTerminalCapture hook in ThreeColumnLayout)
- FOUND: 6791dbc (fix(03-03): move useEffect before early return and add runtime mock in test)
