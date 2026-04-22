---
name: adobe-plugin-development
description: Use when building, modifying, or debugging Adobe Photoshop UXP plugins or Lightroom Classic plugins — panel UI, pixel access, develop-module observation, memory management in long-lived LrC processes, hybrid native bridges, UXP HTML/CSS/Canvas limits, and Spectrum widgets
version: 1.0.0
---

# Adobe Plugin Development

Reference for writing Photoshop UXP plugins (JavaScript/HTML) and Lightroom Classic plugins (Lua). Covers what the SDKs allow, what they don't, and the non-obvious pitfalls that burn hours.

Baseline: UXP 9.2.0 / Manifest v5 / Photoshop 27.4. LrC 15.3 SDK / Lua 5.1.5.

## When to Use

Use when:
- Writing or editing a UXP panel/command plugin (`manifest.json`, `entrypoints.setup`, Spectrum `sp-*` widgets)
- Writing or editing a Lightroom Classic plugin (`Info.lua`, `LrView`, `LrDevelopController`)
- Reading pixel data from Photoshop (`imaging.getPixels`)
- Wiring develop-slider observation in LrC
- Displaying generated images inside a UXP panel (no HTML canvas pixel ops!)
- Bridging LrC to a native binary for pixel processing
- Debugging memory leaks in long-lived LrC floating dialogs
- Using batchPlay for Photoshop action descriptors
- Manipulating Document/Layer DOM (create, modify, filter, composite)
- Recording or replaying Photoshop actions programmatically

Don't use for:
- General web UI — UXP is NOT a browser; different rules apply
- ExtendScript / CEP (legacy Adobe plugin systems) — this covers UXP only
- Lightroom Cloud REST APIs — those are separate from the LrC desktop SDK

## Core Capability Matrix

| Capability | Photoshop UXP | Lightroom Classic |
|---|---|---|
| Language | JavaScript (HTML/CSS) | Lua 5.1.5 |
| Pixel read | `imaging.getPixels()` | **None** — must export rendition + decode externally |
| Pixel write | `imaging.putPixels()` | None |
| UI | HTML/CSS/JS + Spectrum `sp-*` | `LrView` declarative widgets only |
| Canvas/drawing | Limited 2D (no `drawImage`/`getImageData`/`toDataURL`) | None |
| Document DOM | Full: 33 Document props + 29 methods, 28 Layer props + 55 methods (38 filter + 17 other) | Catalog-centric: `LrCatalog`, `LrPhoto` |
| Selection API | Full `Selection` class (v25.0+) | `LrSelection` namespace (rating, flag, label, nav) |
| AI features | `generativeUpscale` (v27.2+) | denoise, reflection removal, distraction detection |
| Develop events | `action.addNotificationListener` | `LrDevelopController.addAdjustmentChangeObserver` |
| Plugin types | Panel, Command | Export, Publish, Metadata, Filter, Library menu |
| Develop-panel UI | Panels dock anywhere | **Cannot** add Develop-right-panel UI — only floating dialog |
| Action recording | `recordAction` API (v25.0+) | Not available |

**Fundamental constraint:** LrC cannot read pixels from Lua. Any LrC plugin needing pixel access must export a thumbnail or rendition and hand it to a compiled external binary (C++/Rust).

## Critical Pitfalls

### UXP: `<canvas>` has no pixel ops
- `<canvas>` exists (v7.0.0+) but has **no** `drawImage`, `getImageData`, `putImageData`, `toDataURL`, `toBlob`.
- To display a generated image: build pixel buffer -> `imaging.createImageDataFromBuffer` -> `imaging.encodeImageData({ base64: true })` -> set `<img src="data:image/jpeg;base64,...">`.
- Windows extra sharp edges: `createLinearGradient`, `createRadialGradient`, `clearRect` may fail entirely.
- Real-render options: software renderer in JS, embedded WebView, or hybrid C++ plugin.

### UXP: Imaging API gotchas
- 16-bit range is **0-32768 by default**, not 0-65535. Pass `{ fullRange: true }` to `getData()` for full range.
- **Always call `imageData.dispose()` after `getData()`**. UDT warns at 600MB.
- Pass `targetSize` to leverage Photoshop's pyramid cache — dramatically faster than full-res reads.
- `encodeImageData` only supports **RGB** (not Lab, not Grayscale).
- `getPixels()` usually works without `executeAsModal`, but any pixel *write* requires it.
- Default layout is "chunky" (interleaved `RGBRGB`); planar is `RRGGBB`.

