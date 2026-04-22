# Photoshop UXP Plugin Reference

Comprehensive reference for UXP (Unified Extensibility Platform) plugins targeting Photoshop. Based on UXP 9.2.0 / Manifest v5 / Photoshop 27.4.

UXP is **not** a browser — it has a custom layout engine, limited DOM, and no WebGL/Workers. Don't assume web platform features work.

---

## Building a Panel Plugin

### Plugin Types

- **Panel**: Persistent UI docked in Photoshop. Lifecycle: `create(rootNode)`, `show(rootNode, data)`, `hide(rootNode, data)`, `destroy(rootNode)`, `invokeMenu(menuId)`. Size via `minimumSize`, `maximumSize`, `preferredDockedSize`, `preferredFloatingSize`.
- **Command**: Headless execution from menu. Lifecycle: `run()`, `cancel()`. No persistent UI.

A single plugin can declare multiple entry points of mixed types.

### Entry Point & Loading

- `main` field in manifest points to an HTML file (e.g., `index.html`)
- Photoshop API: `require('photoshop')` — submodules: `.app`, `.core`, `.action`, `.imaging`, `.constants`
- Plugins loaded/debugged through UXP Developer Tool (UDT): Load, Reload, Watch, Unload
- Lifecycle methods have a **300ms timeout** for promise resolution

### Manifest v5 Configuration

```json
{
  "manifestVersion": 5,
  "id": "com.example.myplugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "main": "index.html",
  "host": {
    "app": "PS",
    "minVersion": "23.3.0",
    "data": { "apiVersion": 2, "loadEvent": "use", "enableMenuRecording": true }
  },
  "featureFlags": { "enableSWCSupport": true, "CSSNextSupport": true },
  "entrypoints": [
    {
      "type": "panel",
      "id": "mainPanel",
      "label": { "default": "My Plugin" },
      "minimumSize": { "width": 230, "height": 230 },
      "preferredDockedSize": { "width": 300, "height": 350 },
      "preferredFloatingSize": { "width": 400, "height": 450 },
      "icons": [
        { "width": 23, "height": 23, "path": "icons/icon-light.png", "scale": [1, 2], "theme": ["light", "lightest"] },
        { "width": 23, "height": 23, "path": "icons/icon-dark.png", "scale": [1, 2], "theme": ["dark", "darkest"] }
      ]
    }
  ],
  "icons": [
    { "width": 48, "height": 48, "path": "icons/plugin-icon.png", "scale": [1, 2], "theme": ["darkest", "dark", "medium", "light", "lightest"], "species": ["generic"] }
  ],
  "requiredPermissions": {
    "network": { "domains": [] },
    "clipboard": "readAndWrite",
    "localFileSystem": "request"
  }
}
```

### Permissions

| Permission | Values | Notes |
|---|---|---|
| `network.domains` | string[] | HTTPS domains the plugin can access |
| `clipboard` | `"readAndWrite"` \| `"read"` | Clipboard access level |
| `localFileSystem` | `"request"` \| `"plugin"` \| `"fullAccess"` | File system access |
| `launchProcess` | `{ schemes, extensions }` | Required for `openExternal`/`openPath` |
| `ipc` | `{ enablePluginCommunication }` | Inter-plugin messaging |
| `webview` | `{ allow, domains }` | WebView in panels (v6.4+) and dialogs (v6.0+). Domains optional since v9.0. |
| `enableUserInfo` | boolean | User GUID access (PS 25.1, UXP 7.3+) |

**Feature flags:** `CSSNextSupport` enables box-shadow, transform-origin, scaleX/scaleY, translate. `enableSWCSupport` enables Spectrum Web Components (35 npm packages, locked to v0.37.0 in UXP 8.0). Manifest v5 requires PS 23.3.0+ / UXP 6.0+.

### Panel Lifecycle

```javascript
entrypoints.setup({
  plugin: {
    create() { /* plugin loaded */ },
    destroy() { /* plugin unloaded */ }
  },
  panels: {
    mainPanel: {
      create(rootNode) { /* panel DOM created */ },
      show(rootNode, data) { /* panel becomes visible */ },
      hide(rootNode, data) { /* panel hidden */ },
      destroy(rootNode) { /* panel destroyed */ },
      invokeMenu(menuId) { /* flyout menu item clicked */ }
    }
  }
});
```

All callbacks support Promises with a 300ms timeout.

**Known issues (PS-57284):** `show` fires only once (not on each re-show). `hide` never fires. Don't rely on either for refresh gating.

### Panel Sizing

| Property | Description | Notes |
|----------|-------------|-------|
| `minimumSize` | Smallest allowed dimensions | Host may not honor in all docking configs |
| `maximumSize` | Largest allowed dimensions | Same caveat |
| `preferredDockedSize` | Initial size when docked | Preference, not guaranteed |
| `preferredFloatingSize` | Initial size when floating | Preference, not guaranteed |

Panel scrolling via CSS `overflow: auto/scroll`. Does NOT auto-scroll to keep focused controls visible on macOS. Panel icons required for marketplace: 23x23 with theme variants.

### Theme Awareness

Photoshop CSS variables update with theme (Darkest, Dark, Light, Lightest):

**Colors:** `--uxp-host-background-color`, `--uxp-host-text-color`, `--uxp-host-border-color`, `--uxp-host-link-text-color`, `--uxp-host-text-color-secondary`, `--uxp-host-link-hover-text-color`, `--uxp-host-label-text-color`, `--uxp-host-widget-hover-background-color`, `--uxp-host-widget-hover-text-color`, `--uxp-host-widget-hover-border-color`

**Font sizes:** `--uxp-host-font-size`, `--uxp-host-font-size-smaller`, `--uxp-host-font-size-larger`

```css
body {
  background-color: var(--uxp-host-background-color);
  color: var(--uxp-host-text-color);
  font-size: var(--uxp-host-font-size);
}
```

Use `@media (prefers-color-scheme: dark/light)` for coarse theme detection. Spectrum UXP components are automatically theme-aware.

### Plugin Packaging

`.ccx` format via UDT (File > Package Plugin). Marketplace: submit to Adobe Exchange. Direct: share `.ccx`, users install via UDT. Development: load unpacked folder, Watch mode auto-reloads.

---

## Document & Layer Model

### Document Class

**Properties:**

