# Adobe Plugin Skill Improvement — Design Spec

## Goal

Comprehensive rewrite of the three skill files (SKILL.md, uxp-photoshop.md, lrc-sdk.md) to achieve full API coverage, verified accuracy, and practical patterns. The tiered approach keeps SKILL.md as the "90% file" that loads on every activation, with deep reference files organized by use-case for the long tail.

## Sources Analyzed

- LrC SDK 15.2: 50+ HTML API reference files, 2 PDF manuals (207-page SDK Guide + 14-page Controller SDK Guide), 12 sample plugins with all Lua source
- UXP Photoshop: AdobeDocs/uxp-photoshop GitHub repo (docs through UXP 9.2.0 / PS 27.4)
- AdobeDocs/lightroom-public-apis: Confirmed as Lightroom Cloud REST APIs — out of scope for this skill
- Chromascope project experience (existing content)

## Key Corrections

| Item | Current | Corrected |
|---|---|---|
| UXP version baseline | UXP 8.1.0 / PS 23.3+ | UXP 9.2.0 / PS 27.4 |
| CSS box-shadow, transforms | "Not supported" | Supported with `CSSNextSupport` feature flag in manifest (`featureFlags: { "CSSNextSupport": true }`) |
| localStorage / sessionStorage | "Not available" (listed in DOM Limitations) | Available as global storage APIs. Not available inside WebView when loading local content. |
| WebView availability | "Modal dialogs only" | Panels since UXP 6.4 / PS 24.1. Local HTML files since UXP 8.0. Domains optional since UXP 9.0. |
| sp-icon count | "36 built-in icons" | 40 built-in icons |
| LrC develop parameters covered | ~12 parameters mentioned | 117 global + 27 local adjustment parameters across process versions 1-6 |
| LrDevelopController functions | ~10 functions documented | 94 total functions |
| UXP Document/Layer classes | Not documented | 28+28 Document properties/methods, 27+46 Layer properties/methods |

## File Structure

```
SKILL.md          (~530-580 lines)  — "90% file", loads on every activation
uxp-photoshop.md  (~1080-1150 lines) — Deep UXP reference, organized by use-case
lrc-sdk.md        (~1000-1060 lines) — Deep LrC reference, organized by use-case
```

---

## SKILL.md — Content Plan

### 1. Frontmatter (~5 lines)
Unchanged name (`adobe-plugin-development`) and description.

### 2. When to Use / Don't Use (~15 lines)
Retained from current. Add: batchPlay operations, Document/Layer DOM manipulation, Action Recording.

### 3. Core Capability Matrix (~20 lines)
Updated table. Add columns/rows for: Document DOM model (UXP has full DOM; LrC has catalog-centric model), AI features (UXP: generativeUpscale; LrC: denoise, reflection removal, distraction detection), Selection API (UXP: full Selection class v25.0+; LrC: LrSelection namespace).

### 4. Critical Pitfalls (~130 lines)

**UXP pitfalls (preserved + new):**
- Canvas has no pixel ops (preserved)
- Imaging API gotchas — 16-bit range, dispose(), targetSize, RGB-only encode, chunky/planar (preserved)
- Layout is flexbox-only (CORRECTED: box-shadow/transforms available with CSSNextSupport feature flag; transitions/animations/Grid still not supported)
- NEW: `CSSNextSupport` feature flag requirement — box-shadow, transform-origin, scaleX/scaleY, translate all require `featureFlags: { "CSSNextSupport": true }` or `enableSWCSupport: true` in manifest. Without this flag, CSS silently ignores these properties.
- NEW: localStorage/sessionStorage ARE available (correction from "not available"). Not available inside WebView with local content.
- NEW: WebView works in panels since UXP 6.4, not just modal dialogs
- NEW: executeAsModal exception anti-pattern — never `try { await batchPlay(...) } catch(e) {}` — this prevents automatic cancellation termination
- NEW: executeAsModal timeOut option (v25.10) — modal collisions now retry for `timeOut` duration instead of immediate error
- NEW: batchPlay execution errors resolve (not reject) with error objects — check return values
- NEW: `updateUI()` (v26.0) has no effect outside tracking contexts (slider handlers)
- Panel show/hide bug PS-57284 (preserved)
- label for="id" not supported (preserved)
- option needs explicit value (preserved)

**LrC pitfalls (preserved + new):**
- Long-running plugins leak catastrophically — all 6 rules preserved:
  1. Frame alternation mandatory (40GB leak)
  2. Guard requestJpegThumbnail callbacks (done flag, nil jpegData)
  3. Debounce async tasks with version counter
  4. No unbounded module-level state
  5. Clean up temp files on dialog open
  6. collectgarbage is unavailable
- NEW: Busy-guard + pending flag coalescing pattern (promoted from lrc-sdk.md only)
- Develop module can't have custom panels (preserved)
- No "active photo changed" observer (preserved)
- NEW: requestJpegThumbnail — must hold reference to returned request object or it may be garbage collected
- NEW: getDevelopSettings has typos in official API: `HueAdjustmentMagenha`, `LuminanceAdjustmentAque`, `Parametriclights`
- NEW: processRenderedPhotos runs in a task LrC creates — do not create your own task inside it
- NEW: LrDialogs namespace NOT available during app shutdown (LrShutdownApp)
- NEW: withWriteAccessDo on Mac: successive calls without user interaction coalesce into single undo event