### UXP: Layout and CSS limits
- No CSS Grid, no `float`, no `transition`/`@keyframes`, no `text-transform`, no `font` shorthand, no `position: sticky`.
- `box-shadow`, `transform-origin`, `scaleX`/`scaleY`, `translate` **require** the `CSSNextSupport` feature flag in manifest. Without it, CSS silently ignores these properties.
- `window.devicePixelRatio` always returns 1 — cannot detect HiDPI.
- `hide` panel lifecycle callback never fires; `show` fires only once (PS-57284). Don't rely on them for refresh gating.
- `<label for="id">` doesn't work — wrap the label around the control instead.
- `<option>` needs an explicit `value` attribute.

### UXP: CSSNextSupport feature flag
Enable in manifest.json to unlock additional CSS properties:
```json
{
  "featureFlags": { "CSSNextSupport": true }
}
```
Unlocks: `box-shadow`, `transform-origin`, `scaleX`, `scaleY`, `translate`. Without this flag these properties are silently ignored — no error, no warning.

### UXP: localStorage and sessionStorage
Both are available as global storage APIs — contrary to some older references listing them as unavailable. However, they are **not** available inside WebView when loading local content.

### UXP: WebView availability
WebView works in **panels** since UXP 6.4 / PS 24.1 (not just modal dialogs). Local HTML files supported since UXP 8.0. Domain restrictions optional since UXP 9.0. The `permissions.webview.allow` field was **removed in UXP 9.1** (PS 27.4) — configure with `domains` only.

### UXP: executeAsModal anti-patterns
- **Never swallow exceptions**: `try { await batchPlay(...) } catch(e) {}` prevents automatic cancellation termination. Let exceptions propagate.
- `timeOut` option (v25.10): modal collisions now retry for `timeOut` duration instead of immediate error.
- `updateUI()` (v26.0) has no effect outside tracking contexts (slider handlers).

### UXP: batchPlay error handling
batchPlay execution errors **resolve** (not reject) with error objects in the result array. Always check return values:
```javascript
const results = await action.batchPlay([descriptor], {});
if (results[0].message) {
  // error occurred — results[0].message contains description
}
```

### LrC: Long-running plugins leak catastrophically
A floating dialog running for hours in LrC's sandbox is unforgiving. Non-negotiable rules:

1. **Frame alternation is mandatory.** `f:picture` must receive a *different* path each update. Writing the same path repeatedly causes LrC to cache every version internally without releasing — this caused a 40GB leak in chromascope. Alternate `scope_0.jpg` / `scope_1.jpg`.
2. **Guard `requestJpegThumbnail` callbacks.** After `done = true`, subsequent callbacks must return immediately. Nil out `jpegData` after writing. **Must hold reference** to the returned request object or it may be garbage collected.
3. **Debounce async tasks with a version counter.** Slider drags spawn hundreds of coroutines without debouncing. Use a `_settleVersion` / `_adjustVersion` counter and drop stale callbacks.
4. **Busy-guard + pending flag coalescing.** If a render is in progress, set a pending flag instead of spawning another. When render completes, check pending and re-run once.
5. **No unbounded module-level state.** Only fixed-size vars: busy flag, frame index, pending flag, settings hash. Never grow a list.
6. **Clean up temp files on dialog open.** LrC sandbox blocks `collectgarbage` — there's no GC escape hatch.

### LrC: Develop module can't have custom panels
LrC SDK exposes Library menu items, export dialogs, metadata fields — **never** a Develop-right-panel UI. The only realistic develop-time feedback loop is a floating dialog (`LrDialogs.presentFloatingDialog`) launched from a Library menu item.

### LrC: No "active photo changed" observer
There's `addAdjustmentChangeObserver` for slider changes, but no event for switching active photo. Poll `catalog:getTargetPhoto()` in a `LrTasks.startAsyncTask` loop with coalescing.

### LrC: requestJpegThumbnail reference retention
Must hold a reference to the returned request object. If the return value is not stored, Lua's garbage collector may collect it and the callback never fires.

### LrC: getDevelopSettings has typos in official API
Known typos in parameter names returned by `getDevelopSettings`: `HueAdjustmentMagenha` (should be Magenta), `LuminanceAdjustmentAque` (should be Aqua), `Parametriclights` (inconsistent casing). Use exact typo'd names when reading these values.

### LrC: processRenderedPhotos task context
`processRenderedPhotos` runs in a task LrC creates — do **not** create your own `LrTasks.startAsyncTask` inside it. Doing so nests tasks incorrectly.