| Property | Type | R/W | Min Ver | Notes |
|---|---|---|---|---|
| `id` | number | R | 22.5 | Unique document ID |
| `name` / `title` | string | R | 22.5 | Without/with extension |
| `path` | string | R | 22.5 | Empty for unsaved |
| `width` / `height` | number | R | 22.5 | Canvas dimensions in pixels |
| `resolution` | number | R | 22.5 | PPI |
| `mode` | DocumentMode | R | 22.5 | RGB, CMYK, Grayscale, Lab, etc. |
| `colorProfileName` | string | R/W | 22.5 | Returns "None" when type not CUSTOM/WORKING |
| `bitsPerChannel` | BitsPerChannel | R | 22.5 | 8, 16, or 32 |
| `pixelAspectRatio` | number | R/W | 22.5 | Non-square pixel ratio |
| `activeLayer` / `activeLayers` | Layer / Layer[] | R | 22.5/23.0 | Selected layer(s) |
| `layers` | Layers | R | 22.5 | Top-level layer collection |
| `backgroundLayer` | Layer? | R | 22.5 | Background layer if present |
| `activeHistoryBrushSource` | HistoryState | R/W | 24.0 | History brush source |
| `activeHistoryState` | HistoryState | R | 24.0 | Current history state |
| `historyStates` | HistoryStates | R | 24.0 | History state collection |
| `activeChannels` / `channels` | Channel[] / Channels | R/W / R | 23.0 | Channel access |
| `colorSamplers` | ColorSamplers | R | 24.0 | Color sampler points |
| `countItems` | CountItems | R | 24.1 | Count items |
| `guides` | Guides | R | 23.0 | Document guides |
| `pathItems` | PathItems | R | 23.3 | Vector paths |
| `layerComps` | LayerComps | R | 24.0 | Saved layer comp states |
| `selection` | Selection | R | 25.0 | Selection object |
| `histogram` | number[] | R | 23.0 | 256-entry; only valid for RGB/CMYK/INDEXEDCOLOR |
| `saved` | boolean | R | 22.5 | True if no unsaved changes |

**Methods:**

| Method | Description | Min Ver |
|---|---|---|
| `createLayer(options?)` / `createLayerGroup(options?)` | Create pixel layer or group | 22.5 |
| `duplicateLayers(layers, doc?)` | Duplicate; cross-document if `doc` given | 22.5 |
| `flatten()` | Merge all into background | 22.5 |
| `mergeVisibleLayers()` | Merge visible (result is NOT background) | 22.5 |
| `groupLayers(layers)` / `linkLayers(layers)` / `unlinkLayers(layers)` | Organize layers | 22.5/23.3 |
| `crop(bounds, angle?, w?, h?)` / `trim(type?, ...)` / `revealAll()` | Canvas manipulation | 22.5 |
| `resizeCanvas(w, h, anchor?)` / `resizeImage(w, h, res?, method?)` | Resize | 22.5 |
| `rotateCanvas(angle)` / `flipCanvas(direction)` | Transform canvas | 22.5 |
| `save()` / `saveAs` / `close(save?)` / `closeWithoutSaving()` | File operations | 22.5 |
| `convertProfile(profile, intent, ...)` / `changeMode(mode)` | Color management | 22.5 |
| `suspendHistory(callback, name)` | Batch edits as single undo | 22.5 |
| `sampleColor(position)` | Sample color at point (returns SolidColor) | 22.5 |
| `calculations(options)` | Channel calculations | 24.2 |
| `splitChannels()` / `trapColors(width)` | Channel/print ops | 22.5 |
| `applyImageAutoColor()` | Auto color correction | 23.5 |

**Gotchas:** `calculations()` requires one unlocked pixel layer. `suspendHistory` scope limited to single document. `mergeVisibleLayers` result is NOT background (unlike `flatten`).

### Layer Class

**Properties:**

| Property | Type | R/W | Min Ver | Notes |
|---|---|---|---|---|
| `id` | number | R | 22.5 | Unique layer ID |
| `name` | string | R/W | 22.5 | |
| `kind` | LayerKind | R | 22.5 | pixel, text, group, smartObject, etc. |
| `opacity` / `fillOpacity` | number | R/W | 22.5 | 0-100 |
| `blendMode` | BlendMode | R/W | 22.5 | Validation throws errors (v24.2+) |
| `visible` / `locked` | boolean | R/W | 22.5 | |
| `allLocked` / `pixelsLocked` / `positionLocked` / `transparentPixelsLocked` | boolean | R/W | 22.5 | Lock variants |
| `isClippingMask` | boolean | R/W | 22.5 | Releasing affects layers above |
| `isBackgroundLayer` | boolean | R | 22.5 | |
| `bounds` / `boundsNoEffects` | Bounds | R | 22.5 | `{left,top,right,bottom}` / without effects |
| `parent` | Layer? | R | 22.5 | null for top-level layers |
| `document` | Document | R | 22.5 | Owning document |
| `layers` | Layers? | R | 22.5 | Child layers (groups only) |
| `linkedLayers` | Layer[] | R | 23.3 | |
| `textItem` | TextItem? | R | 22.5 | Text properties (text layers only) |
| `smartObject` | SmartObject? | R | 24.1 | |
| `itemIndex` | number | R | 22.5 | Position in layer stack |
| `hasVectorMask` / `hasUserMask` / `hasFilterMask` | boolean | R | 24.0 | |

**Non-Filter Methods:**

| Method | Description | Min Ver |
|---|---|---|
| `delete()` | Delete layer | 22.5 |
| `duplicate(relativeObject?, insertionLoc?)` | Duplicate layer | 22.5 |
| `move(relativeLayer, insertionLoc)` | Move in layer stack | 22.5 |
| `moveAbove(layer)` / `moveBelow(layer)` | Move relative to target | 22.5 |
| `link(layer)` / `unlink()` | Link/unlink layers | 23.3 |
| `translate(horizontal, vertical)` | Move layer content | 22.5 |
| `scale(width, height, anchor?, interp?)` | Scale layer | 22.5 |
| `rotate(angle, anchor?, interp?)` | Rotate layer | 22.5 |
| `flip(axis)` | Flip horizontal/vertical | 22.5 |
| `merge()` | Merge down | 22.5 |
| `rasterize(target)` | Rasterize content | 22.5 |
| `bringToFront()` / `sendToBack()` | Reorder in stack | 22.5 |
| `convertToSmartObject()` | Convert to smart object | 24.2 |

### Layer Filter Methods

All require `executeAsModal`. Grouped by category.

**Blur Filters:**

| Method | Key Parameters | Notes |
|---|---|---|
| `applyGaussianBlur(radius)` | radius: 0.1-250 | |
| `applyMotionBlur(angle, distance)` | angle: -360..360, distance: 1-2000 | |
| `applySmartBlur(radius, threshold, quality, mode)` | radius: 0.1-100 | |
| `applyLensBlur(options?)` | source, focalDist, shape, radius | |
| `applyAverageBlur()` / `applyBlur()` / `applyBlurMore()` | None | |
| `applyRadialBlur(amount, method, quality)` | amount: 1-100, spin/zoom | Not 16/32-bit |

