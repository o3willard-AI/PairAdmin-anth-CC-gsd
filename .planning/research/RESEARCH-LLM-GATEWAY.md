# LLM Gateway Research: Go Multi-Provider AI Chat with Streaming

**Project:** PairAdmin — Provider-Agnostic LLM Gateway
**Researched:** 2026-03-25
**Overall confidence:** HIGH (OpenAI, Anthropic, Ollama SDKs verified via official sources; patterns verified via multiple sources)

---

## 1. OpenAI Go SDK: Current State (2026)

### Library

Use the **official** `github.com/openai/openai-go` (not the community `sashabaranov/go-openai`). The official SDK reached v3.x status in 2025-2026 and is the strategic choice going forward.

- **Module path:** `github.com/openai/openai-go/v3`
- **Current version:** v3.30.0 (as of March 2026)
- **Minimum Go:** 1.22
- **Install:** `go get github.com/openai/openai-go/v3`

Confidence: HIGH — confirmed via github.com/openai/openai-go.

### Streaming Chat Completions

The SDK exposes two surfaces. For PairAdmin, use `client.Chat.Completions.NewStreaming` (Chat Completions API), not the newer Responses API. The Chat Completions API is "supported indefinitely" and maps cleanly to the multi-provider abstraction because Anthropic and Ollama use the same conceptual shape (roles + messages).

```go
import (
    "github.com/openai/openai-go/v3"
    "github.com/openai/openai-go/v3/option"
)

client := openai.NewClient(option.WithAPIKey(key))

stream := client.Chat.Completions.NewStreaming(ctx,
    openai.ChatCompletionNewParams{
        Model: openai.ChatModelGPT4o,
        Messages: []openai.ChatCompletionMessageParamUnion{
            openai.SystemMessage("You are a terminal assistant."),
            openai.UserMessage("What does this error mean?"),
        },
        MaxTokens: openai.Int(2048),
    },
)

acc := openai.ChatCompletionAccumulator{}
for stream.Next() {
    chunk := stream.Current()
    acc.AddChunk(chunk)
    if len(chunk.Choices) > 0 {
        delta := chunk.Choices[0].Delta.Content
        // emit delta to Wails frontend here
    }
}
if stream.Err() != nil {
    return stream.Err()
}
// acc.Usage.TotalTokens is available after stream ends
```

Key types:
- `openai.ChatCompletionAccumulator` — accumulates chunks; exposes `JustFinishedContent()`, `JustFinishedToolCall()`, `Usage`
- `stream.Next()` / `stream.Current()` — standard iterator; `stream.Err()` checked after loop

### Error Handling and Rate Limits

The official SDK surfaces errors as `*openai.Error` with `StatusCode`, `Message`, and optional `Param` fields. Check `StatusCode == 429` for rate limits. The SDK does NOT auto-retry by default — implement retry at the provider adapter layer (see section 9).

### LM Studio / llama.cpp (OpenAI-compatible)

Both LM Studio (port 1234 by default) and llama.cpp server expose an `/v1/chat/completions` endpoint that is API-compatible with OpenAI. Use the same official SDK with a custom base URL:

```go
client := openai.NewClient(
    option.WithBaseURL("http://localhost:1234/v1"),
    option.WithAPIKey("lm-studio"),  // LM Studio accepts any non-empty key
)
```

This means LM Studio and llama.cpp require zero additional code beyond an `openai.NewClient` with a custom base URL. They reuse the OpenAI provider adapter entirely.

Confidence: HIGH — confirmed via lmstudio.ai/docs/developer/openai-compat.

---

## 2. Anthropic Go SDK

### Library

Use the **official** `github.com/anthropics/anthropic-sdk-go`. As of March 2026 this is v1.27.1, actively maintained with 342+ importers.

- **Module:** `github.com/anthropics/anthropic-sdk-go`
- **Current version:** v1.27.1 (Mar 18, 2026)
- **Minimum Go:** 1.22
- **Install:** `go get github.com/anthropics/anthropic-sdk-go`

