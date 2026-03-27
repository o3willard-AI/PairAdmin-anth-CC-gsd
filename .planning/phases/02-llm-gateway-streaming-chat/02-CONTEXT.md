# Phase 2: LLM Gateway & Streaming Chat - Context

**Gathered:** 2026-03-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace the Phase 1 mock echo response with real LLM responses. Implement a Go provider interface with adapters for OpenAI, Anthropic, Ollama, OpenRouter, and LM Studio. Stream responses token-by-token to the React frontend via Wails EventsEmit with sequence numbers and 50ms batching. Display streamed responses with react-shiki syntax highlighting, blinking cursor indicator, and "Copy to Terminal" buttons on code blocks. Prepend last 200 lines of terminal context to every message. Apply ANSI stripping and credential redaction before transmission. Show token count in status bar.

</domain>

<decisions>
## Implementation Decisions

### Phase 2 Provider Configuration
- API keys supplied via environment variables: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY` (Ollama/LM Studio need no key)
- Active provider/model selected via env vars: `PAIRADMIN_PROVIDER` (openai|anthropic|ollama|openrouter|lmstudio) and `PAIRADMIN_MODEL`
- All 5 providers implemented in Phase 2: OpenAI + Anthropic + Ollama + OpenRouter + LM Studio (OpenRouter/LM Studio reuse OpenAI adapter with custom BaseURL, zero extra code)
- Config read at `app.go` startup into a `Config` struct passed to LLMService

### Streaming Visual UX
- Blinking cursor `▋` appended to last token while streaming
- Auto-scroll to bottom only if user is already within 100px of the bottom — does not hijack scroll if user scrolled up to review earlier messages
- "Copy to Terminal" button appears after the code block fence closes (stream completes the block) — never mid-block
- react-shiki `delay={50}` — matches the Wails 50ms batch window for smooth rendering

### System Prompt & Terminal Context
- System prompt: "You are a terminal assistant. The user shares their terminal output with you. Help them understand errors, suggest commands, and explain output."
- Last 200 lines of xterm.js buffer prepended to user message as a fenced code block: `` ```terminal\n{lines}\n``` `` followed by the user's question
- When terminal buffer is empty or unavailable: send question without terminal context (no error, omit the code block silently)

### Error Handling Display
- LLM errors shown as error bubbles in chat (assistant role, amber/red styling, error icon) — in-context, not disruptive
- Rate limit (429): "Rate limit reached. Wait a moment and try again." with retry button
- Auth error (401/403): "API key invalid or missing. Set `PAIRADMIN_PROVIDER` and the key env var."
- Stream interrupted mid-response: show partial response + "(stream interrupted)" suffix — preserves what was received

### Claude's Discretion
- Exact Go package structure for LLMService and provider adapters (suggested: `services/llm/`)
- Token counting implementation (approximate client-side or use provider-returned counts)
- Wails event names for streaming chunks (e.g., `llm:chunk`, `llm:done`, `llm:error`)
- Frontend state shape for streaming messages (extend existing `chatStore.isStreaming` field)

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `frontend/src/stores/chatStore.ts` — `ChatMessage` has `isStreaming: boolean`; `addAssistantMessage` exists; needs `appendStreamChunk` and `setStreaming` actions
- `frontend/src/stores/commandStore.ts` — `addCommand` for "Copy to Terminal" button to call
- `frontend/src/stores/terminalStore.ts` — `activeTabId` for per-tab isolation
- `frontend/src/components/chat/ChatPane.tsx` — `handleSend` to replace mock echo with real LLM call
- `services/commands.go` — pattern for Go service with Wails binding; `app.go` `OnStartup` closure pattern for service lifecycle
- `main.go` — `Bind[]` slice pattern for registering services

### Established Patterns
- Go services live in `services/` package; bound to Wails via `Bind[]` in `main.go`
- `OnStartup` closure calls service `.Startup(ctx)` for lifecycle management
- Wails `runtime.EventsEmit(ctx, eventName, data)` for Go→frontend events
- Frontend uses Wails JS runtime: `window.runtime.EventsOn(eventName, callback)` or generated bindings
- Zustand + Immer for all frontend state; devtools middleware in stores
- Env var pattern: `os.Getenv("KEY")` — no Viper needed for Phase 2

### Integration Points
- Replace `setTimeout(() => addAssistantMessage(...), 200)` in `ChatPane.tsx` with real Wails call
- `chatStore` needs new actions: `startStreamingMessage`, `appendChunk`, `finalizeMessage`
- Terminal content from `TerminalPreview.tsx` xterm.js buffer accessible via `term.buffer.active`
- Status bar in `StatusBar.tsx` needs token count props wired from store

</code_context>

<specifics>
## Specific Ideas

- OpenRouter and LM Studio both reuse the OpenAI adapter — just configure `BaseURL` (`https://openrouter.ai/api/v1` for OpenRouter, `http://localhost:1234/v1` for LM Studio) and appropriate API key
- Wails Issue #2759 mitigation: 50ms batching with sequence numbers prevents dropped events under rapid streaming
- ANSI stripping is security requirement (Stage 1 of filter pipeline) — strip before sending to LLM, not just for display
- gitleaks patterns as foundation for credential redaction (established in research)
- Anthropic system prompt goes in `MessageNewParams.System` (top-level field), not as a role in messages array — adapter must handle this split

</specifics>

<deferred>
## Deferred Ideas

- Persistent API key storage (Phase 5 — Settings & Configuration)
- Model selection UI (Phase 5)
- All 4 providers beyond OpenAI+Anthropic+Ollama require Phase 5 settings to be useful long-term, but adapters built now
- Streaming abort/cancel button (nice-to-have, post-Phase 2)
- Conversation history management beyond current tab (Phase 5)

</deferred>
