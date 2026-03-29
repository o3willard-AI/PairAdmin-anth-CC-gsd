# Phase 5: Settings, Configuration & Slash Commands - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

A fully configurable application: settings dialog (5 tabs), OS keychain API key storage via `99designs/keyring`, Viper config persistence for non-secret settings, all 8 slash commands, and clipboard auto-clear. No new terminal capture adapters, no chat history persistence (deferred to v2).

New capabilities delivered: Settings modal with LLM Config / Prompts / Terminals / Hotkeys / Appearance tabs; `services/config` expansion to cover provider/model/settings; keychain integration; connection test; slash command router in ChatPane; /model, /context, /refresh, /export, /rename, /theme, /help; clipboard auto-clear goroutine in CommandService.

Out of scope: SQLite chat history persistence, macOS/Windows adapters, Wails v3 migration.

</domain>

<decisions>
## Implementation Decisions

### Settings Dialog UX
- **D-01:** Settings opens as a **tabbed modal overlay** triggered by the gear icon in `StatusBar.tsx` (currently disabled). Modal overlays `ThreeColumnLayout` — no new layout panels needed.
- **D-02:** Five tabs: LLM Config / Prompts / Terminals / Hotkeys / Appearance. Each tab has its own **Save button** — changes write to Viper/keychain only on explicit save. User can cancel without affecting the current session.
- **D-03:** LLM Config tab connection test result appears **inline below the Test button** — spinner while testing, then green ✓ Connected or red ✗ error message. No toast or second modal.

### Config + Keychain Architecture
- **D-04:** Provider and model are stored in `~/.pairadmin/config.yaml` via Viper (`services/config`). On startup: Viper config takes priority; env vars (`PAIRADMIN_PROVIDER`, `PAIRADMIN_MODEL`) are the fallback if no config file exists. Existing env-var users keep working without change until they open Settings.
- **D-05:** API keys use `99designs/keyring` with **service = "pairadmin"**, **key = provider name** (e.g., "openai", "anthropic", "openrouter", "lmstudio"). One keychain entry per provider. Keys are never written to `~/.pairadmin/config.yaml`.
- **D-06:** API key input in the settings dialog shows **`•••••••• (stored)`** as placeholder when a key exists in the keychain. User clears the field and types a new key to replace. Keychain write only happens on Save. Field is blank (not placeholder) when no key is stored.

### Slash Command Routing
- **D-07:** ChatPane has a **frontend slash command router** that dispatches based on command prefix:
  - **Frontend-only** (no Wails call): `/clear` (existing), `/theme`, `/help`
  - **Go backend call**: `/model`, `/context`, `/refresh`, `/export`, `/rename`, `/filter` (existing)
  - Consistent with the existing `/filter` → `FilterCommand` pattern from Phase 4.
- **D-08:** All slash command output (confirmations, listings, error messages) appears as a **system message inline in the chat** — italic, muted styling, consistent with Phase 4's `/filter` output pattern. `/help` renders a formatted list as a system message.

### Clipboard Auto-Clear
- **D-09:** After a successful clipboard write in `CommandService.CopyToClipboard`, a **goroutine** sleeps for the configured interval and then clears the clipboard (writes an empty string via the same wl-clipboard/xclip path). Timer runs server-side — not subject to frontend component lifecycle.
- **D-10:** The auto-clear interval is configurable via the **Terminals tab** in Settings (grouped with the AT-SPI2 polling interval). Default: 60 seconds. Stored in `~/.pairadmin/config.yaml` via Viper.

### Claude's Discretion
- Tab component choice for the Settings modal (shadcn Tabs vs custom, `@radix-ui/react-tabs` vs `@base-ui/react`)
- Dialog component choice (shadcn Dialog vs `@base-ui/react` Popover/Dialog)
- `AppConfig` struct field naming for new settings (provider, model, polling interval, clipboard clear interval)
- Hotkeys tab implementation approach (capture key combination on focus, store as string)
- `/export` file path choice (e.g., `~/pairadmin-export-YYYY-MM-DD.json`)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Config Infrastructure
- `services/config/config.go` — Viper AppConfig with `CustomPatterns`; expand `AppConfig` struct in this file
- `services/llm_service.go` — `Config` struct + `LoadConfig()` reads env vars; startup logic must be updated to prefer Viper over env vars

