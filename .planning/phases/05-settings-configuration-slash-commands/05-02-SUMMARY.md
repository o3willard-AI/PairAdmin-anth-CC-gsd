---
phase: 05-settings-configuration-slash-commands
plan: "02"
subsystem: ui
tags: [react, zustand, base-ui, settings, dialog, tabs, vitest]

# Dependency graph
requires:
  - phase: 01-application-shell
    provides: ThreeColumnLayout, StatusBar, Zustand store patterns

provides:
  - settingsStore with activeModel and settingsOpen state
  - SettingsDialog: 5-tab modal using @base-ui/react Dialog + Tabs
  - LLMConfigTab: provider/model/masked API key/TestConnection form
  - PromptsTab: read-only system prompt + editable custom extension textarea
  - TerminalsTab: AT-SPI2 polling interval + clipboard auto-clear settings
  - HotkeysTab: keyboard shortcut capture for in-app hotkeys
  - AppearanceTab: dark/light theme toggle via useTheme() + font size input
  - StatusBar: gear icon wired to open settings modal, active model display
  - ThreeColumnLayout: SettingsDialog mounted as child component
  - SettingsService.js: wailsjs stub for vitest import resolution

affects:
  - 05-01 (Go SettingsService backend — this frontend integrates with it)
  - 06-security-hardening (keychain integration ties to LLMConfigTab SaveAPIKey calls)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "@base-ui/react/dialog and @base-ui/react/tabs subpath imports for modal + tabbed UI"
    - "Dynamic import with @vite-ignore for wailsjs RPC calls in tab components"
    - "SettingsService.js wailsjs stub committed with gitignore exception for vitest resolution"
    - "Zustand settingsStore follows chatStore/terminalStore pattern (devtools + immer)"

key-files:
  created:
    - frontend/src/stores/settingsStore.ts
    - frontend/src/components/settings/SettingsDialog.tsx
    - frontend/src/components/settings/LLMConfigTab.tsx
    - frontend/src/components/settings/PromptsTab.tsx
    - frontend/src/components/settings/TerminalsTab.tsx
    - frontend/src/components/settings/HotkeysTab.tsx
    - frontend/src/components/settings/AppearanceTab.tsx
    - frontend/src/components/settings/__tests__/SettingsDialog.test.tsx
    - frontend/wailsjs/go/services/SettingsService.js
  modified:
    - frontend/src/components/layout/StatusBar.tsx
    - frontend/src/components/layout/ThreeColumnLayout.tsx
    - frontend/src/components/__tests__/ThreeColumnLayout.test.tsx
    - .gitignore

key-decisions:
  - "SettingsService.js wailsjs stub at flat path (not subdirectory) — matches wailsjs generation pattern; dynamic import path ../../../wailsjs/go/services/SettingsService resolves correctly from tab components"
  - "gitignore !frontend/wailsjs/go/services/SettingsService.js exception — same pattern as CaptureManager.js stub, force-added since wailsjs/ directory is excluded"
  - "node_modules symlinked from main repo for worktree vitest execution — avoids redundant npm install; added to git info/exclude"

patterns-established:
  - "Settings tabs use dynamic @vite-ignore imports for all wailsjs RPC calls — consistent with useTerminalCapture and ThreeColumnLayout patterns"
  - "Each settings tab has its own Save button calling SaveSettings — no global form submission"
  - "SettingsDialog tests mock wailsjs/go/services/SettingsService via vi.mock at 4-level relative path from __tests__ subdirectory"

requirements-completed: [CFG-01, CFG-06, CFG-07]

# Metrics
duration: 39min
completed: 2026-03-30
---

# Phase 05 Plan 02: Settings UI Summary

**5-tab settings modal (LLM Config, Prompts, Terminals, Hotkeys, Appearance) with settingsStore, StatusBar gear icon wiring, and @base-ui/react Dialog + Tabs**

## Performance

- **Duration:** 39 min
- **Started:** 2026-03-30T01:00:43Z
- **Completed:** 2026-03-30T01:39:19Z
- **Tasks:** 2
- **Files modified:** 13

