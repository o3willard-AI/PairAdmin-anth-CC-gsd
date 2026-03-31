# Phase 6: Security Hardening - Research

**Researched:** 2026-03-30
**Domain:** Go security primitives — memguard, slog+lumberjack audit logging, credential filter reuse, Ollama guard verification
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** memguard scope is keychain boundary only. Lock the key in a `memguard.LockedBuffer` immediately after `keychain.Get()` returns. Pass the sealed buffer (as an `*Enclave`) to `LLMService` / `buildProvider`. The provider reads from the buffer only when constructing the HTTP `Authorization: Bearer` header — the plain-string lifetime is microseconds (header build only, not stored in any struct field). The `LockedBuffer` / `Enclave` is the canonical in-memory storage.
- **D-02:** `LockedBuffer.Destroy()` is called on app exit only (in `OnBeforeClose`). One global Enclave per configured provider. No per-request create/destroy. No destroy on settings save — old Enclave is destroyed and a new one created only when the user saves new API key credentials.
- **D-03:** Use Go's `log/slog` with a JSON handler writing to a `lumberjack.Logger` (file rotation). Output file: `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl`. One line per entry. Rotation: daily with max 30 days retained, max 100 MB per file.
- **D-04:** `AuditLogger` is a struct with a `Write(entry AuditEntry) error` method, injected as a field into `LLMService` and `CommandService` at startup in `main.go`. Nil-safe: if the field is nil, `Write()` is a no-op.
- **D-05:** `session_id` = UUID4 generated once at app startup (in `OnStartup`). `terminal_id` = existing tab ID from `terminalStore` (e.g., `tmux:%3`, `atspi:/path`).
- **D-06:** Five event types: `session_start`, `session_end`, `user_message`, `ai_response`, `command_copied`. `user_message` content = user's chat message text only. `ai_response` content = full assembled response text after response-side credential filter.
- **D-07:** `session_start` and `session_end` are written from `OnStartup` and `OnBeforeClose` in `main.go`.
- **D-08:** After the LLM stream completes and the full response string is assembled, run the same credential regex patterns from the existing filter pipeline (`services/llm/filter`) on the response. Redact any matches with `[REDACTED]`. This is what gets written to the audit log (`ai_response` content). The response displayed to the user is unaffected.
- **D-09:** The filter runs after stream completes, before audit log write. Zero UI impact.
- **D-10:** `validateOllamaHost()` is already implemented in `services/llm/ollama.go` from Phase 2 and has existing tests. Phase 6 includes it only as a security review checklist item — verify test coverage is complete and error message is user-friendly. No code changes expected.

### Claude's Discretion

- Package name for audit infrastructure (`services/audit` vs `audit` at root)
- `AuditEntry` struct field names (snake_case JSON tags)
- `lumberjack` rotation config (max size, max age, compress flag)
- UUID generation approach (`crypto/rand`-based or `github.com/google/uuid`)
- Whether `command_copied` entry includes the originating question text

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SEC-01 | API keys loaded from keychain into memory are protected using `memguard` (mlock, encrypted at rest in process) | memguard v0.23.0 Enclave pattern verified; `NewBufferFromBytes` + `Seal()` + `Open()` is the correct lifecycle |
| SEC-02 | All AI interactions are written to a local audit log at `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl` using Go's `slog` with `lumberjack` rotation | `log/slog` stdlib (Go 1.21+), `gopkg.in/natefinch/lumberjack.v2` v2.2.1 verified; date-stamped filename requires custom `lumberjack.Logger.Filename` with `time.Now().Format` |
| SEC-03 | Audit log entries contain: timestamp, session ID, terminal ID, event type, sanitized content (filtered), command copied | AuditEntry struct design; slog JSON handler produces RFC3339 `time` field automatically; all fields verified against CONTEXT.md D-06 |
| SEC-04 | LLM response content is scanned by the credential filter (lighter-weight pass) to catch model hallucinations of sensitive data | Existing `CredentialFilter.Apply()` in `services/llm/filter/credential.go` is directly reusable; no new dependency needed |
</phase_requirements>

---

## Summary

Phase 6 is a purely Go-side security hardening phase with four discrete workstreams: (1) wrapping in-memory API keys with memguard enclaves after keychain retrieval, (2) implementing an `AuditLogger` struct backed by `slog`+`lumberjack` and wiring it into `LLMService` and `CommandService`, (3) adding a response-side credential scan that filters the assembled LLM response before writing it to the audit log, and (4) a security review checklist confirming the Ollama remote-host guard is complete.

All decisions are locked by CONTEXT.md. The codebase is well-prepared: the filter pipeline, injectable field pattern, keychain client, and Ollama guard are all already implemented. The main additions are two new dependencies (`github.com/awnumar/memguard` and `gopkg.in/natefinch/lumberjack.v2`) and one new package (`services/audit`). The `google/uuid` package is already in `go.mod` at v1.6.0 and can be used directly for session UUID generation.

