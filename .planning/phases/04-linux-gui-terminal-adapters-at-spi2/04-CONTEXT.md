# Phase 4: Linux GUI Terminal Adapters (AT-SPI2) - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Capture terminal content from GNOME Terminal and Konsole (via AT-SPI2 accessibility bus) for users not running tmux. Also adds user-configurable credential filter patterns via `/filter` slash commands.

New capabilities delivered: AT-SPI2 discovery and polling, CaptureManager multi-adapter architecture, AT-SPI2 onboarding flow, Konsole spike, `/filter add|list|remove` slash commands.

Out of scope: macOS/Windows adapters, Settings UI (Phase 5), full Konsole implementation if spike fails.

</domain>

<decisions>
## Implementation Decisions

### CaptureManager Architecture
- **D-01:** Create a new `services/capture/` package with a `CaptureManager` that owns both adapters. The existing `TerminalService` in `services/terminal.go` refactors into `services/capture/tmux.go`. Both tmux and AT-SPI2 become adapters implementing a common interface. This provides clean separation and makes future adapters (macOS, Windows) straightforward.
- **D-02:** `main.go` creates one `CaptureManager` and calls `manager.Startup(ctx)`. The manager starts both tmux and AT-SPI2 adapters internally. Single lifecycle point — consistent with how `TerminalService` is currently wired via `app.OnStartup`.
- **D-03:** Pane ID namespace: tmux panes use `tmux:%N` prefix (e.g., `tmux:%3`), AT-SPI2 panes use `atspi:/org/a11y/atspi/...` prefix. CaptureManager deduplicates by PaneID; combined list fed into existing `terminal:tabs` and `terminal:update` Wails events — no frontend changes to event handling.

### Konsole Spike Strategy
- **D-04:** Time-boxed spike approach. One dedicated plan attempts AT-SPI2 text extraction on Konsole and documents findings. Decision point after spike: if text extraction succeeds → implement full Konsole adapter. If it fails → mark experimental, skip full implementation.
- **D-05:** If Konsole spike fails at runtime (text extraction unavailable), detected Konsole windows appear as tabs in the sidebar with a ⚠ badge and tooltip: "Konsole text extraction not available on this system." Silent failure is not acceptable — user should understand why content isn't captured.

