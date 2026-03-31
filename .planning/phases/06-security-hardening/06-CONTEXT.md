# Phase 6: Security Hardening - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Production-grade security hardening: wrap in-memory API keys with `memguard` locked buffers, implement a `slog`+`lumberjack` audit log with per-event entries, add a response-side credential scan on assembled LLM responses, and verify the Ollama remote-host guard (already implemented — Phase 6 adds a security review checklist item only).

No new UI, no new terminal adapters, no schema changes. All work is Go-side. The audit log writes to local disk only — no network.

</domain>

<decisions>
## Implementation Decisions

### memguard — API Key Protection

- **D-01:** `memguard` scope is **keychain boundary only**. Lock the key in a `memguard.LockedBuffer` immediately after `keychain.Get()` returns. Pass the sealed buffer to `LLMService` / `buildProvider`. The provider reads from the buffer only when constructing the HTTP `Authorization: Bearer` header — the plain-string lifetime is microseconds (header build only, not stored in any struct field). The `LockedBuffer` is the canonical storage.
- **D-02:** `LockedBuffer.Destroy()` is called **on app exit only** (in `OnBeforeClose`). One global buffer per configured provider. No per-request create/destroy. No destroy on settings save — the old buffer is destroyed and a new one created only when the user saves new API key credentials.

### Audit Log — Infrastructure

- **D-03:** Use Go's `log/slog` with a JSON handler writing to a `lumberjack.Logger` (file rotation). Output file: `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl`. One line per entry. Rotation: daily with max 30 days retained, max 100 MB per file.
- **D-04:** `AuditLogger` is a **struct with a `Write(entry AuditEntry) error` method**, injected as a field into `LLMService` and `CommandService` at startup in `main.go`. Consistent with the injectable `emitFn` / `execCommand` / `filterPipelineRebuilder` patterns from prior phases. Nil-safe: if the field is nil (tests that don't inject a logger), `Write()` is a no-op.
- **D-05:** `session_id` = UUID4 generated once at app startup (in `OnStartup`). `terminal_id` = existing tab ID from `terminalStore` (e.g., `tmux:%3`, `atspi:/path`). These are the values already in use across the capture and LLM pipeline.

### Audit Log — Events and Content

- **D-06:** Five event types:
  - `session_start` — emitted in `OnStartup` (app lifecycle)
  - `session_end` — emitted in `OnBeforeClose` (app lifecycle)
  - `user_message` — emitted by `LLMService` before sending to provider; content = **user's chat message text only** (not the assembled system prompt or terminal context prefix)
  - `ai_response` — emitted by `LLMService` after stream completes; content = **full assembled response text** (after response-side credential filter — see D-07)
  - `command_copied` — emitted by `CommandService.CopyToClipboard`; content = the command text copied

- **D-07 (session lifecycle placement):** `session_start` and `session_end` entries are written from `OnStartup` and `OnBeforeClose` in `main.go`. These are the canonical app lifecycle hooks.

### Response-Side Credential Filter

- **D-08:** After the LLM stream completes and the full response string is assembled, **run the same credential regex patterns** from the existing filter pipeline (`services/llm/filter`) on the response. Redact any matches with `[REDACTED]`. This is what gets written to the audit log (`ai_response` content). The response **displayed to the user is unaffected** — the filter runs on the copy written to the audit log only.
- **D-09:** The filter runs **after stream completes, before audit log write**. Zero UI impact — no streaming latency added.

### Ollama Remote-Host Guard

- **D-10:** `validateOllamaHost()` is already implemented in `services/llm/ollama.go` from Phase 2 and has existing tests. Phase 6 includes it only as a **security review checklist item** — verify the test coverage is complete and the error message is user-friendly. No code changes expected.

### Claude's Discretion

- Package name for audit infrastructure (`services/audit` vs `audit` at root)
- `AuditEntry` struct field names (snake_case JSON tags)
- `lumberjack` rotation config (max size, max age, compress flag)
- UUID generation approach (`crypto/rand`-based or `github.com/google/uuid`)
- Whether `command_copied` entry includes the originating question text

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Filter Pipeline (Response Scan Reuse)
- `services/llm/filter/` — existing credential regex patterns and `Apply()` method; response-side scan reuses these patterns
- `services/llm/filter/filter_test.go` — pattern coverage tests; verify coverage for D-10 checklist

### API Key Lifecycle (memguard Integration Points)
- `services/keychain/keychain.go` — `Get()` returns `string`; this is the insertion point for LockedBuffer creation (D-01)
- `services/settings_service.go` — `buildProvider(Config{APIKey: ...})` call site; receives key from keychain; update to pass LockedBuffer
- `services/llm/provider.go` — `Config` struct with `APIKey string`; understand before modifying

### Audit Log Wire Points
- `main.go` — `OnStartup` / `OnBeforeClose` callbacks; `session_start` / `session_end` go here (D-07)
- `services/llm_service.go` — streaming completion path; `user_message` and `ai_response` events go here (D-06)
- `services/commands.go` — `CopyToClipboard`; `command_copied` event goes here (D-06)

### Injectable Pattern Reference (for AuditLogger injection)
- `services/settings_service.go` — `emitFn` injectable field pattern (D-04)
- `services/llm/filter/` — `applyFilterPipeline` pattern (injectable func fields)

### Requirements
- `.planning/REQUIREMENTS.md` §SEC-01–04 — Acceptance criteria for this phase
- `.planning/ROADMAP.md` §Phase 6 — Key deliverables and exit criteria

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `services/llm/filter/` — `Apply(content string) string` method; reuse for response-side scan (D-08)
- `services/keychain/keychain.go` — `Get(provider string) (string, error)`; wrap return value in LockedBuffer (D-01)
- Injectable field pattern (all prior phases) — `emitFn`, `execCommand`, `filterPipelineRebuilder` — same approach for `AuditLogger` field

### Established Patterns
- **Injectable fields for test isolation** — nil-safe `Write()` no-op when field is nil enables tests without a real log file
- **`main.go` OnStartup/OnBeforeClose** — already used for `app.startup`, `commands.Startup`; add `auditLogger.session_start` / `session_end` here
- **UUID-like IDs**: tab IDs in terminalStore already use pane ID format (`tmux:%3`); session UUID is additive (new at startup)

### Integration Points
- `main.go` `OnStartup` → generate session UUID + write `session_start`
- `main.go` `OnBeforeClose` → write `session_end` + call `LockedBuffer.Destroy()`
- `LLMService.Stream()` → emit `user_message` before provider call; collect stream → emit `ai_response` with filtered content
- `CommandService.CopyToClipboard()` → emit `command_copied`

</code_context>

<specifics>
## Specific Ideas

- **LockedBuffer read pattern:** `buf.Reader().Read(p []byte)` or `buf.Bytes()` — check memguard API; the plain string should only exist in the stack frame of the HTTP header builder, not any struct field
- **Audit JSONL schema example:**
  ```json
  {"time":"2026-03-30T10:00:00Z","level":"INFO","event":"user_message","session_id":"uuid4","terminal_id":"tmux:%3","content":"how do I list open ports?"}
  ```
- **Nil AuditLogger pattern:** `if l.auditLogger != nil { l.auditLogger.Write(entry) }` — same nil check pattern as `emitFn` in `SettingsService`

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 06-security-hardening*
*Context gathered: 2026-03-30*