The memguard pattern requires using `Enclave` (encrypted at-rest storage) for the long-lived API key rather than a raw `LockedBuffer` — this is the idiomatic memguard approach. An `Enclave` is created from `NewBufferFromBytes(rawKey).Seal()`, stored as a field on `LLMService`, opened into a temporary `LockedBuffer` only during HTTP header construction, and the `LockedBuffer` destroyed immediately after. On app exit, `memguard.Purge()` is called which destroys all enclaves.

**Primary recommendation:** Implement `services/audit` package first (pure struct, no external deps beyond stdlib + lumberjack), then memguard integration (touches `keychain`, `settings_service`, `llm_service`), then response-side filter (two lines in `SendMessage`), then the Ollama checklist review. This ordering minimizes merge risk.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `github.com/awnumar/memguard` | v0.23.0 | In-memory API key protection: mlock, XSalsa20Poly1305 encryption at rest, guard pages, wipe on exit | Purpose-built Go library for sensitive in-memory data; only option in Go ecosystem for this problem |
| `gopkg.in/natefinch/lumberjack.v2` | v2.2.1 | Log file rotation: max size, max age, optional gzip compression | Standard Go log rotation library; works as `io.Writer` for `slog` handler with zero extra API surface |
| `log/slog` | stdlib (Go 1.21+) | Structured JSON logging | Project already uses Go 1.24.11; slog is the official stdlib answer — no external dep needed |
| `github.com/google/uuid` | v1.6.0 | Session UUID4 generation | Already in `go.mod` as a transitive dependency; `uuid.New().String()` generates RFC4122 UUID4 |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `services/llm/filter` (existing) | — | `CredentialFilter.Apply()` reused for response-side scan | Already compiled into the binary; zero additional cost |
| `services/keychain` (existing) | — | `Client.Get()` insertion point for memguard wrapping | Existing `Get()` returns a `string`; wrap before returning upstream |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `gopkg.in/natefinch/lumberjack.v2` | `gopkg.in/lumberjack.v2` | `natefinch` fork is the actively maintained canonical; `lumberjack.v2` (v2.0.0, 2018) is unmaintained |
| `github.com/google/uuid` | `crypto/rand` manual | `google/uuid` is already in go.mod; manual approach would duplicate its implementation |
| `Enclave` (at-rest) + `LockedBuffer` (transient) | `LockedBuffer` only | Enclaves are the recommended long-term storage in memguard; LockedBuffers are constrained by system mlock limits and should be short-lived |

**Installation:**
```bash
go get github.com/awnumar/memguard@v0.23.0
go get gopkg.in/natefinch/lumberjack.v2@v2.2.1
```

**Version verification (confirmed 2026-03-30):**
- `github.com/awnumar/memguard`: v0.23.0 (released 2025-08-27) — confirmed via Go module proxy
- `gopkg.in/natefinch/lumberjack.v2`: v2.2.1 (released 2023-02-06) — confirmed via Go module proxy
- `github.com/google/uuid`: v1.6.0 — already in go.mod

---

## Architecture Patterns

### Recommended Project Structure

```
services/
├── audit/
│   └── audit.go          # AuditLogger struct, AuditEntry, Write(), NewAuditLogger()
services/
├── llm_service.go        # Add auditLogger field, wire user_message + ai_response events
├── commands.go           # Add auditLogger field, wire command_copied event
main.go                   # Generate session UUID, call NewAuditLogger, inject into services,
                          # write session_start in OnStartup, session_end in OnBeforeClose,
                          # call memguard.Purge() in OnBeforeClose
services/llm/
├── provider.go           # Add APIKeyEnclave field (or accept Enclave in buildProvider)
```

### Pattern 1: Enclave Lifecycle for API Keys

**What:** Create an `*Enclave` from the raw keychain string at app startup. Pass the Enclave to `LLMService`. When `buildProvider` constructs the HTTP client, open the Enclave into a temporary `LockedBuffer`, extract bytes as string for the header, then immediately destroy the `LockedBuffer`.

**When to use:** Always — for all cloud provider API keys (openai, anthropic, openrouter).

```go
// Source: pkg.go.dev/github.com/awnumar/memguard@v0.23.0

// In main.go OnStartup or at provider build time:
rawKey, _ := keychainClient.Get("openai")
buf := memguard.NewBufferFromBytes([]byte(rawKey)) // rawKey bytes are wiped by memguard
enclave := buf.Seal()                               // buf is destroyed; enclave holds encrypted bytes

// Stored as a field: svc.apiKeyEnclave = enclave

// In buildProvider / HTTP header construction:
lockedBuf, err := enclave.Open()
if err != nil { /* handle */ }
apiKey := string(lockedBuf.Bytes()) // stack-only lifetime
req.Header.Set("Authorization", "Bearer " + apiKey)
lockedBuf.Destroy()                 // wipe immediately; apiKey string still lives briefly
// Note: the Go string `apiKey` is a copy; it lives until GC. This is acceptable per D-01.

// In OnBeforeClose:
memguard.Purge() // destroys all live enclaves; called before app exits
```

