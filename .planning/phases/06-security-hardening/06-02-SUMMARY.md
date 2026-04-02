---
phase: 06-security-hardening
plan: 02
subsystem: security
tags: [memguard, audit, credentials, enclaves, testing]
dependency_graph:
  requires: [06-01]
  provides: [SEC-01, SEC-02, SEC-03, SEC-04]
  affects: [main.go, services/llm_service.go, services/commands.go, services/settings_service.go]
tech_stack:
  added: [github.com/awnumar/memguard v0.23.0, github.com/awnumar/memcall v0.4.0]
  patterns:
    - memguard Enclave lifecycle for in-memory API key protection
    - injectable emitFn/clipboardSetFn fields for test isolation
    - response-side credential filter on assembled LLM output before audit write
key_files:
  created: []
  modified:
    - main.go
    - services/llm_service.go
    - services/commands.go
    - services/settings_service.go
    - services/settings_service_test.go
    - services/llm_service_test.go
    - services/commands_test.go
    - go.mod
    - go.sum
decisions:
  - Injectable clipboardSetFn field added to CommandService for test isolation (consistent with emitFn pattern on SettingsService and LLMService)
  - RebuildProvider called immediately after Enclave loading in main() so provider uses Enclave keys from startup
  - clipboardSetFn captures copy function reference at timer setup time (not c.clipboardSetFn) to avoid data race
metrics:
  duration: "8 minutes"
  completed: "2026-04-02T04:59:00Z"
  tasks_completed: 3
  files_changed: 9
---

# Phase 6 Plan 2: memguard Secrets + Audit Wiring Summary

**One-liner:** memguard Enclave lifecycle for API keys, AuditLogger injected into LLMService/CommandService with all 5 event types, response-side credential filter on assembled LLM responses before audit write.

## What Was Built

### Task 1: memguard Enclave integration + audit wiring

**main.go:**
- `memguard.CatchInterrupt()` as first action in `main()` before any Enclave creation
- API keys loaded from keychain for providers ["openai", "anthropic", "openrouter"] and sealed into memguard Enclaves via `llmService.SetAPIKeyEnclave()`
- `llmService.RebuildProvider()` called after Enclave loading so provider uses Enclave keys immediately
- `sessionID = uuid.New().String()` and `audit.NewAuditLogger()` in `OnStartup` closure
- `llmService.SetAuditLogger()` and `commands.SetAuditLogger()` injected in `OnStartup`
- `auditLogger.Write(AuditEntry{Event: "session_start"})` in `OnStartup`
- `OnBeforeClose` hook added: writes `session_end`, calls `memguard.Purge()`
- `sessionID` and `auditLogger` declared in `main()` scope (before `wails.Run`) so both closures share them

**services/llm_service.go:**
- New fields: `auditLogger *audit.AuditLogger`, `sessionID string`, `apiKeyEnclaves map[string]*memguard.Enclave`, `emitFn` (injectable events emitter)
- `SetAuditLogger()`, `SetAPIKeyEnclave()`, `getAPIKeyString()`, `RebuildProvider()` methods added
- `getAPIKeyString()` opens Enclave into a LockedBuffer, copies bytes to string, calls `buf.Destroy()` immediately
- `SendMessage()`: writes `user_message` audit entry (userInput only, NOT terminalContext) before goroutine launch
- `SendMessage()` goroutine: accumulates `assembledParts` alongside `batch`; calls `writeAIResponseAudit()` at each stream completion point (channel closed or `chunk.Done`)
- `writeAIResponseAudit()`: runs assembled text through `filter.NewCredentialFilter().Apply()` before writing `ai_response` audit entry
- All `runtime.EventsEmit` calls replaced with `s.emitFn` calls for test isolation
- `buildProvider()` updated to accept `keyFn func(string) string`; keyFn takes precedence over Config fields

**services/commands.go:**
- New fields: `auditLogger *audit.AuditLogger`, `sessionID string`, `clipboardSetFn func(ctx, text) error`
- `SetAuditLogger()` method added
- `CopyToClipboard()` writes `command_copied` audit entry after successful copy (no `TerminalID` — method has no tabId parameter)
- `clipboardSetFn` replaces direct `runtime.ClipboardSetText` call for test isolation

**services/settings_service.go:**
- `buildProviderFn` type updated to `func(Config, func(string) string) llm.Provider`
- `TestConnection()` passes `nil` as keyFn
- `SaveAPIKey()`: after successful keychain write, seals key into new Enclave, calls `llmService.SetAPIKeyEnclave()` and `llmService.RebuildProvider()`

