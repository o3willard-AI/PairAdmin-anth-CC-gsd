---
phase: "04-linux-gui-terminal-adapters-at-spi2"
plan: "02"
subsystem: "services/capture"
tags: ["atspi", "dbus", "accessibility", "terminal-capture", "gnome-terminal"]
dependency_graph:
  requires: ["04-01"]
  provides: ["ATSPIAdapter", "GetAdapterStatus"]
  affects: ["main.go", "services/capture/manager.go"]
tech_stack:
  added: ["github.com/godbus/dbus/v5 (promoted from indirect to direct use)"]
  patterns: ["injectable function fields for testability", "two-step D-Bus connection (session bus -> a11y bus)", "role-based accessibility object filtering"]
key_files:
  created:
    - "services/capture/atspi.go"
    - "services/capture/atspi_test.go"
  modified:
    - "main.go"
    - "services/capture/manager.go"
decisions:
  - "Injectable function fields pattern (getA11yAddress, listBusNames, getCacheItems, getText, gsettingsOutput, closeConn) used instead of interface mocking — avoids dbus mock framework, consistent with TmuxAdapter's execCommand pattern"
  - "IsAvailable checks org.a11y.Bus.GetAddress not IsEnabled — bus works even when GSettings=false per research anti-pattern"
  - "OnboardingRequired as exported method — CaptureManager uses duck-typing interface to query it without coupling"
  - "GetAdapterStatus added to CaptureManager and exposed as Wails binding — enables frontend onboarding flow"
  - "manager added to Wails Bind list — fulfills plan 02 acceptance criteria (was events-only in plan 01)"
metrics:
  duration: "294 seconds"
  completed_date: "2026-03-29"
  tasks_completed: 2
  files_modified: 4
---

# Phase 04 Plan 02: AT-SPI2 Adapter Summary

**One-liner:** ATSPIAdapter using godbus D-Bus with injectable function fields discovers GNOME Terminal via role=59 and captures via GetText(0,-1); wired into CaptureManager with GetAdapterStatus Wails binding for onboarding.

## Tasks Completed

| Task | Description | Commit |
|------|-------------|--------|
| 1 | ATSPIAdapter implementation + 10 unit tests | a76c50b |
| 2 | Wire ATSPIAdapter into CaptureManager + GetAdapterStatus | 61c109f |

## What Was Built

### Task 1: ATSPIAdapter (services/capture/atspi.go)

Implements the `TerminalAdapter` interface for AT-SPI2 accessibility bus:

- **`IsAvailable`**: Connects to session bus, calls `org.a11y.Bus.GetAddress`. Returns true if a non-empty address is returned (bus reachable). Caches address for subsequent calls.
- **`OnboardingRequired`**: Runs `gsettings get org.gnome.desktop.interface toolkit-accessibility` via injectable `gsettingsOutput`. Returns true if output is not "true".
- **`Discover`**: Enumerates unique names (`:` prefix) on the accessibility bus via `org.freedesktop.DBus.ListNames`. For each name, calls `org.a11y.atspi.Cache.GetItems` and filters items by `Role == RoleTerminal (59)`. Returns `PaneInfo` with `atspi:<busName><path>` IDs.
- **`Capture`**: Parses pane ID to extract bus name and object path, calls `org.a11y.atspi.Text.GetText(0, -1)`, applies ANSI + credential filter pipeline.
- **`Close`**: Calls injectable `closeConn` if set.

Test strategy: 10 unit tests using injectable function fields (`getA11yAddress`, `listBusNames`, `getCacheItems`, `getText`, `gsettingsOutput`, `closeConn`) — no live D-Bus session required.

### Task 2: CaptureManager + main.go wiring

- **main.go**: Added `atspiAdapter := capture.NewATSPIAdapter()` alongside `tmuxAdapter`; both passed to `NewCaptureManager`. Added `manager` to Wails `Bind` list.
- **manager.go**: Added `AdapterStatusInfo` struct (`json:"name"`, `json:"status"`, `json:"message"`) and `GetAdapterStatus()` method. Uses duck-typing to query `OnboardingRequired` on adapters that support it.

## Verification Results

```
go test ./services/capture/... -run TestATSPI -count=1 -v  →  10/10 PASS
go test ./... -count=1                                      →  all packages PASS
go vet ./...                                                →  no errors
grep "NewATSPIAdapter" main.go                             →  found
grep "GetAdapterStatus" services/capture/manager.go        →  found
```

## Decisions Made

1. **Injectable function fields pattern** — chose the simpler approach from the plan's "Alternative simpler approach" section over a full dbus mock interface. Consistent with `TmuxAdapter.execCommand` pattern established in Plan 01.

2. **IsAvailable checks GetAddress, not IsEnabled** — per RESEARCH.md anti-pattern guidance: the bus is reachable and functional even when `IsEnabled=false`. Checking `GetAddress` is the correct availability signal.

3. **OnboardingRequired as duck-typed interface** — `CaptureManager.GetAdapterStatus()` checks `a.(interface{ OnboardingRequired(ctx) bool })` without importing atspi-specific types. Keeps manager.go decoupled from adapter implementation details.

4. **manager added to Wails Bind** — the plan explicitly requires this in Task 2. Note: Plan 01 decision "CaptureManager not in Wails Bind" applied to events-only; Plan 02 adds `GetAdapterStatus` as a queryable RPC, requiring binding.

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — ATSPIAdapter is fully implemented. On a system without the AT-SPI2 bus running, `IsAvailable` returns false and the adapter is silently skipped by CaptureManager (graceful degradation; tmux continues working alone).

## Self-Check: PASSED
