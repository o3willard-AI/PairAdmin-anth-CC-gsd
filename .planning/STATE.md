---
gsd_state_version: 1.0
milestone: v2.0
milestone_name: milestone
current_plan: 04-01
status: In progress
last_updated: "2026-03-29T05:00:00Z"
progress:
  total_phases: 7
  completed_phases: 3
  total_plans: 11
  completed_plans: 11
---

# Project State: PairAdmin v2.0

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-25)

**Core value:** The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.
**Current focus:** Phase 03 — tmux-terminal-capture

## Current Position

**Milestone:** v1.0 — Linux release
**Active phase:** 03-tmux-terminal-capture
**Current plan:** 04-01 (Phase 4 not yet started)
**Next action:** Begin Phase 4 — AT-SPI2 Adapters spike
**Last session:** 2026-03-29T05:00:00Z

## Progress

| Phase | Status |
|-------|--------|
| 1 — Application Shell | green Complete (4/4 plans) |
| 2 — LLM Gateway | green Complete (4/4 plans) |
| 3 — tmux Capture | green Complete (3/3 plans) |
| 4 — AT-SPI2 Adapters | ⬜ Not started |
| 5 — Settings & Config | ⬜ Not started |
| 6 — Security Hardening | ⬜ Not started |
| 7 — Distribution | ⬜ Not started |

## Key Decisions Locked

- **Vite v5 + TypeScript v5** required (Wails scaffold ships v3/4.6 but Tailwind v4 and @base-ui/react require upgrades)
- **shadcn/ui base-nova style** uses @base-ui/react (not Radix UI) — different peer dep tree than classic shadcn
- **frontend/.npmrc legacy-peer-deps=true** required for Wails build to install npm deps without conflicts
- **Wails v2** (not Fyne) — web rendering required for chat UI quality
- **React + TypeScript** (not Vue) — TypeScript binding compatibility with Wails Go codegen
- **Linux-first** — macOS and Windows deferred until hardware/VM access available
- **tmux as primary adapter** — no permissions needed, SSH-compatible
- **99designs/keyring** for OS keychain (not zalando — fails in headless Linux)
- **xterm.js** for terminal preview (direct writes, not React state)
- **Zustand + Immer** for frontend state (3 stores: chat, terminal, commands)
- **react-shiki** for syntax highlighting (streaming delay prop)
- **gitleaks** as credential pattern foundation (regex-only mode in filter package — gitleaks not added as go.mod dep; comprehensive regex patterns cover required credential formats)
- **ANSIFilter uses comprehensive regex** (not go-ansi-parser Cleanse) — library only handles SGR color codes; cursor movement and OSC sequences require full regex approach
- **ANSI stripping is Stage 1** of filter pipeline (security, not cosmetic)
- **vitest 4.x constructor mocks** require class syntax, not vi.fn().mockImplementation
- **CanvasAddon must load after term.open()** — enforced in TerminalPreview (plan 01-02)
- **Injectable lookPath var** for exec.LookPath enables CommandService Wayland tests without interfaces (01-03)
- **main.go OnStartup closure** calls app.startup and commands.Startup in sequence to support multiple service lifetimes (01-03)
- **TooltipTrigger asChild not used** — @base-ui/react TooltipTrigger renders its own button element; nesting a `<button>` inside via asChild creates invalid HTML (button-in-button); pass className/onClick directly on TooltipTrigger instead
- **useWailsClipboard dynamic import** — wailsjs/go bindings are gitignored (generated at wails dev runtime); dynamic import with navigator.clipboard fallback avoids build-time failure
- **@testing-library/dom required** — missing peer dep for @testing-library/react; must be installed explicitly
- **Anthropic buildParams internal test** — unexported method requires package llm (not package llm_test) for white-box testing
- **Ollama localhost-only enforcement** — OLLAMA_HOST must be localhost/127.0.0.1/::1; validated in NewOllamaProvider to prevent remote data leakage
- **OpenAI adapter covers 3 providers** — OpenRouter (custom BaseURL + key) and LM Studio (local BaseURL + empty key) reuse OpenAIProvider; no extra files needed
- **50ms Wails event batching** — mitigates Issue #2759 out-of-order delivery; sequence numbers allow frontend reordering
- **wailsjs/runtime stub committed with .gitignore exception** — `/* @vite-ignore */` only suppresses Vite warnings, not vitest import analysis; stub JS file at `frontend/wailsjs/runtime/runtime.js` must physically exist for vitest to resolve dynamic import path
- **vi.mock path must match resolved absolute path** — test at `__tests__/` must use `../../../wailsjs/runtime/runtime` to reach same absolute path that hook's `../../wailsjs/runtime/runtime` resolves to
- **termRefsMap outside Zustand** — xterm Terminal objects are not serializable; store exposes setTermRef/getTermRef as methods backed by external Map, no re-render on terminal ref changes
- **LMSTUDIO_HOST env var** — allows remote LM Studio endpoints (not just localhost:1234); added to Config struct and LoadConfig() (02-04)
- **react-markdown + react-shiki needed npm install** — both packages were in package.json from plan author but not installed; npm install required as part of 02-04 execution
- **Human verification confirmed live streaming** — LM Studio qwen/qwen3.5-35b-a3b at 192.168.101.56 returned 352 and 1651 chunk counts; streaming chat end-to-end verified (02-04)
- **TerminalService emits post-WaitGroup** — capture goroutines collect results into slice; emitFn called from main goroutine after wg.Wait(); avoids EventsEmit thread-safety concerns (03-01)
- **capturePane filter degradation** — if filter init/apply fails, returns unfiltered content rather than propagating error; terminal availability > filter failure at runtime (03-01)
- **Injectable emitFn field** — TerminalService.emitFn allows test isolation of Wails events without runtime; matches injectable lookPath/execCommand pattern (03-01)
- **Empty initial terminalStore state** — tabs start empty, populated dynamically via terminal:tabs Wails events; first addTab sets activeTabId (03-02)
- **removeTab active-tab auto-switch** — switches to first remaining tab or empty string per D-06; termRefsMap.delete() called outside Immer set for xterm cleanup (03-02)
- **useTerminalCapture hook pattern** — follows useLLMStream.ts: dynamic @vite-ignore import, EventsOn subscriptions, cleanup unsubscribe on unmount (03-02)
- **useTerminalCapture mounted in ThreeColumnLayout** — AppLayout.tsx doesn't exist; ThreeColumnLayout is the correct layout owner for terminal state (03-03)
- **TerminalPreview empty state is early return after hooks** — useEffect must be declared before any conditional return per React Rules of Hooks; tabId empty check placed after useEffect declaration (03-03)
- **useEffect must precede early return in TerminalPreview** — Rules of Hooks; early return guard must appear after all hook declarations (03-03)
- **ThreeColumnLayout test requires wailsjs/runtime mock when useTerminalCapture is mounted** — dynamic import of wailsjs/runtime fails in vitest without a vi.mock stub (03-03)

