---
phase: 02-llm-gateway-streaming-chat
plan: "02"
subsystem: filter
tags: [security, ansi-stripping, credential-redaction, pipeline, tdd]
dependency_graph:
  requires: []
  provides:
    - services/llm/filter.Filter (interface)
    - services/llm/filter.Pipeline (type)
    - services/llm/filter.ANSIFilter (type)
    - services/llm/filter.CredentialFilter (type)
  affects:
    - LLMService (downstream consumer — Phase 02 plans 03+)
tech_stack:
  added: []
  patterns:
    - Filter interface + Pipeline composition (input → ANSIFilter → CredentialFilter → clean output)
    - Regex-based ANSI sequence stripping (CSI/OSC/DCS/cursor movement)
    - Regex credential pattern matching with [REDACTED:<rule_id>] replacement
    - TDD (RED → GREEN) with table-driven tests
key_files:
  created:
    - services/llm/filter/filter.go
    - services/llm/filter/ansi.go
    - services/llm/filter/credential.go
    - services/llm/filter/filter_test.go
  modified: []
decisions:
  - "Replaced go-ansi-parser Cleanse with comprehensive regex (CSI/OSC/DCS coverage) because the library only handles SGR color codes"
  - "Regex-only credential detection (no gitleaks dependency) — gitleaks not in go.mod and adding it would be an architectural change"
metrics:
  duration: "166s"
  completed: "2026-03-27"
  tasks_completed: 2
  files_created: 4
  files_modified: 0
---

# Phase 02 Plan 02: Filter Pipeline Summary

**One-liner:** Two-stage ANSI-stripping + credential-redaction pipeline using comprehensive regex patterns, with `Filter` interface and composable `Pipeline` for pre-LLM content sanitization.

## What Was Built

The `services/llm/filter` package provides security-critical content sanitization before any terminal content is sent to cloud LLM APIs:

1. **`filter.go`** — `Filter` interface and `Pipeline` type. `NewPipeline(filters ...Filter)` composes filters in order; `Apply` short-circuits on error.

2. **`ansi.go`** — `ANSIFilter` strips all ANSI/VT100 escape sequences using a comprehensive regex covering CSI (color, cursor movement, erase), OSC (window title), DCS, APC, PM, SOS, and simple 2-byte ESC sequences. This is Stage 1 — must run before credential scanning.

3. **`credential.go`** — `CredentialFilter` applies 6 regex patterns to redact common credential formats. Each match is replaced with `[REDACTED:<rule_id>]`. Patterns: AWS access keys, GitHub tokens, OpenAI keys, Anthropic keys, bearer tokens, generic API keys.

4. **`filter_test.go`** — 8 tests covering: ANSI color stripping, cursor movement stripping, OSC sequence stripping, plain text passthrough, AWS key redaction, GitHub token redaction, bearer token redaction, safe text passthrough, pipeline order (ANSI before credential), ANSI-wrapped credential redaction.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] go-ansi-parser Cleanse does not strip cursor movement or OSC sequences**
- **Found during:** Task 2 GREEN phase (tests failed with `\x1b[1A` and `\x1b[2J` not stripped)
- **Issue:** `ansi.Cleanse()` from `github.com/leaanthony/go-ansi-parser` only processes SGR color/style sequences (parsed via `Parse()`). Cursor movement sequences (`ESC[1A`, `ESC[2J`) and OSC sequences (`ESC]0;title\x07`) are not recognized by the parser and pass through unmodified.
- **Fix:** Replaced `ansi.Cleanse()` with a comprehensive ANSI regex pattern matching all VT100/CSI/OSC/DCS sequence forms. The import of `go-ansi-parser` was removed from `ansi.go` since the library's coverage was insufficient.
- **Files modified:** `services/llm/filter/ansi.go`
- **Commit:** 21a2761

**2. [Rule 1 - Deviation] Gitleaks not added as dependency — regex-only mode used**
- **Found during:** Task 2 (gitleaks not in go.mod)
- **Issue:** Plan flagged gitleaks integration as "MEDIUM confidence on exact API" and stated "use regex-only mode and log a warning to stderr (do not fail)" if API differs. Since gitleaks is not in go.mod at all, adding it would require `go get github.com/zricethezav/gitleaks/v8` (a new dependency). The plan's fallback strategy of regex-only is the correct path.
- **Fix:** `CredentialFilter` uses regex-only mode. `NewCredentialFilter()` returns a `*CredentialFilter` with no gitleaks dependency. The plan's required regex patterns are all implemented.
- **Files modified:** `services/llm/filter/credential.go`
- **Commit:** 21a2761

## Test Results

```
PASS
ok  pairadmin/services/llm/filter  0.004s
```

8/8 tests pass. `go vet ./services/llm/filter/...` clean.

## Known Stubs

None — all filter logic is wired and functional.

## Self-Check: PASSED