**Sharpen Filters:**

| Method | Key Parameters | Notes |
|---|---|---|
| `applySharpen()` / `applySharpenEdges()` / `applySharpenMore()` | None | |
| `applyUnSharpMask(amount, radius, threshold)` | amount: 0.1-500, radius: 0.1-250, threshold: 0-255 | |

**Noise Filters:**

| Method | Key Parameters | Notes |
|---|---|---|
| `applyAddNoise(amount, distribution, mono)` | amount: 0.1-400 | Not 32-bit |
| `applyMedianNoise(radius)` | radius: 1-500 | |
| `applyDustAndScratches(radius, threshold)` | radius: 1-100, threshold: 0-255 | |
| `applyDespeckle()` | None | |

**Distortion Filters** (most not available for 32-bit):

| Method | Key Parameters |
|---|---|
| `applyDisplace(hScale, vScale, type, undefinedAreas, mapPath)` | scale: -999..999 |
| `applyPinch(amount)` | -100..100 |
| `applyWave(generators, minWavelength, maxWavelength, ...)` | Complex |
| `applyZigZag(amount, ridges, style)` | amount: -100..100 |
| `applyRipple(amount, size)` | amount: -999..999 |
| `applyShear(curve, undefinedAreas)` | Array of {x,y} |
| `applyPolarCoordinates(conversion)` | rectToPolar/polarToRect |
| `applyTwirl(angle)` | -999..999 |
| `applySpherize(amount, mode)` | -100..100 |
| `applyOceanRipple(size, magnitude)` | size: 1-15, mag: 1-20 |

**Effects Filters:**

| Method | Key Parameters | Notes |
|---|---|---|
| `applyHighPass(radius)` | 0.1-250 | |
| `applyClouds()` / `applyDifferenceClouds()` | Uses fg/bg colors | |
| `applyLensFlare(brightness, center, type)` | 10-300 | RGB only |
| `applyDiffuseGlow(graininess, glow, clear)` | | RGB only |
| `applyGlassEffect(distortion, smoothness, ...)` | | RGB only |

**Other Filters:**

| Method | Key Parameters | Notes |
|---|---|---|
| `applyOffset(h, v, undefinedAreas)` | | |
| `applyCustomFilter(matrix5x5, scale, offset)` | 5x5 kernel | |
| `applyDeInterlace(eliminate, create)` | odd/even | Not 32-bit |
| `applyNTSC()` | | RGB only |
| `applyMaximum(radius)` / `applyMinimum(radius)` | 1-500 | |

**Compositing:** `applyImage(options)` — source layer/channel, blending, opacity, mask (v24.5+)

### Selection Class (v25.0+)

**Properties:** `bounds` (Bounds), `solid` (boolean — true if single rectangle), `document`, `parent`, `typename`

**Methods:**

| Method | Description |
|---|---|
| `selectAll()` | Select entire canvas |
| `deselect()` | Remove selection |
| `select(region, type?)` | Select polygon region |
| `selectEllipse(bounds, type?, feather?, antiAlias?)` | Elliptical selection |
| `selectRectangle(bounds, type?, feather?)` | Rectangular selection |
| `selectColumn(x)` / `selectRow(y)` | Single column/row |
| `inverse()` | Invert selection |
| `expand(by)` / `contract(by)` | Expand/contract by pixels |
| `feather(by)` | Feather edge |
| `smooth(radius)` | Smooth selection |
| `grow(tolerance, antiAlias?)` | Grow by color similarity |
| `similar(tolerance, antiAlias?)` | Select similar across canvas |
| `border(width)` | Select border of current |
| `translateBoundary(dX, dY)` | Move selection boundary |
| `rotateBoundary(angle, anchor?)` | Rotate selection boundary |
| `selectColorRange(options)` | Select by color range |
| `saveTo(channel)` / `load(channel, type?)` | Save/load to channel |

Type parameter: `replace` | `diminish` | `extend` | `intersect`

### Other Classes

| Class | Min Ver | Key Members |
|---|---|---|
| LayerComp | 24.0 | name, comment, appearance, position, visibility; `apply()`, `recapture()`, `remove()` |
| ColorSampler | 24.0 | position, color; `move()`, `remove()` |
| CountItem | 24.1 | position; `remove()` |
| PathItem | 23.3 | name, kind, subPathItems; `makeSelection()`, `strokePath()`, `fillPath()` |
| Guide | 23.0 | direction, coordinate; `remove()` |
| HistoryState | 22.5 | name, snapshot |

---

## Working with Pixels

```javascript
const imaging = require('photoshop').imaging;
```

Pixel *writes* require `executeAsModal`. Pixel *reads* via `getPixels()` often work outside modal.

### PhotoshopImageData Object

| Property | Type | Description |
|---|---|---|
| `width`, `height` | number | Dimensions |
| `colorSpace` | string | `"RGB"`, `"Grayscale"`, `"Lab"` |
| `colorProfile` | string | e.g. `"sRGB IEC61966-2.1"` |
| `hasAlpha` | boolean | Alpha channel present |
| `components` | number | 3=RGB, 4=RGBA, 1=Gray |
| `componentSize` | number | 8, 16, or 32 |
| `pixelFormat` | string | `"RGB"`, `"RGBA"`, `"Grayscale"`, `"GrayscaleAlpha"`, `"LAB"`, `"LABAlpha"` |
| `isChunky` | boolean | true=interleaved RGBRGB, false=planar RRGGBB |

**Methods:** `getData(options?)` returns Promise<Uint8Array|Uint16Array|Float32Array> (options: `chunky` default true, `fullRange` default false). `dispose()` synchronous — **always call when done**.

### Component Value Ranges

| Bit Depth | Type | Range | Notes |
|---|---|---|---|
| 8 | Uint8Array | 0-255 | Standard |
| 16 | Uint16Array | 0-32768 default | Photoshop non-standard. `fullRange: true` for 0-65535. |
| 32 | Float32Array | 0.0-1.0+ | HDR may exceed 1.0. 32-bit linear profiles use different naming. |

**Memory layout:** Chunky (default): `[R0,G0,B0, R1,G1,B1, ...]`. Planar: `[R0,R1,..., G0,G1,..., B0,B1,...]`.

### `getPixels(options)`

```javascript
const result = await imaging.getPixels({
  documentID: doc.id,           // optional, defaults to active
  layerID: layer.id,            // optional, omit for composite
  historyStateID: stateId,      // optional
  sourceBounds: { left: 0, top: 0, right: 300, bottom: 300 },
  targetSize: { width: 200, height: 200 },    // downsample via pyramid
  colorSpace: "RGB",
  colorProfile: "sRGB IEC61966-2.1",
  componentSize: 8,             // -1 (source), 8, 16, 32
  applyAlpha: false             // premultiply alpha with white
});
// Returns: { imageData, sourceBounds, level }  (level = pyramid level used)
const pixels = await result.imageData.getData();
result.imageData.dispose();    // MANDATORY
```

