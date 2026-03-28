---
gsd_state_version: 1.0
milestone: v2.0
milestone_name: milestone
current_plan: 03 of 4
status: Phase 2 in progress — Wave 2 frontend streaming layer (02-03 complete)
last_updated: "2026-03-27T22:53:00Z"
progress:
  total_phases: 7
  completed_phases: 1
  total_plans: 8
  completed_plans: 6
---

# Project State: PairAdmin v2.0

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-25)

**Core value:** The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.
**Current focus:** Phase 02 — llm-gateway-streaming-chat

## Current Position

**Milestone:** v1.0 — Linux release
**Active phase:** 02-llm-gateway-streaming-chat
**Current plan:** 03 of 4 (completed 02-03)
**Next action:** Wave 2 continued — ChatPane wiring (02-04)
**Last session:** 2026-03-27T22:53:00Z

## Progress

| Phase | Status |
|-------|--------|
| 1 — Application Shell | green Complete (4/4 plans) |
| 2 — LLM Gateway | 🟡 In progress (3/4 plans) |
| 3 — tmux Capture | ⬜ Not started |
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
*Last updated: 2026-03-28 — Wave 1 complete: 02-01 (LLM gateway + 5 adapters) + 02-02 (filter pipeline ANSI + credential)*
