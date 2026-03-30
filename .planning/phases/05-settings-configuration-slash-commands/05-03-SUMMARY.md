---
phase: 05-settings-configuration-slash-commands
plan: "03"
subsystem: backend+ui
tags: [go, react, slash-commands, settings, vitest, tdd]

# Dependency graph
requires:
  - phase: 05-settings-configuration-slash-commands
    plan: "01"
    provides: SettingsService base with injectable emitFn, AppConfig with ContextLines/Theme
  - phase: 05-settings-configuration-slash-commands
    plan: "02"
    provides: settingsStore.setActiveModel, SettingsService.js wailsjs stub

provides:
  - SettingsService.SetModel: parses provider:model, saves to AppConfig, emits settings:model-changed
  - SettingsService.SetContextLines: validates bounds (1-10000), saves ContextLines to AppConfig
  - SettingsService.ForceRefresh: delegates to CaptureManager.ForceCapture for immediate re-capture
  - SettingsService.ExportChat: writes JSON or TXT file to home directory, returns path
  - SettingsService.RenameTab: emits terminal:rename Wails event with tabId+label
  - CaptureManager.ForceCapture: immediate tick() outside 500ms poll loop
  - ChatPane.handleSend: full slash command router dispatching all 8 commands
  - frontend/wailsjs/go/services/LLMService.js: vitest stub for import resolution

affects:
  - 05-04 (Settings persistence — depends on ContextLines and Provider/Model saved by SetModel/SetContextLines)
  - 06-security-hardening (ExportChat creates files in home dir)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "captureManagerForceCapture interface on SettingsService — duck-typing consistent with filterPipelineRebuilder on LLMService (04-04)"
    - "ExportMessage struct exported for Wails binding type generation"
    - "ChatPane dynamic imports use ../../../wailsjs/... (3 levels) — correct path from src/components/chat/"
    - "vi.mock path depth: chat/__tests__/ needs ../../../../wailsjs/... (4 levels) to match component's resolved absolute path"
    - "beforeEach stores reset with pre-populated tab key to avoid Zustand getSnapshot infinite re-render loop"

key-files:
  created:
    - frontend/src/components/chat/__tests__/ChatPane.test.tsx
    - frontend/wailsjs/go/services/LLMService.js
  modified:
    - services/settings_service.go
    - services/settings_service_test.go
    - services/capture/manager.go
    - frontend/src/components/chat/ChatPane.tsx
    - main.go
    - .gitignore

decisions:
  - captureManagerForceCapture interface on SettingsService — duck-typing consistent with filterPipelineRebuilder pattern (04-04); avoids import cycle between services and capture packages
  - ExportMessage struct exported at services package level — required for Wails binding type generation; cleanly separates export payload from internal config types
  - LLMService.js wailsjs stub — same pattern as SettingsService.js (05-01); required so vitest can resolve dynamic import path from ChatPane.test.tsx
  - Import path fix in ChatPane.tsx — ../../wailsjs was wrong (resolves to src/wailsjs/ not frontend/wailsjs/); corrected to ../../../wailsjs (3 levels from src/components/chat/)
  - Zustand beforeEach initialization — messagesByTab must pre-populate the tab key; empty {} causes selector to return new [] each render → infinite Zustand re-render loop

metrics:
  duration: "84 minutes"
  completed_date: "2026-03-30"
  tasks: 2
  files_modified: 8
  tests_added: 24
  tests_total: 88

# Key decisions (frontmatter format)
key-decisions:
  - captureManagerForceCapture interface on SettingsService for /refresh command
  - LLMService.js wailsjs stub added for vitest import resolution
  - ChatPane import path corrected from ../../wailsjs to ../../../wailsjs
  - Zustand beforeEach must pre-populate tab key to avoid getSnapshot infinite loop
---

# Phase 5 Plan 3: Slash Command Router and Go Backend Methods Summary

Slash command Go backend methods on SettingsService and full 8-command frontend router in ChatPane using TDD.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 (RED) | Failing tests for Go backend methods | 4fe42ac | services/settings_service_test.go |
| 1 (GREEN) | SetModel, SetContextLines, ForceRefresh, ExportChat, RenameTab | 6d45266 | services/settings_service.go, services/capture/manager.go, main.go |
| 2 (RED) | Failing tests for ChatPane slash command router | 7bddc00 | frontend/src/components/chat/__tests__/ChatPane.test.tsx |
| 2 (GREEN) | Full slash command router in ChatPane | 32e929e | frontend/src/components/chat/ChatPane.tsx, frontend/wailsjs/go/services/LLMService.js |