### Frontend Integration Points
- `frontend/src/components/layout/StatusBar.tsx` — Gear icon button (line ~32); wire `onClick` to open settings modal
- `frontend/src/components/layout/ThreeColumnLayout.tsx` — Where settings modal state and component will be mounted
- `frontend/src/components/chat/ChatPane.tsx` — Current slash command handling (`/clear`, `/filter`); router goes here
- `frontend/src/stores/chatStore.ts` — `addSystemMessage` action for slash command output; `clearTab` for `/clear`
- `frontend/src/components/ui/` — Existing: `button.tsx`, `tooltip.tsx`, `scroll-area.tsx`. New: Dialog and Tabs components needed

### Commands Service (clipboard timer)
- `services/commands.go` — `CopyToClipboard` method; clipboard auto-clear goroutine added here

### Phase Context
- `.planning/phases/04-linux-gui-terminal-adapters-at-spi2/04-CONTEXT.md` — Phase 4 decisions: Viper path established (D-09), `services/config` package structure, system message pattern (D-08, D-10, D-11)
- `.planning/REQUIREMENTS.md` §CFG-01–08, SLASH-01–08, CLIP-03 — Acceptance criteria for this phase

### No external specs
- `99designs/keyring` is not yet in go.mod — needs `go get github.com/99designs/keyring`
- Note: `zalando/go-keyring` was rejected in earlier research (fails in headless Linux) — use `99designs/keyring` only

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `services/config/config.go` — `LoadAppConfig`/`SaveAppConfig` with Viper; expand `AppConfig` struct rather than creating a new config service
- `services/llm_service.go` — `FilterCommand` binding pattern is the template for all new Go-side slash command bindings
- `frontend/src/stores/chatStore.ts` — `addSystemMessage(tabId, text)` ready to use for all slash command output
- `frontend/src/components/chat/ChatPane.tsx` — Existing `/filter` dispatch with dynamic import is the pattern for all new Go-side slash dispatches
- `services/commands.go` — `CopyToClipboard` is the exact insertion point for the auto-clear goroutine

### Established Patterns
- Injectable function fields for test isolation (Phase 3/4) — use same pattern for keychain reads/writes to enable unit testing
- System message rendering (italic, muted, `whitespace-pre-wrap`) from Phase 4 `ChatMessageList.tsx` — no changes needed for new slash command output
- `/* @vite-ignore */` dynamic imports for Wails bindings — use for new Go bindings (e.g., `SettingsService`)
- Viper `LoadAppConfig`/`SaveAppConfig` pattern — replicate for new `AppConfig` fields (don't create a second config file)

### Integration Points
- `StatusBar.tsx` gear `<button>` (line ~32, currently `disabled`) → add `onClick` prop → modal open state in `ThreeColumnLayout.tsx`
- `ThreeColumnLayout.tsx` — mount `<SettingsDialog open={open} onClose={...} />` here alongside existing hook calls
- `ChatPane.tsx` slash router → `frontend/wailsjs/go/services/SettingsService.js` stub needed for vitest

</code_context>

<specifics>
## Specific Ideas

- **Key masking display**: `•••••••• (stored)` placeholder — the bullet character `•` (U+2022) renders cleanly in the existing dark zinc theme
- **Terminals tab layout**: AT-SPI2 polling interval (ms, slider or number input) + Clipboard auto-clear interval (seconds, number input with default 60) grouped under a "Capture" section heading
- **StatusBar live update**: After `/model` command or settings Save, StatusBar "No model" text should update to reflect the active provider:model. This requires a `settingsStore` or Wails event from the backend.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 05-settings-configuration-slash-commands*
*Context gathered: 2026-03-29*