**Performance:** `targetSize` leverages Photoshop's image pyramid cache — dramatically faster for previews.

### `putPixels(options)`

```javascript
await imaging.putPixels({
  layerID: layer.id,            // REQUIRED
  documentID: doc.id,           // optional
  imageData: imageData,         // REQUIRED
  replace: true,                // false = blend with existing
  targetBounds: { left: 0, top: 0 },
  commandName: "My Edit"
});
```

### `createImageDataFromBuffer(buffer, options)`

```javascript
const imageData = await imaging.createImageDataFromBuffer(buffer, {
  width: w, height: h, components: c,  // ALL REQUIRED
  colorSpace: "RGB",                     // REQUIRED
  colorProfile: "sRGB IEC61966-2.1",
  chunky: true, fullRange: false
});
```

Buffer element count must equal `width * height * components`.

### `encodeImageData(options)` — for `<img>` display

```javascript
const jpegBase64 = await imaging.encodeImageData({ imageData, base64: true });
img.src = "data:image/jpeg;base64," + jpegBase64;
```

**Only RGB** encodable — not Lab or Grayscale. Without `base64`, returns `Number[]` (raw JPEG bytes).

### Masks & Selection

```javascript
// Layer mask read (single-channel grayscale)
const maskObj = await imaging.getLayerMask({
  layerID: layer.id, kind: "user", // or "vector"
  sourceBounds: { left: 0, top: 0, right: 300, bottom: 300 },
  targetSize: { height: 100 }
});
// Mask write (pixel masks only — kind must be "user", not "vector")
await imaging.putLayerMask({
  layerID: layer.id, kind: "user",
  imageData: grayImageData, replace: true, commandName: "Edit Mask"
});
// Selection read/write
const selObj = await imaging.getSelection({ documentID: doc.id, sourceBounds: {...} });
await imaging.putSelection({ documentID: doc.id, imageData: grayImageData, replace: true });
```

### Point Sampling & Color Profiles

```javascript
const color = await document.sampleColor({ x: 100, y: 100 }); // SolidColor
const rgbProfiles = await require('photoshop').app.getColorProfiles('RGB');
```

### Performance Guidelines

1. Request smallest possible region via `sourceBounds`
2. Use `targetSize` to scale down — leverages pyramid cache
3. `dispose()` **immediately** after extracting pixel data
4. Work in document's native `colorSpace`/`colorProfile` when possible
5. All imaging methods are async — always `await`

---

## batchPlay & Action System

### Action Descriptor Structure

batchPlay executes Photoshop action descriptors — the same commands used by Actions panel recording. Every descriptor has `_obj` (command name), `_target` (reference array), `_options` (execution options).

### Reference Forms

| Form | Syntax | Example |
|---|---|---|
| **ID** | `{ _ref: "layer", _id: 42 }` | Specific layer by ID |
| **Index** | `{ _ref: "layer", _index: 3 }` | Layer at stack position |
| **Name** | `{ _ref: "layer", _name: "Background" }` | Layer by name |
| **Enum** | `{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" }` | Active/first/last |
| **Property** | `{ _ref: "property", _property: "opacity" }` | Specific property |

References chain as arrays (innermost target first):
```javascript
const result = await action.batchPlay([{
  _obj: "get",
  _target: [
    { _ref: "property", _property: "opacity" },
    { _ref: "layer", _enum: "ordinal", _value: "targetEnum" },
    { _ref: "document", _enum: "ordinal", _value: "targetEnum" }
  ],
  _options: { dialogOptions: "silent" }
}], {});
```

### multiGet for Bulk Reads

```javascript
const result = await action.batchPlay([{
  _obj: "multiGet",
  _target: [{ _ref: "layer", _enum: "ordinal", _value: "targetEnum" }],
  extendedReference: [["name", "opacity", "visible", "bounds"]],
  _options: { dialogOptions: "silent" }
}], {});
```

### Error Handling

**batchPlay errors resolve (not reject) with error objects.** Always check: `if (result.message) { ... }`. `batchPlaySync` (v23.1+) available for quick property reads — blocks UI, never for modifications.

### Action Recording (v25.0+)

Enable via `host.data.enableMenuRecording: true` in manifest. Use `action.recordAction({ name, set }, async () => { ... })`. Register handlers for Record Again support.

### Descriptor Discovery

- **Copy As JavaScript**: Actions panel > Copy As JavaScript (PS 2024+)
- **Notification listener**: Log all events to discover descriptor shapes
- **Developer menu**: Preferences > Plugins > Show Developer Menu

### action Module Functions

| Function | Min Ver | Description |
|---|---|---|
| `addNotificationListener(events, cb)` | 23.0 | Subscribe to action events |
| `removeNotificationListener(events, cb)` | 23.0 | Unsubscribe |
| `batchPlay(descriptors, options)` | 22.5 | Execute descriptors async |
| `batchPlaySync(descriptors, options)` | 23.1 | Execute synchronously (blocks UI) |
| `getIDFromString(string)` | 22.5 | Convert string to runtime ID |
| `recordAction(options, callback)` | 25.0 | Record actions |
| `validateReference(ref)` | 22.5 | Check reference validity |

---

## executeAsModal

Required for any operation that modifies Photoshop state.

```javascript
const { core } = require('photoshop');

async function myDocumentEdit(executionContext, descriptor) {
  if (executionContext.isCancelled) throw "User cancelled";
  executionContext.reportProgress({ value: 0.5, commandName: "Processing..." });
  const suspensionID = await executionContext.hostControl.suspendHistory({
    documentID: app.activeDocument.id, name: "My Edit"
  });
  // ... do work ...
  await executionContext.hostControl.resumeHistory(suspensionID, {
    finalName: "Completed Edit"  // optional: rename history state
  });
}

try {
  await core.executeAsModal(myDocumentEdit, {
    commandName: "My Edit",
    descriptor: { myParam: "value" },
    interactive: false,
    timeOut: 5000  // v25.10+: retry duration for modal collisions
  });
} catch (e) {
  if (e.number === 9) console.log("Another plugin holds modal state");
}
```

### ExecutionContext

| Property/Method | Description |
|---|---|
| `isCancelled` | Check if user pressed Escape |
| `onCancel` | Register cancellation callback |
| `reportProgress({ value, commandName })` | Update progress bar (0-1) |
| `hostControl.suspendHistory(options)` | Batch edits as single undo step |
| `hostControl.resumeHistory(id, options?)` | End suspension; `finalName` renames state |
| `hostControl.registerAutoCloseDocument(docId)` | Auto-close on modal exit |
| `hostControl.unregisterAutoCloseDocument(docId)` | Cancel auto-close |