### 5. UXP Quick Reference (~95 lines)

**Read active document pixels** — code snippet (preserved from current)

**Display a software-rendered image** — code snippet (preserved from current)

**Document & layer operations** — NEW code snippet:
```javascript
const doc = app.activeDocument;
await doc.suspendHistory(async () => {
  const layer = await doc.createLayer();
  await layer.applyGaussianBlur(5.0);
}, "My Edit");
```

**batchPlay basics** — NEW code snippet with reference form examples:
```javascript
const result = await action.batchPlay([{
  _obj: "get",
  _target: [{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" },
             { _ref: "document", _enum: "ordinal", _value: "targetEnum" }],
  _options: { dialogOptions: "silent" }
}], {});
```

**Spectrum component catalog** — condensed table of all 19 sp-* components with key attributes/events. Icon count corrected to 40. Note about SWC (35 packages, requires `enableSWCSupport: true`).

**CSS support summary** — corrected. Supported: flexbox, box model, typography, backgrounds, opacity, calc(), CSS variables, linear-gradient (v8.1+). With CSSNextSupport flag: box-shadow, transform-origin, scaleX/scaleY, translate. Still NOT supported: Grid, transitions/animations, float, text-transform, font shorthand, position:sticky.

**Manifest v5 essentials** — updated. Add featureFlags (`CSSNextSupport`, `enableSWCSupport`), host.data fields (`apiVersion`, `loadEvent`, `enableMenuRecording`).

**Event system** — code snippet preserved. Add note about 198 action events and userIdle for deferred heavy re-renders.

**executeAsModal** — code snippet preserved. Add timeOut option, registerAutoCloseDocument, exception anti-pattern warning.

**Refresh on document changes** — code snippet preserved.

### 6. LrC Quick Reference (~90 lines)

**Observe develop-slider changes** — code snippet (preserved)

**Export thumbnail and pipe to external tool** — code snippet (preserved, add note about holding request object reference)

**Live-updating picture display** — code snippet (preserved)

**Debounced async rerender** — code snippet (preserved, add busy-guard + pending flag pattern)

**Info.lua manifest** — expanded table of all entry points:

| Field | Purpose |
|---|---|
| LrSdkVersion | Target SDK version (required) |
| LrSdkMinimumVersion | Minimum supported SDK version |
| LrToolkitIdentifier | Unique reverse-domain ID (required) |
| LrPluginName | Display name (required, SDK 2.0+) |
| LrLibraryMenuItems | Library > Plug-in Extras menu items |
| LrExportMenuItems | File > Plug-in Extras menu items |
| LrHelpMenuItems | Help > Plug-in Extras menu items |
| LrExportServiceProvider | Export/publish service definitions |
| LrExportFilterProvider | Export filter (post-process action) definitions |
| LrMetadataProvider | Custom metadata definition script |
| LrMetadataTagsetFactory | Custom metadata tagset(s) |
| LrPluginInfoProvider | Plugin Manager dialog customization |
| LrPluginInfoUrl | URL shown in Plugin Manager |
| LrInitPlugin | Script run when plugin loads |
| LrForceInitPlugin | Force init at app startup (SDK 4.0+) |
| LrEnablePlugin / LrDisablePlugin | Enable/disable scripts (SDK 3.0+) |
| LrShutdownPlugin | Script on user unload (SDK 3.0+) |
| LrShutdownApp | Script on app exit (SDK 4.0+) |
| URLHandler | Custom lightroom:// URL handling (SDK 4.0+) |
| VERSION | { major, minor, revision, build } |

**Data binding patterns** — NEW. Basic `LrView.bind(key)`, transform binding, conditional via `LrBinding.keyEquals`, cross-table binding.

**Complete develop parameter list** — NEW. Dense format (params listed per panel):

```
adjustPanel: Temperature, Tint, Exposure, Highlights, Shadows, Brightness, Contrast,
  Whites, Blacks, Texture, Clarity, Dehaze, Vibrance, Saturation, PresetAmount, ProfileAmount
tonePanel: ParametricDarks/Lights/Shadows/Highlights, ParametricShadowSplit/MidtoneSplit/
  HighlightSplit, ToneCurve, ToneCurvePV2012/Red/Blue/Green, CurveRefineSaturation
mixerPanel: [Saturation|Hue|Luminance]Adjustment[Red|Orange|Yellow|Green|Aqua|Blue|Purple|
  Magenta], PointColors, GrayMixer[same 8 colors]
colorGradingPanel: SplitToningShadow[Hue|Saturation], ColorGradeShadowLum, SplitToningHighlight
  [Hue|Saturation], ColorGradeHighlightLum, ColorGradeMidtone[Hue|Sat|Lum], ColorGradeGlobal
  [Hue|Sat|Lum], SplitToningBalance, ColorGradeBlending
detailPanel: Sharpness, SharpenRadius/Detail/EdgeMasking, LuminanceSmoothing,
  LuminanceNoiseReduction[Detail|Contrast], ColorNoiseReduction/Detail/Smoothness
effectsPanel: PostCropVignette[Amount|Midpoint|Feather|Roundness|Style|HighlightContrast],
  Grain[Amount|Size|Frequency]
lensCorrectionsPanel: AutoLateralCA, LensProfileEnable/DistortionScale/VignettingScale,
  LensManualDistortionAmount, DefringePurple[Amount|HueLo|HueHi], DefringeGreen[same],
  VignetteAmount/Midpoint, Perspective[Vertical|Horizontal|Rotate|Scale|Aspect|X|Y|Upright]
calibratePanel: ShadowTint, Red[Hue|Saturation], Green[Hue|Saturation], Blue[Hue|Saturation]
lensBlurPanel: LensBlur[Active|Amount|CatEye|HighlightsBoost|FocalRange]
Other: straightenAngle
```

