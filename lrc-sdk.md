# Lightroom Classic SDK Reference

Adobe LrC 15.2 SDK. Lua-based plugin system with severe limitations vs. Photoshop UXP — no pixel access, no canvas, no Develop-panel UI, no GC escape hatch.

---

## 1. Plugin Architecture

### Plugin Types
- `.lrdevplugin` directory containing `Info.lua` manifest + Lua scripts
- Plugin types: Export, Publish, Metadata, Web Gallery, Library Filter
- **No Develop module panel type** — cannot add panels to the Develop right panel

### `Info.lua` Manifest
```lua
return {
  LrSdkVersion = 15.0,
  LrSdkMinimumVersion = 6.0,           -- or 11.0 for broad compatibility
  LrToolkitIdentifier = "com.example.chromascope",
  LrPluginName = "Chromascope",
  LrLibraryMenuItems = {
    { title = "Chromascope", file = "ShowChromascope.lua" }
  },
  VERSION = { major=1, minor=0, revision=0, build=1 }
}
```

### Entry Points
- `LrLibraryMenuItems` — Menu items in Library module
- `LrExportMenuItems` — Menu items in File > Plug-in Extras
- `LrExportServiceProvider` — Full export workflow
- `LrMetadataProvider` — Custom metadata fields

---

## 2. Pixel / Image Data Access

**CRITICAL LIMITATION: No direct pixel access from Lua.**

Closest mechanisms:
- **`photo:requestJpegThumbnail(width, height, callback)`** — Async JPEG thumbnail (binary string). No built-in JPEG decoder in Lua.
- **`LrExportSession`** — Export rendered JPEG/TIFF to temp file. Requires external tool to read pixels.
- **`photo:getDevelopSettings()`** — Slider values only, not pixel data.
- **`photo:getRawMetadata(key)`** — File metadata (dimensions, path, color space).

### Workarounds for Pixel Access
1. `requestJpegThumbnail` + external binary to decode JPEG → compute your visualization
2. Export rendition to temp file + external binary to process
3. Use `LrShell` or `LrTasks.execute()` to invoke external tools

---

## 3. UI System (`LrView`)

**No canvas or drawing API. Widget-based only.**

### Available Controls
- **Layout**: `row`, `column`, `view`, `group_box`, `scrolled_view`, `tab_view`, `spacer`, `separator`
- **Input**: `edit_field`, `checkbox`, `radio_button`, `push_button`, `slider`, `popup_menu`, `combo_box`, `simple_list`, `password_field`
- **Display**: `static_text`, `picture` (static image file), `catalog_photo` (thumbnail), `color_well`
- **Data binding**: `LrView.bind()` for two-way binding to `LrObservableTable`

### Image Display Strategy
The `picture` control displays an image file:
```lua
viewFactory:picture { value = "/path/to/scope.jpg", width = 256, height = 256 }
```

For live updates, bind the path:
```lua
viewFactory:picture { value = LrView.bind("scopePath"), width = 256, height = 256 }
-- Update with: properties.scopePath = newPath
```

### Dialogs
- `LrDialogs.presentModalDialog(args)` — Modal dialog
- `LrDialogs.presentFloatingDialog(args)` — Modeless floating window (best for live develop-time feedback)

---

## 4. Develop Module Integration

### `LrDevelopController`
- **`addAdjustmentChangeObserver(context, observer, callback)`** — Fires on any develop slider change
- **`getValue(param)` / `setValue(param, value)`** — Read/write develop parameters
- **`getRange(param)`** — Get min/max for parameter
- **`revealPanel(panelID)`** — Scroll to panel
- **`getSelectedTool()` / `selectTool(tool)`** — Active tool management

### Observable Events
- `addAdjustmentChangeObserver` — develop parameter changes
- **No direct "active photo changed" observer** — must poll `catalog:getTargetPhoto()` via `LrTasks`

---

## 5. Key Modules