**IMPORTANT API clarification (verified against memguard v0.23.0 docs):** `NewBufferFromBytes` wipes the source slice. `buf.Seal()` encrypts the buffer into an Enclave and destroys the `LockedBuffer`. `enclave.Open()` decrypts into a new immutable `LockedBuffer`. `lockedBuf.Bytes()` returns the plaintext bytes. `memguard.Purge()` resets the session key, making all Enclaves permanently unreadable.

**Where `CatchInterrupt` fits:** `memguard.CatchInterrupt()` registers a signal handler that calls `Purge()` on SIGINT/SIGTERM. Call this once in `main()` before any Enclaves are created.

### Pattern 2: AuditLogger Injectable Field

**What:** `AuditLogger` struct with a `Write(AuditEntry) error` method. Nil-safe — `Write()` returns `nil` immediately if the logger is not initialized. Injected into `LLMService` and `CommandService` as struct fields at wire-up time in `main.go`.

**When to use:** Everywhere an audit event must be emitted. Follows the exact same pattern as `emitFn` in `SettingsService`.

```go
// Source: CONTEXT.md D-04 + existing emitFn pattern in services/settings_service.go

// services/audit/audit.go
type AuditEntry struct {
    Event      string `json:"event"`        // "session_start"|"session_end"|"user_message"|"ai_response"|"command_copied"
    SessionID  string `json:"session_id"`
    TerminalID string `json:"terminal_id,omitempty"`
    Content    string `json:"content,omitempty"`
}

type AuditLogger struct {
    logger *slog.Logger
}

func (a *AuditLogger) Write(entry AuditEntry) error {
    if a == nil || a.logger == nil {
        return nil // nil-safe no-op
    }
    a.logger.Info("audit",
        slog.String("event", entry.Event),
        slog.String("session_id", entry.SessionID),
        slog.String("terminal_id", entry.TerminalID),
        slog.String("content", entry.Content),
    )
    return nil
}

// In LLMService:
type LLMService struct {
    // ... existing fields
    auditLogger *audit.AuditLogger
    sessionID   string
}

// Emit user_message before provider call:
if svc.auditLogger != nil {
    svc.auditLogger.Write(audit.AuditEntry{
        Event:      "user_message",
        SessionID:  svc.sessionID,
        TerminalID: tabId,
        Content:    userInput,
    })
}
```

### Pattern 3: Date-Stamped Audit Log Filename

**What:** lumberjack's `Filename` field does not support date substitution natively. The `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl` requirement means the filename must be computed at `AuditLogger` creation time. Since the app runs for at most one session per process, this is sufficient — the file is `audit-2026-03-30.jsonl` for the session started on 2026-03-30.

**When to use:** `NewAuditLogger()` construction in `main.go` `OnStartup`.

```go
// Source: gopkg.in/natefinch/lumberjack.v2 documentation + CONTEXT.md D-03

func NewAuditLogger(logDir string) (*AuditLogger, error) {
    if err := os.MkdirAll(logDir, 0700); err != nil {
        return nil, fmt.Errorf("audit: failed to create log dir: %w", err)
    }
    dateStr := time.Now().Format("2006-01-02")
    filename := filepath.Join(logDir, fmt.Sprintf("audit-%s.jsonl", dateStr))

    rotator := &lumberjack.Logger{
        Filename: filename,
        MaxSize:  100,   // MB per file (D-03: max 100 MB)
        MaxAge:   30,    // days retained (D-03: max 30 days)
        Compress: false, // keep as plaintext .jsonl for easy grep
    }
    handler := slog.NewJSONHandler(rotator, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    return &AuditLogger{logger: slog.New(handler)}, nil
}
```

**Note on `Compress`:** CONTEXT.md D-03 does not specify compression. Leaving `Compress: false` keeps logs as plain `.jsonl` files, directly readable with `jq`. This is Claude's discretion — recommend `false`.

### Pattern 4: Response-Side Credential Scan

**What:** After the stream loop exits and the full response string is assembled in `SendMessage`, instantiate `CredentialFilter` and run `Apply()` on the assembled string. Store the filtered version for the audit log entry. The unfiltered string continues to be emitted to the frontend as before.

**When to use:** Inside `SendMessage` goroutine, just before writing the `ai_response` audit entry.

```go
// Source: existing services/llm/filter/credential.go Apply() pattern

// Inside the streaming goroutine, after stream completes:
var fullResponse strings.Builder
// ... accumulate chunks into fullResponse ...

assembled := fullResponse.String()

// Response-side scan for audit log (D-08, D-09)
var auditContent string
credFilter, err := filter.NewCredentialFilter()
if err == nil {
    auditContent, _ = credFilter.Apply(assembled)
} else {
    auditContent = assembled // degrade gracefully: log unfiltered on filter init failure
}

if s.auditLogger != nil {
    s.auditLogger.Write(audit.AuditEntry{
        Event:      "ai_response",
        SessionID:  s.sessionID,
        TerminalID: tabId,
        Content:    auditContent,
    })
}
```

**Note on streaming architecture:** The current `SendMessage` goroutine does not accumulate the full response — it batches and emits tokens every 50ms. The response-side filter requires accumulating the full response string first. The accumulation happens in the existing `batch` slice; after the stream ends, join all batches into the full response string for the audit log write. Frontend delivery is unaffected.

