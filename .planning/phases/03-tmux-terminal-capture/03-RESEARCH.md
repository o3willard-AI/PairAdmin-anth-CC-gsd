# Phase 3: tmux Terminal Capture - Research

**Researched:** 2026-03-28
**Domain:** Go subprocess orchestration, tmux CLI API, Wails event bridge, xterm.js live write
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Tab names use `session:window.pane` format — e.g., `main:0.0`, `work:1.2`. Matches tmux conventions; familiar to tmux users; unique without extra logic.
- **D-02:** Pane ID (`%N` format, e.g., `%3`) is the stable internal key used for `CaptureManager` and deduplication — the display name is derived from it but the ID is what's stored.
- **D-03:** When no tmux session is detected at startup, show instruction text in the terminal preview pane: "No tmux session detected. Start a tmux session to begin." followed by the command `$ tmux new-session`. Tab sidebar shows no tabs.
- **D-04:** Polling continues during the no-tmux state. When the user starts tmux after launching PairAdmin, tabs appear automatically within 500ms — no app restart required.
- **D-05:** When a tmux pane closes, its PairAdmin tab is removed immediately from the sidebar. Chat history for that session is discarded (no persistence in v1 — SQLite deferred per PROJECT.md).
- **D-06:** If the closed pane was the active tab, auto-switch to the first remaining tab. If no tabs remain, show the no-tmux empty state (D-03).

### Claude's Discretion

- Go package structure for `TerminalAdapter`, `CaptureManager`, and polling service (suggested: `services/terminal/`)
- Wails event name for terminal content updates (e.g., `terminal:update`)
- Semaphore implementation for bounded concurrency (max 4 concurrent subprocesses)
- FNV64a hash implementation (standard library `hash/fnv`)
- How `TerminalService` integrates with `main.go` `OnStartup` closure (same pattern as `LLMService` and `CommandService`)
- Live scroll behavior: "always scroll to bottom" vs "hold position if scrolled up" (defer to xterm.js simplicity)

### Deferred Ideas (OUT OF SCOPE)

- AT-SPI2 adapter for non-tmux terminals — Phase 4 scope
- Persistent chat history per pane (SQLite) — deferred per PROJECT.md, post-v1
- Tab renaming by user — Phase 5 (Settings)
- Streaming abort/cancel during capture error — nice-to-have, post-Phase 3
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TMUX-01 | Application discovers all active tmux sessions and panes on startup via `tmux list-panes -a` | tmux subprocess API fully documented; format string pattern verified |
| TMUX-02 | Terminal content is captured from each pane via `tmux capture-pane -p` at 500ms polling interval | polling loop pattern + `time.NewTicker` in Go goroutine; golang.org/x/sync semaphore already in go.mod |
| TMUX-03 | New tmux sessions/panes are detected automatically without user action | re-run `list-panes -a` each tick and diff against known set; no persistent daemon needed |
| TMUX-04 | Closed tmux sessions are detected and corresponding tabs are marked inactive | diff on each tick; panes absent from current list emit removal events |
| TMUX-05 | FNV64a hash deduplication prevents sending unchanged content to the LLM pipeline | `hash/fnv` stdlib; FNV-64a is one function call per pane per tick |
| TMUX-06 | Each tmux pane maps to an isolated PairAdmin tab with independent chat history and context | terminalStore `addTab`/`removeTab`; chatStore already keyed by `tabId`; no new storage needed |
</phase_requirements>

---

## Summary

Phase 3 replaces all mock data with a real Go service that shells out to the `tmux` binary. The architecture is straightforward: a `TerminalService` in Go runs a 500ms ticker, calls `tmux list-panes -a` to discover panes, calls `tmux capture-pane -p -t %N` for each pane concurrently (bounded to 4 subprocesses via the `semaphore` package already in go.mod), hashes content with FNV-64a to skip unchanged panes, and emits Wails events to the frontend when content changes or pane membership changes.

The frontend side has minimal new work. `terminalStore` gains three actions (`addTab`, `removeTab`, `initEmpty`). `TerminalPreview` subscribes to `terminal:update` events and calls `term.write()` directly — matching the established xterm.js pattern from Phase 1. The tab lifecycle (appearance, disappearance, active-tab fallback) is driven by the event stream from Go. The chatStore is already keyed by `tabId`, so per-tab chat isolation (TMUX-06) works without changes to the chat layer.