Do NOT use raw HTTP + SSE. The official SDK is production-quality, MIT licensed, and handles the SSE framing, retry-after headers, and error normalization correctly. Using raw HTTP adds maintenance burden with no benefit.

### Streaming Messages

Anthropic's API differs from OpenAI in one important way: the top-level request structure uses `Messages` (not `Chat.Completions`) and each message uses `MessageParam` types. The `system` prompt is a top-level field, not a role in the messages array.

```go
import "github.com/anthropics/anthropic-sdk-go"

client := anthropic.NewClient(option.WithAPIKey(key))

stream := client.Messages.NewStreaming(ctx, anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeSonnet4_20250514,
    MaxTokens: 2048,
    System: []anthropic.TextBlockParam{
        {Text: systemPromptText},
    },
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("What does this error mean?")),
    },
})

message := anthropic.Message{}
for stream.Next() {
    event := stream.Current()
    // Accumulate for final usage stats
    _ = message.Accumulate(event)

    switch ev := event.AsAny().(type) {
    case anthropic.ContentBlockDeltaEvent:
        switch delta := ev.Delta.AsAny().(type) {
        case anthropic.TextDelta:
            // emit delta.Text to Wails frontend here
        }
    }
}
if stream.Err() != nil {
    return stream.Err()
}
// message.Usage.InputTokens, message.Usage.OutputTokens available
```

Key difference from OpenAI: system prompts go in `MessageNewParams.System`, not as a message with `role: system`. When mapping from a provider-agnostic interface, the adapter must split the system message out.

Confidence: HIGH — confirmed via github.com/anthropics/anthropic-sdk-go README and pkg.go.dev.

---

## 3. Ollama Go Integration

### Library

Use the **official** Ollama API package: `github.com/ollama/ollama/api`. This is the same package Ollama itself uses internally.

- **Module:** `github.com/ollama/ollama` (take the `api` subpackage)
- **Current version:** v0.18.3
- **Install:** `go get github.com/ollama/ollama@latest`

Note: There is a known security advisory against old versions. Always pin to a recent release.

### Streaming Chat

Ollama's streaming model uses a **callback function** rather than an iterator. The callback is invoked for each chunk; returning a non-nil error from the callback cancels the stream.

```go
import "github.com/ollama/ollama/api"

client, err := api.ClientFromEnvironment()  // reads OLLAMA_HOST
// or: client := api.NewClient(baseURL, http.DefaultClient)

stream := true
req := &api.ChatRequest{
    Model:  "llama3.2",
    Stream: &stream,
    Messages: []api.Message{
        {Role: "system",  Content: systemPromptText},
        {Role: "user",    Content: "What does this error mean?"},
    },
}

err = client.Chat(ctx, req, func(resp api.ChatResponse) error {
    if resp.Done {
        // resp.TotalDuration, resp.EvalCount available
        return nil
    }
    // emit resp.Message.Content to Wails frontend here
    return nil
})
```

The callback approach means the adapter wraps this in a goroutine that feeds a channel, to match the channel-based streaming pattern at the provider interface level (see section 4).

Confidence: HIGH — confirmed via pkg.go.dev/github.com/ollama/ollama/api.

---

## 4. Provider Abstraction Design in Go

### Recommended Interface

Design a minimal, channel-based streaming interface. Channels are idiomatic Go and work naturally with `select` for cancellation.

