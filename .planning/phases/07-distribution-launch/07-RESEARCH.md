# Phase 7: Distribution & Launch - Research

**Researched:** 2026-03-30
**Domain:** Linux packaging (nFPM, AppImage, shell scripting), Wails v2 build pipeline, GitHub releases
**Confidence:** HIGH

## Summary

Phase 7 is pure distribution infrastructure — no new Go or frontend code. The deliverables are a working nFPM config, `scripts/install-deps.sh`, `build/linux/` assets, an AppImage artifact (best-effort), and a GitHub release with SHA-256 checksums.

The most important architectural finding is confirmed from Wails v2.12.0 source: **`wails build` on Linux is a no-op for packaging** — `packager_linux.go` is three lines that return nil. The binary at `build/bin/pairadmin` is the only output. All packaging (`.deb`, `.rpm`, AppImage) is performed by external tools: nFPM for `.deb`/`.rpm`, and `appimagetool` for AppImage.

The AppImage story is well-understood and honest: Wails Issue #4313 is a fundamental incompatibility between WebKit2GTK's hardcoded subprocess paths and AppImage filesystem isolation. The AppImage is a best-effort artifact; `.deb` is the recommended install path and must be documented as such.

**Primary recommendation:** Build the binary with `wails build -platform linux/amd64 -tags webkit2_41`, package with `nfpm pkg`, build AppImage with `appimagetool` using a hand-crafted AppDir, and release manually via `gh release create`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Release Pipeline**
- D-01: Release creation is manual via `gh` CLI — `gh release create v1.0.0 --title "PairAdmin v1.0.0" ...` with artifact upload. No GitHub Actions workflow for v1.
- D-02: Release tag format: `v1.0.0`. Release assets: `pairadmin_1.0.0_linux_amd64.deb`, `pairadmin_1.0.0_linux_amd64.rpm`, `pairadmin_1.0.0_linux_amd64.AppImage`, `SHA256SUMS`.

**Binary Signing / Checksums**
- D-03: SHA-256 checksums only — generate `SHA256SUMS` with `sha256sum pairadmin_*`, upload alongside binaries. No GPG.
- D-04: No GPG signing in v1. Note in README that GPG signing is planned.

**Package Contents**
- D-05: nFPM packages include binary at `/usr/local/bin/pairadmin`, desktop entry at `/usr/share/applications/pairadmin.desktop`, icon at `/usr/share/icons/hicolor/256x256/apps/pairadmin.png`. Runtime deps: `libwebkit2gtk-4.1-0` (Ubuntu), `webkit2gtk4.1` or `webkitgtk6.0` (Fedora), `at-spi2-core`.
- D-06: Install paths: `/usr/local/bin/pairadmin` (binary), `/usr/share/...` (assets). Not `/usr/bin`.
- D-07: `.desktop` file content locked (see CONTEXT.md §D-07).
- D-08: nFPM config at root: `nfpm.yaml`. Produces both `.deb` and `.rpm`.

**AppImage**
- D-09: AppImage via appimagetool (Wails does not natively produce AppImage in v2). Document webkit limitation and recommend `.deb`.
- D-10: AppImage is best-effort. If webkit bundling fails, ship `.deb` and `.rpm` and document clearly.

**Install Dependency Script**
- D-11: `scripts/install-deps.sh` detects distro via `ID` in `/etc/os-release`. Ubuntu/Debian: `apt-get install -y libwebkit2gtk-4.1-dev at-spi2-core gcc`. Fedora/RHEL: `dnf install -y webkit2gtk4.1-devel at-spi2-atk-devel gcc`. Exit with clear error on unsupported distros.
- D-12: Script is idempotent.

**Clean Install Test**
- D-13: Human checklist, not automated. Ubuntu 22.04 `.deb`, Ubuntu 24.04 `.deb`, Fedora 40 `.rpm`.
- D-14: Final v1 acceptance check is a human verification checkpoint — last plan.

**README**
- D-15: Update root `README.md` with installation section per package type, prerequisites (Ollama/cloud API key), AppImage limitation note, quick start.

### Claude's Discretion

