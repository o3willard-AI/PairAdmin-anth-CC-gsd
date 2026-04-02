# Phase 7: Distribution & Launch - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Installable Linux packages (.deb, .rpm, AppImage) and a clean public GitHub release for v1.0. Deliverables: nFPM packaging config, `build/linux/` assets (.desktop, icon), `scripts/install-deps.sh`, updated root README with install instructions, GitHub release with SHA-256 checksums, and a human checklist-verified clean install on Ubuntu 22.04, Ubuntu 24.04, and Fedora 40.

No new Go code or frontend features. Pure distribution infrastructure. All work is CLI/config/shell.

</domain>

<decisions>
## Implementation Decisions

### Release Pipeline

- **D-01:** Release creation is **manual via `gh` CLI** — `gh release create v1.0.0 --title "PairAdmin v1.0.0" ...` with artifact upload. No GitHub Actions workflow for v1. This keeps distribution simple and avoids CI infrastructure setup. Can automate in v2.
- **D-02:** Release tag format: `v1.0.0` (semver). Release assets attached: `pairadmin_1.0.0_linux_amd64.deb`, `pairadmin_1.0.0_linux_amd64.rpm`, `pairadmin_1.0.0_linux_amd64.AppImage`, `SHA256SUMS`.

### Binary Signing / Checksums

- **D-03:** Artifact verification is **SHA-256 checksums only** — generate `SHA256SUMS` file with `sha256sum pairadmin_*` output, upload alongside binaries. No GPG key for v1. Users can verify with `sha256sum --check SHA256SUMS`.
- **D-04:** No GPG signing in v1. Note in README that GPG signing is planned for future releases.

### Package Contents

- **D-05:** nFPM `.deb` and `.rpm` packages include:
  - Binary: `/usr/local/bin/pairadmin`
  - Desktop entry: `/usr/share/applications/pairadmin.desktop`
  - Icon: `/usr/share/icons/hicolor/256x256/apps/pairadmin.png` (use `build/appicon.png`)
  - Runtime deps declared in nFPM: `libwebkit2gtk-4.1-0` (Ubuntu), `webkit2gtk4.1` or `webkitgtk6.0` (Fedora), `at-spi2-core`
- **D-06:** Install paths: `/usr/local/bin/pairadmin` (binary) and `/usr/share/...` (assets). Not `/usr/bin` — consistent with locally-distributed apps, not system packages.
- **D-07:** `.desktop` file content:
  ```
  [Desktop Entry]
  Name=PairAdmin
  Comment=AI pair programming assistant for terminal workflows
  Exec=/usr/local/bin/pairadmin
  Icon=pairadmin
  Type=Application
  Categories=Development;Utility;
  Terminal=false
  ```
- **D-08:** nFPM config at root: `nfpm.yaml`. Produces both `.deb` and `.rpm` from the same config.

### AppImage

