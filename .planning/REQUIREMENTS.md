# Requirements: PairAdmin v2.0

**Defined:** 2026-03-25
**Core Value:** The AI sees exactly what you see in the terminal — automatically, without copy/paste — so assistance is always in context.

## v1 Requirements

### Application Shell

- [ ] **SHELL-01**: Application launches as a native desktop window using Wails v2 with React + TypeScript frontend
- [x] **SHELL-02**: Three-column layout renders correctly: terminal tabs (left, 160px fixed), chat area (center, flexible), command sidebar (right, 220px collapsible)
- [x] **SHELL-03**: Status bar displays active model, connection status, token usage, and settings button
- [ ] **SHELL-04**: Application builds and runs on Ubuntu 22.04+ with `libwebkit2gtk-4.1-dev` and `-tags webkit2_41`

### Terminal Capture — tmux

- [ ] **TMUX-01**: Application discovers all active tmux sessions and panes on startup via `tmux list-panes -a`
- [ ] **TMUX-02**: Terminal content is captured from each pane via `tmux capture-pane -p` at 500ms polling interval
- [ ] **TMUX-03**: New tmux sessions/panes are detected automatically without user action
- [ ] **TMUX-04**: Closed tmux sessions are detected and corresponding tabs are marked inactive
- [ ] **TMUX-05**: FNV64a hash deduplication prevents sending unchanged content to the LLM pipeline
- [ ] **TMUX-06**: Each tmux pane maps to an isolated PairAdmin tab with independent chat history and context

### Terminal Capture — Linux GUI (AT-SPI2)

- [ ] **ATSPI-01**: Application detects whether AT-SPI2 accessibility is enabled at startup; guides user to enable if not
- [ ] **ATSPI-02**: GNOME Terminal windows (GTK3) are discovered via AT-SPI2 accessibility bus (`ATSPI_ROLE_TERMINAL` objects)
- [ ] **ATSPI-03**: Visible terminal content is read from GNOME Terminal via `org.a11y.atspi.Text.GetText(0, -1)` at 500ms polling interval
- [ ] **ATSPI-04**: Konsole windows are attempted via AT-SPI2; feature degrades gracefully if text extraction fails (experimental)

### Context & Filtering

- [x] **FILT-01**: ANSI/VT100 escape sequences are stripped from all terminal content before any processing (security requirement — prevents injection via invisible sequences)
- [x] **FILT-02**: Built-in credential filter detects and redacts: AWS access keys, AWS secret keys, GitHub tokens, GCP service account keys, generic API keys, SSH private key blocks, database DSN passwords, bearer tokens, password prompt lines
- [x] **FILT-03**: Filtered/redacted content is what gets sent to the LLM — original unredacted content is never transmitted to cloud APIs
- [ ] **FILT-04**: User can add custom filter patterns via `/filter add <name> <regex> <action>` slash command
- [ ] **FILT-05**: User can list and remove custom filter patterns via `/filter list` and `/filter remove <name>`
- [ ] **FILT-06**: Terminal content is truncated to fit within the active provider's context window; most recent content is prioritized
- [x] **FILT-07**: Token count and context usage is displayed in the status bar

### LLM Gateway

