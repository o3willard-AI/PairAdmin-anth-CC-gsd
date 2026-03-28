---
phase: 02-llm-gateway-streaming-chat
plan: "01"
subsystem: llm-gateway
tags: [go, llm, streaming, wails, tdd]
dependency_graph:
  requires: []
  provides:
    - services/llm.Provider (interface)
    - services/llm.OpenAIProvider (openai/openrouter/lmstudio)
    - services/llm.AnthropicProvider
    - services/llm.OllamaProvider
    - services/llm.Registry
    - services/llm.BuildMessages
    - services.LLMService
    - services.Config
    - services.LoadConfig
  affects:
    - main.go (will bind LLMService in plan 02-04)
    - frontend ChatPane (plan 02-02 will wire useLLMStream hook)
tech_stack:
  added:
    - github.com/openai/openai-go/v3 v3.30.0
    - github.com/anthropics/anthropic-sdk-go v1.27.1
    - github.com/ollama/ollama v0.18.3
    - github.com/pkoukk/tiktoken-go v0.1.8
    - github.com/cenkalti/backoff/v4 v4.3.0
    - github.com/zricethezav/gitleaks/v8 v8.30.1
  patterns:
    - Provider interface with channel-based streaming
    - 50ms token batching with Wails EventsEmit sequence numbers
    - TDD red-green cycle for all adapters
key_files:
  created:
    - services/llm/provider.go
    - services/llm/registry.go
    - services/llm/openai.go
    - services/llm/anthropic.go
    - services/llm/ollama.go
    - services/llm/context.go
    - services/llm_service.go
    - services/llm/provider_test.go
    - services/llm/openai_test.go
    - services/llm/anthropic_test.go
    - services/llm/ollama_test.go
    - services/llm_service_test.go
  modified:
    - go.mod
    - go.sum
decisions:
  - "Anthropic buildParams is an unexported method tested via package llm internal tests (not package llm_test) to avoid requiring an exported API surface"
  - "Ollama localhost validation errors on any non-localhost hostname including remote IPs, ::1 accepted"
  - "OpenAI adapter uses package-level SystemMessage/UserMessage/AssistantMessage helpers from openai-go/v3 for clean message construction"
  - "LLMService.SendMessage returns error immediately for nil provider; streaming errors go to llm:error event"
metrics:
  duration: "~24 minutes"
  completed_date: "2026-03-28"
  tasks_completed: 2
  files_created: 13
---

# Phase 2 Plan 1: LLM Gateway - Provider Interface and Adapters Summary

**One-liner:** Go LLM gateway with Provider interface, OpenAI/Anthropic/Ollama adapters, and Wails-bound LLMService streaming tokens via 50ms-batched EventsEmit with sequence numbers.

## What Was Built

Five LLM provider implementations behind a common channel-based streaming interface, plus the Wails-bound LLMService that connects them to the frontend via `llm:chunk`, `llm:done`, `llm:error`, and `llm:usage` events.

### Provider Interface (`services/llm/provider.go`)

Defines `Provider` interface with three methods:
- `Name() string` — identifies the provider
- `Stream(ctx, messages) (<-chan StreamChunk, error)` — returns a channel of token chunks
- `TestConnection(ctx) error` — verifies credentials (for Phase 5 settings dialog)

Supporting types: `Role` (system/user/assistant), `Message`, `StreamChunk` (Text/Done/Error), `Usage`.

### OpenAI Adapter (`services/llm/openai.go`)

`OpenAIProvider` covers three services via a configurable `baseURL`:
- **OpenAI**: empty baseURL, `OPENAI_API_KEY`
- **OpenRouter**: `baseURL="https://openrouter.ai/api/v1"`, `OPENROUTER_API_KEY`
- **LM Studio**: `baseURL="http://localhost:1234/v1"`, empty apiKey

Uses `github.com/openai/openai-go/v3` SSE streaming via `client.Chat.Completions.NewStreaming()`. Empty API key is valid (no panic, supports LM Studio). Package-level `SystemMessage()`/`UserMessage()`/`AssistantMessage()` helpers used for message construction.

### Anthropic Adapter (`services/llm/anthropic.go`)

`AnthropicProvider` handles the Anthropic-specific requirement of extracting system messages to `MessageNewParams.System` (not the messages array). The `buildParams()` method iterates the canonical message slice and places system content in the top-level field. Uses `stream.Next()`/`stream.Current()` iterator with `delta.Type == "text_delta"` check to extract text content.

