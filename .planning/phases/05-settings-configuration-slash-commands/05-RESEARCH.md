# Phase 5: Settings, Configuration & Slash Commands - Research

**Researched:** 2026-03-29
**Domain:** Wails v2 + React settings dialog, 99designs/keyring, Viper config expansion, slash command routing, clipboard auto-clear goroutine
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Settings opens as a tabbed modal overlay triggered by gear icon in `StatusBar.tsx` (currently disabled). Modal overlays `ThreeColumnLayout` — no new layout panels needed.
- **D-02:** Five tabs: LLM Config / Prompts / Terminals / Hotkeys / Appearance. Each tab has its own Save button — changes write to Viper/keychain only on explicit save. User can cancel without affecting the current session.
- **D-03:** LLM Config tab connection test result appears inline below the Test button — spinner while testing, then green checkmark "Connected" or red "x" error message. No toast or second modal.
- **D-04:** Provider and model stored in `~/.pairadmin/config.yaml` via Viper. On startup: Viper config takes priority; env vars (`PAIRADMIN_PROVIDER`, `PAIRADMIN_MODEL`) are the fallback if no config file exists. Existing env-var users keep working without change until they open Settings.
- **D-05:** API keys use `99designs/keyring` with service = "pairadmin", key = provider name (e.g., "openai", "anthropic", "openrouter", "lmstudio"). One keychain entry per provider. Keys are never written to `~/.pairadmin/config.yaml`.
- **D-06:** API key input shows `•••••••• (stored)` as placeholder when a key exists in the keychain. User clears the field and types a new key to replace. Keychain write only happens on Save. Field is blank (not placeholder) when no key is stored.
- **D-07:** ChatPane has a frontend slash command router dispatching by prefix: frontend-only (`/clear`, `/theme`, `/help`); Go backend call (`/model`, `/context`, `/refresh`, `/export`, `/rename`, `/filter`).
- **D-08:** All slash command output appears as a system message inline in chat — italic, muted styling, consistent with Phase 4's `/filter` output pattern. `/help` renders a formatted list as a system message.
- **D-09:** After a successful clipboard write in `CommandService.CopyToClipboard`, a goroutine sleeps for the configured interval and then clears the clipboard (writes empty string via same wl-clipboard/xclip path). Timer runs server-side.
- **D-10:** Auto-clear interval configurable via Terminals tab in Settings (grouped with AT-SPI2 polling interval). Default: 60 seconds. Stored in `~/.pairadmin/config.yaml` via Viper.

### Claude's Discretion

- Tab component choice for the Settings modal (shadcn Tabs vs custom, `@radix-ui/react-tabs` vs `@base-ui/react`)
- Dialog component choice (shadcn Dialog vs `@base-ui/react` Popover/Dialog)
- `AppConfig` struct field naming for new settings (provider, model, polling interval, clipboard clear interval)
- Hotkeys tab implementation approach (capture key combination on focus, store as string)
- `/export` file path choice (e.g., `~/pairadmin-export-YYYY-MM-DD.json`)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CFG-01 | LLM provider and model configurable via settings dialog (LLM Config tab) | SettingsService Go backend + React dialog with @base-ui/react Dialog + Tabs |
| CFG-02 | API keys stored in OS keychain using `99designs/keyring` (not plaintext config files) | 99designs/keyring v1.2.2, SecretServiceBackend + FileBackend fallback; libsecret-1-dev needed at compile time |
| CFG-03 | Connection to configured provider can be tested from settings dialog | TestConnection RPC on SettingsService; inline status below button via React state |
| CFG-04 | User can provide custom system prompt extension via settings Prompts tab | AppConfig.CustomPrompt string field; LLMService.SendMessage reads it when building messages |
| CFG-05 | AT-SPI2 polling interval configurable via settings Terminals tab | AppConfig.ATSPIPollingIntervalMs int field; CaptureManager reads from config |
| CFG-06 | Global hotkeys configurable (copy last command, focus PairAdmin window) | AppConfig.HotkeysCopyLastCommand / HotkeysFocusWindow string fields; frontend keyboard event listener |
| CFG-07 | Dark and light themes available; dark is default | useTheme() hook already exists in theme-provider.tsx; /theme slash command calls setTheme |
| CFG-08 | Settings persisted to `~/.pairadmin/config.yaml` via Viper (no secrets in this file) | Extend existing AppConfig struct + SaveAppConfig in services/config/config.go |
| SLASH-01 | `/model <provider:model>` switches active LLM provider/model | Go: SettingsService.SetModel; frontend router dispatches via dynamic import |
| SLASH-02 | `/context <lines>` sets terminal context window size | Go: SettingsService.SetContextLines; stored in AppConfig; LLMService reads it |
| SLASH-03 | `/refresh` forces re-capture of terminal content | Go: CaptureManager.ForceRefresh (or frontend trigger); re-emits terminal:tabs event |
| SLASH-04 | `/filter add|list|remove` manages filter patterns | Already implemented in Phase 4 as LLMService.FilterCommand — router already exists in ChatPane |
| SLASH-05 | `/export json|txt` exports current session chat history | Go: SettingsService.ExportChat(tabId, format) writes file; returns path as system message |
| SLASH-06 | `/rename <label>` renames current terminal tab | Go: CaptureManager or SettingsService.RenameTab(tabId, label); emits terminal:tabs event |
| SLASH-07 | `/theme dark|light` switches color scheme | Frontend-only: calls useTheme().setTheme; no Go call needed |
| SLASH-08 | `/help` displays available commands | Frontend-only: addSystemMessage with formatted help text |
| CLIP-03 | Clipboard contents copied by PairAdmin auto-cleared after 60 seconds (configurable) | Goroutine in CommandService.CopyToClipboard; interval from AppConfig; wl-clipboard clear via empty string |

