# PairAdmin v2.0

## What This Is

PairAdmin is a standalone cross-platform desktop application that enables "pair administration" — a collaboration model where human sysadmins work alongside an AI assistant in the terminal. It automatically reads terminal session content via platform-native APIs and presents a chat interface where users can ask questions about what's happening in their terminal, get command suggestions, and execute those suggestions with a single click.

## Core Value

The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.

## Requirements

### Validated

- [x] Real-time AI chat with full terminal context injected into every message — Validated in Phase 2: LLM Gateway
- [x] Pre-LLM sensitive data filtering (passwords, API keys, tokens, private keys) — Validated in Phase 2: filter pipeline (ANSI strip + credential redaction)
- [x] Multi-provider LLM support (OpenAI, Anthropic, Ollama/local) — Validated in Phase 2: 5 adapters (OpenAI, Anthropic, Ollama, OpenRouter, LM Studio); live-tested against LM Studio
- [x] Automatic terminal content capture via platform-native APIs (no manual copy/paste) — Validated in Phase 3: tmux adapter (500ms polling, FNV-64a dedup, ANSI filter, Wails events)
- [x] Multi-terminal tab management with isolated context per session — Validated in Phase 3: terminalStore addTab/removeTab/clearTabs + useTerminalCapture hook

### Active

- [ ] One-click "Copy to Terminal" to push suggested commands to clipboard
- [ ] Command history sidebar (reverse-chronological, click-to-reuse)
- [ ] Settings dialog (provider config, prompt extensions, hotkeys, appearance)
- [ ] Slash command interface (/model, /filter, /context, /clear, /export, /rename)
- [x] Secure API key storage via OS keychain — Validated in Phase 5: 99designs/keyring; keys never in config.yaml
- [x] Local audit log of all AI interactions — Validated in Phase 6: slog+lumberjack audit log, 5 event types, credential-filtered content
- [ ] tmux adapter (Linux/macOS — subprocess via `tmux capture-pane`)
- [x] Linux GUI terminal adapters (GNOME Terminal, Konsole via AT-SPI2) — Validated in Phase 4: AT-SPI2 adapter (D-Bus discovery, GetText capture, Konsole degraded badge, onboarding empty state)
- [x] Installable packages: AppImage + .deb/.rpm for Linux — Validated in Phase 7: nFPM .deb/.rpm, AppImage via appimagetool, SHA256SUMS, human-verified on Ubuntu 22.04/24.04 + Fedora 40

### Out of Scope

- macOS Terminal.app adapter — deferred; need hardware for QA validation
- Windows/PuTTY adapter — deferred; per-iteration Windows VM validation is impractical
- OCR fallback for terminal capture — deferred with Windows adapter
- SQLite persistence for chat history — nice-to-have, not v1
- Cloud sync or multi-machine support — out of scope entirely for v1

## Context

- **Architecture pivot from v1.0:** Original design embedded AI into PuTTY via source modification. Proved untenable due to Win32/modern UI incompatibility, unreliable I/O capture, and single-platform limitation. v2.0 is a clean-room redesign as a standalone observer application.
- **Platform strategy:** Linux-first for v1. tmux covers the majority of serious sysadmin workflows. AT-SPI2 covers non-tmux Linux desktop users. macOS and Windows deferred until hardware/VM access is available for proper QA.
- **Security is load-bearing:** PairAdmin reads terminal buffers which routinely contain credentials, tokens, and private keys. The pre-LLM filter pipeline is not optional — it must run before any content reaches a cloud API. Local model support (Ollama) must be a first-class option for users with strict data residency requirements.
- **tmux is the priority adapter:** No special permissions required, works over SSH, well-documented API. Most sysadmins doing serious work are already in tmux.

## Constraints

- **Tech Stack:** Go 1.21+ backend, Wails v2 GUI framework, React + TypeScript frontend — chosen for native webview quality (needed for syntax-highlighted chat UI), TypeScript compatibility with Wails Go bindings, and rich React ecosystem for streaming chat patterns
- **Polling Interval:** 500ms terminal capture interval — balances responsiveness vs CPU overhead
- **Context Window:** Terminal content truncated to fit within LLM context limits; most recent content prioritized
- **Permissions (Linux):** AT-SPI2 requires accessibility enabled (`gsettings set org.gnome.desktop.interface accessibility true`); must detect and guide users through this
- **Permissions (macOS, future):** Accessibility API requires explicit user grant via System Preferences — must handle gracefully with onboarding flow

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Wails v2 over Fyne | Chat-heavy UI with markdown, syntax highlighting, and code blocks is far better served by web rendering than a native widget toolkit | — Pending |
| React + TypeScript over Vue | Wails generates TypeScript bindings; TS+React maximizes type safety at the Go↔JS boundary; stronger ecosystem for streaming chat UI patterns | — Pending |
| Linux-first scope for v1 | macOS hardware unavailable; Windows per-iteration VM QA impractical; tmux + AT-SPI2 covers a complete, shippable target audience | — Pending |
| tmux as primary adapter | No special permissions, works over SSH, reliable subprocess API; covers the majority of serious sysadmin workflows | — Pending |
| Pre-LLM filter pipeline mandatory | Terminal buffers routinely contain credentials; filtering cannot be optional or user-skippable for cloud providers | Validated Phase 2 — ANSI strip + credential regex pipeline in `services/llm/filter/` |
| OS keychain for API key storage | Plaintext config files are unacceptable for credentials; OS keychain is the correct abstraction across all platforms | Validated Phase 5 — 99designs/keyring; API keys never in ~/.pairadmin/config.yaml |
| memguard for in-memory key protection | Post-keychain retrieval, plain strings in process memory are a risk; memguard mlock pages prevent swapping and core dump exposure | Validated Phase 6 — Enclave lifecycle: seal at startup, open only in HTTP header builder stack frame, Purge() on exit |
| Audit log for interaction traceability | Sysadmin tool handling credentials needs a local audit trail; slog+lumberjack provides structured, rotated, credential-filtered JSONL logs | Validated Phase 6 — 5 event types (session_start/end, user_message, ai_response, command_copied); response-side filter on ai_response entries |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-03 — Phase 7 complete: nFPM .deb/.rpm, AppImage via appimagetool, SHA256SUMS, scripts/install-deps.sh, README with install instructions; human-verified clean install on Ubuntu 22.04/24.04 + Fedora 40; v1-approved — all 7 phases complete, PairAdmin v1.0 ready for GitHub release*
