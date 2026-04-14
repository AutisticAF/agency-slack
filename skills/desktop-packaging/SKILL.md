---
name: desktop-packaging
description: Electron Forge packaging in slack-desktop — platform-specific makers, code signing, notarization, webpack production builds, and security fuses.
---

# Desktop Packaging

slack-desktop uses Electron Forge for packaging, with platform-specific makers and a multi-stage code signing pipeline.

## Quick Reference

```bash
# Development build + launch
yarn start

# Package for current platform
yarn package

# Package with options
yarn package --override-arch=arm64
yarn package --dmg=true
yarn package --msi=true
yarn package --appx=true
yarn package --installers=false  # output folder only, no installers
```

## Forge Config (`forge.config.ts`)

The config is an async function returning `ForgeConfig`:

- **packagerConfig** — delegates to `getPackagerOpts()` in `build/make-package.ts`
- **rebuildConfig** — `{ disablePreGypCopy: true, mode: 'parallel' }`
- **makers** — Windows makers (dynamic), `@electron-forge/maker-dmg`, `@electron-forge/maker-zip`
- **plugins** — empty (webpack runs manually in `generateAssets` hook)
- **hooks** — `generateAssets`, `postPackage`, `postMake`

## Platform Packaging

### macOS

**Makers:** DMG (UDBZ format, 600x440 window, custom background) + ZIP (for darwin and mas)

**App bundle:**
- `appBundleId`: `com.tinyspeck.slackmacgap`
- `ElectronTeamID`: `BQR82RBBHL`
- `LSMinimumSystemVersion`: `12.0`
- URL protocol: `slack://`

**Icons:** `slack.icns` (prod), `slack-dogfood.icns` (dogfood), `slack-prototype.icns` (prototype branches)

**MAS builds:** Set `SLACK_DARWIN_PLATFORM=mas` env var.

### Windows

**x64 makers (4):**
1. **MSQ** — Standalone MSI via `electron-wix-msi`
2. **Winstaller** — Squirrel.Windows installer (`.exe` + `.msi` + `.nupkg` + RELEASES)
3. **MSIX DDL** — Direct download MSIX, signed via CI
4. **MSIX Store** — Windows Store AppX

**arm64 makers (2):** Only the two MSIX makers (no Squirrel or standalone MSI)

### Linux

No Forge makers. Custom `createDistroPackage()` from `build/linux-packaging.ts`.

## Code Signing

### macOS

Signing is done **after** Forge's package step (not via `osxSign`). This is intentional — post-package hooks add binaries that need signing.

