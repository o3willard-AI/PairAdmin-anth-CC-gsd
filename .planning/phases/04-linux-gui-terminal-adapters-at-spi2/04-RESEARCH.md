# Phase 4: Linux GUI Terminal Adapters (AT-SPI2) - Research

**Researched:** 2026-03-28
**Domain:** AT-SPI2 D-Bus accessibility, Go adapter architecture, Viper config persistence, slash command routing
**Confidence:** MEDIUM-HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**CaptureManager Architecture**
- **D-01:** Create a new `services/capture/` package with a `CaptureManager` that owns both adapters. The existing `TerminalService` in `services/terminal.go` refactors into `services/capture/tmux.go`. Both tmux and AT-SPI2 become adapters implementing a common interface.
- **D-02:** `main.go` creates one `CaptureManager` and calls `manager.Startup(ctx)`. The manager starts both tmux and AT-SPI2 adapters internally. Single lifecycle point — consistent with how `TerminalService` is currently wired via `app.OnStartup`.
- **D-03:** Pane ID namespace: tmux panes use `tmux:%N` prefix (e.g., `tmux:%3`), AT-SPI2 panes use `atspi:/org/a11y/atspi/...` prefix. CaptureManager deduplicates by PaneID; combined list fed into existing `terminal:tabs` and `terminal:update` Wails events — no frontend changes to event handling.

**Konsole Spike Strategy**
- **D-04:** Time-boxed spike approach. One dedicated plan attempts AT-SPI2 text extraction on Konsole and documents findings. Decision point after spike: if text extraction succeeds → implement full Konsole adapter. If it fails → mark experimental, skip full implementation.
- **D-05:** If Konsole spike fails at runtime, detected Konsole windows appear as tabs with a ⚠ badge and tooltip: "Konsole text extraction not available on this system."

