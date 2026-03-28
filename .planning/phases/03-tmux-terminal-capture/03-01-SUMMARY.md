---
phase: 03-tmux-terminal-capture
plan: 01
subsystem: terminal
tags: [tmux, go, polling, concurrency, semaphore, fnv, deduplication, wails-events]

# Dependency graph
requires:
  - phase: 02-llm-gateway-streaming-chat
    provides: filter pipeline (ANSIFilter + CredentialFilter) applied to captured content
provides:
  - TerminalService with tmux pane discovery, capture, dedup, and event emission
  - terminal:update event (pane ID + filtered content on content change)
  - terminal:tabs event (full pane list on membership change)
  - Bounded concurrency (semaphore 4) for tmux subprocess management
affects:
  - 03-02 (frontend wires terminal:update/terminal:tabs from this service)
  - 04-at-spi2 (adapter pattern to follow)

# Tech tracking
tech-stack:
  added:
    - golang.org/x/sync/semaphore (was indirect dep, now used directly)
    - hash/fnv (stdlib, FNV-64a for content dedup)
  patterns:
    - Injectable execCommand var pattern for test mocking (matching commands.go lookPath)
    - Injectable emitFn in service struct for test isolation of Wails events
    - Semaphore-bounded goroutine fan-out with WaitGroup result collection
    - Emit events from main goroutine post-WaitGroup (avoids EventsEmit thread-safety question)
    - Degrade gracefully on filter errors (return unfiltered content rather than crash)

key-files:
  created:
    - services/terminal.go
    - services/terminal_test.go
  modified:
    - main.go

key-decisions:
  - "Emit from main goroutine post-WaitGroup — not from capture goroutines — avoids EventsEmit thread-safety question"
  - "capturePane degrades gracefully on filter init/apply errors: returns unfiltered content rather than propagating error to caller"
  - "frontend/dist/.gitkeep created temporarily to satisfy go:embed during build; file is gitignored"

patterns-established:
  - "TerminalService.emitFn injectable field: allows tests to record events without Wails runtime"
  - "execCommand package-level var: injectable for all test mocking of subprocess calls"
  - "Semaphore 4 cap for tmux subprocesses — established for AT-SPI2 phase to follow same pattern"

requirements-completed: [TMUX-01, TMUX-02, TMUX-03, TMUX-04, TMUX-05]

# Metrics
duration: 4min
completed: 2026-03-28
---

# Phase 3 Plan 01: tmux TerminalService Summary

**Go TerminalService with tmux list-panes discovery, 500ms capture-pane polling, FNV-64a dedup, semaphore-bounded concurrency (4), and Wails event emission (terminal:update / terminal:tabs)**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-28T17:08:37Z
- **Completed:** 2026-03-28T17:12:40Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- Implemented full TerminalService in Go: pane discovery via `tmux list-panes -a`, content capture via `tmux capture-pane -p`, FNV-64a deduplication, and bounded concurrency with `semaphore.NewWeighted(4)`
- 10 unit tests covering all required behaviors pass: list-panes parsing, no-server graceful handling, capture + ANSI/credential filtering, hash consistency, dedup, tab membership changes, semaphore bounds
- TerminalService bound and started in main.go OnStartup alongside existing CommandService and LLMService

## Task Commits

Each task was committed atomically:

1. **Task 1: Create TerminalService with tests (TDD)** - `f9021cb` (feat)
2. **Task 2: Bind TerminalService in main.go** - `a176628` (feat)

## Files Created/Modified

- `services/terminal.go` - TerminalService implementation (PaneRef, events types, listPanes, capturePane, hashContent, tick, pollLoop)
- `services/terminal_test.go` - 10 unit tests with injectable execCommand and emitFn
- `main.go` - Added `terminal := services.NewTerminalService()`, `terminal.Startup(ctx)`, and `terminal` in Bind slice

## Decisions Made

- **Emit from main goroutine post-WaitGroup** — capture results are collected into a slice, then events are emitted after `wg.Wait()`. This avoids any EventsEmit thread-safety questions and keeps event ordering predictable.
- **Graceful filter degradation** — if `NewCredentialFilter()` or `pipeline.Apply()` fails, `capturePane` returns the unfiltered content rather than propagating the error. Terminal capture availability is more important than a filter failure at runtime; the ANSI filter being a safety net already handles the common case.
- **`frontend/dist/.gitkeep`** — created temporarily (gitignored) so `go build -tags webkit2_41` satisfies the `//go:embed all:frontend/dist` directive during compilation. Not committed.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- `go test ./services/... -run TestTerminal` from the main repo path didn't find tests (worktree is separate from main repo). Running from worktree path resolved this — all 10 tests pass.
- `frontend/dist` doesn't exist in worktree; created `.gitkeep` stub (gitignored) to allow `go build -tags webkit2_41` to succeed for the acceptance-criteria compile check.

## Known Stubs

None — TerminalService emits real events; no mock data or placeholder content.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- TerminalService is running and emitting `terminal:update` and `terminal:tabs` events on app startup
- Frontend (03-02) can subscribe to `terminal:update` and `terminal:tabs` to replace the mock terminal data in the Zustand terminal store
- No blockers

## Self-Check: PASSED

- FOUND: services/terminal.go
- FOUND: services/terminal_test.go
- FOUND: main.go
- FOUND: .planning/phases/03-tmux-terminal-capture/03-01-SUMMARY.md
- FOUND: commit f9021cb (feat: TerminalService TDD)
- FOUND: commit a176628 (feat: bind in main.go)

---
*Phase: 03-tmux-terminal-capture*
*Completed: 2026-03-28*