**Local adjustment parameters** — NEW. Table by process version (v2: 8 params, v3/v4: 18, v5: 23 + curves, v6: adds local_Grain, local_RefineSaturation).

### 7. Common Mistakes Table (~45 lines)
Preserved existing 11 entries. Add:
- Swallowing batchPlay exceptions in try/catch → prevents cancel propagation
- Using CSS transforms without CSSNextSupport flag → silently ignored
- Creating tasks inside processRenderedPhotos → LrC already runs it in a task
- Calling LrDialogs during LrShutdownApp → namespace unavailable
- Not holding requestJpegThumbnail return value → may be garbage collected

### 8. Deeper References (~10 lines)
Updated pointers to both deep reference files with when-to-load guidance. Add note about SWC, batchPlay, Document/Layer DOM, and export pipeline being in the deep references.

---

## uxp-photoshop.md — Content Plan

### 1. Header (~5 lines)
Version baseline: UXP 9.2.0 / Manifest v5 / Photoshop 27.4. "UXP is NOT a browser" warning preserved.

### 2. Building a Panel Plugin (~120 lines)
- **Manifest v5 full spec**: JSON example (preserved), permissions table (preserved, WebView corrected to note panel support since v6.4), NEW: featureFlags (`CSSNextSupport`, `enableSWCSupport`), host.data fields (`apiVersion`, `loadEvent`, `enableMenuRecording`)
- **Panel lifecycle**: code snippet (preserved), 300ms timeout, PS-57284 bug (preserved)
- **Panel sizing**: table (preserved)
- **Theme awareness**: CSS variables (all 13 + 3 font-size, preserved), prefers-color-scheme (preserved), Spectrum auto-theme note (preserved)
- **Plugin packaging**: NEW — .ccx format, UDT packaging, marketplace vs direct distribution

### 3. Document & Layer Model (~200 lines)
All NEW content.

**Document class** — properties table (28 entries with type, R/W, min version) + methods table (28 entries). Key gotchas inline: `calculations()` requires one unlocked pixel layer, `colorProfileName` returns "None" when type not CUSTOM/WORKING, `histogram` only valid for RGB/CMYK/INDEXEDCOLOR, `suspendHistory` scope limited to single document, `mergeVisibleLayers` vs `flatten` background behavior.

**Layer class** — properties table (27 entries) + non-filter methods table (16 entries). Key gotchas: blendMode validation (v24.2+) now throws errors, `isClippingMask` release affects layers above, `parent` returns null for top-level.

**Layer filter methods** — compact table of 30+ filters with parameter ranges and color mode restrictions. Grouped: Blur (gaussian, motion, smart, lens, average, blur/blurMore), Sharpen (sharpen/sharpenEdges/sharpenMore, unSharpMask), Noise (addNoise, median, dustAndScratches, despeckle), Distortion (displace, pinch, wave, zigZag, ripple, shear, polarCoordinates, twirl, spherize, oceanRipple), Effects (highPass, clouds, differenceClouds, lensFlare, diffuseGlow, glassEffect), Other (offset, custom, deInterlace, NTSC, maximum, minimum), Compositing (applyImage v24.5+).

**Selection class** (v25.0+) — 5 properties + 21 methods. Shape creation, modification, transforms, channel ops.

**Other classes** — brief table: LayerComp (v24.0+), ColorSampler (v24.0+), CountItem (v24.1+), PathItem (v23.3+), Guide (v23.0+), HistoryState (v22.5+) with key properties/methods.

### 4. Working with Pixels (~100 lines)
Preserved from current with minor expansions.
- PhotoshopImageData interface (preserved)
- Component value ranges (preserved, the 0-32768 gotcha)
- Memory layout (preserved)
- All 8 imaging methods with full signatures (preserved + minor additions: putLayerMask only "user" kind, level return from getPixels, 32-bit linear profile naming caveat)
- Performance guidelines (preserved)
- Display pipeline code snippet (preserved)

### 5. batchPlay & Action System (~60 lines)
All NEW content.
- Action descriptor structure (`_obj`, `_target`, `_options`)
- All 5 reference forms with examples (ID/Index/Name/Enum/Property)
- `multiGet` command for bulk property reads with code example
- `batchPlaySync` (v23.1)
- Action Recording API (v25.0+) — `recordAction`, handler registration, Record Again
- Descriptor discovery methods (Copy As JavaScript, notification listener, Developer menu)
- Error handling: resolved-with-error vs rejected
- action module: all 7 functions in table (addNotificationListener, batchPlay, batchPlaySync, getIDFromString, recordAction, removeNotificationListener, validateReference)