**AT-SPI2 Onboarding Flow**
- **D-06:** Extend the existing empty state from Phase 3. When no tmux sessions AND AT-SPI2 is disabled or no GUI terminals detected: show "No terminal sessions detected." with two paths: (1) start tmux, (2) enable accessibility.
- **D-07:** After the user enables accessibility and relaunches (or if it's already enabled but no GUI terminals are open), the empty state returns to the standard no-tabs state. Polling continues — tabs appear automatically when GUI terminals open.

**/filter Command UX**
- **D-08:** `/filter list` output appears inline as a system message in the chat pane — formatted table/list. No new UI components.
- **D-09:** Custom filter patterns persist to `~/.pairadmin/config.yaml` (the Viper config file). Patterns survive app restarts. Config write happens immediately on `/filter add` and `/filter remove`.
- **D-10:** Custom filter state lives in Go backend only. Patterns are loaded at startup into `services/llm/filter/` pipeline. Frontend sends `/filter` commands to Go via Wails bindings. No frontend `filterStore`.
- **D-11:** `/filter add <name> <regex> <action>` — `action` values: `redact` and `remove`. Name is a user-friendly label.

### Claude's Discretion
- Go interface design for the `TerminalAdapter` (method names, signatures)
- How `CaptureManager` handles adapter startup failure (AT-SPI2 unavailable → degrade gracefully, keep tmux running)
- Whether AT-SPI2 polling uses a single goroutine per window or the same semaphore-bounded pattern as tmux
- Config key naming for custom filter patterns in `config.yaml`

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ATSPI-01 | Application detects whether AT-SPI2 accessibility is enabled at startup; guides user to enable if not | AT-SPI2 `org.a11y.Status.IsEnabled` D-Bus property; GSettings check; onboarding flow in TerminalPreview |
| ATSPI-02 | GNOME Terminal windows discovered via AT-SPI2 bus (`ATSPI_ROLE_TERMINAL` objects) | `GetItems` on accessibility bus; role=59 filter; godbus/dbus/v5 already in go.mod |
| ATSPI-03 | Visible content read from GNOME Terminal via `org.a11y.atspi.Text.GetText(0, -1)` at 500ms polling | Text interface pattern; polling architecture mirrors existing tmux 500ms ticker |
| ATSPI-04 | Konsole attempted via AT-SPI2; degrades gracefully if text extraction fails (experimental) | Spike plan; Qt5 old Cache signature; ⚠ badge variant on tab component |
| FILT-04 | User can add custom filter patterns via `/filter add <name> <regex> <action>` | Wails binding route; `CustomFilter` struct; Viper YAML persistence |
| FILT-05 | User can list and remove custom filter patterns via `/filter list` and `/filter remove <name>` | Inline system message pattern (mirrors `/clear`); Viper write-on-change |
</phase_requirements>

---

## Summary

Phase 4 adds two capabilities: AT-SPI2-based terminal capture for GUI terminals (GNOME Terminal, Konsole spike), and user-configurable filter patterns via `/filter` slash commands. Both capabilities share a new `services/capture/` package that refactors the existing `TerminalService` into a multi-adapter `CaptureManager`.

The AT-SPI2 work is technically the most novel. The accessibility bus is already running on this machine (verified: `unix:path=/run/user/1000/at-spi/bus`), `gnome-terminal-server` is running (PID 133059), and `at-spi2-core` 2.52.0 is installed. However AT-SPI2 is currently disabled (`IsEnabled=false`) and `libatspi2.0-0` is listed as `un` (not installed). `godbus/dbus/v5 v5.1.0` is already an indirect dependency — it just needs to be promoted to direct use.

The `/filter` slash commands require Viper for persistence to `~/.pairadmin/config.yaml`. Viper is not yet in `go.mod` (currently config is env-var-only). This is the one new library dependency in this phase.

**Primary recommendation:** Structure the phase as four plans — (1) CaptureManager + tmux adapter refactor, (2) AT-SPI2 adapter for GNOME Terminal, (3) Konsole spike, (4) `/filter` slash commands + Viper config. The Konsole spike plan has a conditional exit: full Konsole adapter if text extraction succeeds, ⚠ badge degradation if it fails.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `github.com/godbus/dbus/v5` | v5.1.0 (already in go.mod) | AT-SPI2 D-Bus calls without CGO | Only viable pure-Go D-Bus library; no CGO required; already present as indirect dep |
| `github.com/spf13/viper` | v1.21.0 (latest stable) | Config file persistence (`~/.pairadmin/config.yaml`) | Project requirement (CFG-08); already using `gopkg.in/yaml.v3` which Viper uses internally |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `gopkg.in/yaml.v3` | v3.0.1 (already in go.mod) | YAML marshaling for config | Used internally by Viper; also usable directly for config struct if Viper adds too much weight |
| `golang.org/x/sync/semaphore` | v0.17.0 (already in go.mod) | Bounded concurrent AT-SPI2 calls | Same pattern as existing tmux adapter (max 4 concurrent captures) |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Viper | Hand-rolled YAML with `gopkg.in/yaml.v3` | Phase 5 will fully build out settings; Viper's watch, env-override, and defaults make Phase 5 simpler; hand-rolled is viable but creates rework |
| godbus/dbus/v5 | CGO + libatspi-2.0 | CGO requires C headers, complicates cross-compilation, adds runtime dep; godbus does the same work in pure Go |

**Installation:**
```bash
go get github.com/spf13/viper@v1.21.0
# godbus/dbus/v5 already in go.mod — promote from indirect to direct:
go get github.com/godbus/dbus/v5
```

**Version verification (confirmed 2026-03-28):**
- `godbus/dbus/v5`: v5.1.0 — verified in go.mod/go.sum
- `viper`: v1.21.0 — verified via `go list -m -versions github.com/spf13/viper` (latest stable)

---

## Architecture Patterns

### Recommended Package Structure
```
services/
├── capture/
│   ├── manager.go          # CaptureManager: lifecycle, poll loop, event emit
│   ├── manager_test.go
│   ├── adapter.go          # TerminalAdapter interface + PaneInfo/CaptureResult types
│   ├── tmux.go             # TmuxAdapter (refactored from services/terminal.go)
│   ├── tmux_test.go        # Tests carried over from services/terminal_test.go
│   ├── atspi.go            # ATSPIAdapter: bus connection, discovery, text capture
│   └── atspi_test.go       # Tests with mock dbus connection
├── config/
│   ├── config.go           # AppConfig struct, LoadConfig, SaveConfig (Viper-backed)
│   └── config_test.go
├── llm/
│   └── filter/
│       ├── filter.go       # (existing — unchanged)
│       ├── credential.go   # (existing — unchanged)
│       ├── ansi.go         # (existing — unchanged)
│       ├── custom.go       # NEW: CustomFilter (loads from AppConfig.CustomPatterns)
│       └── custom_test.go
└── llm_service.go          # (extended: FilterCommand Wails binding for /filter)
```

### Pattern 1: TerminalAdapter Interface

**What:** Common interface all capture backends implement.
**When to use:** Every adapter (tmux, atspi, future macOS/Windows) implements this and plugs into CaptureManager.

```go
// Source: .planning/research/RESEARCH-TERMINAL-CAPTURE.md §6 + project context

// PaneInfo describes a discoverable terminal pane with adapter-scoped ID.
type PaneInfo struct {
    ID          string            // namespaced: "tmux:%3" or "atspi:/org/a11y/atspi/..."
    AdapterType string            // "tmux" | "atspi"
    DisplayName string            // human-readable label for tab sidebar
    Degraded    bool              // true when discovery succeeded but capture will fail (Konsole case)
    DegradedMsg string            // tooltip text when Degraded=true
}

// TerminalAdapter is the interface each backend implements.
type TerminalAdapter interface {
    Name() string
    IsAvailable(ctx context.Context) bool
    Discover(ctx context.Context) ([]PaneInfo, error)
    Capture(ctx context.Context, pane PaneInfo) (string, error)
    Close() error
}
```

**Key design choice (Claude's discretion):** `Capture` returns `(string, error)` — the CaptureManager handles hashing and dedup, not the adapter. This keeps adapters thin and testable without needing the full `CaptureResult` struct.

### Pattern 2: CaptureManager Lifecycle

**What:** Single `Startup(ctx)` entry point, owns both adapters, emits existing `terminal:tabs` and `terminal:update` Wails events.
**When to use:** Replaces direct `TerminalService` in `main.go`.

```go
// main.go change (D-02):
// BEFORE: terminal := services.NewTerminalService()  → terminal.Startup(ctx)
// AFTER:  manager := capture.NewCaptureManager(emitFn)  → manager.Startup(ctx)
```

CaptureManager poll loop (same 500ms ticker):
1. Call `Discover` on each available adapter
2. Merge pane lists; detect membership changes → emit `terminal:tabs` if changed
3. Capture all panes concurrently (semaphore bound 4)
4. FNV-64a hash dedup → emit `terminal:update` only on change

**Degraded pane handling:** If `pane.Degraded == true`, skip Capture entirely and include pane in tabs with degraded metadata. Frontend renders ⚠ badge.

### Pattern 3: AT-SPI2 Connection Flow

**What:** Two-step D-Bus connection to accessibility bus.
**When to use:** ATSPIAdapter.IsAvailable and ATSPIAdapter startup.

```go
// Source: .planning/research/RESEARCH-TERMINAL-CAPTURE.md §2

func connectToA11yBus() (*dbus.Conn, error) {
    sessionConn, err := dbus.SessionBus()
    if err != nil {
        return nil, fmt.Errorf("session bus unavailable: %w", err)
    }
    var a11yAddr string
    obj := sessionConn.Object("org.a11y.Bus", "/org/a11y/bus")
    if err := obj.Call("org.a11y.Bus.GetAddress", 0).Store(&a11yAddr); err != nil {
        return nil, fmt.Errorf("a11y bus not running: %w", err)
    }
    a11yConn, err := dbus.Dial(a11yAddr)
    if err != nil {
        return nil, fmt.Errorf("dial a11y bus: %w", err)
    }
    if err := a11yConn.Auth(nil); err != nil {
        a11yConn.Close()
        return nil, err
    }
    if err := a11yConn.Hello(); err != nil {
        a11yConn.Close()
        return nil, err
    }
    return a11yConn, nil
}
```

Runtime verified: `unix:path=/run/user/1000/at-spi/bus` is the address on this machine. `GetAddress` returns it correctly.

### Pattern 4: AT-SPI2 Accessibility Status Check

**What:** Runtime check (D-Bus) vs. GSettings check serve different purposes.
**When to use:** `IsAvailable` uses both; onboarding flow triggers on GSettings=false.

```go
// Source: .planning/research/RESEARCH-TERMINAL-CAPTURE.md §5

// IsEnabled via D-Bus reflects runtime state (bus may run even if GSettings=false)
func isA11yRuntimeEnabled() (bool, error) {
    conn, err := dbus.SessionBus()
    if err != nil { return false, err }
    obj := conn.Object("org.a11y.Bus", "/org/a11y/bus")
    var enabled bool
    err = obj.Call("org.freedesktop.DBus.Properties.Get", 0,
        "org.a11y.Status", "IsEnabled").Store(&enabled)
    return enabled, err
}

// GSettings check: is toolkit-accessibility configured for autostart?
func isToolkitAccessibilityGsettings(ctx context.Context) bool {
    out, err := exec.CommandContext(ctx, "gsettings", "get",
        "org.gnome.desktop.interface", "toolkit-accessibility").Output()
    if err != nil { return false }
    return strings.TrimSpace(string(out)) == "true"
}
```

**Verified on this machine:** `IsEnabled` returns `false` (GSettings not set, but bus is running). `GetAddress` succeeds regardless. GSettings key controls whether apps attach to bus on startup — key for GNOME Terminal GTK3 accessibility.

### Pattern 5: Terminal Object Discovery and Text Capture

**What:** Enumerate accessibility bus, filter by `ATSPI_ROLE_TERMINAL` (role=59), call `Text.GetText`.
**When to use:** ATSPIAdapter.Discover and Capture.

```go
// Source: .planning/research/RESEARCH-TERMINAL-CAPTURE.md §2-§3

const RoleTerminal = uint32(59)

// CacheItem for new GTK4 signature: a((so)(so)(so)iiassusau)
type CacheItem struct {
    Ref           ObjectRef
    AppRef        ObjectRef
    ParentRef     ObjectRef
    IndexInParent int32
    ChildCount    int32
    Interfaces    []string
    Name          string
    Role          uint32
    Description   string
    StateSet      []uint32
}

type ObjectRef struct {
    Name string
    Path dbus.ObjectPath
}

// GetText: call on terminal object with start=0, end=-1 for all text
func getTerminalText(conn *dbus.Conn, ref ObjectRef) (string, error) {
    obj := conn.Object(ref.Name, ref.Path)
    var text string
    err := obj.Call("org.a11y.atspi.Text.GetText", 0, int32(0), int32(-1)).Store(&text)
    return text, err
}
```

**Critical caveat:** GTK4 `Cache.GetItems` uses `a((so)(so)(so)iiassusau)`; Qt5/Konsole uses old signature `a((so)(so)(so)a(so)assusau)`. Must try-and-fallback. See Anti-Patterns.

### Pattern 6: /filter Slash Command Routing

**What:** Frontend routes `/filter ...` to Go backend via Wails binding; backend returns response string for inline display.
**When to use:** ChatPane's `handleSend` already does `/clear` routing — `/filter` follows same pattern.

Existing `/clear` routing in `ChatPane.tsx`:
```typescript
if (text.trim() === "/clear") {
  useChatStore.getState().clearTab(activeTabId);
  return;
}
```

`/filter` needs a Go Wails binding call:
```typescript
// Frontend: ChatPane.tsx addition
if (text.trim().startsWith("/filter")) {
  import(/* @vite-ignore */ "../../wailsjs/go/services/LLMService")
    .then(({ FilterCommand }) => FilterCommand(text.trim()))
    .then((response: string) => {
      useChatStore.getState().addSystemMessage(activeTabId, response);
    });
  return;
}
```

Backend (`LLMService` or a dedicated `FilterService`):
```go
// FilterCommand handles /filter add|list|remove commands.
// Returns a human-readable string to display as a system message.
func (s *LLMService) FilterCommand(command string) (string, error) {
    // parse: /filter list | /filter add <name> <regex> <action> | /filter remove <name>
    // delegate to custom filter store
    // persist to Viper config
    // return formatted response
}
```

**Note:** `chatStore` currently has `addUserMessage` and `addAssistantMessage`. A `addSystemMessage` action needs to be added for inline filter output (consistent with D-08).

### Pattern 7: Viper Config Persistence

**What:** `~/.pairadmin/config.yaml` with Viper; custom patterns stored as a YAML sequence.
**When to use:** FilterCommand handler reads/writes custom patterns; loaded at startup into filter pipeline.

```go
// services/config/config.go

type CustomPattern struct {
    Name   string `mapstructure:"name" yaml:"name"`
    Regex  string `mapstructure:"regex" yaml:"regex"`
    Action string `mapstructure:"action" yaml:"action"` // "redact" | "remove"
}

type AppConfig struct {
    CustomPatterns []CustomPattern `mapstructure:"custom_patterns" yaml:"custom_patterns"`
}

func LoadAppConfig() (*AppConfig, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("$HOME/.pairadmin")
    viper.SetDefault("custom_patterns", []CustomPattern{})
    _ = viper.ReadInConfig() // missing config file is OK
    var cfg AppConfig
    return &cfg, viper.Unmarshal(&cfg)
}

func SaveAppConfig(cfg *AppConfig) error {
    viper.Set("custom_patterns", cfg.CustomPatterns)
    return viper.WriteConfig()
}
```

### Anti-Patterns to Avoid

- **Checking `org.a11y.Status.IsEnabled=false` and aborting:** The bus address is accessible even when IsEnabled=false (verified). IsEnabled=false means no apps are currently attached, not that the bus is broken. Only abort if `GetAddress` fails or returns empty.
- **Using well-known D-Bus names to find AT-SPI apps:** AT-SPI apps register as unique bus names (`:1.X` format). Filter names starting with `:` — do NOT skip them. Well-known names (e.g., `org.a11y.Bus` itself) should be skipped.
- **Assuming GNOME Terminal uses GTK4 Cache signature:** GNOME Terminal on Ubuntu 24.04 with GNOME 46+ may use GTK4 or GTK3 depending on version. Always try new signature first, fall back to old.
- **One goroutine per AT-SPI window with no bound:** Mirrors the tmux mistake avoided in Phase 3. Use same `semaphore.NewWeighted(4)` pattern.
- **Persisting filter patterns to `services/llm/filter/` package state:** Patterns belong in the Viper config layer, loaded into the filter package at startup. The filter package must not import Viper.
- **Calling `dbus.SessionBus()` on every poll tick:** Session bus connection is persistent. Establish once at ATSPIAdapter startup, reconnect only on error.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| D-Bus protocol communication | Custom socket/protocol parser | `godbus/dbus/v5` | D-Bus protocol is complex (type system, alignment, marshaling); godbus handles all of it |
| YAML config read/write | `os.WriteFile` with `yaml.Marshal` directly | Viper | Viper handles default values, env overrides, `WriteConfig` creates parent dirs, consistent with Phase 5 full settings |
| AT-SPI2 object tree walking | Custom recursive tree walker | `Cache.GetItems` bulk fetch | GetItems returns full flat tree in one call; recursion via `org.a11y.atspi.Accessible.GetChildren` requires N D-Bus calls |
| Regex compilation per call | Compile regex in filter Apply | Compile once at filter construction | `regexp.Compile` is expensive; call it in `NewCustomFilter` or when patterns change, cache compiled result |

**Key insight:** D-Bus has a typed wire format that requires careful alignment and type tagging. Using `godbus/dbus/v5` eliminates an entire class of marshaling bugs — it's the only reasonable choice for a pure-Go implementation.

---

## Common Pitfalls

### Pitfall 1: Old vs. New Cache.GetItems Signature
**What goes wrong:** `GetItems` call fails silently or panics on type mismatch for Konsole (Qt5) vs. GNOME Terminal (GTK4/GTK3).
**Why it happens:** GTK4 uses `a((so)(so)(so)iiassusau)` (int32 counts for IndexInParent/ChildCount). Qt5/older uses `a((so)(so)(so)a(so)assusau)` (array of ObjectRef for children). godbus/dbus will return a type mismatch error when you try to unmarshal the wrong struct.
**How to avoid:** Try new signature (GTK4 struct) first. If Store returns a type error, retry with the old Qt5 struct. Log which path succeeded.
**Warning signs:** `dbus: cannot unmarshal` errors in AT-SPI discovery.

### Pitfall 2: AT-SPI2 Not Enabled for GNOME Terminal GTK3
**What goes wrong:** GNOME Terminal is running but does not appear on the accessibility bus.
**Why it happens:** GTK3 apps require `gsettings set org.gnome.desktop.interface toolkit-accessibility true` before they start. Apps don't retroactively join the bus. Even on this machine (ubuntu:GNOME, GNOME Terminal running), GSettings returns `false` currently.
**How to avoid:** (1) Check GSettings in IsAvailable. (2) If false, report `adapterStatus = "onboarding"` to frontend — triggers D-06 empty state. (3) Note in onboarding that user must restart GNOME Terminal after enabling.
**Warning signs:** Bus address is valid, `IsEnabled` is false, no terminal objects discovered.

### Pitfall 3: Persistent D-Bus Connection vs. Per-Call
**What goes wrong:** Opening a new D-Bus connection per poll tick causes connection leak or slowdown.
**Why it happens:** `dbus.SessionBus()` returns a shared singleton session bus connection. `dbus.Dial(a11yAddr)` opens a new connection each call — this is expensive.
**How to avoid:** Establish the accessibility bus connection once in ATSPIAdapter startup. Store as `conn *dbus.Conn`. On connection error during a tick, attempt one reconnect; if it fails, mark adapter unavailable for this tick and log.
**Warning signs:** Growing number of file descriptors, increasing poll latency.

### Pitfall 4: Scrollback Buffer Not Accessible via AT-SPI2
**What goes wrong:** Expecting `GetText(0, -1)` to return scrollback history like `tmux capture-pane -S -`.
**Why it happens:** AT-SPI2 exposes only the visible terminal screen grid, not the scrollback buffer. `GetCharacterCount` reflects only visible characters.
**How to avoid:** Document this in the adapter. The 500ms polling of visible content is the correct model — same as the tmux visible-screen-only capture.
**Warning signs:** Character count is suspiciously small but constant even as the user types lots of content.

### Pitfall 5: libatspi2.0-0 Not Installed
**What goes wrong:** `dpkg -l libatspi2.0-0` shows `un` (not installed) on this machine. CGO consumers of libatspi would fail. Pure godbus/dbus approach is unaffected — it does not need libatspi2.0-0.
**Why it happens:** `at-spi2-core` 2.52.0 is installed (the D-Bus daemon), but the C client library is separate. Pure D-Bus needs only the daemon.
**How to avoid:** No action needed for godbus approach. Document that this phase does NOT require libatspi2.0-0 to be installed.
**Warning signs:** Temptation to add CGO bindings — resist it.

### Pitfall 6: Viper Config File Not Yet Existing
**What goes wrong:** `viper.WriteConfig()` fails if `~/.pairadmin/` directory or `config.yaml` doesn't exist.
**Why it happens:** Viper's `WriteConfig` writes to the existing config file path. If no config was read (file didn't exist), the path is not set.
**How to avoid:** Use `viper.SafeWriteConfig()` on first write, which creates the file. Or: create the directory with `os.MkdirAll` and use `viper.WriteConfigAs(path)` on first run. Check error and handle `viper.ConfigFileNotFoundError` gracefully.
**Warning signs:** `/filter add` appears to succeed but patterns are not persisted across restarts.

### Pitfall 7: addSystemMessage Missing from chatStore
**What goes wrong:** `/filter list` output has nowhere to go in the chat store — only `addUserMessage` and `addAssistantMessage` exist currently.
**Why it happens:** Phase 1-3 only needed user and assistant message types.
**How to avoid:** Add `addSystemMessage(tabId, text)` to `chatStore` and a matching `type: "system"` message variant to the `ChatMessage` type. The chat message list component needs to render system messages distinctly (no avatar, muted style).
**Warning signs:** TypeScript type errors when adding system messages.

---

## Code Examples

### AT-SPI2 Adapter: IsAvailable
```go
// Source: .planning/research/RESEARCH-TERMINAL-CAPTURE.md §5
// ATSPIAdapter.IsAvailable — checks both runtime and GSettings

func (a *ATSPIAdapter) IsAvailable(ctx context.Context) bool {
    conn, err := dbus.SessionBus()
    if err != nil {
        return false // no session bus at all
    }
    obj := conn.Object("org.a11y.Bus", "/org/a11y/bus")
    var addr string
    if err := obj.Call("org.a11y.Bus.GetAddress", 0).Store(&addr); err != nil {
        return false // org.a11y.Bus not running
    }
    return addr != ""
    // Note: IsEnabled=false just means no apps are attached yet.
    // The adapter is still "available" — onboarding handles the user flow.
}
```

### AT-SPI2 Adapter: Onboarding Status
```go
// ATSPIAdapter.OboardingRequired — returns true when user needs to run gsettings
func (a *ATSPIAdapter) OnboardingRequired(ctx context.Context) bool {
    out, err := exec.CommandContext(ctx, "gsettings", "get",
        "org.gnome.desktop.interface", "toolkit-accessibility").Output()
    if err != nil {
        return true // gsettings not available → show onboarding
    }
    return strings.TrimSpace(string(out)) != "true"
}
```

### CaptureManager: Adapter Degradation Handling
```go
// CaptureManager.Startup — registers adapters, starts with graceful degradation
func (m *CaptureManager) Startup(ctx context.Context) {
    m.ctx, m.cancel = context.WithCancel(ctx)

    for _, adapter := range m.adapters {
        if adapter.IsAvailable(ctx) {
            m.available = append(m.available, adapter)
        } else {
            // log but do not crash — tmux works even if AT-SPI2 is unavailable
            slog.Info("adapter unavailable at startup", "adapter", adapter.Name())
        }
    }

    go m.pollLoop()
}
```

### CustomFilter: Apply Method
```go
// Source: follows services/llm/filter/credential.go pattern

type customPattern struct {
    name   string
    re     *regexp.Regexp
    action string // "redact" | "remove"
}

type CustomFilter struct {
    patterns []*customPattern
}

func NewCustomFilter(patterns []config.CustomPattern) (*CustomFilter, error) {
    f := &CustomFilter{}
    for _, p := range patterns {
        re, err := regexp.Compile(p.Regex)
        if err != nil {
            return nil, fmt.Errorf("custom filter %q: invalid regex: %w", p.Name, err)
        }
        f.patterns = append(f.patterns, &customPattern{name: p.Name, re: re, action: p.Action})
    }
    return f, nil
}

func (f *CustomFilter) Apply(content string) (string, error) {
    result := content
    for _, p := range f.patterns {
        switch p.action {
        case "redact":
            result = p.re.ReplaceAllString(result, fmt.Sprintf("[REDACTED:%s]", p.name))
        case "remove":
            // Remove entire lines containing a match
            lines := strings.Split(result, "\n")
            kept := lines[:0]
            for _, line := range lines {
                if !p.re.MatchString(line) {
                    kept = append(kept, line)
                }
            }
            result = strings.Join(kept, "\n")
        }
    }
    return result, nil
}
```

### Tab Degraded State: Frontend Badge Pattern
```typescript
// frontend/src/components/terminal/TerminalTab.tsx
// Extend existing tab component with optional degraded variant

interface TerminalTabProps {
  id: string;
  name: string;
  isActive: boolean;
  degraded?: boolean;
  degradedMsg?: string;
  onClick: () => void;
}

// In render: when degraded=true, show ⚠ icon with Tooltip
// Use @base-ui/react Tooltip (not asChild pattern — per STATE.md decision)
// Pass className/onClick directly on TooltipTrigger
```

### chatStore: System Message Addition
```typescript
// frontend/src/stores/chatStore.ts — addSystemMessage pattern
// ChatMessage type needs: type: "user" | "assistant" | "system"
addSystemMessage: (tabId: string, text: string) => {
  set((state) => {
    if (!state.messages[tabId]) state.messages[tabId] = [];
    state.messages[tabId].push({
      id: crypto.randomUUID(),
      type: "system",
      text,
      timestamp: Date.now(),
    });
  });
},
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| ATK bridge (`at-spi2-atk`) for GTK3 | GTK4 uses `GtkAccessibleText` directly, no ATK | GNOME ~46, 2024 | GTK4 is simpler, but GTK3 VTE still common on Ubuntu 22.04; must handle both |
| `QT_ACCESSIBILITY=1` env required for Konsole | Qt5+ auto-loads AT-SPI bridge when daemon running | Qt 5.x+ | Konsole on modern Ubuntu should auto-attach without env var override |
| `viper` v1.x (old module path) | `github.com/spf13/viper` v1.21.0 | Ongoing | Stable API; v2 is in progress but not released; v1.21 is current stable |

**Deprecated/outdated:**
- `at-spi2-atk` bridge for GTK4: GTK4 apps no longer need it; still required for GTK3. Ubuntu 24.04 ships both.
- `org.a11y.atspi.Cache.GetItems` old signature `a((so)(so)(so)a(so)assusau)`: Deprecated in at-spi2-core >= 2.46 but still emitted by Qt5.

---

## Open Questions

1. **Does GetItems work on at-spi2-core 2.52.0 with GSettings accessibility disabled?**
   - What we know: Bus address is accessible. `IsEnabled=false`. `gnome-terminal-server` is running (PID 133059).
   - What's unclear: Whether GNOME Terminal has attached to the bus despite GSettings=false (unlikely but possible since Wayland compositor sometimes triggers it).
   - Recommendation: The Konsole spike plan should include a test: after `gsettings set toolkit-accessibility true` and restarting gnome-terminal-server, enumerate names on the a11y bus and verify role=59 objects appear.

2. **Qt5 vs Qt6 for Konsole: which Cache signature?**
   - What we know: at-spi2-core 2.52.0 is installed. Qt version unknown.
   - What's unclear: Whether installed Konsole (if any) is Qt5 or Qt6. Qt6 behavior with AT-SPI2 Cache signature.
   - Recommendation: Konsole spike plan should detect Qt version via `konsole --version` and document which Cache signature was needed.

3. **Viper v1 vs. direct yaml.v3 for Phase 4 config needs**
   - What we know: Only custom_patterns need to persist in Phase 4. Viper adds ~2MB to binary.
   - What's unclear: Whether Phase 5 will definitely adopt Viper (CFG-08 says it will).
   - Recommendation: Add Viper now — Phase 5 is committed to `~/.pairadmin/config.yaml` via Viper (CFG-08). Adding it in Phase 4 means Phase 5 builds on a working foundation, not introduces a new pattern.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `at-spi2-core` daemon | AT-SPI2 adapter | ✓ | 2.52.0 | — |
| `org.a11y.Bus` session service | AT-SPI2 IsAvailable | ✓ | Running (verified) | — |
| Accessibility bus address | AT-SPI2 connection | ✓ | `unix:path=/run/user/1000/at-spi/bus` (verified) | — |
| `gsettings` CLI | Onboarding check | ✓ | Available at `/usr/bin/gsettings` | Skip GSettings check, rely on runtime IsEnabled only |
| `gnome-terminal-server` | ATSPI-02/03 validation | ✓ | Running (PID 133059) | — |
| `libatspi2.0-0` (C library) | Not needed (pure godbus) | ✗ | `un` (not installed) | Not needed — godbus is pure Go |
| `godbus/dbus/v5` | AT-SPI2 D-Bus calls | ✓ (in go.mod indirect) | v5.1.0 | — |
| `github.com/spf13/viper` | Config persistence | ✗ (not in go.mod) | v1.21.0 available | — |
| Konsole | ATSPI-04 spike | Unknown | Not confirmed running | ⚠ badge degradation path (D-05) |
| `wails dev` / `wails build` | Integration | ✓ | v2.12.0 | — |

**Missing dependencies with no fallback:**
- `github.com/spf13/viper` — must be added via `go get` before Plan 4 (filter commands). No fallback without rewriting D-09.

**Missing dependencies with fallback:**
- `libatspi2.0-0` — not needed (fallback: pure godbus/dbus/v5 used instead of C library)
- Konsole — not confirmed present; fallback is D-05 ⚠ badge degradation

**AT-SPI2 current state — important:** GSettings `toolkit-accessibility` is currently `false` on this machine. This means GNOME Terminal has likely NOT attached to the accessibility bus even though the bus is running. The onboarding flow (D-06/D-07) is needed on this machine before testing. The spike plan must include: `gsettings set org.gnome.desktop.interface toolkit-accessibility true`, restart gnome-terminal-server, then validate.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Go framework | `testing` (stdlib) — no test runner dep |
| Frontend framework | Vitest 4.x (in vite.config.ts `test` block) |
| Go quick run | `go test ./services/capture/... -count=1` |
| Go full suite | `go test ./... -count=1` |
| Frontend quick run | `cd frontend && npm run test -- --run` |
| Frontend full suite | `cd frontend && npm run test -- --run` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ATSPI-01 | IsAvailable returns false when bus unavailable | unit | `go test ./services/capture/... -run TestATSPIAdapter_IsAvailable` | ❌ Wave 0 |
| ATSPI-01 | OnboardingRequired returns true when GSettings=false | unit | `go test ./services/capture/... -run TestATSPIAdapter_OnboardingRequired` | ❌ Wave 0 |
| ATSPI-02 | Discover returns role=59 objects | unit (mock dbus) | `go test ./services/capture/... -run TestATSPIAdapter_Discover` | ❌ Wave 0 |
| ATSPI-03 | GetText returns terminal content | unit (mock dbus) | `go test ./services/capture/... -run TestATSPIAdapter_Capture` | ❌ Wave 0 |
| ATSPI-04 | Konsole degraded → pane.Degraded=true | unit | `go test ./services/capture/... -run TestATSPIAdapter_KonsoleDegradation` | ❌ Wave 0 |
| FILT-04 | /filter add persists pattern to config | unit | `go test ./services/... -run TestLLMService_FilterCommand_Add` | ❌ Wave 0 |
| FILT-05 | /filter list returns formatted output | unit | `go test ./services/... -run TestLLMService_FilterCommand_List` | ❌ Wave 0 |
| FILT-05 | /filter remove deletes pattern | unit | `go test ./services/... -run TestLLMService_FilterCommand_Remove` | ❌ Wave 0 |
| FILT-04 | CustomFilter.Apply redacts matching content | unit | `go test ./services/llm/filter/... -run TestCustomFilter_Apply_Redact` | ❌ Wave 0 |
| FILT-04 | CustomFilter.Apply removes matching lines | unit | `go test ./services/llm/filter/... -run TestCustomFilter_Apply_Remove` | ❌ Wave 0 |
| ATSPI-01 | TerminalPreview shows AT-SPI2 onboarding empty state | unit (vitest) | `cd frontend && npm run test -- --run --reporter=verbose` | ❌ Wave 0 |
| FILT-05 | chatStore.addSystemMessage adds system message | unit (vitest) | `cd frontend && npm run test -- --run --reporter=verbose` | ❌ Wave 0 |
| CaptureManager | tmux + AT-SPI2 pane IDs never collide (namespace) | unit | `go test ./services/capture/... -run TestCaptureManager_PaneIDNamespace` | ❌ Wave 0 |
| CaptureManager | Membership change emits terminal:tabs event | unit | `go test ./services/capture/... -run TestCaptureManager_MembershipEvent` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `go test ./services/capture/... -count=1` (Go) + `cd frontend && npm run test -- --run` (frontend)
- **Per wave merge:** `go test ./... -count=1`
- **Phase gate:** Full Go + frontend test suites green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `services/capture/adapter.go` — defines `TerminalAdapter` interface and shared types
- [ ] `services/capture/manager_test.go` — CaptureManager unit tests with injectable emitFn
- [ ] `services/capture/tmux_test.go` — TmuxAdapter tests (refactored from services/terminal_test.go)
- [ ] `services/capture/atspi_test.go` — ATSPIAdapter tests with mock dbus connection interface
- [ ] `services/llm/filter/custom_test.go` — CustomFilter Apply tests (redact + remove actions)
- [ ] `services/config/config_test.go` — AppConfig LoadConfig/SaveConfig with temp dir
- [ ] Frontend: chatStore test for `addSystemMessage`
- [ ] Frontend: TerminalPreview test for AT-SPI2 onboarding empty state (`adapterStatus` prop)

*(No framework install needed — Go stdlib testing and Vitest are already configured)*

---

## Sources

### Primary (HIGH confidence)
- `.planning/research/RESEARCH-TERMINAL-CAPTURE.md` — AT-SPI2 D-Bus API, godbus patterns, CacheItem signatures, GNOME Terminal tree, Konsole D-Bus interface
- `services/terminal.go` + `services/terminal_test.go` — exact existing patterns to replicate in capture package
- `services/llm/filter/filter.go` + `credential.go` — filter pipeline to extend with CustomFilter
- `go.mod` — confirmed godbus/dbus/v5 v5.1.0 present; viper absent; yaml.v3 present
- Runtime verification: `dbus-send` confirmed `org.a11y.Bus.GetAddress` returns `unix:path=/run/user/1000/at-spi/bus`; `gsettings get` returns `false`; `gnome-terminal-server` running (PID 133059); `at-spi2-core` 2.52.0 installed

### Secondary (MEDIUM confidence)
- `go list -m -versions github.com/spf13/viper` — confirmed v1.21.0 is latest stable (2026-03-28)
- `dpkg -l at-spi2-core libatspi2.0-0` — confirmed at-spi2-core installed, libatspi2.0-0 NOT installed

### Tertiary (LOW confidence)
- Qt5/Qt6 AT-SPI2 Cache signature behavior for Konsole — not directly tested; from prior research notes; Konsole not confirmed installed on this machine

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — godbus confirmed in go.mod; Viper confirmed available at v1.21.0; both have stable APIs
- Architecture: HIGH — follows established Phase 3 patterns; adapter interface from prior research; runtime environment verified
- AT-SPI2 GNOME Terminal: MEDIUM — bus confirmed accessible; GSettings disabled currently; actual text extraction from gnome-terminal-server not live-tested (will be validated in spike plan)
- Konsole support: LOW — not installed, Qt5/Qt6 Cache signature unknown, text extraction unconfirmed
- Filter commands: HIGH — follows existing `/clear` pattern; Viper API is stable and well-documented
- Pitfalls: HIGH — verified against runtime state on this machine

**Research date:** 2026-03-28
**Valid until:** 2026-04-27 (30 days — AT-SPI2 and Viper are stable; godbus has no imminent major version)