- [x] **LLM-01**: OpenAI provider is implemented using `github.com/openai/openai-go/v3`; supports streaming chat completions
- [x] **LLM-02**: Anthropic provider is implemented using `github.com/anthropics/anthropic-sdk-go`; system prompt is handled as top-level field (not message role)
- [x] **LLM-03**: Ollama provider is implemented using `github.com/ollama/ollama/api`; validates that `OLLAMA_HOST` is a localhost address before use
- [x] **LLM-04**: LM Studio and llama.cpp are supported by reusing the OpenAI adapter with a configurable base URL
- [x] **LLM-05**: All providers implement a common channel-based streaming interface: `Stream() (<-chan StreamChunk, error)`
- [x] **LLM-06**: Streaming responses are delivered to the React frontend via Wails EventsEmit with sequence numbers and 50ms batching (mitigates Wails Issue #2759)
- [x] **LLM-07**: When Ollama is selected, no terminal content is transmitted over any network interface

### Chat Interface

- [x] **CHAT-01**: User can type a question in the chat input and send it (Enter to send, Shift+Enter for newline)
- [x] **CHAT-02**: Every outgoing message includes the current terminal context (filtered) assembled as a system prompt prefix
- [x] **CHAT-03**: AI responses stream token-by-token into the chat area as they arrive
- [ ] **CHAT-04**: AI-suggested commands are rendered in syntax-highlighted code blocks (react-shiki) with a "Copy to Terminal" button
- [ ] **CHAT-05**: Chat history is isolated per terminal tab; switching tabs shows that tab's conversation only
- [ ] **CHAT-06**: `/clear` command clears chat history for the current tab

### Command Sidebar

- [ ] **CMD-01**: Every command block the AI suggests is automatically added to the command sidebar
- [ ] **CMD-02**: Commands in the sidebar are displayed in reverse-chronological order (newest at top)
- [x] **CMD-03**: Clicking a command in the sidebar copies it to the clipboard
- [x] **CMD-04**: Hovering over a sidebar command shows the original question that generated it
- [ ] **CMD-05**: "Clear History" button removes all commands from the sidebar for the current tab

### Clipboard & Command Execution

- [x] **CLIP-01**: "Copy to Terminal" button copies the command to the system clipboard
- [x] **CLIP-02**: Application detects Wayland display server at startup and warns if `wl-clipboard` is not installed
- [ ] **CLIP-03**: Clipboard contents copied by PairAdmin are automatically cleared after 60 seconds (configurable)

### Configuration & Settings

- [ ] **CFG-01**: LLM provider and model are configurable via settings dialog (LLM Config tab)
- [ ] **CFG-02**: API keys are stored in OS keychain using `99designs/keyring` (not plaintext config files)
- [ ] **CFG-03**: Connection to configured provider can be tested from the settings dialog
- [ ] **CFG-04**: User can provide a custom system prompt extension (appended to built-in prompt) via settings Prompts tab
- [ ] **CFG-05**: AT-SPI2 polling interval is configurable via settings Terminals tab
- [ ] **CFG-06**: Global hotkeys are configurable (copy last command, focus PairAdmin window)
- [ ] **CFG-07**: Dark and light themes are available; dark is default
- [ ] **CFG-08**: Settings are persisted to `~/.pairadmin/config.yaml` via Viper (no secrets in this file)

### Slash Commands

- [ ] **SLASH-01**: `/model <provider:model>` switches the active LLM provider/model
- [ ] **SLASH-02**: `/context <lines>` sets the terminal context window size
- [ ] **SLASH-03**: `/refresh` forces re-capture of terminal content
- [ ] **SLASH-04**: `/filter add|list|remove` manages sensitive data filter patterns
- [ ] **SLASH-05**: `/export json|txt` exports current session chat history
- [ ] **SLASH-06**: `/rename <label>` renames the current terminal tab
- [ ] **SLASH-07**: `/theme dark|light` switches color scheme
- [ ] **SLASH-08**: `/help` displays available commands

### Security & Audit

- [ ] **SEC-01**: API keys loaded from keychain into memory are protected using `memguard` (mlock, encrypted at rest in process)
- [ ] **SEC-02**: All AI interactions are written to a local audit log at `~/.pairadmin/logs/audit-YYYY-MM-DD.jsonl` using Go's `slog` with `lumberjack` rotation
- [ ] **SEC-03**: Audit log entries contain: timestamp, session ID, terminal ID, event type, sanitized content (filtered), command copied
- [ ] **SEC-04**: LLM response content is scanned by the credential filter (lighter-weight pass) to catch model hallucinations of sensitive data

### Distribution

- [ ] **DIST-01**: Application builds as a `.deb` package via nFPM with `libwebkit2gtk-4.1-0` declared as a runtime dependency
- [ ] **DIST-02**: Application builds as an AppImage (with documented fallback to `.deb` for webkit runtime issues)
- [ ] **DIST-03**: Application builds as an `.rpm` package via nFPM
- [ ] **DIST-04**: Install script (`scripts/install-deps.sh`) installs all build-time dependencies on Ubuntu/Debian and Fedora/RHEL

## v2 Requirements

### macOS Support

- **MACOS-01**: macOS Terminal.app content captured via Accessibility API (CGO/Objective-C bindings)
- **MACOS-02**: macOS permission flow: detect, prompt, open System Preferences, re-check
- **MACOS-03**: Application distributed as `.dmg` and `.app`

### Windows Support

- **WIN-01**: PuTTY content captured via Windows UI Automation API
- **WIN-02**: OCR fallback if UI Automation text extraction fails on PuTTY
- **WIN-03**: Application distributed as `.msi` installer
- **WIN-04**: Windows Credential Manager used for API key storage on Windows

### Enhanced Features

- **ENH-01**: Chat history persisted to local SQLite database across app restarts (with optional encryption)
- **ENH-02**: Session export to PDF format
- **ENH-03**: Multi-pane view (show terminal preview inline rather than in sidebar)
- **ENH-04**: Wails v3 migration for typed events and native packaging improvements

## Out of Scope

| Feature | Reason |
|---------|--------|
| Cloud sync / multi-machine | Data residency and security concerns; PairAdmin is a local-first tool |
| Real-time collaboration | Out of scope for a sysadmin assistance tool |
| Built-in terminal emulator | PairAdmin observes terminals, it does not replace them |
| Auto-executing commands | User must always paste — auto-execute is a safety boundary |
| Web/browser version | Desktop application architecture is load-bearing for terminal access |
| Mobile app | Terminal administration on mobile is not a target use case |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SHELL-01 | Phase 1 | Pending |
| SHELL-02 | Phase 1 | Complete |
| SHELL-03 | Phase 1 | Complete |
| SHELL-04 | Phase 1 | Pending |
| CHAT-01 | Phase 1 | Complete |
| CHAT-05 | Phase 1 | Pending |
| CHAT-06 | Phase 1 | Pending |
| CMD-01 | Phase 1 | Pending |
| CMD-02 | Phase 1 | Pending |
| CMD-03 | Phase 1 | Complete |
| CMD-04 | Phase 1 | Complete |
| CMD-05 | Phase 1 | Pending |
| CLIP-01 | Phase 1 | Complete |
| CLIP-02 | Phase 1 | Complete |
| FILT-01 | Phase 2 | Complete |
| FILT-02 | Phase 2 | Complete |
| FILT-03 | Phase 2 | Complete |
| FILT-06 | Phase 2 | Pending |
| FILT-07 | Phase 2 | Complete |
| LLM-01 | Phase 2 | Complete |
| LLM-02 | Phase 2 | Complete |
| LLM-03 | Phase 2 | Complete |
| LLM-04 | Phase 2 | Complete |
| LLM-05 | Phase 2 | Complete |
| LLM-06 | Phase 2 | Complete |
| LLM-07 | Phase 2 | Complete |
| CHAT-02 | Phase 2 | Complete |
| CHAT-03 | Phase 2 | Complete |
| CHAT-04 | Phase 2 | Pending |
| TMUX-01 | Phase 3 | Pending |
| TMUX-02 | Phase 3 | Pending |
| TMUX-03 | Phase 3 | Pending |
| TMUX-04 | Phase 3 | Pending |
| TMUX-05 | Phase 3 | Pending |
| TMUX-06 | Phase 3 | Pending |
| ATSPI-01 | Phase 4 | Pending |
| ATSPI-02 | Phase 4 | Pending |
| ATSPI-03 | Phase 4 | Pending |
| ATSPI-04 | Phase 4 | Pending |
| FILT-04 | Phase 4 | Pending |
| FILT-05 | Phase 4 | Pending |
| CFG-01 | Phase 5 | Pending |
| CFG-02 | Phase 5 | Pending |
| CFG-03 | Phase 5 | Pending |
| CFG-04 | Phase 5 | Pending |
| CFG-05 | Phase 5 | Pending |
| CFG-06 | Phase 5 | Pending |
| CFG-07 | Phase 5 | Pending |
| CFG-08 | Phase 5 | Pending |
| SLASH-01 | Phase 5 | Pending |
| SLASH-02 | Phase 5 | Pending |
| SLASH-03 | Phase 5 | Pending |
| SLASH-04 | Phase 5 | Pending |
| SLASH-05 | Phase 5 | Pending |
| SLASH-06 | Phase 5 | Pending |
| SLASH-07 | Phase 5 | Pending |
| SLASH-08 | Phase 5 | Pending |
| CLIP-03 | Phase 5 | Pending |
| SEC-01 | Phase 6 | Pending |
| SEC-02 | Phase 6 | Pending |
| SEC-03 | Phase 6 | Pending |
| SEC-04 | Phase 6 | Pending |
| DIST-01 | Phase 7 | Pending |
| DIST-02 | Phase 7 | Pending |
| DIST-03 | Phase 7 | Pending |
| DIST-04 | Phase 7 | Pending |

**Coverage:**
- v1 requirements: 57 total
- Mapped to phases: 57
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-25*
*Last updated: 2026-03-25 after initial definition*
