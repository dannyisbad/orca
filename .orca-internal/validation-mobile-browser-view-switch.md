# Validation: Mobile Browser Web/Mobile View Switch

**Worktree:** `allow-switching-to-monile-website`  
**Date:** 2026-06-20  
**Scope:** Validation plus follow-up fix for browser view-mode remount persistence

## Code Review

### Correctness — PASS

| Requirement | Status | Evidence |
|---|---|---|
| Default to Web view | PASS | `mobile-browser-view-mode-state.ts:6-15` returns `web` for browser pages without a saved mode |
| Web/Mobile switch on toolbar right | PASS | `MobileBrowserPane.tsx:1114-1118` — switch rendered after `flex:1` address input in toolbar row |
| Mobile mode resubscribes with phone viewport | PASS | `browser-screencast-request.ts:66-75` adds `viewportWidth/Height`, `deviceScaleFactor: 2`, `mobile: true`; `MobileBrowserPane.tsx:348-350` rebuilds `streamRequest` on mode change; stream effect (`378-545`) depends on `streamRequest` + `cacheKey` |
| Web mode leaves remote viewport unchanged | PASS | `browser-screencast-request.ts:51-76` omits viewport fields for `'web'`; desktop `browser-screencast-stream.ts:624-626` clears device metrics on stream stop |
| Separate frame cache per mode | PASS | `MobileBrowserPane.tsx:1302-1304` cache key includes `viewMode` |
| Zoom reset on mode switch | PASS | `MobileBrowserPane.tsx:1035-1043` |

### UX / Design — PASS

- Segmented control uses `colors`, `radii`, `spacing` from `mobile-theme` (`MobileBrowserViewModeSwitch.tsx:66-99`).
- Selected segment uses inverted `textPrimary` / `bgBase` contrast.
- Accessibility: per-segment `accessibilityLabel` and `accessibilityState.selected` (`MobileBrowserViewModeSwitch.tsx:57-59`).
- Toolbar icon buttons extracted to `MobileBrowserToolbarIconButton.tsx` (no behavior change).

### Issues Found

**None.** Implementation aligns with stated behavior and existing desktop screencast emulation contract.

### Minor Observations (non-blocking)

1. View mode now uses page-scoped in-memory state, so new browser pages default to `web` but a user-selected Web/Mobile mode survives normal pane remounts for the same `worktreeId` + `browserPageId`.
2. Switch is disabled while `controlsDisabled` (no client/page/screencast), but remains visible — reasonable affordance.

## Automated Checks

| Check | Result |
|---|---|
| `pnpm test src/browser/browser-screencast-request.test.ts src/browser/mobile-browser-pane-source.test.ts src/browser/mobile-browser-view-mode-state.test.ts` | **7/7 passed** |
| `pnpm exec oxlint` on changed browser files | **0 warnings, 0 errors** |
| `pnpm exec oxfmt --check` on changed browser files | **PASS** |
| `pnpm exec tsc --noEmit` (mobile) | **PASS** |

**Note:** `mobile/node_modules` was missing initially; installed via `pnpm install` to run tests (no source edits).

## Emulator Validation

### Environment

- Simulator: iPhone 17 Pro (booted), Orca emulator helper running (`orca emulator attach --worktree active`)
- Metro: started with `EXPO_NO_TELEMETRY=1 pnpm start --host lan` — `packager-status:running`
- Dev client URL opened via `xcrun simctl openurl booted`

### Results

| Check | Result |
|---|---|
| Bundle completes without redbox | **PASS** — `iOS Bundled 2396ms` (4156 modules) |
| App loads in simulator | **PASS** — Orca Mobile home screen renders |
| Navigate to browser tab | **PASS via Host 2** — target workspace opened and a `New Browser` tab was created from the mobile New Tab drawer |
| Web/Mobile control visible in browser toolbar | **PASS** — toolbar rendered with address input and Web/Mobile segmented control; new browser defaulted to `Web` |
| Toggle Mobile → resubscribe with viewport emulation | **BLOCKED** — connected Host 2 runtime did not advertise `browser.screencast.v1`, so `controlsDisabled` kept the switch disabled |
| Layout/redbox issues on changed screens | **NONE OBSERVED** on home, workspace list, new-tab drawer, New Browser sheet, or browser pane |

### Host / Pairing Notes

- The simulator stores host metadata under AsyncStorage key `orca:hosts`; pairing tokens are in SecureStore/Keychain, so editing only the JSON metadata is not enough to repair a mismatched host.
- Host 1 metadata now points at `ws://192.168.1.32:6768`, and local Orca is listening on `*:6768`, but the mobile app remains stuck at `Connecting...`. That means the endpoint is no longer connection-refused; the remaining failure is likely pairing/auth/E2EE token mismatch.
- Host 2 connects at `ws://192.168.1.32:57067`, but that process is `Orca: fix-mobile-terminal-wakeup`, not this worktree. It allowed workspace/browser-pane navigation using this worktree's Metro bundle on `8081`, but did not provide the browser screencast capability needed to exercise the mode switch.

### Screenshots (local)

- `/tmp/orca-mobile-ios-after-metro.png` — home screen, app loaded
- `/tmp/orca-mobile-ios-tap-host2.png` — Host 1 workspace list, loading spinner (desktop unreachable)
- `/tmp/orca-mobile-ios-new-tab-drawer-2.png` — New Tab drawer with Browser row selected in the follow-up run
- `/tmp/orca-mobile-ios-browser-blank-opened-2.png` — browser tab created; `Web` selected by default and `Mobile` visible
- `/tmp/orca-mobile-ios-browser-mobile-mode-2.png` — attempted Mobile toggle; switch remained disabled because screencast capability was unavailable

## Residual Risks

1. **Live toggle verification gap:** Web/Mobile segmented control placement/default state was verified in the emulator, but toggle-driven resubscribe behavior still needs a paired host whose `status.get` includes `browser.screencast.v1`.
2. **End-to-end emulation:** Desktop-side `Emulation.setDeviceMetricsOverride` + stream teardown on mode switch is correct in code, but not exercised live in this run.

## Verdict

**Code-level validation: PASS (no issues).**  
**Emulator validation: PARTIAL+** — app bundles, loads, navigates to the browser pane, and shows the Web/Mobile control defaulting to Web. The actual Mobile toggle remains blocked by local host pairing/capability state, not by the mobile UI implementation.
