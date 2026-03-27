# Phase 2: LLM Gateway & Streaming Chat - Research

**Researched:** 2026-03-26
**Domain:** Go multi-provider LLM streaming + Wails EventsEmit + React streaming chat UI
**Confidence:** HIGH (all primary libraries verified via proxy.golang.org and npm registry; prior dedicated research document confirmed and extended)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- API keys supplied via environment variables: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY` (Ollama/LM Studio need no key)
- Active provider/model selected via env vars: `PAIRADMIN_PROVIDER` (openai|anthropic|ollama|openrouter|lmstudio) and `PAIRADMIN_MODEL`
- All 5 providers implemented in Phase 2: OpenAI + Anthropic + Ollama + OpenRouter + LM Studio (OpenRouter/LM Studio reuse OpenAI adapter with custom BaseURL, zero extra code)
- Config read at `app.go` startup into a `Config` struct passed to LLMService
- Blinking cursor `▋` appended to last token while streaming
- Auto-scroll to bottom only if user is already within 100px of the bottom
- "Copy to Terminal" button appears after the code block fence closes — never mid-block
- react-shiki `delay={50}` — matches the Wails 50ms batch window for smooth rendering
- System prompt: "You are a terminal assistant. The user shares their terminal output with you. Help them understand errors, suggest commands, and explain output."
- Last 200 lines of xterm.js buffer prepended as a fenced code block: `` ```terminal\n{lines}\n``` `` followed by the user's question
- When terminal buffer is empty or unavailable: send question without terminal context (no error, omit silently)
- LLM errors shown as error bubbles in chat (assistant role, amber/red styling, error icon)
- Rate limit (429): "Rate limit reached. Wait a moment and try again." with retry button
- Auth error (401/403): "API key invalid or missing. Set `PAIRADMIN_PROVIDER` and the key env var."
- Stream interrupted mid-response: show partial response + "(stream interrupted)" suffix

### Claude's Discretion

- Exact Go package structure for LLMService and provider adapters (suggested: `services/llm/`)
- Token counting implementation (approximate client-side or use provider-returned counts)
- Wails event names for streaming chunks (e.g., `llm:chunk`, `llm:done`, `llm:error`)
- Frontend state shape for streaming messages (extend existing `chatStore.isStreaming` field)

### Deferred Ideas (OUT OF SCOPE)

- Persistent API key storage (Phase 5)
- Model selection UI (Phase 5)
- All 4 providers beyond OpenAI+Anthropic+Ollama require Phase 5 settings to be useful long-term, but adapters built now
- Streaming abort/cancel button (nice-to-have, post-Phase 2)
- Conversation history management beyond current tab (Phase 5)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LLM-01 | OpenAI provider via `github.com/openai/openai-go/v3`; supports streaming chat completions | Section: Standard Stack, OpenAI adapter pattern |
| LLM-02 | Anthropic provider via `github.com/anthropics/anthropic-sdk-go`; system prompt as top-level field | Section: Anthropic adapter pattern, system prompt split |
| LLM-03 | Ollama provider via `github.com/ollama/ollama/api`; validates OLLAMA_HOST is localhost | Section: Ollama adapter, localhost validation |
| LLM-04 | LM Studio and llama.cpp supported by reusing OpenAI adapter with configurable base URL | Section: Standard Stack (zero extra code path) |
| LLM-05 | All providers implement common channel-based streaming interface: `Stream() (<-chan StreamChunk, error)` | Section: Architecture Patterns, Provider interface |
| LLM-06 | Streaming responses via Wails EventsEmit with sequence numbers and 50ms batching | Section: Wails streaming pattern, Issue #2759 mitigation |
| LLM-07 | When Ollama selected, no terminal content transmitted over any network interface | Section: Ollama localhost validation |
| CHAT-02 | Every outgoing message includes current terminal context (filtered) assembled as system prompt prefix | Section: xterm.js buffer reading, terminal context assembly |
| CHAT-03 | AI responses stream token-by-token into chat area as they arrive | Section: Wails EventsEmit, chatStore streaming actions |
| CHAT-04 | AI-suggested commands rendered in syntax-highlighted code blocks (react-shiki) with "Copy to Terminal" button | Section: react-shiki, code block detection |
| FILT-01 | ANSI/VT100 escape sequences stripped before any processing | Section: ANSI stripping, leaanthony/go-ansi-parser |
| FILT-02 | Built-in credential filter detects and redacts: AWS keys, GitHub tokens, GCP keys, API keys, SSH keys, DSNs, bearer tokens, password lines | Section: gitleaks credential filter |
| FILT-03 | Filtered/redacted content is what gets sent to the LLM | Section: Filter pipeline |
| FILT-06 | Terminal content truncated to fit active provider's context window; most recent content prioritized | Section: Context window management |
| FILT-07 | Token count and context usage displayed in status bar | Section: Token counting, StatusBar wiring |
</phase_requirements>

---

## Summary

Phase 2 replaces the mock echo response in `ChatPane.tsx` with real LLM responses streaming token-by-token to the frontend. The technical scope spans three layers: a Go LLM gateway with a provider interface and five adapters, a Wails event-streaming bridge with 50ms batching and sequence numbers (mitigating confirmed Issue #2759 event ordering race), and a React streaming chat UI using react-shiki for code block highlighting with "Copy to Terminal" buttons.

All five providers are implemented in this phase. OpenAI, OpenRouter, and LM Studio share one adapter (the latter two simply configure a different BaseURL on the official OpenAI SDK client). Anthropic and Ollama each require their own adapters, with Anthropic requiring that the system message be extracted into the top-level `MessageNewParams.System` field rather than the messages array, and Ollama requiring that its callback-based streaming API be wrapped into the channel-based interface. A two-stage filter runs before every LLM transmission: ANSI stripping (`leaanthony/go-ansi-parser.Cleanse` — already an indirect Wails dependency) and credential redaction (gitleaks v8 patterns via regex). Terminal context is sourced from the xterm.js `buffer.active` line iterator already present in `TerminalPreview.tsx`, limited to the last 200 lines.

The prior dedicated research document (`RESEARCH-LLM-GATEWAY.md`) provides verified, production-quality code patterns for all three SDK streaming models. This research validates and extends those findings, resolves the ANSI stripping library choice (use the already-present `leaanthony/go-ansi-parser`), confirms react-shiki 0.9.2 has the `delay` prop, and maps the xterm.js buffer reading API to the existing `TerminalPreview` component structure.

**Primary recommendation:** Create `services/llm/` package with `provider.go` (interface), `registry.go`, `openai.go`, `anthropic.go`, `ollama.go`, and `filter/` subpackage for ANSI stripping + credential redaction. Wire via `LLMService` struct in `services/llm_service.go` following the same `Startup(ctx)` lifecycle pattern as `CommandService`.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `github.com/openai/openai-go/v3` | v3.30.0 | OpenAI + OpenRouter + LM Studio streaming | Official SDK; Chat Completions maps to multi-provider abstraction |
| `github.com/anthropics/anthropic-sdk-go` | v1.27.1 | Anthropic Claude streaming | Official SDK; MIT license; iterator streaming pattern |
| `github.com/ollama/ollama/api` | v0.18.3 | Ollama local model streaming | Official internal package; callback streaming wrapped to channel |
| `github.com/pkoukk/tiktoken-go` | v0.1.8 | Token counting for OpenAI models | 344+ importers; offline after vocab download; matches Python tiktoken |
| `github.com/cenkalti/backoff/v4` | v4.3.0 | Exponential backoff for retry | Standard Go backoff library; context-aware |
| `react-shiki` | 0.9.2 | Syntax-highlighted code blocks with streaming | Has `delay` prop; Shiki/TextMate grammars; confirmed current latest |
| `react-markdown` | 10.1.0 | Markdown rendering for chat text | Standard choice; remark/rehype pipeline |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `github.com/leaanthony/go-ansi-parser` | v1.6.1 | ANSI/VT100 stripping (`Cleanse` function) | Already indirect dep via Wails — no new dependency needed |
| `github.com/zricethezav/gitleaks/v8` | v8.30.1 | Credential pattern detection | Foundation for FILT-02; 150+ compiled regex rules |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `leaanthony/go-ansi-parser.Cleanse` | `acarl005/stripansi` | stripansi is simpler but requires adding a new dep; go-ansi-parser is already in go.sum via Wails |
| `react-shiki` | `react-syntax-highlighter` | react-syntax-highlighter lacks streaming delay prop; react-shiki is already the decided choice |
| Full gitleaks library | Hand-rolled regex | Hand-rolling misses 150+ maintained patterns; gitleaks is the decision |

**Installation (Go):**
```bash
go get github.com/openai/openai-go/v3
go get github.com/anthropics/anthropic-sdk-go
go get github.com/ollama/ollama@v0.18.3
go get github.com/pkoukk/tiktoken-go
go get github.com/cenkalti/backoff/v4
go get github.com/zricethezav/gitleaks/v8
```

**Installation (frontend):**
```bash
cd frontend && npm install react-shiki react-markdown
```

**Version verification (confirmed 2026-03-26 via proxy.golang.org and npm registry):**
- openai-go/v3: v3.30.0 (published 2026-03-25)
- anthropic-sdk-go: v1.27.1 (published 2026-03-18)
- ollama: v0.18.3
- tiktoken-go: v0.1.8
- backoff/v4: v4.3.0
- react-shiki: 0.9.2 (latest, confirmed via `npm view react-shiki version`)
- react-markdown: 10.1.0 (confirmed via `npm view react-markdown version`)

---

## Architecture Patterns

### Recommended Project Structure
```
services/
├── llm/
│   ├── provider.go          # Provider interface, StreamChunk, Message, Usage types
│   ├── registry.go          # Registry struct; Get/Register methods
│   ├── openai.go            # OpenAIProvider (also handles OpenRouter, LM Studio via BaseURL)
│   ├── anthropic.go         # AnthropicProvider
│   ├── ollama.go            # OllamaProvider; localhost validation
│   ├── context.go           # BuildMessages, TruncateTerminal, TrimHistory
│   └── filter/
│       ├── filter.go        # Filter interface; Pipeline type
│       ├── ansi.go          # ANSIFilter: leaanthony/go-ansi-parser.Cleanse wrapper
│       └── credential.go    # CredentialFilter: gitleaks Detector wrapper
├── llm_service.go           # LLMService: Wails-bound struct; SendMessage; Startup(ctx)
└── commands.go              # Existing CommandService (unchanged)
```

### Pattern 1: Provider Interface (Channel-Based Streaming)
**What:** A minimal Go interface all LLM adapters implement, returning `<-chan StreamChunk`
**When to use:** All five providers; the Wails event emitter consumes the channel in a goroutine

```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 4)
// services/llm/provider.go

