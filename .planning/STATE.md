# Project State: PairAdmin v2.0

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-25)

**Core value:** The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.
**Current focus:** Planning complete — ready for Phase 1

## Current Position

**Milestone:** v1.0 — Linux release
**Active phase:** None — not started
**Next action:** `/gsd:plan-phase 1`

## Progress

| Phase | Status |
|-------|--------|
| 1 — Application Shell | ⬜ Not started |
| 2 — LLM Gateway | ⬜ Not started |
| 3 — tmux Capture | ⬜ Not started |
| 4 — AT-SPI2 Adapters | ⬜ Not started |
| 5 — Settings & Config | ⬜ Not started |
| 6 — Security Hardening | ⬜ Not started |
| 7 — Distribution | ⬜ Not started |

## Key Decisions Locked

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
- **Ubuntu 24.04 build**: requires `-tags webkit2_41` — verify build machine config in Phase 1

## Todos

(none)

---
*Initialized: 2026-03-25*
*Last updated: 2026-03-25 after project initialization*