### Key Behaviors

- Only one plugin holds modal state at a time; menu items disabled during modal
- Progress bar appears after 2+ seconds; Escape cancels
- Event notifications suppressed during non-interactive modal scope
- Nested modal scopes share global modal state
- PS only creates a history state if the document was actually modified
- **Modal collision retry** (v25.10+): with `timeOut`, requests retry instead of immediate error 9
- `getPixels()` is read-only and often works outside modal

### Exception Anti-Pattern

**Never swallow exceptions inside executeAsModal** — this prevents cancellation from propagating:

```javascript
// BAD: catch swallows cancel signal
try { await action.batchPlay([...], {}); } catch(e) { }
// GOOD: check isCancelled, let errors propagate
if (ctx.isCancelled) throw "Cancelled";
await action.batchPlay([...], {});
```

### Modal Event Monitoring

```javascript
await action.addNotificationListener(
  ['modalJavaScriptScopeEnter', 'modalJavaScriptScopeExit'], handler
);
```

---

## Building UIs

### HTML Element Support

Subset of standard HTML. Confirmed working:

**Structural:** `<html>`, `<head>`, `<body>`, `<script>`, `<style>`, `<link>`, `<div>`, `<span>`, `<p>`, `<a>`, `<h1>`-`<h6>`, `<hr>`, `<footer>`

**Form:**
- `<input>` — types: `text`, `password`, `number`, `range`, `checkbox`, `radio`, `search`. NOT: `file`, `color`.
- `<select>` — `<option>` requires explicit `value` or `select.value` returns undefined.
- `<textarea>` — size via CSS only, not `rows`/`cols`.
- `<button>`, `<progress>` (not theme-aware).
- `<label>` — **`for="id"` NOT supported**; wrap label around control.
- `<form>` — only `method="dialog"`.

**Media:**
- `<img>` — needs explicit dimensions in dialogs. Ignores EXIF rotation. Grayscale fails to render.
- `<video>` — full API (v7.4+). `timeUpdate` event (v9.1+). `<audio>` basic support (v9.1+).
- `<canvas>` — v7.0.0+ with many limitations (see Canvas API below).
- `<webview>` — panels (v6.4+/PS 24.1+) and dialogs (v6.0+). Local HTML (v8.0+). Domains optional (v9.0+). Script injection (v9.0.2+). DnD (v9.1+).

**Other:** `<dialog>` (modal/non-modal, `showModal()` returns Promise), `<template>`, `<slot>`, `<menu>`, `<menuitem>`.

**NOT supported (treated as `<div>`):** `<ul>`, `<ol>`, `<li>`, `<i>`, `<em>`, `<title>`, `<iframe>` (use `<webview>`).

**Unsupported attributes:** `aria-*`, `contenteditable`, `draggable`/`dropzone`, `hidden`, `spellcheck`. `tabindex` partial. Use `addEventListener` not inline handlers. Unitless dimensions don't work (v3.1.0+).

### CSS Support

#### Supported

**Layout (flexbox-based):** `display` (none/inline/block/inline-block/flex/inline-flex), `flex`/`flex-basis`/`flex-direction`/`flex-grow`/`flex-shrink`/`flex-wrap`, `align-content`/`align-items`/`align-self`/`justify-content`, `position` with offsets, `overflow`/`overflow-x`/`overflow-y`

**Box model:** `width`/`height`/`min-*`/`max-*`, `margin`, `padding`, `border` (+ directional), `border-radius` (+ corner-specific)

**Typography:** `font-family`, `font-size`, `font-style`, `font-weight`, `color`, `letter-spacing`, `text-align`, `text-overflow`, `white-space`

**Background:** `background`, `background-color`, `background-image` (`url('plugin://assets/...')`, `linear-gradient()` v8.1+, `radial-gradient()`), `background-size`, `background-attachment`. `background-repeat` declared but non-functional. **Visual:** `opacity`, `visibility`

**Units:** `px`, `em`, `rem`, `vh`, `vw`, `vmin`, `vmax`, `cm`, `mm`, `in`, `pc`, `pt`

**Features:** `calc()`, CSS Variables, `var()`, `prefers-color-scheme`

**Selectors:** type, class, id, `*`, attribute, child `>`, descendant, adjacent `+`, general sibling `~`

**Pseudo-classes:** `active`, `checked`, `defined`, `disabled`, `empty`, `enabled`, `first-child`, `focus`, `hover`, `last-child`, `nth-child`, `nth-last-child`, `nth-of-type`, `only-child`, `root`

**Pseudo-elements:** `::after`, `::before`. **Media queries:** `width`, `height`, `prefers-color-scheme`

#### Supported with CSSNextSupport Feature Flag

Requires `"featureFlags": { "CSSNextSupport": true }` in manifest. Without this flag, these properties are **silently ignored**:

- `box-shadow`
- `transform-origin`
- `scaleX`, `scaleY`
- `translate`

#### NOT Supported

| Feature | Status |
|---|---|
| `font` shorthand | Use individual `font-*` |
| `text-transform` | No uppercase/lowercase/capitalize |
| `transition`, `@keyframes` | No animations |
| `grid` layout | Flexbox only |
| `position: sticky` | Not supported |
| `z-index` | Limited support |
| `float`, `clear` | Not supported |
| `cursor` | Limited; may not revert |
| `background-repeat` | Declared but non-functional |
| `baseline` alignment | Buggy/unreliable |
| `text-decoration` | Limited |

UXP uses a custom layout engine, **not a browser engine**. Flexbox is the primary mechanism.

### Canvas API (v7.0.0+)

```javascript
const canvas = document.createElement('canvas');
canvas.width = 400; canvas.height = 400;
const ctx = canvas.getContext('2d'); // Only '2d' supported
```

**Available:** `lineWidth`, `lineJoin`, `lineCap`, `globalAlpha`, `fillStyle`, `strokeStyle`; paths (`beginPath`, `closePath`, `moveTo`, `lineTo`, `arc`, `arcTo`, `bezierCurveTo`, `quadraticCurveTo`, `rect`); drawing (`fill`, `stroke`, `fillRect`, `strokeRect`, `clearRect`); gradients (`createLinearGradient`, `createRadialGradient`); `Path2D`.

**NOT available:** `toDataURL`, `toBlob`, `drawImage`, `getImageData`, `putImageData`, `createImageData`, `fillText`, `strokeText`, `measureText`, `save`, `restore`, transforms (`translate`, `rotate`, `scale`, `setTransform`), `clip`, `createPattern`, `globalCompositeOperation`, shadow properties, `imageSmoothingEnabled`, WebGL.