### 6. executeAsModal (~50 lines)
Expanded from current ~40 lines.
- Full options (commandName, descriptor, interactive, timeOut v25.10) — preserved + new
- ExecutionContext object (isCancelled, onCancel, reportProgress) — preserved
- HostControl methods: suspendHistory/resumeHistory (preserved) + NEW: registerAutoCloseDocument/unregisterAutoCloseDocument, `finalName` on resumeHistory
- Modal collision retry (v25.10) — NEW
- Exception anti-pattern warning — NEW
- "PS only creates history state if document was modified" — NEW
- Nested scopes, event silencing, interactive mode — NEW
- Modal events (preserved)

### 7. Building UIs (~250 lines)
Mostly preserved with corrections.

**HTML element support** (~80 lines, preserved). Minor additions: video timeUpdate event (v9.1), audio support (v9.1).

**CSS support** (~100 lines, CORRECTED). Supported list preserved. NOT Supported table corrected: box-shadow/transforms moved to "Supported with CSSNextSupport flag" category. Add `featureFlags` manifest requirement. Add `linear-gradient` as supported since v8.1.0. Still NOT supported: Grid, transitions/animations, float, text-transform, font shorthand, position:sticky.

**Spectrum UXP components** (~50 lines, preserved). Icon count corrected to 40 with full icon list. NEW: SWC overview — 35 npm packages, setup instructions (`enableSWCSupport: true`), locked to v0.37.0 in UXP 8.0, list of SWC-only components not available as sp-*.

**React + Spectrum** — code snippet preserved.

**DOM limitations** (~30 lines, CORRECTED). localStorage/sessionStorage removed from "not available" — they ARE available. Other limitations preserved. NEW: `URLSearchParams` (v9.1), `Navigator.language` (v26.0), `HTMLElement.append()/prepend()/replaceChildren()` (v26.0), system sleep/awake events (v27.4). `SecureStorage` now Web Storage API compliant (v9.0).

**Dialog behavior** — preserved (showModal returns Promise, removal required, REJECTION_REASON constants).

**Canvas API** (~40 lines, preserved). No changes — canvas limitations are unchanged.

### 8. Text & Typography (~40 lines)
All NEW content.
- TextItem class: properties (contents, isParagraphText, isPointText, orientation, textClickPoint), methods (convertToParagraphText, convertToPointText, convertToShape, createWorkPath)
- CharacterStyle: 34 properties listed (font, size, fauxBold, fauxItalic, tracking, leading, baselineShift, color, antiAliasMethod, etc.)
- ParagraphStyle: 14 properties (justification, hyphenation, indentation, spacing, kinsoku, mojikumi)
- WarpStyle: 5 properties (bend, direction, horizontalDistortion, verticalDistortion, style — 16 warp types)

### 9. Color Management (~25 lines)
All NEW content.
- SolidColor class: constructor, import path, color space accessors (.rgb, .cmyk, .hsb, .lab, .gray, .nearestWebColor), isEqual()
- Individual color classes with value ranges: RGBColor (0-255), CMYKColor (0-100), HSBColor (hue 0-360, sat/bright 0-100), LabColor (l 0-100, a/b -128..127), GrayColor (0-100)
- `core.convertColor(sourceColor, targetModel)` (v23.0+)
- `app.getColorProfiles(mode)` (v24.1+) — code snippet preserved

### 10. External Communication (~50 lines)
Mostly preserved with corrections.
- **Network APIs**: fetch (recommended), XHR (limitations preserved), WebSocket (preserved). NEW: FormData multipart support (v7.3+).
- **File system**: storage locations, permissions (preserved). NEW: `fs.createReadStream()` (v8.2+).
- **WebView**: CORRECTED — panel support since v6.4. NEW: local HTML (v8.0+), domains optional (v9.0+), drag-and-drop (v9.1+), script injection (v9.0.2+).
- **Hybrid C++ bridge**: preserved (PSDLLMain, JS↔C++ messaging, build requirements).

### 11. App & Core APIs (~40 lines)
All NEW content.
- Photoshop app class: 9 properties (activeDocument, foregroundColor/backgroundColor v24.2, currentTool, documents, fonts, preferences v24.0, actionTree) + 7 methods (createDocument, open, bringToFront, showAlert, convertUnits v23.4, getColorProfiles v24.1, updateUI v26.0)
- Core module key functions: convertColor, convertGlobalToLocal (v26.0), getActiveTool, performMenuCommand, getLayerTree[Sync], getLayerGroupContents[Sync], isModal, historySuspended, calculateDialogSize, getCPUInfo/getGPUInfo, setExecutionMode, translateUIString, redrawDocument
- Preferences API: 13 groups listed (cursors, fileHandling, general, etc.)
- Prototype extension system: 5 extensible classes (Photoshop, Document, Layer, ActionSet, Action)

### 12. Known Issues (~40 lines)
Preserved existing 35 items, reorganized. Add notes about which items are affected by CSSNextSupport flag. Add note that the known issues page is not systematically updated for fixed items. Add new items: `lockDocumentFocus: true` workaround for dialog modal state.

---

## lrc-sdk.md — Content Plan

### 1. Header (~5 lines)
LrC 15.2+ SDK. Lua 5.1.5. Sandbox constraints summary.

### 2. Building a Plugin (~100 lines)