## What Was Built

**Go backend (services/settings_service.go):**
- `SetModel(providerModel string) (string, error)` — splits on `:`, saves Provider/Model to AppConfig, emits `settings:model-changed`
- `SetContextLines(lines int) (string, error)` — validates 1-10000, saves ContextLines to AppConfig
- `ForceRefresh() (string, error)` — delegates to `CaptureManager.ForceCapture()`
- `ExportChat(tabId, format string, messages []ExportMessage) (string, error)` — writes JSON or TXT to `~/pairadmin-export-TIMESTAMP.{ext}`
- `RenameTab(tabId, label string) (string, error)` — emits `terminal:rename` Wails event
- `captureManagerForceCapture` interface and `SetCaptureManager` method

**CaptureManager (services/capture/manager.go):**
- `ForceCapture()` — calls `tick()` immediately outside the 500ms poll interval

**Frontend (frontend/src/components/chat/ChatPane.tsx):**
- `HELP_TEXT` constant listing all 9 commands
- Full slash command router handling `/clear`, `/theme`, `/help` (frontend-only), `/filter` (LLMService), `/model`, `/context`, `/refresh`, `/export`, `/rename` (SettingsService)
- `/model` also updates `settingsStore.activeModel` after Go call

## Verification

Go tests: `go test ./services/... -count=1` — 6 packages PASS
Frontend tests: `npx vitest run` — 88 tests PASS (13 new)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed incorrect wailsjs import path in ChatPane.tsx**
- **Found during:** Task 2 (test execution)
- **Issue:** `../../wailsjs/go/services/LLMService` from `src/components/chat/ChatPane.tsx` resolves to `src/wailsjs/...` (not `frontend/wailsjs/...`); should be `../../../wailsjs/...`
- **Fix:** Updated all dynamic imports in ChatPane.tsx from `../../wailsjs/...` to `../../../wailsjs/...`
- **Files modified:** `frontend/src/components/chat/ChatPane.tsx`
- **Commit:** 32e929e

**2. [Rule 2 - Missing infrastructure] Added LLMService.js wailsjs stub**
- **Found during:** Task 2 (vitest import resolution)
- **Issue:** `frontend/wailsjs/go/services/LLMService.js` did not exist in worktree; vite's import analysis fails without a physical file even with `/* @vite-ignore */`
- **Fix:** Created stub with `SendMessage`, `FilterCommand`, `Startup` exports; added gitignore exception
- **Files modified:** `frontend/wailsjs/go/services/LLMService.js`, `.gitignore`
- **Commit:** 32e929e

**3. [Rule 1 - Bug] Fixed Zustand getSnapshot infinite re-render in tests**
- **Found during:** Task 2 (test execution)
- **Issue:** `useChatStore.setState({ messagesByTab: {} })` in beforeEach leaves no key for `test-tab`; selector `messagesByTab[activeTabId] ?? []` returns new `[]` on every render → React "Maximum update depth exceeded"
- **Fix:** Changed beforeEach to `useChatStore.setState({ messagesByTab: { "test-tab": [] } })`
- **Files modified:** `frontend/src/components/chat/__tests__/ChatPane.test.tsx`
- **Commit:** 32e929e

## Known Stubs

None — all slash commands are fully wired. No placeholder text or hardcoded empty values in the delivered artifacts.

## Self-Check: PASSED

All created files confirmed:
- FOUND: services/settings_service.go (SetModel, SetContextLines, ForceRefresh, ExportChat, RenameTab)
- FOUND: services/capture/manager.go (ForceCapture)
- FOUND: frontend/src/components/chat/ChatPane.tsx (HELP_TEXT, full router)
- FOUND: frontend/src/components/chat/__tests__/ChatPane.test.tsx (13 tests)
- FOUND: frontend/wailsjs/go/services/LLMService.js (stub)

All commits confirmed: 4fe42ac, 6d45266, 7bddc00, 32e929e
