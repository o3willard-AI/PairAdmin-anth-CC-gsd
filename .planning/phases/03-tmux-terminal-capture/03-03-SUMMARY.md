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
decisions:
  - useTerminalCapture mounted in ThreeColumnLayout (not App.tsx) — ThreeColumnLayout already owns terminal state via useTerminalStore and renders both TerminalTabList and TerminalPreview; correct lifecycle scope
  - Empty state uses early return before useEffect — React rules of hooks require conditional returns only before hooks; tabId check placed before useEffect call (uses ref pattern that satisfies linter)
key_decisions:
  - useTerminalCapture mounted in ThreeColumnLayout rather than App.tsx — ThreeColumnLayout is the true top-level owner of terminal UI
metrics:
  duration: "~3 minutes"
  completed_date: "2026-03-29"
  tasks_completed: 2
  tasks_total: 3
  files_modified: 2
requirements: [TMUX-01, TMUX-02, TMUX-03, TMUX-04, TMUX-06]
---

# Phase 03 Plan 03: UI Wiring — Live Terminal Events Summary

Wire TerminalService Wails events into xterm.js display: remove mock data, add no-tmux empty state in TerminalPreview, mount useTerminalCapture in ThreeColumnLayout layout component.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Remove mocks, add empty state in TerminalPreview | 38f3fe1 | frontend/src/components/terminal/TerminalPreview.tsx |
| 2 | Mount useTerminalCapture in ThreeColumnLayout | 71c0bea | frontend/src/components/layout/ThreeColumnLayout.tsx |
| 3 | Human verify end-to-end live tmux capture | — | (checkpoint — awaiting human) |

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

## Verification Results

- `npx vitest run` (worktree with symlinked node_modules): 9 test files, 62 tests — all passed
- `go test ./services/... -count=1`: `ok pairadmin/services`, `ok pairadmin/services/llm`, `ok pairadmin/services/llm/filter` — all passed
- `go build -tags webkit2_41`: fails with "frontend/dist: no matching files found" — expected in dev without `wails dev` build step (not a code error)

## Known Stubs

None — live content now flows from Go service through Wails events to xterm.js. No mock data remains.

## Checkpoint Pending

Task 3 is a `checkpoint:human-verify` gate requiring human validation of:
1. No-tmux empty state displays when tmux not running
2. Tabs auto-appear within ~500ms when tmux session starts
3. Terminal preview shows live pane content
4. Tab closes when pane exits, active tab auto-switches
5. Empty state reappears when all tmux sessions killed

## Self-Check: PASSED

Files created/modified:
- FOUND: /home/sblanken/working/ppa2/.claude/worktrees/agent-a6a7e5ff/frontend/src/components/terminal/TerminalPreview.tsx
- FOUND: /home/sblanken/working/ppa2/.claude/worktrees/agent-a6a7e5ff/frontend/src/components/layout/ThreeColumnLayout.tsx

Commits:
- FOUND: 38f3fe1 (feat(03-03): remove mock content, add no-tmux empty state in TerminalPreview)
- FOUND: 71c0bea (feat(03-03): mount useTerminalCapture hook in ThreeColumnLayout)
