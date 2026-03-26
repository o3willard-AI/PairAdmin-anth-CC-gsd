# Project Research Summary

**Project:** PairAdmin v2.0
**Domain:** Go desktop application — LLM-assisted terminal context assistant
**Researched:** 2026-03-25
**Confidence:** MEDIUM-HIGH overall (individual domains range from HIGH to MEDIUM)

---

## Executive Summary

PairAdmin v2.0 is a Go desktop application built with Wails v2 (WebKit2GTK + React/TypeScript frontend) that captures terminal content from the user's running sessions and streams it as context to one of several LLM providers. The core architecture involves three parallel systems that must be coordinated: a terminal capture subsystem that polls tmux and GUI terminals (GNOME Terminal, Konsole) at 500ms intervals, a credential filter pipeline that redacts secrets before any data leaves the machine, and a provider-agnostic LLM gateway that streams token output back to the frontend in real time. All three must meet hard latency budgets — the filter must complete in under 50ms, the capture must complete within the 500ms polling window, and the streaming pathway must not accumulate latency.

The recommended approach is to build the tmux adapter first (high confidence, pure subprocess calls, no external dependencies beyond the tmux binary) and validate the end-to-end pipeline before tackling GUI terminal adapters. The LLM gateway uses a channel-based streaming interface in Go with a provider registry pattern, supporting OpenAI (and OpenAI-compatible endpoints like LM Studio and llama.cpp via base URL override), Anthropic, and Ollama. Security is not an afterthought — it is structurally load-bearing: ANSI stripping must happen before regex scanning, credential detection must happen before LLM transmission, and the gitleaks library provides the rule base. API keys must never touch config files; the 99designs/keyring library handles OS keychain integration with encrypted-file fallback for headless Linux environments.

The primary risks are: (1) GUI terminal capture reliability — AT-SPI2 text access for GNOME Terminal is documented as unreliable for change events and Konsole AT-SPI2 text access is unconfirmed; plan for graceful degradation to polling-only with reduced fidelity for non-tmux terminals. (2) Wails v2 event ordering — high-frequency EventsEmit calls are confirmed to deliver out of order; the 50ms batching + sequence number pattern is mandatory, not optional. (3) Prompt injection via terminal content — terminal output is untrusted data that can contain adversarial instructions; ANSI stripping and XML delimiters around terminal context in the system prompt are non-negotiable mitigations.

---

## Key Findings

### Recommended Stack

PairAdmin runs as a native Linux desktop binary. The Go backend (1.22+) embeds all frontend assets and exposes a Wails v2 IPC bridge to a React 18 + TypeScript frontend served by Vite during development. The webview host is WebKit2GTK, which means the system must have `libwebkit2gtk-4.0` (Ubuntu 22.04) or `libwebkit2gtk-4.1` (Ubuntu 24.04+) installed — this is an explicit runtime dependency that must appear in packaging metadata. Build on Ubuntu 22.04 LTS for broadest AppImage compatibility; use `-tags webkit2_41` when targeting Ubuntu 24.04+.

The frontend state management stack is Zustand 5.x for business state (chat messages, terminal pane list) and Jotai 2.x for fine-grained UI atoms (active tab, sidebar state). Terminal output in the xterm.js panel bypasses React state entirely — direct writes to the xterm.js instance are critical for performance. Chat code blocks use react-shiki (Shiki/TextMate grammars, streaming support via `delay` prop). React context is explicitly ruled out for streaming updates due to full-subtree re-render cost.

