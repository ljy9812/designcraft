# DesignCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `designcraft-web` build — the same static site published as
`designcraft-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from
the HAP's own `rawfile`: no network origin, no server-side code.

This branch (`ohos-webshell`) is the integrated form: the ArkWeb shell
packaging under `packaging/ohos/` plus the wasm-side CJK font change
merged from the `cjk-web-face` branch (which remains the clean
single-purpose form for an upstream PR). Build either variant from this
one checkout.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.designcraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks — attached (document listener) and programmatic/detached (`HTMLAnchorElement.prototype.click` override) — are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` (with a filename field) |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### CJK (Chinese) variant — in this branch

`crates/fonts/build.rs` whitelists the Hans interface face for wasm builds — with CRAFT_FONTS_DIR set, the zh UI renders natively (the old English-forcing is retired).

Self-build recipe (Windows host, Rust stable, `trunk` 0.21.x; pin versions
per the upstream repo — on networks where the wasm-opt download hangs,
build with the optimizer off or assemble the .stage outputs by hand):

```powershell
git clone -b hans-wqy-microhei https://github.com/ljy9812/craft-fonts.git craft-fonts-cjk
$env:CRAFT_FONTS_DIR = "<repo>\craft-fonts-cjk"      # absolute path
cd apps\designcraft-web
trunk build --release
# deploy dist\web\ into packaging\ohos\entry\src\main\resources\rawfile\web\:
# swap the wasm+glue pair, copy snippets\ and fonts\ when the build produced
# them, and rewrite index.html's asset hash — a stale hash (or a stale SRI
# integrity attribute) double-fetches the wasm behind a white window.
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.designcraft.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23, physical keyboard +
touchscreen): install & launch, zh-Hans UI rendering (embedded Hans face),
element create/move/resize, save round-trip (`.designcraft` ZIP container,
byte-verified), and CJK input preserved byte-exact in `document.json`.

Known limitations:

- Canvas text elements render CJK as tofu although the face is embedded and
  the content bytes are intact — an application-layer font-routing issue
  (upstream fix needed, distinct from the missing-face family).