type Role string
const (
    RoleSystem    Role = "system"
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
)

type Message struct {
    Role    Role
    Content string
}

type StreamChunk struct {
    Text  string
    Done  bool
    Error error
}

type Provider interface {
    Name() string
    Stream(ctx context.Context, messages []Message) (<-chan StreamChunk, error)
    TestConnection(ctx context.Context) error  // for Phase 5 settings dialog
}
```

Add `TestConnection` to the interface now even though Phase 5 uses it — this avoids an interface break later.

### Pattern 2: LLMService Wails Binding (Goroutine + EventsEmit)
**What:** The Go service exposed to Wails returns immediately from `SendMessage`; tokens arrive via events
**When to use:** All streaming; never buffer all tokens and emit once

```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 7)
// services/llm_service.go

type ChatTokenEvent struct {
    Seq   int    `json:"seq"`    // sequence number for out-of-order detection
    Text  string `json:"text"`
    Done  bool   `json:"done"`
    Error string `json:"error,omitempty"`
}

func (s *LLMService) SendMessage(tabId, userInput string) error {
    // Build messages with terminal context
    messages := s.buildMessages(tabId, userInput)

    go func() {
        ctx, cancel := context.WithTimeout(s.ctx, 5*time.Minute)
        defer cancel()

        ch, err := s.activeProvider.Stream(ctx, messages)
        if err != nil {
            runtime.EventsEmit(s.ctx, "llm:error", ChatTokenEvent{
                Error: err.Error(), Done: true,
            })
            return
        }

        seq := 0
        var batch []string
        ticker := time.NewTicker(50 * time.Millisecond)
        defer ticker.Stop()

        flush := func() {
            if len(batch) == 0 {
                return
            }
            runtime.EventsEmit(s.ctx, "llm:chunk", ChatTokenEvent{
                Seq:  seq,
                Text: strings.Join(batch, ""),
            })
            seq++
            batch = batch[:0]
        }

        for {
            select {
            case chunk, ok := <-ch:
                if !ok {
                    flush()
                    runtime.EventsEmit(s.ctx, "llm:done", ChatTokenEvent{Seq: seq, Done: true})
                    return
                }
                if chunk.Error != nil {
                    flush()
                    runtime.EventsEmit(s.ctx, "llm:error", ChatTokenEvent{
                        Seq: seq, Error: chunk.Error.Error(), Done: true,
                    })
                    return
                }
                if chunk.Done {
                    flush()
                    runtime.EventsEmit(s.ctx, "llm:done", ChatTokenEvent{Seq: seq, Done: true})
                    return
                }
                batch = append(batch, chunk.Text)
            case <-ticker.C:
                flush()
            case <-ctx.Done():
                return
            }
        }
    }()

    return nil
}
```

**50ms batching rationale:** Wails Issue #2759 confirms out-of-order delivery at rapid-fire emit rates. Batching over 50ms windows collapses bursts into fewer emissions. Sequence numbers let the frontend detect and reorder if needed.

### Pattern 3: React Streaming Hook (Frontend)
**What:** A `useLLMStream` hook that subscribes to Wails events and feeds into chatStore
**When to use:** Replace the mock echo timeout in `ChatPane.tsx`

```typescript
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 7)
// frontend/src/hooks/useLLMStream.ts

