# Phase 4: Linux GUI Terminal Adapters (AT-SPI2) - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-29
**Phase:** 04-linux-gui-terminal-adapters-at-spi2
**Areas discussed:** CaptureManager architecture, Konsole spike strategy, AT-SPI2 onboarding flow, /filter command UX

---

## CaptureManager Architecture

| Option | Description | Selected |
|--------|-------------|----------|
| New services/capture/ package | CaptureManager interface, tmux and AT-SPI2 as adapters, TerminalService refactors into services/capture/tmux.go | ✓ |
| Extend TerminalService in-place | Add AT-SPI2 goroutine inside existing TerminalService | |
| Two parallel services, shared emitter | Keep TerminalService as-is, add AtSpiService as sibling | |

**User's choice:** New services/capture/ package (Recommended)
**Notes:** Clean separation, easy to add future adapters.

| Option | Description | Selected |
|--------|-------------|----------|
| CaptureManager owns both adapters | main.go creates one CaptureManager, calls manager.Startup(ctx) | ✓ |
| CaptureManager passed into app struct | Field on App struct alongside LLMService and CommandService | |
| You decide | Claude picks based on main.go fit | |

**User's choice:** CaptureManager owns both adapters (Recommended)
**Notes:** Single lifecycle point, consistent with current TerminalService wiring.

---

## Konsole Spike Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Time-boxed spike, then decide | One plan attempts AT-SPI2 on Konsole, documents findings, decides after | ✓ |
| Implement optimistically, mark experimental | Write adapter assuming success, degrade at runtime if it fails | |
| Skip Konsole entirely in Phase 4 | Defer to future phase | |

**User's choice:** Time-boxed spike, then decide (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Silent — windows just don't appear | No UI indication | |
| Tooltip/badge on the tab | ⚠ badge with tooltip: "Konsole text extraction not available" | ✓ |
| Log to console only | Warning in Go log output, no UI | |

**User's choice:** Tooltip/badge on the tab (Recommended)

---

## AT-SPI2 Onboarding Flow

| Option | Description | Selected |
|--------|-------------|----------|
| Extend the empty state | Reuse Phase 3 no-tmux empty state pattern, add AT-SPI2 path | ✓ |
| Banner at top of terminal sidebar | Persistent amber banner above tab list | |
| Startup modal dialog | Modal on app launch if AT-SPI2 disabled | |

**User's choice:** Extend the empty state (Recommended)
**Notes:** Consistent with Phase 3's $ tmux new-session pattern.

| Option | Description | Selected |
|--------|-------------|----------|
| Code block, same as tmux | Show gsettings command inline | ✓ |
| You decide | Claude picks presentation | |

**User's choice:** Code block, same as tmux (Recommended)

---

## /filter Command UX

| Option | Description | Selected |
|--------|-------------|----------|
| Chat message, like a bot reply | Output appears inline as system message in chat pane | ✓ |
| Modal or sidebar panel | Dedicated filter manager panel | |

**User's choice:** Chat message, like a bot reply (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — persist to config file | Saved to ~/.pairadmin/config.yaml via Viper | ✓ |
| Session-only | In-memory only, reset on restart | |

**User's choice:** Yes — persist to config file (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Go backend only | Custom patterns in services/llm/filter/ pipeline, no frontend store | ✓ |
| Frontend filterStore + backend sync | Zustand filterStore for display, sync to Go | |

**User's choice:** Go backend only (Recommended)

---

## Claude's Discretion

- Go interface design for TerminalAdapter
- CaptureManager adapter startup failure handling
- AT-SPI2 concurrency model (per-window goroutine vs semaphore pool)
- Config key naming for custom filter patterns

## Deferred Ideas

None.
