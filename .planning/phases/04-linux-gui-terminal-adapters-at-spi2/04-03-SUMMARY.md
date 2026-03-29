---
phase: 04-linux-gui-terminal-adapters-at-spi2
plan: 03
subsystem: ui
tags: [at-spi2, dbus, golang, react, typescript, zustand, base-ui, tooltip, accessibility]

requires:
  - phase: 04-02
    provides: ATSPIAdapter with injectable fields and OnboardingRequired; GetAdapterStatus RPC

provides:
  - ATSPIAdapter.Discover probes GetText per pane and marks Degraded=true/DegradedMsg on failure
  - TerminalTab interface with optional degraded/degradedMsg fields
  - TerminalTab component with amber warning badge and @base-ui/react Tooltip
  - TerminalPreview extended empty state with AT-SPI2 onboarding instructions (D-06/D-07)
  - ThreeColumnLayout fetches GetAdapterStatus and passes adapterStatus down to TerminalPreview
  - CaptureManager.js wailsjs stub for vitest import resolution

affects: [04-04, phase-5-settings-config, verifier]

tech-stack:
  added: ["@base-ui/react Tooltip (already installed, now used in TerminalTab)"]
  patterns:
    - "GetText probe during Discover for graceful degradation without silent failure"
    - "adapterStatus prop threading: ThreeColumnLayout → TerminalPreview for onboarding UX"
    - "wailsjs stub pattern extended to go/services/capture/CaptureManager.js with .gitignore exception"
    - "vi.mock path from __tests__ subdir: ../../../../wailsjs/go/services/capture/CaptureManager"

key-files:
  created:
    - frontend/src/components/terminal/__tests__/TerminalPreview.test.tsx
    - frontend/src/components/terminal/__tests__/TerminalTab.test.tsx
    - frontend/wailsjs/go/services/capture/CaptureManager.js
  modified:
    - services/capture/atspi.go
    - services/capture/atspi_test.go
    - frontend/src/stores/terminalStore.ts
    - frontend/src/components/terminal/TerminalTab.tsx
    - frontend/src/components/terminal/TerminalPreview.tsx
    - frontend/src/components/layout/ThreeColumnLayout.tsx
    - frontend/src/components/__tests__/ThreeColumnLayout.test.tsx
    - .gitignore

key-decisions:
  - "GetText probe during Discover: probe each terminal object with getText during discovery to detect type mismatches (Konsole Qt5); mark Degraded=true rather than silently skipping"
  - "wailsjs CaptureManager stub: CaptureManager.js stub at frontend/wailsjs/go/services/capture/ added with .gitignore exception; enables ThreeColumnLayout test without vite:import-analysis error"
  - "vi.mock path depth: from frontend/src/components/__tests__/ to wailsjs stub requires 4 levels up (../../../../wailsjs/...)"
  - "adapterStatus prop threading chosen over local fetch in TerminalPreview: ThreeColumnLayout owns the Wails binding call; passes result as prop to keep TerminalPreview testable without dynamic import complexity"

patterns-established:
  - "Degraded probe pattern: call getText during Discover, not during Capture; prevents silent capture failure"
  - "TooltipTrigger asChild not used (STATE.md constraint): pass className directly on Tooltip.Trigger"
  - "Wails binding stubs for new Go bindings: create .js stub at wailsjs/go/services/{package}/ with .gitignore exception"

requirements-completed: [ATSPI-04]

duration: 6min
completed: 2026-03-29
---

# Phase 04 Plan 03: Konsole AT-SPI2 Spike and Frontend Onboarding UX Summary

**Konsole detection with GetText probe degradation; amber warning badge via @base-ui/react Tooltip; extended empty state with gsettings accessibility onboarding**

## Performance

- **Duration:** ~30 min (including human verification)
- **Started:** 2026-03-29T06:01:50Z
- **Completed:** 2026-03-29T06:12:00Z
- **Tasks:** 2 (Task 1 auto TDD, Task 2 human-verify checkpoint — approved)
- **Files modified:** 11

## Accomplishments

- ATSPIAdapter.Discover now probes GetText on each discovered terminal object during discovery; marks `Degraded=true` and `DegradedMsg="Konsole text extraction not available on this system."` if the probe fails — avoids silent capture failure (D-05)
- TerminalTab component shows an amber `⚠` warning badge with @base-ui/react Tooltip when `tab.degraded` is true; tooltip content comes from `degradedMsg`
- TerminalPreview empty state extended per D-06/D-07: shows "No terminal sessions detected." with Option 1 (tmux) always visible and Option 2 (gsettings toolkit-accessibility) shown conditionally when AT-SPI2 adapter has `status="onboarding"`
- ThreeColumnLayout fetches `GetAdapterStatus` from Go on mount via dynamic Wails import and threads `adapterStatus` down to TerminalPreview
- 12 backend tests pass (2 new: KonsoleDegradation + KonsoleSuccess); 68 frontend tests pass (6 new across TerminalPreview and TerminalTab test files)

## Task Commits

Each task was committed atomically:

