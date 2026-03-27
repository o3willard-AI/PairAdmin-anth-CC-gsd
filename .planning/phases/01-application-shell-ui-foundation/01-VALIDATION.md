---
phase: 1
slug: application-shell-ui-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-26
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest (via Vite ecosystem; standard for Wails React-TS projects) + Go test |
| **Config file** | `frontend/vite.config.ts` (add `test` block) |
| **Quick run command** | `cd frontend && npm run test -- --run` |
| **Full suite command** | `cd frontend && npm run test -- --run && go test ./...` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd frontend && npm run test -- --run`
- **After every plan wave:** Run `cd frontend && npm run test -- --run && go test ./...`
- **Before `/gsd:verify-work`:** Full suite must be green + manual smoke test (wails dev launches, tabs clickable, echo response works, clipboard copy works)
- **Max feedback latency:** ~15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-shell-01 | scaffold | 0 | SHELL-01 | smoke | Manual: `wails dev` + visual check | ❌ Wave 0 | ⬜ pending |
| 1-shell-02 | layout | 1 | SHELL-02 | unit | `cd frontend && npm run test -- --run layout` | ❌ Wave 0 | ⬜ pending |
| 1-shell-03 | status-bar | 2 | SHELL-03 | unit | `cd frontend && npm run test -- --run status-bar` | ❌ Wave 0 | ⬜ pending |
| 1-shell-04 | build | 3 | SHELL-04 | smoke | `wails build -tags webkit2_41` (exit 0) | ❌ Wave 0 | ⬜ pending |
| 1-chat-01 | chat | 1 | CHAT-01 | unit | `cd frontend && npm run test -- --run chat-input` | ❌ Wave 0 | ⬜ pending |
| 1-chat-05 | chat | 1 | CHAT-05 | unit | `cd frontend && npm run test -- --run chat-store` | ❌ Wave 0 | ⬜ pending |
| 1-chat-06 | chat | 1 | CHAT-06 | unit | `cd frontend && npm run test -- --run chat-store` | ❌ Wave 0 | ⬜ pending |
| 1-cmd-01 | commands | 1 | CMD-01 | unit | `cd frontend && npm run test -- --run command-store` | ❌ Wave 0 | ⬜ pending |
| 1-cmd-02 | commands | 1 | CMD-02 | unit | `cd frontend && npm run test -- --run command-store` | ❌ Wave 0 | ⬜ pending |
| 1-cmd-03 | commands | 1 | CMD-03 | unit | `cd frontend && npm run test -- --run command-card` | ❌ Wave 0 | ⬜ pending |
| 1-cmd-04 | commands | 1 | CMD-04 | unit | `cd frontend && npm run test -- --run command-card` | ❌ Wave 0 | ⬜ pending |
| 1-cmd-05 | commands | 1 | CMD-05 | unit | `cd frontend && npm run test -- --run command-store` | ❌ Wave 0 | ⬜ pending |
| 1-clip-01 | clipboard | 2 | CLIP-01 | unit | `go test ./services/... -run TestCopyToClipboard` | ❌ Wave 0 | ⬜ pending |
| 1-clip-02 | clipboard | 2 | CLIP-02 | unit | `go test ./services/... -run TestWaylandDetection` | ❌ Wave 0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `frontend/src/stores/__tests__/chatStore.test.ts` — stubs for CHAT-05, CHAT-06
- [ ] `frontend/src/stores/__tests__/commandStore.test.ts` — stubs for CMD-01, CMD-02, CMD-05
- [ ] `frontend/src/components/__tests__/ChatInput.test.tsx` — stubs for CHAT-01
- [ ] `frontend/src/components/__tests__/CommandCard.test.tsx` — stubs for CMD-03, CMD-04
- [ ] `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx` — stubs for SHELL-02
- [ ] `services/commands_test.go` — stubs for CLIP-01, CLIP-02
- [ ] Vitest install: `npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom`
- [ ] `frontend/vite.config.ts` test block: `test: { environment: "jsdom", globals: true }`

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| App launches as native window | SHELL-01 | No headless webview testing standard for Wails v2 | Run `wails dev`; confirm window appears with correct title "PairAdmin" |
| Build succeeds on Ubuntu 24.04 | SHELL-04 | Build smoke test requires actual build toolchain | Run `wails build -tags webkit2_41`; confirm exit 0 and binary in `build/bin/` |
| xterm.js renders in WebKit2GTK | SHELL-02 (partial) | WebGL/Canvas renderer behavior in WebKit2GTK requires visual check | Launch app; confirm terminal preview area shows "No terminal connected" message |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
