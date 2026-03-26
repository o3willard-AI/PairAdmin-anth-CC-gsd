# Terminal Capture Research for PairAdmin

**Project:** PairAdmin (Go desktop app, LLM context assistant)
**Researched:** 2026-03-25
**Scope:** Linux terminal content capture — tmux, GNOME Terminal (VTE/AT-SPI2), Konsole
**Overall confidence:** MEDIUM-HIGH (tmux: HIGH, AT-SPI2/GNOME Terminal: MEDIUM, Konsole: LOW-MEDIUM)

---

## 1. tmux capture-pane: Complete Go Subprocess API

### How It Works

`tmux capture-pane` is the canonical way to read terminal content from a tmux pane. It writes the visible pane content (and optionally history) to a tmux buffer or directly to stdout. There is no IPC or socket — you call the `tmux` binary as a subprocess.

**Confidence: HIGH** — Verified against tmux man page and source code.

### Key Flags

| Flag | Meaning |
|------|---------|
| `-p` | Print to stdout instead of a buffer (required for programmatic use) |
| `-t <target>` | Target pane in `session:window.pane` notation (e.g., `mysession:0.1`) |
| `-S <start>` | Start line. `0` = first visible line. Negative = N lines back in history. `-` = beginning of scrollback |
| `-E <end>` | End line. `-` = last visible line (default). Use `-E -1` for last line |
| `-e` | Include ANSI escape sequences for color/attributes in output |
| `-J` | Join wrapped lines and preserve trailing spaces; implies `-T` |
| `-b <name>` | Named buffer destination (when not using `-p`) |

### Minimal Go Implementation

```go
package capture

import (
    "context"
    "os/exec"
    "strings"
)

// CapturePane captures visible content of one pane.
// target format: "session:window.pane" e.g. "main:0.0"
func CapturePane(ctx context.Context, target string) (string, error) {
    out, err := exec.CommandContext(ctx,
        "tmux", "capture-pane",
        "-p",          // print to stdout
        "-t", target,  // target pane
    ).Output()
    if err != nil {
        return "", err
    }
    return strings.TrimRight(string(out), "\n"), nil
}

// CapturePaneHistory captures full scrollback history.
func CapturePaneHistory(ctx context.Context, target string) (string, error) {
    out, err := exec.CommandContext(ctx,
        "tmux", "capture-pane",
        "-p",
        "-S", "-",   // from beginning of scrollback
        "-E", "-",   // to end of visible area
        "-t", target,
    ).Output()
    if err != nil {
        return "", err
    }
    return string(out), nil
}
```

**Important:** `-S -` captures the entire scrollback buffer, which can be megabytes for a long-running session. For a 500ms polling loop targeting "recent context," use `-S -50` (last 50 lines) or `-S -` with a line-count cap applied afterward.

### Discovering All Sessions, Windows, and Panes

Use `list-panes -a` with a custom format to enumerate everything in one call:

```go
func ListAllPanes(ctx context.Context) ([]PaneRef, error) {
    // -a = all sessions; -F = custom format with a unique separator
    out, err := exec.CommandContext(ctx,
        "tmux", "list-panes", "-a",
        "-F", "#{session_name}\t#{window_index}\t#{pane_index}\t#{pane_id}\t#{pane_pid}\t#{pane_width}\t#{pane_height}",
    ).Output()
    if err != nil {
        // tmux not running or no sessions: treat as empty, not fatal
        if strings.Contains(err.Error(), "no server running") {
            return nil, nil
        }
        return nil, err
    }

    var panes []PaneRef
    for _, line := range strings.Split(strings.TrimSpace(string(out)), "\n") {
        if line == "" {
            continue
        }
        parts := strings.Split(line, "\t")
        if len(parts) < 7 {
            continue
        }
        panes = append(panes, PaneRef{
            Session:     parts[0],
            WindowIndex: parts[1],
            PaneIndex:   parts[2],
            PaneID:      parts[3], // e.g. "%0" — stable across renames
            PID:         parts[4],
            Width:       parts[5],
            Height:      parts[6],
            Target:      parts[0] + ":" + parts[1] + "." + parts[2],
        })
    }
    return panes, nil
}
```

**Format variables for `-F`:**

| Variable | Meaning |
|----------|---------|
| `#{session_name}` | Session name string |
| `#{session_id}` | Stable session ID (e.g. `$0`) |
| `#{window_index}` | 0-based window number |
| `#{window_name}` | Window title string |
| `#{pane_index}` | 0-based pane number within window |
| `#{pane_id}` | Stable pane ID (e.g. `%3`) — survives moves/renames |
| `#{pane_pid}` | PID of process in pane |
| `#{pane_width}` / `#{pane_height}` | Dimensions |
| `#{pane_current_command}` | Command name running in pane |
| `#{pane_current_path}` | CWD of pane process |

### Targeting by Stable Pane ID

After enumeration, use the stable pane ID (e.g. `%3`) as the capture target rather than `session:window.pane`, which can shift when windows are moved:

```go
// Capture by stable pane ID
func CapturePaneByID(ctx context.Context, paneID string) (string, error) {
    out, err := exec.CommandContext(ctx,
        "tmux", "capture-pane", "-p", "-t", paneID,
    ).Output()
    // ...
}
```

### Scroll Position and Encoding

