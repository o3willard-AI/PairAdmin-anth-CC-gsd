---
phase: 02-llm-gateway-streaming-chat
plan: "03"
subsystem: frontend-state
tags: [zustand, streaming, xterm, wails-events, tdd, hooks]
dependency_graph:
  requires:
    - 02-01  # Wails event contract (llm:chunk/done/error/usage names locked)
    - 01-01  # chatStore and terminalStore scaffolded in Phase 1
  provides:
    - chatStore streaming actions (startStreamingMessage, appendChunk, finalizeMessage, setStreamError)
    - terminalStore terminal ref registry (setTermRef, getTermRef)
    - readTerminalLines xterm buffer utility
    - useLLMStream Wails event subscription hook
  affects:
    - 02-04  # ChatPane will consume useLLMStream + chatStore streaming actions
tech_stack:
  added:
    - react-shiki (syntax highlighting with streaming delay prop)
    - react-markdown (markdown rendering for chat messages)
  patterns:
    - Zustand Immer mutation for streaming message append
    - External Map for non-serializable xterm Terminal refs (not in Zustand state)
    - Dynamic Wails runtime import with seq reorder buffer
    - vi.mock virtual module pattern for Wails runtime in vitest
key_files:
  created:
    - frontend/src/hooks/useLLMStream.ts
    - frontend/src/utils/terminalContext.ts
    - frontend/src/__mocks__/wailsjs/runtime/runtime.ts
    - frontend/wailsjs/runtime/runtime.js
    - frontend/wailsjs/runtime/runtime.d.ts
    - frontend/src/hooks/__tests__/useLLMStream.test.ts
  modified:
    - frontend/src/stores/chatStore.ts
    - frontend/src/stores/terminalStore.ts
    - frontend/src/stores/__tests__/chatStore.test.ts
    - .gitignore
decisions:
  - "vi.mock path must be relative to test file and resolve to the SAME absolute path that Vite resolves the hook's import to — test at __tests__/ level needs ../../../wailsjs/runtime/runtime to match hook's ../../wailsjs/runtime/runtime"
  - "wailsjs/runtime stub committed with .gitignore exception — dynamic import path must be physically resolvable; /* @vite-ignore */ only suppresses Vite warnings, not vitest import analysis"
  - "termRefsMap kept outside Zustand store — xterm Terminal objects are not serializable; store exposes setTermRef/getTermRef as methods backed by external Map"
  - "setStreamError with existing msgId appends \\n\\n(stream interrupted) to preserve partial content per CONTEXT.md decision"
metrics:
  duration_seconds: 711
  completed_date: "2026-03-27"
  tasks_completed: 2
  files_created: 6
  files_modified: 4
---

# Phase 02 Plan 03: Frontend Streaming State Layer Summary

Extended the frontend state layer for real LLM streaming: chatStore gains 4 streaming actions with ▋ cursor management, terminalStore gains external-Map-backed terminal ref registry, readTerminalLines reads xterm active buffers, and useLLMStream subscribes to Wails events with a sequence-number reorder buffer — all TDD-tested before ChatPane wiring.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 (RED) | Write failing tests for chatStore streaming and useLLMStream | 451f1fc | chatStore.test.ts (extended), useLLMStream.test.ts (new) |
| 2 (GREEN) | Implement all 4 files + stubs + gitignore fix | c193ac0 | chatStore.ts, terminalStore.ts, terminalContext.ts, useLLMStream.ts, wailsjs stubs, .gitignore |

## What Was Built

**chatStore extensions** (`frontend/src/stores/chatStore.ts`):
- `ChatMessage` interface: added `tokenCount?: number` and `isError?: boolean`
- `startStreamingMessage(tabId)` — creates `{role:"assistant", content:"", isStreaming:true}`, returns id
- `appendChunk(tabId, msgId, text)` — strips trailing ▋, appends text, re-appends ▋ cursor
- `finalizeMessage(tabId, msgId, tokenCount?)` — strips trailing ▋, sets `isStreaming:false`, optionally sets `tokenCount`
- `setStreamError(tabId, msgId|null, errorText)` — null creates new error message; existing msgId appends `\n\n(stream interrupted)` and sets `isError:true`