</phase_requirements>

---

## Summary

Phase 5 delivers three loosely-coupled subsystems that share a common config layer. The Go backend gains a new `SettingsService` (Wails-bound), an expanded `AppConfig` struct, and a `KeychainService` wrapper around `99designs/keyring`. The React frontend gains a `SettingsDialog` component using `@base-ui/react` Dialog and Tabs (already installed at v1.3.0) plus a full slash command router in `ChatPane`. The clipboard auto-clear goroutine in `CommandService` is the smallest and most isolated piece.

The primary complexity is the keychain integration. `99designs/keyring` uses CGO (`gsterjov/go-libsecret`) for the SecretServiceBackend, requiring `libsecret-1-dev` headers at compile time. These headers are not currently installed (only `libsecret-1-0` runtime is present). Either install them with `sudo apt install libsecret-1-dev` or configure `AllowedBackends` to use `FileBackend` as the exclusive backend for test isolation. The production path uses `SecretServiceBackend`; gnome-keyring is already running on this machine.

The theme system is already fully implemented via `ThemeProvider`/`useTheme()` in `frontend/src/theme/theme-provider.tsx`. The `/theme` slash command and Appearance tab toggle simply call `setTheme("dark"|"light")` — no new infrastructure needed. Similarly, `addSystemMessage` is already ready for all slash command output.

**Primary recommendation:** Implement in four task groups — (1) expand `AppConfig` + `SettingsService` Go backend including keychain, (2) settings dialog React UI with all 5 tabs, (3) slash command router expansion in ChatPane, (4) clipboard auto-clear goroutine.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `99designs/keyring` | v1.2.2 (latest) | OS keychain read/write | Project decision (D-05); only keyring library that handles headless Linux via FileBackend fallback |
| `@base-ui/react` (Dialog) | v1.3.0 (installed) | Modal dialog for settings | Already installed; project uses base-ui-nova style; Dialog + Tabs available |
| `@base-ui/react` (Tabs) | v1.3.0 (installed) | Tab navigation within settings modal | Same package, already installed |
| `github.com/spf13/viper` | v1.21.0 (in go.mod) | Config file persistence | Already in use in `services/config/config.go` |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `lucide-react` | v1.7.0 (installed) | Settings/check/x icons in dialog | Already used in StatusBar |
| `zustand` + `immer` | v5.0.12 + v11.1.4 | `settingsStore` for live model display in StatusBar | Follow existing chatStore/terminalStore pattern |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `@base-ui/react` Dialog/Tabs | shadcn Dialog/Tabs (Radix) | Radix is NOT in use; project is on base-ui-nova style; adding Radix would create duplicate peer dep tree |
| `99designs/keyring` SecretService | `zalando/go-keyring` | Explicitly rejected in project history — fails in headless Linux |
| `99designs/keyring` SecretService | FileBackend exclusively | Valid for headless servers; but gnome-keyring IS running here; use SecretService with FileBackend as fallback |

**Installation:**
```bash
# Go: add keyring dependency
cd /home/sblanken/working/ppa2 && go get github.com/99designs/keyring@v1.2.2

# System: install CGO headers for go-libsecret (SecretServiceBackend)
sudo apt install libsecret-1-dev

# Frontend: no new npm installs — @base-ui/react Dialog and Tabs already installed at v1.3.0
```

**Version verification (performed 2026-03-29):**
- `99designs/keyring` latest: v1.2.2 (confirmed via proxy.golang.org)
- `@base-ui/react` installed: v1.3.0 (confirmed via node_modules/package.json)
- `@base-ui/react` exports: `Dialog`, `Tabs`, `Select` confirmed present in installed package

---

## Architecture Patterns

### Recommended Project Structure

New files this phase:
```
services/
├── settings_service.go       # New: Wails-bound SettingsService
├── settings_service_test.go  # New: tests for SettingsService
├── keychain/
│   └── keychain.go           # New: thin wrapper around 99designs/keyring
│   └── keychain_test.go      # New: injectable fn fields for test isolation
├── config/
│   └── config.go             # MODIFIED: expand AppConfig struct
│   └── config_test.go        # MODIFIED: add tests for new fields

frontend/src/
├── components/
│   └── settings/
│       ├── SettingsDialog.tsx         # New: modal root with Dialog
│       ├── LLMConfigTab.tsx           # New: provider/model/key/test
│       ├── PromptsTab.tsx             # New: built-in display + custom extension
│       ├── TerminalsTab.tsx           # New: polling interval + clipboard clear
│       ├── HotkeysTab.tsx             # New: hotkey capture inputs
│       └── AppearanceTab.tsx          # New: theme toggle + font size
├── stores/
│   └── settingsStore.ts               # New: active provider:model for StatusBar

frontend/wailsjs/go/services/
├── SettingsService.js                  # New stub: committed with .gitignore exception
```

