---
phase: 7
slug: distribution-launch
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Shell scripts (bash -n, file existence checks, grep); no Go/vitest tests — this phase is pure distribution infrastructure |
| **Config file** | none |
| **Quick run command** | `bash -n scripts/install-deps.sh && test -f nfpm.yaml && test -f build/linux/pairadmin.desktop` |
| **Full suite command** | `bash -n scripts/install-deps.sh && nfpm pkg --packager deb -f nfpm.yaml && nfpm pkg --packager rpm -f nfpm.yaml && ls build/bin/pairadmin *.deb *.rpm` |
| **Estimated runtime** | ~60 seconds (includes wails build) |

---

## Sampling Rate

- **After every task commit:** Run quick check — `bash -n scripts/install-deps.sh` or `test -f <expected_file>`
- **After every plan wave:** Run full artifact build if tooling is installed
- **Before `/gsd:verify-work`:** Human checklist must be run (D-13 — cannot automate clean install test)
- **Max feedback latency:** 60 seconds for automated; human checklist is asynchronous

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 1 | DIST-01/03 | file-check | `test -f nfpm.yaml && grep 'libwebkit2gtk-4.1-0' nfpm.yaml` | ❌ W0 | ⬜ pending |
| 07-01-02 | 01 | 1 | DIST-01/03 | file-check | `test -f build/linux/pairadmin.desktop && grep 'Name=PairAdmin' build/linux/pairadmin.desktop` | ❌ W0 | ⬜ pending |
| 07-01-03 | 01 | 1 | DIST-01/03 | file-check | `test -f build/linux/pairadmin.png` | ❌ W0 | ⬜ pending |
| 07-02-01 | 02 | 2 | DIST-04 | syntax-check | `bash -n scripts/install-deps.sh && grep 'ubuntu\|debian\|fedora' scripts/install-deps.sh` | ❌ W0 | ⬜ pending |
| 07-02-02 | 02 | 2 | DIST-01/02/03 | build | `wails build -platform linux/amd64 && ls build/bin/pairadmin` | ✅ (wails installed) | ⬜ pending |
| 07-02-03 | 02 | 2 | DIST-01/03 | build | `nfpm pkg --packager deb -f nfpm.yaml && ls *.deb` | ❌ W0 (nfpm) | ⬜ pending |
| 07-02-04 | 02 | 2 | DIST-02 | build | `test -f *.AppImage || echo "AppImage-limitation-documented"` | ❌ W0 (appimagetool) | ⬜ pending |
| 07-03-01 | 03 | 3 | DIST-01/02/03/04 | manual | Human checklist — install on Ubuntu 22.04, 24.04, Fedora 40 | N/A (human) | ⬜ pending |
| 07-03-02 | 03 | 3 | all | manual | Final v1 acceptance criteria check against REQUIREMENTS.md | N/A (human) | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `nfpm` installed: `go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest`
- [ ] `gh` CLI installed: `sudo apt install gh` or `sudo dnf install gh`
- [ ] `appimagetool` downloaded: `wget https://github.com/AppImage/appimagetool/releases/download/continuous/appimagetool-x86_64.AppImage -O appimagetool && chmod +x appimagetool`
- [ ] `build/linux/` directory created with `.desktop` and resized icon

*These are prerequisites, not stubs — Wave 1 plan must include installation steps or document them for the executor.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Clean install from .deb on Ubuntu 22.04 | DIST-01 | Requires real/VM system with fresh OS | `sudo dpkg -i pairadmin_*.deb && pairadmin` — verify app launches |
| Clean install from .deb on Ubuntu 24.04 | DIST-01 | Requires fresh OS environment | `sudo dpkg -i pairadmin_*.deb && pairadmin` |
| Clean install from .rpm on Fedora 40 | DIST-03 | Requires Fedora VM | `sudo rpm -i pairadmin_*.rpm && pairadmin` |
| AppImage runtime limitation | DIST-02 | WebKit path issue requires running the AppImage | Run AppImage, observe webkit error, confirm documented in README |
| Final v1 acceptance criteria | All | Requires running app + manual interaction | Check all REQUIREMENTS.md items against live app |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s for automated tasks
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
