---
phase: 04-linux-gui-terminal-adapters-at-spi2
plan: "01"
subsystem: terminal-capture
tags: [go, refactor, adapter-pattern, tmux, capture-manager]
dependency_graph:
  requires: []
  provides: [services/capture/adapter.go, services/capture/manager.go, services/capture/tmux.go]
  affects: [main.go, services/terminal.go (deleted)]
tech_stack:
  added: []
  patterns: [TerminalAdapter interface, CaptureManager multi-adapter poll loop, injectable execCommand struct field]
key_files:
  created:
    - services/capture/adapter.go
    - services/capture/manager.go
    - services/capture/manager_test.go
    - services/capture/tmux.go
    - services/capture/tmux_test.go
  modified:
    - main.go
  deleted:
    - services/terminal.go
    - services/terminal_test.go
decisions:
  - "CaptureManager.active populated by Startup() IsAvailable check; test helper bypasses via direct field set"
  - "execCommand is a TmuxAdapter struct field (not package-level var) enabling per-instance test injection"
  - "Pane IDs namespaced: tmux:%N; frontend events unchanged (terminal:tabs, terminal:update same JSON structs)"
  - "applyFilterPipeline in manager.go applies ANSI + credential filter; degraded behavior on filter failure"
  - "terminal was removed from Wails Bind slice since CaptureManager has no bound methods (events-only)"
metrics:
  duration_minutes: 4
  completed_date: "2026-03-29T05:49:11Z"
  tasks_completed: 2
  files_created: 5
  files_modified: 1
  files_deleted: 2
---

# Phase 04 Plan 01: CaptureManager Multi-Adapter Architecture Summary

**One-liner:** Refactored monolithic TerminalService into services/capture/ package with TerminalAdapter interface, CaptureManager poll loop, and TmuxAdapter with "tmux:" namespaced pane IDs.

## What Was Built

The existing `services/terminal.go` TerminalService was decomposed into three files in a new `services/capture/` package:

1. **`services/capture/adapter.go`** — Defines `TerminalAdapter` interface and shared types: `PaneInfo` (with `ID`, `AdapterType`, `DisplayName`, `Degraded`, `DegradedMsg`), `TerminalTabsEvent`, `TerminalUpdateEvent`, `TabInfo`. JSON tags preserved for frontend compatibility.

2. **`services/capture/manager.go`** — `CaptureManager` owns multiple adapters. `Startup(ctx)` filters adapters by `IsAvailable()`, then starts a 500ms poll loop. `tick()` calls `Discover()` on all active adapters, merges pane lists, detects membership changes, emits `terminal:tabs` on change, captures non-degraded panes concurrently (semaphore 4), FNV-64a deduplicates, emits `terminal:update` on content change. `applyFilterPipeline()` applies ANSI strip + credential redaction with graceful degradation.

3. **`services/capture/tmux.go`** — `TmuxAdapter` implements `TerminalAdapter`: `IsAvailable()` runs `tmux list-panes` and returns false on "no server running"/"error connecting"; `Discover()` returns `PaneInfo` with `"tmux:"` prefixed IDs; `Capture()` strips prefix before passing to `tmux capture-pane` and applies filter pipeline. `execCommand` is a struct field for per-instance test injection.

4. **`main.go`** — Updated to create `capture.NewTmuxAdapter()` + `capture.NewCaptureManager(...)` replacing `services.NewTerminalService()`. `CaptureManager` removed from Wails `Bind` (events-only, no RPC methods exposed).

5. **Deleted** `services/terminal.go` and `services/terminal_test.go` — all logic and tests migrated to `services/capture/`.

## Test Coverage

- **manager_test.go**: 5 tests — discover+tabs, dedup, two-adapter merge, degraded adapter graceful degradation, skip degraded panes in capture
- **tmux_test.go**: 13 tests — IsAvailable true/false, Discover (normal + no-server + connection-error), Capture, Name, hashContent, dedup, dedup-changed, new-pane tabs, removed-pane tabs, semaphore bounds

Total: 18 tests, all passing.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Test helper needed to bypass Startup() for unit tests**
- **Found during:** Task 1 RED phase
- **Issue:** `newTestCaptureManager` bypassed `Startup()` but `tick()` relied on `m.active` which is populated by `Startup()`. Tests failing with 0 events emitted.
- **Fix:** Added `m.active = adapters` line to `newTestCaptureManager` to pre-populate active adapters; all 5 tests passed after fix.
- **Files modified:** services/capture/manager_test.go
- **Commit:** e4136fa

**2. [Rule 2 - Missing Wails Bind entry removed] CaptureManager not bound to Wails**
- **Found during:** Task 2
- **Issue:** The old `terminal` was in Wails `Bind` but `CaptureManager` has no exported methods intended for Wails RPC (it is events-only). Adding it to Bind would expose no meaningful methods.
- **Fix:** Removed from Bind slice; only `app`, `commands`, and `llmService` remain in Bind.
- **Files modified:** main.go

## Known Stubs

None — all capture functionality is fully wired.

## Self-Check: PASSED

| Check | Result |
|-------|--------|
| services/capture/adapter.go | FOUND |
| services/capture/manager.go | FOUND |
| services/capture/manager_test.go | FOUND |
| services/capture/tmux.go | FOUND |
| services/capture/tmux_test.go | FOUND |
| services/terminal.go deleted | CONFIRMED |
| services/terminal_test.go deleted | CONFIRMED |
| Task 1 commit e4136fa | FOUND |
| Task 2 commit 4c04849 | FOUND |
