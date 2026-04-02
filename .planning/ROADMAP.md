# Roadmap: PairAdmin v2.0

**Milestone:** v1.0 — Linux release
**Created:** 2026-03-25
**Phases:** 7

---

## Phase 1: Application Shell & UI Foundation

**Goal:** A working Wails + React desktop app with the three-column layout, mock terminal tabs, static chat UI, and clipboard support. No real terminal capture or LLM yet — but the skeleton is interactive and correct.

**Requirements:** SHELL-01, SHELL-02, SHELL-03, SHELL-04, CHAT-01, CHAT-05, CHAT-06, CMD-01��05, CLIP-01, CLIP-02

**Plans:** 4/4 plans complete

Plans:
- [x] 01-01-PLAN.md ��� Scaffold Wails project, configure Tailwind/shadcn/Vitest, create Zustand stores
- [x] 01-02-PLAN.md — Three-column layout, terminal tabs, xterm.js preview, status bar
- [x] 01-03-PLAN.md — Go clipboard service with Wayland detection
- [x] 01-04-PLAN.md — Chat UI with echo response, command sidebar with click-to-copy

**Key deliverables:**
- Wails v2 project scaffolded with React + TypeScript template; builds and launches on Ubuntu 22.04+
- Three-column layout with correct proportions (160px tabs / flex chat / 220px sidebar)
- Terminal tab component (mock data, active/inactive states, "+ New" placeholder)
- Chat area: message bubbles (user right, AI left), auto-expanding input, send button
- Command sidebar: reverse-chronological cards, hover tooltip, Clear History button
- Status bar with placeholder model selector, connection info, token meter
- xterm.js integrated into the terminal preview pane (direct writes, bypassing React state)
- Zustand + Immer: 3 stores (chat, terminal, commands) with granular selectors
- Clipboard copy to system clipboard; Wayland `wl-clipboard` detection and warning
- Build verified on Ubuntu 22.04 and 24.04 (`-tags webkit2_41`)

**Exit criteria:** App launches, tabs are clickable, user can type in the chat input and see a hardcoded echo response, clicking a mock command card copies text to clipboard.

---

## Phase 2: LLM Gateway & Streaming Chat

**Goal:** Real AI responses. The user can type a question, it goes to a real LLM, and the response streams back token-by-token into the chat area with working code block highlighting and "Copy to Terminal" buttons.

**Requirements:** LLM-01–07, FILT-01, FILT-02, FILT-03, FILT-06, FILT-07, CHAT-02, CHAT-03, CHAT-04

**Plans:** 4/4 plans complete

Plans:
- [x] 02-01-PLAN.md �� Go LLM provider interface, five provider adapters (OpenAI/OpenRouter/LM Studio/Anthropic/Ollama), LLMService with 50ms batching EventsEmit
- [x] 02-02-PLAN.md — Filter pipeline: ANSI stripping (leaanthony/go-ansi-parser) + credential redaction (gitleaks + fallback regex)
- [x] 02-03-PLAN.md — Frontend: chatStore streaming actions, terminalStore xterm ref registry, useLLMStream hook with reorder buffer
- [x] 02-04-PLAN.md — Full wiring: ChatPane to LLMService, CodeBlock react-shiki component, TerminalPreview ref registration, StatusBar token count, human verify