- nFPM version to use (latest stable — confirmed v2.46.0)
- Exact nFPM `version_schema` and changelog format
- AppImage toolchain choice (appimagetool vs wails native) — researcher resolved: appimagetool required; Wails v2 has no native AppImage support
- Whether to create a `Makefile` with `make dist` target

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DIST-01 | Application builds as a `.deb` package via nFPM with `libwebkit2gtk-4.1-0` declared as a runtime dependency | nFPM v2.46.0 `nfpm pkg --packager deb`; overrides block for deb-specific deps |
| DIST-02 | Application builds as an AppImage (with documented fallback to `.deb` for webkit runtime issues) | appimagetool AppDir structure; confirmed Issue #4313 is unfixed in Wails v2; best-effort strategy defined |
| DIST-03 | Application builds as an `.rpm` package via nFPM | Same nFPM config; `overrides.rpm.depends` with `webkit2gtk4.1`; `nfpm pkg --packager rpm` |
| DIST-04 | Install script (`scripts/install-deps.sh`) installs all build-time dependencies on Ubuntu/Debian and Fedora/RHEL | `/etc/os-release` ID detection pattern documented; exact packages confirmed |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| nFPM | v2.46.0 | `.deb` and `.rpm` packaging from single YAML config | Zero-dependency Go tool; GoReleaser standard; no Ruby/FPM required |
| appimagetool | continuous (v2.x) | Create AppImage from AppDir | Official AppImage project tool; single-binary download; no installation required |
| wails | v2.12.0 (installed) | Build Go+React binary for Linux | Already in use; `-platform linux/amd64` flag confirmed |
| gh CLI | system (not currently installed) | Create GitHub release, upload artifacts | Official GitHub CLI; script-friendly |
| sha256sum | 9.4 (GNU coreutils, installed) | Generate SHA256SUMS file | Pre-installed on all Linux systems |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| ImageMagick convert | 6.9.12 (apt-installable) | Resize `appicon.png` from 1024x1024 to 256x256 | Required to produce nFPM icon at correct size |
| dpkg-deb | system (installed) | Inspect/verify `.deb` package contents | Testing/validation only |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| nFPM | fpm (Ruby) | fpm requires Ruby runtime; nFPM is zero-dependency Go binary |
| appimagetool | linuxdeploy + plugin | linuxdeploy automates dependency bundling (less useful here since webkit can't be bundled anyway); appimagetool is simpler for a no-bundling AppImage |
| appimagetool | Wails native AppImage | Wails v2 `packager_linux.go` is a no-op (confirmed from source) — no native AppImage support in v2 |

**Installation (nFPM):**
```bash
# Via go install (requires Go 1.24+ in PATH)
go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest

# OR via GoReleaser APT repo
echo 'deb [trusted=yes] https://repo.goreleaser.com/apt/ /' | sudo tee /etc/apt/sources.list.d/goreleaser.list
sudo apt update && sudo apt install nfpm
```

**Installation (appimagetool):**
```bash
wget -q "https://github.com/AppImage/AppImageKit/releases/download/continuous/appimagetool-x86_64.AppImage" -O appimagetool
chmod +x appimagetool
# Run with: ARCH=x86_64 ./appimagetool AppDir/ pairadmin_1.0.0_linux_amd64.AppImage
```

**Installation (gh CLI):**
```bash
sudo apt install gh    # or: sudo dnf install gh
```

**Version verification (nFPM):** Confirmed v2.46.0 published 2026-03-31 via GitHub Releases API.

## Architecture Patterns

### Recommended Project Structure
```
build/
├── appicon.png          # 1024×1024 source icon (existing)
├── bin/                 # wails build output (pairadmin binary)
├── darwin/              # existing macOS assets (reference)
└── linux/               # NEW: Linux build assets
    ├── pairadmin.desktop
    └── pairadmin.png    # 256×256 resized from appicon.png

scripts/
└── install-deps.sh      # NEW: build-time dependency installer

nfpm.yaml                # NEW: root-level nFPM config
```

### Pattern 1: nFPM YAML with Per-Format Dependency Overrides

**What:** A single `nfpm.yaml` that produces both `.deb` and `.rpm` using the `overrides` block to specify format-specific dependency names.

**When to use:** The Ubuntu package name `libwebkit2gtk-4.1-0` differs from the Fedora name `webkit2gtk4.1` — both must be declared in the same config.

**Example:**
```yaml
# Source: https://nfpm.goreleaser.com/docs/configuration/
name: pairadmin
arch: amd64
platform: linux
version: "${VERSION}"
maintainer: PairAdmin Dev <sblanken@pairadmin.local>
description: AI pair programming assistant for terminal workflows
homepage: https://github.com/sblanken/pairadmin
license: MIT

# deb-specific runtime deps (default; overridden for rpm below)
depends:
  - libwebkit2gtk-4.1-0
  - at-spi2-core

overrides:
  rpm:
    depends:
      - webkit2gtk4.1
      - at-spi2-atk

contents:
  - src: build/bin/pairadmin
    dst: /usr/local/bin/pairadmin
    file_info:
      mode: 0755
  - src: build/linux/pairadmin.desktop
    dst: /usr/share/applications/pairadmin.desktop
  - src: build/linux/pairadmin.png
    dst: /usr/share/icons/hicolor/256x256/apps/pairadmin.png
```

**Build commands:**
```bash
VERSION=1.0.0 nfpm pkg --packager deb --target .
VERSION=1.0.0 nfpm pkg --packager rpm --target .
```

### Pattern 2: AppImage AppDir Structure

**What:** A directory with specific layout that `appimagetool` converts to a single `.AppImage` file.

**When to use:** For the best-effort AppImage artifact.

**AppDir layout:**
```
pairadmin.AppDir/
├── AppRun                    # executable entry point script
├── pairadmin.desktop         # copy of .desktop file (same as build/linux/)
├── pairadmin.png             # 256×256 icon
└── usr/
    └── bin/
        └── pairadmin         # the Wails binary
```

**AppRun script:**
```bash
#!/bin/bash
# Source: AppImage documentation https://docs.appimage.org/packaging-guide/manual.html
SELF=$(readlink -f "$0")
HERE="${SELF%/*}"
export PATH="${HERE}/usr/bin:${PATH}"
exec "${HERE}/usr/bin/pairadmin" "$@"
```

**Build command:**
```bash
ARCH=x86_64 ./appimagetool pairadmin.AppDir/ pairadmin_1.0.0_linux_amd64.AppImage
```

**Important:** The AppImage will likely fail at runtime with `WebKitNetworkProcess` errors (Issue #4313). Build it, document the limitation, and instruct users to prefer `.deb`.

### Pattern 3: install-deps.sh Distro Detection

**What:** Detects distro from `/etc/os-release` and installs the correct build-time packages.

**Key pattern:**
```bash
#!/bin/bash
# Source: D-11 decision + /etc/os-release standard
set -e

. /etc/os-release

case "$ID" in
  ubuntu|debian)
    apt-get install -y libwebkit2gtk-4.1-dev at-spi2-core gcc
    ;;
  fedora)
    dnf install -y webkit2gtk4.1-devel at-spi2-atk-devel gcc
    ;;
  *)
    if echo "$ID_LIKE" | grep -q "rhel\|centos"; then
      dnf install -y webkit2gtk4.1-devel at-spi2-atk-devel gcc
    else
      echo "Unsupported distro: $ID" >&2
      exit 1
    fi
    ;;
esac
```

### Pattern 4: Wails Build with webkit2_41 Tag

**What:** The correct build command for Ubuntu 24.04 and Fedora 40 (webkit2gtk 4.1).

**When to use:** Always — Ubuntu 22.04 supports this tag too (has webkit 4.1 via backports or the package exists).

```bash
wails build -platform linux/amd64 -tags webkit2_41
# Output: build/bin/pairadmin
```

**Ubuntu 22.04 caveat:** May need `libwebkit2gtk-4.1-dev` installed (it's in the repo). The `-tags webkit2_41` flag links against webkit2gtk-4.1 instead of 4.0. Since the project already confirmed this in Phase 1 (`CONFIRMED WORKING`), use this tag consistently.

### Pattern 5: GitHub Release via gh CLI

```bash
sha256sum pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage > SHA256SUMS

gh release create v1.0.0 \
  pairadmin_1.0.0_linux_amd64.deb \
  pairadmin_1.0.0_linux_amd64.rpm \
  pairadmin_1.0.0_linux_amd64.AppImage \
  SHA256SUMS \
  --title "PairAdmin v1.0.0" \
  --notes-file RELEASE_NOTES.md
```

### Anti-Patterns to Avoid

- **Using `/usr/bin` instead of `/usr/local/bin`:** D-06 is explicit: locally-distributed apps go in `/usr/local/bin`.
- **Declaring only deb depends at top level:** RPM packages use different package names; use the `overrides.rpm.depends` block.
- **Relying on `wails build` to produce AppImage:** Confirmed no-op on Linux in v2.12.0.
- **Using `-tags webkit2_40` or no webkit tag:** The build system in v2.12.0 has three tag tiers: legacy (no tag, uses 4.0), webkit2_40, webkit2_41. The project requires 4.1.
- **Shipping appicon.png directly as nFPM icon:** appicon.png is 1024×1024; icon spec at `/usr/share/icons/hicolor/256x256/` requires 256×256. Must resize first.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| .deb / .rpm packaging | Custom dpkg-deb scripts | nFPM | Handles control files, permissions, checksums, pre/post scripts, dependency resolution correctly |
| AppImage creation | Custom squashfs + FUSE scripts | appimagetool | AppImage runtime selection, type 2 format, squashfs params are non-trivial |
| Distro detection | String parsing of /etc/os-release | `. /etc/os-release` (source it) | Provides shell variables ID, ID_LIKE, VERSION_ID correctly without brittle grep |
| GitHub release | curl to GitHub API | `gh release create` | Handles auth, multipart upload, asset content-type, retry |

**Key insight:** Linux packaging has well-maintained tooling for every sub-problem. The only hand-rolling needed is the `AppRun` script (4 lines) and the `.desktop` file (9 lines).

## Common Pitfalls

### Pitfall 1: Wails v2 Does Not Produce AppImage Natively
**What goes wrong:** Running `wails build` expecting an AppImage output file — none is produced. The `-nopackage` flag hints at packaging existing, but `packager_linux.go` is literally `func packageApplication(_ *Options) error { return nil }`.
**Why it happens:** AppImage support exists in Wails v3 (alpha), not v2.
**How to avoid:** Use `appimagetool` with a hand-crafted AppDir. Do not use `wails build --help` output to infer AppImage support.
**Warning signs:** `build/bin/` contains only the binary after `wails build`.

### Pitfall 2: AppImage Runtime Failure (Issue #4313)
**What goes wrong:** AppImage launches but immediately fails: `Unable to spawn a new child process: Failed to spawn child process "/usr/lib/x86_64-linux-gnu/webkit2gtk-4.0/WebKitNetworkProcess"`.
**Why it happens:** WebKit2GTK subprocess helper binaries are hardcoded to absolute system paths. Inside AppImage's SquashFS mount, those paths don't exist.
**How to avoid:** Accept this as a known limitation. Do not spend time trying to fix it for v1. Ship the `.deb` as the recommended install path. Document clearly in README.
**Warning signs:** AppImage crashes on first launch with a WebKit subprocess error.

### Pitfall 3: Wrong Fedora WebKit Package Name
**What goes wrong:** nFPM RPM `depends` lists `libwebkit2gtk-4.1-0` (the Ubuntu/Debian package name) — this package doesn't exist on Fedora. `rpm -i` installs without error, but the app fails to launch.
**Why it happens:** Debian/Ubuntu use `libwebkit2gtk-4.1-0` while Fedora uses `webkit2gtk4.1`. The names are completely different.
**How to avoid:** Use nFPM `overrides.rpm.depends` to specify `webkit2gtk4.1` for RPM format.
**Warning signs:** RPM installs clean but `pairadmin` segfaults or prints `libwebkit2gtk` errors.

### Pitfall 4: appicon.png Size Mismatch
**What goes wrong:** nFPM or AppImage build fails, or the system icon appears blurry/oversized. The `appicon.png` in `build/` is 1024×1024.
**Why it happens:** The install path `/usr/share/icons/hicolor/256x256/apps/` is spec'd for 256×256 icons.
**How to avoid:** Resize with `convert build/appicon.png -resize 256x256 build/linux/pairadmin.png` (ImageMagick is apt-installable; `PIL` is also available).
**Warning signs:** `gtk-update-icon-cache` warnings, or desktop icon looks wrong.

### Pitfall 5: nFPM `file_info.mode` Missing on Binary
**What goes wrong:** Installed binary at `/usr/local/bin/pairadmin` is not executable (mode 0644 default). Application won't launch from desktop entry.
**Why it happens:** nFPM defaults file mode to 0644. Binaries need 0755.
**How to avoid:** Add `file_info: { mode: 0755 }` to the binary content entry in `nfpm.yaml`.
**Warning signs:** `permission denied` when running `pairadmin`, or application doesn't launch from `.desktop`.

### Pitfall 6: VERSION Not Set When Running nFPM
**What goes wrong:** nFPM uses `${VERSION}` in the config; if the env var is not set, the package version becomes empty or literal `${VERSION}`.
**Why it happens:** Shell env var expansion in YAML values requires the var to be exported.
**How to avoid:** Always run as `VERSION=1.0.0 nfpm pkg ...` or `export VERSION=1.0.0` first.
**Warning signs:** `.deb` filename contains empty version or `${VERSION}` literally.

### Pitfall 7: gh CLI Not Installed
**What goes wrong:** `gh` is not present on this build machine (confirmed not installed).
**Why it happens:** Not a standard system package; must be installed separately.
**How to avoid:** Install via `sudo apt install gh` (Ubuntu) or `sudo dnf install gh` (Fedora). Or download binary from GitHub CLI releases. The plan must include a `gh` install step.
**Warning signs:** `command not found: gh` when running release command.

## Code Examples

Verified patterns from official sources:

### Full nfpm.yaml (confirmed syntax from nfpm.goreleaser.com)
```yaml
# Source: https://nfpm.goreleaser.com/docs/configuration/
name: pairadmin
arch: amd64
platform: linux
version: "${VERSION}"
version_schema: semver
release: "1"
maintainer: PairAdmin Dev <sblanken@pairadmin.local>
description: AI pair programming assistant for terminal workflows
homepage: https://github.com/sblanken/pairadmin
license: MIT

depends:
  - libwebkit2gtk-4.1-0
  - at-spi2-core

overrides:
  rpm:
    depends:
      - webkit2gtk4.1
      - at-spi2-atk

contents:
  - src: build/bin/pairadmin
    dst: /usr/local/bin/pairadmin
    file_info:
      mode: 0755
  - src: build/linux/pairadmin.desktop
    dst: /usr/share/applications/pairadmin.desktop
  - src: build/linux/pairadmin.png
    dst: /usr/share/icons/hicolor/256x256/apps/pairadmin.png
```

### AppDir AppRun Script
```bash
#!/bin/bash
# Source: https://docs.appimage.org/packaging-guide/manual.html
SELF=$(readlink -f "$0")
HERE="${SELF%/*}"
export PATH="${HERE}/usr/bin:${PATH}"
exec "${HERE}/usr/bin/pairadmin" "$@"
```

### Full Build Pipeline Script (Makefile-style or bash)
```bash
#!/bin/bash
# Complete build pipeline for distribution
set -e

VERSION="1.0.0"
BINARY="build/bin/pairadmin"

# Step 1: Build binary
wails build -platform linux/amd64 -tags webkit2_41 -clean

# Step 2: Prepare Linux assets
mkdir -p build/linux
convert build/appicon.png -resize 256x256 build/linux/pairadmin.png
# .desktop file must already exist at build/linux/pairadmin.desktop

# Step 3: Package .deb and .rpm
VERSION="${VERSION}" nfpm pkg --packager deb --target .
VERSION="${VERSION}" nfpm pkg --packager rpm --target .

# Step 4: Build AppImage (best-effort)
mkdir -p pairadmin.AppDir/usr/bin
cp "${BINARY}" pairadmin.AppDir/usr/bin/pairadmin
cp build/linux/pairadmin.desktop pairadmin.AppDir/
cp build/linux/pairadmin.png pairadmin.AppDir/
# AppRun must be created with contents above
ARCH=x86_64 ./appimagetool pairadmin.AppDir/ pairadmin_${VERSION}_linux_amd64.AppImage || \
  echo "WARNING: AppImage build failed — expected due to webkit bundling limitation (Issue #4313)"

# Step 5: Checksums
sha256sum pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage 2>/dev/null > SHA256SUMS || \
  sha256sum pairadmin_*.deb pairadmin_*.rpm > SHA256SUMS
```

### .desktop File
```ini
[Desktop Entry]
Name=PairAdmin
Comment=AI pair programming assistant for terminal workflows
Exec=/usr/local/bin/pairadmin
Icon=pairadmin
Type=Application
Categories=Development;Utility;
Terminal=false
```

### install-deps.sh (full idempotent version)
```bash
#!/bin/bash
# Source: D-11 decision; /etc/os-release is FreeDesktop standard
set -e

if [ "$(id -u)" -ne 0 ]; then
  echo "Run as root (sudo)" >&2
  exit 1
fi

. /etc/os-release

case "$ID" in
  ubuntu|debian)
    apt-get update -qq
    apt-get install -y --no-upgrade \
      libwebkit2gtk-4.1-dev at-spi2-core gcc
    ;;
  fedora)
    dnf install -y \
      webkit2gtk4.1-devel at-spi2-atk-devel gcc
    ;;
  *)
    if echo "${ID_LIKE:-}" | grep -qE "rhel|centos|fedora"; then
      dnf install -y \
        webkit2gtk4.1-devel at-spi2-atk-devel gcc
    else
      echo "Unsupported distribution: ${ID}" >&2
      echo "Supported: Ubuntu, Debian, Fedora, RHEL/CentOS" >&2
      exit 1
    fi
    ;;
esac

echo "Dependencies installed successfully."
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `webkit2gtk-4.0` (GTK3+libsoup2) | `webkit2gtk-4.1` (GTK3+libsoup3) | Fedora 40 (2024) | Must use `webkit2gtk4.1` on Fedora; Ubuntu still ships both |
| `fpm` (Ruby packager) | `nFPM` (Go, zero-dep) | ~2020 | No Ruby runtime required; single binary |
| Wails v3 AppImage (alpha) | appimagetool external tool | N/A for v2 | Wails v2 has no native AppImage — always external |

**Deprecated/outdated:**
- `webkit2gtk-4.0` on Fedora 40: Removed from main webkitgtk source; still exists as separate `webkit2gtk4.0` source package but should not be targeted for new distributions
- `webkit2gtk4.0` Fedora RPM dep: Use `webkit2gtk4.1` instead for Fedora 40

## Open Questions

1. **Ubuntu 22.04 and `webkit2_41` build tag**
   - What we know: Phase 1 confirmed `CONFIRMED WORKING` on Ubuntu 24.04 with `-tags webkit2_41`; STATE.md references both Ubuntu 22.04+ with `libwebkit2gtk-4.1-dev`
   - What's unclear: Whether Ubuntu 22.04 ships `libwebkit2gtk-4.1-dev` in the default repos or requires a PPA
   - Recommendation: The install script installs `libwebkit2gtk-4.1-dev` first; if clean install test on Ubuntu 22.04 fails, investigate PPA. The clean install checklist (D-13) will catch this.

2. **AppImage runtime behavior**
   - What we know: Issue #4313 is unfixed in Wails v2; the error is `WebKitNetworkProcess` path failure
   - What's unclear: Whether any runtime environment variable (`WEBKIT_DISABLE_COMPOSITING_MODE`, `WEBKIT_DISABLE_DMABUF_RENDERER`) mitigates the issue
   - Recommendation: Ship AppImage as-is; document known limitation; do not block release on AppImage functionality

3. **Fedora 40 `webkitgtk6.0` vs `webkit2gtk4.1`**
   - What we know: D-05 mentions both as alternatives; Fedora 40 packages both; `webkit2gtk4.1` is for GTK3+libsoup3 (what Wails uses)
   - What's unclear: Which is actually linked at runtime for the Wails binary
   - Recommendation: Use `webkit2gtk4.1` as the RPM dependency (confirmed as replacement for 4.0 in Fedora 40); the clean install test on Fedora 40 will validate

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| wails | Binary build | ✓ | v2.12.0 at `/home/sblanken/go/bin/wails` | — |
| go | wails build | ✓ | 1.24.11 | — |
| node/npm | wails frontend build | ✓ | node v24.12.0, npm 11.6.2 | — |
| sha256sum | Checksums | ✓ | GNU coreutils 9.4 | — |
| dpkg-deb | .deb inspection/testing | ✓ | system | — |
| nFPM | .deb / .rpm packaging | ✗ | — | `go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest` |
| gh CLI | GitHub release creation | ✗ | — | `sudo apt install gh` |
| appimagetool | AppImage build | ✗ | — | Download: `wget .../appimagetool-x86_64.AppImage` |
| ImageMagick convert | Icon resizing (1024→256) | ✗ (apt-installable) | — | `sudo apt install imagemagick` OR use Python PIL (available) |
| rpmbuild | RPM verification | ✗ | — | nFPM builds RPM without rpmbuild; verification skipped |

**Missing dependencies with no fallback:**
- None — all missing tools have install paths.

**Missing dependencies with fallback:**
- nFPM: Install via `go install` (Go 1.24 present) — fastest path
- gh CLI: Install via `sudo apt install gh` — required for release step
- appimagetool: Download AppImage from GitHub releases — no install needed
- ImageMagick: Install via `sudo apt install imagemagick` OR use Python PIL one-liner: `python3 -c "from PIL import Image; Image.open('build/appicon.png').resize((256,256)).save('build/linux/pairadmin.png')"`

**Icon resize PIL one-liner (available immediately, no install):**
```bash
python3 -c "from PIL import Image; img=Image.open('build/appicon.png'); img.resize((256,256), Image.LANCZOS).save('build/linux/pairadmin.png')"
```

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | vitest (configured in `frontend/vite.config.ts`) |
| Config file | `frontend/vite.config.ts` (test block inline) |
| Quick run command | `cd frontend && npm test -- --run` |
| Full suite command | `cd frontend && npm test -- --run` |

### Phase Requirements → Test Map

Distribution infrastructure (nFPM config, shell scripts, build assets) is not unit-testable in the traditional sense. These are configuration files and shell scripts verified by execution. Mapping:

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DIST-01 | `.deb` package produced with correct deps | smoke/manual | `dpkg --info pairadmin_*.deb \| grep libwebkit2gtk-4.1-0` | ❌ Wave 0 — build artifact |
| DIST-02 | AppImage produced (best-effort, documented limitation) | manual-only | N/A — visual/runtime check | ❌ Wave 0 — build artifact |
| DIST-03 | `.rpm` package produced with correct deps | smoke/manual | `rpm -qip pairadmin_*.rpm \| grep webkit2gtk4.1` | ❌ Wave 0 — build artifact |
| DIST-04 | `scripts/install-deps.sh` runs correctly on supported distros | manual (requires distro VM) | `bash scripts/install-deps.sh` on target distro | ❌ Wave 0 — file to create |

**Note on automation:** DIST-01 and DIST-03 can be partially validated with `dpkg --info` and `rpm -qip` commands locally after building packages. DIST-02 and DIST-04 require human verification per D-13.

The existing vitest suite (ThreeColumnLayout, ChatMessageList, etc.) is unchanged by this phase and remains green. No new frontend test files are needed.

### Sampling Rate
- **Per task commit:** Existing frontend test suite: `cd /home/sblanken/working/ppa2/frontend && npm test -- --run` (no new frontend code)
- **Per wave merge:** Same — no new automated tests added this phase
- **Phase gate:** Human verification checklist (D-13, D-14) before `/gsd:verify-work`

### Wave 0 Gaps
- None for automated tests — this phase produces build artifacts, not testable code modules.
- The verification plan (last plan) constitutes the test: install each artifact on target systems and run the acceptance checklist.

## Sources

### Primary (HIGH confidence)
- Wails v2.12.0 source at `/home/sblanken/go/pkg/mod/github.com/wailsapp/wails/v2@v2.12.0/pkg/commands/build/packager_linux.go` — confirmed no-op Linux packaging
- Wails v2.12.0 source — confirmed webkit build tag behavior (webkit2_41)
- `wails build --help` output — confirmed no AppImage flag in v2.12.0
- nFPM v2.46.0 docs at https://nfpm.goreleaser.com/docs/configuration/ — overrides syntax, depends format
- nFPM latest release confirmed via GitHub API: v2.46.0 (2026-03-31)
- AppImage docs at https://docs.appimage.org/packaging-guide/manual.html — AppDir structure, AppRun

### Secondary (MEDIUM confidence)
- GitHub Issue #4313 (https://github.com/wailsapp/wails/issues/4313) — WebKit subprocess path hardcoding; root cause confirmed by multiple reporters; no fix in v2
- Fedora Project Wiki on webkit2gtk-4.0 removal — `webkit2gtk4.1` confirmed as correct Fedora 40 package name
- nFPM install docs (https://nfpm.goreleaser.com/docs/install/) — install methods verified

### Tertiary (LOW confidence)
- WebSearch: AppImage icon/desktop entry mismatch warnings — mentioned in multiple GitHub issues; single-source but plausible pitfall

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — nFPM version confirmed via GitHub API; wails behavior confirmed from source code; appimagetool from official AppImage project
- Architecture: HIGH — nFPM config syntax verified from official docs; wails binary output confirmed from dry-run; AppImage structure from official docs
- Pitfalls: HIGH — Wails packaging no-op confirmed from source; Issue #4313 confirmed from GitHub; package naming confirmed from Fedora wiki

**Research date:** 2026-03-30
**Valid until:** 2026-06-30 (nFPM and AppImage are stable; Fedora package names are stable; Wails v2 is in maintenance mode)