### Task 2: Unit tests

**services/llm_service_test.go** — 5 new tests:
- `TestSendMessageAuditUserMessage`: verifies `user_message` event written with userInput, without terminalContext
- `TestSendMessageAuditAIResponse`: verifies `ai_response` event written with `[REDACTED:anthropic-api-key]` when response contains credential pattern
- `TestGetAPIKeyStringFromEnclave`: verifies memguard Enclave round-trips API key correctly
- `TestGetAPIKeyStringNilEnclave`: verifies empty string when no Enclaves set
- `TestAuditLoggerNilNoOpLLMService`: verifies no panic with nil auditLogger

**services/commands_test.go** — 2 new tests:
- `TestCopyToClipboardAuditCommandCopied`: verifies `command_copied` event with command text
- `TestAuditLoggerNilNoOp`: verifies no panic with nil auditLogger

### Task 3: Security review checklist verification

All verifications passed — no code changes needed.

**Security Checklist Results:**

| Check | Result | Details |
|-------|--------|---------|
| Ollama remote-host guard | PASS | 9 tests: empty, localhost, 127.0.0.1, ::1 pass; remotehost, 192.168.1.100 rejected |
| Ollama error message | PASS | "OLLAMA_HOST must be localhost or 127.0.0.1; remote hosts are not allowed" |
| Filter pipeline coverage | PASS | 6 patterns: AWS key IDs, GitHub tokens, OpenAI keys, Anthropic keys, Bearer tokens, generic API keys |
| Filter patterns present | PASS | aws-access-key-id, github-token, openai-api-key, anthropic-api-key, bearer-token, generic-api-key |
| Filter patterns not covered | NOTE | GCP service account keys, SSH private key blocks, database DSN passwords (out of Phase 6 scope) |
| Config file secrets | PASS | AppConfig has no API key string fields — keys stored in keychain only |
| Audit log content rules | PASS | user_message = userInput only; ai_response = credential-filtered assembled text |

## Deviations from Plan

### Auto-added Missing Critical Functionality

**1. [Rule 2 - Testability] Injectable clipboardSetFn on CommandService**
- **Found during:** Task 2 implementation
- **Issue:** `CopyToClipboard()` calls `runtime.ClipboardSetText()` directly, which panics without a real Wails context in tests. No injectable hook existed.
- **Fix:** Added `clipboardSetFn func(ctx context.Context, text string) error` field, defaulting to `runtime.ClipboardSetText` in `NewCommandService()`. All call sites updated to use `clipboardSetFn`.
- **Files modified:** `services/commands.go`
- **Commit:** e967c65 (included in Task 2 commit)

### Auto-fixed Build Errors

**2. [Rule 1 - Bug] settings_service_test.go: buildProviderFn signature mismatch**
- **Found during:** Task 1 - first `go test` run
- **Issue:** 3 test fixtures assigned `func(_ Config) llm.Provider` to `buildProviderFn` which now requires `func(Config, func(string) string) llm.Provider`
- **Fix:** Updated all 3 mock assignments in `settings_service_test.go` to use `func(_ Config, _ func(string) string) llm.Provider`
- **Files modified:** `services/settings_service_test.go`
- **Commit:** ccdf285

## Security Checklist

- **memguard Enclave lifecycle:** PASS — API keys sealed at startup, opened into LockedBuffer only for HTTP header construction, destroyed immediately, purged on exit
- **session_start/session_end:** PASS — UUID4 session ID, events written in OnStartup/OnBeforeClose
- **user_message:** PASS — userInput only (not terminalContext), written before goroutine
- **ai_response:** PASS — credential-filtered assembled text, written after stream completion
- **command_copied:** PASS — command text written after successful clipboard copy
- **Response-side filter:** PASS — `filter.NewCredentialFilter().Apply()` runs on assembled response before audit write; user-displayed stream is unaffected
- **Ollama guard:** PASS — 9 tests covering all required host scenarios
- **No secrets in config:** PASS — AppConfig contains no API key fields

## Self-Check: PASSED

Files verified:
- main.go: FOUND
- services/llm_service.go: FOUND
- services/commands.go: FOUND
- services/settings_service.go: FOUND
- services/llm_service_test.go: FOUND
- services/commands_test.go: FOUND

Commits verified:
- ccdf285: feat(06-02): memguard Enclave lifecycle + AuditLogger wiring
- e967c65: test(06-02): unit tests for audit wiring, memguard integration, and response-side filter