**Complete Info.lua field reference** — table of all 20+ fields with type, required status, SDK version, notes. Include: LrSdkVersion, LrSdkMinimumVersion, LrToolkitIdentifier, LrPluginName, LrPluginInfoProvider, LrPluginInfoUrl, LrInitPlugin, LrForceInitPlugin, LrShutdownPlugin, LrShutdownApp, LrEnablePlugin, LrDisablePlugin, LrExportMenuItems, LrLibraryMenuItems, LrHelpMenuItems, LrExportServiceProvider, LrExportFilterProvider, LrMetadataProvider, LrMetadataTagsetFactory, URLHandler, VERSION, LrAlsoUseBuiltInTranslations, LrLimitNumberOfTempRenditions.

**Info.lua restricted environment** — only `string` namespace, `LOC`, `WIN_ENV`/`MAC_ENV`, `_VERSION` available. No `import` or `require`.

**Plugin lifecycle** — loading sequence (Info.lua → LrInitPlugin → on-demand or LrForceInitPlugin at startup). Shutdown protocol: LrShutdownApp returns `{ LrShutdownFunction = function(doneFunction, progressFunction) }` with 10-second timeout. LrDialogs NOT available during shutdown.

**Installation paths** — table: macOS user (`~/Library/Application Support/Adobe/Lightroom/Modules`), macOS all users, Windows paths. Auto-loaded plugins can't be removed, only disabled.

**Lua environment** — available globals (assert, dofile, error, pairs, ipairs, pcall, etc.), NOT available (collectgarbage, getfenv, setfenv, module, package, newproxy). Available namespaces: io, math, string. Partially available: os (only clock/date/time/tmpname), table (no getn/setn/maxn), coroutine (only canYield/running), debug (only getInfo).

**SDK version history** — compact table: 1.3 (original), 2.0 (+filters, +metadata, +pluginInfo), 3.0 (+publish, +enable/disable, +tagsets), 4.0 (+shutdown, +forceInit, +URLHandler), 5.0 (+rendition throttling, +video filters), 6.0+ (controller SDK).

**Debugging** — LrLogger (enable with 'print'/'logfile'), log locations (Mac: `~/Library/Logs/Adobe/Lightroom/LrClassicLogs`, Windows: `%LOCALAPPDATA%\Adobe\Lightroom\Logs\LrClassicLogs`), deprecation tracking via config.lua.

### 3. Building an Export/Publish Service (~90 lines)

**5-stage export pipeline**:
1. Deciding render settings: `updateExportSettings(exportSettings)`
2. Filtering photos: each filter's `shouldRenderPhoto(exportSettings, photo)` per-photo
3. Requesting renditions: `processRenderedPhotos(functionContext, exportContext)` — each filter runs in own task
4. Processing rendered photos: `rendition:waitForRender()` blocks until upstream completes
5. Error reporting and cleanup: temp files cleaned, error summary, "Previous Export" collection

**Export service provider callbacks** — table: startDialog, endDialog (why: 'ok'/'cancel'/'changedServiceProvider'), sectionsForTopOfDialog, sectionsForBottomOfDialog, processRenderedPhotos, updateExportSettings. Properties: allowFileFormats (JPEG/PSD/PSB/TIFF/DNG/PNG/JXL/AVIF/ORIGINAL), allowColorSpaces, canExportVideo, exportPresetFields, hideSections/showSections, supportsIncrementalPublish.

**Export processing loop** — code snippet with configureProgress, renditions iterator, waitForRender, recordPublishedPhotoId/Url, file cleanup.

**Export property keys** — compact table of key properties: LR_format, LR_export_colorSpace, LR_export_bitDepth, LR_jpeg_quality, LR_size_doConstrain, LR_size_resizeType, LR_size_maxWidth/Height, LR_outputSharpeningOn/Media/Level, LR_minimizeEmbeddedMetadata, LR_useWatermark, LR_cantExportBecause.

**Publish service additions** — key callbacks: deletePhotosFromPublishedCollection, metadataThatTriggersRepublish, getCollectionBehaviorInfo, goToPublishedCollection/Photo. Properties: small_icon, supportsCustomSortOrder.

**Export filter provider** — shouldRenderPhoto, postProcessRenderedPhotos, renditions loop with sourceRendition/renditionToSatisfy pattern.

### 4. Building a Live Develop Visualization (~80 lines)
Preserved from current with expansions.

**Architecture** — 5-step process preserved. Key challenges preserved.

**Memory leak prevention** — all 6 rules preserved with code snippets:
1. Frame alternation (40GB leak context)
2. requestJpegThumbnail guard (add: must hold reference to returned request object)
3. Debounce with version counter + busy-guard/pending flag pattern (both patterns)
4. No unbounded module-level state
5. Temp file cleanup on dialog open
6. collectgarbage unavailable

**External binary bridge** — condensed. Keep platform detection code (WIN_ENV, arch detection, binary path with .exe suffix). Trim Chromascope-specific CLI flags to a brief "example CLI convention" showing the decode/render two-step pattern.

### 5. Working with the Catalog (~110 lines)

**LrCatalog** — key methods with signatures: withWriteAccessDo (actionName, func, timeoutParams), withPrivateWriteAccessDo, withProlongedWriteAccessDo. Batch operations: batchGetRawMetadata, batchGetFormattedMetadata, batchGetPropertyForPlugin. Search: findPhotos with searchDesc (criteria list, operations, combining with union/intersect/exclude, plugin metadata via `sdktext:plugin_id.field_name`). Tip: export smart collection settings to get searchDesc format. Other key methods: getTargetPhoto, getTargetPhotos, getMultipleSelectedOrAllPhotos, setSelectedPhotos, addPhoto (with metadataPresetUUID/developPresetUUID SDK 12.5), createCollection/CollectionSet/SmartCollection, getActiveSources/setActiveSources. Gotchas: findPhotoByUuid param named `path` but is UUID string, new objects not accessible until write gate returns, withWriteAccessDo undo coalescing on Mac.

