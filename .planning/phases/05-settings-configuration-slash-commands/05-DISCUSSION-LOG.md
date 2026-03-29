# Phase 5: Settings, Configuration & Slash Commands - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-29
**Phase:** 05-settings-configuration-slash-commands
**Areas discussed:** Settings dialog UX, Config + keychain architecture, Slash command routing, Clipboard auto-clear

---

## Settings Dialog UX

| Option | Description | Selected |
|--------|-------------|----------|
| Tabbed modal | Click gear → modal overlay with 5 tabs. No new layout code. | ✓ |
| Slide-in drawer | Panel slides in from right, pushing layout left. Needs Drawer component. | |
| Replaces center column | Gear click swaps chat pane for settings view. | |

**User's choice:** Tabbed modal (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Inline status below button | Spinner → green ✓ or red ✗ inline below Test button | ✓ |
| Toast notification | Brief toast in corner — result disappears while user tries to fix error | |
| Modal result dialog | Second modal for result — overkill for a one-liner | |

**User's choice:** Inline status below the button (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Save button per tab | Each tab has own Save. Changes write on explicit save. | ✓ |
| Apply immediately | Changes take effect live on input. Risky for API key fields. | |
| Single Save at footer | One Save saves all tabs. User must visit every tab. | |

**User's choice:** Save button per tab (Recommended)

---

## Config + Keychain Architecture

| Option | Description | Selected |
|--------|-------------|----------|
| Viper for provider/model, env vars as fallback | Settings writes to config.yaml; env vars as fallback on first run | ✓ |
| Env vars only, no Viper for provider | Inconsistent with Phase 4 Viper path | |
| Viper only, drop env var support | Breaking change — not recommended for v1 | |

**User's choice:** Viper for provider/model, env vars as fallback (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| pairadmin + provider name | service="pairadmin", key="openai" etc. One entry per provider. | ✓ |
| pairadmin + full field name | key="OPENAI_API_KEY" — mirrors env vars but redundant | |
| You decide | Claude picks keychain identifiers | |

**User's choice:** pairadmin + provider name (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Show placeholder, clear to replace | "•••••••• (stored)" when key exists; clear field to replace | ✓ |
| Show partial key | "sk-ab••••••••" — leaks partial secret to screen | |
| Always empty on open | User must re-enter key every time — poor UX | |

**User's choice:** Show placeholder, clear to replace (Recommended)

---

## Slash Command Routing

| Option | Description | Selected |
|--------|-------------|----------|
| Frontend router + mixed backend | Frontend dispatches: pure-frontend (/clear, /theme, /help) vs Go calls (/model, /filter, /export, /rename) | ✓ |
| All in Go backend | Single Go CommandDispatch binding — /theme and /help awkward in Go | |
| All in frontend | All TypeScript — /model and /filter need Wails event wiring back to Go | |

**User's choice:** Frontend router + mixed backend (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| System message in chat | Italic muted inline system message — consistent with Phase 4 /filter pattern | ✓ |
| Toast notification | Fine for one-liners but /help content wouldn't fit | |
| Mix: system for content, toast for confirmations | More nuanced but adds router complexity | |

**User's choice:** System message in chat (Recommended)

---

## Clipboard Auto-Clear

| Option | Description | Selected |
|--------|-------------|----------|
| Go backend goroutine | After clipboard write, goroutine sleeps interval then clears. Not subject to frontend lifecycle. | ✓ |
| Frontend timer in useWailsClipboard | setTimeout after copy — lost if component unmounts | |

**User's choice:** Go backend, after clipboard write (Recommended)

| Option | Description | Selected |
|--------|-------------|----------|
| Terminals tab | Grouped with AT-SPI2 polling interval — both are timing settings | ✓ |
| Separate Clipboard section in Terminals tab | Same tab, visually separated heading | |
| You decide | Claude picks placement | |

**User's choice:** Terminals tab (Recommended)

---

## Claude's Discretion

- Tab component choice for Settings modal
- Dialog component choice
- `AppConfig` struct field naming for new settings
- Hotkeys tab implementation approach
- `/export` file path convention

## Deferred Ideas

None.