**Windows breakage:** `clearRect`, `createLinearGradient`, `createRadialGradient` may fail.

**Workarounds:** software renderer -> `createImageDataFromBuffer` -> `encodeImageData({ base64: true })` -> `<img src="data:...">`. SVG for vector plots. WebView (panels v6.4+ or dialogs). Hybrid C++ plugin.

### Spectrum UXP Components

Native Photoshop-styled widgets following Adobe's Spectrum design system. Since UXP v4.1. Components are automatically theme-aware.

#### `sp-button`

```html
<!-- Variants: cta (default), primary, secondary, warning, overBackground -->
<sp-button variant="primary">Click Me</sp-button>
<sp-button variant="warning" quiet>Delete</sp-button>
<sp-button variant="primary">
  <sp-icon name="ui:Magnifier" size="s" slot="icon"></sp-icon>
  Search
</sp-button>
```

Event: `click`. `sp-action-button`: same API with `quiet`, `selected` attributes.

#### `sp-checkbox`

```html
<sp-checkbox checked>Pre-checked</sp-checkbox>
<sp-checkbox indeterminate>Partial</sp-checkbox>
```

Events: `change`, `input` — access `evt.target.checked`

#### `sp-dropdown`

```html
<sp-dropdown placeholder="Select..." style="width: 320px">
  <sp-menu slot="options">
    <sp-menu-item>Option 1</sp-menu-item>
    <sp-menu-item disabled>Option 2 (disabled)</sp-menu-item>
    <sp-menu-divider></sp-menu-divider>
    <sp-menu-item>Option 3</sp-menu-item>
  </sp-menu>
</sp-dropdown>
```

Event: `change` — access `evt.target.selectedIndex`
**Known issues:** needs explicit width or displays oddly. Doesn't respond to arrow keys.

#### `sp-slider`

```html
<sp-slider min="0" max="100" value="50">
  <sp-label slot="label">Opacity</sp-label>
</sp-slider>
<sp-slider min="0" max="360" value="0" variant="filled" show-value>
  <sp-label slot="label">Rotation</sp-label>
</sp-slider>
```

Attributes: `min`, `max`, `value`, `disabled`, `variant="filled"`, `fill-offset`, `value-label`, `show-value`
Events: `input` (during drag), `change` (on release) — access `evt.target.value`

#### `sp-textfield`

```html
<sp-textfield placeholder="Enter text">
  <sp-label isrequired="true" slot="label">Name</sp-label>
</sp-textfield>
<sp-textfield quiet placeholder="Search..."></sp-textfield>
```

Types: `text` (default), `numeric` (-214748.36 to 214748.36), `search`, `password`
**Known issue:** Password value unreadable on macOS. Workaround: toggle type between `password`/`text` on focus/blur.

#### `sp-textarea` / `sp-radio-group`

```html
<sp-textarea placeholder="Enter description...">
  <sp-label slot="label">Description</sp-label>
</sp-textarea>

<sp-radio-group>
  <sp-label slot="label">Color Space:</sp-label>
  <sp-radio value="ycbcr">YCbCr BT.601</sp-radio>
  <sp-radio value="hsl">HSL</sp-radio>
</sp-radio-group>
```

`sp-radio-group` event: `change` — access `evt.target.value`. Add `column` for vertical layout.

#### Other Components

| Component | Key Attributes | Events/Notes |
|---|---|---|
| `sp-menu` / `sp-menu-item` | disabled; `sp-menu-divider` | `change` (.selectedIndex). Divider only renders in dropdown. |
| `sp-icon` | name="ui:...", size (xxs-xxl) | 40 built-in icons |
| `sp-progressbar` | max, value, show-value, size | |
| `sp-divider` | size (large/medium/small) | |
| `sp-link` | href, quiet | href navigation can't be prevented |
| `sp-heading` | size (XXS-XXXL) | |
| `sp-body` | size (XS-XL) | |
| `sp-detail` / `sp-label` | size / isrequired, slot="label" | |
| `sp-tooltip` | | `location` controls tip direction, not position |
| `sp-action-menu` | | `change` |

40 built-in icons: chevrons, arrows, checkmarks, alerts, magnifier, cross, star, folder, info, question, settings, etc.

#### Spectrum Web Components (SWC)

SWC provides 35 npm packages with additional components not available as `sp-*` elements. Requires `"enableSWCSupport": true` in manifest `featureFlags`. Locked to v0.37.0 in UXP 8.0.

SWC-only components include: accordion, avatar, badge, banner, card, dialog, field-group, illustrated-message, meter, number-field, picker, popover, search, sidenav, split-button, status-light, swatch, switch, table, tabs, toast, tooltip, top-nav, tray.

#### React + Spectrum

Custom element events don't flow through React's synthetic event system. Use refs + `addEventListener`:

```jsx
function MySlider() {
  const ref = useRef(null);
  useEffect(() => {
    const handler = (e) => console.log(e.target.value);
    ref.current.addEventListener('input', handler);
    return () => ref.current.removeEventListener('input', handler);
  }, []);
  return <sp-slider ref={ref} min="0" max="100" value="50" />;
}
```

### Event System

#### Action Notification Listener

Document-modifying events (layer changes, selections, filters). Since v23.0. 198 action events available.

```javascript
const { action } = require('photoshop');

await action.addNotificationListener(
  ['open', 'close', 'save', 'select', 'make', 'set', 'delete'],
  (eventName, descriptor) => {
    console.log(`Event: ${eventName}`, descriptor);
  }
);

await action.removeNotificationListener(['open', 'close', 'save'], handler);
```

**Common events:** layer ops (merge, duplicate, group, ungroup), selection changes, pixel mods (blur, sharpen, levels, curves), layer effects, transforms (rotate, flip, crop, resize).

#### Core Notification Listener

UI and OS events (v23.3+):

```javascript
const { core } = require('photoshop');

await core.addNotificationListener('UI', [{ event: 'userIdle' }], (event, descriptor) => {
  if (descriptor.idleEnd) console.log('User returned from idle');
});

await core.addNotificationListener('OS', ['activationChanged'], handler);
```

**UI events:** `userIdle`, `minimizeAppWindow`, `panelVisibilityChanged`, workspace events. **OS events:** `activationChanged`, `displayConfigurationChanged`.

**`userIdle` is the right trigger for deferred heavy re-renders** — fires after the user stops interacting.

**Important:** Event notifications are suppressed during non-interactive `executeAsModal` scope.

#### DOM Events

Standard DOM events work with limits:
Supported: `click`, `change`, `input`, `focus`, `blur`, `mousedown`/`mouseup`/`mousemove`/`mouseenter`/`mouseleave`, `pointerdown`/`pointerup`/`pointermove`, `keydown`/`keyup`, `wheel`, `resize` (via ResizeObserver), `load` (images).