**terminalStore extensions** (`frontend/src/stores/terminalStore.ts`):
- `setTermRef(tabId, term)` and `getTermRef(tabId)` — backed by `termRefsMap` (Map outside Zustand) to avoid serializing xterm Terminal objects

**terminalContext utility** (`frontend/src/utils/terminalContext.ts`):
- `readTerminalLines(term, maxLines=200)` — reads xterm active buffer, trims trailing empty lines, returns joined string

**useLLMStream hook** (`frontend/src/hooks/useLLMStream.ts`):
- Subscribes to `llm:chunk`, `llm:done`, `llm:error` via dynamic Wails import
- Reorder buffer: `pendingRef` Map keyed by seq; flushes in-order when next seq arrives
- Creates streaming message on first chunk; finalizes on done; handles error with msgId or null

## Test Results

```
Test Files  7 passed (7)
Tests       39 passed (39)  [26 Phase 1 + 7 chatStore streaming + 6 useLLMStream]
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Wails runtime stub for vitest import resolution**
- **Found during:** Task 2 (implementing useLLMStream)
- **Issue:** `/* @vite-ignore */` suppresses Vite build warnings but NOT vitest's import analysis phase — dynamic import to `../../wailsjs/runtime/runtime` failed resolution since the file doesn't exist (gitignored, generated at Wails dev time)
- **Fix:** Created `frontend/wailsjs/runtime/runtime.js` + `.d.ts` stub files; added `.gitignore` exception (`!frontend/wailsjs/runtime/runtime.js`, `!frontend/wailsjs/runtime/runtime.d.ts`) to commit them; updated test `vi.mock` path to `"../../../wailsjs/runtime/runtime"` (correct relative path from `__tests__/` to resolve to same absolute path as hook's `../../wailsjs/runtime/runtime`)
- **Files modified:** `.gitignore`, `frontend/wailsjs/runtime/runtime.js` (new), `frontend/wailsjs/runtime/runtime.d.ts` (new)
- **Commit:** c193ac0

**2. [Rule 2 - Missing] TypeScript cast for Wails EventsOn callback type**
- **Found during:** Task 2 TypeScript check
- **Issue:** useLLMStream handlers typed as `(event: {seq:number; text:string}) => void` are not assignable to EventsOn's `(...args: unknown[]) => void` callback parameter
- **Fix:** Added `as (...args: unknown[]) => void` cast at EventsOn call sites — correct since Wails runtime delivers typed payloads at runtime regardless of TS signature
- **Files modified:** `frontend/src/hooks/useLLMStream.ts`
- **Commit:** c193ac0

### Pre-existing Issue (Not Fixed)

`useWailsClipboard.ts(12): Cannot find module '../../wailsjs/go/services/CommandService'` — pre-existing from Phase 1; wailsjs/go bindings are also gitignored, same category as the runtime stub. Out of scope for this plan.

## Known Stubs

None — all streaming actions are fully implemented and tested with real behavior.

## Self-Check: PASSED

Files created/modified:
- [x] `frontend/src/stores/chatStore.ts` — FOUND
- [x] `frontend/src/stores/terminalStore.ts` — FOUND
- [x] `frontend/src/utils/terminalContext.ts` — FOUND
- [x] `frontend/src/hooks/useLLMStream.ts` — FOUND
- [x] `frontend/src/hooks/__tests__/useLLMStream.test.ts` — FOUND
- [x] `frontend/wailsjs/runtime/runtime.js` — FOUND
- [x] `frontend/wailsjs/runtime/runtime.d.ts` — FOUND

Commits:
- [x] 451f1fc — FOUND (test RED)
- [x] c193ac0 — FOUND (feat GREEN)