**LrPhoto** — key methods: getRawMetadata(key), getFormattedMetadata(key), setRawMetadata(key, value), getPropertyForPlugin, setPropertyForPlugin, getDevelopSettings (experimental, typos: HueAdjustmentMagenha, LuminanceAdjustmentAque, Parametriclights), applyDevelopPreset (presetAmount 0-200, updateAISettings), applyDevelopSettings (experimental), requestJpegThumbnail, buildSmartPreview/deleteSmartPreview, copySettings/pasteSettings (SDK 10.3+), quickDevelop* methods, rotateLeft/rotateRight, addKeyword/removeKeyword, isAvailableForEditing (SDK 15.3), needsUpdateAISettings/updateAISettings.

**getRawMetadata key table** — all 40+ keys grouped by SDK version with types and notes. Key entries: path, fileName, uuid, isVirtualCopy, rating, colorNameForLabel, pickStatus, fileFormat, width/height/aspectRatio, isCropped, dateTimeOriginal (Cocoa epoch!), gps, keywords, customMetadata, smartPreviewInfo, bitDepth (SDK 12.1), altTextAccessibility (SDK 13.2).

**getFormattedMetadata key table** — all 70+ keys grouped by category (file info, exposure, camera, IPTC contact, IPTC content, location, IPTC extension/PLUS). Focus on commonly-used keys.

**setRawMetadata writable keys** — compact list of all settable keys.

**Custom metadata** — definition script format (metadataFieldsForPhotos, schemaVersion, updateFromEarlierSchemaVersion), field properties (id, title, dataType, searchable, browsable), reading/writing (getPropertyForPlugin, setPropertyForPlugin with withPrivateWriteAccessDo), tagsets (com.adobe.* wildcard for all plugin fields).

### 6. Building UIs (~80 lines)

**LrView factory methods** — table of all 25 methods grouped: containers (column, row, view, group_box, tab_view, tab_view_item, scrolled_view), controls (checkbox, color_well, combo_box, edit_field, password_field, picture, popup_menu, push_button, radio_button, slider, simple_list, static_text, catalog_photo), layout helpers (separator, spacer, control_spacing, dialog_spacing, label_spacing). Key properties per widget noted inline.

**Property categories** — compact reference: view properties (bind_to_object, tooltip, visible — hidden items still affect layout), control properties (enabled, font with canonical names `<system>/<system/small>/<system/bold>/<system/small/bold>`, size), text properties (height_in_lines, width_in_chars, width_in_digits), edit properties (key ones: immediate, validate, precision, min/max, placeholder_string, string_to_value/value_to_string), node layout (fill/fill_horizontal/fill_vertical [0..1], width/height, place_horizontal/place_vertical), child layout (margin/margin_*, place: vertical/horizontal/overlapping, spacing).

**Data binding** — patterns with code: basic `LrView.bind(key)`, transform binding (value + transform function), multi-key computed binding (keys array + operation function), LrBinding helpers (keyEquals, keyIsNil, keyIsNotNil, negativeOfKey, andAllKeys, orAllKeys). Cross-table binding via `bind_to_object` override per control. Conditional enablement via `LrBinding.keyEquals`. Observer pattern: `observableTable:addObserver(key, func)`, ALL observer, guard flags to prevent loops.

**LrDialogs** — all 14 functions. Key signatures: presentModalDialog (cancelVerb="< exclude >" hides cancel, accessoryView mutually exclusive with otherVerb), presentFloatingDialog (blockTask for observers, selectionChangeObserver/sourceChangeObserver, windowWillClose), confirm (returns "ok"/"cancel"/"other"), runOpenPanel/runSavePanel, showBezel (only one at a time), showModalProgressDialog, stopModalWithResult. showError for LrErrors strings.

**View layout patterns** — shared widths via LrView.share('label_width'), overlapping placement for conditional visibility, 'skipped item' string for conditional views, repeat...until true for early-return in non-function context.

### 7. External Communication (~40 lines)

**LrSocket** — bind with functionContext, address, port (0=auto), mode (send/receive), callbacks (onConnecting, onConnected, onMessage, onClosed, onError). Methods: send, reconnect, close. Auto-closed if plugin disabled.

**Controller SDK architecture** — two-component: LrC plugin establishes sockets, external driver communicates via TCP on localhost. Simple key=value text protocol. Main loop pattern with _G.running flag and LrTasks.sleep(1/2) yield.

**LrHttp** — get(url, headers, timeout), post(url, postBody, headers, method, timeout), postMultipart(url, content, headers). Must be in async task. Error handling: returns nil + info with error table (errorCode: "cancelled"/"badURL"/"timedOut"/"cannotFindHost"/etc). Content-Type='skip' to omit. Streaming uploads via callback functions (SDK 4.0+).

**LrTasks.execute(cmd)** — shell command execution, returns exit code. Blocks only calling task. Mac App Store sandbox caveat.

**LrShell** — revealInShell, openFilesInApp, openPathsViaCommandLine (Mac: sequential due to fork; Windows: >1500 char command lines split).

### 8. Develop Parameter Reference (~100 lines)

