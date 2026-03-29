---
phase: 4
slug: linux-gui-terminal-adapters-at-spi2
status: draft
nyquist_compliant: false
wave_0_complete: true
created: 2026-03-29
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | go test (backend) + vitest (frontend) |
| **Config file** | go.mod / frontend/package.json |
| **Quick run command** | `go test ./services/capture/... -count=1 -timeout 30s` |
| **Full suite command** | `go test ./... -count=1 -timeout 60s && cd frontend && npx vitest run` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `go test ./services/capture/... -count=1 -timeout 30s`
- **After every plan wave:** Run `go test ./... -count=1 && cd frontend && npx vitest run`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 04-01-01 | 01 | 1 | — | unit | `go test ./services/capture/... -run TestCaptureManager` | TDD | pending |
| 04-01-02 | 01 | 1 | — | unit | `go test ./services/capture/... -run TestTmuxAdapter` | TDD | pending |
| 04-02-01 | 02 | 2 | ATSPI-01 | unit | `go test ./services/capture/... -run TestAtSpiEnabled` | TDD | pending |
| 04-02-02 | 02 | 2 | ATSPI-02 | unit | `go test ./services/capture/... -run TestAtSpiDiscover` | TDD | pending |
| 04-02-03 | 02 | 2 | ATSPI-03 | unit | `go test ./services/capture/... -run TestAtSpiCapture` | TDD | pending |
| 04-03-01 | 03 | 3 | ATSPI-04 | unit | `go test ./services/capture/... -run TestKonsoleSpike` | TDD | pending |
| 04-04-01 | 04 | 4 | FILT-04 | unit | `go test ./services/... -run TestFilterAdd` | TDD | pending |
| 04-04-02 | 04 | 4 | FILT-05 | unit | `go test ./services/... -run TestFilterListRemove` | TDD | pending |
| 04-04-03 | 04 | 4 | FILT-04 | unit | `cd frontend && npx vitest run --reporter=verbose src/hooks/__tests__/useSlashCommand` | TDD | pending |
| 04-04-04 | 04 | 4 | FILT-04 | unit | `go test ./services/capture/... -run TestCaptureManager.*Filter` | TDD | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

All plans use TDD-first task ordering: test files are created as the first step within each task's RED phase before production code is written. No separate Wave 0 stub creation is needed.

- [x] Test files are created during TDD RED phase within each task
- [x] `go test ./services/capture/...` will compile once task execution begins (test file created before production code)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| GNOME Terminal tabs appear within 500ms | ATSPI-02/03 | Requires live AT-SPI2 bus + gnome-terminal running | Enable gsettings toolkit-accessibility, open GNOME Terminal, launch app, verify tab appears |
| Accessibility onboarding empty state | ATSPI-01 | Requires gsettings=false state | Disable toolkit-accessibility, launch app, verify empty state shows gsettings command |
| Konsole spike result | ATSPI-04 | Requires Konsole installed | Install Konsole, run spike, document outcome |
| /filter patterns survive app restart | FILT-04 | Requires config file persistence | Add pattern, restart app, run /filter list, verify pattern present |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references (TDD-first ordering sufficient)
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