| Module | Purpose |
|--------|---------|
| `LrApplication` | App state, active catalog |
| `LrApplicationView` | Module switching, current module |
| `LrCatalog` | Photo access, collections, metadata |
| `LrPhoto` | Photo properties, thumbnails, develop settings |
| `LrDevelopController` | Develop slider control and observation |
| `LrView` | UI widget factory |
| `LrDialogs` | Modal/floating dialogs |
| `LrTasks` | Async tasks, coroutines, `execute()` for shell commands |
| `LrShell` | Launch external apps |
| `LrColor` | Simple RGBA color (no color space conversion) |
| `LrFileUtils` | File operations |
| `LrPathUtils` | Path manipulation |
| `LrLogger` | Debug logging |
| `LrFunctionContext` | Resource management, cleanup handlers |

---

## 6. Recommended Architecture for Live-Updating Develop Plugins

Given the severe limitations, the only viable approach for pixel-based visualizations (chromascope, histogram, etc.):

1. **Floating dialog** opened from Library/Export menu item
2. **External binary** (compiled C/C++/Rust) that:
   - Receives image path or JPEG data
   - Decodes the image
   - Computes the visualization
   - Renders to PNG/JPG file
3. **`picture` control** in floating dialog displays the result
4. **`addAdjustmentChangeObserver`** triggers re-export + re-render
5. **Polling via `LrTasks`** for active photo changes

### Key Challenges
- Latency: export → external process → render → display loop adds delay
- JPEG thumbnails may not reflect current develop settings (cached previews)
- No real-time pixel access like Photoshop UXP
- External binary adds distribution/installation complexity

---

## 7. Memory Leak Prevention (Non-Negotiable)

LrC runs for hours in a long-lived process. These rules were hard-won through debugging a 40GB memory leak:

### 7.1 Frame alternation is mandatory
`f:picture` must receive a *different* file path each update. Writing to the same path causes Lightroom to cache every version internally without releasing.

```lua
local _frameIndex = 0
local function nextScopePath()
  _frameIndex = (_frameIndex + 1) % 2
  return scopeDir .. "/scope_" .. _frameIndex .. ".jpg"
end
```

### 7.2 Guard `requestJpegThumbnail` callbacks
Callbacks may fire more than once. After `done = true`, subsequent callbacks must return immediately. Nil out `jpegData` after writing so it can be GC'd.

```lua
local done = false
photo:requestJpegThumbnail(w, h, function(jpegData, errorMsg)
  if done then return end
  if not jpegData then return end
  done = true
  LrFileUtils.writeFile(tempJpegPath, jpegData)
  jpegData = nil
end)
```

### 7.3 Debounce async tasks with a version counter
Without debouncing, slider drags spawn hundreds of coroutines. Use `_settleVersion` / `_adjustVersion` counters and drop stale callbacks:

```lua
local _settleVersion = 0

local function scheduleRerender()
  _settleVersion = _settleVersion + 1
  local myVersion = _settleVersion
  LrTasks.startAsyncTask(function()
    LrTasks.sleep(0.1)  -- settle window
    if myVersion ~= _settleVersion then return end
    -- ... do the work ...
  end)
end
```

Also coalesce with a busy-guard + pending flag so at most one render is queued:

```lua
local _busy = false
local _pending = false

local function maybeRender()
  if _busy then _pending = true; return end
  _busy = true
  LrTasks.startAsyncTask(function()
    repeat
      _pending = false
      renderOnce()          -- export + decode + render + display
    until not _pending
    _busy = false
  end)
end
```

### 7.4 No unbounded module-level state
Only fixed-size module-level vars: busy flag, frame index, pending flag, settings hash. Never accumulate items in a list or table at module scope.

### 7.5 Clean up temp files on dialog open
Stale temp files from previous runs should be cleared at entry:

```lua
local function cleanup()
  local dir = LrPathUtils.getStandardFilePath('temp') .. '/chromascope'
  if LrFileUtils.exists(dir) then
    LrFileUtils.delete(dir)
  end
  LrFileUtils.createAllDirectories(dir)
end
```

### 7.6 `collectgarbage` is unavailable
LrC sandbox blocks `collectgarbage`. There is no manual-GC escape hatch — rules 7.1–7.5 are the only defense.

---

## 8. External Binary Bridge (Processor CLI Example)

