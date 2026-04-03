---
phase: 07-distribution-launch
plan: 01
subsystem: infra
tags: [nfpm, deb, rpm, linux-packaging, desktop-entry, freedesktop, appicon, install-deps]

# Dependency graph
requires:
  - phase: 06-security-hardening
    provides: completed Go binary build infrastructure
provides:
  - nfpm.yaml packaging config for .deb and .rpm production
  - build/linux/pairadmin.desktop freedesktop desktop entry
  - build/linux/pairadmin.png 256x256 application icon
  - scripts/install-deps.sh build dependency installer for Ubuntu/Debian and Fedora/RHEL
affects: [07-distribution-launch plan 02 — consumes nfpm.yaml and build/linux/ to produce artifacts]

# Tech tracking
tech-stack:
  added: [nFPM v2.46.0 (packaging tool — install separately), Python PIL (icon resize — system-available)]
  patterns: [nFPM overrides.rpm.depends for per-format dependency names, /etc/os-release sourcing for distro detection, --no-upgrade apt-get for idempotent installs]

key-files:
  created:
    - nfpm.yaml
    - build/linux/pairadmin.desktop
    - build/linux/pairadmin.png
    - scripts/install-deps.sh
  modified: []

key-decisions:
  - "nFPM overrides.rpm.depends used to specify webkit2gtk4.1 for RPM while libwebkit2gtk-4.1-0 remains top-level deb dep"
  - "Binary file_info mode: 0755 in nfpm.yaml prevents permission denied on installed binary (pitfall 5)"
  - "Install path /usr/local/bin/pairadmin per D-06 (locally-distributed apps, not /usr/bin)"
  - "Icon resized with PIL LANCZOS — available immediately without apt install imagemagick"
  - "--no-upgrade apt-get flag makes install-deps.sh idempotent on Ubuntu/Debian per D-12"

patterns-established:
  - "nFPM overrides block: per-format dependency names when deb and rpm package names differ"
  - "distro detection via . /etc/os-release then case $ID — sources the file to get clean shell variables"
  - "ID_LIKE fallback with grep -qE for RHEL/CentOS variants"

requirements-completed: [DIST-01, DIST-03, DIST-04]

# Metrics
duration: 5min
completed: 2026-04-03
---

# Phase 7 Plan 01: Distribution Config Files Summary

**nFPM packaging config with per-distro webkit deps, freedesktop .desktop entry, 256x256 icon (PIL LANCZOS), and idempotent install-deps.sh for Ubuntu/Debian and Fedora/RHEL**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-03T06:59:17Z
- **Completed:** 2026-04-03T07:04:00Z
- **Tasks:** 2
- **Files modified:** 4 created

## Accomplishments

- nfpm.yaml with correct per-distro runtime deps: `libwebkit2gtk-4.1-0` (deb) and `webkit2gtk4.1` (rpm via overrides block)
- Binary entry with `file_info: mode: 0755` preventing the common pitfall of non-executable installed binaries
- build/linux/pairadmin.desktop with locked D-07 content (`Exec=/usr/local/bin/pairadmin`, `Terminal=false`)
- build/linux/pairadmin.png resized to 256x256 from 1024x1024 appicon.png using Python PIL LANCZOS
- scripts/install-deps.sh with root check, /etc/os-release sourcing, ubuntu|debian and fedora branches, ID_LIKE RHEL/CentOS fallback, and --no-upgrade for idempotency

## Task Commits

Each task was committed atomically:

1. **Task 1: Create nFPM config and Linux build assets** - `071916f` (feat)
2. **Task 2: Create install-deps.sh build dependency script** - `8963db0` (feat)

## Files Created/Modified

- `nfpm.yaml` - nFPM packaging config producing .deb and .rpm with correct per-distro runtime dependencies
- `build/linux/pairadmin.desktop` - Freedesktop desktop entry per D-07 spec
- `build/linux/pairadmin.png` - 256x256 application icon resized from build/appicon.png
- `scripts/install-deps.sh` - Build dependency installer detecting Ubuntu/Debian vs Fedora/RHEL via /etc/os-release

## Decisions Made

- Used Python PIL for icon resize (available immediately, no apt install required vs ImageMagick)
- Used `Image.LANCZOS` resize filter for highest quality downscaling
- nFPM `overrides.rpm.depends` block is the correct approach for different package names across distro families
- `--no-upgrade` flag in apt-get makes the script idempotent per D-12 without complex version checking

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Plan 02 can now consume nfpm.yaml and build/linux/ to run `nfpm pkg --packager deb` and `nfpm pkg --packager rpm`
- nFPM must be installed before Plan 02: `go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest`
- Binary must be built first: `wails build -platform linux/amd64 -tags webkit2_41`
- All four config/asset files exist and verified

## Known Stubs

None - all files are complete and functional. No stub values or placeholder content.

## Self-Check: PASSED

- FOUND: nfpm.yaml
- FOUND: build/linux/pairadmin.desktop
- FOUND: build/linux/pairadmin.png
- FOUND: scripts/install-deps.sh
- FOUND: .planning/phases/07-distribution-launch/07-01-SUMMARY.md
- FOUND commit: 071916f (Task 1)
- FOUND commit: 8963db0 (Task 2)

---
*Phase: 07-distribution-launch*
*Completed: 2026-04-03*