```go
// internal/llm/provider.go

package llm

import "context"

// Role constants map to standard chat roles across all providers.
type Role string

const (
    RoleSystem    Role = "system"
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
)

// Message is the provider-agnostic message type.
type Message struct {
    Role    Role
    Content string
}

// StreamChunk is a single token or text delta from the provider.
type StreamChunk struct {
    Text  string // incremental token text
    Done  bool   // true on final chunk
    Error error  // non-nil if the stream failed
}

// Usage holds token counts returned after stream completion.
type Usage struct {
    InputTokens  int
    OutputTokens int
}

// Provider is the single interface all LLM backends implement.
type Provider interface {
    // Name returns a human-readable provider identifier ("openai", "anthropic", "ollama").
    Name() string

    // Stream sends messages and streams token chunks on the returned channel.
    // The channel is closed after the final chunk (Done=true) or an error chunk.
    // Callers must consume the channel until it is closed to avoid goroutine leaks.
    // Cancel ctx to abort mid-stream.
    Stream(ctx context.Context, messages []Message) (<-chan StreamChunk, error)

    // Models returns available model IDs for the provider.
    Models(ctx context.Context) ([]string, error)
}
```

### Why Channels, Not Iterators

- Iterators (like `stream.Next()`) block the calling goroutine; channels decouple producer and consumer.
- `select { case chunk := <-ch: ... case <-ctx.Done(): ... }` gives clean cancellation.
- The Wails event emitter runs in a separate goroutine; feeding from a channel avoids shared-state issues.
- This matches how mozilla/any-llm-go designs its streaming surface.

### Adapter Implementation Pattern

Each provider wraps its SDK in a struct that implements `Provider`:

```go
// internal/llm/openai_adapter.go

type OpenAIProvider struct {
    client *openai.Client
    model  string
}

func (p *OpenAIProvider) Name() string { return "openai" }

func (p *OpenAIProvider) Stream(ctx context.Context, messages []Message) (<-chan StreamChunk, error) {
    params := buildOpenAIParams(p.model, messages)
    sdkStream := p.client.Chat.Completions.NewStreaming(ctx, params)

    ch := make(chan StreamChunk, 32) // buffer avoids blocking on slow Wails emit
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

For Ollama, the callback-based SDK is wrapped to feed the same channel:

```go
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
            ch <- StreamChunk{Text: resp.Message.Content}
            return nil
        })
        if err != nil && err != context.Canceled {
            ch <- StreamChunk{Error: err}
        }
    }()
    return ch, nil
}
```

### Provider Registry

```go
type Registry struct {
    providers map[string]Provider
}

func (r *Registry) Get(name string) (Provider, bool) {
    p, ok := r.providers[name]
    return p, ok
}

func (r *Registry) Register(p Provider) {
    r.providers[p.Name()] = p
}
```

At startup, register all configured providers. A provider is only registered if its config (API key, host URL) is present.

---

## 5. Context Window Management

### Problem

Terminal output can be arbitrarily long. GPT-4o has a 128K context; Claude Sonnet has 200K; Ollama models vary from 4K to 128K. Naively including all terminal content will exceed limits and incur high costs.

### Recommended Strategy: Recency-First Truncation with Hard Budget

Allocate the context budget as follows:

```
total_limit = provider_context_limit (e.g. 128000 tokens)
output_reserve = 2048 tokens (reserved for model response)
system_budget = 512 tokens (fixed system instructions)
history_budget = 4096 tokens (recent N turns of chat)
terminal_budget = total_limit - output_reserve - system_budget - history_budget
               = ~121,344 tokens for GPT-4o