import { useEffect } from "react"
import { useChatStore } from "@/stores/chatStore"

export function useLLMStream(tabId: string) {
  const { startStreamingMessage, appendChunk, finalizeMessage, setError } =
    useChatStore.getState()

  useEffect(() => {
    let msgId: string | null = null

    const handleChunk = async (event: { seq: number; text: string }) => {
      if (!msgId) msgId = startStreamingMessage(tabId)
      appendChunk(tabId, msgId, event.text)
    }

    const handleDone = async () => {
      if (msgId) finalizeMessage(tabId, msgId)
      msgId = null
    }

    const handleError = async (event: { error: string }) => {
      if (msgId) setError(tabId, msgId, event.error)
      else startStreamingMessage(tabId)  // create an error bubble
      msgId = null
    }

    // Dynamic import — wailsjs generated at runtime; fallback pattern established in useWailsClipboard
    let unsubChunk: (() => void) | null = null
    let unsubDone: (() => void) | null = null
    let unsubError: (() => void) | null = null

    import(/* @vite-ignore */ "../../wailsjs/runtime/runtime").then((rt) => {
      unsubChunk = rt.EventsOn("llm:chunk", handleChunk)
      unsubDone = rt.EventsOn("llm:done", handleDone)
      unsubError = rt.EventsOn("llm:error", handleError)
    })

    return () => {
      unsubChunk?.()
      unsubDone?.()
      unsubError?.()
    }
  }, [tabId])
}
```

### Pattern 4: chatStore Streaming Actions
**What:** Three new Zustand/Immer actions to manage in-progress assistant messages
**When to use:** chatStore already has `isStreaming: boolean` on `ChatMessage`; extend it

```typescript
// Additions to chatStore.ts
startStreamingMessage: (tabId: string) => string;  // creates message with isStreaming=true, returns id
appendChunk: (tabId: string, msgId: string, text: string) => void;
finalizeMessage: (tabId: string, msgId: string) => void;  // isStreaming=false, strip trailing cursor ▋
setStreamError: (tabId: string, msgId: string | null, errorText: string) => void;
```

The `ChatMessage` interface gains:
```typescript
tokenCount?: number;   // populated on finalizeMessage via provider-returned or estimated count
```

### Pattern 5: Anthropic System Prompt Split
**What:** Anthropic's API takes `system` as a top-level field; the adapter must extract it from the canonical message list

```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 6)
// services/llm/anthropic.go