### LrC: LrDialogs unavailable during shutdown
The `LrDialogs` namespace is **not** available during `LrShutdownApp`. Do not attempt to show dialogs in shutdown scripts.

### LrC: withWriteAccessDo undo coalescing (Mac)
On Mac, successive `withWriteAccessDo` calls without user interaction coalesce into a single undo event. Design accordingly if undo granularity matters.

## UXP Quick Reference

### Read active document pixels
```javascript
const { core, app, imaging } = require('photoshop');

await core.executeAsModal(async () => {
  const doc = app.activeDocument;
  const result = await imaging.getPixels({
    documentID: doc.id,
    targetSize: { width: 256, height: 256 },  // pyramid cache
    colorSpace: "RGB",
    componentSize: 8
  });
  const pixels = await result.imageData.getData(); // Uint8Array, interleaved RGB
  // ... process ...
  result.imageData.dispose();  // mandatory
}, { commandName: "Analyze" });
```

### Display a software-rendered image
```javascript
const imageData = await imaging.createImageDataFromBuffer(buffer, {
  width, height, components: 3, colorSpace: "RGB"
});
const jpegBase64 = await imaging.encodeImageData({ imageData, base64: true });
img.src = "data:image/jpeg;base64," + jpegBase64;
imageData.dispose();
```

### Document and layer operations
```javascript
const doc = app.activeDocument;
await core.executeAsModal(async (ctx) => {
  await doc.suspendHistory(async () => {
    const layer = await doc.createLayer();
    await layer.applyGaussianBlur(5.0);
  }, "My Edit");
}, { commandName: "Edit Layer" });
```

### batchPlay basics
```javascript
const { action } = require('photoshop');
const result = await action.batchPlay([{
  _obj: "get",
  _target: [{ _property: "opacity" },
            { _ref: "layer", _enum: "ordinal", _value: "targetEnum" },
            { _ref: "document", _enum: "ordinal", _value: "targetEnum" }],
  _options: { dialogOptions: "silent" }
}], {});
// 5 reference forms: ID (_id), Index (_index), Name (_name), Enum (_enum/_value), Property (_property, bare — no _ref)
```

**multiGet for bulk reads** (more efficient than multiple get calls):
```javascript
const result = await action.batchPlay([{
  _obj: "multiGet",
  _target: [{ _ref: "document", _enum: "ordinal", _value: "targetEnum" }],
  extendedReference: [["width", "height", "resolution", "mode"]]
}], {});
```

**Discovery:** Use Photoshop's "Copy As JavaScript" from the Actions panel to capture descriptors. Use `action.addNotificationListener` to observe descriptors fired by manual operations.

### Spectrum UXP component catalog

| Component | Key attributes / events |
|---|---|
| `sp-action-button` | `quiet`, `selected`, `disabled`, `click` |
| `sp-action-group` | `compact`, `justified`, `quiet` |
| `sp-action-menu` | `placement`, `quiet`, `change` — menu with `sp-menu-item` children |
| `sp-body` | Typography body text |
| `sp-button` | `variant` (cta/primary/secondary/warning), `quiet`, `disabled`, `click` |
| `sp-button-group` | Container for button sets |
| `sp-checkbox` | `checked`, `indeterminate`, `disabled`, `change` |
| `sp-detail` | Typography detail text |
| `sp-divider` | `size` (small/medium/large) |
| `sp-dropdown` | `placeholder`, `quiet`, `change` — wraps `sp-menu` |
| `sp-heading` | Typography heading text |
| `sp-icon` | `name`, `size` (s/m/l/xl/xxl) — **40 built-in icons** |
| `sp-label` | Typography label text |
| `sp-link` | `href`, `quiet` |
| `sp-menu` | Wraps `sp-menu-item` and `sp-menu-divider` |
| `sp-menu-item` | `selected`, `disabled` |
| `sp-radio` | `checked`, `disabled`, `change` |
| `sp-radio-group` | `selected`, `change` |
| `sp-slider` | `min`, `max`, `value`, `step`, `disabled`, `input`/`change` |
| `sp-textfield` | `placeholder`, `quiet`, `type`, `disabled`, `input`/`change` |
| `sp-textarea` | `placeholder`, `quiet`, `disabled`, `input`/`change` |

**SWC (Spectrum Web Components):** 35 npm packages for advanced Spectrum UI. Requires `enableSWCSupport: true` in manifest. Locked to v0.37.0 in UXP 8.0. Provides components not available as `sp-*` (color picker, toast, accordion, tabs, etc.).

### CSS support summary