### Anti-Patterns to Avoid

- **Storing API keys as `string` fields on any struct:** A Go `string` is immutable and GC-controlled; memguard cannot protect it once it escapes to the heap. The Enclave-then-LockedBuffer pattern keeps the plaintext in a mlock'd region. The brief moment where `string(lockedBuf.Bytes())` creates a Go string for the HTTP header is acceptable per D-01.
- **Calling `LockedBuffer.Destroy()` on the Enclave's buffer before the HTTP request completes:** Destroy the LockedBuffer immediately AFTER the header is set, not before. Header values are copied by the HTTP library.
- **Creating a new `CredentialFilter` per request in the hot path:** `NewCredentialFilter()` is cheap (no compilation at call time — patterns are package-level `var`), but instantiating it in the goroutine's main loop is unnecessary. Create it once and pass it in, or create it once after stream completion.
- **Writing unfiltered `terminalContext` to the audit log:** Per D-06 and D-08, `user_message` content = user's chat text only, not the terminal context. `ai_response` content = filtered response. Never log raw terminal snapshot content.
- **Using `lumberjack.v2` (gopkg.in/lumberjack.v2) instead of `natefinch/lumberjack.v2`:** The non-natefinch path is v2.0.0 from 2018 and unmaintained. Always use `gopkg.in/natefinch/lumberjack.v2`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| In-memory credential protection | Custom mlock wrapper, encrypted byte slice | `github.com/awnumar/memguard` | Guard pages, signal handling, encrypted at rest, cross-platform mlock — 8 years of production hardening |
| Log file rotation | Custom `os.Rename` + size check | `gopkg.in/natefinch/lumberjack.v2` | Handles concurrent writes, atomic rotation, gzip, max age cleanup — correctness is non-trivial |
| UUID4 generation | `crypto/rand` with manual formatting | `github.com/google/uuid` v1.6.0 | Already in go.mod; correct RFC4122 UUID4; no reason to duplicate |
| Response credential scan | New regex compilation per response | Reuse `filter.NewCredentialFilter()` + `.Apply()` | Same patterns, same code path — prevents drift between input and response filtering |

**Key insight:** All the hard security primitives are either already in go.mod or have been in production use for years. Phase 6 is integration work, not cryptography work.

---

## Common Pitfalls

### Pitfall 1: Enclave vs LockedBuffer Confusion

**What goes wrong:** Developer stores API key as a `LockedBuffer` long-term and hits mlock system limits, OR creates a new `LockedBuffer` from the Enclave on every HTTP request and forgets to destroy it, causing a LockedBuffer leak.

**Why it happens:** The memguard README shows `LockedBuffer` prominently; the `Enclave` concept is explained later. First-time users often skip Enclaves.

**How to avoid:** The lifecycle is: `rawBytes → NewBufferFromBytes → buf.Seal() → *Enclave` (long-lived, encrypted). When needed: `enclave.Open() → *LockedBuffer → .Bytes() → use → .Destroy()` (short-lived, plaintext). The Enclave is the durable form; the LockedBuffer is the transient read-out.

**Warning signs:** `LockedBuffer` stored as a struct field that lives past the function call that created it; no `.Destroy()` call paired with `enclave.Open()`.

### Pitfall 2: Accumulating Full Response for Audit Requires Refactoring SendMessage

**What goes wrong:** The current `SendMessage` goroutine batches tokens into `batch []string` and flushes every 50ms — it does not accumulate a full `assembledResponse` string. Adding the response-side scan and audit log write requires also accumulating all tokens into a separate buffer.

**Why it happens:** The streaming architecture is optimized for low latency, not for post-stream analysis.

**How to avoid:** Add a `var assembledParts []string` alongside the existing `batch []string`. Append each `chunk.Text` to both. After the stream ends (either `chunk.Done` or channel closed), join `assembledParts` for the audit write. This adds one `append` per chunk — negligible cost.

**Warning signs:** Tests that verify audit log content but use a mock stream with many chunks — the test should assert on the joined content, not individual chunks.

### Pitfall 3: lumberjack Filename vs Daily Rotation

**What goes wrong:** Using a fixed filename like `audit.jsonl` with `MaxAge: 30` — lumberjack renames the file on size rotation (e.g., `audit-2026-03-30T10:00:00.000.jsonl`) but does not rotate by date. The `audit-YYYY-MM-DD.jsonl` naming in CONTEXT.md D-03 requires computing the date at `NewAuditLogger()` call time.

**Why it happens:** lumberjack's rotation is size and age based, not calendar based.

**How to avoid:** Compute the filename at construction time: `fmt.Sprintf("audit-%s.jsonl", time.Now().Format("2006-01-02"))`. Since `OnStartup` is called once per process, the file name is fixed for the session lifetime. If the app runs across midnight, logs continue into the same file — this is acceptable for a desktop application.

**Warning signs:** `Filename: filepath.Join(logDir, "audit.jsonl")` without date — produces lumberjack's default rotation naming, not the YYYY-MM-DD format.

