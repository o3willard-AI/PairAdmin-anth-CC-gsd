---
phase: 02
slug: llm-gateway-streaming-chat
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-27
---

# Phase 02 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest (frontend) + go test (backend) |
| **Config file** | `frontend/vite.config.ts` (vitest) + standard go test |
| **Quick run command** | `cd frontend && npx vitest run --reporter=dot` |
| **Full suite command** | `cd frontend && npx vitest run && cd .. && go test ./services/... -v` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd frontend && npx vitest run --reporter=dot` + `go test ./services/...`
- **After every plan wave:** Run full suite command
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 1 | LLM-01 | unit | `go test ./services/llm/... -run TestProviderInterface` | ❌ W0 | ⬜ pending |
| 02-01-02 | 01 | 1 | LLM-02 | unit | `go test ./services/llm/... -run TestOpenAIAdapter` | ❌ W0 | ⬜ pending |
| 02-01-03 | 01 | 1 | LLM-03 | unit | `go test ./services/llm/... -run TestAnthropicAdapter` | ❌ W0 | ⬜ pending |
| 02-01-04 | 01 | 1 | LLM-04 | unit | `go test ./services/llm/... -run TestOllamaAdapter` | ❌ W0 | ⬜ pending |
| 02-02-01 | 02 | 1 | FILT-01 | unit | `go test ./services/filter/... -run TestANSIStrip` | ❌ W0 | ⬜ pending |
| 02-02-02 | 02 | 1 | FILT-02 | unit | `go test ./services/filter/... -run TestCredentialRedaction` | ❌ W0 | ⬜ pending |
| 02-03-01 | 03 | 2 | CHAT-02 | unit | `cd frontend && npx vitest run useLLMStream` | ❌ W0 | ⬜ pending |
| 02-03-02 | 03 | 2 | CHAT-03 | unit | `cd frontend && npx vitest run ChatBubble` | ❌ update | ⬜ pending |
| 02-03-03 | 03 | 2 | CHAT-04 | manual | visual: streaming cursor, scroll behavior | — | ⬜ pending |
| 02-04-01 | 04 | 3 | LLM-06 | unit | `cd frontend && npx vitest run CodeBlock` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `services/llm/provider_test.go` — interface + mock provider stubs
- [ ] `services/llm/openai_test.go` — OpenAI adapter unit tests (mocked HTTP)
- [ ] `services/llm/anthropic_test.go` — Anthropic adapter unit tests
- [ ] `services/llm/ollama_test.go` — Ollama adapter unit tests
- [ ] `services/filter/filter_test.go` — ANSI strip + credential redaction tests
- [ ] `frontend/src/hooks/__tests__/useLLMStream.test.ts` — frontend streaming hook tests
- [ ] `frontend/src/components/__tests__/CodeBlock.test.tsx` — code block Copy button tests

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Streaming cursor `▋` blinks while LLM responds | CHAT-02 | Visual animation requires human eye | Run `wails dev`, send a chat message, observe cursor |
| Auto-scroll follows stream when near bottom | CHAT-02 | Scroll physics require interaction testing | Send message while scrolled to bottom; verify scroll follows |
| Auto-scroll stops when user scrolls up | CHAT-02 | Interaction state | Scroll up, send a message, verify no scroll hijack |
| react-shiki highlights code blocks during stream | CHAT-03 | Visual rendering | Prompt LLM for code; verify syntax colors appear as it streams |
| Token count updates in status bar after response | LLM-07 | UI state binding | Send message; check status bar shows non-zero token count |
| Credential redaction prevents key transmission | FILT-02 | Requires live LLM to verify actual payload | Check with Ollama; include fake API key in terminal; verify not in LLM request |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