**Supported:** flexbox (all properties), box model (margin, padding, border, width/height, overflow), typography (font-family, font-size, font-weight, line-height, text-align, color, text-decoration, white-space, word-wrap), backgrounds (background-color, background-image), opacity, `calc()`, CSS custom properties (`--var`), `linear-gradient` (v8.1+).

**With CSSNextSupport flag:** `box-shadow`, `transform-origin`, `scaleX`, `scaleY`, `translate`.

**NOT supported:** Grid, `transition`, `animation`/`@keyframes`, `float`, `text-transform`, `font` shorthand, `position: sticky`, `::before`/`::after` (partial), `:nth-child` (partial).

**Theme CSS variables** (auto-update on theme change):
`--uxp-host-background-color`, `--uxp-host-text-color`, `--uxp-host-border-color`, `--uxp-host-link-text-color`, `--uxp-host-widget-hover-background-color`, `--uxp-host-widget-hover-border-color`. Font sizes: `--uxp-host-font-size`, `--uxp-host-font-size-smaller`, `--uxp-host-font-size-larger`. Also supports `@media (prefers-color-scheme: dark|light)`. Spectrum `sp-*` widgets auto-theme without extra CSS.

### Panel lifecycle and entrypoints
```javascript
const { entrypoints } = require("uxp");
entrypoints.setup({
  plugin: { create() {}, destroy() {} },
  panels: {
    myPanel: {
      create() { /* DOM ready — set up listeners */ },
      show() { /* fires once, not on every show (PS-57284) */ },
      hide() { /* never fires (PS-57284) */ },
      destroy() { /* cleanup */ },
      menuItems: [
        { id: "reload", label: "Reload", enabled: true,
          checked: false, oninvoke: () => refresh() }
      ]
    }
  },
  commands: {
    myCommand: { run() { /* one-shot execution */ } }
  }
});
```
- Panel `show`/`hide` are unreliable (PS-57284). Use `document.visibilitychange` if available.
- 300ms timeout on panel `create` — defer heavy init with `setTimeout`.

### Manifest v5 essentials
```json
{
  "manifestVersion": 5,
  "id": "com.example.myplugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "main": "index.html",
  "host": { "app": "PS", "minVersion": "24.0.0",
            "data": { "apiVersion": 2, "loadEvent": "use" } },
  "requiredPermissions": {
    "localFileSystem": "fullAccess",
    "network": { "domains": ["https://api.example.com"] },
    "clipboard": "readAndWrite",
    "webview": { "domains": ["https://example.com"] }
  },
  "featureFlags": { "enableSWCSupport": true, "CSSNextSupport": true }
}
```
- `apiVersion: 2` required for modern imaging API
- `loadEvent: "use"` (lazy) vs `"startup"` (eager)
- `enableMenuRecording: true` — enables action recording support

### Event system
```javascript
const { action, core } = require('photoshop');
await action.addNotificationListener(
  ['set', 'select', 'make', 'delete', 'open'],
  (name, desc) => scheduleRefresh()
);
// ~219 action events available
// Prefer userIdle for heavy re-renders:
await core.addNotificationListener('UI', ['userIdle'], refresh);
```

### executeAsModal
```javascript
await core.executeAsModal(async (executionContext) => {
  // executionContext provides: isCancelled, onCancel, reportProgress
  // hostControl provides: suspendHistory, resumeHistory,
  //   registerAutoCloseDocument, unregisterAutoCloseDocument
  const hostControl = executionContext.hostControl;
  const suspensionID = await hostControl.suspendHistory({ documentID: doc.id, name: "My Op" });
  // ... operations ...
  await hostControl.resumeHistory(suspensionID);
}, {
  commandName: "My Operation",
  // interactive: true,   // allows UI interaction during modal
  // timeOut: 5000,        // v25.10: retry duration for modal collisions (ms)
});
```
- PS only creates a history state if document was actually modified
- Never swallow exceptions — prevents cancel propagation
- Nested `executeAsModal` calls are allowed

## LrC Quick Reference

