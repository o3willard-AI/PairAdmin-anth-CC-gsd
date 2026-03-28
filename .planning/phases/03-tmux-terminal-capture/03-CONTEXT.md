# Phase 3: tmux Terminal Capture - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace mock terminal tabs and mock content with real tmux-discovered panes. Implement a Go `TerminalAdapter` interface + tmux adapter, 500ms polling via `CaptureManager`, FNV64a hash deduplication, and pane lifecycle management. Each tmux pane maps to an isolated PairAdmin tab with independent chat and command history. The terminal preview pane shows live tmux output via xterm.js direct write. Every chat message is prefixed with real filtered terminal context.

Exit criteria: With tmux running, PairAdmin auto-discovers all sessions/panes, shows live terminal content in xterm.js, and AI responses reference real terminal output.

</domain>

<decisions>
## Implementation Decisions

### Pane Naming
- **D-01:** Tab names use `session:window.pane` format — e.g., `main:0.0`, `work:1.2`. Matches tmux conventions; familiar to tmux users; unique without extra logic.
- **D-02:** Pane ID (`%N` format, e.g., `%3`) is the stable internal key used for `CaptureManager` and deduplication — the display name is derived from it but the ID is what's stored.

### No-tmux Empty State
- **D-03:** When no tmux session is detected at startup, show instruction text in the terminal preview pane: "No tmux session detected. Start a tmux session to begin." followed by the command `$ tmux new-session`. Tab sidebar shows no tabs.
- **D-04:** Polling continues during the no-tmux state. When the user starts tmux after launching PairAdmin, tabs appear automatically within 500ms — no app restart required.

### Closed Pane Lifecycle
- **D-05:** When a tmux pane closes, its PairAdmin tab is removed immediately from the sidebar. Chat history for that session is discarded (no persistence in v1 — SQLite deferred per PROJECT.md).
- **D-06:** If the closed pane was the active tab, auto-switch to the first remaining tab. If no tabs remain, show the no-tmux empty state (D-03).

### Live Scroll Behavior
- **D-07:** Claude's discretion — defer to what's simpler/more correct given xterm.js capabilities. Both "always scroll to bottom" and "hold position if scrolled up" are acceptable; planner should choose based on xterm.js Terminal API feasibility.

### Claude's Discretion
- Go package structure for `TerminalAdapter`, `CaptureManager`, and polling service (suggested: `services/terminal/`)
- Wails event name for terminal content updates (e.g., `terminal:update`)
- Semaphore implementation for bounded concurrency (max 4 concurrent subprocesses)
- FNV64a hash implementation (standard library `hash/fnv`)
- How `TerminalService` integrates with `main.go` `OnStartup` closure (same pattern as `LLMService` and `CommandService`)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Requirements
- `.planning/REQUIREMENTS.md` §TMUX-01–06 — the six requirements this phase must satisfy

### Prior Phase Patterns
- `.planning/phases/01-application-shell-ui-foundation/01-CONTEXT.md` — xterm.js integration pattern, termRefsMap, TerminalPreview lifecycle
- `.planning/phases/02-llm-gateway-streaming-chat/02-CONTEXT.md` — filter pipeline, terminal context assembly (last 200 lines), LLMService pattern

### Codebase Integration Points
- `frontend/src/stores/terminalStore.ts` — `tabs`, `activeTabId`, `setTermRef`/`getTermRef`; Phase 3 must extend with `addTab`, `removeTab`, `setTabContent` or equivalent
- `frontend/src/components/terminal/TerminalPreview.tsx` — xterm.js instantiation; needs `terminal:update` Wails event listener wired in
- `frontend/src/components/terminal/TerminalTabList.tsx` — reads `terminalStore.tabs`; existing rendering works once store is populated with real panes
- `services/commands.go` — pattern reference for Go service lifecycle (`Startup(ctx)`)
- `main.go` — `Bind[]` + `OnStartup` closure pattern for registering new `TerminalService`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `useTerminalStore` — `tabs: TerminalTab[]`, `activeTabId`, `setTermRef`/`getTermRef` exist; needs `addTab(id, name)`, `removeTab(id)` actions added
- `TerminalPreview.tsx` — xterm.js lifecycle fully set up; needs `runtime.EventsOn("terminal:update", ...)` wired in to receive real content chunks
- `TerminalTabList.tsx` — renders tabs from `terminalStore.tabs` with no changes needed if store shape is maintained
- `services/llm_service.go` — `Startup(ctx)` lifecycle + Wails `runtime.EventsEmit` pattern; replicate for `TerminalService`
- `filter` package — `filter.NewPipeline(filter.NewANSIFilter(), credFilter).Apply(content)` ready to use on captured pane content before sending to LLM

### Established Patterns
- xterm.js direct `.write()` calls (not React state) — established Phase 1, maintained here
- Go services in `services/` package; bound via `Bind[]` in `main.go`; `OnStartup` closure calls `.Startup(ctx)`
- Wails `runtime.EventsEmit(ctx, eventName, payload)` for Go→frontend; `runtime.EventsOn(name, cb)` on frontend
- Zustand + Immer for all frontend state mutations

### Integration Points
- `TerminalPreview.tsx` mock `term.writeln(...)` lines are replaced by real `terminal:update` event subscription
- `terminalStore.tabs` mock seed data replaced by real tabs populated from `TerminalService.Discover()`
- Chat context assembly in `llm_service.go` `SendMessage` already reads from `termRefsMap` via xterm buffer — this continues to work as-is once real content is written to xterm

</code_context>

<specifics>
## Specific Ideas

- Pane IDs use `%N` format internally as the stable key — display name derived as `session:window.pane` using `tmux list-panes -a -F "#{pane_id} #{session_name}:#{window_index}.#{pane_index}"`
- CaptureManager semaphore: max 4 concurrent `tmux capture-pane` subprocesses to avoid thundering-herd on large tmux configs
- FNV64a dedup: only emit `terminal:update` event when hash changes — reduces xterm.js churn and avoids redundant LLM context updates
- `tmux capture-pane -p -t %3` (pane ID target, not session:win.pane) — stable across renames

</specifics>

<deferred>
## Deferred Ideas

- AT-SPI2 adapter for non-tmux terminals — Phase 4 scope
- Persistent chat history per pane (SQLite) — deferred per PROJECT.md, post-v1
- Tab renaming by user — Phase 5 (Settings)
- Streaming abort/cancel during capture error — nice-to-have, post-Phase 3

</deferred>

---

*Phase: 03-tmux-terminal-capture*
*Context gathered: 2026-03-28*