- Without `-S`, `capture-pane` captures only the **visible terminal screen** (what you'd see on the terminal right now), not history.
- With `-S -`, it captures the entire scrollback buffer from the beginning. Content may contain UTF-8 multibyte characters; Go's `string(out)` handles this correctly — treat the output as `[]rune` only if you need to count characters, not bytes.
- The `-e` flag includes raw ANSI escape codes in output. For LLM context, omit `-e` (plain text is cleaner and far smaller).
- Wrapped long lines appear as separate lines unless `-J` is used.

### Multiple Concurrent Panes

Run captures concurrently using goroutines with a semaphore to avoid spawning too many subprocesses:

```go
import "golang.org/x/sync/semaphore"

func CaptureAllPanes(ctx context.Context, panes []PaneRef) map[string]string {
    sem := semaphore.NewWeighted(4) // max 4 concurrent tmux subprocesses
    results := make(map[string]string, len(panes))
    var mu sync.Mutex
    var wg sync.WaitGroup

    for _, p := range panes {
        wg.Add(1)
        go func(pr PaneRef) {
            defer wg.Done()
            sem.Acquire(ctx, 1)
            defer sem.Release(1)
            content, err := CapturePaneByID(ctx, pr.PaneID)
            if err == nil {
                mu.Lock()
                results[pr.PaneID] = content
                mu.Unlock()
            }
        }(p)
    }
    wg.Wait()
    return results
}
```

**Practical limit:** Each `tmux capture-pane` invocation forks a process and communicates via tmux's Unix socket. For 10–20 panes at 500ms, 4 concurrent subprocesses is a safe upper bound. Each call typically completes in < 5ms for visible-only captures.

### Available Go Libraries

| Library | Notes |
|---------|-------|
| `github.com/jubnzv/go-tmux` | Has `Pane.Capture()` but uses `session:window.pane` notation (not stable pane ID). Thin wrapper. No scrollback support. |
| `github.com/k1LoW/tcmux/tmux` | Similar thin wrapper. |

**Recommendation:** Write your own thin wrapper around `exec.Command` rather than depending on these libraries. The API is simple enough (4–5 commands total), and the libraries do not offer scrollback capture or stable pane ID targeting.

---

## 2. AT-SPI2 on Linux: Architecture and Go Access

### Architecture Overview

AT-SPI2 is a D-Bus-based accessibility protocol. It does **not** use the regular session bus directly — it runs on a **separate accessibility bus**.

Connection flow:
1. On the session bus, contact `org.a11y.Bus` at object path `/org/a11y/bus`
2. Call `org.a11y.Bus.GetAddress` — returns the address of the accessibility bus (e.g., `unix:abstract=/tmp/dbus-XXXXXX`)
3. Open a new D-Bus connection to that address
4. On the accessibility bus, contact `org.a11y.atspi.Registry` at `/org/a11y/atspi/registry`
5. Enumerate applications by calling `GetItems` on each app's Cache interface

**Confidence: HIGH** — Verified against official at-spi2-core documentation and README.

### Is There a Go Binding for AT-SPI2?

**No native Go AT-SPI2 binding exists.** (LOW confidence that one exists; extensive search found none.) Your options are:

1. **Pure Go via godbus/dbus/v5** — Call the raw D-Bus interfaces directly. No CGO required. This is the right approach.
2. **CGO via libatspi** — Call the C library `libatspi-2.0` via CGO. Adds a C dependency, complicates cross-compilation, increases binary size. Not recommended unless raw D-Bus turns out to be inadequate.

**Recommendation: Use `github.com/godbus/dbus/v5` with raw D-Bus calls.** No CGO needed.

### Connecting to the Accessibility Bus in Go

```go
import (
    "fmt"
    dbus "github.com/godbus/dbus/v5"
)

func connectToA11yBus() (*dbus.Conn, error) {
    // Step 1: Get the accessibility bus address from session bus
    sessionConn, err := dbus.SessionBus()
    if err != nil {
        return nil, fmt.Errorf("session bus: %w", err)
    }

    var a11yAddr string
    obj := sessionConn.Object("org.a11y.Bus", "/org/a11y/bus")
    if err := obj.Call("org.a11y.Bus.GetAddress", 0).Store(&a11yAddr); err != nil {
        return nil, fmt.Errorf("get a11y bus address: %w", err)
    }

    // Step 2: Connect to the accessibility bus directly
    a11yConn, err := dbus.Dial(a11yAddr)
    if err != nil {
        return nil, fmt.Errorf("dial a11y bus: %w", err)
    }
    if err := a11yConn.Auth(nil); err != nil {
        a11yConn.Close()
        return nil, fmt.Errorf("auth: %w", err)
    }
    if err := a11yConn.Hello(); err != nil {
        a11yConn.Close()
        return nil, fmt.Errorf("hello: %w", err)
    }
    return a11yConn, nil
}
```

### Checking Whether Accessibility Is Enabled

```go
func IsA11yEnabled() (bool, error) {
    sessionConn, err := dbus.SessionBus()
    if err != nil {
        return false, err
    }
    obj := sessionConn.Object("org.a11y.Bus", "/org/a11y/bus")

    var isEnabled bool
    if err := obj.Call("org.freedesktop.DBus.Properties.Get", 0,
        "org.a11y.Status", "IsEnabled").Store(&isEnabled); err != nil {
        // If org.a11y.Bus is not present at all, a11y is not running
        return false, nil
    }
    return isEnabled, nil
}
```

Alternatively, check GSettings:
```go
// Shell equivalent: gsettings get org.gnome.desktop.interface toolkit-accessibility
// Returns "true" or "false"
func IsToolkitAccessibilityEnabled(ctx context.Context) bool {
    out, err := exec.CommandContext(ctx,
        "gsettings", "get",
        "org.gnome.desktop.interface", "toolkit-accessibility",
    ).Output()
    if err != nil {
        return false
    }
    return strings.TrimSpace(string(out)) == "true"
}
```

**Note:** GSettings controls auto-start at login; `org.a11y.Status.IsEnabled` reflects current runtime state. The accessibility bus can be running even if the GSettings key is false (e.g., something called `GetAddress` manually). Check `IsEnabled` for runtime state.

### Key D-Bus Interfaces on the Accessibility Bus

| Interface | Object Path | Purpose |
|-----------|-------------|---------|
| `org.a11y.atspi.Registry` | `/org/a11y/atspi/registry` | Event registration |
| `org.a11y.atspi.Accessible` | per-object | Object hierarchy traversal |
| `org.a11y.atspi.Text` | per-object | Read text content |
| `org.a11y.atspi.Cache` | per-app | Bulk fetch all accessible objects |
| `org.a11y.atspi.Component` | per-object | Screen coordinates |

### Enumerating Accessible Applications and Finding Terminals

The pattern used by libatspi itself: call `GetItems` on each application's Cache interface to bulk-fetch its entire accessible tree, then filter by role.

```go
// ObjectRef is a D-Bus (service, path) pair
type ObjectRef struct {
    Name string          `dbus:"name"`
    Path dbus.ObjectPath `dbus:"path"`
}

// CacheItem matches the D-Bus signature: a((so)(so)(so)iiassusau)
type CacheItem struct {
    Ref        ObjectRef
    AppRef     ObjectRef
    ParentRef  ObjectRef
    IndexInParent int32
    ChildCount    int32
    Interfaces    []string
    Name          string
    Role          uint32
    Description   string
    StateSet      []uint32
}

const RoleTerminal = uint32(59) // ATSPI_ROLE_TERMINAL

func FindTerminalObjects(a11yConn *dbus.Conn) ([]ObjectRef, error) {
    // List names on the accessibility bus
    var names []string
    busObj := a11yConn.BusObject()
    if err := busObj.Call("org.freedesktop.DBus.ListNames", 0).Store(&names); err != nil {
        return nil, err
    }

    var terminals []ObjectRef
    for _, name := range names {
        if !strings.HasPrefix(name, ":") {
            // Skip well-known names; apps register as unique names
            continue
        }
        // Each app exposes /org/a11y/atspi/accessible/root
        // and a Cache interface at the same path or /org/a11y/atspi/cache
        appObj := a11yConn.Object(name, "/org/a11y/atspi/accessible/root")

        var items []CacheItem
        // NOTE: signature varies by toolkit — try new signature first
        if err := appObj.Call("org.a11y.atspi.Cache.GetItems", 0).Store(&items); err != nil {
            continue // app may not implement Cache
        }

        for _, item := range items {
            if item.Role == RoleTerminal {
                terminals = append(terminals, item.Ref)
            }
        }
    }
    return terminals, nil
}
```

**Important caveat:** The GetItems signature in the cache interface differs between GTK4 and Qt5:
- GTK4 (new): `a((so)(so)(so)iiassusau)`
- Qt5 (old): `a((so)(so)(so)a(so)assusau)`

You may need to attempt both and handle the type error.

### Reading Text Content via org.a11y.atspi.Text

Once you have a `(service, objectPath)` pair for a terminal object:

```go
// GetText retrieves text content from an AT-SPI2 Text interface object.
// startOffset=0, endOffset=-1 retrieves all text.
func GetAccessibleText(a11yConn *dbus.Conn, ref ObjectRef, start, end int32) (string, error) {
    obj := a11yConn.Object(ref.Name, ref.Path)
    var text string
    err := obj.Call("org.a11y.atspi.Text.GetText", 0, start, end).Store(&text)
    return text, err
}

// GetCharacterCount returns the total number of characters.
func GetCharacterCount(a11yConn *dbus.Conn, ref ObjectRef) (int32, error) {
    obj := a11yConn.Object(ref.Name, ref.Path)
    var count int32
    err := obj.Call("org.a11y.atspi.Text.GetCharacterCount", 0).Store(&count)
    return count, err
}
```

The `GetText` D-Bus method signature (from `xml/Text.xml`):
```xml
<method name="GetText">
  <arg direction="in" name="startOffset" type="i"/>
  <arg direction="in" name="endOffset" type="i"/>
  <arg direction="out" type="s"/>
</method>
```

Pass `startOffset=0, endOffset=-1` to get all text. The returned string is UTF-8.

---

## 3. GNOME Terminal Specifically

### Application Identity

GNOME Terminal runs as a DBus-activated service: `org.gnome.Terminal`. The terminal server process is `gnome-terminal-server`. In the AT-SPI tree, the application name is `"gnome-terminal-server"` (not `"gnome-terminal"`).

### AT-SPI2 Object Tree

GNOME Terminal (via VTE) exposes this hierarchy:

```
Application (gnome-terminal-server)
  └── Frame / Window
        ├── Menu Bar (optional, if shown)
        └── Tab Bar (page tab list)
              └── Page Tab [per tab]
                    └── Scroll Pane
                          └── Terminal (role=59, ATSPI_ROLE_TERMINAL)
```

The **Terminal** node is the widget of interest. It implements `org.a11y.atspi.Text`.

### GTK3 vs GTK4 Situation (important caveat)

- **GTK3 GNOME Terminal (legacy):** Uses ATK bridge (`at-spi2-atk`). The VTE widget implements the ATK text interface via `vteAccessibleGetText()`. Text is accessible but may not emit `object:text-changed` events reliably (per VTE GitLab issue #88, filed 2020, status unclear as of research date).
- **GTK4 GNOME Terminal (GNOME 47+, shipping ~2024):** Uses `GtkAccessibleText` directly without ATK. VTE implements this interface. As of GNOME 46, VTE supports text accessibility via the new interface. **However**, the Newton/AccessKit project noted that the GTK4 `GtkAccessibleText` interface is not yet supported by all accessibility stacks as of mid-2024.

**Practical implication:** For GNOME Terminal GTK3 (Ubuntu 22.04, older systems), AT-SPI2 text access works but may be unreliable for change events. For GNOME Terminal GTK4 (Ubuntu 24.04+, GNOME 47+), text access should work but is newer and has less battle-testing.

**Recommendation:** Use AT-SPI2 for GNOME Terminal as a best-effort adapter, with polling as the fallback mechanism.

### Finding the Terminal Widget

```go
func FindGNOMETerminalPanes(a11yConn *dbus.Conn) ([]ObjectRef, error) {
    // GNOME Terminal server registers under its own unique bus name,
    // not a well-known name on the a11y bus.
    // Search all registered names for objects with role TERMINAL.
    return FindTerminalObjects(a11yConn) // from section 2
}
```

You can also filter by application name via `org.a11y.atspi.Application.GetToolkitName` or by checking `GetAttributes` for the app name, but filtering by role=59 is simpler.

### Gotchas

1. **Process name is `gnome-terminal-server`, not `gnome-terminal`.** The terminal process is a single instance — all terminal windows share one server process.
2. **Tabs appear as page tabs in the tree**, not as separate application instances. You must descend the tree to find all Terminal widgets.
3. **Text content is the visible screen only**, not scrollback history. AT-SPI2 does not expose the scrollback buffer — only the current viewport.
4. **Carriage returns and control characters** may appear in the raw accessible text since the terminal renders a grid, not a stream.
5. **GTK4 migration is ongoing.** Detecting the GTK version at runtime requires checking the toolkit name via `org.a11y.atspi.Application.GetToolkitName`.

---

## 4. Konsole Specifically

### D-Bus Interface (Control, Not Content)

Konsole exposes a rich D-Bus control interface but it is designed for **sending commands**, not reading content.

Service discovery:
```bash
# Each Konsole window registers as: org.kde.konsole-<PID>
qdbus | grep konsole
# e.g.: org.kde.konsole-12345
```

Available interfaces and methods:
```
Service: org.kde.konsole-<PID>

/MainWindow_<N>  (one per window)
  org.kde.konsole.Window:
    - newSession() → int
    - currentSession() → int
    - setCurrentSession(int)
    - resizeSplits(...)

/Sessions/<N>  (one per tab)
  org.kde.konsole.Session:
    - sendText(text: string)
    - runCommand(command: string)
    - setTitle(titleType: int, title: string)
    - title(titleType: int) → string
    - environment() → string[]
    - setEnvironment(string[])
    - setTabTitleFormat(type: int, format: string)
    - foregroundProcessName() → string  [process running in tab]
    - currentWorkingDirectory() → string
```

**There is no `getScreenContent`, `getBuffer`, or equivalent method in Konsole's D-Bus interface.** The interface was designed for automation (scripting workflows) and does not expose terminal buffer reads.

**Confidence: MEDIUM** — Verified against official KDE documentation and community scripting guides. The absence of a content-read method is consistent across all sources found.

### AT-SPI2 for Konsole

Qt5 (Konsole is Qt5/Qt6) exposes accessibility via the `qtatspi` bridge plugin. The bridge requires:
- `QT_ACCESSIBILITY=1` environment variable set **before Konsole starts** (Qt4/early Qt5 only; modern Qt5+ loads it automatically if AT-SPI2 is present)
- `at-spi2-core` packages installed

When enabled, Konsole exposes its widget tree via AT-SPI2 on the accessibility bus. The terminal widget (a `QTermWidget`-derived class) should expose role `ATSPI_ROLE_TERMINAL` (59).

**However:** Qt5's `Cache.GetItems` uses the **old deprecated signature** `a((so)(so)(so)a(so)assusau)` rather than the new `a((so)(so)(so)iiassusau)`. Your Go code must handle both.

```go
// Handle both old Qt5 signature and new GTK4 signature
func getItemsFromApp(conn *dbus.Conn, busName string) ([]ObjectRef, error) {
    obj := conn.Object(busName, "/org/a11y/atspi/accessible/root")

    // Try new signature first (GTK4, at-spi2-core >= 2.46)
    var newItems []CacheItem  // uses iiassusau
    if err := obj.Call("org.a11y.atspi.Cache.GetItems", 0).Store(&newItems); err == nil {
        return extractTerminalRefs(newItems), nil
    }

    // Fall back to old Qt5 signature (a(so) children instead of ii counts)
    // This requires a separate struct with []ObjectRef for children
    // ... (handle old format)
    return nil, nil
}
```

**Key uncertainty:** Whether Konsole's Qt-based terminal widget actually implements `org.a11y.atspi.Text.GetText` reliably is unconfirmed. The Qt AT-SPI bridge exposes the widget tree, but terminal widget text exposure depends on Qt's `QAccessibleTextInterface` implementation for `QTermWidget`. No authoritative source confirms this works end-to-end for Konsole specifically.

**Recommendation:** Treat Konsole AT-SPI2 text access as **experimental**. Implement it, test it, and degrade gracefully. The `foregroundProcessName` D-Bus method can confirm a Konsole session is active even when text capture fails.

---

## 5. Permission Requirements and Detection

### Required Packages

On Debian/Ubuntu:
```
at-spi2-core         (the daemon and bus launcher)
libatspi2.0-0        (runtime library — only needed for C consumers)
```

On Fedora/RHEL:
```
at-spi2-core
at-spi2-atk          (needed for GTK3 apps)
```

No special filesystem permissions are needed. The accessibility bus runs as the user's session process.

### What Must Be Configured

For **GNOME Terminal** (GTK3): GSettings key must be enabled for the AT-SPI bridge to load at app startup:
```bash
gsettings set org.gnome.desktop.interface toolkit-accessibility true
```

For **GTK4** apps (GNOME Terminal GTK4, newer): GTK4 calls D-Bus directly; the GSettings key is not strictly required, but `at-spi-bus-launcher` must be running.

For **Konsole** (Qt5/Qt6): The `QT_ACCESSIBILITY=1` env var is needed for Qt4 only. Qt5 and Qt6 load the AT-SPI bridge automatically when `at-spi-bus-launcher` is running.

### Programmatic Detection

```go
type A11yStatus struct {
    BusAvailable     bool
    BusEnabled       bool
    ScreenReaderRunning bool
    BusAddress       string
}

func DetectA11yStatus() A11yStatus {
    status := A11yStatus{}

    conn, err := dbus.SessionBus()
    if err != nil {
        return status
    }
    defer conn.Close()

    obj := conn.Object("org.a11y.Bus", "/org/a11y/bus")

    // Check IsEnabled property
    var enabled bool
    if err := obj.Call("org.freedesktop.DBus.Properties.Get", 0,
        "org.a11y.Status", "IsEnabled").Store(&enabled); err == nil {
        status.BusAvailable = true
        status.BusEnabled = enabled
    }

    // Check IsScreenReaderRunning property
    var srRunning bool
    if err := obj.Call("org.freedesktop.DBus.Properties.Get", 0,
        "org.a11y.Status", "IsScreenReaderRunning").Store(&srRunning); err == nil {
        status.ScreenReaderRunning = srRunning
    }

    // Get the actual bus address
    var addr string
    if err := obj.Call("org.a11y.Bus.GetAddress", 0).Store(&addr); err == nil {
        status.BusAddress = addr
    }

    return status
}
```

**The GSettings check is a separate concern** from runtime status. The GSettings key controls autostart; the runtime `IsEnabled` property reflects whether apps are currently connecting to the bus. Check both:

```go
func FullA11yReadiness(ctx context.Context) (gsettingsEnabled bool, runtimeEnabled bool) {
    out, _ := exec.CommandContext(ctx, "gsettings", "get",
        "org.gnome.desktop.interface", "toolkit-accessibility").Output()
    gsettingsEnabled = strings.TrimSpace(string(out)) == "true"

    s := DetectA11yStatus()
    runtimeEnabled = s.BusEnabled
    return
}
```

---

## 6. Adapter Interface Design

### Core Types

```go
// PaneID is an opaque stable identifier for a terminal pane.
// For tmux: the tmux pane ID (e.g. "%3")
// For GUI terminals: the AT-SPI object path or a synthetic ID
type PaneID string

// PaneInfo describes a discoverable terminal pane.
type PaneInfo struct {
    ID          PaneID
    AdapterType string            // "tmux", "gnome-terminal", "konsole"
    DisplayName string            // human-readable label
    Metadata    map[string]string // adapter-specific (e.g. "session", "pid", "tab")
}

// CaptureResult holds the result of one capture attempt.
type CaptureResult struct {
    PaneID    PaneID
    Content   string
    Timestamp time.Time
    Hash      uint64 // FNV-1a hash for deduplication
    Error     error
}

// TerminalAdapter is the interface each backend must implement.
type TerminalAdapter interface {
    // Name returns a stable identifier for this adapter type.
    Name() string

    // IsAvailable returns true if this adapter can function in the current environment.
    // Called once at startup and cached. Should not block.
    IsAvailable(ctx context.Context) bool

    // Discover returns all currently available panes/sessions for this adapter.
    // Called each poll cycle or on a slower discovery interval.
    Discover(ctx context.Context) ([]PaneInfo, error)

    // Capture returns the current content of the given pane.
    // Must return within the polling interval (500ms budget total).
    Capture(ctx context.Context, pane PaneInfo) (CaptureResult, error)

    // Close releases any persistent connections held by the adapter.
    Close() error
}
```

### Adapter Registry

```go
// CaptureManager coordinates multiple adapters.
type CaptureManager struct {
    adapters []TerminalAdapter
    mu       sync.RWMutex
    known    map[PaneID]PaneInfo // currently known panes
}

func NewCaptureManager(adapters ...TerminalAdapter) *CaptureManager {
    return &CaptureManager{
        adapters: adapters,
        known:    make(map[PaneID]PaneInfo),
    }
}

// PollAll discovers and captures all available panes across all adapters.
// Returns only panes whose content changed since last call.
func (m *CaptureManager) PollAll(ctx context.Context, prevHashes map[PaneID]uint64) ([]CaptureResult, error) {
    var allPanes []PaneInfo

    for _, adapter := range m.adapters {
        panes, err := adapter.Discover(ctx)
        if err != nil {
            // Log and continue — one adapter failure should not block others
            continue
        }
        allPanes = append(allPanes, panes...)
    }

    results := make([]CaptureResult, 0, len(allPanes))
    var mu sync.Mutex
    var wg sync.WaitGroup

    for _, pane := range allPanes {
        wg.Add(1)
        go func(p PaneInfo) {
            defer wg.Done()
            adapter := m.adapterFor(p.AdapterType)
            if adapter == nil {
                return
            }
            result, err := adapter.Capture(ctx, p)
            if err != nil {
                return
            }
            // Only emit if content changed
            if prevHashes[p.ID] != result.Hash {
                mu.Lock()
                results = append(results, result)
                mu.Unlock()
            }
        }(pane)
    }
    wg.Wait()
    return results, nil
}

func (m *CaptureManager) adapterFor(name string) TerminalAdapter {
    for _, a := range m.adapters {
        if a.Name() == name {
            return a
        }
    }
    return nil
}
```

### Concrete Adapter Implementations (skeleton)

```go
// TmuxAdapter implements TerminalAdapter for tmux sessions.
type TmuxAdapter struct{}

func (a *TmuxAdapter) Name() string { return "tmux" }

func (a *TmuxAdapter) IsAvailable(ctx context.Context) bool {
    _, err := exec.LookPath("tmux")
    if err != nil {
        return false
    }
    // Also check that a tmux server is actually running
    err = exec.CommandContext(ctx, "tmux", "list-sessions").Run()
    return err == nil
}

func (a *TmuxAdapter) Discover(ctx context.Context) ([]PaneInfo, error) {
    panes, err := ListAllPanes(ctx)
    if err != nil {
        return nil, err
    }
    result := make([]PaneInfo, 0, len(panes))
    for _, p := range panes {
        result = append(result, PaneInfo{
            ID:          PaneID("tmux:" + p.PaneID),
            AdapterType: "tmux",
            DisplayName: p.Session + ":" + p.WindowIndex + "." + p.PaneIndex,
            Metadata: map[string]string{
                "session": p.Session,
                "pane_id": p.PaneID,
                "pid":     p.PID,
            },
        })
    }
    return result, nil
}

func (a *TmuxAdapter) Capture(ctx context.Context, pane PaneInfo) (CaptureResult, error) {
    paneID := pane.Metadata["pane_id"]
    content, err := CapturePaneByID(ctx, paneID)
    if err != nil {
        return CaptureResult{PaneID: pane.ID, Error: err}, err
    }
    return CaptureResult{
        PaneID:    pane.ID,
        Content:   content,
        Timestamp: time.Now(),
        Hash:      fnv1aHash(content),
    }, nil
}

func (a *TmuxAdapter) Close() error { return nil }

// ATSPIAdapter implements TerminalAdapter for GUI terminals via AT-SPI2.
type ATSPIAdapter struct {
    conn *dbus.Conn
}

func (a *ATSPIAdapter) Name() string { return "atspi" }

func (a *ATSPIAdapter) IsAvailable(ctx context.Context) bool {
    s := DetectA11yStatus()
    return s.BusAvailable && s.BusEnabled
}

// ... Discover and Capture using FindTerminalObjects and GetAccessibleText from above
```

---

## 7. Polling vs. Event-Driven

### tmux: Control Mode for Event-Driven Notifications

tmux supports a "control mode" (`tmux -C`) that streams machine-readable output over stdin/stdout. In control mode, any pane output causes a `%output` notification:

```
%output %0 some text from the pane
%output %3 another line of output
```

Format:
```
%output <pane-id> <octal-escaped content>
%extended-output <pane-id> <lag-ms> : <content>  (when flow control enabled)
```

This is **genuinely event-driven** — content changes trigger output immediately, not on a polling schedule. An alternative to polling `capture-pane`:

```go
// StartControlMode launches a tmux control mode client and streams %output events.
// Note: control mode attaches to ONE session only.
func StartControlMode(ctx context.Context, session string, out chan<- ControlEvent) error {
    cmd := exec.CommandContext(ctx, "tmux", "-C", "attach-session", "-t", session)
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return err
    }
    cmd.Stdin = os.Stdin // control mode requires stdin
    if err := cmd.Start(); err != nil {
        return err
    }

    go func() {
        scanner := bufio.NewScanner(stdout)
        for scanner.Scan() {
            line := scanner.Text()
            if strings.HasPrefix(line, "%output ") {
                parts := strings.SplitN(line, " ", 3)
                if len(parts) == 3 {
                    out <- ControlEvent{PaneID: parts[1], Content: unescapeOctal(parts[2])}
                }
            }
        }
    }()
    return nil
}
```

**Limitations of control mode:**
- A control mode client attaches to **one session at a time**. If you have multiple tmux sessions, you need one control mode connection per session.
- The `%output` notification gives you the raw incremental output (new bytes written to the pane), **not the full screen content**. You would need to maintain a virtual terminal state machine to reconstruct screen content, or still call `capture-pane` on `%output` events.
- Complexity is substantially higher than polling. For a 500ms polling loop, the control mode approach is not worth it.

**Verdict:** Use polling with `capture-pane` at 500ms. The `%output` event is useful to know a pane has changed (to skip capture when nothing happened), but reconstructing content from incremental output requires a VT100 state machine, which is complex.

**Practical hybrid approach:** Use `tmux` hooks (not control mode) to write a timestamp to a file when any pane changes, then only call `capture-pane` if the timestamp changed. But this adds filesystem complexity. The simpler approach — just poll `capture-pane` and deduplicate by hash — is sufficient.

### AT-SPI2: Event-Driven Is Available But Unreliable for Terminals

AT-SPI2 emits `object:text-changed` events via D-Bus signals. You can subscribe:

```go
func SubscribeToTextChanges(a11yConn *dbus.Conn, events chan<- *dbus.Signal) error {
    if err := a11yConn.AddMatchSignal(
        dbus.WithMatchInterface("org.a11y.atspi.Event.Object"),
        dbus.WithMatchMember("TextChanged"),
    ); err != nil {
        return err
    }
    ch := make(chan *dbus.Signal, 64)
    a11yConn.Signal(ch)
    go func() {
        for sig := range ch {
            events <- sig
        }
    }()
    return nil
}
```

However, VTE (GNOME Terminal's widget) has a **documented history of not emitting `object:text-changed` events** reliably (VTE GitLab issue #88). The GTK4 port with `GtkAccessibleText` improves this, but deployment is incomplete across distros.

**Verdict:** Do not rely on AT-SPI2 events as your primary change notification. Use polling. If an event fires, use it to trigger an immediate capture (saving up to 500ms of latency), but always fall back to the poll timer.

### Summary: Polling Is the Right Default

| Source | Event Support | Recommendation |
|--------|--------------|----------------|
| tmux `capture-pane` | None natively | Poll at 500ms |
| tmux control mode `%output` | Yes, but incremental only | Optional enhancement |
| AT-SPI2 `object:text-changed` | Yes, but unreliable on VTE | Optional enhancement, not primary |
| Konsole D-Bus | No content events | Poll only |

---

## 8. Cross-Adapter Deduplication

### Hash-Based Deduplication

The simplest and most efficient approach: compute a hash of the captured content and skip sending to the LLM if the hash matches the previous capture.

```go
import "hash/fnv"

// fnv1aHash computes a non-cryptographic hash suitable for change detection.
// FNV-1a is fast (no table lookups) and has good avalanche behavior for strings.
func fnv1aHash(s string) uint64 {
    h := fnv.New64a()
    h.Write([]byte(s))
    return h.Sum64()
}
```

**Why FNV-1a over CRC32 or MD5:**
- FNV-1a: zero allocation, ~1 ns/byte, handles short strings well
- CRC32: better for large blocks (hardware acceleration), higher collision rate for short strings
- MD5/SHA: cryptographic overhead not needed here; 10-100x slower

For terminal content (typically 80 columns × 24 rows = ~2 KB), FNV-1a is ideal.

### State Tracking in the CaptureManager

```go
type PollState struct {
    LastHash    uint64
    LastContent string
    LastSeen    time.Time
    MissedPolls int  // how many consecutive polls found no change
}

type CaptureManager struct {
    // ...
    state map[PaneID]*PollState
    mu    sync.RWMutex
}

func (m *CaptureManager) Poll(ctx context.Context) []CaptureResult {
    // ... discover panes ...
    var changed []CaptureResult

    for _, pane := range allPanes {
        result, err := adapter.Capture(ctx, pane)
        if err != nil {
            continue
        }
        m.mu.Lock()
        prev, exists := m.state[pane.ID]
        if !exists || prev.LastHash != result.Hash {
            changed = append(changed, result)
            m.state[pane.ID] = &PollState{
                LastHash:    result.Hash,
                LastContent: result.Content,
                LastSeen:    time.Now(),
                MissedPolls: 0,
            }
        } else {
            m.state[pane.ID].MissedPolls++
        }
        m.mu.Unlock()
    }
    return changed
}
```

### Diffing vs. Hashing

For an LLM context use case, send the full content (not a diff) when the hash changes. Diffs add implementation complexity and LLMs work better with full context anyway.

**Exception:** If you track scrollback history (large captures), send only the last N lines even on a hash change. Use `strings.Split` and take the tail:

```go
func TailLines(s string, n int) string {
    lines := strings.Split(s, "\n")
    if len(lines) <= n {
        return s
    }
    return strings.Join(lines[len(lines)-n:], "\n")
}
```

### Disappearing Panes

When a pane is closed (tmux session ends, terminal window closes), it will no longer appear in `Discover()`. Track pane presence and emit a "pane closed" event to purge it from state:

```go
func (m *CaptureManager) reconcilePanes(discovered []PaneInfo) (new, removed []PaneID) {
    m.mu.Lock()
    defer m.mu.Unlock()

    seenNow := make(map[PaneID]bool)
    for _, p := range discovered {
        seenNow[p.ID] = true
        if _, exists := m.state[p.ID]; !exists {
            new = append(new, p.ID)
        }
    }
    for id := range m.state {
        if !seenNow[id] {
            removed = append(removed, id)
            delete(m.state, id)
        }
    }
    return
}
```

---

## Implementation Priorities and Ordering

Based on research, here is the recommended build order:

### Phase 1: tmux Adapter (highest confidence, fastest to ship)
- Implement `ListAllPanes` with `list-panes -a -F`
- Implement `CapturePaneByID` with `capture-pane -p -t <id>`
- Implement FNV-1a deduplication
- Define the `TerminalAdapter` interface
- **Estimated complexity:** Low. All pure subprocess calls.

### Phase 2: Core Polling Loop
- 500ms ticker with `time.NewTicker`
- Concurrent capture with goroutines + semaphore
- Content deduplication state tracking
- Channel-based output to LLM sender
- **Estimated complexity:** Low-Medium.

### Phase 3: GNOME Terminal Adapter
- `godbus/dbus/v5` connection to accessibility bus
- `GetAddress` flow to connect to accessibility bus
- `FindTerminalObjects` by role=59
- `GetText(0, -1)` for full content
- Graceful degradation if AT-SPI2 is unavailable
- **Estimated complexity:** Medium. D-Bus API is clear; reliability is the uncertainty.

### Phase 4: Konsole Adapter
- Enumerate `org.kde.konsole-*` services on session bus
- Use `foregroundProcessName` to confirm active pane
- Attempt AT-SPI2 text read (experimental)
- Fall back to a "Konsole is active" signal without content if AT-SPI2 fails
- **Estimated complexity:** Medium-High. Significant uncertainty around Qt AT-SPI2 text access.

---

## Key Dependencies

```go
// go.mod additions
require (
    github.com/godbus/dbus/v5 v5.1.0  // D-Bus for AT-SPI2
    // NO CGO dependency needed
    // hash/fnv is stdlib
    // os/exec is stdlib
)
```

No CGO, no C libraries, no special kernel features required for the tmux adapter. AT-SPI2 via godbus/dbus is pure Go.

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|------------|--------|
| tmux capture-pane flags | HIGH | Verified against man page and source |
| tmux Go subprocess implementation | HIGH | Pattern well-established; exec.Command is straightforward |
| AT-SPI2 D-Bus architecture | HIGH | Verified against official GNOME docs |
| godbus/dbus/v5 API | HIGH | Official package, actively maintained |
| GNOME Terminal AT-SPI2 tree structure | MEDIUM | Inferred from accerciser patterns; not end-to-end tested |
| GNOME Terminal GTK4 text accessibility | MEDIUM | Documented in GTK 4.14 blog; deployment variable |
| Konsole D-Bus interface (control methods) | MEDIUM | Verified via KDE docs and scripting guides |
| Konsole AT-SPI2 text access | LOW | Qt AT-SPI bridge exists but terminal widget text interface unconfirmed |
| tmux control mode as change notification | MEDIUM | Protocol documented; complexity makes it not recommended |
| AT-SPI2 text-changed events reliability | LOW | VTE has known event emission bugs (issue #88) |

---

## Sources

- [tmux man page](https://www.man7.org/linux/man-pages/man1/tmux.1.html)
- [tmux Control Mode wiki](https://github.com/tmux/tmux/wiki/Control-Mode)
- [jubnzv/go-tmux pane.go](https://github.com/jubnzv/go-tmux/blob/master/pane.go)
- [godbus/dbus/v5 package docs](https://pkg.go.dev/github.com/godbus/dbus/v5)
- [at-spi2-core bus README](https://github.com/GNOME/at-spi2-core/blob/main/bus/README.md)
- [org.a11y.atspi.Accessible interface](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Accessible.html)
- [org.a11y.atspi.Cache interface](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Cache.html)
- [org.a11y.atspi.Registry interface](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/doc-org.a11y.atspi.Registry.html)
- [AT-SPI2 Text.xml (raw)](https://raw.githubusercontent.com/GNOME/at-spi2-core/main/xml/Text.xml)
- [AT-SPI2 Accessible.xml (raw)](https://raw.githubusercontent.com/GNOME/at-spi2-core/main/xml/Accessible.xml)
- [atspi-constants.h (role enum)](https://github.com/GNOME/at-spi2-core/blob/main/atspi/atspi-constants.h)
- [libatspi2 docs](https://docs.gtk.org/atspi2/)
- [GTK 4.14 Accessibility improvements blog](https://blog.gtk.org/2024/03/08/accessibility-improvements-in-gtk-4-14/)
- [GNOME Terminal automation via AT-SPI2](https://modehnal.github.io/)
- [at-spi2-core toolkit implementations](https://gnome.pages.gitlab.gnome.org/at-spi2-core/devel-docs/toolkits.html)
- [Scripting KDE Konsole (2024)](https://byabbe.se/2024/10/21/scripting-kde-konsole)
- [Start and control Konsole with D-Bus](https://www.linuxjournal.com/content/start-and-control-konsole-dbus)
- [gsettings toolkit-accessibility discussion](https://discourse.gnome.org/t/gsettings-set-org-gnome-desktop-interface-toolkit-accessibility-true-fails/15440)
- [KDE Qt AT-SPI wiki](https://community.kde.org/Accessibility/qt-atspi)
- [VTE missing text insertion event issue](https://gitlab.gnome.org/GNOME/vte/-/issues/88)
- [hash/fnv package](https://pkg.go.dev/hash/fnv)