## Research Completed

| File | Domain |
|------|--------|
| `.planning/research/RESEARCH-WAILS.md` | Wails v2 + React ecosystem |
| `.planning/research/RESEARCH-TERMINAL-CAPTURE.md` | tmux + AT-SPI2 |
| `.planning/research/RESEARCH-LLM-GATEWAY.md` | LLM providers + streaming |
| `.planning/research/RESEARCH-SECURITY.md` | Credential filtering + keychain |
| `.planning/research/SUMMARY.md` | Cross-cutting synthesis |

## Open Questions / Risks

- **Konsole AT-SPI2**: text extraction unconfirmed — Phase 4 begins with a spike
- **GNOME Terminal GTK4**: reliability uncertain — needs validation during Phase 4
- **xterm.js WebGL in WebKit2GTK**: test canvas fallback during Phase 1
- **Wails v3**: in alpha — check status at Phase 7; API-incompatible with v2
- **Ubuntu 24.04 build**: requires `-tags webkit2_41` — CONFIRMED WORKING in Phase 1 plan 01

## Todos

(none)

---
*Initialized: 2026-03-25*
*Last updated: 2026-03-29 — Phase 3 complete: Plan 03-03 human-verified. Full tmux capture pipeline operational end-to-end. 62 frontend tests pass. Rules of Hooks fix and wailsjs/runtime vitest mock applied.*
