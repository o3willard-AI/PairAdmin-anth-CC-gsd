---
phase: 04-linux-gui-terminal-adapters-at-spi2
plan: 04
subsystem: filter
tags: [viper, yaml, config, custom-filter, slash-commands, wails, zustand, chat-ux]

# Dependency graph
requires:
  - phase: 04-02
    provides: CaptureManager multi-adapter architecture with filter.Pipeline
  - plan: 04-04-task1
    provides: AppConfig/CustomPattern Viper config layer and CustomFilter implementation

provides:
  - FilterCommand Wails binding for /filter add|list|remove slash commands
  - Custom filter patterns persisted in ~/.pairadmin/config.yaml via Viper
  - /filter routing in ChatPane with system message feedback in chat pane
  - addSystemMessage action in chatStore for system role messages
  - System message rendering in ChatMessageList (italic, muted, no avatar)
  - CaptureManager buildFilterPipeline/RebuildFilterPipeline for live custom filter pipeline
  - CustomFilter injected into CaptureManager pipeline and applied to terminal:update content

affects:
  - 05-settings-config (may reuse AppConfig/Viper layer for settings persistence)
  - 06-security-hardening (custom filter pipeline is the extension point for additional credential patterns)

# Tech tracking
tech-stack:
  added:
    - github.com/spf13/viper@v1.21.0 (YAML config persistence for custom filter patterns)
  patterns:
    - filterPipelineRebuilder interface on LLMService decouples LLMService from capture package
    - addSystemMessage in chatStore for inline command output (not just user/assistant roles)
    - CaptureManager.buildFilterPipeline nil-returns when no patterns for zero-overhead path
    - RebuildFilterPipeline called synchronously after SaveAppConfig for immediate effect
    - applyFilterPipeline package-level function retained for ATSPIAdapter per-capture filtering

key-files:
  created:
    - services/config/config.go
    - services/config/config_test.go
    - services/llm/filter/custom.go
    - services/llm/filter/custom_test.go
  modified:
    - services/llm_service.go (FilterCommand + SetCaptureManager + filterPipelineRebuilder interface)
    - services/capture/manager.go (pipeline field, buildFilterPipeline, RebuildFilterPipeline)
    - services/capture/manager_test.go (5 new filter pipeline tests)
    - frontend/src/stores/chatStore.ts (system role + addSystemMessage)
    - frontend/src/components/chat/ChatPane.tsx (/filter routing)
    - frontend/src/components/chat/ChatMessageList.tsx (system message rendering)
    - main.go (llmService.SetCaptureManager(manager))

key-decisions:
  - "filterPipelineRebuilder interface on LLMService — avoids capture package import in services package; duck-typing consistent with OnboardingRequired pattern in 04-02"
  - "CaptureManager.pipeline nil when no custom patterns — avoids allocating empty pipeline on hot path; nil check in tick before Apply"
  - "applyFilterPipeline retained as package-level function — ATSPIAdapter uses it for per-capture ANSI+credential filtering; CaptureManager.pipeline adds CustomFilter on top after adapter capture"
  - "system role in ChatMessage — addSystemMessage adds inline command output without triggering LLM stream machinery; renders with italic/muted styling distinguishable from assistant responses"

patterns-established:
  - "Pipeline nil-return pattern: buildFilterPipeline returns nil when no filters to avoid zero-filter allocation on hot poll path"
  - "Interface decoupling: filterPipelineRebuilder interface on LLMService enables rebuild without importing capture package"
  - "System message UX: /filter command output appears inline in chat as italic muted text with whitespace-pre-wrap for multi-line list output"

requirements-completed: [FILT-04, FILT-05]

# Metrics
duration: 35min
completed: 2026-03-28
---

# Phase 4 Plan 04: Custom Filter Slash Commands and CaptureManager Pipeline Wiring Summary

**User-configurable /filter add|list|remove slash commands with Viper YAML persistence, custom filter pipeline injected into CaptureManager, and system message rendering in chat pane.**

## Performance

- **Duration:** 35 min
- **Started:** 2026-03-28T01:05:00Z
- **Completed:** 2026-03-28T01:40:00Z
- **Tasks:** 3/3 (Task 1 pre-committed, Tasks 2 and 3 executed in this session)
- **Files modified:** 10

## Accomplishments

### Task 1 (pre-committed 688ad4e): Viper config layer and CustomFilter implementation
- `services/config/config.go` — AppConfig, CustomPattern, LoadAppConfig, SaveAppConfig (Viper, ~/.pairadmin/config.yaml)
- `services/config/config_test.go` — 5 tests covering round-trip, directory creation, missing file
- `services/llm/filter/custom.go` — CustomFilter implementing Filter interface (redact/remove actions, CustomPatternInput to avoid Viper import in filter package)
- `services/llm/filter/custom_test.go` — 5 tests (compile, invalid regex, redact, remove, multi-pattern)
- go.mod/go.sum updated with viper@v1.21.0

### Task 2 (9e03dba): FilterCommand Wails binding and /filter chat routing
- `services/llm_service.go` — FilterCommand method with add|list|remove subcommands, regex validation, duplicate name check; SetCaptureManager wiring via filterPipelineRebuilder interface; regexp and config imports added
- `main.go` — llmService.SetCaptureManager(manager) wiring
- `frontend/src/stores/chatStore.ts` — system role added to ChatMessage type; addSystemMessage action added
- `frontend/src/components/chat/ChatPane.tsx` — /filter routing using dynamic Wails FilterCommand import, addSystemMessage for response/error
- `frontend/src/components/chat/ChatMessageList.tsx` — system role case renders italic muted text with whitespace-pre-wrap

### Task 3 (b079a77): CaptureManager filter pipeline wiring
- `services/capture/manager.go` — pipeline *filter.Pipeline field; buildFilterPipeline() loads AppConfig and builds CustomFilter; RebuildFilterPipeline() acquires mu and rebuilds; tick applies pipeline to captured content before terminal:update emit; applyFilterPipeline package-level function retained for ATSPIAdapter
- `services/capture/manager_test.go` — 5 new tests: buildFilterPipeline with/without patterns, RebuildFilterPipeline picks up new patterns, redact applied during tick, remove applied during tick

## Test Results

- `go test ./services/...` — all packages green (services, capture, config, llm, llm/filter)
- `cd frontend && npx vitest run` — 68 tests pass (11 test files)

## Files

### Created
- `services/config/config.go`
- `services/config/config_test.go`
- `services/llm/filter/custom.go`
- `services/llm/filter/custom_test.go`

### Modified
- `services/llm_service.go`
- `services/capture/manager.go`
- `services/capture/manager_test.go`
- `frontend/src/stores/chatStore.ts`
- `frontend/src/components/chat/ChatPane.tsx`
- `frontend/src/components/chat/ChatMessageList.tsx`
- `main.go`

## Decisions Made

- **filterPipelineRebuilder interface on LLMService** — duck-typing consistent with OnboardingRequired pattern from 04-02; avoids capture package import in services
- **CaptureManager.pipeline nil when no custom patterns** — zero-overhead on the hot 500ms poll path when no patterns configured
- **applyFilterPipeline retained as package-level function** — ATSPIAdapter.Capture calls it directly for ANSI+credential filtering; CaptureManager pipeline adds CustomFilter as a second pass on top
- **system role in ChatMessage** — extends existing user/assistant types; addSystemMessage is the injection point for any future slash command output

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all custom filter functionality is fully wired end-to-end: /filter commands persist via Viper, CustomFilter is built from AppConfig at CaptureManager startup, RebuildFilterPipeline updates patterns immediately after /filter add|remove.

## Self-Check: PASSED