### LrC plugin lifecycle
```
Info.lua → LrInitPlugin → on-demand script loading (or LrForceInitPlugin at startup)
```
- `LrShutdownApp` returns `{ LrShutdownFunction = function(doneFunc, progressFunc) }` with 10-second timeout
- `LrDialogs` is NOT available during `LrShutdownApp`
- Installation paths:
  - macOS: `~/Library/Application Support/Adobe/Lightroom/Modules/`
  - Windows: `%APPDATA%\Adobe\Lightroom\Modules\`
- Auto-loaded plugins can't be removed, only disabled
- Debugging: `LrLogger` with `enable('print')` or `enable('logfile')`
  - Mac logs: `~/Library/Logs/Adobe/Lightroom/LrClassicLogs/`
  - Windows logs: `%LOCALAPPDATA%\Adobe\Lightroom\Logs\LrClassicLogs\`

### Observe develop-slider changes
```lua
LrFunctionContext.callWithContext("observer", function(context)
  LrDevelopController.addAdjustmentChangeObserver(context, {}, function()
    scheduleRerender()
  end)
end)
```

### Export thumbnail and pipe to external tool
```lua
local done = false
local request = photo:requestJpegThumbnail(w, h, function(jpegData, errorMsg)
  if done then return end          -- mandatory guard
  if not jpegData then return end
  done = true
  LrFileUtils.writeFile(tempJpegPath, jpegData)
  jpegData = nil                    -- release reference
  LrTasks.execute(binaryPath .. " decode --input " .. tempJpegPath .. " ...")
end)
-- Must hold `request` reference or callback may never fire (GC)
```

### Live-updating picture display (alternating paths)
```lua
local frameIndex = 0
local function nextScopePath()
  frameIndex = (frameIndex + 1) % 2
  return scopeDir .. "/scope_" .. frameIndex .. ".jpg"
end

viewFactory:picture { value = LrView.bind("scopePath"), width = 256, height = 256 }
-- On update: properties.scopePath = nextScopePath() then write the file there
```

### Debounced async rerender with busy-guard
```lua
local _settleVersion = 0
local _busy = false
local _pending = false

local function scheduleRerender()
  _settleVersion = _settleVersion + 1
  local myVersion = _settleVersion
  LrTasks.startAsyncTask(function()
    LrTasks.sleep(0.1)                 -- settle window
    if myVersion ~= _settleVersion then return end  -- superseded
    if _busy then _pending = true; return end       -- coalesce
    _busy = true
    -- ... export + render ...
    _busy = false
    if _pending then _pending = false; scheduleRerender() end
  end)
end
```

### Info.lua manifest fields

| Field | Purpose |
|---|---|
| `LrSdkVersion` | Target SDK version (required) |
| `LrSdkMinimumVersion` | Minimum supported SDK version |
| `LrToolkitIdentifier` | Unique reverse-domain ID (required) |
| `LrPluginName` | Display name (required, SDK 2.0+) |
| `LrLibraryMenuItems` | Library > Plug-in Extras menu items |
| `LrExportMenuItems` | File > Plug-in Extras menu items |
| `LrHelpMenuItems` | Help > Plug-in Extras menu items |
| `LrExportServiceProvider` | Export/publish service definitions |
| `LrExportFilterProvider` | Export filter (post-process action) definitions |
| `LrMetadataProvider` | Custom metadata definition script |
| `LrMetadataTagsetFactory` | Custom metadata tagset(s) |
| `LrPluginInfoProvider` | Plugin Manager dialog customization |
| `LrPluginInfoUrl` | URL shown in Plugin Manager |
| `LrInitPlugin` | Script run when plugin loads |
| `LrForceInitPlugin` | Force init at app startup (SDK 4.0+) |
| `LrEnablePlugin` / `LrDisablePlugin` | Enable/disable scripts (SDK 3.0+) |
| `LrShutdownPlugin` | Script on user unload (SDK 3.0+) |
| `LrShutdownApp` | Script on app exit (SDK 4.0+) — LrDialogs NOT available |
| `URLHandler` | Custom `lightroom://` URL handling (SDK 4.0+) |
| `LrAlsoUseBuiltInTranslations` | Use built-in LOC translations |
| `LrLimitNumberOfTempRenditions` | Max concurrent temp renditions during export |
| `VERSION` | `{ major, minor, revision, build }` |

### Data binding patterns
```lua
-- Basic binding
f:edit_field { value = LrView.bind("myKey") }

-- Transform binding (display conversion)
f:static_text { title = LrView.bind { key = "tempC", transform = function(value)
  return string.format("%.1f F", value * 9/5 + 32)
end } }

-- Conditional enablement
f:push_button { enabled = LrBinding.keyEquals("mode", "advanced") }

-- Cross-table binding
f:edit_field { value = LrView.bind { bind_to_object = otherTable, key = "sharedKey" } }

-- Observer pattern (guard flag prevents loops)
local updating = false
props:addObserver("inputKey", function(t, k, v)
  if updating then return end
  updating = true
  t.outputKey = v * 2
  updating = false
end)
```

### Complete develop parameters (by panel)