NOT supported: `keypress`, `dragstart`/`drag`/`dragend`/`drop` (DnD fully unsupported). Interactive elements swallow most events — clicks on buttons don't bubble to parents.

#### Observers

Both `IntersectionObserver` and `ResizeObserver` are available.

```javascript
const observer = new ResizeObserver(entries => {
  for (const entry of entries) {
    console.log(entry.contentRect.width, entry.contentRect.height);
  }
});
observer.observe(myElement);
```

`MutationObserver` is **not** available.

### DOM Limitations

#### Missing / Limited APIs

| API | Status |
|---|---|
| `document.createElement('iframe')` | Use `<webview>` in panels (v6.4+) or dialogs |
| `window.devicePixelRatio` | Always returns 1 — cannot detect HiDPI |
| `localStorage` / `sessionStorage` | **Available** as global storage APIs. NOT inside WebView with local content. |
| `XMLHttpRequest` cookies | Not supported |
| `XMLHttpRequest` Blob send | Use ArrayBuffer |
| `WebSocket` extensions | Not supported |
| `fetch` | Available with limits |
| `CustomEvent` | Available |
| `CustomElementRegistry` | Available |
| `ShadowRoot` | Available |
| `TreeWalker` | Available |
| `MutationObserver` | **Not available** |
| `requestAnimationFrame` | Limited |
| Web/Service Workers | Not available |
| `IndexedDB` | Not available |
| `WebGL` | Not available |
| `WebAssembly` | Not available |
| `URLSearchParams` | Available (v9.1+) |
| `Navigator.language` | Available (v26.0+) |
| `HTMLElement.append/prepend/replaceChildren` | Available (v26.0+) |
| `SecureStorage` | Web Storage API compliant (v9.0+) |

System sleep/awake events available (v27.4+).

#### Available DOM APIs

`Document`, `DocumentFragment`, `Element`, `Node`, `Text`, `Comment`; `querySelector(All)`, `getElementById`; `createElement`, `createTextNode`, `createDocumentFragment`; `appendChild`, `removeChild`, `insertBefore`, `replaceChild`; `setAttribute`, `getAttribute`, `removeAttribute`; `classList`; `innerHTML`, `outerHTML`, `textContent`, `innerText`; `clientWidth`/`Height`, `offsetWidth`/`Height`, `scrollLeft`/`Top`/`Width`/`Height`; `getBoundingClientRect`; `getComputedStyle`; `AbortController`/`AbortSignal`; `alert`/`confirm`/`prompt`.

#### Dialog Behavior

- Closed dialogs remain in DOM — must call `element.remove()`
- `showModal()` returns a Promise (not void like browsers)
- Only one dialog at a time
- `HTMLDialogElement.REJECTION_REASON_NOT_ALLOWED` — another dialog already open
- `HTMLDialogElement.REJECTION_REASON_DETACHED` — node removed from DOM

#### `innerHTML` Caveat

Inline event handlers (`onclick="..."`) in `innerHTML` are parsed but behavior is unreliable. Always use `addEventListener`.

---

## Text & Typography

### TextItem Class

Available on text layers via `layer.textItem`.

**Properties:** `contents` (R/W string), `isParagraphText` (R), `isPointText` (R), `orientation` (R/W), `textClickPoint` (R, `{x, y}`)

**Methods:** `convertToParagraphText()`, `convertToPointText()`, `convertToShape()`, `createWorkPath()`

### CharacterStyle

34 properties via `textItem.characterStyle`: `font`, `size`, `fauxBold`, `fauxItalic`, `tracking`, `leading`, `autoLeading`, `baselineShift`, `color` (SolidColor), `antiAliasMethod`, `horizontalScale`, `verticalScale`, `ligatures`, `oldStyle`, `noBreak`, `strikethrough`, `underline`, `capitalization`, `baselineDirection`, `digitSet`, `kashidas`, `justificationAlternates`, `diacriticPosition`, `diacVPos`, `markYDistFromBaseline`, `otfStylisticAlternate`, `otfContextualAlternate`, `otfSwash`, `otfOrdinals`, `otfFractions`, `otfTitling`, `otfSlashedZero`, `otfHistorical`, `otfJustificationAlternate`

### ParagraphStyle

14 properties via `textItem.paragraphStyle`: `justification` (left/center/right/justifyLeft/Center/Right/All), `hyphenation`, `firstLineIndent`, `startIndent`, `endIndent`, `spaceBefore`, `spaceAfter`, `autoHyphenate`, `minimumWordScaling`, `maximumWordScaling`, `desiredWordScaling`, `kinsoku`, `mojikumi`, `burasagari`

### WarpStyle

Via `textItem.warpStyle`: `style` (16 types: arc, arcLower, arcUpper, arch, bulge, flag, fish, fisheye, inflate, none, rise, shellLower, shellUpper, squeeze, twist, wave), `bend` (-100..100), `direction` (horizontal/vertical), `horizontalDistortion` (-100..100), `verticalDistortion` (-100..100)

---

## Color Management

### SolidColor Class

```javascript
const color = new (require('photoshop').app.SolidColor)();
color.rgb.red = 255; color.rgb.green = 128; color.rgb.blue = 0;
app.foregroundColor = color;  // v24.2+
```

**Accessors:** `.rgb`, `.cmyk`, `.hsb`, `.lab`, `.gray`, `.nearestWebColor`. **Method:** `isEqual(other)`.

### Color Classes

| Class | Properties | Ranges |
|---|---|---|
| RGBColor | red, green, blue | 0-255 |
| CMYKColor | cyan, magenta, yellow, black | 0-100 |
| HSBColor | hue, saturation, brightness | hue 0-360, sat/bright 0-100 |
| LabColor | l, a, b | l 0-100, a/b -128..127 |
| GrayColor | gray | 0-100 |

### Color Conversion

```javascript
const labColor = core.convertColor(rgbColor, "labColor");        // v23.0+
const profiles = await app.getColorProfiles('RGB');               // v24.1+
```

---

## External Communication

### Network APIs

**fetch** (recommended): standard API with HTTPS domain whitelisting. FormData multipart (v7.3+).

**XMLHttpRequest:** no cookies, binary via ArrayBuffer only (not Blob).

**WebSocket:** available; extensions unsupported; self-signed certs fail on macOS.

### File System

```javascript
const fs = require('uxp').storage.localFileSystem;
```

Permissions: `"plugin"` (own folder), `"request"` (user-picked), `"fullAccess"` (marketplace approval required). `fs.createReadStream()` (v8.2+).

### WebView

Panels (v6.4+/PS 24.1+), dialogs (v6.0+). Local HTML (v8.0+). Domains optional (v9.0+). Script injection (v9.0.2+). DnD (v9.1+). `localStorage`/`sessionStorage` NOT available in WebView with local content.