1. **Task 1 RED: Failing tests for Konsole degradation + AT-SPI2 onboarding UX** - `571a41b` (test)
2. **Task 1 GREEN: Implementation** - `57432fc` (feat)
3. **Task 2: Human-verify checkpoint** - approved (no code commit)

## Files Created/Modified

- `services/capture/atspi.go` - Added GetText probe in Discover loop; sets Degraded/DegradedMsg on failure
- `services/capture/atspi_test.go` - Added TestATSPIAdapter_KonsoleDegradation and TestATSPIAdapter_KonsoleSuccess
- `frontend/src/stores/terminalStore.ts` - TerminalTab interface extended with degraded?/degradedMsg?; addTab accepts optional degraded args
- `frontend/src/components/terminal/TerminalTab.tsx` - Amber badge + @base-ui/react Tooltip for degraded tabs
- `frontend/src/components/terminal/TerminalPreview.tsx` - Extended empty state with adapterStatus prop and AT-SPI2 onboarding
- `frontend/src/components/layout/ThreeColumnLayout.tsx` - Fetches GetAdapterStatus on mount; passes adapterStatus to TerminalPreview
- `frontend/src/components/__tests__/ThreeColumnLayout.test.tsx` - Added vi.mock for CaptureManager binding
- `frontend/src/components/terminal/__tests__/TerminalPreview.test.tsx` - Created; tests onboarding conditional rendering
- `frontend/src/components/terminal/__tests__/TerminalTab.test.tsx` - Created; tests degraded badge presence
- `frontend/wailsjs/go/services/capture/CaptureManager.js` - Stub for vitest import resolution
- `.gitignore` - Added exception for CaptureManager.js stub

## Decisions Made

- **GetText probe during Discover**: Called `getText` on each discovered terminal object during Discover (not during Capture). This ensures the degraded flag is set at discovery time so the tab shows the warning badge immediately, before any capture attempt.
- **wailsjs CaptureManager stub**: ThreeColumnLayout uses a dynamic import of the Wails-generated CaptureManager binding. Like the runtime stub, a `.js` stub file was added to `frontend/wailsjs/go/services/capture/` with a `.gitignore` exception so vitest can resolve the import during tests.
- **adapterStatus prop threading**: GetAdapterStatus is called in ThreeColumnLayout (the component that owns Wails binding calls) and passed as a prop to TerminalPreview. This keeps TerminalPreview simpler and testable without needing to mock dynamic imports in its own tests.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed incorrect wailsjs import path depth in ThreeColumnLayout**
- **Found during:** Task 1 (implementing GetAdapterStatus call)
- **Issue:** Initial import used `../../wailsjs/...` (2 levels up from `src/components/layout/`) but the correct path from that directory is `../../../wailsjs/...` (3 levels up to reach `frontend/`)
- **Fix:** Corrected path to `../../../wailsjs/go/services/capture/CaptureManager`; correspondingly updated test mock to use `../../../../wailsjs/...` (4 levels from `__tests__/` subdirectory)
- **Files modified:** ThreeColumnLayout.tsx, ThreeColumnLayout.test.tsx
- **Verification:** All 68 frontend tests pass
- **Committed in:** 57432fc (feat commit)

**2. [Rule 3 - Blocking] Fixed TerminalTab test using getByRole("button") with multiple buttons**
- **Found during:** Task 1 (running GREEN tests)
- **Issue:** With @base-ui/react Tooltip.Trigger rendered inside the outer `<button>`, there were two buttons in the DOM — `getByRole("button")` threw "multiple elements found" error
- **Fix:** Changed test to use `screen.getByText("⚠")` to directly query the warning icon
- **Files modified:** TerminalTab.test.tsx
- **Verification:** TerminalTab tests pass with specific warning icon query
- **Committed in:** 57432fc (feat commit)

---

**Total deviations:** 2 auto-fixed (2 blocking)
**Impact on plan:** Both fixes required for correct test execution. No scope creep.

## Issues Encountered

- **node_modules symlink required**: Worktree doesn't have its own `node_modules`; created symlink `frontend/node_modules` → `/home/sblanken/working/ppa2/frontend/node_modules` to allow vitest to run from the worktree. This is a standard worktree setup issue.

## Known Stubs

None — all data flows are wired. The `adapterStatus` state starts as `[]` (empty array) until the Wails runtime calls `GetAdapterStatus()`. The conditional rendering of the AT-SPI2 onboarding section correctly handles the empty array case (no onboarding shown) without stubs.

## Next Phase Readiness

- Human verification completed and approved: empty state with AT-SPI2 onboarding, degraded tab badge, and live capture UX confirmed correct
- AT-SPI2 ATSPI-04 requirement fully satisfied (code + human verification)
- Phase 04 Plan 04 (`/filter` slash commands + Viper config) is unblocked and ready to proceed

## Self-Check: PASSED

All 10 claimed files exist. Both commits verified: 571a41b (test RED), 57432fc (feat GREEN). Human-verify checkpoint (Task 2) approved. 68 frontend tests and Go capture tests confirmed green post-approval.

---
*Phase: 04-linux-gui-terminal-adapters-at-spi2*
*Completed: 2026-03-29*