adjustPanel: `Temperature`, `Tint`, `Exposure`, `Highlights`, `Shadows`, `Brightness`, `Contrast`, `Whites`, `Blacks`, `Texture`, `Clarity`, `Dehaze`, `Vibrance`, `Saturation`, `PresetAmount`, `ProfileAmount`

tonePanel: `ParametricDarks`, `ParametricLights`, `ParametricShadows`, `ParametricHighlights`, `ParametricShadowSplit`, `ParametricMidtoneSplit`, `ParametricHighlightSplit`, `ToneCurve`, `ToneCurvePV2012`, `ToneCurvePV2012Red`, `ToneCurvePV2012Blue`, `ToneCurvePV2012Green`, `CurveRefineSaturation`

mixerPanel: `SaturationAdjustmentRed`, `SaturationAdjustmentOrange`, `SaturationAdjustmentYellow`, `SaturationAdjustmentGreen`, `SaturationAdjustmentAqua`, `SaturationAdjustmentBlue`, `SaturationAdjustmentPurple`, `SaturationAdjustmentMagenta`, `HueAdjustmentRed`, `HueAdjustmentOrange`, `HueAdjustmentYellow`, `HueAdjustmentGreen`, `HueAdjustmentAqua`, `HueAdjustmentBlue`, `HueAdjustmentPurple`, `HueAdjustmentMagenta`, `LuminanceAdjustmentRed`, `LuminanceAdjustmentOrange`, `LuminanceAdjustmentYellow`, `LuminanceAdjustmentGreen`, `LuminanceAdjustmentAqua`, `LuminanceAdjustmentBlue`, `LuminanceAdjustmentPurple`, `LuminanceAdjustmentMagenta`, `PointColors`, `GrayMixerRed`, `GrayMixerOrange`, `GrayMixerYellow`, `GrayMixerGreen`, `GrayMixerAqua`, `GrayMixerBlue`, `GrayMixerPurple`, `GrayMixerMagenta`

colorGradingPanel: `SplitToningShadowHue`, `SplitToningShadowSaturation`, `ColorGradeShadowLum`, `SplitToningHighlightHue`, `SplitToningHighlightSaturation`, `ColorGradeHighlightLum`, `ColorGradeMidtoneHue`, `ColorGradeMidtoneSat`, `ColorGradeMidtoneLum`, `ColorGradeGlobalHue`, `ColorGradeGlobalSat`, `ColorGradeGlobalLum`, `SplitToningBalance`, `ColorGradeBlending`

detailPanel: `Sharpness`, `SharpenRadius`, `SharpenDetail`, `SharpenEdgeMasking`, `LuminanceSmoothing`, `LuminanceNoiseReductionDetail`, `LuminanceNoiseReductionContrast`, `ColorNoiseReduction`, `ColorNoiseReductionDetail`, `ColorNoiseReductionSmoothness`

effectsPanel: `PostCropVignetteAmount`, `PostCropVignetteMidpoint`, `PostCropVignetteFeather`, `PostCropVignetteRoundness`, `PostCropVignetteStyle`, `PostCropVignetteHighlightContrast`, `GrainAmount`, `GrainSize`, `GrainFrequency`

lensCorrectionsPanel: `AutoLateralCA`, `LensProfileEnable`, `LensProfileDistortionScale`, `LensProfileVignettingScale`, `LensManualDistortionAmount`, `DefringePurpleAmount`, `DefringePurpleHueLo`, `DefringePurpleHueHi`, `DefringeGreenAmount`, `DefringeGreenHueLo`, `DefringeGreenHueHi`, `VignetteAmount`, `VignetteMidpoint`, `PerspectiveVertical`, `PerspectiveHorizontal`, `PerspectiveRotate`, `PerspectiveScale`, `PerspectiveAspect`, `PerspectiveX`, `PerspectiveY`, `PerspectiveUpright`

calibratePanel: `ShadowTint`, `RedHue`, `RedSaturation`, `GreenHue`, `GreenSaturation`, `BlueHue`, `BlueSaturation`

lensBlurPanel: `LensBlurActive`, `LensBlurAmount`, `LensBlurCatEye`, `LensBlurHighlightsBoost`, `LensBlurFocalRange`

Other: `straightenAngle`

### Local adjustment parameters by process version

**v2 (8 params):** `local_Exposure`, `local_Contrast`, `local_Clarity`, `local_Saturation`, `local_Sharpness`, `local_ToningLuminance`, `local_ToningHue`, `local_ToningSaturation`