Modified files:
```
main.go                                 # Add SettingsService to Bind list
services/commands.go                    # Add auto-clear goroutine to CopyToClipboard
services/llm_service.go                 # Update startup to prefer Viper over env vars
frontend/src/components/layout/
├── StatusBar.tsx                       # Wire gear onClick; display active model from settingsStore
└── ThreeColumnLayout.tsx              # Mount <SettingsDialog>
frontend/src/components/chat/
└── ChatPane.tsx                        # Expand slash command router
```

### Pattern 1: SettingsService — Wails-Bound Config Backend

**What:** New `services/SettingsService` struct bound in `main.go` that exposes RPC methods for reading/writing settings and testing provider connections. Follows exact same pattern as `LLMService` and `CommandService`.

**When to use:** Any setting that requires persistence (Viper) or keychain access.

```go
// Source: pattern from services/llm_service.go
type SettingsService struct {
    ctx          context.Context
    keychainOpen func(cfg keyring.Config) (keyring.Keyring, error)
}

func NewSettingsService() *SettingsService {
    return &SettingsService{
        keychainOpen: keyring.Open, // injectable for tests
    }
}

func (s *SettingsService) Startup(ctx context.Context) {
    s.ctx = ctx
}

// GetSettings reads AppConfig from Viper; returns it as a JSON-serializable struct.
func (s *SettingsService) GetSettings() (*config.AppConfig, error) { ... }

// SaveSettings writes non-secret settings to Viper config file.
func (s *SettingsService) SaveSettings(cfg *config.AppConfig) error { ... }

// GetAPIKey returns masked status: returns "stored" if key exists in keychain, "" if not.
// Never returns the actual key value to the frontend.
func (s *SettingsService) GetAPIKeyStatus(provider string) (string, error) { ... }

// SaveAPIKey writes key to keychain (service="pairadmin", key=provider).
func (s *SettingsService) SaveAPIKey(provider, key string) error { ... }

// TestConnection attempts to create a provider and make a minimal API call.
func (s *SettingsService) TestConnection(provider, model string) (string, error) { ... }

// SetModel switches the active provider:model at runtime.
func (s *SettingsService) SetModel(providerModel string) error { ... }

// SetContextLines sets terminal context window size.
func (s *SettingsService) SetContextLines(lines int) error { ... }

// ExportChat exports tab chat history to ~/pairadmin-export-YYYY-MM-DD.json
func (s *SettingsService) ExportChat(tabId, format string, messages []ExportMessage) (string, error) { ... }

// RenameTab renames a terminal tab label.
func (s *SettingsService) RenameTab(tabId, label string) error { ... }
```

### Pattern 2: AppConfig Struct Expansion

**What:** Add new fields to `AppConfig` in `services/config/config.go`. Use the same `mapstructure` + `yaml` tags pattern. Update `SaveAppConfig` to serialize all fields.

**When to use:** Every non-secret setting this phase adds.

```go
// Source: services/config/config.go — expand existing struct
type AppConfig struct {
    CustomPatterns      []CustomPattern `mapstructure:"custom_patterns" yaml:"custom_patterns"`
    Provider            string          `mapstructure:"provider" yaml:"provider"`
    Model               string          `mapstructure:"model" yaml:"model"`
    CustomPrompt        string          `mapstructure:"custom_prompt" yaml:"custom_prompt"`
    ATSPIPollingMs      int             `mapstructure:"atspi_polling_ms" yaml:"atspi_polling_ms"`
    ClipboardClearSecs  int             `mapstructure:"clipboard_clear_secs" yaml:"clipboard_clear_secs"`
    HotkeyCopyLast      string          `mapstructure:"hotkey_copy_last" yaml:"hotkey_copy_last"`
    HotkeyFocusWindow   string          `mapstructure:"hotkey_focus_window" yaml:"hotkey_focus_window"`
    Theme               string          `mapstructure:"theme" yaml:"theme"`
    FontSize            int             `mapstructure:"font_size" yaml:"font_size"`
    ContextLines        int             `mapstructure:"context_lines" yaml:"context_lines"`
}
```

**Critical:** `SaveAppConfig` currently uses `v.Set()` per field, then `WriteConfigAs`. When expanding the struct, update `SaveAppConfig` to iterate all new fields. The simpler approach is marshaling `cfg` to a map and calling `v.MergeConfigMap` before write, but the existing direct-set pattern works and should be extended consistently.

### Pattern 3: Keychain Integration with Injectable Function Field

**What:** A thin `services/keychain` package wrapping `99designs/keyring`. Uses injectable function fields (same as `execCommand` on `TmuxAdapter`) so tests can replace `keyring.Open` without a live secret store.

**When to use:** Any read/write of API keys.

