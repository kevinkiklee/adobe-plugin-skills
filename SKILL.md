---
name: adobe-plugin-development
description: Use when building, modifying, or debugging Adobe Photoshop UXP plugins or Lightroom Classic plugins — panel UI, pixel access, develop-module observation, memory management in long-lived LrC processes, hybrid native bridges, UXP HTML/CSS/Canvas limits, and Spectrum widgets
---

# Adobe Plugin Development

Reference for writing Photoshop UXP plugins (JavaScript/HTML) and Lightroom Classic plugins (Lua). Covers what the SDKs allow, what they don't, and the non-obvious pitfalls that burn hours.

## When to Use

Use when:
- Writing or editing a UXP panel/command plugin (`manifest.json`, `entrypoints.setup`, Spectrum `sp-*` widgets)
- Writing or editing a Lightroom Classic plugin (`Info.lua`, `LrView`, `LrDevelopController`)
- Reading pixel data from Photoshop (`imaging.getPixels`)
- Wiring develop-slider observation in LrC
- Displaying generated images inside a UXP panel (no HTML canvas pixel ops!)
- Bridging LrC to a native binary for pixel processing
- Debugging memory leaks in long-lived LrC floating dialogs

Don't use for:
- General web UI — UXP is NOT a browser; different rules apply
- ExtendScript / CEP (legacy Adobe plugin systems) — this covers UXP only

## Core Capability Matrix

| Capability | Photoshop UXP | Lightroom Classic |
|---|---|---|
| Language | JavaScript (HTML/CSS) | Lua |
| Pixel read | `imaging.getPixels()` | **None** — must export rendition + decode externally |
| Pixel write | `imaging.putPixels()` | None |
| UI | HTML/CSS/JS + Spectrum `sp-*` | `LrView` declarative widgets only |
| Canvas/drawing | Limited 2D (no `drawImage`/`getImageData`/`toDataURL`) | None |
| Develop events | `action.addNotificationListener` | `LrDevelopController.addAdjustmentChangeObserver` |
| Plugin types | Panel, Command | Export, Publish, Metadata, Filter, Library menu |
| Develop-panel UI | Panels dock anywhere | **Cannot** add Develop-right-panel UI — only floating dialog |

**Fundamental constraint:** LrC cannot read pixels from Lua. Any LrC plugin needing pixel access must export a thumbnail or rendition and hand it to a compiled external binary (C++/Rust).

## Critical Pitfalls

### UXP: `<canvas>` has no pixel ops
- `<canvas>` exists (v7.0.0+) but has **no** `drawImage`, `getImageData`, `putImageData`, `toDataURL`, `toBlob`.
- To display a generated image: build pixel buffer → `imaging.createImageDataFromBuffer` → `imaging.encodeImageData({ base64: true })` → set `<img src="data:image/jpeg;base64,...">`.
- Windows extra sharp edges: `createLinearGradient`, `createRadialGradient`, `clearRect` may fail entirely.
- Real-render options: software renderer in JS, embedded WebView (modal dialogs only), or hybrid C++ plugin.

### UXP: Imaging API gotchas
- 16-bit range is **0-32768 by default**, not 0-65535. Pass `{ fullRange: true }` to `getData()` for full range.
- **Always call `imageData.dispose()` after `getData()`**. UDT warns at 600MB.
- Pass `targetSize` to leverage Photoshop's pyramid cache — dramatically faster than full-res reads.
- `encodeImageData` only supports **RGB** (not Lab, not Grayscale).
- `getPixels()` usually works without `executeAsModal`, but any pixel *write* requires it.
- Default layout is "chunky" (interleaved `RGBRGB`); planar is `RRGGBB`.

### UXP: Layout is flexbox-only, no animations
- No CSS Grid, no `transform`, no `transition`/`@keyframes`, no `box-shadow`, no `float`.
- `window.devicePixelRatio` always returns 1 — cannot detect HiDPI.
- `hide` panel lifecycle callback never fires; `show` fires only once (PS-57284). Don't rely on them for refresh gating.
- `<label for="id">` doesn't work — wrap the label around the control instead.
- `<option>` needs an explicit `value` attribute.

### LrC: Long-running plugins leak catastrophically
A floating dialog running for hours in LrC's sandbox is unforgiving. Non-negotiable rules:

1. **Frame alternation is mandatory.** `f:picture` must receive a *different* path each update. Writing the same path repeatedly causes LrC to cache every version internally without releasing — this caused a 40GB leak in chromascope. Alternate `scope_0.jpg` / `scope_1.jpg`.
2. **Guard `requestJpegThumbnail` callbacks.** After `done = true`, subsequent callbacks must return immediately. Nil out `jpegData` after writing.
3. **Debounce async tasks with a version counter.** Slider drags spawn hundreds of coroutines without debouncing. Use a `_settleVersion` / `_adjustVersion` counter and drop stale callbacks.
4. **No unbounded module-level state.** Only fixed-size vars: busy flag, frame index, pending flag, settings hash. Never grow a list.
5. **Clean up temp files on dialog open.** LrC sandbox blocks `collectgarbage` — there's no GC escape hatch.