**v3/v4 (18 params):** `local_Temperature`, `local_Tint`, `local_Exposure`, `local_Contrast`, `local_Highlights`, `local_Shadows`, `local_Clarity`, `local_Saturation`, `local_ToningHue`, `local_ToningSaturation`, `local_Sharpness`, `local_LuminanceNoise`, `local_Moire`, `local_Defringe`, `local_Blacks`, `local_Whites`, `local_Dehaze`, `local_PointColors`

**v5 (25 params):** all v3/v4 + `local_Texture`, `local_Hue`, `local_Amount`, `local_Maincurve`, `local_Redcurve`, `local_Greencurve`, `local_Bluecurve`

**v6 (27 params):** all v5 + `local_Grain`, `local_RefineSaturation`

### LrDevelopController key functions (95 in SDK)

**Get/Set:** `getValue(param)`, `setValue(param, value)`, `getRange(param)`, `increment(param)`, `decrement(param)`, `resetToDefault(param)`, `resetAllDevelopAdjustments()`, `startTracking(param)`, `stopTracking()`, `setTrackingDelay(seconds)`, `setMultipleAdjustmentThreshold(amount)`

**Observation:** `addAdjustmentChangeObserver(context, table, callback)`

**Tools:** `selectTool(name)` — loupe, crop, dust, redeye, masking, upright, point_color, local_point_color, depth_refinement. `getSelectedTool()`, `goToRemove()`, `goToMasking()`, `goToEyeCorrection()`, `editInPhotoshop()`

**Masks (SDK 11.0+):** `createNewMask(type, subtype)`, `addToCurrentMask(type, subtype)`, `subtractFromCurrentMask(type, subtype)`, `intersectWithCurrentMask(type, subtype)`, `getAllMasks()`, `getSelectedMask()`, `selectMask(id, param)`, `deleteMask(id, param)`, `invertMask(id, param)`, `duplicateAndInvertMask(id, param)`, `toggleHideMask(id, param)`, `toggleOverlay()`
- Mask types: brush, gradient, radialGradient, rangeMask, aiSelection
- Mask subtypes: color, luminance, depth, subject, sky, background, objects, people, landscape

**Spots (SDK 14.1+):** `countAllSpots()`, `getAllSpots()`, `getSelectedSpotIndex()`, `getSelectedSpotParams()`, `getSelectedSpotType()`, `setSelectedSpotIndex(index)`, `setSelectedSpotParams(params)`, `setSelectedSpotType(spotType, useGenAI)`, `moveSelectedSpot(dx, dy)`, `deleteSelectedSpot()`, `deleteSelectedVariation()`, `refreshSelectedSpot()`, `gotoNextVariation()`, `gotoPreviousVariation()`
- Spot types: `heal_patchmatch`, `heal`, `clone` (NOT "remove" — `goToRemove()` opens the Remove tool but the inner spot types are these three)

**AI/Enhance (SDK 14.5+):** `setEnhance(paramName, value, denoiseAmount)` (SDK 15.3 — `paramName` ∈ `"denoise"`, `"rawDetails"`, `"superRes"`; `value` boolean), `toggleEnhance(paramName, denoiseAmount, callback, callbackFuncArgTable)` (deprecated, use `setEnhance`), `changeDenoiseAmount(amount)`, `getEnhancePanelState()`, `toggleReflectionRemoval()`, `changeReflectionRemovalAmount(amount)`, `changeReflectionRemovalQuality(amount)`, `getReflectionRemovalPanelState()`, `detectDistractingPeople()`, `applyRemovalOnDetectedDistractingPeople()`

**Point Color (SDK 13.2+):** `addPointColorSwatch()`, `deletePointColorSwatch()`, `selectPointColorSwatch(index)`, `updateSelectedPointColorSwatch(params)`, `getSelectedPointColorSwatchIndex()`

**Lens Blur (SDK 13.3+):** LensBlur params (Active, Amount, CatEye, HighlightsBoost, FocalRange), `setLensBlurBokeh(type)` — Circle, SoapBubble, Blade, Ring, Anamorphic. `toggleLensBlurDepthVisualization()`

**Process Version:** `getProcessVersion()`, `setProcessVersion(version)` — "Version 1" through "Version 6"