```go
// Source: pattern from services/capture/tmux_adapter.go (injectable field)
// services/keychain/keychain.go
package keychain

import (
    "github.com/99designs/keyring"
)

const ServiceName = "pairadmin"

type Client struct {
    open func(cfg keyring.Config) (keyring.Keyring, error)
}

func New() *Client {
    return &Client{open: keyring.Open}
}

func (c *Client) openKeyring() (keyring.Keyring, error) {
    return c.open(keyring.Config{
        ServiceName:     ServiceName,
        AllowedBackends: []keyring.BackendType{
            keyring.SecretServiceBackend,
            keyring.FileBackend,          // fallback for headless
        },
        FileDir:          "~/.pairadmin/keyring",
        FilePasswordFunc: keyring.FixedStringPrompt("pairadmin"),
    })
}

func (c *Client) Get(provider string) (string, error) {
    kr, err := c.openKeyring()
    if err != nil { return "", err }
    item, err := kr.Get(provider)
    if err == keyring.ErrKeyNotFound { return "", nil }
    return string(item.Data), err
}

func (c *Client) Set(provider, key string) error {
    kr, err := c.openKeyring()
    if err != nil { return err }
    return kr.Set(keyring.Item{Key: provider, Data: []byte(key)})
}

func (c *Client) Remove(provider string) error {
    kr, err := c.openKeyring()
    if err != nil { return err }
    return kr.Remove(provider)
}
```

**Test isolation:** In `keychain_test.go`, create `Client{open: func(...) (keyring.Keyring, error) { return &fakeKeyring{...}, nil }}` — same injectable-field pattern as `TmuxAdapter.execCommand`.

### Pattern 4: @base-ui/react Dialog and Tabs Usage

**What:** `@base-ui/react` v1.3.0 ships `Dialog.*` and `Tabs.*` namespaces following the same Radix-inspired compound component pattern seen in the existing `tooltip.tsx`.

```tsx
// Source: @base-ui/react installed package exports (verified in node_modules)
// Pattern matches existing tooltip.tsx wrapping approach
import { Dialog } from "@base-ui/react/dialog";
import { Tabs } from "@base-ui/react/tabs";

// Dialog usage
<Dialog.Root open={open} onOpenChange={setOpen}>
  <Dialog.Portal>
    <Dialog.Backdrop className="fixed inset-0 bg-black/50" />
    <Dialog.Popup className="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 ...">
      <Dialog.Title>Settings</Dialog.Title>
      {/* tab content */}
      <Dialog.Close />
    </Dialog.Popup>
  </Dialog.Portal>
</Dialog.Root>

// Tabs usage (within Dialog.Popup)
<Tabs.Root defaultValue="llm-config">
  <Tabs.List>
    <Tabs.Tab value="llm-config">LLM Config</Tabs.Tab>
    <Tabs.Tab value="prompts">Prompts</Tabs.Tab>
    <Tabs.Tab value="terminals">Terminals</Tabs.Tab>
    <Tabs.Tab value="hotkeys">Hotkeys</Tabs.Tab>
    <Tabs.Tab value="appearance">Appearance</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="llm-config"><LLMConfigTab /></Tabs.Panel>
  {/* ... */}
</Tabs.Root>
```

**Critical import path:** Use `@base-ui/react/dialog` and `@base-ui/react/tabs` (subdirectory imports), NOT `@base-ui/react` (no direct named export). Matches `@base-ui/react/tooltip` pattern in `tooltip.tsx`.

### Pattern 5: Slash Command Router in ChatPane

**What:** Expand `handleSend` in `ChatPane.tsx` with a command parser that dispatches before the LLM call. Frontend-only commands return immediately; Go-side commands use the same dynamic import pattern as the existing `/filter` handler.

```tsx
// Source: pattern from frontend/src/components/chat/ChatPane.tsx existing /filter block
const handleSend = async (text: string) => {
  useChatStore.getState().addUserMessage(activeTabId, text);

  const trimmed = text.trim();

  if (trimmed === "/clear") { /* existing */ return; }

  // Frontend-only commands
  if (trimmed.startsWith("/theme ")) {
    const t = trimmed.slice(7).trim();
    if (t === "dark" || t === "light") setTheme(t); // useTheme() hook
    useChatStore.getState().addSystemMessage(activeTabId, `Theme set to ${t}`);
    return;
  }
  if (trimmed === "/help") {
    useChatStore.getState().addSystemMessage(activeTabId, HELP_TEXT);
    return;
  }

  // Go backend commands
  if (trimmed.startsWith("/filter")) { /* existing */ return; }
  if (trimmed.startsWith("/model ") || trimmed.startsWith("/context ") ||
      trimmed === "/refresh" || trimmed.startsWith("/export") ||
      trimmed.startsWith("/rename ")) {
    import(/* @vite-ignore */ "../../wailsjs/go/services/SettingsService")
      .then(({ SetModel, SetContextLines, ForceRefresh, ExportChat, RenameTab }) => {
        // dispatch to correct RPC based on command
      })
      .then((response: string) => {
        useChatStore.getState().addSystemMessage(activeTabId, response);
      })
      .catch((err: Error) => {
        useChatStore.getState().addSystemMessage(activeTabId, `Error: ${err?.message ?? "Unknown"}`);
      });
    return;
  }
  // ... LLM call as before
};
```

### Pattern 6: Clipboard Auto-Clear Goroutine

**What:** After `CopyToClipboard` writes to clipboard, launch a goroutine that sleeps for the configured interval and writes an empty string.