**Core technologies:**
- `github.com/wailsapp/wails/v2` v2.10.1: Desktop shell (WebKit2GTK + IPC bridge)
- `github.com/openai/openai-go/v3` v3.30.0: Official OpenAI SDK; also used for LM Studio and llama.cpp via base URL override
- `github.com/anthropics/anthropic-sdk-go` v1.27.1: Official Anthropic SDK; note system prompt goes in top-level `System` field, not as a role
- `github.com/ollama/ollama/api` v0.18.x: Official Ollama SDK; callback-based streaming must be wrapped to the channel interface
- `github.com/godbus/dbus/v5` v5.1.0: Pure Go D-Bus for AT-SPI2 accessibility bus access
- `github.com/zricethezav/gitleaks/v8`: Credential detection engine (150+ compiled rules)
- `github.com/99designs/keyring`: OS keychain with encrypted-file fallback for headless Linux
- `github.com/awnumar/memguard`: Secure in-memory storage for API keys (mlock, guard pages, XSalsa20Poly1305)
- `github.com/pkoukk/tiktoken-go`: Offline token counting for OpenAI models
- `github.com/cenkalti/backoff/v4`: Exponential backoff for LLM API retry
- `@xterm/xterm` ^5.5.0 + addons: Terminal rendering (bypass React, GPU-accelerated canvas)
- `zustand` ^5.0.0 + `jotai` ^2.10.0: Frontend state management
- `react-shiki` ^0.6.0: Syntax highlighting for chat code blocks
- `react-markdown` ^9.0.0: Markdown rendering for chat responses

### Expected Features

Research determined features by what the product domain requires rather than a separate FEATURES.md file. The following are implied by the architecture and security constraints:

**Must have (table stakes):**
- tmux pane discovery and capture — the only fully reliable capture path; users working in tmux get full functionality
- Credential redaction filter (gitleaks-based) — must run before every LLM transmission; not configurable off
- ANSI escape sequence stripping — must be Stage 1 of filter pipeline; security-critical (Trail of Bits April 2025)
- Multi-provider LLM gateway (OpenAI, Anthropic, Ollama) with streaming — core value proposition
- OS keychain storage for API keys — never store credentials in config files
- Structured audit log of every LLM transmission — local only, with redaction of detected secrets
- Recency-first context truncation — terminal tail is most relevant; conversation history uses sliding window
- Provider selector UI — user must be able to see and switch which provider/model is active

**Should have (competitive):**
- GNOME Terminal capture via AT-SPI2 — expands user base beyond tmux-only users; implement as best-effort with fallback
- User-configurable filter rules (TOML at `~/.config/pairadmin/filters.toml`) — slash command interface for add/remove/test
- LM Studio and llama.cpp support — zero additional code; configure OpenAI adapter with custom base URL
- Context window budget display — show estimated token usage across providers
- Konsole capture — experimental; implement as "active confirmation" without content if AT-SPI2 fails
- Clipboard security (auto-clear after 30s, hash-match before clearing) — for any copy actions PairAdmin initiates

**Defer (v2+):**
- tmux control mode event-driven capture — complexity not justified; polling is sufficient
- Anthropic token counting (offline) — no official Go tokenizer; approximation (1 token ≈ 3.5 chars) is sufficient for budget
- Wails v3 migration — v3 is still alpha; API in flux; plan migration path but do not block on it
- Multi-window support — Wails v2 is single-window; acceptable constraint for v1
- LLM function/tool calling from terminal content — explicit non-feature due to prompt injection risk

### Architecture Approach

The architecture is a three-panel Wails application: terminal panel (xterm.js, managed entirely outside React), chat panel (streaming markdown with code highlighting), and command sidebar. The Go backend is structured around three service structs bound to Wails: `TerminalService` (capture management, polling loop), `ChatService` (LLM gateway, streaming, context management), and `CommandService` (slash commands, filter management). A `CaptureManager` coordinates multiple `TerminalAdapter` implementations (tmux, ATSPI), running concurrent captures with a semaphore (max 4 concurrent subprocesses). The LLM gateway uses a channel-based `Provider` interface with a `Registry` for runtime provider selection; each SDK's streaming model is normalized to `<-chan StreamChunk`. The filter pipeline is a staged sequence: ANSI strip → keyword fast pass → regex match → entropy check → redact → audit log → send. All config lives in `~/.config/pairadmin/`; all sensitive data in the OS keychain.