**Complete parameter list by panel** — dense format (same as SKILL.md, all 117 parameters).

**Local adjustment parameters by process version** — table (v2: 8 params, v3/v4: 18, v5: 23 + 4 curves, v6: +local_Grain, +local_RefineSaturation).

**LrDevelopController key functions** — grouped by category:
- **Get/Set**: getValue, setValue, getRange, increment, decrement, resetToDefault, resetAllDevelopAdjustments, startTracking, stopTracking, trackValue, setTrackingDelay
- **Observation**: addAdjustmentChangeObserver
- **Tools**: selectTool (loupe/crop/dust/redeye/masking/upright/point_color/local_point_color/depth_refinement), getSelectedTool, goToRemove, goToMasking, goToEyeCorrection, editInPhotoshop
- **Masks** (SDK 11.0+): createNewMask, addToCurrentMask, subtractFromCurrentMask, intersectWithCurrentMask, getAllMasks, getSelectedMask, selectMask, deleteMask, invertMask, duplicateAndInvertMask, toggleHideMask, toggleOverlay. Mask types: brush/gradient/radialGradient/rangeMask/aiSelection. Mask subtypes: color/luminance/depth/subject/sky/background/objects/people/landscape.
- **Spots** (SDK 14.1+): countAllSpots, getAllSpots, getSelectedSpotIndex/Params/Type, setSelectedSpotIndex/Params/Type, moveSelectedSpot, deleteSelectedSpot, refreshSelectedSpot, gotoNextVariation, gotoPreviousVariation
- **AI/Enhance** (SDK 14.5+): setEnhance (SDK 15.3, replaces toggleEnhance), changeDenoiseAmount, getEnhancePanelState, toggleReflectionRemoval, changeReflectionRemovalAmount/Quality, getReflectionRemovalPanelState, detectDistractingPeople, applyRemovalOnDetectedDistractingPeople
- **Point Color** (SDK 13.2+): addPointColorSwatch, deletePointColorSwatch, selectPointColorSwatch, updateSelectedPointColorSwatch, getSelectedPointColorSwatchIndex
- **Lens Blur** (SDK 13.3+): LensBlur parameters, setLensBlurBokeh (Circle/SoapBubble/Blade/Ring/Anamorphic), toggleLensBlurDepthVisualization
- **Color Grading** (SDK 10.0+): setActiveColorGradingView (3-way/shadow/midtone/highlight/global)
- **Process Version**: getProcessVersion, setProcessVersion ("Version 1" through "Version 6")
- **Other**: setAutoTone, setAutoWhiteBalance, showClipping, revealPanel/revealPanelIfVisible, revealAdjustedControls, resetCrop/resetTransforms/resetMasking/resetHealing/resetRedeye

**Enumerated values** — compact lists: panel IDs (9 + 2 sub-panels), tool names (9), mask types (5), mask subtypes (9), spot types (3), bokeh types (5), color grading views (5), process versions (6), filter names (3), eye correction types (2).

### 9. Module Quick Reference (~60 lines)