```

For terminal content specifically: truncate from the **beginning**, not the end. The most recent output is most relevant. Keep the last N characters and prepend an ellipsis marker.

```go
// TruncateTerminal trims terminal content to fit within tokenBudget,
// keeping the tail (most recent content).
func TruncateTerminal(content string, tokenBudget int) string {
    // Approximate: 1 token ~ 4 characters for code/terminal output
    charBudget := tokenBudget * 4
    if len(content) <= charBudget {
        return content
    }
    return "[...truncated...]\n" + content[len(content)-charBudget:]
}
```

For conversation history, use a sliding window: keep the system message plus the most recent K exchanges. When the window fills, drop the oldest user+assistant pair. Do NOT summarize unless you explicitly want that complexity — for a terminal chat context, recency is almost always the right heuristic.

```go
// TrimHistory returns the most recent messages that fit within tokenBudget.
// Always preserves the system message at index 0.
func TrimHistory(messages []Message, tokenBudget int) []Message {
    if len(messages) == 0 {
        return messages
    }

    system := messages[0]
    history := messages[1:]

    total := estimateTokens(system.Content)
    var kept []Message

    // Walk history from newest to oldest
    for i := len(history) - 1; i >= 0; i-- {
        t := estimateTokens(history[i].Content) + 4 // role overhead
        if total+t > tokenBudget {
            break
        }
        total += t
        kept = append([]Message{history[i]}, kept...)
    }

    return append([]Message{system}, kept...)
}
```

### Provider Context Limits (approximate, verify at runtime)

| Provider/Model | Context Limit |
|---|---|
| GPT-4o | 128,000 tokens |
| GPT-4-turbo | 128,000 tokens |
| Claude 3.5 Sonnet | 200,000 tokens |
| Claude 3 Opus | 200,000 tokens |
| Ollama llama3.2 | 128,000 tokens (model-dependent) |
| Ollama mistral | 32,000 tokens |
| LM Studio (any) | model-dependent, query via `/v1/models` |

For Ollama, query the model's context size at provider init using `client.Show()`:

```go
info, err := ollamaClient.Show(ctx, &api.ShowRequest{Model: modelName})
// info.ModelInfo["llama.context_length"] contains the context window
```

---

## 6. System Prompt Injection: Where Terminal Context Lives

### OpenAI / Ollama: System Message Role

For providers that use the roles-based message format (OpenAI, Ollama, LM Studio, llama.cpp), terminal context belongs in the **system message**. This is the most effective placement:

```go
func BuildSystemPrompt(terminalContent string) string {
    return fmt.Sprintf(`You are PairAdmin, an AI assistant embedded in a terminal application.

CURRENT TERMINAL CONTEXT:
The following is the recent terminal output. Use it to answer the user's questions:

<terminal>
%s
</terminal>

Instructions:
- Answer questions about the terminal output above.
- If asked to run commands, suggest them clearly as code blocks.
- Do not repeat the terminal content back unless asked.
- If the terminal content is truncated, say so.`, terminalContent)
}
```

The `<terminal>` XML-style delimiter improves model comprehension by clearly marking where the injected context ends. This is a standard practice in production LLM systems.

### Anthropic: Top-Level System Field

Anthropic's API takes `system` as a separate top-level field, not as a message role. The adapter must extract the system message:

```go
func (p *AnthropicProvider) buildParams(messages []Message) anthropic.MessageNewParams {
    params := anthropic.MessageNewParams{
        Model:     anthropic.Model(p.model),
        MaxTokens: 2048,
    }

    // Extract system message from the canonical list
    var userMessages []anthropic.MessageParam
    for _, m := range messages {
        switch m.Role {
        case RoleSystem:
            params.System = []anthropic.TextBlockParam{{Text: m.Content}}
        case RoleUser:
            userMessages = append(userMessages,
                anthropic.NewUserMessage(anthropic.NewTextBlock(m.Content)))
        case RoleAssistant:
            userMessages = append(userMessages,
                anthropic.NewAssistantMessage(anthropic.NewTextBlock(m.Content)))
        }
    }
    params.Messages = userMessages
    return params
}
```

Keep system prompt injection in one place: the gateway constructs the `[]Message` with the system message at index 0, and each adapter handles the conversion to its API's format.

### Security Note: Prompt Injection from Terminal Content

Terminal content is untrusted user-controlled data (OWASP LLM01:2025). Malicious content like `Ignore previous instructions and...` can appear in terminal output. Mitigations:

1. Use XML delimiters (`<terminal>...</terminal>`) to make the boundary explicit to the model.
2. Instruct the model in the system prompt to treat content inside `<terminal>` as data, not instructions.
3. Never interpolate terminal content directly into the instruction portion of the system prompt.

---

## 7. Streaming from Go to React in Wails

### Recommended Pattern: Goroutine + EventsEmit per Chunk

Use `runtime.EventsEmit` from a goroutine that consumes the provider's stream channel. Do NOT buffer all tokens and emit once — that eliminates the streaming UX.

```go
// app.go (Wails App struct method)