tmux is NOT available on the development machine at the time of research. This is expected — the application targets a Linux deployment where tmux is installed. Tests must mock the subprocess boundary using a replaceable `exec.Command` variable (matching the `lookPath` pattern in `commands.go`) rather than requiring a live tmux instance.

**Primary recommendation:** Implement `TerminalService` in `services/terminal.go` (flat, not a sub-package) to match the `CommandService` / `LLMService` pattern. Use `os/exec` directly — no third-party tmux library. Use `golang.org/x/sync/semaphore` (already in go.mod) for bounded concurrency. Use `hash/fnv` stdlib for deduplication.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `os/exec` (stdlib) | Go 1.24 | Shell out to tmux binary | No IPC/socket exists; subprocess is the only tmux API |
| `hash/fnv` (stdlib) | Go 1.24 | FNV-64a content hashing for dedup | D.C. (CONTEXT.md) specifies `hash/fnv`; locked in |
| `golang.org/x/sync/semaphore` | v0.17.0 (in go.mod) | Bound concurrent tmux subprocesses to 4 | Already a transitive dep; no new module needed |
| `sync` (stdlib) | Go 1.24 | Mutex + WaitGroup for concurrent map writes | Standard Go concurrency primitives |
| `time` (stdlib) | Go 1.24 | `time.NewTicker(500ms)` for polling loop | No external scheduler needed |
| `context` (stdlib) | Go 1.24 | Cancellation of ticker goroutine on app exit | Consistent with Wails `OnStartup` ctx pattern |
| `github.com/wailsapp/wails/v2/pkg/runtime` | v2.12.0 | `EventsEmit` for Go→frontend push | All Go services use this; already in go.mod |

### Frontend

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `@xterm/xterm` | (installed) | `term.write()` for live content update | Phase 1 locked this; direct write, not React state |
| Zustand + Immer | (installed) | `addTab`, `removeTab` store mutations | Project-locked; all stores use this pattern |
| Wails runtime JS | (generated) | `EventsOn("terminal:update", ...)` | Same dynamic-import pattern as `useLLMStream.ts` |

### No New Dependencies Required

All required packages are already in `go.mod` (verified). No `go get` needed.

---

## Architecture Patterns

### Recommended Go Structure

```
services/
├── terminal.go          # TerminalService (Startup/Stop lifecycle, Wails bound methods)
├── terminal_test.go     # Unit tests with mocked execCommand var
├── commands.go          # Existing — no changes needed
├── llm_service.go       # Existing — no changes needed
└── llm/                 # Existing
```

The `TerminalAdapter` interface and `CaptureManager` live in `terminal.go` as unexported types. The only exported type is `TerminalService` — matching the `CommandService` shape.

### Pattern 1: TerminalService Lifecycle (matches existing services)

**What:** `TerminalService` has `NewTerminalService()`, `Startup(ctx)`, and an internal goroutine started in `Startup`.

**When to use:** All Wails-bound services follow this pattern.

```go
// Source: services/commands.go pattern + services/llm_service.go pattern
type TerminalService struct {
    ctx    context.Context
    cancel context.CancelFunc
}

func NewTerminalService() *TerminalService {
    return &TerminalService{}
}

func (t *TerminalService) Startup(ctx context.Context) {
    t.ctx, t.cancel = context.WithCancel(ctx)
    go t.pollLoop()
}
```

In `main.go`, add alongside existing services:
```go
terminal := services.NewTerminalService()
// in OnStartup:
terminal.Startup(ctx)
// in Bind:
terminal,
```

### Pattern 2: Subprocess Mockability (matches commands.go `lookPath` pattern)

**What:** Package-level `var execCommand = exec.CommandContext` so tests can inject a fake.

**When to use:** Any Go service that shells out needs this to be unit-testable without live tmux.

```go
// Source: services/commands.go — lookPath var pattern
var execCommand = exec.CommandContext

func listPanes(ctx context.Context) ([]PaneRef, error) {
    out, err := execCommand(ctx, "tmux", "list-panes", "-a",
        "-F", "#{pane_id}\t#{session_name}\t#{window_index}\t#{pane_index}",
    ).Output()
    // ...
}
```