**Major components:**
1. `CaptureManager` + `TerminalAdapter` implementations — discovers and polls panes; deduplicates by FNV-1a hash; runs on 500ms ticker
2. Filter pipeline (`Filter` interface, gitleaks detector, Shannon entropy) — redacts credentials before any transmission; ANSI strip is Stage 1
3. LLM gateway (`Provider` interface + adapters for OpenAI/Anthropic/Ollama + `Registry`) — channel-based streaming; context window management; retry with exponential backoff
4. Wails service layer (`TerminalService`, `ChatService`, `CommandService`) — binds Go to frontend; uses goroutine + 50ms batched EventsEmit for streaming
5. React frontend (Zustand stores, xterm.js terminal panel, chat panel with react-shiki, command sidebar) — granular Zustand selectors; React.memo on stable items; streaming bubble uses ref buffer + 50ms flush

### Critical Pitfalls

1. **Wails EventsEmit out-of-order delivery at high rates** — Confirmed data race (issue #2448). Mitigation: emit batched chunks every 50ms with sequence numbers; implement reordering buffer in the React `useChatStream` hook. Never emit per-token without batching.

2. **ANSI escape injection into LLM context** — Trail of Bits (April 2025) and a February 2026 Codex CLI finding confirm that hidden ANSI sequences can carry adversarial instructions that the LLM processes but the human cannot see. Mitigation: ANSI strip is unconditional Stage 1 in the filter pipeline; use XML delimiters (`<terminal>...</terminal>`) in system prompt.

3. **AT-SPI2 reliability for terminal content capture** — VTE (GNOME Terminal's widget) has a documented history of not emitting `object:text-changed` events reliably (VTE GitLab issue #88). Konsole AT-SPI2 text access is unverified end-to-end. Mitigation: use polling as primary path; never depend on AT-SPI2 events as the change notification mechanism; implement graceful degradation in `IsAvailable()`.

4. **Ubuntu 24.04 webkit2gtk version split** — Ubuntu 24.04 dropped `libwebkit2gtk-4.0-dev`; only 4.1 is available. Mitigation: detect at build time; use `-tags webkit2_41` flag; declare correct runtime package in nFPM metadata (`libwebkit2gtk-4.1-0`, not `-dev`).

5. **Credential detection gap: scrollback buffer contains old secrets** — A terminal snapshot includes scrollback content typed at earlier times. Credentials typed, then scrolled out of view, remain in the buffer. Mitigation: run the credential filter on all captured content, not just new lines since last poll; never cache unredacted snapshots to disk.

6. **Prompt injection via terminal content (indirect)** — OWASP LLM01:2025, present in 73% of production AI audits. Malicious content in `curl` responses or other terminal output can instruct the LLM. Mitigation: XML delimiters in system prompt; system prompt instructs model to treat `<terminal>` content as data not instructions; do not grant LLM tool-call capabilities triggered by terminal content.

7. **Ollama remote host misconfiguration** — If `OLLAMA_HOST` is set to a remote address, terminal content leaves the machine over an unauthenticated, non-TLS connection. Mitigation: validate Ollama endpoint is localhost at config load time; refuse to send if remote; display warning in UI.

---

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation — Wails Shell + tmux Capture Pipeline

**Rationale:** tmux capture is the highest-confidence, lowest-risk capture path (pure subprocess, no external dependencies, well-documented). Establishing the Wails shell + end-to-end data flow from capture through filter to a stub LLM call validates the entire pipeline architecture before touching the harder problems (GUI terminal capture, real LLM streaming). Getting the 50ms batched EventsEmit pattern right from the start prevents the ordering bugs from being baked in.

**Delivers:** A working Wails desktop app that discovers tmux panes, polls them at 500ms, runs the credential filter, and displays redacted content in the UI (stub LLM response acceptable at this phase).

**Addresses:**
- tmux pane discovery and capture (table stakes)
- ANSI stripping (security-critical, must be established early)
- Basic credential redaction with gitleaks (security-critical)
- Wails shell with xterm.js terminal panel and three-panel layout
- 50ms batched EventsEmit + sequence number streaming pattern

**Avoids:**
- Out-of-order event delivery pitfall — establish the pattern correctly from day one
- Ubuntu webkit2gtk version split — choose correct build configuration upfront

**Research flag:** Standard patterns — well-documented; skip research-phase.

---

### Phase 2: LLM Gateway — Provider Abstraction + Streaming

**Rationale:** The channel-based `Provider` interface with adapters is well-researched and has clear implementation patterns for all three SDKs. This phase wires the filter output to real LLM calls and streams responses back to the chat panel. Context window management (recency-first truncation, sliding history window) must be implemented here because token budgets affect how capture content is assembled for each request.

**Delivers:** Real LLM streaming from Ollama (local, no API key required for testing), OpenAI, and Anthropic. Chat panel with react-shiki code highlighting. Token budget management. Retry with exponential backoff. OS keychain integration for API keys.

**Addresses:**
- Multi-provider LLM gateway (core value proposition)
- Streaming chat with sequence-numbered batching
- Recency-first context truncation
- API key storage in OS keychain (never in config files)
- Ollama localhost validation (security)
- Provider selector UI

**Avoids:**
- Mid-stream retry (no — only retry on initial connection failure)
- Buffering all tokens before emitting (defeats streaming UX)
- Anthropic system prompt in message role (must be top-level `System` field)

**Research flag:** Standard patterns — channel interface and adapter pattern are well-documented; Ollama callback-to-channel wrapping is the only non-obvious element. Skip research-phase.

---

### Phase 3: Security Hardening — Full Filter Pipeline + Audit Logging + Memory Protection

**Rationale:** Phase 1 establishes basic gitleaks detection and ANSI stripping. Phase 3 completes the security model: Shannon entropy for high-entropy unknown secrets, user-configurable filter rules via TOML with slash command interface, audit log (slog + lumberjack, XDG data dir, 0600 permissions), memguard for in-memory API key protection, and LLM response scanning (defense-in-depth for credential exfiltration). These features are structurally independent of GUI terminal adapters, making Phase 3 a natural point to harden the core before expanding capture sources.

**Delivers:** Complete filter pipeline (ANSI → keyword → regex → entropy → redact → audit), user rule management slash commands, structured audit log with rotation, memguard-protected API keys, response-side credential scanning, `PR_SET_DUMPABLE=0` for memory protection, clipboard auto-clear for any copy actions.

**Addresses:**
- Full credential detection coverage (all gitleaks patterns + Shannon entropy + user rules)
- Audit log (local-only, types not values, SHA-256 hash of redacted payload)
- User-configurable filter rules (`/filter add`, `/filter list`, `/filter test`)
- Response-side scanning (defense-in-depth)
- Clipboard security

**Avoids:**
- Logging credential values (log type, count, and hash only)
- User rules that can disable built-in detection (built-ins compiled into binary)
- Config files containing API keys (migration command `/migrate-keys`)

**Research flag:** Standard patterns for most items. The Shannon entropy + gitleaks library integration may benefit from a short spike to confirm the `detect.Detector` API accepts arbitrary strings cleanly. MEDIUM confidence on library integration; HIGH confidence on the security logic.

---

### Phase 4: GUI Terminal Adapters — GNOME Terminal + Konsole

**Rationale:** GUI terminal capture depends on AT-SPI2, which has meaningful reliability uncertainty (confirmed for GNOME Terminal GTK3, less certain for GTK4, unconfirmed for Konsole text access). Building this after the core pipeline is proven means failures here cannot block the product's primary value. Each adapter must implement `IsAvailable()` with graceful degradation — a failure here should surface as "capture unavailable for this terminal" in the UI, never as a crash.

**Delivers:** GNOME Terminal pane discovery and content capture via AT-SPI2 (best-effort). Konsole adapter with `foregroundProcessName` confirmation and experimental AT-SPI2 text access. A11y bus detection and status display in UI. Graceful degradation path when AT-SPI2 is unavailable or unreliable.

**Addresses:**
- GNOME Terminal capture (broadens user base beyond tmux-only)
- Konsole capture (experimental, with clear UI signaling of limitations)
- Permission/configuration detection (GSettings, runtime `IsEnabled`)

**Avoids:**
- Relying on AT-SPI2 `object:text-changed` events as primary change notification (use polling, events are optional enhancement)
- Blocking on Konsole text access being unavailable (degrade to "active" signal without content)
- GTK4 Cache.GetItems signature assumption (try new signature, fall back to Qt5 old signature)

**Research flag:** NEEDS deeper research/spike before implementation. Konsole AT-SPI2 text access is unconfirmed. GTK4 `GtkAccessibleText` interface reliability is uncertain. Plan for a focused spike: set up a test environment with each terminal emulator and verify text access end-to-end before writing production code.

---

### Phase 5: Distribution + Polish

**Rationale:** Distribution (AppImage, .deb via nFPM) has a known rough edge (AppImage cannot fully bundle WebKit) that needs a deliberate decision about the supported installation story. This phase also covers xterm.js WebGL vs canvas renderer validation on target systems and any remaining Wails gotchas (window background flicker, CSP considerations for LLM-generated HTML).

**Delivers:** AppImage and .deb packages. Verified rendering on Ubuntu 22.04 and 24.04 LTS. Build pipeline documentation. Runtime dependency declaration in packaging metadata.

**Addresses:**
- AppImage with linuxdeploy + linuxdeploy-plugin-gtk
- .deb and .rpm via nFPM with correct WebKit runtime dep declaration
- xterm.js renderer validation (WebGL vs canvas fallback)
- Window background color set to prevent white flash on startup

**Avoids:**
- Declaring `-dev` packages as runtime deps in nFPM (use `libwebkit2gtk-4.1-0` not `-dev`)
- Building on Ubuntu 24.04 for AppImage intended to run on 22.04 (build on older for broader compat)

**Research flag:** Mostly standard patterns. AppImage WebKit bundling limitation is a known open issue (#4313) — accept it, document it, or validate workaround. No research-phase needed; implementation spike to verify AppImage on target systems is sufficient.

---

### Phase Ordering Rationale

- Phase 1 before Phase 2 because the end-to-end data flow (capture → filter → LLM → render) must be validated before layering in multiple providers and streaming complexity.
- Phase 3 (security hardening) before Phase 4 (GUI adapters) because the security model must be complete before expanding the capture attack surface to more terminal types. New adapter = new surface for credential leakage.
- Phase 4 is isolated because its reliability uncertainty should not block earlier phases. The tmux adapter alone provides sufficient value for v1.
- Phase 5 last because distribution cannot be finalized until the feature set stabilizes.

### Research Flags

Phases needing deeper research or spikes during planning:
- **Phase 4 (GUI Terminal Adapters):** Konsole AT-SPI2 text access is unconfirmed end-to-end. GNOME Terminal GTK4 `GtkAccessibleText` reliability is uncertain. Run a focused spike with a real GNOME Terminal and Konsole instance before writing production adapter code.

Phases with standard patterns (skip research-phase):
- **Phase 1:** tmux subprocess API is stable and well-documented. Wails v2 architecture is well-documented from official sources.
- **Phase 2:** LLM SDK streaming patterns are confirmed from official SDK READMEs. Channel-based provider abstraction is a standard Go pattern.
- **Phase 3:** gitleaks library integration, slog + lumberjack, 99designs/keyring, and memguard are all documented. Security logic follows established patterns.
- **Phase 5:** Distribution tooling (linuxdeploy, nFPM) is well-documented. Issues are known and have workarounds.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Wails v2 architecture and bindings | HIGH | Official docs; confirmed bug reports provide clear workarounds |
| Wails v2 event ordering issue | HIGH | Confirmed data race in issue tracker; 50ms batch pattern is the fix |
| Wails v2 packaging (AppImage/deb) | MEDIUM | Known open issue with WebKit bundling; nFPM patterns from merged PR |
| LLM SDKs (OpenAI, Anthropic) | HIGH | Official SDK READMEs; pkg.go.dev verified |
| Ollama SDK | HIGH | Official package; callback→channel wrapping pattern is straightforward |
| LM Studio / llama.cpp compatibility | HIGH | OpenAI-compatible endpoint; base URL override confirmed via official docs |
| Token counting (OpenAI) | HIGH | tiktoken-go has 344+ importers; same encoding tables as Python tiktoken |
| Token counting (Anthropic) | MEDIUM | No official Go tokenizer; character approximation is within 15% |
| tmux capture-pane | HIGH | Verified against man page and source code |
| AT-SPI2 protocol and D-Bus interface | HIGH | Official at-spi2-core documentation; godbus/dbus/v5 is correct approach |
| GNOME Terminal AT-SPI2 text access | MEDIUM | GTK3: works but unreliable events; GTK4: newer, less tested |
| Konsole AT-SPI2 text access | LOW-MEDIUM | Qt AT-SPI2 bridge exists but text access end-to-end is unconfirmed |
| Credential detection (gitleaks) | HIGH | 18k+ star project; patterns sourced directly from gitleaks.toml |
| ANSI injection security finding | HIGH | Trail of Bits April 2025; February 2026 Codex CLI incident; confirmed |
| OS keychain (99designs/keyring) | HIGH | Active project; encrypted-file fallback confirmed for headless Linux |
| memguard | HIGH | awnumar/memguard; mlock and guard page behavior documented |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **Konsole AT-SPI2 text access:** No authoritative source confirms `org.a11y.atspi.Text.GetText` works on Konsole's Qt terminal widget. This gap must be resolved with a hands-on spike before Phase 4 planning. Fallback plan: expose Konsole as "active without content" using D-Bus `foregroundProcessName`.

- **GNOME Terminal GTK4 text access reliability:** The VTE `GtkAccessibleText` implementation in GNOME 47+ is newer and has less community documentation. Test on a current Ubuntu 24.04 system with GNOME 47+ before committing to the GTK4 code path.

- **xterm.js WebGL renderer in WebKit2GTK:** The xterm.js WebGL renderer may not work in WebKit2GTK on all Linux GPU configurations. Must validate on target systems and prepare the canvas renderer fallback (`rendererType: 'canvas'`). This is flagged as an open question in the Wails research.

- **gitleaks library vs. binary integration:** The research recommends importing `github.com/zricethezav/gitleaks/v8/detect` as a library. The API for scanning arbitrary strings (not git objects) should be confirmed as stable before committing to it — gitleaks is primarily a CLI tool and the library surface may be less stable than the binary interface.

- **Wails v3 migration path:** v3 is in alpha with API changes still in progress. For a production app starting now, v2 is correct. However, v3's redesigned event system (fixes the ordering race) and native nFPM packaging are attractive. Monitor the v3 GA timeline and plan a migration path rather than deep-coupling to v2-specific workarounds.

---

## Sources

### Primary (HIGH confidence)

- [Wails v2 official docs](https://wails.io/docs/) — architecture, bindings, events, Linux support, packaging
- [github.com/openai/openai-go](https://github.com/openai/openai-go) — official OpenAI Go SDK; streaming API
- [github.com/anthropics/anthropic-sdk-go](https://github.com/anthropics/anthropic-sdk-go) — official Anthropic Go SDK; system prompt field difference
- [pkg.go.dev/github.com/ollama/ollama/api](https://pkg.go.dev/github.com/ollama/ollama/api) — official Ollama API package; callback streaming
- [lmstudio.ai/docs/developer/openai-compat](https://lmstudio.ai/docs/developer/openai-compat) — OpenAI-compatible endpoint confirmed
- [at-spi2-core documentation](https://docs.gtk.org/atspi2/) — accessibility bus protocol; D-Bus interfaces
- [gitleaks/gitleaks](https://github.com/gitleaks/gitleaks) — credential detection patterns sourced from gitleaks.toml
- [awnumar/memguard](https://pkg.go.dev/github.com/awnumar/memguard) — secure memory enclave API
- [99designs/keyring](https://github.com/99designs/keyring) — keychain backends and file fallback
- [OWASP LLM Top 10 2025](https://genai.owasp.org/llmrisk/llm01-prompt-injection/) — prompt injection risk classification
- [tmux man page](https://man7.org/linux/man-pages/man1/tmux.1.html) — capture-pane flags; list-panes format variables
- [xtermjs.org](https://xtermjs.org/) — xterm.js official site

### Secondary (MEDIUM confidence)

- [Wails GitHub Issue #2759](https://github.com/wailsapp/wails/issues/2759) — out-of-order EventsEmit at high rates
- [Wails GitHub Issue #2448](https://github.com/wailsapp/wails/issues/2448) — data race in events system (confirmed)
- [Wails GitHub Issue #4313](https://github.com/wailsapp/wails/issues/4313) — AppImage WebKit bundling limitation (open)
- [Wails GitHub Issue #3513](https://github.com/wailsapp/wails/issues/3513) — Ubuntu 24.04 webkit2gtk version split (resolved)
- [Wails GitHub PR #4481](https://github.com/wailsapp/wails/pull/4481) — correct nFPM runtime dependency declaration
- [mozilla/any-llm-go (Mozilla AI blog)](https://blog.mozilla.ai/run-openai-claude-mistral-llamafile-and-more-from-one-interface-now-in-go/) — provider abstraction reference pattern
- [github.com/pkoukk/tiktoken-go](https://github.com/pkoukk/tiktoken-go) — token counting (344+ importers)
- [Ollama telemetry clarification — issue #2567](https://github.com/ollama/ollama/issues/2567) — confirmed: no prompt data leaves machine
- [VTE GitLab issue #88](https://gitlab.gnome.org/GNOME/vte/-/issues/88) — object:text-changed events unreliable
- [natefinch/lumberjack](https://github.com/natefinch/lumberjack) — log rotation

### Tertiary (LOW-MEDIUM confidence, needs validation)

- [Trail of Bits: ANSI codes in MCP (Apr 2025)](https://blog.trailofbits.com/2025/04/29/deceiving-users-with-ansi-terminal-codes-in-mcp/) — ANSI injection attack via terminal output (HIGH confidence on the finding itself)
- [ANSI Escape Injection in Codex CLI (Feb 2026)](https://dganev.com/posts/2026-02-12-ansi-escape-injection-codex-cli/) — recent real-world incident confirming the threat
- [Cisco Talos: Exposed Ollama servers](https://blogs.cisco.com/security/detecting-exposed-llm-servers-shodan-case-study-on-ollama) — 1100+ publicly accessible Ollama servers (motivates localhost validation)
- [KDE D-Bus interface documentation](https://docs.kde.org/) — Konsole session D-Bus interface; absence of content-read method confirmed
- [react-shiki GitHub README](https://github.com/AVGVSTVS96/react-shiki) — streaming delay prop; WASM size
- [Zustand vs Jotai performance guide (ReactLibraries 2025)](https://www.reactlibraries.com/blog/zustand-vs-jotai-vs-valtio-performance-guide-2025) — state management recommendation

---

*Research completed: 2026-03-25*
*Ready for roadmap: yes*
