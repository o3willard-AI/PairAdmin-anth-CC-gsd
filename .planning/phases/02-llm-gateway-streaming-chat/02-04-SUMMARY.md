---
phase: 02-llm-gateway-streaming-chat
plan: "04"
subsystem: ui
tags: [react, wails, streaming, react-shiki, react-markdown, zustand, xterm, llm]

# Dependency graph
requires:
  - phase: 02-llm-gateway-streaming-chat/02-01
    provides: LLMService Go struct with SendMessage, 5 provider adapters, Wails event emission
  - phase: 02-llm-gateway-streaming-chat/02-02
    provides: ANSI + credential filter pipeline (filter.NewPipeline, NewANSIFilter, NewCredentialFilter)
  - phase: 02-llm-gateway-streaming-chat/02-03
    provides: chatStore streaming actions, terminalStore.setTermRef/getTermRef, useLLMStream hook, readTerminalLines util
provides:
  - Full end-to-end streaming chat: ChatPane calls LLMService.SendMessage via Wails binding
  - react-shiki CodeBlock with syntax highlighting and conditional Copy to Terminal button
  - ChatMessageList with ReactMarkdown rendering, error bubbles (amber/red), auto-scroll
  - LLMService bound to Wails in main.go with Startup lifecycle
  - TerminalPreview registers xterm Terminal in terminalStore after term.open()
  - StatusBar displays token count from chatStore
  - LMSTUDIO_HOST env var for remote LM Studio endpoints
  - Filter pipeline (ANSI + credential) applied in SendMessage before building LLM context
affects:
  - Phase 3 (tmux capture) — terminal context read path already wired via readTerminalLines + setTermRef
  - Phase 5 (settings) — LMSTUDIO_HOST pattern establishes remote-endpoint config approach

# Tech tracking
tech-stack:
  added:
    - react-markdown (ESM, markdown rendering with custom component override)
    - react-shiki 0.9.2 (streaming-aware syntax highlighting with delay prop)
  patterns:
    - Dynamic Wails binding import: `import(/* @vite-ignore */ "../../wailsjs/go/services/LLMService")`
    - TDD: RED commit then GREEN commit per feature
    - Filter pipeline applied at SendMessage boundary (security gate before LLM context)
    - lastSentRef pattern for retry on rate-limit errors

key-files:
  created:
    - frontend/src/components/chat/CodeBlock.tsx
    - frontend/src/components/__tests__/CodeBlock.test.tsx
    - frontend/src/components/__tests__/ChatMessageList.test.tsx
  modified:
    - frontend/src/components/chat/ChatMessageList.tsx
    - frontend/src/components/chat/ChatPane.tsx
    - frontend/src/components/terminal/TerminalPreview.tsx
    - frontend/src/components/layout/StatusBar.tsx
    - main.go
    - services/llm_service.go

key-decisions:
  - "LMSTUDIO_HOST env var: allows remote LM Studio endpoints (not just localhost:1234); added to Config struct and LoadConfig()"
  - "react-markdown + react-shiki: ESM packages were in package.json from plan author but needed npm install — installed as part of this plan"
  - "Human verification confirmed: streaming works against live LM Studio qwen/qwen3.5-35b-a3b at 192.168.101.56 — 352 and 1651 chunk counts confirmed correct"
  - "Filter pipeline wired at SendMessage boundary — ANSI stripping + credential redaction applied before terminal context reaches LLM adapters"

patterns-established:
  - "Dynamic Wails import pattern for LLMService: same /* @vite-ignore */ approach as useWailsClipboard from Phase 1"
  - "lastSentRef for retry: useRef stores {text, terminalContext} so rate-limit retry can resend original payload"
  - "Conditional Copy to Terminal: isStreaming prop hides button mid-stream, shows after finalize"

requirements-completed:
  - LLM-01
  - LLM-02
  - LLM-03
  - LLM-04
  - LLM-06
  - LLM-07
  - CHAT-02
  - CHAT-03
  - CHAT-04
  - FILT-06
  - FILT-07
  - CMD-01

# Metrics
duration: 90min
completed: 2026-03-27
---

# Phase 2 Plan 04: Wire End-to-End Streaming Chat Summary

**Real LLM streaming chat end-to-end: LLMService bound to Wails, ChatPane wired to SendMessage, react-shiki code blocks with Copy to Terminal, error bubbles, token count in status bar — verified live against LM Studio qwen/qwen3.5-35b-a3b with 352–1651 streaming chunks**

## Performance

- **Duration:** ~90 min
- **Started:** 2026-03-27T22:53:00Z
- **Completed:** 2026-03-27T23:45:00Z
- **Tasks:** 3 (including human-verify checkpoint, approved)
- **Files modified:** 10

## Accomplishments

- CodeBlock component with react-shiki syntax highlighting; Copy to Terminal button hidden during streaming, visible after stream completes; clicking adds command to commandStore
- ChatMessageList upgraded to ReactMarkdown with custom code renderer (fenced blocks → CodeBlock), error bubble styling (amber/red + ⚠ icon), auto-scroll within 100px of bottom, retry button for rate-limit errors
- ChatPane fully wired: useLLMStream called at component top, LLMService.SendMessage called via dynamic Wails import, lastSentRef for retry, terminal context read from terminalStore via readTerminalLines
- LLMService bound in main.go Bind[] with llmService.Startup(ctx) called in OnStartup
- Filter pipeline (ANSI + credential) applied in services/llm_service.go SendMessage before terminal context reaches LLM adapters
- TerminalPreview registers xterm Terminal in terminalStore.setTermRef after term.open()
- StatusBar shows "Tokens: N" from last finalized message, "Tokens: —" when none
- LMSTUDIO_HOST env var added to Config struct and LoadConfig() for remote endpoint support
- Human verification APPROVED: streaming confirmed working, 50/50 frontend tests pass, all Go service tests pass

