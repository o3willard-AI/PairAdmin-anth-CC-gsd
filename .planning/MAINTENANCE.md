# PairAdmin v1.0 — Maintenance & Operations Notes

**Written:** 2026-04-03
**Author:** Post-v1.0 release

---

## Known Limitations Accepted at v1.0

These were explicitly accepted during final v1 sign-off. Track as v2 backlog items:

| ID | Description | Workaround |
|----|-------------|------------|
| CHAT-05 | Per-tab chat isolation — tabs may share history in edge cases | Restart app to reset state |
| CHAT-06 | `/clear` slash command not wired to chat history | Use app restart |
| CMD-02 | Command sidebar not guaranteed reverse-chronological | Visual only — commands still functional |
| CMD-05 | "Clear History" button in command sidebar missing | Restart app |
| AppImage | WebKit subprocess path isolation fails at runtime (Wails Issue #4313) | Use `.deb` or `.rpm` instead |
| SHELL-01 | Not formally regression-tested as a launched window | Verified via wails build success + human install test |

---

## Rebuild Instructions

### Prerequisites

```bash
# Ubuntu 22.04 / 24.04
sudo apt install -y golang-go nodejs npm libwebkit2gtk-4.1-dev at-spi2-core gcc
go install github.com/wailsapp/wails/v2/cmd/wails@latest
go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest

# Confirm webkit tag required on Ubuntu 24.04
# wails build uses -tags webkit2_41 automatically via build constraints
```

### Build binary

```bash
wails build -platform linux/amd64
# Output: build/bin/pairadmin
```

### Package

```bash
# Set version
export VERSION=1.0.1

# .deb and .rpm
nfpm pkg --packager deb -f nfpm.yaml
nfpm pkg --packager rpm -f nfpm.yaml

# AppImage (best-effort)
mkdir -p PairAdmin.AppDir/usr/bin
cp build/bin/pairadmin PairAdmin.AppDir/usr/bin/
# ... (see 07-02-SUMMARY.md for full AppDir layout)
./appimagetool PairAdmin.AppDir pairadmin_${VERSION}_linux_amd64.AppImage

# Checksums
sha256sum pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage > SHA256SUMS
```

### Release

```bash
git tag v${VERSION}
git push origin v${VERSION}

# Upload via curl or gh CLI once installed:
# gh release create v${VERSION} pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage SHA256SUMS \
#   --title "PairAdmin v${VERSION}" --notes-file RELEASE_NOTES.md
```

---

## Dependency Watch

| Dependency | Current | Watch For |
|------------|---------|-----------|
| `github.com/wailsapp/wails/v2` | v2.x | Wails v3 is in alpha — API-incompatible, do not upgrade without planning |
| `github.com/awnumar/memguard` | v0.23.0 | Security-critical — monitor for CVEs |
| `github.com/99designs/keyring` | latest | OS keychain API changes per distro |
| `github.com/ollama/ollama/api` | latest | Ollama API changes with new model releases |
| `react-shiki` | latest | Shiki highlighter API is in flux |
| `@base-ui/react` | latest | Pre-1.0, breaking changes expected |
| WebKit2GTK | system | Ubuntu 24.04 requires `webkit2_41` build tag — check on new LTS releases |

---

## Common User Issues (Anticipated)

**"App won't launch — white screen"**
→ WebKit2GTK not installed. Run `scripts/install-deps.sh` or manually:
`sudo apt install -y libwebkit2gtk-4.1-0 at-spi2-core`

**"Terminal tabs not appearing"**
→ Check tmux is running: `tmux ls`. AT-SPI2 requires accessibility enabled:
`gsettings set org.gnome.desktop.interface accessibility true`

**"Ollama connection refused"**
→ Ollama must be running locally: `ollama serve`. Remote Ollama hosts are blocked by design (data residency).

**"API key not saving"**
→ Keychain may require unlock on first use. On headless systems, `99designs/keyring` falls back to file-based storage in `~/.pairadmin/`.

**"AppImage fails to launch"**
→ Known limitation — WebKit subprocess paths are hardcoded in the system WebKit. Use `.deb` instead.

**"Audit log growing too large"**
→ Log rotates daily at 100MB, retained 30 days. Location: `~/.pairadmin/logs/`. Manual cleanup: `rm ~/.pairadmin/logs/audit-*.jsonl`

---

## Security Response

If a credential leak or security issue is reported:

1. Check audit log first: `~/.pairadmin/logs/audit-*.jsonl` — all AI interactions are logged with credential filtering applied
2. The filter pipeline is in `services/llm/filter/` — gitleaks-pattern regex, not the gitleaks binary
3. memguard Enclave lifecycle: keys sealed at startup, opened only during HTTP header construction, purged on exit
4. If a provider credential is suspected leaked: rotate the key immediately, then investigate filter bypass

---

## Architecture Quick Reference

```
main.go
├── services/llm_service.go       — LLM streaming, audit, filter pipeline
├── services/commands.go          — Command sidebar, clipboard, audit
├── services/settings_service.go  — Config, keychain, Enclave management
├── services/capture/             — CaptureManager, TmuxAdapter, ATSPIAdapter
├── services/audit/               — AuditLogger (slog + lumberjack)
├── services/llm/                 — Provider adapters (OpenAI, Anthropic, Ollama, LM Studio, OpenRouter)
│   └── filter/                   — ANSI strip + credential redaction pipeline
├── services/config/              — Viper config loader
└── services/keychain/            — 99designs/keyring wrapper

frontend/src/
├── components/chat/              — ChatPane, ChatMessageList, CodeBlock
├── components/layout/            — ThreeColumnLayout, StatusBar
├── components/settings/          — 5-tab settings dialog
├── components/terminal/          — TerminalPreview (xterm.js)
├── components/commands/          — CommandSidebar
└── stores/                       — Zustand: chatStore, terminalStore, commandStore
```

---

## v2 Priorities (from v1 accepted limitations + roadmap)

1. Fix CHAT-05/06, CMD-02/05 — tab isolation and clear history
2. macOS Terminal.app adapter (needs Mac hardware for QA)
3. SQLite chat history persistence across restarts
4. GPG artifact signing for releases
5. Wails v3 migration (typed events, native packaging)
6. GitHub Actions CI/CD for automated release builds