### Pitfall 4: Nil AuditLogger in Tests Without Explicit Guard

**What goes wrong:** `LLMService` or `CommandService` test calls `Write()` on a nil `*audit.AuditLogger` pointer, causing a nil pointer dereference panic.

**Why it happens:** Go nil receiver dispatch works for methods on concrete types, but the nil check must be on the receiver inside the method, not on the pointer itself.

**How to avoid:** Implement `Write()` with an explicit nil check on the receiver:
```go
func (a *AuditLogger) Write(entry AuditEntry) error {
    if a == nil || a.logger == nil {
        return nil
    }
    // ...
}
```
This matches the `emitFn nil` guard in `SettingsService` and allows test code to leave `auditLogger` as its zero value (`nil`) without crashing.

### Pitfall 5: memguard Purge vs Individual Destroy Ordering

**What goes wrong:** Calling individual `enclave` destroy before `memguard.Purge()` in `OnBeforeClose` causes double-free behavior — `Purge()` attempts to destroy already-destroyed enclaves.

**Why it happens:** `memguard.Purge()` destroys ALL live enclaves globally. If individual enclaves have already been destroyed, Purge is still safe (it resets the session key) but individual Destroy calls before Purge are redundant.

**How to avoid:** Per D-02, only call `memguard.Purge()` in `OnBeforeClose`. Do NOT call individual `enclave.Destroy()` elsewhere (except when replacing a key on settings save — old enclave destroyed, new one created). Let `Purge()` be the sole cleanup mechanism at exit.

---

## Code Examples

Verified patterns from official sources:

### AuditLogger Struct and NewAuditLogger Constructor

```go
// services/audit/audit.go
// Source: CONTEXT.md D-03, D-04 + log/slog stdlib + gopkg.in/natefinch/lumberjack.v2 API

package audit

import (
    "fmt"
    "log/slog"
    "os"
    "path/filepath"
    "time"

    "gopkg.in/natefinch/lumberjack.v2"
)

type AuditEntry struct {
    Event      string `json:"event"`
    SessionID  string `json:"session_id"`
    TerminalID string `json:"terminal_id,omitempty"`
    Content    string `json:"content,omitempty"`
}

type AuditLogger struct {
    logger *slog.Logger
}

func NewAuditLogger(logDir string) (*AuditLogger, error) {
    if err := os.MkdirAll(logDir, 0700); err != nil {
        return nil, fmt.Errorf("audit: mkdir %s: %w", logDir, err)
    }
    filename := filepath.Join(logDir, fmt.Sprintf("audit-%s.jsonl", time.Now().Format("2006-01-02")))
    rotator := &lumberjack.Logger{
        Filename: filename,
        MaxSize:  100, // MB
        MaxAge:   30,  // days
        Compress: false,
    }
    return &AuditLogger{
        logger: slog.New(slog.NewJSONHandler(rotator, &slog.HandlerOptions{Level: slog.LevelInfo})),
    }, nil
}

func (a *AuditLogger) Write(entry AuditEntry) error {
    if a == nil || a.logger == nil {
        return nil
    }
    a.logger.Info("audit",
        slog.String("event", entry.Event),
        slog.String("session_id", entry.SessionID),
        slog.String("terminal_id", entry.TerminalID),
        slog.String("content", entry.Content),
    )
    return nil
}
```

### memguard Enclave Lifecycle in main.go

```go
// Source: pkg.go.dev/github.com/awnumar/memguard@v0.23.0 + CONTEXT.md D-01, D-02

import "github.com/awnumar/memguard"

func main() {
    memguard.CatchInterrupt() // registers SIGINT/SIGTERM handler calling Purge()

    // ... service setup ...

    err := wails.Run(&options.App{
        OnStartup: func(ctx context.Context) {
            // Load API key and seal into Enclave immediately
            rawKey, _ := keychainClient.Get("openai")
            if rawKey != "" {
                buf := memguard.NewBufferFromBytes([]byte(rawKey)) // wipes rawKey []byte
                llmService.apiKeyEnclave = buf.Seal()              // buf destroyed
            }
            // Generate session UUID
            sessionID := uuid.New().String()
            llmService.sessionID = sessionID

            auditLogger, _ := audit.NewAuditLogger(filepath.Join(homeDir, ".pairadmin", "logs"))
            llmService.auditLogger = auditLogger
            commands.auditLogger = auditLogger
            auditLogger.Write(audit.AuditEntry{Event: "session_start", SessionID: sessionID})

            // existing startup calls...
            app.startup(ctx)
            commands.Startup(ctx)
            llmService.Startup(ctx)
        },
        OnBeforeClose: func(ctx context.Context) bool {
            auditLogger.Write(audit.AuditEntry{Event: "session_end", SessionID: sessionID})
            memguard.Purge() // destroys all enclaves; call last
            return false
        },
    })
}
```

### Using Enclave in buildProvider (HTTP header construction)