In tests, override `execCommand` to return canned output without spawning tmux.

### Pattern 3: FNV-64a Deduplication

**What:** Hash each pane's captured string; skip emit if hash unchanged.

```go
// Source: stdlib hash/fnv documentation
import "hash/fnv"

func hashContent(s string) uint64 {
    h := fnv.New64a()
    h.Write([]byte(s))
    return h.Sum64()
}

// In CaptureManager — per-pane hash map protected by mutex
type captureState struct {
    lastHash uint64
}
```

### Pattern 4: Bounded Concurrent Capture

**What:** Semaphore from `golang.org/x/sync/semaphore` limits to 4 concurrent `tmux capture-pane` processes.

```go
// Source: RESEARCH-TERMINAL-CAPTURE.md — semaphore pattern
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(4)

for _, pane := range panes {
    wg.Add(1)
    go func(p PaneRef) {
        defer wg.Done()
        if err := sem.Acquire(ctx, 1); err != nil { return }
        defer sem.Release(1)
        content, err := capturePane(ctx, p.PaneID)
        // ...
    }(pane)
}
wg.Wait()
```

### Pattern 5: Wails Event Push to Frontend

**What:** `runtime.EventsEmit(ctx, "terminal:update", payload)` for content changes; `runtime.EventsEmit(ctx, "terminal:tabs", tabList)` for tab membership changes.

**When to use:** All Go→frontend communication in this project uses EventsEmit.

```go
// Source: services/llm_service.go EventsEmit pattern
type TerminalUpdateEvent struct {
    PaneID  string `json:"paneId"`
    Content string `json:"content"`
}

type TerminalTabsEvent struct {
    Tabs []TabInfo `json:"tabs"`
}

type TabInfo struct {
    ID   string `json:"id"`   // pane_id e.g. "%3"
    Name string `json:"name"` // "session:window.pane" e.g. "main:0.0"
}
```

### Pattern 6: Frontend Wails Event Subscription (matches useLLMStream.ts)

**What:** Dynamic import of Wails runtime, subscribe in `useEffect`, return unsubscribe.

```typescript
// Source: frontend/src/hooks/useLLMStream.ts — EventsOn pattern
useEffect(() => {
    let unsubUpdate: (() => void) | null = null;
    let unsubTabs: (() => void) | null = null;

    import(/* @vite-ignore */ "../../wailsjs/runtime/runtime").then((rt) => {
        unsubUpdate = rt.EventsOn("terminal:update", (event: TerminalUpdateEvent) => {
            const term = useTerminalStore.getState().getTermRef(event.paneId);
            if (term) {
                term.clear();
                term.write(event.content);
            }
        });
        unsubTabs = rt.EventsOn("terminal:tabs", (event: TerminalTabsEvent) => {
            // call store addTab/removeTab based on diff
        });
    });

    return () => { unsubUpdate?.(); unsubTabs?.(); };
}, []);
```

### Pattern 7: xterm.js Content Write

**What:** `term.clear()` then `term.write(content)` to replace display with latest capture.

**Important nuance:** `capture-pane` output without `-e` flag is plain text. xterm.js `write()` accepts raw strings. `clear()` + `write()` is simpler than diffing and more correct for "replace viewport" semantics.