## Accomplishments

- Settings modal opens from StatusBar gear icon via settingsStore open state
- SettingsDialog renders 5 tabs using @base-ui/react Dialog + Tabs with correct subpath imports
- LLMConfigTab: provider dropdown (5 providers), model input, masked API key with stored placeholder, Test Connection with inline status, Save wires to settingsStore.setActiveModel
- AppearanceTab uses existing useTheme() hook for dark/light toggle with active-state styling
- HotkeysTab captures keyboard combinations via keydown event listener (Ctrl/Shift/Alt/Meta + key)
- All 75 vitest tests pass (68 pre-existing + 7 new)

## Task Commits

Each task was committed atomically:

1. **Task 1: settingsStore + SettingsDialog root + all 5 tab components** - `5b9a093` (feat)
2. **Task 2: Wire StatusBar gear icon + mount SettingsDialog in ThreeColumnLayout + tests** - `6318fec` (feat)

**Plan metadata:** (see final docs commit)

## Files Created/Modified

- `frontend/src/stores/settingsStore.ts` - Zustand+Immer store: activeModel, settingsOpen, setters
- `frontend/src/components/settings/SettingsDialog.tsx` - Modal root with @base-ui/react Dialog + 5-tab Tabs
- `frontend/src/components/settings/LLMConfigTab.tsx` - Provider/model/masked-key form + TestConnection + Save
- `frontend/src/components/settings/PromptsTab.tsx` - Read-only system prompt + editable custom extension
- `frontend/src/components/settings/TerminalsTab.tsx` - AT-SPI2 polling interval + clipboard auto-clear settings
- `frontend/src/components/settings/HotkeysTab.tsx` - Keyboard shortcut capture inputs with keydown listener
- `frontend/src/components/settings/AppearanceTab.tsx` - Dark/light theme toggle via useTheme() + font size
- `frontend/src/components/settings/__tests__/SettingsDialog.test.tsx` - 6 tests: open/closed, all 5 tabs, tab switching
- `frontend/wailsjs/go/services/SettingsService.js` - Wailsjs stub for vitest import resolution
- `frontend/src/components/layout/StatusBar.tsx` - Gear icon wired; activeModel || "No model" display
- `frontend/src/components/layout/ThreeColumnLayout.tsx` - SettingsDialog mounted as child component
- `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx` - Added SettingsDialog test + SettingsService mock
- `.gitignore` - Added SettingsService.js and CaptureManager.js stub exceptions

## Decisions Made

- SettingsService.js stub at flat path `frontend/wailsjs/go/services/SettingsService.js` (not nested subdirectory) — matches wailsjs generation conventions; dynamic import path `../../../wailsjs/go/services/SettingsService` resolves correctly from component files at `src/components/settings/`
- gitignore `!frontend/wailsjs/go/services/SettingsService.js` exception added — wailsjs/ directory is globally excluded; must force-add individual stubs same as CaptureManager.js
- node_modules symlinked from main repo (`/home/sblanken/working/ppa2/frontend/node_modules`) for worktree vitest execution; added to `$GIT_DIR/info/exclude`

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- **node_modules missing in worktree:** Worktrees don't inherit node_modules. Resolved by symlinking from main repo. Added to git info/exclude to avoid tracking.
- **SettingsService.js gitignore:** The `frontend/wailsjs/` pattern excludes all wailsjs files. Required `git add -f` to force-add the stub, same pattern established by CaptureManager.js in Phase 04.

## User Setup Required

None — this plan is pure frontend UI. Go backend integration (Plan 05-01) provides the actual RPC implementations.

## Next Phase Readiness

- Settings UI fully functional when paired with Plan 05-01 Go backend
- settingsStore.setActiveModel() ready to receive model string from LLMConfigTab Save
- SettingsService.js stub enables full vitest coverage without live Wails runtime
- Appearance/theme toggle works independently (no backend needed)

## Self-Check: PASSED

All 10 created/modified files found. Both task commits verified (5b9a093, 6318fec).

---
*Phase: 05-settings-configuration-slash-commands*
*Completed: 2026-03-30*