```go
// Source: CONTEXT.md D-01 + memguard Enclave.Open() API

// In LLMService (or passed to buildProvider):
func (s *LLMService) getAPIKeyString() string {
    if s.apiKeyEnclave == nil {
        return s.cfg.OpenAIKey // fallback to env var
    }
    buf, err := s.apiKeyEnclave.Open()
    if err != nil {
        return ""
    }
    key := string(buf.Bytes()) // Go string copy; lives until GC
    buf.Destroy()              // wipe LockedBuffer immediately
    return key
}
```

### Response-Side Credential Scan in SendMessage

```go
// Source: CONTEXT.md D-08, D-09 + existing filter.CredentialFilter.Apply() pattern

// In SendMessage goroutine, add accumulation:
var assembledParts []string // add alongside existing `batch`

// In chunk handling:
case chunk, ok := <-ch:
    if !ok {
        flush()
        // Assemble full response for audit
        assembled := strings.Join(assembledParts, "")
        s.writeAIResponseAudit(tabId, assembled)
        // ...emit llm:done...
        return
    }
    if !chunk.Done && chunk.Error == nil {
        batch = append(batch, chunk.Text)
        assembledParts = append(assembledParts, chunk.Text) // accumulate
    }

// Helper:
func (s *LLMService) writeAIResponseAudit(tabId, assembled string) {
    if s.auditLogger == nil {
        return
    }
    credFilter, err := filter.NewCredentialFilter()
    if err != nil {
        return
    }
    filtered, _ := credFilter.Apply(assembled)
    s.auditLogger.Write(audit.AuditEntry{
        Event:      "ai_response",
        SessionID:  s.sessionID,
        TerminalID: tabId,
        Content:    filtered,
    })
}
```

### Expected JSONL Output

```json
{"time":"2026-03-30T10:00:00.123456789Z","level":"INFO","msg":"audit","event":"session_start","session_id":"550e8400-e29b-41d4-a716-446655440000","terminal_id":"","content":""}
{"time":"2026-03-30T10:01:23.456Z","level":"INFO","msg":"audit","event":"user_message","session_id":"550e8400-e29b-41d4-a716-446655440000","terminal_id":"tmux:%3","content":"how do I list open ports?"}
{"time":"2026-03-30T10:01:24.789Z","level":"INFO","msg":"audit","event":"ai_response","session_id":"550e8400-e29b-41d4-a716-446655440000","terminal_id":"tmux:%3","content":"You can use: ss -tlnp or netstat -tlnp"}
{"time":"2026-03-30T10:01:30.000Z","level":"INFO","msg":"audit","event":"command_copied","session_id":"550e8400-e29b-41d4-a716-446655440000","terminal_id":"tmux:%3","content":"ss -tlnp"}
```

---

## Security Review Checklist (for Phase 6 Exit Criteria)

Items to verify as part of the security checklist task:

### Filter Pipeline Coverage (SEC-04 / D-10)
- [ ] `credentialPatterns` in `services/llm/filter/credential.go` covers: AWS access key IDs, GitHub tokens (`ghp_`, `github_pat_`), OpenAI keys (`sk-`, `sk-proj-`), Anthropic keys (`sk-ant-`), Bearer tokens, generic API keys
- [ ] `filter_test.go` has passing tests for: AWS key redaction, GitHub token redaction, bearer token redaction, safe text unchanged, pipeline ordering (ANSI + credential)
- [ ] Missing patterns to check against REQUIREMENTS.md FILT-02: GCP service account keys, SSH private key blocks, database DSN passwords, password prompt lines — currently NOT covered in `credentialPatterns`. Checklist should note these gaps.

### Ollama Remote-Host Guard (D-10)
- [ ] `validateOllamaHost("")` returns nil (empty host = default localhost)
- [ ] `validateOllamaHost("http://localhost:11434")` returns nil
- [ ] `validateOllamaHost("http://127.0.0.1:11434")` returns nil
- [ ] `validateOllamaHost("http://[::1]:11434")` returns nil
- [ ] `validateOllamaHost("http://remotehost:11434")` returns error
- [ ] `validateOllamaHost("http://192.168.1.100:11434")` returns error
- [ ] Error message is user-friendly: "OLLAMA_HOST must be localhost or 127.0.0.1; remote hosts are not allowed" (confirmed in `ollama.go` line 36)
- [ ] All above tests exist and pass in `services/llm/ollama_test.go` (confirmed — 8 tests, all passing)

### No Secrets in Logs or Config Files
- [ ] `~/.pairadmin/config.yaml` does not contain API key fields (confirmed by `config.AppConfig` struct — no key fields)
- [ ] Audit log `user_message` content = user chat text only, not terminal context
- [ ] Audit log `ai_response` content = filtered response (not raw)
- [ ] No API key value appears in any slog output

---

## Existing Code Insights (Critical for Planning)

### What Already Exists and Must Not Be Duplicated