```go
// Source: services/commands.go — insertion after clipboard write succeeds
func (c *CommandService) CopyToClipboard(text string) error {
    var err error
    if isWayland() {
        err = copyViaWlCopy(text)
    } else {
        runtime.ClipboardSetText(c.ctx, text)
    }
    if err != nil { return err }

    // Start auto-clear goroutine
    cfg, _ := config.LoadAppConfig()
    secs := cfg.ClipboardClearSecs
    if secs <= 0 { secs = 60 }
    go func() {
        time.Sleep(time.Duration(secs) * time.Second)
        if isWayland() {
            _ = copyViaWlCopy("")
        } else {
            runtime.ClipboardSetText(c.ctx, "")
        }
    }()
    return nil
}
```

**Test isolation:** Inject `sleepFn func(time.Duration)` field on `CommandService` (consistent with injectable field pattern from Phases 1-4) to allow tests to fast-forward the sleep without `time.Sleep` in test runs.

### Pattern 7: StatusBar Live Model Display

**What:** Add a `settingsStore` (Zustand) holding `activeModel string`. StatusBar reads from it. After `/model` or Settings Save, `SettingsService.SetModel` emits a Wails event `"settings:model-changed"` and the frontend `useSettingsEvents` hook updates the store.

**Why a separate store:** Avoids coupling StatusBar to terminal store; consistent with `chatStore`/`terminalStore` separation pattern.

```ts
// New: frontend/src/stores/settingsStore.ts
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

interface SettingsState {
  activeModel: string;
  setActiveModel: (model: string) => void;
}

export const useSettingsStore = create<SettingsState>()(
  immer((set) => ({
    activeModel: "",
    setActiveModel: (model) => set((s) => { s.activeModel = model; }),
  }))
);
```

### Anti-Patterns to Avoid

- **Writing API keys to Viper/config.yaml:** Explicitly forbidden per D-05. `SaveAppConfig` must not have key fields. The keychain client handles keys exclusively.
- **Calling `keyring.Open` directly in SettingsService:** Always go through the `keychain.Client` wrapper — enables test injection.
- **Frontend fetching the actual API key value:** `GetAPIKeyStatus` returns "stored" or "" — never the plaintext key. Frontend only writes keys, never reads them back for display.
- **Single global `viper.SetDefault` + `ReadInConfig`:** The existing `LoadAppConfig` creates a fresh `viper.New()` instance each call. This is the correct pattern — it avoids state leakage between tests. Maintain this pattern when adding fields.
- **`@base-ui/react` component imports from root package:** The package uses subdirectory exports. `import { Dialog } from "@base-ui/react"` will fail. Use `import { Dialog } from "@base-ui/react/dialog"`.
- **Adding Dialog/Tabs wailsjs stubs:** No Go bindings needed for frontend-only components; stubs are only needed for Go RPC calls. Add `SettingsService.js` stub only.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| OS keychain R/W | Custom secret file encryption | `99designs/keyring` | Handles SecretService, Kwallet, File fallback; edge cases in key locking, daemon unavailability handled |
| Config persistence | Custom YAML marshal/unmarshal | Viper (already in use) | Already handles file creation, defaults, mapstructure tagging |
| Modal/overlay | CSS `position: fixed` + state management | `@base-ui/react` Dialog | Focus trap, keyboard close (Escape), aria-modal, backdrop click — all handled |
| Tab navigation | Button array + CSS show/hide | `@base-ui/react` Tabs | Keyboard navigation, aria-selected, panel association |
| Theme persistence | `document.classList` + localStorage manually | Existing `ThemeProvider`/`useTheme()` | Already implemented; `/theme` slash command just calls `setTheme()` |

**Key insight:** The keychain is the only truly new dependency. All UI primitives are already in the installed `@base-ui/react` package. The config infrastructure is already in `services/config`.

---

## Common Pitfalls

### Pitfall 1: libsecret-1-dev Missing at Compile Time
**What goes wrong:** `go get github.com/99designs/keyring` succeeds, but `wails build` or `go build` fails with `cgo: C compiler "gcc" failed` or `pkg-config: libsecret-1` not found.
**Why it happens:** `gsterjov/go-libsecret` (SecretServiceBackend dependency) uses CGO and requires `libsecret-1-dev` headers. Only `libsecret-1-0` (runtime) is installed; dev headers are absent.
**How to avoid:** `sudo apt install libsecret-1-dev` before `go get`. Verify with `pkg-config --cflags libsecret-1`.
**Warning signs:** Build error mentioning `gsterjov/go-libsecret` or `cgo` failure on `libsecret`.

### Pitfall 2: SaveAppConfig Overwrites Existing Fields
**What goes wrong:** `SaveAppConfig` sets only the fields it knows about, wiping out fields from future phases or user manual edits in YAML.
**Why it happens:** Current implementation uses `v.Set()` per known field then `WriteConfigAs` — any field not explicitly set is absent from output.
**How to avoid:** Before writing, call `v.ReadInConfig()` to load existing file, then `v.Set()` for each field being updated, then `WriteConfigAs`. This merges rather than replaces.
**Warning signs:** Round-trip test that saves subset of fields and reloads fails because other fields disappeared.

### Pitfall 3: @base-ui/react Import Path Errors
**What goes wrong:** `import { Dialog } from "@base-ui/react"` compiles but throws at runtime, or TypeScript errors "has no exported member 'Dialog'".
**Why it happens:** `@base-ui/react` uses subpath exports — each component is in its own subpath (`@base-ui/react/dialog`, `@base-ui/react/tabs`).
**How to avoid:** Always import from subpaths: `import { Dialog } from "@base-ui/react/dialog"`. Matches pattern of existing `tooltip.tsx`: `import { Tooltip as TooltipPrimitive } from "@base-ui/react/tooltip"`.
**Warning signs:** TypeScript error "Module '@base-ui/react' has no exported member" or runtime crash in Dialog.Root.