**Other:** `setActiveColorGradingView(view)`, `getActiveColorGradingView()`, `setAutoTone()`, `setAutoWhiteBalance()`, `showClipping()` (no args), `revealPanel(paramOrPanelID, subPanelID?)`, `revealPanelIfVisible(panelID)`, `revealAdjustedControls()`, `resetCrop()`, `resetTransforms()`, `resetMasking()`, `resetRedeye()`, `resetFilterWithName(name)`. Deprecated: `resetHealing()`, `resetBrushing()`, `resetCircularGradient()`, `resetGradient()`, `resetSpotRemoval()`, `goToHealing()`, `goToSpotRemoval()`, `goToDevelopGraduatedFilter()`, `goToDevelopRadialFilter()`.

Panel IDs: `adjustPanel`, `tonePanel`, `mixerPanel`, `colorGradingPanel`, `detailPanel`, `effectsPanel`, `lensCorrectionsPanel`, `calibratePanel`, `lensBlurPanel`. Mixer sub-panel IDs (for `revealPanel(panelID, subPanelID)`, SDK 13.2+): `hslColorPanel`, `pointColorPanel`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Calling `canvas.toDataURL()` in UXP | Not supported. Use `imaging.encodeImageData({ base64: true })`. |
| Skipping `imageData.dispose()` | Leaks native memory. UDT warns at 600MB. |
| Writing LrC `f:picture` to same path repeatedly | LrC caches every version — massive leak. Alternate two paths. |
| Running `collectgarbage()` in LrC | Blocked by sandbox. Design around it (fixed-size state, path alternation). |
| No debounce on LrC develop observer | Drags spawn hundreds of coroutines. Use version-counter pattern. |
| CSS Grid / `transform` / `transition` in UXP | Grid and transitions not supported. Transforms require `CSSNextSupport` flag. |
| `z-index` for overlays in UXP | Flaky. Use DOM order / flex order. |
| 16-bit UXP pixels expecting 0-65535 | Default is 0-32768. Pass `{ fullRange: true }` to `getData()`. |
| Trying to add LrC UI to Develop panel | Not a supported plugin type. Use floating dialog from Library menu. |
| Polling inside `requestJpegThumbnail` callback without `done` guard | Callback may fire more than once. Set and check `done`. |
| Using `<label for="id">` in UXP | Not supported. Wrap the label around the control. |
| Swallowing batchPlay exceptions in try/catch | Prevents cancel propagation. Let exceptions from `executeAsModal` propagate. |
| Using CSS transforms without `CSSNextSupport` flag | Properties silently ignored. Add `featureFlags: { "CSSNextSupport": true }` to manifest. |
| Creating tasks inside `processRenderedPhotos` | LrC already runs it in a task. Nesting tasks causes incorrect behavior. |
| Calling `LrDialogs` during `LrShutdownApp` | Namespace unavailable during app shutdown. Use `LrShutdownPlugin` instead. |
| Not holding `requestJpegThumbnail` return value | May be garbage collected, callback never fires. Store in a local variable. |
| Using `break` in LrC `directoryEntries`/`files`/`recursiveFiles` loops | Not safe. Collect results into a table or use a flag + early `return`. |
| Modifying nested tables in `LrPrefs` expecting auto-save | Only top-level key reassignment triggers persistence. Reassign the root key after mutating nested values. Use `prefs:pairs()` not Lua `pairs(prefs)`. |

## Deeper References (in this skill directory)

- **`lrc-sdk.md`** — Full Lightroom Classic SDK reference: all plugin types (export, publish, filter, metadata), `Info.lua` manifest, `LrView` widgets and data binding, `LrDevelopController` (95 functions), full module catalog (40+ modules), LrC memory-leak prevention details, export pipeline, catalog operations, external communication (LrSocket, LrHttp, Controller SDK), and the processor CLI spec for LrC-to-Rust bridging.
- **`ps-uxp-sdk.md`** — Full Photoshop UXP reference: Manifest v5, Document/Layer DOM (33+29 Document, 28+55 Layer properties/methods, 38 filter methods), Selection class, Imaging API full signatures, batchPlay and action system (5 reference forms, action recording), executeAsModal details, Spectrum UXP + SWC component catalogs, HTML/CSS support and limitations, Canvas API limits, event system, text/typography API, color management, external communication (WebView, hybrid C++ bridge, network), and the known-issues catalog.

**When to load deep references:**
- `ps-uxp-sdk.md` — when using batchPlay descriptors, Document/Layer DOM methods, Selection class, filter methods, SWC components, text/typography APIs, or debugging known issues
- `lrc-sdk.md` — when building export/publish services, working with catalog queries (`findPhotos`, custom metadata), building UIs with `LrDialogs.presentFloatingDialog`, using LrSocket/Controller SDK, or needing the full module API reference

The quick-ref snippets above cover common cases. Load deep references only for non-trivial implementations.