| Asset | Location | Status |
|-------|----------|--------|
| `CredentialFilter` + `Apply()` | `services/llm/filter/credential.go` | Complete — reuse directly for response scan |
| `Filter` interface + `Pipeline` | `services/llm/filter/filter.go` | Complete — `Pipeline.Apply()` for chained filters |
| `validateOllamaHost()` | `services/llm/ollama.go` | Complete — checklist review only, no code change |
| Ollama host validation tests | `services/llm/ollama_test.go` | 8 tests, all passing — verify completeness only |
| `emitFn` injectable pattern | `services/settings_service.go` | Pattern reference for `AuditLogger` injection |
| `google/uuid` v1.6.0 | `go.mod` | Already present — `uuid.New().String()` available |

### What Does NOT Exist and Must Be Created

| Item | Where | Notes |
|------|-------|-------|
| `services/audit/` package | New directory | `AuditLogger`, `AuditEntry`, `NewAuditLogger()` |
| `auditLogger` field on `LLMService` | `services/llm_service.go` | + `sessionID string` field |
| `auditLogger` field on `CommandService` | `services/commands.go` | + `sessionID string` field |
| `apiKeyEnclave *memguard.Enclave` field | `LLMService` or Config path | Where enclave lives between keychain load and header build |
| `memguard.CatchInterrupt()` call | `main()` | Before any services created |
| `memguard.Purge()` call | `OnBeforeClose` | Last action before returning `false` |
| Session UUID generation | `OnStartup` | `uuid.New().String()` |

### LLMService Config / buildProvider Dependency Note

The current `buildProvider(cfg Config)` receives `cfg.OpenAIKey` etc. as plain strings. To pass the Enclave cleanly, either:
1. Add an `*memguard.Enclave` field to `Config` struct (modifies the type used broadly), OR
2. Add the Enclave as a separate field on `LLMService` and have `buildProvider` / `getAPIKeyString()` read from it.