**Flow:**
1. Ad-hoc `codesign --deep` for M1 compatibility (after fuse flipping invalidates Electron's signature)
2. `npm run codesign` runs:
   - `codesignMacApp()` via `@electron/osx-sign`
   - `codesignNodeFiles()` — signs all `.node` files individually
   - Optional `--notarize` triggers `notarizeApp()` via `@electron/notarize`

**Signing identities:**
- DDL: `Developer ID Application: SLACK TECHNOLOGIES L.L.C. (BQR82RBBHL)`
- MAS: `3rd Party Mac Developer Application: SLACK TECHNOLOGIES L.L.C. (BQR82RBBHL)`
- Dev: `Apple Development: Desktop Release (44TN467826)`

**DDL builds:** Hardened runtime enabled, custom designated requirements.
**MAS builds:** Per-entitlements signing (separate entitlement files for Plugin, GPU, Renderer, inherit).

### Windows

**Tool:** Jsign (Java-based) with AWS KMS as keystore.
- `KMS_REGION`: `us-east-1`
- `KMS_ALIAS`: `alias/code-signing`
- Timestamp: `http://timestamp.digicert.com`

**When signing happens:**
1. `postPackage` hook — scans and signs all unsigned `.dll`, `.node`, `.exe`
2. Winstaller — custom Jsign wrapper replaces `vendor/signtool.exe` for Squirrel releasify
3. MSIX (CI mode) — `windows-sign-hook.js` calls Jsign
4. MSQ — signs support binaries and final `.msi`

**Skip condition:** Prototype builds (version ending `.65535`) skip signing unless on `support-build/` branch.

## Webpack Production Build

Triggered by Forge's `generateAssets` hook calling `createCompiledResources()`.

**Architecture:**
- Parallelized via `jest-worker` threads
- `src/common` pre-bundled with esbuild before webpack runs
- Transpiler: `esbuild-loader` (target: `esnext`, JSX factory: `h` for Preact)
- Minifier: `EsbuildPlugin` (ES2022 target, replaces Terser)

**Build targets** (`webpack/configs/`):

| Target | Config | Process |
|--------|--------|---------|
| Boot | `boot/boot.ts` | Startup |
| Browser | `browser/main.ts` | Main process |
| Preload (default) | `preload/preload.default.ts` | Preload |
| Preload (main) | `preload/preload.main.ts` | Main window preload |
| Renderer (default) | `renderer/renderer.default.ts` | Renderer |
| Window Chrome | `renderer/windowchrome.ts` | Window chrome renderer |
| Utility | `utility/utility.default.ts` | Utility process |
| Vendor DLL | `vendor/vendor.dll.ts` | Pre-built vendor bundle |

**Post-build:** `.map` files moved from `dist/` to `dist_sourcemap/` (uploaded to S3 separately, not shipped).

## Security Fuses

Set in `build/hooks/common/flip-fuses.ts` via `@electron/fuses` after Electron binary extraction:

| Fuse | Value | Purpose |
|------|-------|---------|
| `RunAsNode` | **false** | Prevents using the app as a Node.js runtime |
| `EnableCookieEncryption` | **true** | Encrypts cookies at rest |
| `EnableNodeCliInspectArguments` | **false** | Disables `--inspect` in production |
| `EnableNodeOptionsEnvironmentVariable` | **false** | Ignores `NODE_OPTIONS` env var |
| `EnableEmbeddedAsarIntegrityValidation` | **true** (except Linux) | Validates ASAR integrity |
| `OnlyLoadAppFromAsar` | **true** | Prevents loading app from plain directories |
| `GrantFileProtocolExtraPrivileges` | **false** | Restricts `file://` protocol |
| `WasmTrapHandlers` | **true** | Enables Wasm trap handlers |

## Build Hooks

**afterCopy** (after app source copied):
1. `stripSourcemapUrlsHook` — strips sourcemap references
2. `deleteStrayFilesHook` — removes unnecessary files
3. `copyInLicenseFilesHook` — copies license files
4. `copyInSoundFilesHook` — copies sound assets
5. `copyInCRTFilesHook` — (Windows) C runtime files
6. `copyInVisualElementsFilesHook` — (Windows) visual elements manifest
7. `deleteUnsupportedLanguageFilesHook` — (Darwin) removes unsupported `.lproj` dirs
8. `resetUtimeHook` — (Darwin) resets timestamps for reproducible builds

**afterExtract:** `flipFusesHook` (see Security Fuses above)

**postPackage:** `signWindowsLibrariesHook` (signs unsigned binaries)

**postMake:** `copyOutArtifacts` + `maybeNotarize` (DMG only)

## Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `SLACK_OVERRIDE_ARCH` | Target architecture (`arm64` / `x64`) |
| `SLACK_DARWIN_PLATFORM` | Set to `mas` for Mac App Store builds |
| `BUILD_TYPE` | `dogfood` or `release` — changes icons and behavior |
| `BUILD_ONLY` | Skip notarization |
| `SLACK_ELECTRON_ZIP_DIR` | Custom local Electron zip |
| `WEBPACK_BUNDLE_ANALYZE` | Generate bundle analysis `.stats.json` |

## Key Files

| File | Purpose |
|------|---------|
| `forge.config.ts` | Top-level Forge config |
| `build/package-cli.ts` | Packaging CLI entry point |
| `build/make-package.ts` | Packager options, hooks, `createInitialPackages()` |
| `build/makers/win/index.ts` | Windows maker orchestration |
| `build/mac-code-signing.ts` | macOS codesign logic |
| `build/windows-code-signing.ts` | Windows Jsign + AWS KMS |
| `build/utils/apple-notarize.ts` | Apple notarization |
| `build/hooks/common/flip-fuses.ts` | Electron fuse configuration |
| `webpack/runner.ts` | Webpack build orchestrator |
| `webpack/constants/base-config.ts` | Shared webpack base config |
