---
status: partial
phase: 07-distribution-launch
source: [07-03-PLAN.md]
started: 2026-04-02T00:00:00.000Z
updated: 2026-04-02T00:00:00.000Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Ubuntu 22.04 — .deb clean install
expected: dpkg installs without errors; pairadmin command launches app window; desktop entry appears in app menu
result: [pending]

### 2. Ubuntu 24.04 — .deb clean install
expected: dpkg installs without errors; app launches and renders correctly; can connect to Ollama and complete chat
result: [pending]

### 3. Fedora 40 — .rpm clean install
expected: rpm installs without errors; app launches and renders correctly
result: [pending]

### 4. AppImage — any Linux
expected: AppImage launches (or fails with WebKit error per Issue #4313 — README documents limitation)
result: [pending]

### 5. Checksum verification
expected: sha256sum --check SHA256SUMS — all checksums match
result: [pending]

### 6. Final v1 acceptance criteria check
expected: All REQUIREMENTS.md v1 items satisfied by live app
result: [pending]

## Summary

total: 6
passed: 0
issues: 0
pending: 6
skipped: 0
blocked: 0

## Gaps
