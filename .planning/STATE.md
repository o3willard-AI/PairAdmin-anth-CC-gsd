---
gsd_state_version: 1.0
milestone: v2.0
milestone_name: milestone
status: Ready to execute
last_updated: "2026-03-28T05:31:33.538Z"
progress:
  total_phases: 7
  completed_phases: 1
  total_plans: 8
  completed_plans: 4
---

# Project State: PairAdmin v2.0

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-25)

**Core value:** The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.
**Current focus:** Phase 02 — llm-gateway-streaming-chat

## Current Position

Phase: 02 (llm-gateway-streaming-chat) — EXECUTING
Plan: 3 of 4

## Progress

| Phase | Status |
|-------|--------|
| 1 — Application Shell | green Complete (4/4 plans) |
| 2 — LLM Gateway | ⬜ Not started |
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
- **gitleaks** as credential pattern foundation
- **ANSI stripping is Stage 1** of filter pipeline (security, not cosmetic)
- **vitest 4.x constructor mocks** require class syntax, not vi.fn().mockImplementation
- **CanvasAddon must load after term.open()** — enforced in TerminalPreview (plan 01-02)
- **Injectable lookPath var** for exec.LookPath enables CommandService Wayland tests without interfaces (01-03)
- **main.go OnStartup closure** calls app.startup and commands.Startup in sequence to support multiple service lifetimes (01-03)
- **TooltipTrigger asChild not used** — @base-ui/react TooltipTrigger renders its own button element; nesting a `<button>` inside via asChild creates invalid HTML (button-in-button); pass className/onClick directly on TooltipTrigger instead
- **useWailsClipboard dynamic import** — wailsjs/go bindings are gitignored (generated at wails dev runtime); dynamic import with navigator.clipboard fallback avoids build-time failure
- **@testing-library/dom required** — missing peer dep for @testing-library/react; must be installed explicitly

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
*Last updated: 2026-03-27 — 01-04 complete, human visual verification approved, Phase 1 done*
