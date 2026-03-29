---
phase: 5
slug: settings-configuration-slash-commands
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | go test (backend) + vitest (frontend) |
| **Config file** | frontend/vite.config.ts (vitest), go.mod (Go) |
| **Quick run command** | `cd frontend && npx vitest run --reporter=verbose 2>&1 | tail -5` |
| **Full suite command** | `go test ./... 2>&1 | tail -20 && cd frontend && npx vitest run 2>&1 | tail -10` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd frontend && npx vitest run --reporter=verbose 2>&1 | tail -5`
- **After every plan wave:** Run `go test ./... 2>&1 | tail -20 && cd frontend && npx vitest run 2>&1 | tail -10`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 5-01-01 | 01 | 1 | CFG-08 | unit | `go test ./services/config/...` | ❌ W0 | ⬜ pending |
| 5-01-02 | 01 | 1 | CFG-02 | unit | `go test ./services/config/...` | ❌ W0 | ⬜ pending |
| 5-01-03 | 01 | 1 | CFG-03 | unit | `go test ./services/settings/...` | ❌ W0 | ⬜ pending |
| 5-02-01 | 02 | 1 | CFG-01 | unit | `cd frontend && npx vitest run` | ❌ W0 | ⬜ pending |
| 5-02-02 | 02 | 1 | CFG-07 | unit | `cd frontend && npx vitest run` | ❌ W0 | ⬜ pending |
| 5-03-01 | 03 | 2 | SLASH-01 | unit | `go test ./services/...` | ❌ W0 | ⬜ pending |
| 5-03-02 | 03 | 2 | SLASH-02..08 | unit | `cd frontend && npx vitest run` | ❌ W0 | ⬜ pending |
| 5-04-01 | 04 | 2 | CLIP-03 | unit | `go test ./services/...` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `services/config/config_test.go` — stubs for CFG-08 (AppConfig expansion, SaveAppConfig safe merge)
- [ ] `services/config/keychain_test.go` — stubs for CFG-02 (keyring read/write/delete)
- [ ] `services/settings/settings_test.go` — stubs for CFG-01, CFG-03, CFG-04, CFG-05, CFG-06, CFG-07
- [ ] `frontend/src/components/settings/__tests__/SettingsDialog.test.tsx` — stubs for CFG-01, CFG-07
- [ ] `frontend/src/components/chat/__tests__/ChatPane.test.tsx` — already exists; extend for new slash commands

*Existing test infrastructure (68 frontend tests, go test coverage for services) covers the test runner setup.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| API key stored in OS keychain (gnome-keyring) | CFG-02 | Requires live gnome-keyring daemon on test machine | 1. Open Settings → LLM Config, enter key, Save. 2. Run `secret-tool lookup service pairadmin account openai`. |
| Global hotkey triggers copy-last-command | CFG-06 | Wails v2 has no system-level global hotkey API; in-app only — requires focus | Press configured hotkey while PairAdmin is focused; verify last command copied to clipboard. |
| Connection test shows inline status | CFG-03 | Requires live LLM provider endpoint | Click Test Connection; verify spinner → green ✓ or red ✗ appears inline below button (not toast). |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