func (p *AnthropicProvider) buildParams(messages []Message) anthropic.MessageNewParams {
    params := anthropic.MessageNewParams{
        Model:     anthropic.Model(p.model),
        MaxTokens: 2048,
    }
    var userMsgs []anthropic.MessageParam
    for _, m := range messages {
        switch m.Role {
        case RoleSystem:
            params.System = []anthropic.TextBlockParam{{Text: m.Content}}
        case RoleUser:
            userMsgs = append(userMsgs, anthropic.NewUserMessage(anthropic.NewTextBlock(m.Content)))
        case RoleAssistant:
            userMsgs = append(userMsgs, anthropic.NewAssistantMessage(anthropic.NewTextBlock(m.Content)))
        }
    }
    params.Messages = userMsgs
    return params
}
```

### Pattern 6: Ollama Callback-to-Channel Wrapping
**What:** Ollama's SDK uses a callback; wrap it in a goroutine to feed the channel interface
**When to use:** OllamaProvider.Stream()

```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 3 + 4)
// services/llm/ollama.go

func (p *OllamaProvider) Stream(ctx context.Context, messages []Message) (<-chan StreamChunk, error) {
    req := buildOllamaRequest(p.model, messages)
    ch := make(chan StreamChunk, 32)
    go func() {
        defer close(ch)
        err := p.client.Chat(ctx, req, func(resp api.ChatResponse) error {
            if ctx.Err() != nil {
                return ctx.Err()
            }
            if resp.Done {
                ch <- StreamChunk{Done: true}
                return nil
            }
            select {
            case ch <- StreamChunk{Text: resp.Message.Content}:
            case <-ctx.Done():
                return ctx.Err()
            }
            return nil
        })
        if err != nil && !errors.Is(err, context.Canceled) {
            ch <- StreamChunk{Error: err}
        }
    }()
    return ch, nil
}
```

### Pattern 7: Ollama Localhost Validation (LLM-07)
**What:** Reject configurations where `OLLAMA_HOST` points to a non-localhost address
**When to use:** LLMService startup / provider construction

```go
// services/llm/ollama.go
func validateOllamaHost(host string) error {
    if host == "" {
        return nil // default localhost
    }
    u, err := url.Parse(host)
    if err != nil {
        return fmt.Errorf("invalid OLLAMA_HOST: %w", err)
    }
    h := u.Hostname()
    if h != "localhost" && h != "127.0.0.1" && h != "::1" {
        return fmt.Errorf("OLLAMA_HOST must be localhost for security (got %q — terminal content must not leave the machine)", h)
    }
    return nil
}
```

### Pattern 8: xterm.js Buffer Reading (Terminal Context)
**What:** Read last 200 lines from xterm.js `buffer.active` to assemble terminal context
**When to use:** Before each `SendMessage` call

xterm.js 6.x (installed as `@xterm/xterm ^6.0.0`) provides:
- `terminal.buffer.active` — the active buffer object
- `buffer.length` — total line count (includes scrollback)
- `buffer.getLine(y)` — returns `IBufferLine | undefined`
- `bufferLine.translateToString(trimRight?: boolean)` — returns line text

```typescript
// frontend/src/utils/terminalContext.ts
export function readTerminalLines(term: Terminal, maxLines = 200): string {
  const buf = term.buffer.active
  const start = Math.max(0, buf.length - maxLines)
  const lines: string[] = []
  for (let y = start; y < buf.length; y++) {
    const line = buf.getLine(y)
    if (line) lines.push(line.translateToString(true))  // trimRight=true
  }
  return lines.join("\n").trimEnd()
}
```

**Access pattern:** `TerminalPreview.tsx` already stores `termRef.current`. The terminal ref must be exposed upward (via a Map in `terminalStore` or a React context) so `ChatPane.tsx` can read it when the user sends a message.

Recommended: add `termRefs: Map<string, Terminal>` to `terminalStore` with `setTermRef(tabId, term)` and `getTermRef(tabId)`. Called from `TerminalPreview`'s `useEffect` after `term.open()`.

### Pattern 9: Filter Pipeline (ANSI + Credential)
**What:** Two-stage pipeline — ANSI strip first (security), then credential redact
**When to use:** Applied to terminal buffer content before assembling messages

```go
// services/llm/filter/ansi.go
import "github.com/leaanthony/go-ansi-parser"

func StripANSI(s string) (string, error) {
    return ansi.Cleanse(s)
}
```

For credential filtering in Phase 2, the research recommends building a simple regex-based filter using the most critical gitleaks patterns (AWS keys, GitHub tokens, bearer tokens, API key patterns) as a Phase 2 foundation. Full gitleaks library integration (with the `detect.Detector` API) is the Phase 2 deliverable per FILT-02.

Gitleaks library usage:
```go
// services/llm/filter/credential.go
import (
    "github.com/zricethezav/gitleaks/v8/config"
    "github.com/zricethezav/gitleaks/v8/detect"
)

func NewCredentialFilter() (*CredentialFilter, error) {
    cfg, err := config.GetDefaultConfig()
    if err != nil {
        return nil, err
    }
    detector := detect.NewDetector(cfg)
    return &CredentialFilter{detector: detector}, nil
}

