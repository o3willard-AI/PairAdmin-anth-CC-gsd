---
phase: 01-application-shell-ui-foundation
plan: 03
subsystem: clipboard-service
tags: [go, clipboard, wayland, x11, wails-runtime, tdd]
dependency_graph:
  requires: ["01-01"]
  provides: ["CommandService", "CopyToClipboard", "CheckWayland", "WaylandWarning"]
  affects: ["frontend clipboard bindings via wailsjs/"]
tech_stack:
  added: []
  patterns:
    - "Injectable lookPath var for exec.LookPath to enable unit testing of system binary presence"
    - "Wails OnStartup closure to call multiple service startup methods"
key_files:
  created:
    - services/commands.go
    - services/commands_test.go
  modified:
    - main.go
decisions:
  - "Used injectable lookPath variable (var lookPath = exec.LookPath) instead of direct exec.LookPath call to allow test overrides without build tags or interfaces"
  - "OnStartup in main.go uses a closure to call both app.startup and commands.Startup in sequence"
metrics:
  duration: "~8 minutes"
  completed: "2026-03-27"
  tasks_completed: 1
  files_created: 2
  files_modified: 1
---

# Phase 1 Plan 3: CommandService — Clipboard and Wayland Detection Summary

**One-liner:** Go CommandService with dual-path clipboard (Wails ClipboardSetText for X11, wl-copy exec for Wayland) and startup Wayland detection that emits an app:warning event when wl-clipboard is missing.

## What Was Built

`services/commands.go` implements a `CommandService` struct bound to Wails:

- `CopyToClipboard(text string) error` — routes to `runtime.ClipboardSetText` on X11 or `exec.Command("wl-copy", text)` on Wayland
- `CheckWayland() *WaylandWarning` — checks `WAYLAND_DISPLAY` env var and `exec.LookPath("wl-copy")`; returns `WaylandWarning{Code: "WAYLAND_CLIPBOARD"}` when wl-copy is missing
- `Startup(ctx context.Context)` — called by Wails on startup; emits `runtime.EventsEmit(ctx, "app:warning", warning)` if Wayland but no wl-copy

`main.go` updated to create and bind CommandService, calling `commands.Startup(ctx)` alongside `app.startup(ctx)` in the `OnStartup` closure.

## Tasks

| # | Name | Type | Commit | Status |
|---|------|------|--------|--------|
| 1 | CommandService with clipboard copy and Wayland detection | auto/tdd | 54566f5 | Done |

TDD commits:
- RED: b3f0960 — failing tests for CommandService
- GREEN: 54566f5 — implementation + main.go wiring

## Deviations from Plan

### Auto-fixed Issues

None — plan executed exactly as written, with one implementation note:

**Implementation detail:** The plan's `<action>` called for `exec.LookPath` directly, but tests require overriding it (to simulate wl-copy absent/present without needing actual binary). Used an injectable `var lookPath = exec.LookPath` — a minimal Go testability pattern that avoids interfaces/mocks while enabling the required `TestWaylandDetection_WaylandMissing` test case.

## Known Stubs

None. This is a pure Go service — no UI stubs.

## Verification

- `go test ./services/... -v` — 5 PASS, 1 SKIP (wl-copy not installed in this environment)
- `go vet ./services/...` — clean
- `go build` blocked by missing `frontend/dist` (pre-existing in this worktree — frontend not built here)
- All acceptance criteria met per grep checks

## Self-Check: PASSED