**Scroll behavior (D-07, Claude's discretion):** Use `term.clear()` + `term.write()` — xterm.js auto-scrolls to bottom after write. This is the simpler path. No scroll position tracking needed in v1.

### Pattern 8: terminalStore Extension

The store needs three new actions. All mutations via Immer as established:

```typescript
// Extension to terminalStore.ts
addTab: (id: string, name: string) => void;
removeTab: (id: string) => void;
clearTabs: () => void;  // for no-tmux state
```

The `tabs` array shape (`{ id, name }`) is unchanged — just populated from real pane data instead of hardcoded mock values.

### Anti-Patterns to Avoid

- **Targeting panes by `session:window.pane` for capture:** Use stable pane ID (`%3`) with `tmux capture-pane -p -t %3`. Session:window.pane indices shift when windows are moved or renamed.
- **Capturing with `-e` flag for LLM context:** `-e` includes raw ANSI escape sequences, bloating content sent to the filter pipeline. Omit `-e` for plain text capture. The filter's ANSIFilter still runs as a safety net.
- **Storing xterm Terminal objects in Zustand state:** xterm objects are not serializable. The `termRefsMap` pattern (external Map, accessed via `setTermRef`/`getTermRef`) is established and must be maintained.
- **Spawning unlimited goroutines for pane capture:** Use the semaphore. A user with 20 tmux panes would otherwise fork 20 subprocesses simultaneously every 500ms.
- **Fatal error when tmux not running:** `tmux list-panes -a` exits with error when no server is running. This is expected (no-tmux state D-03). Treat as empty pane list, not crash.
- **Calling `term.write()` with unfiltered content:** Always run captured content through `filter.NewPipeline(filter.NewANSIFilter(), credFilter).Apply(content)` before writing to xterm or including in LLM context.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| tmux subprocess API | Custom IPC/socket | `exec.CommandContext` + tmux binary | tmux has no Go library with stable pane ID support; subprocess is the canonical API |
| Bounded goroutine pool | Custom worker pool with channels | `golang.org/x/sync/semaphore` (already in go.mod) | Semaphore is simpler, already available, correct for this use case |
| Content hashing | CRC32 or custom checksum | `hash/fnv.New64a()` (stdlib) | Locked decision from CONTEXT.md; one-liner; no collisions for terminal content sizes |
| Concurrent map writes | Hand-rolled lock-free map | `sync.Mutex` + regular map | Standard Go pattern; no performance concern at 4-pane poll rate |
| Frontend tab diff | Custom reconciliation algorithm | Compare new pane ID set to current `tabs` array | The diff is simple set subtraction; no library needed |

**Key insight:** The entire tmux integration is 4 CLI commands with simple string parsing. Custom solutions add complexity with no benefit.

---

## Common Pitfalls

### Pitfall 1: tmux "no server running" Exit Code Is Not an Error

**What goes wrong:** `exec.Command("tmux", "list-panes", "-a").Output()` returns an error when no tmux server is running. Treating this as a fatal error crashes the service or logs noise on every 500ms tick.

**Why it happens:** tmux exits non-zero when there is no server. The error message contains "no server running" or "error connecting to".

**How to avoid:** Check the error message string. Return an empty `[]PaneRef` for this specific error. All other errors should still propagate.

```go
if err != nil {
    if strings.Contains(err.Error(), "no server running") ||
       strings.Contains(err.Error(), "error connecting to") {
        return nil, nil  // no tmux — expected D-03 state
    }
    return nil, err
}
```

**Warning signs:** Steady stream of error logs every 500ms when user has no tmux running.

### Pitfall 2: Race Condition Between Tab Removal and Content Events

**What goes wrong:** A pane closes; Go detects it and emits `terminal:tabs` removal. But a `terminal:update` event for that pane was already queued. Frontend receives update for a tab that no longer exists in the store, causing a nil-ref on `getTermRef`.

**Why it happens:** Event ordering through Wails EventsEmit is not guaranteed to be synchronous with store mutations.

**How to avoid:** In the `terminal:update` handler, always guard with a `getTermRef` nil check. If term ref is null (tab was removed), silently discard the event.

```typescript
const term = useTerminalStore.getState().getTermRef(event.paneId);
if (!term) return;  // tab already removed — discard
```

**Warning signs:** Console errors about calling `.write()` on null/undefined.

### Pitfall 3: Pane Index Drift

**What goes wrong:** Using `session:window.pane` notation (e.g., `main:0.1`) as the capture target. When windows are moved or renamed, pane indices change. A capture for `main:0.1` silently hits a different pane.

**Why it happens:** Pane notation is positional, not stable. Only `#{pane_id}` (`%N`) is stable across session lifetime.

**How to avoid:** Use `%N` as both the internal key and the `-t` target for `tmux capture-pane`. Derive the display name (`session:window.pane`) during `list-panes` but never use it as a capture target.

**Warning signs:** Terminal preview shows wrong content after user reorganizes tmux windows.

### Pitfall 4: Large Scrollback Buffer Capture

**What goes wrong:** `tmux capture-pane -p -t %N -S -` (full scrollback) returns megabytes for long-running sessions. Emitting this via Wails EventsEmit and writing to xterm every 500ms causes UI jank and high memory pressure.

**Why it happens:** `-S -` means "beginning of scrollback" — potentially thousands of lines.

**How to avoid:** Omit `-S` flag entirely. Default behavior captures only the visible screen (typically 24-80 lines). If scrollback is needed later, use `-S -200` to cap at last 200 lines. The `filter` package's 200-line truncation for LLM context is a separate concern from what gets displayed.

**Warning signs:** High memory usage, slow UI after sessions accumulate history.

### Pitfall 5: xterm `term.write()` vs `term.writeln()` for Captured Content

**What goes wrong:** `term.writeln(content)` adds a trailing newline after the entire captured block, shifting the display. Or each line is written as a separate `writeln` call, doubling newlines because captured content already contains `\n`.

**Why it happens:** tmux capture output ends lines with `\n`. `writeln` adds another `\n`.

**How to avoid:** Use `term.clear()` then `term.write(content)` — write the full capture as one string. The captured content's newlines are preserved correctly.

### Pitfall 6: Frontend Tab ID Namespace Collision

**What goes wrong:** Phase 1 mock tabs use `"bash-1"`, `"bash-2"` as IDs. Phase 3 uses `"%3"`, `"%5"` etc. The `terminalStore.test.ts` hardcodes the Phase 1 IDs. If the store is initialized with mock data at module load time (not just in `beforeEach`), tests may see stale state.

**Why it happens:** The current `terminalStore.ts` initializes `tabs` with hardcoded mock data in the Zustand `create()` call. Phase 3 replaces this with an empty array (populated by real pane discovery).

**How to avoid:** Change the initial `tabs` state to `[]` and `activeTabId` to `""`. Update `terminalStore.test.ts` to use the new initial state. All mock tab tests become `addTab`/`removeTab` tests.

---

## Code Examples

### Discovery Command

```go
// Source: RESEARCH-TERMINAL-CAPTURE.md + tmux man page (verified)
// Emits one tab-delimited line per pane across all sessions
out, err := execCommand(ctx, "tmux", "list-panes", "-a",
    "-F", "#{pane_id}\t#{session_name}\t#{window_index}\t#{pane_index}",
).Output()
// Parse: fields[0]="%3", fields[1]="main", fields[2]="0", fields[3]="0"
// Display name: fields[1] + ":" + fields[2] + "." + fields[3] = "main:0.0"
```

### Capture by Stable Pane ID

```go
// Source: RESEARCH-TERMINAL-CAPTURE.md
// -p = print to stdout; no -e = plain text (no ANSI); no -S = visible screen only
out, err := execCommand(ctx, "tmux", "capture-pane", "-p", "-t", paneID).Output()
content := strings.TrimRight(string(out), "\n")
```

### FNV-64a Hash

```go
// Source: Go stdlib hash/fnv package
import "hash/fnv"

func hashContent(s string) uint64 {
    h := fnv.New64a()
    h.Write([]byte(s))
    return h.Sum64()
}
```

### 500ms Polling Loop

```go
// Source: Go stdlib time.NewTicker
func (t *TerminalService) pollLoop() {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()
    for {
        select {
        case <-t.ctx.Done():
            return
        case <-ticker.C:
            t.tick()
        }
    }
}
```

### Test Mock for execCommand

```go
// Source: services/commands.go lookPath var pattern
var execCommand = exec.CommandContext  // package-level var, overridable in tests

// In _test.go:
func TestListPanesNoTmux(t *testing.T) {
    orig := execCommand
    execCommand = func(ctx context.Context, name string, args ...string) *exec.Cmd {
        return exec.CommandContext(ctx, "false")  // exits non-zero, simulates no server
    }
    defer func() { execCommand = orig }()
    // ...
}
```

### Frontend: useTerminalCapture hook skeleton

```typescript
// Mirrors useLLMStream.ts pattern
export function useTerminalCapture() {
    useEffect(() => {
        let unsubUpdate: (() => void) | null = null;
        let unsubTabs: (() => void) | null = null;

        import(/* @vite-ignore */ "../../wailsjs/runtime/runtime").then((rt) => {
            unsubTabs = rt.EventsOn("terminal:tabs", handleTabsUpdate as (...args: unknown[]) => void);
            unsubUpdate = rt.EventsOn("terminal:update", handleContentUpdate as (...args: unknown[]) => void);
        });

        return () => { unsubUpdate?.(); unsubTabs?.(); };
    }, []);
}
```

---

## Runtime State Inventory

> This is not a rename/refactor phase. No runtime state migration is required.

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | None — no database in use (SQLite deferred) | None |
| Live service config | None | None |
| OS-registered state | None | None |
| Secrets/env vars | None new — existing PAIRADMIN_* env vars unchanged | None |
| Build artifacts | Mock tab data in `terminalStore.ts` initial state | Code edit — change initial `tabs: []` and `activeTabId: ""` |

The only "state migration" is replacing the hardcoded mock tab seed in `terminalStore.ts` with an empty initial state. This is a code edit, not a data migration.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| tmux binary | All TMUX-01–06 requirements | ✗ | — | Not applicable — app targets deployment env; test with mocked execCommand |
| `golang.org/x/sync` | Semaphore (TMUX-02) | ✓ | v0.17.0 (in go.mod) | — |
| `hash/fnv` | Dedup (TMUX-05) | ✓ | stdlib | — |
| Go 1.24 | All Go services | ✓ | 1.24 (go.mod) | — |
| vitest 4.x | Frontend tests | ✓ | v4.1.2 (verified by test run) | — |

**Missing dependencies with no fallback:**
- `tmux` binary — not present on development machine. This is expected. All Go unit tests MUST mock the subprocess boundary via the `execCommand` variable. Integration/smoke testing requires a tmux session.

**Missing dependencies with fallback:**
- None.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Go framework | `testing` stdlib + `go test ./services/...` |
| Frontend framework | vitest v4.1.2 |
| Go config file | none (stdlib testing) |
| Frontend config | `frontend/vite.config.ts` (test section) |
| Go quick run | `go test ./services/... -run TestTerminal` |
| Go full suite | `go test ./services/... ./services/llm/...` |
| Frontend quick run | `cd frontend && npx vitest run src/stores/__tests__/terminalStore.test.ts` |
| Frontend full suite | `cd frontend && npx vitest run` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| TMUX-01 | `listPanes` parses tmux output into `[]PaneRef` correctly | unit | `go test ./services/... -run TestListPanes` | ❌ Wave 0 |
| TMUX-01 | `listPanes` returns empty slice when tmux not running (no-server exit) | unit | `go test ./services/... -run TestListPanesNoTmux` | ❌ Wave 0 |
| TMUX-02 | `capturePane` calls `tmux capture-pane -p -t <paneID>` with correct args | unit | `go test ./services/... -run TestCapturePane` | ❌ Wave 0 |
| TMUX-03 | New pane in discovery result triggers `terminal:tabs` add event | unit | `go test ./services/... -run TestPollNewPane` | ❌ Wave 0 |
| TMUX-04 | Pane absent from discovery result triggers `terminal:tabs` remove event | unit | `go test ./services/... -run TestPollRemovedPane` | ❌ Wave 0 |
| TMUX-05 | FNV-64a hash matches for identical content — event NOT emitted | unit | `go test ./services/... -run TestDedup` | ❌ Wave 0 |
| TMUX-05 | FNV-64a hash differs for changed content — event IS emitted | unit | `go test ./services/... -run TestDedupChanged` | ❌ Wave 0 |
| TMUX-06 | `addTab` / `removeTab` store actions maintain correct tab list | unit | `cd frontend && npx vitest run src/stores/__tests__/terminalStore.test.ts` | ❌ Wave 0 (update existing file) |
| TMUX-06 | Active tab auto-switches to first remaining tab after removal | unit | `cd frontend && npx vitest run src/stores/__tests__/terminalStore.test.ts` | ❌ Wave 0 (update existing file) |
| TMUX-06 | No-tmux empty state shown when tabs list is empty | unit | `cd frontend && npx vitest run src/components/__tests__/` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `go test ./services/... && cd frontend && npx vitest run`
- **Per wave merge:** `go test ./services/... ./services/llm/... && cd frontend && npx vitest run`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `services/terminal_test.go` — covers TMUX-01 through TMUX-05 (Go unit tests)
- [ ] `frontend/src/stores/__tests__/terminalStore.test.ts` — update existing file to test `addTab`, `removeTab`, empty-state behavior (TMUX-06)
- [ ] `frontend/src/components/__tests__/TerminalPreview.test.tsx` — no-tmux empty state rendering (TMUX-03/TMUX-06)

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mock tab data hardcoded in store | Empty initial state, populated by Go service | Phase 3 | All tab tests must reset to empty state in `beforeEach` |
| Mock `term.writeln(...)` in TerminalPreview | `term.clear()` + `term.write(content)` on event | Phase 3 | Remove mock writeln block; add EventsOn subscription |

**Deprecated/outdated after Phase 3:**
- Hardcoded `tabs: [{ id: "bash-1" }, { id: "bash-2" }]` in `terminalStore.ts` initial state
- Mock `term.writeln(...)` content block in `TerminalPreview.tsx` (lines 50–60)
- `[No terminal connected — Phase 1 mock]` warning line in TerminalPreview

---

## Open Questions

1. **`terminal:update` payload: full content or diff?**
   - What we know: `capture-pane` always returns the full visible screen (not a diff). Full content is simpler.
   - What's unclear: For very wide/tall terminals, content strings could be 10-20KB. EventsEmit payload size limits in Wails v2 are undocumented.
   - Recommendation: Use full content (simpler, correct). If payload size becomes an issue post-Phase 3, switch to line-based diff. Do not pre-optimize.

2. **Wails EventsEmit thread safety**
   - What we know: `runtime.EventsEmit` is called from Go goroutines in `LLMService` (established in Phase 2). This works correctly.
   - What's unclear: Whether calling EventsEmit from multiple goroutines simultaneously (one per pane) is safe.
   - Recommendation: Emit from the poll loop's main goroutine after collecting all results (after `wg.Wait()`), not from the per-pane capture goroutines. This avoids any thread-safety question and is simpler.

3. **`useTerminalCapture` hook placement**
   - What we know: `useLLMStream` is a hook used in ChatPane. Terminal capture events affect the layout-level (tab list changes) and per-tab (content updates).
   - What's unclear: Whether to put capture subscription in a top-level layout component or create a dedicated hook.
   - Recommendation: Create `useTerminalCapture` hook, mount it once in the top-level layout component (same level as `useLLMStream`). Tab list updates affect the sidebar; content updates affect individual TerminalPreview instances.

---

## Sources

### Primary (HIGH confidence)

- Prior research `RESEARCH-TERMINAL-CAPTURE.md` in this repo — tmux subprocess API, list-panes format variables, pane ID stability, semaphore pattern, capture flags (verified against tmux man page at research time)
- Go stdlib `hash/fnv` — standard library, no verification needed
- Go stdlib `os/exec` — standard library, no verification needed
- `go.mod` (verified by Read tool) — confirms `golang.org/x/sync v0.17.0` already present, no new deps needed
- Existing source files (verified by Read tool): `services/commands.go`, `services/llm_service.go`, `main.go`, `frontend/src/hooks/useLLMStream.ts`, `frontend/src/stores/terminalStore.ts`, `frontend/src/components/terminal/TerminalPreview.tsx`
- Go test suite (verified by Bash run): `go test ./services/...` passes — 3 packages, 0.006-0.009s

### Secondary (MEDIUM confidence)

- CONTEXT.md locked decisions (D-01 through D-07) — authoritative for this project; no external verification needed
- vitest run output (verified): 9 test files, 50 tests passing — frontend test infrastructure confirmed working

### Tertiary (LOW confidence)

- Wails EventsEmit thread safety from multiple goroutines — not explicitly documented in Wails v2 docs; mitigated by architectural recommendation to emit from main poll goroutine only

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all packages verified in go.mod; no new deps
- Architecture patterns: HIGH — based on verified existing code patterns in this repo
- tmux subprocess API: HIGH — from prior research verified against tmux man page
- FNV-64a dedup: HIGH — stdlib, one function call
- Pitfalls: HIGH — derived from tmux behavior and existing code patterns
- Wails EventsEmit thread safety: MEDIUM — mitigated by design recommendation

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable technology; tmux and Go stdlib do not change rapidly)
