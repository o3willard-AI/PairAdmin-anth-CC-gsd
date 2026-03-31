# Phase 6: Security Hardening — Discussion Log

**Session:** 2026-03-30
**Areas discussed:** memguard scope, Audit log content, Response filter approach, Audit log placement

---

## memguard scope

**Q: How deep should memguard protection go?**
Options: Keychain boundary only / Full provider wrapping / Keychain wrapper only
**Selected:** Keychain boundary only
> Lock the key in LockedBuffer immediately after keychain.Get(). Pass the sealed buffer to LLMService. Provider reads from it only when building HTTP headers.

**Q: When should LockedBuffer.Destroy() be called?**
Options: App exit only / After each HTTP call / On provider rebuild
**Selected:** App exit only
> One global buffer per provider, destroyed in OnBeforeClose.

---

## Audit log content

**Q: What does a user_message entry log?**
Options: Message text only, no context prefix / Full assembled prompt / Message hash + length only
**Selected:** Message text only, no context prefix

**Q: What does an ai_response entry log?**
Options: Full response text after stream completes / Per-chunk entries / First N chars + response hash
**Selected:** Full response text after stream completes

**Q: How should session and terminal IDs be generated?**
Options: Random UUID per app launch + tab ID / Timestamp-based IDs / Incremental counter
**Selected:** Random UUID per app launch + tab ID

---

## Response filter approach

**Q: What should the response-side filter do?**
Options: Reuse existing regex patterns, redact matches / Keyword-only scan, flag but don't redact / Keyword scan + redact on match
**Selected:** Reuse existing regex patterns, redact matches

**Q: When does the response filter run?**
Options: After stream completes, before audit log write / On each streamed chunk / Before audit log only, display unfiltered
**Selected:** After stream completes, before audit log write

---

## Audit log placement

**Q: How should the AuditLogger be wired to services?**
Options: Injectable field per service / Global singleton logger / Wails-bound AuditService
**Selected:** Injectable field per service

**Q: Where does session_start / session_end get emitted?**
Options: OnStartup / OnBeforeClose in main.go / SettingsService.Startup / LLMService first call
**Selected:** OnStartup / OnBeforeClose in main.go

---

*Generated: 2026-03-30*