### Ollama Adapter (`services/llm/ollama.go`)

`OllamaProvider` wraps Ollama's callback-based `client.Chat()` into the channel interface via a goroutine. Enforces that `OLLAMA_HOST` must be localhost/127.0.0.1/::1 via `validateOllamaHost()` — remote hosts are rejected to prevent terminal data leaking over network. Empty host defaults to `localhost:11434`.

### Context Assembly (`services/llm/context.go`)

`BuildMessages()` assembles `[system, user]` message slices. When terminal context is non-empty, it is prepended as a fenced code block: ` ```terminal\n{ctx}\n``` ` followed by the user input. `EstimateTokens()` provides a fast 4-chars-per-token approximation.

### Registry (`services/llm/registry.go`)

Simple map-based `Registry` with `Register(Provider)` and `Get(name) (Provider, error)` for Phase 5 settings dialog and multi-provider management.

### LLMService (`services/llm_service.go`)

Wails-bound service following the `CommandService` lifecycle pattern:
- `Config` struct with all env var fields
- `LoadConfig()` reads from `PAIRADMIN_PROVIDER`, `PAIRADMIN_MODEL`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENROUTER_API_KEY`, `OLLAMA_HOST`
- `NewLLMService(cfg)` builds the active provider via `buildProvider()`
- `Startup(ctx)` saves the Wails context
- `SendMessage(tabId, userInput, terminalContext)` — returns immediately; goroutine streams tokens through 50ms batching ticker → emits `llm:chunk` events with sequence numbers, terminates with `llm:done` or `llm:error`

**50ms batching rationale:** Wails Issue #2759 confirms out-of-order delivery at rapid-fire emit rates. The ticker collapses token bursts into ≤20 events/second. Sequence numbers allow the frontend to detect and handle ordering issues.

## TDD Execution

**RED commit** (`3c05871`): All test files created with failing tests (compile errors for undefined types/functions). Confirmed by `go test ./services/...` showing "undefined: X" errors.

**GREEN commit** (`1ed6602`): All implementations written; `go test ./services/... -count=1` exits 0.

## Verification Results

```
go test ./services/... -count=1
ok  pairadmin/services       0.005s
ok  pairadmin/services/llm   0.008s

go build pairadmin/services/...  (exit 0)
```

Note: `go build ./...` fails with "pattern all:frontend/dist: no matching files found" — this is a pre-existing Wails build constraint requiring the frontend to be compiled first. Not a regression; confirmed in Phase 1 as expected behavior.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing functionality] Internal test package for unexported buildParams**
- **Found during:** Task 1 (RED)
- **Issue:** `anthropic_test.go` tests the unexported `buildParams` method; `package llm_test` cannot access it
- **Fix:** Changed `anthropic_test.go` to `package llm` (internal test) instead of `package llm_test`
- **Files modified:** `services/llm/anthropic_test.go`
- **Commit:** 3c05871

**2. [Rule 1 - Bug] RawContentBlockDeltaUnion type assertion fix**
- **Found during:** Task 2 (GREEN) writing anthropic.go
- **Issue:** `anyRawContentBlockDelta` is an unexported interface; type assertion `delta.AsAny().(anthropic.TextDelta)` would not compile
- **Fix:** Used `delta.Type == "text_delta"` check followed by `delta.AsTextDelta()` call instead
- **Files modified:** `services/llm/anthropic.go`
- **Commit:** 1ed6602

**3. [Rule 2 - Missing] All test files moved to package llm (internal)**
- **Found during:** Task 1 design
- **Issue:** All test files referencing unexported fields (`p.baseURL`, `p.buildParams`) need `package llm` scope
- **Fix:** All `*_test.go` files in `services/llm/` use `package llm` (internal) to enable direct field access for white-box testing
- **Files modified:** All test files in `services/llm/`
- **Commit:** 3c05871

## Known Stubs

None — all provider adapters connect to real SDKs. The `activeProvider` is wired correctly in `NewLLMService`. No placeholder data flows to the frontend.

## Self-Check: PASSED

All implementation files confirmed present:
- services/llm/provider.go: FOUND
- services/llm/registry.go: FOUND
- services/llm/openai.go: FOUND
- services/llm/anthropic.go: FOUND
- services/llm/ollama.go: FOUND
- services/llm/context.go: FOUND
- services/llm_service.go: FOUND

Commits verified:
- 3c05871: test(02-01) - FOUND
- 1ed6602: feat(02-01) - FOUND
