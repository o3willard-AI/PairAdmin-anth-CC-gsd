# Phase 7: Distribution & Launch — Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 07-distribution-launch
**Areas discussed:** Release pipeline, Binary signing, Package contents, Install test scope

---

## Release pipeline

| Option | Description | Selected |
|--------|-------------|----------|
| Manual via gh CLI | Run gh release create with artifacts — no CI infrastructure needed, fits a one-person v1 launch. Can automate later. | ✓ |
| GitHub Actions workflow | Triggered on git tag push. Builds artifacts in CI, uploads to release. Requires secrets setup and runner config. | |
| Manual upload via GitHub UI | Build locally, upload binaries in the browser. Simplest for one-time launch but no reproducible process. | |

**User's choice:** Manual via gh CLI
**Notes:** Keeps distribution simple for v1. CI automation deferred to v2.

---

## Binary signing

| Option | Description | Selected |
|--------|-------------|----------|
| SHA-256 checksums only | sha256sum of each artifact written to SHA256SUMS file, uploaded with release. No GPG key management. | ✓ |
| GPG sign + checksums | Generate a GPG keypair, sign each artifact, publish public key. More setup, more trustworthy. | |
| Defer — no signing for v1 | Ship binaries without any signing or checksums. Fastest but reduces user trust. | |

**User's choice:** SHA-256 checksums only
**Notes:** GPG signing noted as planned for future releases, mentioned in README.

---

## Package contents

**Q: What should .deb/.rpm packages include?**

| Option | Description | Selected |
|--------|-------------|----------|
| Binary + .desktop + icon | App appears in GNOME/KDE application launcher with name and icon. Standard for desktop apps. | ✓ |
| Binary only | Minimal package — no launcher integration. Users must run from terminal. | |
| Binary + .desktop + icon + man page | Full Linux packaging with man page. Extra effort for v1. | |

**Q: Where should binary and .desktop be installed?**

| Option | Description | Selected |
|--------|-------------|----------|
| /usr/local/bin + /usr/share | Binary to /usr/local/bin/pairadmin, .desktop to /usr/share/applications/. Standard local install paths. | ✓ |
| /usr/bin + /usr/share | Binary to /usr/bin/pairadmin — system-wide paths, typical for distro-packaged apps. | |

**User's choices:** Binary + .desktop + icon; /usr/local/bin + /usr/share
**Notes:** App will appear in application launchers on GNOME and KDE.

---

## Install test scope

| Option | Description | Selected |
|--------|-------------|----------|
| Human checklist on real/VM systems | Written checklist: install .deb on Ubuntu 22.04 + 24.04, install .rpm on Fedora 40, verify app launches. | ✓ |
| Automated install test script | Shell script that installs in a container and checks for success signal. | |
| Best-effort: test one distro only | Only test Ubuntu 24.04, accept lower priority for others. | |

**User's choice:** Human checklist on real/VM systems
**Notes:** Planner should create a non-autonomous checkpoint plan for human install verification across all three distros.

---

*Generated: 2026-04-02*