func (f *CredentialFilter) Redact(content string) string {
    findings := f.detector.DetectString(content)
    // Replace each found secret span with [REDACTED:<rule_id>]
    // Walk findings in reverse order to preserve offsets
    ...
}
```

**Note on gitleaks library stability:** The `detect.DetectString` API is confirmed present in v8.x. The gitleaks library is primarily a CLI tool; the `detect` package is less documented than the binary interface. Flag as MEDIUM confidence — verify the exact API by inspecting `pkg.go.dev/github.com/zricethezav/gitleaks/v8/detect` at implementation time.

### Pattern 10: Token Counting + Status Bar
**What:** Estimate tokens before send; display in StatusBar; update after stream completes with actual provider counts
**When to use:** Before each send (estimate) and after stream (actual if available)

```go
// services/llm/context.go

// Approximate for Anthropic and Ollama pre-flight
func EstimateTokens(text string) int {
    return int(math.Ceil(float64(len(text)) / 4.0))  // 1 token ≈ 4 chars
}

// After stream: use acc.Usage.TotalTokens (OpenAI), message.Usage (Anthropic),
// resp.EvalCount + resp.PromptEvalCount (Ollama)
```

StatusBar must receive token count via store. Add `tokenCount` to a lightweight `sessionStore` (or reuse `chatStore`) and have `LLMService` emit a `llm:tokens` event updating it.

### Anti-Patterns to Avoid
- **Emitting per-token without batching:** Wails Issue #2759 confirms event loss and ordering failures at high rates. Always batch over 50ms window.
- **Anthropic system message as a role:** Must go in `MessageNewParams.System`, not messages array. Adapter handles the split.
- **Mid-stream retry:** Retry only on initial connection failure. Partial token delivery means the UI already has content.
- **Buffering all tokens before emitting:** Defeats the streaming UX entirely.
- **ANSI strip after credential filter:** ANSI sequences can obfuscate credential patterns. Strip first, always.
- **Not consuming the channel to completion:** Go goroutine leaks if the channel is not drained. Always drain even when cancelling via ctx.
- **Putting terminal ref in React state:** xterm.js Terminal objects must not be stored in React state — use a ref or external map (terminalStore).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ANSI escape stripping | Custom regex | `leaanthony/go-ansi-parser.Cleanse` | Already in go.sum; handles OSC, CSI, and private sequences correctly |
| Exponential backoff | Manual sleep loop | `cenkalti/backoff/v4` | Context-aware; respects Retry-After headers |
| OpenAI streaming accumulation | Manual delta concatenation | `openai.ChatCompletionAccumulator` | Handles tool calls, partial JSON, usage stats |
| Credential pattern detection | Inline regex list | `zricethezav/gitleaks/v8` | 150+ maintained patterns covering AWS, GCP, GitHub, etc. |
| Token counting (OpenAI) | Character division | `pkoukk/tiktoken-go` | Same BPE tables as Python tiktoken; offline |

**Key insight:** Credential detection pattern maintenance is an ongoing security concern. Using gitleaks means PairAdmin inherits upstream fixes for new credential formats without code changes.

---

## Common Pitfalls

### Pitfall 1: Wails EventsEmit Out-of-Order Delivery
**What goes wrong:** Rapid per-token emissions arrive at the frontend out of sequence, causing jumbled text assembly.
**Why it happens:** Wails v2 Issue #2759 — the internal goroutine-based event dispatch has a confirmed race under high emit rates.
**How to avoid:** 50ms batching window on the Go side; sequence numbers on every emission; frontend reorder buffer keyed by sequence number.
**Warning signs:** Streamed text that looks scrambled or has missing tokens.

### Pitfall 2: Anthropic System Prompt in Messages Array
**What goes wrong:** Anthropic API returns 400 "system role not allowed in messages" error.
**Why it happens:** Anthropic's API structure differs from OpenAI — system is a top-level field, not a role.
**How to avoid:** The `AnthropicProvider.buildParams` method must extract `RoleSystem` messages into `MessageNewParams.System`.
**Warning signs:** 400 Bad Request immediately on any Anthropic call.

### Pitfall 3: Goroutine Leak from Unclosed Channel
**What goes wrong:** Provider goroutines accumulate indefinitely if the channel is not drained.
**Why it happens:** Go channels block the sender if the receiver exits early (context cancel, frontend unsubscribe).
**How to avoid:** All channel sends must use `select { case ch <- chunk: case <-ctx.Done(): return }`. Use `defer close(ch)` in producer goroutines.
**Warning signs:** Growing goroutine count visible in `runtime.NumGoroutine()`.

### Pitfall 4: Ollama OLLAMA_HOST Pointing to Remote Server
**What goes wrong:** Terminal content (including filtered but potentially sensitive data) transmitted to a remote Ollama server over an unauthenticated HTTP connection.
**Why it happens:** Ollama's `api.ClientFromEnvironment()` reads `OLLAMA_HOST` directly; no built-in localhost check.
**How to avoid:** `validateOllamaHost()` called at provider construction; `LLMService.SendMessage` returns error if Ollama is active and validation failed.
**Warning signs:** 1100+ publicly accessible Ollama servers confirmed by Cisco Talos — this is a real threat pattern.

### Pitfall 5: xterm.js Terminal Ref in React State
**What goes wrong:** Storing `Terminal` instance in Zustand state causes React re-render cycles; xterm.js may not serialize properly.
**Why it happens:** React state is expected to be serializable; Terminal objects are complex with DOM references.
**How to avoid:** Store Terminal instances in a plain `Map<string, Terminal>` in `terminalStore` using `useRef` semantics (not actual state); update the map imperatively from `TerminalPreview` effect.
**Warning signs:** xterm.js console errors about `dispose()` being called on an already-disposed terminal.

### Pitfall 6: ANSI Sequences Carrying Prompt Injection
**What goes wrong:** Malicious terminal content with `\x1b]0;Ignore previous instructions...\x07` (OSC title sequence) gets sent to the LLM; the model processes the invisible instruction.
**Why it happens:** Trail of Bits April 2025 and a February 2026 Codex CLI incident confirm this attack vector.
**How to avoid:** ANSI strip is unconditional Stage 1 in the filter pipeline; never allow raw terminal bytes to reach the LLM. Use XML delimiters around terminal content in the system prompt.
**Warning signs:** Model behaving strangely or refusing to answer after certain terminal sessions.

### Pitfall 7: react-shiki Receiving Incomplete Code Fence
**What goes wrong:** Partial code block (open triple-backtick without closing) renders incorrectly or causes syntax highlighter errors.
**Why it happens:** Streaming delivers partial markdown; a code block may not be closed for seconds.
**How to avoid:** Parse the streaming content in the frontend to detect open code fences. Only show `ShikiHighlighter` once the closing fence arrives. The "Copy to Terminal" button decision is locked to post-close per CONTEXT.md.
**Warning signs:** UI flicker or malformed code block display during streaming.

---

## Code Examples

### OpenAI Streaming Adapter

```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 1)
// services/llm/openai.go