### Pitfall 4: Wails Stub for SettingsService Not Committed
**What goes wrong:** Vitest tests for `ChatPane` (which imports `SettingsService` via dynamic import) fail with "Cannot resolve module" even with `vi.mock`.
**Why it happens:** `frontend/wailsjs/` is gitignored; the `SettingsService.js` stub needs a `.gitignore` exception like `CaptureManager.js`.
**How to avoid:** Add `!frontend/wailsjs/go/services/SettingsService.js` to `.gitignore` and commit the stub file.
**Warning signs:** Vitest errors about `../../wailsjs/go/services/SettingsService` during test runs.

### Pitfall 5: Clipboard Clear Race with Concurrent Copies
**What goes wrong:** User copies command A, then 30 seconds later copies command B. The goroutine from copy A fires at t=60s and clears command B's content.
**Why it happens:** Each `CopyToClipboard` call launches a new goroutine; there is no cancellation of previous timers.
**How to avoid:** Use a `sync.Mutex`-protected `*time.Timer` on `CommandService`. Reset the timer on each new copy. Cancel previous timer before starting new one.
**Warning signs:** Intermittent reports of clipboard clearing "too early" or wiping a freshly copied command.

### Pitfall 6: Viper Env-Var Priority Inversion
**What goes wrong:** User sets `PAIRADMIN_PROVIDER=openai` in env, opens Settings, saves "anthropic" — on next restart, env var overrides the Viper-saved value.
**Why it happens:** `viper.AutomaticEnv()` + `BindEnv` makes env vars take higher priority than config file by default.
**How to avoid:** Per D-04, the logic is: if Viper config file has `provider` set (non-empty), use it; else fall back to env vars. Implement this in `LoadConfig()` or `LLMService` startup — read Viper first, then check env vars only if Viper value is empty. Do NOT call `viper.AutomaticEnv()` in the config package.
**Warning signs:** User sees env-var provider in Status bar even after explicitly saving a different provider in Settings.

### Pitfall 7: TooltipTrigger asChild Pattern Does Not Apply to Dialog
**What goes wrong:** Attempting to wrap the gear button with `Dialog.Trigger asChild` causes button-in-button nesting.
**Why it happens:** `@base-ui/react` v1.3.0 uses `render` prop rather than `asChild` for custom rendering in some components. The Dialog.Trigger renders its own element.
**How to avoid:** Per existing STATE.md decision on TooltipTrigger: "pass className/onClick directly on the trigger element." Check @base-ui/react Dialog.Trigger API — either use `render` prop or handle `open` state manually with `Dialog.Root open={open}` controlled mode and wire `StatusBar`'s gear button onClick to set open state.
**Warning signs:** React warning "cannot nest button inside button" or keyboard events not propagating.

---

## Code Examples

### Keychain Get/Set (verified API from pkg.go.dev)

```go
// Source: pkg.go.dev/github.com/99designs/keyring v1.2.2
kr, err := keyring.Open(keyring.Config{
    ServiceName:     "pairadmin",
    AllowedBackends: []keyring.BackendType{
        keyring.SecretServiceBackend,
        keyring.FileBackend,
    },
    FileDir:          "~/.pairadmin/keyring",
    FilePasswordFunc: keyring.FixedStringPrompt("pairadmin"),
})

// Set (write API key)
err = kr.Set(keyring.Item{Key: "openai", Data: []byte("sk-...")})

// Get (read API key)
item, err := kr.Get("openai")
if err == keyring.ErrKeyNotFound {
    // no key stored
}
key := string(item.Data)

// Remove
err = kr.Remove("openai")
```

### Dialog + Tabs Component Structure

```tsx
// Source: @base-ui/react v1.3.0 installed package, mirroring tooltip.tsx pattern
import { Dialog } from "@base-ui/react/dialog";
import { Tabs } from "@base-ui/react/tabs";

interface SettingsDialogProps {
  open: boolean;
  onClose: () => void;
}

export function SettingsDialog({ open, onClose }: SettingsDialogProps) {
  return (
    <Dialog.Root open={open} onOpenChange={(o) => { if (!o) onClose(); }}>
      <Dialog.Portal>
        <Dialog.Backdrop className="fixed inset-0 z-40 bg-black/60" />
        <Dialog.Popup className="fixed left-1/2 top-1/2 z-50 w-[640px] -translate-x-1/2 -translate-y-1/2 rounded-lg bg-zinc-900 border border-zinc-700 shadow-xl">
          <Dialog.Title className="px-6 py-4 text-sm font-semibold text-zinc-100 border-b border-zinc-800">
            Settings
          </Dialog.Title>
          <Tabs.Root defaultValue="llm-config" className="flex flex-col">
            <Tabs.List className="flex border-b border-zinc-800 px-4">
              <Tabs.Tab value="llm-config" className="px-3 py-2 text-xs text-zinc-400 data-[selected]:text-zinc-100 data-[selected]:border-b-2 data-[selected]:border-zinc-400">
                LLM Config
              </Tabs.Tab>
              {/* ... other tabs */}
            </Tabs.List>
            <Tabs.Panel value="llm-config" className="p-6">
              <LLMConfigTab onClose={onClose} />
            </Tabs.Panel>
          </Tabs.Root>
        </Dialog.Popup>
      </Dialog.Portal>
    </Dialog.Root>
  );
}
```

