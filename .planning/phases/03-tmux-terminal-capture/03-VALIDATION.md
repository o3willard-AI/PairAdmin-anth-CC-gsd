---
phase: 03
slug: tmux-terminal-capture
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-28
---

# Phase 03 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | `testing` stdlib (Go) + vitest v4.1.2 (frontend) |
| **Config file** | none (Go stdlib) / `frontend/vite.config.ts` (vitest) |
| **Quick run command** | `go test ./services/... -run TestTerminal && cd frontend && npx vitest run --reporter=dot` |
| **Full suite command** | `go test ./services/... && cd frontend && npx vitest run` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `go test ./services/... -run TestTerminal && cd frontend && npx vitest run --reporter=dot`
- **After every plan wave:** Run `go test ./services/... && cd frontend && npx vitest run`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | TMUX-01 | unit | `go test ./services/... -run TestListPanes` | ❌ W0 | ⬜ pending |
| 03-01-02 | 01 | 1 | TMUX-01 | unit | `go test ./services/... -run TestListPanesNoTmux` | ❌ W0 | ⬜ pending |
| 03-01-03 | 01 | 1 | TMUX-02 | unit | `go test ./services/... -run TestCapturePane` | ❌ W0 | ⬜ pending |
| 03-01-04 | 01 | 1 | TMUX-03 | unit | `go test ./services/... -run TestPollNewPane` | ❌ W0 | ⬜ pending |
| 03-01-05 | 01 | 1 | TMUX-04 | unit | `go test ./services/... -run TestPollRemovedPane` | ❌ W0 | ⬜ pending |
| 03-01-06 | 01 | 1 | TMUX-05 | unit | `go test ./services/... -run TestDedup` | ❌ W0 | ⬜ pending |
| 03-01-07 | 01 | 1 | TMUX-05 | unit | `go test ./services/... -run TestDedupChanged` | ❌ W0 | ⬜ pending |
| 03-02-01 | 02 | 2 | TMUX-06 | unit | `cd frontend && npx vitest run src/stores/__tests__/terminalStore.test.ts` | ❌ W0 | ⬜ pending |
| 03-02-02 | 02 | 2 | TMUX-06 | unit | `cd frontend && npx vitest run src/stores/__tests__/terminalStore.test.ts` | ❌ W0 | ⬜ pending |
| 03-02-03 | 02 | 2 | TMUX-06 | unit | `cd frontend && npx vitest run src/components/__tests__/` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `services/terminal_test.go` — covers TMUX-01 through TMUX-05 (Go unit tests with injectable `execCommand` mock)
- [ ] `frontend/src/stores/__tests__/terminalStore.test.ts` — update existing file to test `addTab`, `removeTab`, `clearTabs`, and active-tab auto-switch behavior (TMUX-06)
- [ ] `frontend/src/components/__tests__/TerminalPreview.test.tsx` — no-tmux empty state rendering when `tabs` is empty

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Live tmux pane content appears in xterm preview | TMUX-02 | Requires running tmux session | Run `wails dev`, open tmux session, verify content appears in terminal preview within 500ms |
| New pane tab auto-appears in sidebar | TMUX-03 | Requires live tmux | Run app, create new tmux pane (`Ctrl-b %`), verify new tab appears within 500ms |
| Closed pane tab auto-disappears | TMUX-04 | Requires live tmux | Kill a pane, verify its tab is removed within 500ms |
| AI response references real terminal output | TMUX-06 | End-to-end requires LLM | Run a command in tmux, ask AI about it, verify response references the actual command output |
| No-tmux empty state visible | TMUX-01 | Visual rendering | Launch app without tmux running; verify instruction text appears in terminal panel |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