For reference, here's the CLI convention used by chromascope's Rust `processor` binary. The LrC plugin invokes this via `LrTasks.execute()`.

### `processor decode`
Decode a JPEG or TIFF image to raw RGB bytes.

```
processor decode --input <path> --output <path> [--width 256] [--height 256]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--input` | required | Input JPEG/TIFF path |
| `--output` | required | Output raw RGB path |
| `--width` | 256 | Output width in pixels |
| `--height` | 256 | Output height in pixels |

### `processor render`
Render a vectorscope JPEG from raw RGB pixel data.

```
processor render --input <path> --output <path> [options]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--input` | required | Input raw RGB path |
| `--output` | required | Output JPEG path |
| `--width` | 256 | Input image width |
| `--height` | 256 | Input image height |
| `--size` | 512 | Output vectorscope size |
| `--density` | scatter | Mode: scatter, bloom, heatmap |
| `--color-space` | hsl | Space: hsl, ycbcr, cieluv |
| `--scheme` | — | Harmony: complementary, splitComplementary, triadic, tetradic, analogous |
| `--rotation` | 0.0 | Harmony rotation in degrees |
| `--overlay-color` | yellow | Color: white, yellow, cyan, green, magenta, orange |
| `--hide-skin-tone` | false | Hide 123 degree skin tone line |

### Example pipeline
```bash
# Decode a JPEG to 128x128 raw RGB
processor decode --input photo.jpg --output pixels.rgb --width 128 --height 128

# Render a vectorscope with bloom and complementary harmony
processor render --input pixels.rgb --output scope.jpg \
  --width 128 --height 128 --size 512 \
  --density bloom --scheme complementary --rotation 45
```

### Platform binaries
Ship per-platform builds and pick at runtime:
- `bin/macos-arm64/processor`
- `bin/macos-x64/processor`
- `bin/win-x64/processor.exe`

Detect platform in Lua:
```lua
local WIN_ENV = WIN_ENV or (LrShell.pathToMsdosPath ~= nil)
local arch = WIN_ENV and "win-x64" or (os.getenv("HOSTTYPE") == "x86_64" and "macos-x64" or "macos-arm64")
local binary = pluginDir .. "/bin/" .. arch .. "/processor" .. (WIN_ENV and ".exe" or "")
```

---

## 9. Comparison: LrC vs. Photoshop UXP

| Capability | Photoshop UXP | Lightroom Classic |
|---|---|---|
| **Language** | JavaScript (HTML/CSS/JS) | Lua |
| **Pixel Access** | Full read/write via Imaging API | None — rendered export only |
| **UI Framework** | HTML/CSS/JS + Spectrum components | `LrView` declarative widgets |
| **Canvas/Drawing** | Limited 2D canvas; SVG + generated images work | No drawing primitives at all |
| **WebView** | Available with permissions (modal dialogs) | `LrWebViewFactory` — limited |
| **Color Data** | Full pixel arrays, color space conversion, profiling | `LrColor` for develop settings only |
| **Event System** | Rich notification listeners | `LrCatalog:addPropertyChangeObserver`, limited events |
| **Native Code** | Hybrid plugins (C++ bridge) | External tool launched via `LrTasks.execute` |
| **Develop Control** | Direct pixel manipulation, `batchPlay` | `LrDevelopController` for sliders only |
| **Document Model** | Full DOM (Document, Layer, Channel, Path, Selection, History) | Catalog-centric (LrCatalog, LrPhoto, LrCollection) |
| **Plugin Types** | Panel, Command | Export, Publish, Metadata, Web Gallery, Filter |
| **File Access** | Sandboxed with permissions | Full Lua file I/O |

### Key architectural consequence
Photoshop UXP can directly read pixel data from the active document or any layer via `imaging.getPixels()` — this is the fundamental requirement for pixel analysis plugins.

Lightroom Classic SDK has **no equivalent**. Plugins cannot read pixels from Lua. The workaround is: export rendered JPEG/TIFF via `LrExportRendition` or `photo:requestJpegThumbnail`, then decode in an external binary. This makes any pixel-analysis LrC plugin fundamentally a two-process architecture.
