---
phase: 01-application-shell-ui-foundation
plan: "02"
subsystem: frontend-layout
tags: [layout, xterm, terminal, react, tailwind]
dependency_graph:
  requires: ["01-01"]
  provides: ["ThreeColumnLayout", "TerminalPreview", "StatusBar", "TerminalTabList", "TerminalTab"]
  affects: ["frontend/src/App.tsx"]
tech_stack:
  added: ["@xterm/xterm", "@xterm/addon-fit", "@xterm/addon-canvas", "@testing-library/dom"]
  patterns: ["useRef+useEffect imperative lifecycle", "ResizeObserver for terminal auto-resize", "class-based vitest mocks for DOM-heavy libs"]
key_files:
  created:
    - frontend/src/components/layout/ThreeColumnLayout.tsx
    - frontend/src/components/layout/StatusBar.tsx
    - frontend/src/components/terminal/TerminalTabList.tsx
    - frontend/src/components/terminal/TerminalTab.tsx
    - frontend/src/components/terminal/TerminalPreview.tsx
    - frontend/src/components/__tests__/ThreeColumnLayout.test.tsx
  modified:
    - frontend/src/App.tsx
    - frontend/package.json
decisions:
  - "ResizeObserver and xterm.js mocks require class syntax (not vi.fn().mockImplementation) in vitest 4.x"
  - "CanvasAddon must be loaded after term.open() — enforced in implementation per plan spec"
  - "@testing-library/dom missing peer dep installed as devDependency"
metrics:
  duration_minutes: 15
  completed: "2026-03-27T07:15:33Z"
  tasks_completed: 2
  tasks_total: 2
  files_created: 6
  files_modified: 2
---

# Phase 01 Plan 02: Three-Column Layout and xterm.js Terminal Preview Summary

**One-liner:** Three-column shell with 160px/flex/220px layout, interactive terminal tabs, xterm.js canvas preview, and status bar — all rendering from live Zustand store state.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Three-column layout, terminal tabs, and status bar | 5d70a06 | ThreeColumnLayout.tsx, StatusBar.tsx, TerminalTabList.tsx, TerminalTab.tsx, App.tsx, ThreeColumnLayout.test.tsx |
| 2 | xterm.js terminal preview with canvas addon and mock content | c7cf77a | TerminalPreview.tsx |

## What Was Built

**ThreeColumnLayout** (`frontend/src/components/layout/ThreeColumnLayout.tsx`):
- Root flex layout: `h-screen w-screen` with left (w-40), center (flex-1), right (w-[220px]) columns
- Left aside: `border-r border-zinc-800` — hosts TerminalTabList
- Center main: upper flex area for children + lower `h-[30%]` section for TerminalPreview
- Right aside: `border-l border-zinc-800` — hosts sidebar prop
- StatusBar rendered below the three-column flex row
- Reads `activeTabId` from `useTerminalStore` to pass to TerminalPreview

**TerminalTabList** (`frontend/src/components/terminal/TerminalTabList.tsx`):
- Reads `tabs` and `activeTabId` from `useTerminalStore`
- "Terminals" header with zinc-500 uppercase tracking
- Maps tabs to TerminalTab components with active state
- Disabled "+ New" placeholder button at bottom

**TerminalTab** (`frontend/src/components/terminal/TerminalTab.tsx`):
- Active: `bg-zinc-800 text-zinc-100 border-l-2 border-blue-500`
- Inactive: `text-zinc-400 hover:bg-zinc-900 border-l-2 border-transparent`
- Green dot indicator for active, zinc-600 for inactive
- `onClick` calls `useTerminalStore.getState().setActiveTab(tab.id)`

**StatusBar** (`frontend/src/components/layout/StatusBar.tsx`):
- 28px bottom bar: zinc-900 bg with border-t
- Left: grey dot + "No model" badge
- Center: "Disconnected" status
- Right: "0 / 0 tokens" + disabled Settings gear icon (lucide-react)

**TerminalPreview** (`frontend/src/components/terminal/TerminalPreview.tsx`):
- xterm.js Terminal with dark theme (bg #0d0d0d, fg #d4d4d4)
- FitAddon loaded pre-open, CanvasAddon loaded post-open (required order)
- Mock bash session output + "[No terminal connected — Phase 1 mock]" message
- ResizeObserver auto-fits terminal on container resize
- Full cleanup: observer.disconnect() + term.dispose() on unmount

## Test Results

```
Test Files: 4 passed (4)
Tests:     18 passed (18)
```

Includes 6 new ThreeColumnLayout tests covering:
- Three-column structure (left/center/right)
- Terminals header in left column
- Status bar "No model" text
- bash:1 and bash:2 tabs rendered
- Children prop (chat area)
- Sidebar prop (commands)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Dep] Installed @testing-library/dom missing peer dep**
- **Found during:** Task 1 test run
- **Issue:** `@testing-library/react` v16 requires `@testing-library/dom` as a peer dependency; not listed in package.json but required at runtime
- **Fix:** `npm install --save-dev @testing-library/dom`
- **Files modified:** `frontend/package.json`, `frontend/package-lock.json`
- **Commit:** 5d70a06

**2. [Rule 1 - Bug] Fixed vitest constructor mock pattern**
- **Found during:** Task 1 test run
- **Issue:** vitest 4.x `vi.fn().mockImplementation(() => ({...}))` does not work as constructor mock (throws "is not a constructor"). Both `Terminal`/`FitAddon`/`CanvasAddon` (from vi.mock factories) and `ResizeObserver` (from global assignment) required this fix.
- **Fix:** Used class syntax in `vi.mock` factories and `beforeEach` global assignment
- **Files modified:** `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx`
- **Commit:** 5d70a06

## Known Stubs

| Stub | File | Reason |
|------|------|--------|
| Chat area placeholder | `frontend/src/App.tsx:9` | Chat component implemented in Plan 04 |
| Commands sidebar placeholder | `frontend/src/App.tsx:12` | Commands sidebar implemented in Plan 04 |
| "No model" status | `frontend/src/components/layout/StatusBar.tsx:11` | LLM gateway in Phase 2 |
| "Disconnected" status | `frontend/src/components/layout/StatusBar.tsx:17` | Backend connection in Phase 2 |
| "0 / 0 tokens" | `frontend/src/components/layout/StatusBar.tsx:22` | Token counting in Phase 2 |

These stubs are intentional scaffolding — the visual shell is complete, real data wiring occurs in later phases as noted.

## Self-Check: PASSED

Files verified:
- FOUND: frontend/src/components/layout/ThreeColumnLayout.tsx
- FOUND: frontend/src/components/layout/StatusBar.tsx
- FOUND: frontend/src/components/terminal/TerminalTabList.tsx
- FOUND: frontend/src/components/terminal/TerminalTab.tsx
- FOUND: frontend/src/components/terminal/TerminalPreview.tsx
- FOUND: frontend/src/components/__tests__/ThreeColumnLayout.test.tsx
- FOUND: commit 5d70a06
- FOUND: commit c7cf77a