**Key deliverables:**
- Provider interface: `Stream() (<-chan StreamChunk, error)` + `Complete()` + `TestConnection()`
- OpenAI adapter (`github.com/openai/openai-go/v3`) with streaming
- Anthropic adapter (`github.com/anthropics/anthropic-sdk-go`) with system-prompt top-level field handling
- Ollama adapter (`github.com/ollama/ollama/api`) with callback-to-channel wrapping; `OLLAMA_HOST` localhost validation
- LM Studio / llama.cpp via OpenAI adapter with configurable `BaseURL`
- Wails EventsEmit streaming with sequence numbers + 50ms batching (mitigates Issue #2759); frontend-ready signal before first emit
- react-shiki syntax highlighting in code blocks with streaming `delay` prop
- "Copy to Terminal" button on every code block; adds command to sidebar store
- ANSI/VT100 stripping (Stage 1 of filter pipeline — security requirement)
- Built-in credential redaction patterns (AWS keys, GitHub tokens, API keys, SSH keys, DB DSNs, bearer tokens)
- Context truncation to provider token limit; token count displayed in status bar

**Exit criteria:** User can chat with OpenAI/Anthropic/Ollama; responses stream; code blocks have syntax highlighting and Copy button; credentials are redacted before transmission.

---

## Phase 3: tmux Terminal Capture

**Goal:** Automatic terminal content capture from tmux. No more mock data — the terminal preview pane shows live tmux output and every chat message is prefixed with real terminal context.

**Requirements:** TMUX-01–06

**Plans:** 3/3 plans complete

Plans:
- [x] 03-01-PLAN.md — Go TerminalService: tmux discovery, capture, 500ms polling, FNV-64a dedup, bounded concurrency, Wails event emission
- [x] 03-02-PLAN.md — Frontend: terminalStore addTab/removeTab/clearTabs actions, useTerminalCapture Wails event hook
- [x] 03-03-PLAN.md — UI wiring: TerminalPreview live content + no-tmux empty state, ThreeColumnLayout hook mount, human verification

**Key deliverables:**
- `TerminalAdapter` interface: `IsAvailable()`, `Discover() []PaneID`, `Capture(PaneID) (string, error)`, `Close()`
- tmux adapter: `tmux list-panes -a -F "..."` for discovery; `tmux capture-pane -p -t <pane-id>` for content; targets stable pane ID (`%3` format) not session:window.pane
- 500ms polling loop with `CaptureManager` bounded concurrency (semaphore, max 4 concurrent subprocesses)
- FNV64a hash deduplication: skip LLM context update when hash unchanged
- Pane lifecycle: new panes detected and tabbed automatically; closed panes marked inactive
- Each pane gets an isolated session (chat history, command history, context buffer)
- Terminal preview pane updates via xterm.js direct write on content change
- Chat messages include filtered terminal context assembled as system prompt prefix

**Exit criteria:** With tmux running, PairAdmin auto-discovers sessions/panes, shows live terminal content, and AI responses reference real terminal output.

---

## Phase 4: Linux GUI Terminal Adapters (AT-SPI2)

**Goal:** Capture content from GNOME Terminal and Konsole for users not running tmux.

**Requirements:** ATSPI-01–04, FILT-04, FILT-05

**Plans:** 4/4 plans complete

Plans:
- [x] 04-01-PLAN.md — CaptureManager architecture + TmuxAdapter refactor from services/terminal.go
- [x] 04-02-PLAN.md — AT-SPI2 adapter: D-Bus accessibility bus connection, GNOME Terminal discovery and text capture
- [x] 04-03-PLAN.md — Konsole spike + frontend degraded tab badge + AT-SPI2 onboarding empty state
- [x] 04-04-PLAN.md — /filter add|list|remove slash commands with Viper config persistence and CustomFilter

**Note:** Konsole support is experimental ��� AT-SPI2 text access is unconfirmed. This phase includes a spike to validate before full implementation.

**Key deliverables:**
- AT-SPI2 adapter using `github.com/godbus/dbus/v5` (no CGO): connect to accessibility bus via `org.a11y.Bus.GetAddress`, enumerate apps, filter by `ATSPI_ROLE_TERMINAL` (role 59)
- GNOME Terminal capture: `org.a11y.atspi.Text.GetText(0, -1)` on the terminal widget; GTK3 path validated; GTK4 path tested with note on reliability
- `gsettings set org.gnome.desktop.interface toolkit-accessibility true` detection and onboarding flow
- Konsole spike: attempt AT-SPI2 text access; document outcome; implement if viable, mark experimental if not
- `/filter add|list|remove` slash commands for user-configurable credential patterns
- Multi-adapter `CaptureManager`: tmux and AT-SPI2 run concurrently; combined pane list deduplicated by PaneID namespace (`tmux:%3` vs `atspi:/path`)
- Graceful degradation: if AT-SPI2 fails for a specific window, log reason and skip (don't crash)

**Exit criteria:** Non-tmux GNOME Terminal windows appear as tabs in PairAdmin with live content. Custom filter patterns work via slash commands.

---

## Phase 5: Settings, Configuration & Slash Commands

**Goal:** A fully configurable application. Users can switch providers, manage API keys, customize prompts, set hotkeys, and use all slash commands.

**Requirements:** CFG-01–08, SLASH-01–08, CLIP-03

**Plans:** 4/4 plans executed

Plans:
- [x] 05-01-PLAN.md — Go backend: expanded AppConfig, keychain integration, SettingsService RPCs, clipboard auto-clear
- [x] 05-02-PLAN.md — Frontend settings dialog with 5 tabs, settingsStore, StatusBar wiring
- [x] 05-03-PLAN.md — Slash command Go methods + frontend router for all 8 commands
- [x] 05-04-PLAN.md — Integration test suite + human verification checkpoint

**Key deliverables:**
- Settings dialog with 5 tabs: LLM Config, Prompts, Terminals, Hotkeys, Appearance
- LLM Config tab: provider dropdown, model dropdown, API key input (masked), connection test button, status indicator
- Prompts tab: read-only built-in system prompt display, editable custom prompt extension textarea
- Terminals tab: AT-SPI2 polling interval slider, per-terminal capture enable/disable
- Hotkeys tab: global hotkey configuration (copy last command, focus PairAdmin)
- Appearance tab: dark/light theme toggle, font size
- `99designs/keyring` for API key storage — no keys in `~/.pairadmin/config.yaml`
- Viper config persistence for all non-secret settings
- All 8 slash commands implemented in chat input parser
- Clipboard auto-clear after 60 seconds (configurable in settings)

**Exit criteria:** User can configure any provider with API key via settings, switch models mid-session via `/model`, and all slash commands work.

---

## Phase 6: Security Hardening

**Goal:** Production-grade security: in-memory credential protection, full audit log, response-side filtering, and Ollama remote-host guard.

**Requirements:** SEC-01–04

**Plans:** 2 plans

Plans:
- [ ] 06-01-PLAN.md — Audit infrastructure: AuditLogger + AuditEntry package, memguard/lumberjack dependencies
- [ ] 06-02-PLAN.md — memguard Enclave API key lifecycle, audit wiring into LLMService/CommandService/main.go, response-side credential filter, security review checklist

**Key deliverables:**
- `memguard` integration: API keys moved to locked memory pages after keychain retrieval; source slice zeroed; `LockedBuffer.Destroy()` on app exit
- Audit logger using `slog` with JSON handler + `lumberjack` rotation: writes to `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl`
- Audit log entries for: `user_message`, `ai_response`, `command_copied`, `session_start`, `session_end`
- Audit log content is always sanitized (post-filter, never raw terminal content)
- Response-side filter scan: lightweight keyword-only pass on LLM responses to catch hallucinated credentials
- Ollama `OLLAMA_HOST` validation at provider initialization (reject non-localhost addresses with a clear error)
- Security review checklist: verify filter pipeline covers all built-in patterns, verify no secrets in logs or config files

**Exit criteria:** Security checklist passes. memguard wraps all in-memory API keys. Audit log captures all interactions. Ollama with a remote host is rejected at config time.

---

## Phase 7: Distribution & Launch

**Goal:** Installable Linux packages and a clean public release.

**Requirements:** DIST-01–04

**Key deliverables:**
- nFPM config for `.deb` and `.rpm` with correct runtime dependencies (`libwebkit2gtk-4.1-0`, `at-spi2-core`)
- AppImage build via Wails; document webkit bundling limitation (Issue #4313) and `.deb` as the recommended install path
- `scripts/install-deps.sh`: detects Debian/Ubuntu vs Fedora/RHEL, installs all build-time and runtime dependencies
- Wails build pipeline: `wails build -platform linux/amd64` produces all three artifacts
- Clean install test on: Ubuntu 22.04, Ubuntu 24.04, Fedora 40
- README with installation instructions for each package type
- GitHub Releases with signed binaries and checksums
- Final acceptance criteria check against all v1 requirements

**Exit criteria:** Clean install from `.deb` on a fresh Ubuntu 22.04 VM. Application launches, connects to Ollama, captures a tmux pane, and completes an AI chat interaction.

---

## Summary

| Phase | Name | Key Output |
|-------|------|-----------|
| 1 | Application Shell | Working Wails/React UI skeleton |
| 2 | LLM Gateway | Real AI chat with streaming and credential filtering |
| 3 | tmux Capture | Automatic terminal context from tmux |
| 4 | AT-SPI2 Adapters | GNOME Terminal/Konsole capture |
| 5 | Settings & Config | Full user configuration, slash commands |
| 6 | Security Hardening | memguard, audit log, response filtering |
| 7 | Distribution | .deb, .rpm, AppImage, clean install |

**Status:** Phase 5 complete — Phase 6 (Security Hardening) ready for execution