Expanded table of all modules with purpose and 2-3 key methods each:
- LrApplication (activeCatalog, versionTable, addHapticObserver SDK 15.0, shutdown SDK 14.3, developPresetFolders, filenamePresets, purchaseSource)
- LrApplicationView (switchToModule, showView, getCurrentModuleName, toggleZoom, zoomToOneToOne SDK 10.0, showSecondaryView)
- LrSelection (27 functions: setRating 0-5, setColorLabel, flagAsPick/Reject/removeFlag, getRating/getColorLabel/getFlag, selectAll/None/Inverse/First/Last, nextPhoto/previousPhoto, extendSelection, removeFromCatalog SDK 14.3)
- LrTasks (startAsyncTask, startAsyncTaskWithoutErrorHandler, sleep, execute, pcall, canYield, yield)
- LrFunctionContext (callWithContext, postAsyncTaskWithContext, addCleanupHandler/addFailureHandler, callWithEmptyEnvironment)
- LrBinding (makePropertyTable, bind helpers: keyEquals, keyIsNil/NotNil, keyIsNot, negativeOfKey, andAllKeys, orAllKeys, kUnsupportedDirection)
- LrObservableTable (addObserver with 'ALL' key, removeObserver, pairs)
- LrExportSession (countRenditions, renditions, doExportOnCurrentTask/NewTask, recordRemoteCollectionId/Url)
- LrExportRendition (waitForRender, recordPublishedPhotoId/Url, renditionIsDone, skipRender, uploadFailed; properties: destinationPath, photo, publishedPhotoId, wasSkipped)
- LrExportContext (configureProgress, renditions, startRendering; properties: exportSession, propertyTable, publishService, publishedCollection)
- LrExportSettings (extensionForFormat — formats: ORIGINAL/JPEG/JXL/AVIF/PNG/TIFF/PSD/PSB/DNG, addVideoExportPresets, supportableVideoExportFormats)
- LrColor (constructor with multiple signatures: (), (r,g,b,a), (name), (name,a); named colors: black/white/gray/red/green/blue/cyan/yellow/magenta/orange/purple/brown; methods: red/green/blue/alpha)
- LrDate (currentTime, timeFromComponents, timestampToComponents — Cocoa epoch: seconds since Jan 1 2001 NOT Unix epoch; formatShortDate/MediumDate/LongDate, timeToW3CDate, timeFromPosixDate/timeToPosixDate)
- LrFileUtils (exists returns 'file'/'directory'/false, readFile SDK 2.2, copy/move, createAllDirectories, delete vs moveToTrash, directoryEntries/files/recursiveFiles — do NOT use break in these loops, chooseUniqueFileName, fileAttributes)
- LrPathUtils (child, getStandardFilePath: home/temp/desktop/appPrefs/pictures/documents/appData, parent — returns nil for root since SDK 2.0, extension/addExtension/removeExtension/replaceExtension, standardizePath)
- LrHttp (get, post, postMultipart, openUrlInBrowser, parseCookie; error codes)
- LrSocket (bind with mode send/receive)
- LrShell (revealInShell, openFilesInApp, openPathsViaCommandLine)
- LrLogger (constructor, enable 'print'/'logfile', levels: trace/debug/info/warn/error/fatal + format variants, quickf for tight loops)
- LrPrefs (prefsForPlugin — deep table gotcha: reassign to trigger save, use prefs:pairs() not Lua pairs())
- LrStringUtils (trimWhitespace, lower/upper Unicode-aware, encodeBase64/decodeBase64, truncate UTF-8-safe, compareStrings, numberToStringWithSeparators)
- LrDigest (SHA256/SHA512 factories, digest/init/update; deprecated: MD4/MD5/SHA1 since LrC 13.5)
- LrMD5 (digest — still works but LrDigest preferred)
- LrXml (createXmlBuilder + beginBlock/endBlock/tag/text/serialize; parseXml + childAtIndex/childCount/name/text/attributes/transform; platform whitespace differences)
- LrErrors (throwUserError, throwCanceled, isCanceledError)
- LrRecursionGuard (constructor, performWithGuard, active property)
- LrProgressScope (constructor, setPortionComplete, isCanceled, done, setCaption, setIndeterminate — indeterminate only works in showModalProgressDialog)
- LrSystemInfo (appWindowSize, numCPUs, memSize, is64Bit, architecture, osVersion, ipAddress, displayInfo)
- LrLocalization (LOC for ZString lookup with ^1-^9 substitution, currentLanguage, ZString escapes: ^r=CR ^n=LF ^B=bullet etc., supported languages: de/en/es/fr/it/ja/ko/nl/pt/ru/sv/th/zh_cn/zh_tw)
- LrPasswords (store/retrieve with OS keychain, scoped by plugin ID)
- LrFtp (create with protocol ftp/sftp, putFile, exists, makeDirectory, disconnect, makeFtpPresetPopup)
- LrTether (startTether/stopTether, triggerCapture/triggerCaptureBlocking, isTetherActive)
- LrSounds (getSystemSounds, playSystemSound — both require async task)
- LrUndo (undo/redo, canUndo/canRedo)
- LrSlideshow (startSlideshow/stopSlideshow)
- LrPlugin (_PLUGIN global: enabled, id, path, hasResource/resourceId)
- LrPhotoInfo (fileAttributes — returns {width, height} for any supported photo file, not necessarily in catalog)
- LrVideoExportPreset (extension, formatID, name, presetPath)
- LrFilterContext (renditions alias, properties: propertyTable, renditionsToSatisfy, sourceExportSession)
- LrWebViewFactory (specialized factory for web-engine plugins: panel_content, slider_row, checkbox_row, labeled_text_input, identity_plate, etc.)
- Publish classes: LrPublishService (createPublishedCollection, getChildCollections, getPublishSettings, getPluginId — may have ".N" suffix), LrPublishedCollection (getPublishedPhotos, addPhotoByRemoteId, publishNow, setRemoteId/Url), LrPublishedPhoto (getEditedFlag, getPhoto, getRemoteId/Url, setEditedFlag)
- Collection classes: LrCollection (addPhotos, removePhotos, getPhotos, getName, setSearchDescription for smart), LrCollectionSet (getChildCollections/Sets, delete), LrFolder (getPhotos with includeChildren, getPath), LrKeyword (getAttributes — keywordType="person" for face tags SDK 6.0, getChildren, getName, getSynonyms)

---

## Content Preservation Checklist

Every piece of information from the existing files is mapped to a section above:

- All 7 code snippets from SKILL.md → preserved in sections 5-6
- All 11 common mistakes → preserved + 4 new in section 7
- All 15 UXP pitfalls → preserved in section 4 (corrected where needed)
- All 6 LrC memory leak rules → preserved in lrc-sdk.md section 4
- All 35 known issues → preserved in uxp-photoshop.md section 12
- Capability matrix → preserved in section 3
- Busy-guard pattern → promoted to SKILL.md section 4 + preserved in lrc-sdk.md section 4
- External binary bridge pattern → preserved (condensed) in lrc-sdk.md section 4
- Platform detection code → preserved in lrc-sdk.md section 4
- Comparison table → absorbed into capability matrix

---

## Implementation Notes

- Develop parameters use dense "Panel: param1, param2, ..." format (not one-row-per-param tables) to save ~60 lines per file
- Metadata key tables (getRawMetadata, getFormattedMetadata) use compact format grouped by SDK version
- Module quick reference uses inline key-methods format rather than separate tables per module
- Code snippets are concise (5-15 lines) with comments only for non-obvious behavior
- Version annotations use inline format: "method(params) — description. (SDK X.Y+)" rather than separate version columns