### Startup Config Priority (D-04)

```go
// In LLMService startup or NewLLMService — prefer Viper over env vars
func LoadConfigWithViper() Config {
    appCfg, _ := config.LoadAppConfig()
    envCfg := LoadConfig() // existing env-var reader

    // Viper wins if set; env var is fallback
    provider := appCfg.Provider
    if provider == "" { provider = envCfg.Provider }
    model := appCfg.Model
    if model == "" { model = envCfg.Model }

    return Config{
        Provider:      provider,
        Model:         model,
        OpenAIKey:     envCfg.OpenAIKey,     // keys always from env for now; keychain replaces this
        AnthropicKey:  envCfg.AnthropicKey,
        OpenRouterKey: envCfg.OpenRouterKey,
        OllamaHost:    envCfg.OllamaHost,
        LMStudioHost:  envCfg.LMStudioHost,
    }
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Env vars for provider/model | Viper config file + env var fallback | This phase (D-04) | Existing env-var users unaffected until they open Settings |
| No API key storage | 99designs/keyring SecretServiceBackend | This phase (D-05) | Keys survive app restart; never in config.yaml |
| `/filter` as only slash command | Full 8-command router in ChatPane | This phase (D-07) | All Phase 5 slash commands handled in one router block |
| Static "No model" in StatusBar | Live `activeModel` from settingsStore | This phase | StatusBar reflects current session provider:model |

**Not deprecated — still valid:**
- `LoadConfig()` reading env vars — kept as fallback per D-04 (not replaced)
- `wl-copy` for Wayland clipboard — same mechanism used for auto-clear with empty string

---

## Open Questions

1. **CaptureManager.ForceRefresh for `/refresh`**
   - What we know: `CaptureManager` has `Startup` and `GetAdapterStatus` but no explicit refresh RPC
   - What's unclear: Should `/refresh` call a new `ForceCapture` RPC that triggers immediate `tick()`, or is it sufficient to have the frontend emit a Wails event that resets the debounce state?
   - Recommendation: Add `ForceRefresh() error` to `SettingsService` that calls `captureManager.Tick()` (expose a `Tick` method or `ForceCapture`). Simpler than adding a new RPC to `CaptureManager` directly. Requires wiring `CaptureManager` reference into `SettingsService`.

2. **Hotkeys tab — global hotkey registration**
   - What we know: CFG-06 requires configurable global hotkeys; the tab exists but no implementation approach is locked.
   - What's unclear: Wails v2 does not natively support global hotkeys (outside the app window). This typically requires a system-level hook (`robotgo`, `golang-set/hotkey`).
   - Recommendation: Implement global hotkeys as in-app keyboard shortcuts only (window must be focused) for this phase. Store the key combination string in `AppConfig`. Register a `document.addEventListener('keydown')` listener in the frontend. Flag in RESEARCH that true global hotkeys (system-level, outside app focus) are out of scope for Phase 5 and would require a Phase 6+ dependency.

3. **LM Studio and Ollama API key handling**
   - What we know: LM Studio uses no API key (empty string); Ollama uses no API key.
   - What's unclear: The keychain `Get("lmstudio")` would return "not found" — this is the correct "blank" state per D-06. But the UI should not show a key input field for these providers at all.
   - Recommendation: LLM Config tab conditionally renders the API key section based on selected provider. `lmstudio` and `ollama` show "No API key required" instead of the key input.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `libsecret-1-0` | 99designs/keyring SecretServiceBackend runtime | Yes | 0.21.4 | — |
| `libsecret-1-dev` | CGO compile of `gsterjov/go-libsecret` | **No** | — | `sudo apt install libsecret-1-dev` (available in apt cache) |
| `gnome-keyring-daemon` | SecretServiceBackend at runtime | Yes | PID 131981 confirmed running | FileBackend fallback |
| `wl-copy` | Wayland clipboard (existing feature) | Not confirmed present | — | xclip (X11) |
| `go` runtime | All Go compilation | Yes | 1.24.11 | — |
| `@base-ui/react` Dialog | SettingsDialog component | Yes | v1.3.0 (installed) | — |
| `@base-ui/react` Tabs | SettingsDialog tabs | Yes | v1.3.0 (installed) | — |

**Missing dependencies with no fallback:**
- `libsecret-1-dev` — must install before `go get github.com/99designs/keyring`: `sudo apt install libsecret-1-dev`

**Missing dependencies with fallback:**
- None that block execution after libsecret-1-dev is installed.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | vitest v4.1.2 (frontend) + Go `testing` stdlib (backend) |
| Config file | `frontend/vite.config.ts` (test section inline) |
| Quick run command (frontend) | `cd frontend && npx vitest run` |
| Quick run command (Go) | `go test ./services/...` |
| Full suite command | `go test ./services/... && cd frontend && npx vitest run` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CFG-02 | `keychain.Client.Get/Set/Remove` round-trip | unit | `go test ./services/keychain/...` | Wave 0 |
| CFG-02 | `keychain.Client` injectable fn field enables mock keyring in tests | unit | `go test ./services/keychain/...` | Wave 0 |
| CFG-08 | `AppConfig` new fields persist to YAML and load back | unit | `go test ./services/config/...` | Wave 0 (extend existing) |
| CFG-08 | `SaveAppConfig` does not overwrite unrelated fields | unit | `go test ./services/config/...` | Wave 0 |
| CFG-01 | `SettingsService.GetSettings` returns current AppConfig | unit | `go test ./services/...` | Wave 0 |
| CFG-01 | `SettingsService.SaveSettings` writes to config file | unit | `go test ./services/...` | Wave 0 |
| CFG-03 | `SettingsService.TestConnection` returns success for mock provider | unit | `go test ./services/...` | Wave 0 |
| SLASH-01 | `/model openai:gpt-4` dispatched to SettingsService.SetModel | unit | `go test ./services/...` | Wave 0 |
| CLIP-03 | `CommandService.CopyToClipboard` launches auto-clear goroutine | unit (injectable sleep) | `go test ./services/...` | Wave 0 |
| D-07 | ChatPane `/theme dark` dispatches frontend-only (no Go call) | unit | `cd frontend && npx vitest run` | Wave 0 |
| D-07 | ChatPane `/help` adds system message with help text | unit | `cd frontend && npx vitest run` | Wave 0 |
| D-07 | ChatPane `/model` dispatches to SettingsService mock | unit | `cd frontend && npx vitest run` | Wave 0 |
| CFG-07 | ThreeColumnLayout renders SettingsDialog when open state is true | unit | `cd frontend && npx vitest run` | Wave 0 (extend existing) |

### Sampling Rate
- **Per task commit:** `go test ./services/... && cd /home/sblanken/working/ppa2/frontend && npx vitest run`
- **Per wave merge:** same (full suite)
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `services/keychain/keychain_test.go` — covers CFG-02 injectable mock keyring
- [ ] `services/settings_service_test.go` — covers CFG-01, CFG-03, SLASH-01
- [ ] `services/commands_test.go` — extend with auto-clear goroutine test (injectable sleep field)
- [ ] `services/config/config_test.go` — extend with new AppConfig field round-trip tests
- [ ] `frontend/src/components/__tests__/ChatPane.test.tsx` — slash command router tests
- [ ] `frontend/wailsjs/go/services/SettingsService.js` — vitest stub + `.gitignore` exception

---

## Sources

### Primary (HIGH confidence)
- `99designs/keyring` pkg.go.dev — Open, Config, Item, BackendType constants, ErrKeyNotFound, FileBackend fallback pattern
- `proxy.golang.org/github.com/99designs/keyring` — confirmed latest version v1.2.2, go.mod dependencies
- Project source: `services/config/config.go` — verified existing AppConfig struct, LoadAppConfig/SaveAppConfig patterns
- Project source: `services/llm_service.go` — verified FilterCommand pattern, buildProvider, injectable fields
- Project source: `services/commands.go` — verified CopyToClipboard, isWayland, copyViaWlCopy
- Project source: `frontend/src/theme/theme-provider.tsx` — verified useTheme() hook, setTheme, localStorage persistence
- Project source: `frontend/node_modules/@base-ui/react/dialog/index.d.ts` — verified Dialog.Root, Portal, Popup, Backdrop, Title, Close exports
- Project source: `frontend/node_modules/@base-ui/react/tabs/index.d.ts` — verified Tabs.Root, List, Tab, Panel exports
- Project source: `frontend/src/components/ui/tooltip.tsx` — verified `@base-ui/react/tooltip` subpath import pattern
- Project source: `frontend/src/components/layout/StatusBar.tsx` — verified disabled gear button at line ~32
- Project source: `frontend/src/components/chat/ChatPane.tsx` — verified /filter dispatch pattern for new slash commands
- Project source: `.gitignore` — verified wailsjs exception pattern for committed stubs
- System check: `gnome-keyring` PID 131981 — confirmed running
- System check: `libsecret-1-0` v0.21.4 — runtime installed; `libsecret-1-dev` NOT installed

### Secondary (MEDIUM confidence)
- `@base-ui/react` v1.3.0 changelog (inferred from installed package) — Dialog.Root `onOpenChange` API follows Radix/headlessui convention; `render` prop for custom element rendering
- 99designs/keyring `go-libsecret` CGO requirement — inferred from go.mod dependency; standard Linux secret service pattern

### Tertiary (LOW confidence)
- Global hotkey limitation in Wails v2 — based on Wails v2 feature set knowledge; not verified against Wails v2.12.0 release notes specifically. Recommendation to use in-app keyboard shortcuts only may need validation.

---

## Project Constraints (from CLAUDE.md)

CLAUDE.md does not exist in the working directory. No project-specific directives to enforce beyond the decisions in CONTEXT.md above.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries verified against installed packages and proxy.golang.org
- Architecture: HIGH — all patterns derived from existing codebase conventions, not assumptions
- Pitfalls: HIGH (CGO/libsecret, Viper overwrite, @base-ui import paths) / MEDIUM (Tabs asChild, clipboard race)
- Validation architecture: HIGH — test patterns verified against existing test files

**Research date:** 2026-03-29
**Valid until:** 2026-04-29 (stable libraries; @base-ui/react 1.x API stable)