### AT-SPI2 Onboarding Flow
- **D-06:** Extend the existing empty state from Phase 3. When no tmux sessions AND AT-SPI2 is disabled or no GUI terminals detected: show "No terminal sessions detected." with two paths: (1) start tmux, (2) enable accessibility. The `$ gsettings set org.gnome.desktop.interface toolkit-accessibility true` command appears as a code block — same pattern as `$ tmux new-session` in Phase 3.
- **D-07:** After the user enables accessibility and relaunches (or if it's already enabled but no GUI terminals are open), the empty state returns to the standard no-tabs state. Polling continues — tabs appear automatically when GUI terminals open, same as tmux auto-detection.

### /filter Command UX
- **D-08:** `/filter list` output appears inline as a system message in the chat pane — formatted table/list of active custom patterns. Consistent with chat-centric slash command UX (Discord/Slack pattern). No new UI components needed.
- **D-09:** Custom filter patterns persist to `~/.pairadmin/config.yaml` (the Viper config file that Phase 5 will fully build out). Patterns survive app restarts. Config write happens immediately on `/filter add` and `/filter remove`.
- **D-10:** Custom filter state lives in the Go backend only. Patterns are loaded at startup into the existing `services/llm/filter/` pipeline. Frontend sends `/filter` commands to Go via Wails bindings. No frontend `filterStore` — filtering is a server-side pipeline concern that runs on every LLM call.
- **D-11:** `/filter add <name> <regex> <action>` — `action` values: `redact` (replace match with `[REDACTED]`) and `remove` (strip the entire line). Same actions as the built-in credential filter. Name is a user-friendly label for listing/removing.

### Claude's Discretion
- Go interface design for the `TerminalAdapter` (method names, signatures)
- How `CaptureManager` handles adapter startup failure (AT-SPI2 unavailable → degrade gracefully, keep tmux running)
- Whether AT-SPI2 polling uses a single goroutine per window or the same semaphore-bounded pattern as tmux
- Config key naming for custom filter patterns in `config.yaml`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Adapter (to be refactored)
- `services/terminal.go` — Current tmux TerminalService; refactors into services/capture/tmux.go in this phase
- `services/terminal_test.go` — Test patterns to replicate for the new capture package

### Filter Pipeline
- `services/llm/filter/filter.go` — Entry point; custom patterns hook in here
- `services/llm/filter/credential.go` — Existing built-in credential patterns; custom patterns follow same structure
- `services/llm/filter/ansi.go` — ANSI stripping (Stage 1); custom patterns are Stage 3

### Frontend Integration Points
- `frontend/src/hooks/useTerminalCapture.ts` — Subscribes to terminal:tabs and terminal:update events; no changes needed if events stay the same
- `frontend/src/stores/terminalStore.ts` — Tab lifecycle store; AT-SPI2 tabs use same addTab/removeTab flow
- `frontend/src/components/terminal/TerminalPreview.tsx` — Empty state pattern (D-06 extends this)

### Phase Context
- `.planning/phases/03-tmux-terminal-capture/03-CONTEXT.md` — Phase 3 decisions (pane naming D-01/D-02, empty state D-03/D-04, tab lifecycle D-05/D-06)
- `.planning/REQUIREMENTS.md` §ATSPI-01–04, FILT-04–05 — Acceptance criteria for this phase

### No external specs
- `godbus/dbus/v5` is already in go.mod as an indirect dep — no new library needed
- AT-SPI2 D-Bus API is documented at https://www.freedesktop.org/wiki/Accessibility/AT-SPI2/ (researcher should validate current method signatures)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `services/terminal.go` — Full tmux adapter to be refactored; injectable `execCommand` and `emitFn` test patterns should be preserved in the new package
- `services/llm/filter/filter.go` — Custom patterns plug into the existing pipeline; no new filter infrastructure needed
- `frontend/src/components/terminal/TerminalPreview.tsx` — Empty state JSX (the `if (!tabId)` block) is the base for the extended AT-SPI2 onboarding state
- `frontend/src/hooks/useTerminalCapture.ts` — No changes expected; works at the Wails event level, adapter-agnostic

### Established Patterns
- Injectable `execCommand`/`emitFn` vars for test isolation (from Phase 3) — replicate in AT-SPI2 adapter
- `golang.org/x/sync/semaphore` for bounded concurrency — already in go.mod
- Wails dynamic import pattern (`/* @vite-ignore */`) for runtime bindings — established in Phase 2/3

### Integration Points
- `main.go` `OnStartup` closure — where `CaptureManager.Startup(ctx)` is called (replacing current `TerminalService` startup)
- `app.go` — `App` struct; `CaptureManager` becomes a field alongside `llmService` and `commands`
- `frontend/src/components/layout/ThreeColumnLayout.tsx` — Where `useTerminalCapture` is mounted; AT-SPI2 tabs surface automatically here

</code_context>

<specifics>
## Specific Ideas

- **Konsole badge pattern** — The ⚠ badge on tabs when Konsole text extraction fails should use the same tab component as normal tabs; a simple `variant="degraded"` prop or conditional CSS class is sufficient. Tooltip text: "Konsole text extraction not available on this system."
- **Empty state layering** — Phase 3's empty state is a single `if (!tabId)` check in TerminalPreview. Phase 4 may need to pass an `adapterStatus` prop down so the empty state knows whether to show the AT-SPI2 onboarding path vs the plain no-tmux message.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 04-linux-gui-terminal-adapters-at-spi2*
*Context gathered: 2026-03-29*