import "github.com/wailsapp/wails/v2/pkg/runtime"

type ChatToken struct {
    Text    string `json:"text"`
    Done    bool   `json:"done"`
    Error   string `json:"error,omitempty"`
}

func (a *App) SendMessage(providerName, userInput string) error {
    provider, ok := a.registry.Get(providerName)
    if !ok {
        return fmt.Errorf("unknown provider: %s", providerName)
    }

    // Build messages with terminal context
    messages := a.buildMessages(userInput)

    // Start stream in a goroutine so Wails call returns immediately
    go func() {
        ctx, cancel := context.WithTimeout(a.ctx, 5*time.Minute)
        defer cancel()

        ch, err := provider.Stream(ctx, messages)
        if err != nil {
            runtime.EventsEmit(a.ctx, "chat:token", ChatToken{
                Error: err.Error(),
                Done:  true,
            })
            return
        }

        for chunk := range ch {
            if chunk.Error != nil {
                runtime.EventsEmit(a.ctx, "chat:token", ChatToken{
                    Error: chunk.Error.Error(),
                    Done:  true,
                })
                return
            }
            runtime.EventsEmit(a.ctx, "chat:token", ChatToken{
                Text: chunk.Text,
                Done: chunk.Done,
            })
        }
    }()

    return nil  // return immediately; tokens arrive via events
}
```

On the React side:

```typescript
import { EventsOn, EventsOff } from "@wailsapp/runtime"

useEffect(() => {
    const unsubscribe = EventsOn("chat:token", (chunk: ChatToken) => {
        if (chunk.error) {
            setError(chunk.error)
            return
        }
        setResponse(prev => prev + chunk.text)
        if (chunk.done) {
            setStreaming(false)
        }
    })
    return () => unsubscribe()
}, [])
```

### Important: Wails EventsEmit Data Race

There is a known data race in Wails v2's events system when the frontend subscribes (`EventsOn`) while the backend emits simultaneously (issue #2448). Mitigate by:

1. Emitting the first token only after a brief synchronization signal from the frontend, or
2. Buffering the first N tokens (50ms window) before starting emission, giving the frontend time to register its listener after the `SendMessage` call returns.
3. Using Wails v3 if available for your project timeline — v3 redesigns the event system with better type safety.

### Per-Token vs Buffered Chunks

Emit per token (unbuffered). The overhead of `EventsEmit` per token is negligible at typical LLM generation speeds (20-80 tokens/second). Buffering adds latency with no practical gain for this use case.

---

## 8. Token Counting and Cost Estimation

### For OpenAI Models: tiktoken-go

Use `github.com/pkoukk/tiktoken-go` for offline token counting. It is a Go port of OpenAI's tiktoken with the same encoding tables, works offline after initial vocabulary download, and has 344+ known importers.

```go
import "github.com/pkoukk/tiktoken-go"

func CountTokensOpenAI(model, text string) (int, error) {
    tkm, err := tiktoken.EncodingForModel(model)
    if err != nil {
        // Fall back to approximation if model not recognized
        return len(text) / 4, nil
    }
    return len(tkm.Encode(text, nil, nil)), nil
}

