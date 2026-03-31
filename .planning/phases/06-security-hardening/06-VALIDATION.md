---
phase: 6
slug: security-hardening
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-30
---

# Phase 6 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | go test |
| **Config file** | none — standard Go testing |
| **Quick run command** | `go test ./services/... ./... -count=1 -timeout 30s` |
| **Full suite command** | `go test ./... -count=1 -timeout 60s` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `go test ./... -count=1 -timeout 30s`
- **After every plan wave:** Run `go test ./... -count=1 -timeout 60s`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 06-01-01 | 01 | 1 | SEC-01 | unit | `go test ./services/audit/... -run TestAuditLogger` | ❌ W0 | ⬜ pending |
| 06-01-02 | 01 | 1 | SEC-02/03 | unit | `go test ./services/audit/... -run TestAuditEntry` | ❌ W0 | ⬜ pending |
| 06-01-03 | 01 | 1 | SEC-01 | unit | `go test ./services/... -run TestMemguard` | ❌ W0 | ⬜ pending |
| 06-01-04 | 01 | 1 | SEC-04 | unit | `go test ./services/llm/... -run TestResponseFilter` | ❌ W0 | ⬜ pending |
| 06-02-01 | 02 | 2 | SEC-02/03 | integration | `go test ./... -run TestAuditIntegration` | ❌ W0 | ⬜ pending |
| 06-02-02 | 02 | 2 | SEC-01 | integration | `go test ./... -run TestMemguardLifecycle` | ❌ W0 | ⬜ pending |
| 06-03-01 | 03 | 3 | SEC-04 | unit | `go test ./services/llm/filter/... -count=1` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `services/audit/audit_test.go` — stubs for AuditLogger, AuditEntry (SEC-02/03)
- [ ] `services/audit/audit.go` — package skeleton so tests compile
- [ ] memguard and lumberjack dependencies added to go.mod before any tests run

*Existing `services/llm/filter/filter_test.go` covers SEC-04 pattern tests — no Wave 0 needed there.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| LockedBuffer memory pages are actually locked (mlock) | SEC-01 | Requires OS-level memory inspection; cannot automate in CI | Run app, attach gdb, verify pages marked with `PROT_READ\|PROT_WRITE` and locked |
| Audit JSONL files rotate at midnight | SEC-02 | Time-based rotation; simulating midnight in unit tests is impractical | Check `~/.pairadmin/logs/` after 24h or manually trigger lumberjack with `MaxSize=1` |
| session_start/session_end appear on real app launch/close | SEC-03 | Requires Wails runtime context | Launch app, close app, grep audit log for `session_start` and `session_end` entries |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