**Recommendation (Claude's discretion):** Option 2 — add `apiKeyEnclave map[string]*memguard.Enclave` to `LLMService`. Providers are built inside `buildProvider(cfg)` which currently receives plain-string keys; update `buildProvider` to accept an additional `enclave *memguard.Enclave` parameter, or make providers accept a key-getter func. Keeping `Config` struct clean avoids pulling `memguard` as a transitive type dependency into test helpers.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Go 1.24.11 | All Go builds | Yes | 1.24.11 (from go.mod) | — |
| `github.com/awnumar/memguard` | SEC-01 | Not yet in go.mod | v0.23.0 | `go get` required |
| `gopkg.in/natefinch/lumberjack.v2` | SEC-02 | Not yet in go.mod | v2.2.1 | `go get` required |
| `github.com/google/uuid` | session UUID | Already in go.mod | v1.6.0 | — |
| `log/slog` | SEC-02 | stdlib (Go 1.21+) | — | — |
| Linux `mlock` syscall | memguard | Available on Linux 6.17 | — | memguard degrades gracefully if mlock limit is 0 |

**Missing dependencies with no fallback:**
- `github.com/awnumar/memguard@v0.23.0` — required for SEC-01; Wave 0 task must `go get` this
- `gopkg.in/natefinch/lumberjack.v2@v2.2.1` — required for SEC-02; Wave 0 task must `go get` this

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Go testing stdlib (`go test`) |
| Config file | none — `go test ./...` from project root |
| Quick run command | `go test ./services/audit/... ./services/llm/... ./services/... -run TestAudit -v` |
| Full suite command | `go test ./...` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SEC-01 | memguard Enclave created from keychain key; bytes wiped; Destroy called on exit | unit | `go test ./services/... -run TestMemguard -v` | Wave 0 |
| SEC-01 | `getAPIKeyString()` returns plaintext briefly, destroys LockedBuffer | unit | `go test ./services/... -run TestGetAPIKeyString -v` | Wave 0 |
| SEC-02 | `AuditLogger.Write()` writes JSON line to file | unit | `go test ./services/audit/... -v` | Wave 0 |
| SEC-02 | `AuditLogger.Write()` is nil-safe | unit | `go test ./services/audit/... -run TestAuditLoggerNilSafe -v` | Wave 0 |
| SEC-02 | log file created at `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl` | integration | `go test ./services/audit/... -run TestNewAuditLogger -v` | Wave 0 |
| SEC-03 | `session_start` event emitted in OnStartup | unit | `go test ./services/... -run TestSessionStart -v` | Wave 0 |
| SEC-03 | `user_message` event content = user chat text only (not terminal ctx) | unit | `go test ./services/... -run TestUserMessageAudit -v` | Wave 0 |
| SEC-03 | `ai_response` event content is credential-filtered | unit | `go test ./services/... -run TestAIResponseAudit -v` | Wave 0 |
| SEC-03 | `command_copied` event emitted on CopyToClipboard | unit | `go test ./services/... -run TestCommandCopiedAudit -v` | Wave 0 |
| SEC-04 | Response-side filter redacts credential patterns in assembled LLM response | unit | `go test ./services/llm/filter/... -run TestCredentialFilter -v` (existing) | Exists |
| SEC-04 | Response-side filter applied to assembled response, not per-chunk | unit | `go test ./services/... -run TestResponseSideFilter -v` | Wave 0 |

### Sampling Rate

- **Per task commit:** `go test ./services/audit/... ./services/llm/... -v`
- **Per wave merge:** `go test ./...`
- **Phase gate:** `go test ./...` all green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `services/audit/audit_test.go` — covers SEC-02, SEC-03
- [ ] `services/audit/audit.go` — the package itself (created in Wave 1)
- [ ] `go get github.com/awnumar/memguard@v0.23.0` — required before SEC-01 implementation
- [ ] `go get gopkg.in/natefinch/lumberjack.v2@v2.2.1` — required before SEC-02 implementation

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `LockedBuffer` for long-term storage | `Enclave` for long-term; `LockedBuffer` only for transient read-out | memguard v0.2+ | Enclaves are encrypted at rest; LockedBuffers are plaintext in mlock'd pages |
| `memguard.NewLockedBufferFromBytes` | `memguard.NewBufferFromBytes(...).Seal()` | API stabilized at v0.22+ | Seal/Open is the idiomatic round-trip for Enclave management |
| `log/logrus` or `log/zerolog` | `log/slog` (stdlib) | Go 1.21 (August 2023) | No external dep needed for structured logging |

---

## Open Questions

1. **Where does the `*Enclave` field live — on `LLMService` or threaded through `buildProvider`?**
   - What we know: `buildProvider(cfg Config)` currently takes plain strings; `LLMService` owns the active provider
   - What's unclear: The cleanest change — modify `buildProvider` signature vs. add getter method on `LLMService`
   - Recommendation: Add `apiKeyEnclave map[string]*memguard.Enclave` to `LLMService`. Change `buildProvider` to accept an optional `keyFn func() string` or have the Provider structs store a key-getter. This keeps the `Config` struct unmodified and does not require memguard as a compile-time dependency in test helpers.

2. **Should `command_copied` audit entry include the originating question text?**
   - What we know: CONTEXT.md lists this as Claude's discretion
   - What's unclear: The `CopyToClipboard(text string)` method currently only has the command text; the question would require an additional parameter
   - Recommendation: Include only the command text for now. Adding question context would require threading tabId + question through `CommandService.CopyToClipboard`, which is a larger change. Keep it simple.

3. **Filter pattern gaps: GCP keys, SSH blocks, DSN passwords, password prompts are in FILT-02 but not in `credentialPatterns`**
   - What we know: FILT-02 lists these as requirements; `credential.go` currently covers 6 patterns (AWS, GitHub, OpenAI, Anthropic, Bearer, generic API key)
   - What's unclear: Phase 6 scope — SEC-04 says "credential filter (lighter-weight pass)" for responses; it does not say to add new patterns
   - Recommendation: Document the gap in the security checklist. If new patterns are needed for SEC-04 coverage, add them to `credential.go` in a focused sub-task. Do not add patterns that were not in the original filter (e.g., GCP, SSH) unless the checklist review concludes they are needed.

---

## Sources

### Primary (HIGH confidence)

- `pkg.go.dev/github.com/awnumar/memguard@v0.23.0` — verified Enclave API, `NewBufferFromBytes`, `Seal()`, `Open()`, `Destroy()`, `Purge()`, `CatchInterrupt()`
- Go module proxy `proxy.golang.org/github.com/awnumar/memguard/@latest` — confirmed v0.23.0 (2025-08-27)
- Go module proxy `proxy.golang.org/gopkg.in/natefinch/lumberjack.v2/@latest` — confirmed v2.2.1 (2023-02-06)
- `go.mod` in this repo — confirmed `github.com/google/uuid v1.6.0` is already a dependency
- `services/llm/filter/credential.go`, `filter.go`, `custom.go` — read directly; confirmed `CredentialFilter.Apply()` interface
- `services/llm/ollama.go` + `ollama_test.go` — read directly; confirmed `validateOllamaHost()` implementation and 8 passing tests
- `services/llm_service.go` — read directly; confirmed `SendMessage` streaming architecture and `emitFn` injectable pattern
- `services/settings_service.go` — read directly; confirmed injectable field pattern (`emitFn`)
- `main.go` — read directly; confirmed `OnStartup`/`OnBeforeClose` hook locations
- `log/slog` documentation (stdlib, Go 1.21+) — `slog.NewJSONHandler`, `slog.HandlerOptions`
- `.planning/research/RESEARCH-SECURITY.md` — prior security research for this project

### Secondary (MEDIUM confidence)

- `gopkg.in/natefinch/lumberjack.v2` GitHub README — `lumberjack.Logger` struct fields (`Filename`, `MaxSize`, `MaxAge`, `Compress`)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — memguard v0.23.0 and lumberjack v2.2.1 verified via module proxy; google/uuid confirmed in go.mod
- Architecture: HIGH — all integration points verified by reading actual source files; memguard API verified against official docs
- Pitfalls: HIGH — streaming accumulation pitfall found by reading actual `SendMessage` implementation; Enclave vs LockedBuffer pitfall verified against memguard docs

**Research date:** 2026-03-30
**Valid until:** 2026-04-30 (memguard and lumberjack are stable; slog is stdlib)