### LrC: Develop module can't have custom panels
LrC SDK exposes Library menu items, export dialogs, metadata fields — **never** a Develop-right-panel UI. The only realistic develop-time feedback loop is a floating dialog (`LrDialogs.presentFloatingDialog`) launched from a Library menu item.

### LrC: No "active photo changed" observer
There's `addAdjustmentChangeObserver` for slider changes, but no event for switching active photo. Poll `catalog:getTargetPhoto()` in a `LrTasks.startAsyncTask` loop with coalescing.

## Quick Reference

### UXP: Read active document pixels
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

### UXP: Display a software-rendered image
```javascript
const imageData = await imaging.createImageDataFromBuffer(buffer, {
  width, height, components: 3, colorSpace: "RGB"
});
const jpegBase64 = await imaging.encodeImageData({ imageData, base64: true });
img.src = "data:image/jpeg;base64," + jpegBase64;
imageData.dispose();
```

### UXP: Refresh on document changes (with idle debounce)
```javascript
const { action, core } = require('photoshop');
await action.addNotificationListener(
  ['set', 'select', 'make', 'delete', 'open'],
  (name, desc) => scheduleRefresh()
);
// Prefer userIdle for heavy re-renders
await core.addNotificationListener('UI', [{ event: 'userIdle' }], refresh);
```

### LrC: Observe develop-slider changes
```lua
LrFunctionContext.callWithContext("observer", function(context)
  LrDevelopController.addAdjustmentChangeObserver(context, {}, function()
    scheduleRerender()
  end)
end)
```

### LrC: Export thumbnail and pipe to external tool
```lua
local done = false
photo:requestJpegThumbnail(w, h, function(jpegData, errorMsg)
  if done then return end          -- mandatory guard
  if not jpegData then return end
  done = true
  LrFileUtils.writeFile(tempJpegPath, jpegData)
  jpegData = nil                    -- release reference
  LrTasks.execute(binaryPath .. " decode --input " .. tempJpegPath .. " ...")
end)
```

### LrC: Live-updating picture display (alternating paths)
```lua
local frameIndex = 0
local function nextScopePath()
  frameIndex = (frameIndex + 1) % 2
  return scopeDir .. "/scope_" .. frameIndex .. ".jpg"
end

viewFactory:picture { value = LrView.bind("scopePath"), width = 256, height = 256 }
-- On update: properties.scopePath = nextScopePath() then write the file there
```

### LrC: Debounced async rerender pattern
```lua
local _settleVersion = 0

local function scheduleRerender()
  _settleVersion = _settleVersion + 1
  local myVersion = _settleVersion
  LrTasks.startAsyncTask(function()
    LrTasks.sleep(0.1)                 -- settle window
    if myVersion ~= _settleVersion then return end  -- superseded
    -- ... export + render ...
  end)
end
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Calling `canvas.toDataURL()` in UXP | Not supported. Use `imaging.encodeImageData({ base64: true })`. |
| Skipping `imageData.dispose()` | Leaks native memory. UDT warns at 600MB. |
| Writing LrC `f:picture` to same path repeatedly | LrC caches every version — massive leak. Alternate two paths. |
| Running `collectgarbage()` in LrC | Blocked by sandbox. Design around it (fixed-size state, path alternation). |
| No debounce on LrC develop observer | Drags spawn hundreds of coroutines. Use version-counter pattern. |
| CSS Grid / `transform` / `transition` in UXP | None supported. Flexbox + immediate visual changes only. |
| `z-index` for overlays in UXP | Flaky. Use DOM order / flex order. |
| 16-bit UXP pixels expecting 0-65535 | Default is 0-32768. Pass `{ fullRange: true }` to `getData()`. |
| Trying to add LrC UI to Develop panel | Not a supported plugin type. Use floating dialog from Library menu. |
| Polling inside `requestJpegThumbnail` callback without `done` guard | Callback may fire more than once. Set and check `done`. |
| Using `<label for="id">` in UXP | Not supported. Wrap the label around the control. |

## Deeper References (in this skill directory)

- **`lrc-sdk.md`** — Full Lightroom Classic SDK reference: plugin types, `Info.lua` manifest, `LrView` widgets, `LrDevelopController` API, full module catalog, LrC chromascope architecture, memory-leak prevention details, and the processor CLI spec for LrC→Rust bridging.
- **`uxp-photoshop.md`** — Full Photoshop UXP reference: Manifest v5, Imaging API full signatures, Spectrum UXP component catalog, HTML/CSS support and limitations, Canvas API limits, event system, `executeAsModal` details, hybrid C++ bridge, and the known-issues catalog.

Load the relevant reference only when writing non-trivial code against those APIs — the quick-ref snippets above cover common cases.