func (p *OpenAIProvider) Stream(ctx context.Context, messages []Message) (<-chan StreamChunk, error) {
    params := openai.ChatCompletionNewParams{
        Model:    openai.ChatModel(p.model),
        Messages: convertMessages(messages),
    }
    sdkStream := p.client.Chat.Completions.NewStreaming(ctx, params)

    ch := make(chan StreamChunk, 32)
    go func() {
        defer close(ch)
        acc := openai.ChatCompletionAccumulator{}
        for sdkStream.Next() {
            chunk := sdkStream.Current()
            acc.AddChunk(chunk)
            if len(chunk.Choices) > 0 {
                text := chunk.Choices[0].Delta.Content
                if text != "" {
                    select {
                    case ch <- StreamChunk{Text: text}:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }
        if err := sdkStream.Err(); err != nil {
            ch <- StreamChunk{Error: err}
            return
        }
        ch <- StreamChunk{Done: true}
    }()
    return ch, nil
}
```

### Config Loading at Startup
```go
// app.go (or services/llm_service.go)
type Config struct {
    Provider         string // PAIRADMIN_PROVIDER
    Model            string // PAIRADMIN_MODEL
    OpenAIKey        string // OPENAI_API_KEY
    AnthropicKey     string // ANTHROPIC_API_KEY
    OpenRouterKey    string // OPENROUTER_API_KEY
    LMStudioBaseURL  string // defaults to http://localhost:1234/v1
}

func LoadConfig() Config {
    return Config{
        Provider:        os.Getenv("PAIRADMIN_PROVIDER"),
        Model:           os.Getenv("PAIRADMIN_MODEL"),
        OpenAIKey:       os.Getenv("OPENAI_API_KEY"),
        AnthropicKey:    os.Getenv("ANTHROPIC_API_KEY"),
        OpenRouterKey:   os.Getenv("OPENROUTER_API_KEY"),
        LMStudioBaseURL: getEnvOrDefault("LMSTUDIO_BASE_URL", "http://localhost:1234/v1"),
    }
}
```

### LM Studio / OpenRouter — Zero Extra Code
```go
// Source: .planning/research/RESEARCH-LLM-GATEWAY.md (Section 1)
// Both reuse OpenAIProvider with custom BaseURL:

// OpenRouter
openRouterClient := openai.NewClient(
    option.WithBaseURL("https://openrouter.ai/api/v1"),
    option.WithAPIKey(cfg.OpenRouterKey),
)

// LM Studio
lmStudioClient := openai.NewClient(
    option.WithBaseURL(cfg.LMStudioBaseURL),
    option.WithAPIKey("lm-studio"),  // non-empty key required
)
```

### React Streaming ChatBubble (Blinking Cursor)
```tsx
// frontend/src/components/chat/StreamingBubble.tsx
import ShikiHighlighter from "react-shiki"

const CURSOR = "▋"

export function StreamingBubble({ content, isStreaming }: Props) {
  const display = isStreaming ? content + CURSOR : content
  // Detect code blocks within display and render with ShikiHighlighter
  // Plain text segments rendered as markdown
  // delay={50} matches the 50ms Wails batch window
  return <div>{renderMarkdownWithCode(display, { delay: 50 })}</div>
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Community `sashabaranov/go-openai` | Official `github.com/openai/openai-go/v3` | 2024-2025 | Official SDK is now strategic; v3.30.0 is current |
| Per-token EventsEmit | 50ms batched EventsEmit + sequence numbers | Wails v2 #2759 (confirmed) | Required to prevent out-of-order delivery |
| react-syntax-highlighter | react-shiki | 2024+ | Shiki provides TextMate grammars; `delay` prop for streaming |

**Deprecated/outdated:**
- `sashabaranov/go-openai`: Superseded by official OpenAI Go SDK — do not use
- Anthropic system prompt as `{role: "system"}` message: Invalid in Anthropic API — always top-level `System` field

---

## Open Questions

1. **gitleaks `detect.DetectString` API signature**
   - What we know: `detect.Detector` struct exists; `detect.NewDetector(cfg)` is documented
   - What's unclear: Exact method signature for scanning an arbitrary string (not a git diff); may need `DetectBytes` or wrapping as a `Fragment`
   - Recommendation: At Wave 0 implementation, inspect `pkg.go.dev/github.com/zricethezav/gitleaks/v8/detect` for the string/bytes scanning surface; have a regex fallback for Phase 2 if the API is unclear

2. **Terminal ref exposure from TerminalPreview to ChatPane**
   - What we know: `termRef.current` is a local ref inside `TerminalPreview.tsx`; it currently has no upward exposure
   - What's unclear: Best pattern — add `termRefs: Map` to `terminalStore`, use a React context, or expose via a custom hook
   - Recommendation: Add `termRefs: Map<string, Terminal>` to `terminalStore` with imperative setters (not state); avoids React re-renders and keeps xterm.js outside the React rendering cycle

3. **tokenCount source for StatusBar**
   - What we know: OpenAI provides `acc.Usage.TotalTokens`; Anthropic provides `message.Usage`; Ollama provides `resp.EvalCount + resp.PromptEvalCount`; pre-flight estimation uses ~4 chars/token
   - What's unclear: Whether to show estimated count pre-send or only actual count post-receive
   - Recommendation: Show estimated tokens pre-send (immediate feedback); update with actual count after stream completes via a `llm:usage` event

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Go 1.23 | All Go code | ✓ | 1.23.0 | — |
| Node.js | Frontend build | ✓ | 24.12.0 | — |
| Wails v2 | Desktop binding | ✓ | v2.12.0 (go.mod) | — |
| OpenAI API key | LLM-01 testing | Unknown | — | Use Ollama for CI testing |
| Anthropic API key | LLM-02 testing | Unknown | — | Use Ollama for CI testing |
| Ollama binary | LLM-03 testing | Unknown | — | Skip Ollama integration tests in CI |
| internet access (npm/proxy.golang.org) | Package install | ✓ | — | — |

**Missing dependencies with no fallback:**
- None that block implementation; all SDKs install from public registries.

**Missing dependencies with fallback:**
- LLM API keys: Ollama (local, no key needed) can be used for all streaming integration testing. Integration tests for OpenAI/Anthropic should be skipped in CI unless keys are available via env vars.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Go tests | `go test ./...` (stdlib testing package, no framework needed) |
| Frontend tests | vitest 4.1.2 with jsdom (configured in `vite.config.ts`) |
| Go quick run | `go test ./services/...` |
| Go full suite | `go test ./...` |
| Frontend quick | `cd frontend && npx vitest run --reporter=verbose` |
| Frontend full | `cd frontend && npx vitest run` |

### Testability Classification

#### Unit Testable (automated, fast, no external services)

| Req ID | Behavior | Test Type | File |
|--------|----------|-----------|------|
| LLM-05 | Provider interface: mock provider returning known chunks | Go unit | `services/llm/provider_test.go` |
| LLM-05 | StreamChunk channel drains correctly; goroutine does not leak | Go unit | `services/llm/openai_test.go` (mock SDK) |
| LLM-03 | validateOllamaHost rejects non-localhost | Go unit | `services/llm/ollama_test.go` |
| LLM-07 | validateOllamaHost accepts 127.0.0.1, localhost, ::1 | Go unit | `services/llm/ollama_test.go` |
| FILT-01 | ANSIFilter.Cleanse strips escape sequences from known inputs | Go unit | `services/llm/filter/ansi_test.go` |
| FILT-01 | ANSI OSC title injection stripped correctly | Go unit | `services/llm/filter/ansi_test.go` |
| FILT-02 | CredentialFilter redacts AWS_ACCESS_KEY, GitHub token, bearer token patterns | Go unit | `services/llm/filter/credential_test.go` |
| FILT-03 | Filter pipeline applies ANSI before credential scan (order enforced by type) | Go unit | `services/llm/filter/filter_test.go` |
| FILT-06 | TruncateTerminal keeps tail; prepends ellipsis marker | Go unit | `services/llm/context_test.go` |
| FILT-07 | EstimateTokens returns sane values for known strings | Go unit | `services/llm/context_test.go` |
| CHAT-02 | buildMessages assembles correct message list with terminal context prefix | Go unit | `services/llm/context_test.go` |
| CHAT-02 | buildMessages omits terminal block when terminal content is empty | Go unit | `services/llm/context_test.go` |
| CHAT-03 | chatStore startStreamingMessage creates message with isStreaming=true | TS unit | `frontend/src/stores/__tests__/chatStore.test.ts` |
| CHAT-03 | chatStore appendChunk accumulates text correctly | TS unit | `frontend/src/stores/__tests__/chatStore.test.ts` |
| CHAT-03 | chatStore finalizeMessage sets isStreaming=false, strips cursor | TS unit | `frontend/src/stores/__tests__/chatStore.test.ts` |

#### Integration Tests (require live LLM service — skip in CI by default)

| Req ID | Behavior | How to Run | Skip Condition |
|--------|----------|------------|----------------|
| LLM-01 | OpenAI adapter receives real streaming chunks | `OPENAI_API_KEY=... go test -tags=integration ./services/llm/...` | No `OPENAI_API_KEY` |
| LLM-02 | Anthropic adapter receives chunks; system prompt in correct field | `ANTHROPIC_API_KEY=... go test -tags=integration ./services/llm/...` | No `ANTHROPIC_API_KEY` |
| LLM-03 | Ollama adapter streams from local model | `go test -tags=integration ./services/llm/...` (needs running Ollama) | Ollama not running |

Guard with build tag: `//go:build integration` in test files.

#### Human Verification Required

| Req ID | Behavior | Why Not Automated |
|--------|----------|-------------------|
| CHAT-03 | Blinking cursor `▋` visible while streaming | Visual; requires human observation |
| CHAT-03 | Streaming text appears smoothly without jumps | Perceptual quality; no automated assertion |
| CHAT-04 | "Copy to Terminal" button appears only after code block closes | Timing + visual; hard to assert programmatically |
| CHAT-04 | react-shiki `delay={50}` produces smooth highlighting during stream | Perceptual; no automated assertion |
| LLM-06 | 50ms batching produces smooth UX (no visible stuttering) | Perceptual |
| FILT-01 | ANSI-stripped content looks correct in status bar | Visual |
| Auto-scroll | Auto-scroll does not hijack when user has scrolled up | User interaction simulation |

### Suggested Test File Structure

```
services/
├── llm/
│   ├── provider_test.go          # MockProvider, channel drain, goroutine leak check
│   ├── openai_test.go            # buildOpenAIParams, message conversion; mock SDK stream
│   ├── anthropic_test.go         # buildParams: system field extraction from messages array
│   ├── ollama_test.go            # validateOllamaHost; callback-to-channel wrap test
│   ├── context_test.go           # buildMessages, TruncateTerminal, EstimateTokens
│   └── filter/
│       ├── ansi_test.go          # StripANSI: CSI, OSC, private sequences, no-op on clean text
│       ├── credential_test.go    # RedactCredentials: AWS, GitHub, bearer, no-op on clean text
│       └── filter_test.go        # Pipeline: ANSI first, then credential; chained correctly

frontend/src/
├── stores/__tests__/
│   └── chatStore.test.ts         # Add: startStreamingMessage, appendChunk, finalizeMessage, setStreamError
├── utils/__tests__/
│   └── terminalContext.test.ts   # readTerminalLines: mock xterm.js Buffer; 200-line limit
└── hooks/__tests__/
    └── useLLMStream.test.ts      # Mock Wails runtime; verify store actions called on events
```

### Wave 0 Gaps (tests that must exist before implementation begins)

- [ ] `services/llm/filter/ansi_test.go` — covers FILT-01; does not exist yet
- [ ] `services/llm/filter/credential_test.go` — covers FILT-02; does not exist yet
- [ ] `services/llm/context_test.go` — covers FILT-06, CHAT-02 message assembly; does not exist yet
- [ ] `services/llm/ollama_test.go` — covers LLM-07 localhost validation; does not exist yet
- [ ] `frontend/src/stores/__tests__/chatStore.test.ts` — EXISTS but needs new actions added: `startStreamingMessage`, `appendChunk`, `finalizeMessage`, `setStreamError`
- [ ] `frontend/src/utils/__tests__/terminalContext.test.ts` — new; covers xterm.js buffer reading

### Sampling Rate
- **Per task commit:** `go test ./services/... && cd frontend && npx vitest run`
- **Per wave merge:** `go test ./... && cd frontend && npx vitest run`
- **Phase gate:** Full suite green before `/gsd:verify-work`

---

## Sources

### Primary (HIGH confidence)
- `.planning/research/RESEARCH-LLM-GATEWAY.md` — Dedicated prior research: OpenAI, Anthropic, Ollama SDK patterns, Wails streaming, token counting; all verified against official sources
- `proxy.golang.org` — Version verification for all Go packages (2026-03-26)
- `npm registry` — Version verification for react-shiki (0.9.2) and react-markdown (10.1.0) (2026-03-26)
- `xtermjs.org/docs/api/terminal/interfaces/ibuffer/` and `ibufferline/` — xterm.js 6.x buffer reading API
- `github.com/AVGVSTVS96/react-shiki` — `delay` prop confirmed present in 0.9.2
- `github.com/leaanthony/go-ansi-parser` — `Cleanse` function confirmed; already in go.sum as indirect Wails dep

### Secondary (MEDIUM confidence)
- Wails Issue #2759 — EventsEmit out-of-order at high rates (confirmed, referenced in SUMMARY.md)
- `.planning/research/SUMMARY.md` — Cross-cutting synthesis; architecture overview; pitfall catalog
- `pkg.go.dev/github.com/zricethezav/gitleaks/v8/detect` — Library API surface (MEDIUM: primarily CLI tool; library API less documented)

### Tertiary (LOW confidence, flagged)
- gitleaks `DetectString` exact method signature — flagged for verification at implementation time

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified via proxy.golang.org and npm registry on 2026-03-26
- Architecture patterns: HIGH — code patterns sourced from dedicated prior research document verified against official SDK READMEs
- ANSI stripping library choice: HIGH — `leaanthony/go-ansi-parser.Cleanse` confirmed present in go.sum; verified via GitHub README
- gitleaks integration: MEDIUM — library vs. binary API surface less documented; recommend verification at implementation
- xterm.js buffer reading: HIGH — API confirmed via xtermjs.org docs
- Pitfalls: HIGH — sourced from confirmed issue tracker reports and security research

**Research date:** 2026-03-26
**Valid until:** 2026-04-25 (stable libraries; 30-day window standard)