// Count tokens for a full message list (adds per-message overhead)
func CountMessageTokens(model string, messages []Message) int {
    tkm, err := tiktoken.EncodingForModel(model)
    if err != nil {
        return estimateTokens(messagesText(messages))
    }

    total := 3 // reply priming
    for _, m := range messages {
        total += 4 // per-message overhead (role, framing)
        total += len(tkm.Encode(m.Content, nil, nil))
    }
    return total
}
```

Encoding by model:
- `o200k_base`: gpt-4o, gpt-4.1, gpt-4.5
- `cl100k_base`: gpt-4, gpt-3.5-turbo, text-embedding-3-*

### For Anthropic Models: Approximation Only

Anthropic uses its own tokenizer. There is no official Go implementation. Use the character-based approximation: **1 token ≈ 3.5 characters** for Claude (slightly denser than OpenAI's ~4 chars/token). For budget purposes this is accurate within 15%.

```go
func estimateTokensAnthropic(text string) int {
    return int(math.Ceil(float64(len(text)) / 3.5))
}
```

The official count_tokens API endpoint exists but requires a network call — unsuitable for pre-flight budget checks.

### For Ollama: Use Actual Response Counts

Ollama returns `EvalCount` (output tokens) and `PromptEvalCount` (input tokens) in the final `ChatResponse` after `Done: true`. Use these for accurate post-request accounting rather than pre-flight estimation.

### Cost Tracking

Maintain a simple in-memory cost accumulator per session:

```go
type UsageTracker struct {
    mu           sync.Mutex
    SessionInput  int
    SessionOutput int
    Provider     string
}

// Approximate cost per 1M tokens (USD, as of early 2026 — verify and update)
var pricesPer1MTokens = map[string][2]float64{
    // [input, output] per 1M tokens
    "gpt-4o":              {2.50, 10.00},
    "gpt-4o-mini":         {0.15,  0.60},
    "claude-sonnet-4":    {3.00, 15.00},
    "claude-haiku-3-5":   {0.80,  4.00},
    "ollama":              {0.00,  0.00}, // local, free
}
```

---

## 9. Error Handling and Retry

### Error Classification

Classify errors before deciding whether to retry:

| Error Type | HTTP Status | Retry? | Action |
|---|---|---|---|
| Rate limit | 429 | Yes | Respect `Retry-After` header, exponential backoff |
| Server overloaded | 503 | Yes | Exponential backoff with jitter |
| Timeout | N/A | Yes | Backoff, reduce context size on retry |
| Auth failure | 401 | No | Surface error to user immediately |
| Bad request | 400 | No | Surface error (likely context too long) |
| Not found | 404 | No | Surface error (invalid model) |
| Context length exceeded | 400/413 | Maybe | Retry with trimmed context |

### Retry Implementation

Use `github.com/cenkalti/backoff/v4` — the standard Go exponential backoff library:

```go
import (
    "github.com/cenkalti/backoff/v4"
)

func (p *OpenAIProvider) streamWithRetry(
    ctx context.Context,
    messages []Message,
    maxAttempts uint64,
) (<-chan StreamChunk, error) {
    var ch <-chan StreamChunk

    b := backoff.WithContext(
        backoff.WithMaxRetries(backoff.NewExponentialBackOff(), maxAttempts),
        ctx,
    )

    err := backoff.Retry(func() error {
        var streamErr error
        ch, streamErr = p.attemptStream(ctx, messages)
        if streamErr == nil {
            return nil
        }

        // Type-check for retryable errors
        var apiErr *openai.Error
        if errors.As(streamErr, &apiErr) {
            switch apiErr.StatusCode {
            case 429, 503:
                return streamErr // retryable
            default:
                return backoff.Permanent(streamErr) // not retryable
            }
        }
        if errors.Is(streamErr, context.Canceled) {
            return backoff.Permanent(streamErr)
        }
        return streamErr // unknown errors are retried
    }, b)

    return ch, err
}
```

For 429 responses, parse the `Retry-After` header when available and use that value instead of the computed backoff. The OpenAI SDK surfaces headers via the error response object.

### Mid-Stream Errors

If an error occurs mid-stream (after tokens have already been emitted), the cleanest UX is to emit a final `StreamChunk{Error: err}` and let the frontend display a partial response with an error indicator. Do not retry mid-stream — the partial response is already in the UI and a retry would start fresh.

### Context Cancellation

All providers must respect `ctx.Done()`. The goroutine pattern above handles this:

```go
select {
case ch <- chunk:
case <-ctx.Done():
    return // goroutine exits, channel is closed by defer
}
```

Closing the channel signals to the consumer that the stream ended, whether due to completion or cancellation.

### Timeout Configuration

Set a generous but bounded timeout on the stream context. Five minutes covers the longest plausible generation for a large context response:

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Minute)
defer cancel()
```