- **D-09:** AppImage built via `wails build -platform linux/amd64` native AppImage support (or `appimagetool` if Wails doesn't produce AppImage directly). The ROADMAP notes Wails Issue #4313 (webkit bundling limitation) — document this limitation and recommend `.deb` as the primary install path in README.
- **D-10:** AppImage is a best-effort artifact. If webkit bundling fails in AppImage, ship the `.deb` and `.rpm` and document the AppImage limitation clearly.

### Install Dependency Script

- **D-11:** `scripts/install-deps.sh` detects distro via `ID` in `/etc/os-release`:
  - Debian/Ubuntu (`ID=ubuntu` or `ID=debian`): `apt-get install -y libwebkit2gtk-4.1-dev at-spi2-core gcc` (build-time); `libwebkit2gtk-4.1-0 at-spi2-core` (runtime)
  - Fedora/RHEL (`ID=fedora` or `ID_LIKE=rhel`): `dnf install -y webkit2gtk4.1-devel at-spi2-atk-devel gcc` (build-time)
  - Exit with clear error on unsupported distros
- **D-12:** Script is idempotent — running twice does not cause errors.

### Clean Install Test

- **D-13:** Clean install verification is a **human checklist** — not automated. Planner should create a written checklist as part of a non-autonomous checkpoint plan:
  - Install `.deb` on Ubuntu 22.04 VM/container: `dpkg -i pairadmin_*.deb`
  - Install `.deb` on Ubuntu 24.04: `dpkg -i pairadmin_*.deb`
  - Install `.rpm` on Fedora 40: `rpm -i pairadmin_*.rpm`
  - Each: verify app launches, connects to Ollama (if available), shows terminal tabs, completes a chat interaction
  - Document pass/fail per distro in VERIFICATION notes
- **D-14:** The final acceptance criteria check (all v1 requirements) is also a human verification checkpoint — planner should include this as the last plan.

### README

- **D-15:** Update root `README.md` (or create if missing) with:
  - Installation section per package type: `.deb`, `.rpm`, AppImage
  - Prerequisite: Ollama setup or cloud API key
  - Note: AppImage webkit limitation and `.deb` as recommended path
  - Quick start section (launch, connect to tmux, send first message)

### Claude's Discretion

- nFPM version to use (latest stable)
- Whether to use `wails build --nsis` or nFPM for packaging (nFPM preferred per ROADMAP)
- Exact nFPM `version_schema` and changelog format
- AppImage toolchain choice (appimagetool vs wails native) — researcher should investigate
- Whether to create a `Makefile` with `make dist` target (helpful but not required)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` §DIST-01–04 — Acceptance criteria for this phase

### Roadmap
- `.planning/ROADMAP.md` §Phase 7 — Key deliverables and exit criteria (including Wails Issue #4313 reference for AppImage)

### Existing Build Assets
- `wails.json` — Current Wails project config; researcher should check for linux-specific fields
- `build/appicon.png` — App icon source for .desktop / package icon
- `build/darwin/` — Reference for how macOS build assets are structured (analogous Linux assets go in `build/linux/`)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `build/appicon.png` — 512×512 PNG icon; use as source for nFPM icon at `/usr/share/icons/hicolor/256x256/apps/pairadmin.png` (resize or symlink)
- `wails.json` — Existing build config; researcher should check if `wails build` natively supports `--package` or AppImage flags

### Established Patterns
- No existing Linux build/ directory — must create `build/linux/` with `.desktop` file and nFPM config
- No existing `scripts/` directory — must create

### Integration Points
- `wails build -platform linux/amd64` produces the binary at `build/bin/pairadmin`
- nFPM takes the binary from `build/bin/pairadmin` and packages it with assets from `build/linux/`
- `gh release create` uploads from local filesystem after build + nFPM run

</code_context>

<specifics>
## Specific Ideas

- nFPM config example for reference:
  ```yaml
  name: pairadmin
  arch: amd64
  platform: linux
  version: "${VERSION}"
  maintainer: PairAdmin Dev <sblanken@pairadmin.local>
  description: AI pair programming assistant for terminal workflows
  homepage: https://github.com/sblanken/pairadmin
  license: MIT
  depends:
    - libwebkit2gtk-4.1-0
    - at-spi2-core
  contents:
    - src: build/bin/pairadmin
      dst: /usr/local/bin/pairadmin
    - src: build/linux/pairadmin.desktop
      dst: /usr/share/applications/pairadmin.desktop
    - src: build/linux/pairadmin.png
      dst: /usr/share/icons/hicolor/256x256/apps/pairadmin.png
  ```
- SHA256SUMS generation: `sha256sum pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage > SHA256SUMS`
- `gh release create` command: `gh release create v1.0.0 pairadmin_*.deb pairadmin_*.rpm pairadmin_*.AppImage SHA256SUMS --title "PairAdmin v1.0.0" --notes-file RELEASE_NOTES.md`

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 07-distribution-launch*
*Context gathered: 2026-04-02*
