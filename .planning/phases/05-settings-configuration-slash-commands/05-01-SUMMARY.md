---
phase: 05-settings-configuration-slash-commands
plan: "01"
subsystem: settings-backend
tags:
  - go
  - settings
  - keychain
  - config
  - clipboard
dependency_graph:
  requires:
    - services/config/config.go
    - services/keychain/keychain.go
    - services/llm_service.go
    - services/commands.go
    - main.go
  provides:
    - services/settings_service.go (SettingsService Wails RPC)
    - services/keychain/keychain.go (keychain client with NewWithOpenFunc)
    - services/config/config.go (expanded AppConfig)
    - LoadConfigWithViper (D-04 config priority)
  affects:
    - main.go (new SettingsService bind + LoadConfigWithViper)
    - services/commands.go (clipboard auto-clear timer)
tech_stack:
  added:
    - github.com/99designs/keyring v1.2.2
  patterns:
    - Injectable function field (emitFn, buildProviderFn, open) for test isolation
    - Wails runtime.EventsEmit guarded by injectable emitFn to avoid log.Fatalf in tests
    - time.AfterFunc for non-blocking clipboard auto-clear with sync.Mutex cancellation
key_files:
  created:
    - services/settings_service.go
    - services/settings_service_test.go
    - services/keychain/keychain.go
    - services/keychain/keychain_test.go
    - frontend/wailsjs/go/services/SettingsService.js
  modified:
    - services/config/config.go (expanded AppConfig + SaveAppConfig merge fix)
    - services/config/config_test.go (updated for new fields)
    - services/commands.go (clearTimer + clearMu fields)
    - services/commands_test.go (auto-clear timer tests)
    - main.go (SettingsService wired, LoadConfigWithViper)
    - services/keychain/keychain.go (added NewWithOpenFunc)
    - go.mod / go.sum (99designs/keyring dependency)
    - .gitignore (SettingsService.js exception)
decisions:
  - "Injectable emitFn field on SettingsService — guards against Wails log.Fatalf on non-Wails contexts in tests; same pattern as injectable lookPath in CommandService"
  - "NewWithOpenFunc constructor on keychain.Client — exposes injectable open field for cross-package test injection without making the field exported"
  - "buildProviderFn package-level var on settings_service — allows TestConnection to use mock provider in unit tests"
  - "LoadConfigWithViper in settings_service.go (not llm_service.go) — keeps LLMService unchanged; new function bridges Viper config and existing Config struct"
  - "clearTimer uses time.AfterFunc not time.Sleep — non-blocking; Stop() cancels without goroutine leak"
metrics:
  duration: "~45 minutes"
  completed_date: "2026-03-30"
  tasks_completed: 2
  files_changed: 13
requirements:
  - CFG-01
  - CFG-02
  - CFG-03
  - CFG-04
  - CFG-05
  - CFG-08
  - CLIP-03
---

# Phase 05 Plan 01: Settings Backend (Go) Summary

Go-side infrastructure for Phase 5 settings: expanded AppConfig with 10 new fields, 99designs/keyring-backed keychain client with injectable open function, SettingsService Wails RPC with GetSettings/SaveSettings/GetAPIKeyStatus/SaveAPIKey/TestConnection, Viper-first config priority (LoadConfigWithViper), and clipboard auto-clear via cancellable time.AfterFunc timer.

## Tasks Completed

| Task | Description | Commit |
|------|-------------|--------|
| 1 | Expand AppConfig + keychain package + SaveAppConfig merge fix | e6056f7 |
| 2 | SettingsService + clipboard auto-clear + main.go wiring | 76c246f |

## What Was Built

### AppConfig Expansion (services/config/config.go)
Added 10 new fields to `AppConfig`: `Provider`, `Model`, `CustomPrompt`, `ATSPIPollingMs`, `ClipboardClearSecs`, `HotkeyCopyLast`, `HotkeyFocusWindow`, `Theme`, `FontSize`, `ContextLines` — all with `mapstructure` and `yaml` tags. `SaveAppConfig` now calls `v.ReadInConfig()` before `v.Set()` to merge without overwriting unrelated existing fields.

### Keychain Client (services/keychain/keychain.go)
Thin wrapper around `99designs/keyring` with injectable `open func(keyring.Config) (keyring.Keyring, error)` field. `Get` returns `""` (not error) on `keyring.ErrKeyNotFound`. `NewWithOpenFunc` constructor allows cross-package test injection. Uses `SecretServiceBackend + FileBackend` with `~/.pairadmin/keyring` fallback.