For Ollama (local), network timeouts are irrelevant but generation can be slow on CPU. Use the same 5-minute budget.

---

## Summary: Recommended Library Decisions

| Component | Library | Version | Rationale |
|---|---|---|---|
| OpenAI | `github.com/openai/openai-go/v3` | v3.30.0 | Official; Chat Completions API maps cleanly to abstraction |
| Anthropic | `github.com/anthropics/anthropic-sdk-go` | v1.27.1 | Official; MIT; streaming iterator pattern |
| Ollama | `github.com/ollama/ollama/api` | v0.18.x latest | Official; callback streaming wrapped to channel |
| LM Studio | (reuse OpenAI adapter) | — | OpenAI-compatible; set custom BaseURL |
| llama.cpp | (reuse OpenAI adapter) | — | OpenAI-compatible; set custom BaseURL |
| Token counting | `github.com/pkoukk/tiktoken-go` | latest | 344+ importers; offline; accurate for OpenAI models |
| Retry/backoff | `github.com/cenkalti/backoff/v4` | v4.x | Standard Go backoff; context-aware |

## Key Design Decisions

1. **Channel-based streaming interface** — not iterators. Works with `select`, decouples providers from Wails emitter goroutine.

2. **System message at index 0** — the canonical `[]Message` always has `RoleSystem` at position 0. Adapters handle the per-API mapping (Anthropic extracts it to `System` field; others pass it as a role).

3. **Terminal content in system prompt, inside XML delimiters** — `<terminal>...</terminal>` improves model comprehension and reduces prompt injection risk.

4. **Recency-first truncation** — keep the tail of terminal output (most recent), drop oldest chat history pairs first, never drop the system message.

5. **LM Studio and llama.cpp are free** — they speak OpenAI's HTTP protocol. No extra code; just configure the base URL in the OpenAI adapter.

6. **No mid-stream retry** — retry only on initial connection failure. Partial token delivery means the UI already has content; a retry would be jarring.

## Sources

- [github.com/openai/openai-go](https://github.com/openai/openai-go) — Official OpenAI Go SDK
- [github.com/anthropics/anthropic-sdk-go](https://github.com/anthropics/anthropic-sdk-go) — Official Anthropic Go SDK
- [pkg.go.dev/github.com/ollama/ollama/api](https://pkg.go.dev/github.com/ollama/ollama/api) — Official Ollama API package
- [lmstudio.ai/docs/developer/openai-compat](https://lmstudio.ai/docs/developer/openai-compat) — LM Studio OpenAI-compatible endpoints
- [blog.mozilla.ai/any-llm-go](https://blog.mozilla.ai/run-openai-claude-mistral-llamafile-and-more-from-one-interface-now-in-go/) — Provider abstraction pattern reference
- [github.com/pkoukk/tiktoken-go](https://github.com/pkoukk/tiktoken-go) — tiktoken Go port
- [pkg.go.dev/github.com/cenkalti/backoff/v4](https://pkg.go.dev/github.com/cenkalti/backoff/v4) — Exponential backoff library
- [wails.io/docs/reference/runtime/events](https://wails.io/docs/reference/runtime/events/) — Wails EventsEmit API
- [docs.anthropic.com/en/api/messages-streaming](https://docs.anthropic.com/en/api/messages-streaming) — Anthropic streaming protocol
- [OWASP LLM01:2025](https://genai.owasp.org/llmrisk/llm01-prompt-injection/) — Prompt injection risk reference