## Task Commits

Each task was committed atomically:

1. **Task 1: CodeBlock + ChatMessageList with tests (TDD RED+GREEN)** - `e3e8079` (feat)
2. **Task 2: Wire ChatPane, TerminalPreview, main.go, StatusBar** - `38e4e2f` (feat)
3. **Post-task fix: LMSTUDIO_HOST env var + install react-markdown/react-shiki** - `2f4c1ed` (fix)
4. **Pre-summary: requirements + roadmap update** - `ea8a296` (docs)

**Plan metadata:** (this commit — docs: complete plan)

_Note: Task 1 used TDD (RED commit then GREEN commit pattern)_

## Files Created/Modified

- `frontend/src/components/chat/CodeBlock.tsx` — react-shiki syntax highlighting, conditional Copy to Terminal button, calls commandStore.addCommand
- `frontend/src/components/chat/ChatMessageList.tsx` — ReactMarkdown renderer, CodeBlock for fenced code, error bubbles (amber/red), auto-scroll, retry button for rate-limit errors
- `frontend/src/components/chat/ChatPane.tsx` — wired to LLMService.SendMessage via dynamic Wails import, useLLMStream at component top, lastSentRef for retry
- `frontend/src/components/terminal/TerminalPreview.tsx` — calls terminalStore.setTermRef after term.open()
- `frontend/src/components/layout/StatusBar.tsx` — displays last token count from chatStore
- `frontend/src/components/__tests__/CodeBlock.test.tsx` — tests: streaming hides Copy button, click adds command, language prop forwarded
- `frontend/src/components/__tests__/ChatMessageList.test.tsx` — tests: user/assistant alignment, streaming cursor visible, error bubble styling, code block rendering, markdown bold
- `main.go` — LLMService added to Bind[] and Startup lifecycle
- `services/llm_service.go` — filter pipeline wired in SendMessage, LMSTUDIO_HOST env var support
- `frontend/package.json` / `frontend/package-lock.json` — react-markdown and react-shiki installed

## Decisions Made

- **LMSTUDIO_HOST env var:** Remote LM Studio endpoint support was discovered needed during live testing — user's LM Studio was at 192.168.101.56:1234 (not localhost). Added LMSTUDIO_HOST to Config struct and LoadConfig() using same env var pattern as other providers.
- **react-markdown + react-shiki needed npm install:** Both packages were listed in package.json from the plan author but had not been installed. Applied `npm install` in frontend/ as part of plan execution (Rule 3 — blocking).
- **Filter pipeline placement:** Applied at SendMessage boundary in Go service layer rather than in adapters — single choke point ensures no terminal context reaches any LLM provider without being filtered.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] npm install react-markdown and react-shiki**
- **Found during:** Task 1 (CodeBlock and ChatMessageList implementation)
- **Issue:** react-markdown and react-shiki were in package.json but not installed; import resolution would fail at build/test time
- **Fix:** Ran `npm install` in frontend/ to install all listed dependencies
- **Files modified:** frontend/package-lock.json
- **Verification:** vitest run passed (50/50) after install
- **Committed in:** 2f4c1ed

**2. [Rule 2 - Missing Critical] Added LMSTUDIO_HOST env var to Config struct**
- **Found during:** Task 2 post-verification (live testing)
- **Issue:** LM Studio provider in llm_service.go used hardcoded localhost:1234 base URL; user's LM Studio was on a remote host (192.168.101.56:1234)
- **Fix:** Added LMSTUDIO_HOST field to Config struct in services/llm_service.go; LoadConfig() reads it with default fallback to "http://localhost:1234"; LMStudio provider initialization uses cfg.LMStudioHost
- **Files modified:** services/llm_service.go
- **Verification:** Confirmed via live LM Studio streaming test (352 and 1651 chunk responses)
- **Committed in:** 2f4c1ed

---

**Total deviations:** 2 auto-fixed (1 blocking dependency install, 1 missing remote endpoint config)
**Impact on plan:** Both fixes necessary for the plan to work in real-world usage. No scope creep.

## Issues Encountered

- react-shiki and react-markdown were listed in package.json by the plan author but not installed — vitest would have failed on import resolution. Discovered and fixed before tests were run.
- LM Studio requires a configurable base URL for remote hosts — found during human verification with live endpoint at 192.168.101.56. LMSTUDIO_HOST env var added post-task as a targeted fix.

## User Setup Required

To use LM Studio at a remote host, set before `wails dev`:
```bash
export LMSTUDIO_HOST=http://192.168.101.56:1234
export PAIRADMIN_PROVIDER=lmstudio
export PAIRADMIN_MODEL=qwen/qwen3.5-35b-a3b
```

For local LM Studio (default), no additional config needed — defaults to http://localhost:1234.

## Known Stubs

None — all chat, streaming, code block, and token count features are wired to live data sources. No placeholder or mock data paths remain in the completed components.

## Next Phase Readiness

- Phase 2 complete: LLM gateway + streaming chat fully operational
- Phase 3 (tmux capture) can now wire real terminal output into the chat context via the terminalStore.setTermRef path already established
- The readTerminalLines utility in ChatPane is ready to receive actual tmux output once Phase 3 provides it
- No blockers for Phase 3

---
*Phase: 02-llm-gateway-streaming-chat*
*Completed: 2026-03-27*