### SettingsService (services/settings_service.go)
Wails-bound service with:
- `GetSettings()` — calls `config.LoadAppConfig()`
- `SaveSettings(cfg)` — calls `config.SaveAppConfig(cfg)`, emits `settings:changed` via injectable `emitFn`
- `GetAPIKeyStatus(provider)` — returns `"stored"` or `""` (never the actual key)
- `SaveAPIKey(provider, key)` — delegates to `keychainClient.Set/Remove`
- `TestConnection(provider, model)` — builds temp provider via `buildProviderFn`, calls `TestConnection(ctx)`, returns `"Connected"` or error

### LoadConfigWithViper (services/settings_service.go)
`LoadConfigWithViper()` implements D-04: reads `Provider` and `Model` from AppConfig (Viper/YAML), falls back to `PAIRADMIN_PROVIDER`/`PAIRADMIN_MODEL` env vars when those fields are empty. `main.go` now uses `LoadConfigWithViper()` instead of `LoadConfig()`.

### Clipboard Auto-Clear (services/commands.go)
Added `clearTimer *time.Timer` and `clearMu sync.Mutex` to `CommandService`. After each successful clipboard write, the previous timer is stopped and a new `time.AfterFunc` is created using `ClipboardClearSecs` from config (defaults to 60s). Clear goroutine calls `wl-copy ""` on Wayland or `runtime.ClipboardSetText(ctx, "")` on X11.

### SettingsService.js Stub
`frontend/wailsjs/go/services/SettingsService.js` committed with `.gitignore` exception for vitest import resolution. Exports: `GetSettings`, `SaveSettings`, `GetAPIKeyStatus`, `SaveAPIKey`, `TestConnection`, `SetModel`, `SetContextLines`, `ForceRefresh`, `ExportChat`, `RenameTab`.

## Test Results

```
ok  pairadmin/services          (11 new tests pass)
ok  pairadmin/services/config   (existing tests pass)
ok  pairadmin/services/keychain (5 tests pass)
ok  pairadmin/services/capture  (existing tests pass)
ok  pairadmin/services/llm      (existing tests pass)
ok  pairadmin/services/llm/filter (existing tests pass)
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing critical functionality] Injectable emitFn on SettingsService**
- **Found during:** Task 2 — test implementation
- **Issue:** `runtime.EventsEmit` calls `log.Fatalf` (os.Exit) when called with a non-Wails `context.Background()`. `SaveSettings` emits an event, making `TestSettingsService_SaveSettings` fail with process exit.
- **Fix:** Added injectable `emitFn func(ctx context.Context, event string, optionalData ...interface{})` field to `SettingsService`. Set to `runtime.EventsEmit` in `NewSettingsService`; tests set it to `nil`.
- **Files modified:** `services/settings_service.go`, `services/settings_service_test.go`
- **Commit:** 76c246f

**2. [Rule 2 - Missing critical functionality] NewWithOpenFunc cross-package constructor**
- **Found during:** Task 2 — writing settings_service_test.go
- **Issue:** `keychain.Client.open` is unexported; tests in `package services` cannot inject a mock keyring using the existing `makeTestClient` helper (which is `package keychain` only).
- **Fix:** Added `NewWithOpenFunc(openFn func(keyring.Config) (keyring.Keyring, error)) *Client` to `services/keychain/keychain.go`.
- **Files modified:** `services/keychain/keychain.go`
- **Commit:** 76c246f

**3. [Rule 3 - Blocking] Cherry-pick Task 1 from sibling worktree**
- **Found during:** Start of execution
- **Issue:** Task 1 commit `f9f5cbf` was on branch `worktree-agent-a743b0ba`, not on `worktree-agent-ac92c622`. The current worktree did not have the expanded AppConfig or keychain package.
- **Fix:** Cherry-picked `f9f5cbf` into this branch via `git cherry-pick f9f5cbf --no-commit` and committed as `e6056f7`.
- **Files modified:** All Task 1 files (go.mod, go.sum, services/config/config.go, services/config/config_test.go, services/keychain/keychain.go, services/keychain/keychain_test.go)
- **Commit:** e6056f7

## Known Stubs

The following stubs are intentional and tracked for Phase 5 frontend plans:

- `frontend/wailsjs/go/services/SettingsService.js` — all exports return empty resolved promises. Real bindings are generated by `wails dev` at runtime. This stub exists solely for vitest import resolution. Phase 05-02 (settings dialog frontend) will consume these RPCs.

## Self-Check: PASSED

Files verified:
- `services/settings_service.go` — FOUND
- `services/settings_service_test.go` — FOUND
- `services/keychain/keychain.go` — FOUND (with NewWithOpenFunc)
- `frontend/wailsjs/go/services/SettingsService.js` — FOUND
- Commit e6056f7 — FOUND (Task 1)
- Commit 76c246f — FOUND (Task 2)