```json
"webview": { "allow": "yes", "domains": ["https://example.com"] }
```

### Hybrid Plugins (C++ Bridge)

Combine JS with native C++ (`.dylib`/`.dll`). Use for: high-performance pixel processing, custom rendering, relaxed sandbox, Photoshop C++ SDK (`PIUXPSuite`). Entry: `export SPErr PSDLLMain(const char* selector, SPBasicSuite* basicSuite, PIActionDescriptor descriptor);`

**JS->C++:** `core.sendSDKPluginMessage("NativeID", data)` / C++ `AddUXPMessageListener`
**C++->JS:** C++ `SendUXPMessage(ref, pluginId, desc)` / JS `core.addSDKMessagingListener(cb)`

Build: Hybrid Plugin SDK from Adobe Developer Console. macOS unsigned need security approval. Debug: JS via UDT, C++ via debugger attach.

---

## App & Core APIs

### Photoshop App Class

`require('photoshop').app`

**Properties:**

| Property | Type | R/W | Min Ver | Description |
|---|---|---|---|---|
| `activeDocument` | Document | R/W | 22.5 | Currently active document |
| `foregroundColor` | SolidColor | R/W | 24.2 | Foreground color |
| `backgroundColor` | SolidColor | R/W | 24.2 | Background color |
| `currentTool` | string | R | 22.5 | Active tool name |
| `documents` | Documents | R | 22.5 | Open document collection |
| `fonts` | TextFonts | R | 22.5 | Available fonts |
| `preferences` | Preferences | R | 24.0 | Application preferences |
| `actionTree` | ActionSet[] | R | 23.0 | Recorded action sets |
| `displayDialogs` | DialogModes | R/W | 22.5 | Dialog display mode |

**Methods:**

| Method | Description | Min Ver |
|---|---|---|
| `createDocument(options?)` | Create new document | 22.5 |
| `open(entry?)` | Open file or file picker | 22.5 |
| `bringToFront()` | Bring Photoshop to front | 22.5 |
| `showAlert(message)` | Display alert dialog | 22.5 |
| `convertUnits(value, from, to)` | Convert between units | 23.4 |
| `getColorProfiles(mode)` | List color profiles for mode | 24.1 |
| `updateUI()` | Force UI refresh (no effect outside tracking contexts) | 26.0 |

### Core Module

`require('photoshop').core`

| Function | Description | Min Ver |
|---|---|---|
| `executeAsModal(callback, options)` | Enter modal editing scope | 22.5 |
| `convertColor(sourceColor, targetModel)` | Convert between color models | 23.0 |
| `convertGlobalToLocal(point)` | Screen to panel coords | 26.0 |
| `getActiveTool()` | Get active tool info | 22.5 |
| `performMenuCommand(options)` | Execute menu command by ID | 22.5 |
| `getLayerTree(options?)` / `getLayerTreeSync(options?)` | Layer hierarchy | 22.5 |
| `getLayerGroupContents(opts)` / `getLayerGroupContentsSync(opts)` | Group children | 22.5 |
| `isModal()` | Check if in modal state | 22.5 |
| `historySuspended()` | Check if history suspended | 22.5 |
| `calculateDialogSize(options)` | Compute dialog dimensions | 22.5 |
| `getCPUInfo()` / `getGPUInfo()` | Hardware information | 22.5 |
| `setExecutionMode(mode)` | Set plugin execution mode | 22.5 |
| `translateUIString(key)` | Localize string | 22.5 |
| `redrawDocument(docId)` | Force document redraw | 22.5 |
| `addSDKMessagingListener(callback)` | Listen for C++ bridge messages | 22.5 |
| `sendSDKPluginMessage(id, data)` | Send to C++ bridge | 22.5 |

### Preferences API (v24.0+)

13 groups via `app.preferences`: `cursors`, `fileHandling`, `general`, `guidesGridsAndSlices`, `history`, `interface`, `performance`, `plugins`, `tools`, `transparency`, `type`, `units`, `workspace`

### Prototype Extensions

5 extensible classes: `Photoshop` (`app.__proto__`), `Document`, `Layer`, `ActionSet`, `Action`. Add via prototype:

```javascript
Document.prototype.getVisibleLayers = function() {
  return this.layers.filter(l => l.visible);
};
```

---

## Known Issues

**Note:** Adobe's known issues page is not systematically updated for fixed items. Test against your target PS version.

### Critical
1. **Canvas APIs broken on Windows:** `createLinearGradient`, `createRadialGradient`, `clearRect` may fail
2. **No `toDataURL`/`toBlob`** — cannot export canvas content directly
3. **No `drawImage`** — cannot composite images on canvas
4. **No `getImageData`/`putImageData`** — cannot read/write individual pixels on canvas
5. **`window.devicePixelRatio` always returns 1** — cannot detect HiDPI
6. **Panel `show`/`hide` unreliable:** `show` fires once only, `hide` never fires (PS-57284)
7. **No CSS transitions/animations** — all visual changes immediate
8. **No CSS Grid** — flexbox only (box-shadow/transforms available with `CSSNextSupport` flag)

### UI
9. Text fields always render above other content — no element can overlay them
10. Complex SVG may fail or render unexpectedly
11. Grayscale `<img>` fails to render but occupies DOM space
12. `<img>` in dialogs needs explicit width/height or dialog resizes incorrectly
13. `<img>` ignores embedded EXIF rotation
14. `sp-dropdown` needs explicit width
15. `sp-tooltip` `location` controls tip direction, not position
16. Numeric `sp-textfield` range: -214748.36 to 214748.36
17. DnD fully unsupported
18. Scroll views don't auto-scroll to focused controls (macOS)
19. `<label for="id">` unsupported — wrap label around control
20. `<option>` requires explicit `value`
21. `<option>` doesn't support `disabled`
22. HTML5 input validation unsupported
23. `<textarea>` size via CSS only, not `rows`/`cols`
24. `<form>` only supports `method="dialog"`

### Network
25. Self-signed certs unsupported for secure WebSockets on macOS
26. WebSocket extensions unsupported
27. XHR can only send binary via ArrayBuffer, not Blob
28. XHR doesn't support cookies

### Other
29. Closed dialogs remain in DOM — must call `remove()`
30. `entrypoints.setup()` delayed >20ms may cause uncatchable errors
31. Clipboard APIs throw in panel-less plugins
32. `keypress` unsupported — use `keydown`/`keyup`
33. Password field values unreadable on macOS
34. `font` shorthand CSS unsupported
35. `text-transform` CSS unsupported
36. **Workaround:** `lockDocumentFocus: true` in dialog options prevents focus escape from dialog modal state
