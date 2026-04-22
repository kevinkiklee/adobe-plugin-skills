# Lightroom Classic SDK Reference

Adobe LrC 15.3 SDK. Lua 5.1.5 sandbox. Use-case organized deep reference for plugin development.

**Load this file when:** building export/publish services, working with catalog/photo APIs, building plugin UIs, wiring develop-parameter observation, or needing the full develop parameter list.

---

## 1. Building a Plugin

### Info.lua Manifest — Complete Field Reference

Every plugin requires an `Info.lua` file at the root of its `.lrdevplugin` directory. This file runs in a restricted environment — only `string` namespace, `LOC()`, `WIN_ENV`/`MAC_ENV`, and `_VERSION` are available. No `import` or `require`.

```lua
return {
  LrSdkVersion = 15.0,
  LrSdkMinimumVersion = 6.0,
  LrToolkitIdentifier = "com.example.myplugin",
  LrPluginName = "My Plugin",
  LrLibraryMenuItems = {
    { title = "My Plugin", file = "ShowDialog.lua" }
  },
  VERSION = { major=1, minor=0, revision=0, build=1 }
}
```

| Field | Type | Req | SDK | Notes |
|---|---|---|---|---|
| LrSdkVersion | number | yes | 1.3 | Target SDK version (e.g. 15.0) |
| LrSdkMinimumVersion | number | no | 1.3 | Minimum SDK the plugin supports |
| LrToolkitIdentifier | string | yes | 1.3 | Unique reverse-domain ID |
| LrPluginName | string | yes | 2.0 | Display name in Plugin Manager |
| LrPluginInfoProvider | string | no | 2.0 | Script for Plugin Manager dialog customization |
| LrPluginInfoUrl | string | no | 2.0 | URL shown in Plugin Manager |
| LrInitPlugin | string | no | 2.0 | Script run when plugin loads (on-demand or startup) |
| LrForceInitPlugin | boolean | no | 4.0 | If true, LrInitPlugin runs at app startup |
| LrShutdownPlugin | string | no | 3.0 | Script on user-initiated unload |
| LrShutdownApp | string | no | 4.0 | Script on app exit (10-second timeout) |
| LrEnablePlugin | string | no | 3.0 | Script when plugin is re-enabled |
| LrDisablePlugin | string | no | 3.0 | Script when plugin is disabled |
| LrExportMenuItems | table | no | 1.3 | File > Plug-in Extras menu items |
| LrLibraryMenuItems | table | no | 1.3 | Library > Plug-in Extras menu items |
| LrHelpMenuItems | table | no | 3.0 | Help > Plug-in Extras menu items |
| LrExportServiceProvider | table | no | 1.3 | Export/publish service definitions |
| LrExportFilterProvider | table | no | 2.0 | Export filter (post-process) definitions |
| LrMetadataProvider | string | no | 2.0 | Custom metadata definition script |
| LrMetadataTagsetFactory | table | no | 3.0 | Custom metadata tagset(s) |
| URLHandler | function | no | 4.0 | Custom `lightroom://` URL handling |
| VERSION | table | no | 1.3 | `{ major, minor, revision, build }` |
| LrAlsoUseBuiltInTranslations | boolean | no | 3.0 | Fall back to SDK's built-in translations |
| LrLimitNumberOfTempRenditions | number | no | 5.0 | Max concurrent renditions during export |

### Plugin Lifecycle

**Loading sequence:**
1. `Info.lua` is evaluated (restricted environment)
2. If `LrForceInitPlugin = true`, `LrInitPlugin` script runs at app startup
3. Otherwise, `LrInitPlugin` runs on-demand when plugin is first needed
4. Menu item scripts run when user selects them

**Shutdown protocol:**
`LrShutdownApp` script must return a table:
```lua
return {
  LrShutdownFunction = function(doneFunction, progressFunction)
    -- perform cleanup, call progressFunction(0..1) for progress
    -- MUST call doneFunction() within 10 seconds or LrC force-kills
    doneFunction()
  end
}
```
**Warning:** `LrDialogs` namespace is NOT available during `LrShutdownApp`. Do not attempt to show dialogs during shutdown.

**Enable/disable:** `LrEnablePlugin` and `LrDisablePlugin` scripts run when the user toggles the plugin in Plugin Manager. Use these to start/stop background tasks (e.g. socket listeners).

### Installation Paths

| Platform | Path |
|---|---|
| macOS (user) | `~/Library/Application Support/Adobe/Lightroom/Modules/` |
| macOS (all users) | `/Library/Application Support/Adobe/Lightroom/Modules/` |
| Windows (user) | `%APPDATA%\Adobe\Lightroom\Modules\` |
| Windows (all users) | `%PROGRAMFILES%\Adobe\Lightroom\Modules\` |

Auto-loaded plugins (in Modules folder) cannot be removed via Plugin Manager, only disabled.

### Lua 5.1.5 Sandbox

**Available globals:** assert, dofile, error, getmetatable, setmetatable, ipairs, pairs, next, pcall, xpcall, rawequal, rawget, rawset, require, select, tonumber, tostring, type, unpack, _VERSION, print

**NOT available:** collectgarbage, getfenv, setfenv, module, package, newproxy, loadfile, load, loadstring

**Available namespaces:**
- `io` — full file I/O
- `math` — full math library
- `string` — full string library (also available in Info.lua)

**Partially available:**
- `os` — only `clock`, `date`, `time`, `tmpname`
- `table` — no `getn`, `setn`, `maxn`
- `coroutine` — only `canYield`, `running`
- `debug` — only `getinfo`

### SDK Version History

| SDK | LrC | Key Additions |
|---|---|---|
| 1.3 | Original | Core export, LrView, LrTasks, LrHttp |
| 2.0 | — | +filters, +metadata, +pluginInfo, +LrFileUtils.readFile |
| 3.0 | — | +publish services, +enable/disable, +tagsets, +help menu |
| 4.0 | — | +LrShutdownApp, +LrForceInitPlugin, +URLHandler, +streaming uploads |
| 5.0 | — | +rendition throttling, +video filters |
| 6.0+ | — | +controller SDK (LrDevelopController, LrSocket, LrSelection) |
| 10.0 | — | +color grading views, +copy/pasteSettings (10.3) |
| 11.0 | — | +mask creation/management |
| 12.1 | — | +bitDepth in getRawMetadata, +addPhoto presets (12.5) |
| 13.2 | — | +altTextAccessibility, +point color, +revealPanel subPanelID |
| 13.3 | — | +lens blur, +updateAISettings |
| 14.1 | — | +spot editing, +goToEyeCorrection |
| 14.3 | — | +LrApplication.shutdown, +LrSelection.removeFromCatalog |
| 14.5 | — | +AI/enhance (toggleEnhance), +hasFilterWithName, +resetFilterWithName |
| 15.0 | — | +haptic observers |
| 15.3 | — | +setEnhance (replaces toggleEnhance), +isAvailableForEditing, +needsUpdateAISettings |

### Debugging

**LrLogger setup:**
```lua
local logger = import 'LrLogger'('MyPlugin')
logger:enable('print')  -- or 'logfile'
logger:info("Plugin loaded")
```

**Log file locations:**
- macOS: `~/Library/Logs/Adobe/Lightroom/LrClassicLogs/`
- Windows: `%LOCALAPPDATA%\Adobe\Lightroom\Logs\LrClassicLogs\`

**Deprecation tracking:** Set `AggressiveGarbageCollection = 2` in `config.lua` to get warnings about deprecated API usage.

---

## 2. Building an Export/Publish Service

### 5-Stage Export Pipeline

1. **Deciding render settings** — `updateExportSettings(exportSettings)` called before rendering starts. Modify format, size, color space here.
2. **Filtering photos** — each filter's `shouldRenderPhoto(exportSettings, photo)` runs per-photo. Return false to skip.
3. **Requesting renditions** — `processRenderedPhotos(functionContext, exportContext)` — the main processing entry. Each filter runs in its own task. **Do NOT create your own task inside this function** — LrC already runs it in a task.
4. **Processing rendered photos** — `rendition:waitForRender()` blocks until upstream completes. Returns `success, pathOrMessage`.
5. **Error reporting and cleanup** — temp files cleaned automatically, error summary shown to user, "Previous Export" collection created.

### Export Service Provider

**Callbacks:** startDialog(propertyTable, exportContext), endDialog(propertyTable, why) (why: 'ok'/'cancel'/'changedServiceProvider'), sectionsForTopOfDialog(viewFactory, propertyTable), sectionsForBottomOfDialog(viewFactory, propertyTable), updateExportSettings(exportSettings), processRenderedPhotos(functionContext, exportContext)

**Properties:** allowFileFormats (JPEG/PSD/PSB/TIFF/DNG/PNG/JXL/AVIF/ORIGINAL), allowColorSpaces, canExportVideo, exportPresetFields (`{ key, default }` pairs), hideSections/showSections, supportsIncrementalPublish

### Export Processing Loop

```lua
function exportServiceProvider.processRenderedPhotos(functionContext, exportContext)
  local exportSession = exportContext.exportSession
  exportSession:configureProgress{
    renderPortion = 0.5,
    renderMessage = functionContext:actionWithProgress("Exporting"),
  }

  for i, rendition in exportContext:renditions{ stopIfCanceled = true } do
    local success, pathOrMessage = rendition:waitForRender()
    if success then
      -- pathOrMessage is the temp file path
      local photoId = uploadToService(pathOrMessage)
      rendition:recordPublishedPhotoId(photoId)
      rendition:recordPublishedPhotoUrl("https://example.com/photo/" .. photoId)
      LrFileUtils.delete(pathOrMessage)
    else
      rendition:uploadFailed(pathOrMessage)
    end
  end
end
```

### Export Property Keys

**Format:** LR_format (JPEG/PSD/PSB/TIFF/DNG/PNG/JXL/AVIF/ORIGINAL), LR_export_colorSpace (sRGB/AdobeRGB/ProPhotoRGB/DisplayP3), LR_export_bitDepth (8/16), LR_jpeg_quality (0-100)

**Sizing:** LR_size_doConstrain, LR_size_resizeType (wh/longEdge/shortEdge/megapixels/dimensions), LR_size_maxWidth, LR_size_maxHeight

**Output:** LR_outputSharpeningOn, LR_outputSharpeningMedia (screen/matte/glossy), LR_outputSharpeningLevel (1-3), LR_minimizeEmbeddedMetadata, LR_useWatermark, LR_cantExportBecause (set to disable export with message)

### Publish Service Additions

Publish services extend export services with: deletePhotosFromPublishedCollection(publishSettings, photoIds, deletedCallback, localCollectionId), metadataThatTriggersRepublish (table of keys), getCollectionBehaviorInfo(publishSettings), goToPublishedCollection/Photo(publishSettings, info). Properties: `small_icon` (16x16 icon path), `supportsCustomSortOrder`.

### Export Filter Provider

Filters process renditions after the main export service. Each filter sees the output of the previous stage.

```lua
function filterProvider.postProcessRenderedPhotos(functionContext, filterContext)
  for i, rendition in filterContext:renditions() do
    local success, pathOrMessage = rendition:waitForRender()
    if success then
      local sourceRendition = pathOrMessage  -- input from previous stage
      local outputPath = rendition.destinationPath  -- where to write
      processFile(sourceRendition, outputPath)
      -- rendition is satisfied when outputPath exists
    end
  end
end
```

Key pattern: `sourceRendition` is the input file (from upstream), `rendition.destinationPath` is where the filter must write its output.

---

## 3. Building a Live Develop Visualization

### Architecture — 5-Step Process

Given LrC's severe limitations, the only viable approach for pixel-based visualizations (scopes, histograms, etc.):

1. **Floating dialog** opened from Library/Export menu item
2. **External binary** (compiled C/C++/Rust) that receives image data, decodes it, computes the visualization, and renders to PNG/JPG
3. **`picture` control** in floating dialog displays the result file
4. **`addAdjustmentChangeObserver`** triggers re-export + re-render on slider changes
5. **Polling via `LrTasks`** for active photo changes (no direct observer available)

### Key Challenges

- **Latency**: export -> external process -> render -> display loop adds delay
- **Stale thumbnails**: JPEG thumbnails from `requestJpegThumbnail` may not reflect current develop settings (cached previews)
- **No real-time pixel access**: unlike Photoshop UXP, there is no equivalent to `imaging.getPixels()`
- **Distribution complexity**: external binary adds installation/packaging burden per platform

### Memory Leak Prevention (Non-Negotiable)

LrC runs for hours in a long-lived process. These rules were hard-won through debugging a 40GB memory leak:

#### Rule 1: Frame alternation is mandatory

`f:picture` must receive a *different* file path each update. Writing to the same path causes Lightroom to cache every version internally without releasing.

```lua
local _frameIndex = 0
local function nextScopePath()
  _frameIndex = (_frameIndex + 1) % 2
  return scopeDir .. "/scope_" .. _frameIndex .. ".jpg"
end
```

#### Rule 2: Guard `requestJpegThumbnail` callbacks

Callbacks may fire more than once. After `done = true`, subsequent callbacks must return immediately. Nil out `jpegData` after writing so it can be GC'd. **Must hold a reference** to the returned request object or it may be garbage collected before the callback fires.

```lua
local done = false
local requestObj = photo:requestJpegThumbnail(w, h, function(jpegData, errorMsg)
  if done then return end
  if not jpegData then return end
  done = true
  LrFileUtils.writeFile(tempJpegPath, jpegData)
  jpegData = nil
end)
```

#### Rule 3: Debounce async tasks with a version counter

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

#### Rule 4: No unbounded module-level state

Only fixed-size module-level vars: busy flag, frame index, pending flag, settings hash. Never accumulate items in a list or table at module scope.

#### Rule 5: Clean up temp files on dialog open

Stale temp files from previous runs should be cleared at entry:

```lua
local function cleanup()
  local dir = LrPathUtils.getStandardFilePath('temp') .. '/myplugin'
  if LrFileUtils.exists(dir) then
    LrFileUtils.delete(dir)
  end
  LrFileUtils.createAllDirectories(dir)
end
```

#### Rule 6: `collectgarbage` is unavailable

LrC sandbox blocks `collectgarbage`. There is no manual-GC escape hatch — rules 1-5 are the only defense.

### External Binary Bridge

For pixel-analysis plugins, ship a compiled binary and invoke via `LrTasks.execute()`.

**Platform detection and binary selection:**
```lua
local WIN_ENV = WIN_ENV or (LrShell.pathToMsdosPath ~= nil)
local arch = WIN_ENV and "win-x64"
  or (os.getenv("HOSTTYPE") == "x86_64" and "macos-x64" or "macos-arm64")
local binary = pluginDir .. "/bin/" .. arch .. "/processor" .. (WIN_ENV and ".exe" or "")
```

**Ship per-platform builds:**
- `bin/macos-arm64/processor`
- `bin/macos-x64/processor`
- `bin/win-x64/processor.exe`

**Typical CLI convention** (decode + render two-step):
```bash
# Step 1: Decode image to raw pixels
processor decode --input photo.jpg --output pixels.rgb --width 128 --height 128

# Step 2: Render visualization from raw pixels
processor render --input pixels.rgb --output scope.jpg \
  --width 128 --height 128 --size 512 --density bloom
```

The Lua side calls this via:
```lua
local cmd = string.format('"%s" decode --input "%s" --output "%s" --width %d --height %d',
  binary, inputPath, pixelPath, width, height)
local exitCode = LrTasks.execute(cmd)
```

---

## 4. Working with the Catalog

### LrCatalog — Write Gates

All catalog modifications must happen inside a write gate. Three types:

```lua
local catalog = LrApplication.activeCatalog()

-- Standard write access (creates undo entry)
catalog:withWriteAccessDo("Add keywords", function(context)
  photo:addKeyword(keyword)
end)

-- Private write access (no undo, for plugin-private metadata)
-- Signature: withPrivateWriteAccessDo(func, timeoutParams?) — NO action name
catalog:withPrivateWriteAccessDo(function(context)
  photo:setPropertyForPlugin(_PLUGIN, 'lastSync', os.time())
end)

-- Prolonged write access (long operations, blocks UI)
-- Signature: withProlongedWriteAccessDo(params, timeoutParams)
catalog:withProlongedWriteAccessDo({
  title = "Batch update",
  pluginName = "My Plugin",
  func = function(context, progress)
    -- long-running batch operations; yield periodically with LrTasks.yield()
  end,
}, { timeout = 30 })
```

**Gotchas:**
- On Mac, successive `withWriteAccessDo` calls without user interaction coalesce into a single undo event
- New objects created inside a write gate are not accessible until the gate returns
- `withPrivateWriteAccessDo` does not create undo entries and does not require user confirmation

### Batch Operations

For performance, use batch methods instead of per-photo loops:

```lua
local photos = catalog:getTargetPhotos()
local rawData = catalog:batchGetRawMetadata(photos, { 'path', 'rating', 'uuid' })
local fmtData = catalog:batchGetFormattedMetadata(photos, { 'fileName', 'shutterSpeed' })
local pluginData = catalog:batchGetPropertyForPlugin(photos, _PLUGIN, { 'lastSync' })

for _, photo in ipairs(photos) do
  local info = rawData[photo]
  logger:infof("Photo: %s, rating: %d", info.path, info.rating or 0)
end
```

### Finding Photos — searchDesc

`catalog:findPhotos{ searchDesc = desc }` uses a structured search descriptor:

```lua
local results = catalog:findPhotos{
  searchDesc = {
    criteria = "rating",
    operation = ">=",
    value = 3,
    {
      criteria = "fileFormat",
      operation = "==",
      value = "RAW",
    },
    combine = "intersect",  -- "union", "intersect", "exclude"
  }
}
```

**Plugin metadata search:** Use `sdktext:com.example.myplugin.fieldName` as criteria to search custom metadata fields.

**Tip:** Export a smart collection's settings from Plugin Manager to see the exact `searchDesc` format LrC uses internally.

### Key Catalog Methods

**Selection:** getTargetPhoto(), getTargetPhotos(), getMultipleSelectedOrAllPhotos(), setSelectedPhotos(primary, others)

**Import:** addPhoto(path, stackWithPhoto, position, metadataPresetUUID, developPresetUUID) — presetUUIDs first supported SDK 12.5

**Collections:** createCollection(name, parent, canReturnPrior), createCollectionSet(name, parent, canReturnPrior), createSmartCollection(name, searchDesc, parent, canReturnPrior), getActiveSources(), setActiveSources(sources)

**Search:** findPhotos{searchDesc=...}, findPhotoByUuid(uuid) — **Gotcha:** param is named `path` but is a UUID string

### LrPhoto — Core Methods

**Metadata:** getRawMetadata(key), getFormattedMetadata(key), setRawMetadata(key, value) (requires write gate), getPropertyForPlugin(plugin, key), setPropertyForPlugin(plugin, key, value, version) (requires private write gate)

**Develop:** getDevelopSettings() (experimental), applyDevelopPreset(preset, plugin) (presetAmount 0-200, updateAISettings), applyDevelopSettings(settings, optHistoryName, optFlattenAutoNow) (experimental), copySettings/pasteSettings(opts) (SDK 10.3+), quickDevelopAdjustImage/Crop/WhiteBalance (no `quickDevelopAdjustValue` — the real methods are `quickDevelopAdjustImage`, `quickDevelopAdjustWhiteBalance`, `quickDevelopSetWhiteBalance`, `quickDevelopSetTreatment`, `quickDevelopCropAspect`)

**Thumbnails:** requestJpegThumbnail(width, height, callback) — returns request object (hold reference!), buildSmartPreview(), deleteSmartPreview()

**Other:** rotateLeft/rotateRight(), addKeyword/removeKeyword(keyword) (requires write gate), isAvailableForEditing() (SDK 15.3), needsUpdateAISettings() (SDK 15.3), updateAISettings() (SDK 13.3)

### getDevelopSettings — Typo Warnings

The official API has these typos in returned keys. Use exactly as shown:
- `HueAdjustmentMagenha` (not Magenta)
- `LuminanceAdjustmentAque` (not Aqua)
- `Parametriclights` (not ParametricLights — lowercase L)

### getRawMetadata Key Reference

| Key | Type | Notes |
|---|---|---|
| path | string | Full file path |
| fileName | string | File name only |
| uuid | string | Unique photo ID |
| isVirtualCopy | boolean | |
| virtualCopyName | string | nil if not a virtual copy |
| masterPhoto | LrPhoto | Original if virtual copy |
| copyName | string | Copy name |
| rating | number | 0-5, nil if unrated |
| colorNameForLabel | string | red/yellow/green/blue/purple/none |
| pickStatus | number | 1=flagged, 0=unflagged, -1=rejected |
| fileFormat | string | RAW, DNG, JPG, TIFF, PSD, VIDEO (per LrPhoto docs) |
| width / height | number | Pixel dimensions (uncropped) |
| aspectRatio | number | width/height |
| isCropped | boolean | |
| croppedWidth / croppedHeight | number | After crop |
| orientation | string | AB (normal), BA, CD, DC, etc. |
| bitDepth | number | SDK 12.1 |
| fileSize | number | Bytes |
| dateTimeOriginal | number | **Cocoa epoch** — seconds since Jan 1 2001, NOT Unix |
| dateTimeDigitized / dateTime | number | Cocoa epoch |
| gps | table | `{ latitude, longitude, altitude }` or nil |
| countryCode | string | ISO 3166-1 |
| keywords | table | Array of LrKeyword objects |
| keywordTags | string | Comma-separated keyword names |
| keywordTagsForExport | string | Keywords marked for export |
| containedCollections | table | Array of LrCollection objects |
| developSettings | table | Partial — use getDevelopSettings() for full |
| editingMode | string | develop, library, etc. |
| isInStackInFolder | boolean | |
| stackPositionInFolder | number | |
| stackInFolderIsCollapsed | boolean | |
| isRawFile | boolean | |
| smartPreviewInfo | table | `{ smartPreviewPath, smartPreviewSize }` |
| customMetadata | table | All plugin metadata |
| altTextAccessibility | string | Alt text (SDK 13.2) |
| lastEditTime | number | Cocoa epoch |
| hasDevelopAdjustments | boolean | |
| locationIsPrivate | boolean | |

### getFormattedMetadata Key Reference

Returns human-readable strings. Grouped by category:

**File info:** fileName, copyName, folderName, fileSize, fileType, dateCreated

**Exposure:** exposure, shutterSpeed, aperture, isoSpeedRating, focalLength, focalLength35mm, exposureBias, flash, meteringMode, exposureProgram, brightnessValue, maxAperture, subjectDistance, lens

**Camera:** cameraMake, cameraModel, cameraSerialNumber, lensModel, lensInfo

**IPTC Contact:** creator, creatorJobTitle, creatorAddress, creatorCity, creatorStateProvince, creatorPostalCode, creatorCountry, creatorPhone, creatorEmail, creatorUrl

**IPTC Content:** title, headline, caption, iptcSubjectCode, descriptionWriter, intellectualGenre, scene, iptcCategory, iptcOtherCategories

**Location:** sublocation, city, stateProvince, country, isoCountryCode, jobIdentifier, instructions, provider, source

**IPTC Extension/PLUS:** personShown, nameOfOrgShown, codeOfOrgShown, event, additionalModelInfo, modelAge, modelReleaseStatus, modelReleaseID, imageSupplier, imageSupplierImageId, registryId, maxAvailWidth, maxAvailHeight, sourceType, copyrightOwner, licensor, minorModelAge, propertyReleaseID, propertyReleaseStatus, artworkOrObject, digitalSourceType

### setRawMetadata — Writable Keys

These keys can be set via `photo:setRawMetadata(key, value)` inside a write gate:

`rating`, `colorNameForLabel`, `pickStatus`, `gps` (table with latitude/longitude), `dateTimeOriginal`, `title`, `caption`, `copyright`, `creator`, `headline`, `altTextAccessibility`, `locationIsPrivate`

### Custom Metadata

**Definition script** (referenced by `LrMetadataProvider` in Info.lua):

```lua
return {
  schemaVersion = 1,
  updateFromEarlierSchemaVersion = function(catalog, previousSchemaVersion)
    -- migration logic
  end,

  metadataFieldsForPhotos = {
    {
      id = 'lastSync',
      title = "Last Sync Time",
      dataType = 'string',      -- 'string', 'enum', 'url'
      searchable = true,
      browsable = true,
    },
    {
      id = 'status',
      title = "Sync Status",
      dataType = 'enum',
      values = {
        { value = 'pending', title = "Pending" },
        { value = 'synced', title = "Synced" },
      },
    },
  },
}
```

**Reading/writing custom metadata:**
```lua
-- Read
local value = photo:getPropertyForPlugin(_PLUGIN, 'lastSync')

-- Write (requires private write gate; no action name arg)
catalog:withPrivateWriteAccessDo(function()
  photo:setPropertyForPlugin(_PLUGIN, 'lastSync', tostring(os.time()))
end)
```

**Tagsets** — expose custom metadata in the Metadata panel:
```lua
-- In a tagset factory script
return {
  title = "My Plugin Metadata",
  id = 'myPluginTagset',
  items = {
    'com.adobe.label',           -- built-in fields
    'com.adobe.rating',
    _PLUGIN.id .. '.*',          -- wildcard: all fields from this plugin
  },
}
```

---

## 5. Building UIs

### LrView Factory Methods

**Containers:**
- `column { ... }` — vertical stack
- `row { ... }` — horizontal stack
- `view { ... }` — generic container
- `group_box { title = "...", ... }` — titled group
- `tab_view { ... }` — tabbed container
- `tab_view_item { title = "...", identifier = "...", ... }` — individual tab
- `scrolled_view { width, height, ... }` — scrollable area

**Controls:**
- `checkbox { title, value = bind('key') }` — boolean toggle
- `color_well { value = bind('color') }` — color picker
- `combo_box { value = bind('key'), items = {...} }` — editable dropdown
- `edit_field { value = bind('key'), width_in_chars = 20 }` — text input
- `password_field { value = bind('key') }` — masked input
- `picture { value = bind('path'), width, height }` — image display from file
- `popup_menu { value = bind('key'), items = {{title, value}, ...} }` — dropdown
- `push_button { title = "...", action = function() end }` — button
- `radio_button { title = "...", value = bind('key'), checked_value = "..." }` — radio option
- `slider { value = bind('key'), min, max, integral }` — numeric slider
- `simple_list { items = {...}, allows_multiple_selection }` — list view
- `static_text { title = "..." }` — label
- `catalog_photo { photo = lrPhoto, width, height }` — photo thumbnail

**Layout helpers:**
- `separator { fill_horizontal = 1 }` — horizontal line
- `spacer { height = 10 }` — empty space
- `control_spacing()` — standard control gap
- `dialog_spacing()` — standard dialog gap
- `label_spacing()` — standard label gap

### Property Categories

**View properties:** `bind_to_object` (override binding target), `tooltip`, `visible` (hidden items still affect layout)

**Control properties:** `enabled`, `font` (canonical names: `<system>`, `<system/small>`, `<system/bold>`, `<system/small/bold>`), `size` ('mini', 'small', 'regular')

**Text properties:** `height_in_lines`, `width_in_chars`, `width_in_digits`

**Edit field properties:** `immediate` (update on every keystroke), `validate` (function returning true/false + value), `precision` (decimal places), `min`/`max`, `placeholder_string`, `string_to_value`/`value_to_string` (custom conversion)

**Node layout:** `fill`/`fill_horizontal`/`fill_vertical` (0..1 proportion), `width`/`height` (fixed pixels), `place_horizontal`/`place_vertical` ('left'/'center'/'right', 'top'/'center'/'bottom')

**Child layout:** `margin`/`margin_horizontal`/`margin_vertical`/`margin_left`/`margin_right`/`margin_top`/`margin_bottom`, `place` ('vertical'/'horizontal'/'overlapping'), `spacing`

### Data Binding Patterns

**Basic binding:**
```lua
local props = LrBinding.makePropertyTable(context)
props.name = "default"

f:edit_field { value = LrView.bind('name') }
```

**Transform binding:**
```lua
f:static_text {
  title = LrView.bind {
    key = 'count',
    transform = function(value) return string.format("%d items", value or 0) end,
  },
}
```

**Multi-key computed binding:**
```lua
f:static_text {
  title = LrView.bind {
    keys = { 'width', 'height' },
    operation = function(binder, values, fromTable)
      return string.format("%dx%d", values.width or 0, values.height or 0)
    end,
  },
}
```

**LrBinding helpers:**
- `LrBinding.keyEquals(key, value)` — true when key equals value
- `LrBinding.keyIsNil(key)` / `LrBinding.keyIsNotNil(key)`
- `LrBinding.keyIsNot(key, value)` — true when key does not equal value
- `LrBinding.negativeOfKey(key)` — boolean negation
- `LrBinding.andAllKeys(key1, key2, ...)` — logical AND
- `LrBinding.orAllKeys(key1, key2, ...)` — logical OR

**Cross-table binding:**
```lua
f:edit_field {
  bind_to_object = otherTable,  -- override default binding target
  value = LrView.bind('otherKey'),
}
```

**Observer pattern:**
```lua
props:addObserver('selectedFormat', function(properties, key, newValue)
  -- respond to changes
end)

-- Watch ALL keys
props:addObserver('ALL', function(properties, key, newValue)
  -- fires on any change — use guard flags to prevent loops
end)
```

### LrDialogs — All 14 Functions

| Function | Purpose |
|---|---|
| presentModalDialog(args) | Modal dialog. `cancelVerb = "< exclude >"` hides cancel button. `accessoryView` mutually exclusive with `otherVerb`. |
| presentFloatingDialog(plugin, args) | Modeless floating window. First arg is `_PLUGIN`. `blockTask = true` to keep task alive for observers. `selectionChangeObserver`, `sourceChangeObserver`, `windowWillClose` callbacks. |
| confirm(message, info, actionVerb, cancelVerb, otherVerb) | Returns `'ok'`/`'cancel'`/`'other'` |
| message(message, info, style) | Simple message dialog |
| messageWithDoNotShow(args) | Message with "don't show again" checkbox |
| showError(errorString) | Error dialog. Handles LrErrors strings. |
| showBezel(message, fadeDelay) | Toast notification. Only one at a time. |
| showModalProgressDialog(params) | Progress dialog with cancel support |
| stopModalWithResult(dialog, result) | Dismiss modal from within |
| runOpenPanel(args) | File/folder open dialog |
| runSavePanel(args) | File save dialog |
| attachErrorDialogToFunctionContext(context) | Auto-show errors on context failure |
| resetDoNotShowFlag(actionPrefKey) | Reset "don't show again" flags |
| promptForActionWithDoNotShow(args) | Dialog with "don't show again" checkbox |

### View Layout Patterns

**Shared widths** for aligned labels:
```lua
local labelWidth = LrView.share('label_width')
f:row {
  f:static_text { title = "Name:", width = labelWidth },
  f:edit_field { value = LrView.bind('name'), fill_horizontal = 1 },
}
f:row {
  f:static_text { title = "Description:", width = labelWidth },
  f:edit_field { value = LrView.bind('desc'), fill_horizontal = 1 },
}
```

**Overlapping placement** for conditional visibility:
```lua
f:view {
  place = 'overlapping',
  f:static_text { title = "Loading...", visible = LrBinding.negativeOfKey('loaded') },
  f:picture { value = LrView.bind('imagePath'), visible = LrView.bind('loaded') },
}
```

**Conditional view with 'skipped item':**
```lua
f:column {
  showAdvanced and f:group_box { title = "Advanced", ... } or 'skipped item',
}
```

**Early return in non-function context** — use `repeat...until true`:
```lua
repeat
  if not condition then break end
  -- main logic
until true
```

---

## 6. External Communication

### LrSocket

```lua
local sender = LrSocket.bind {
  functionContext = context,
  plugin = _PLUGIN,
  port = 0,           -- 0 = auto-assign
  mode = 'send',      -- or 'receive'
  onConnecting = function(socket, port) end,
  onConnected = function(socket, port) end,
  onMessage = function(socket, message) end,
  onClosed = function(socket) end,
  onError = function(socket, err) end,
}
sender:send("key=value\n")
sender:close()
```

Sockets are auto-closed if the plugin is disabled.

### Controller SDK Architecture

Two-component system: LrC plugin establishes sockets, external driver communicates via TCP on localhost using a simple `key=value` text protocol.

```lua
-- Main loop pattern
_G.running = true
while _G.running do
  -- process messages from external controller
  LrTasks.sleep(1/2)  -- yield to keep LrC responsive
end
```

### LrHttp

```lua
-- GET
local body, headers = LrHttp.get(url, requestHeaders, timeout)

-- POST
local body, headers = LrHttp.post(url, postBody, requestHeaders, method, timeout)

-- Multipart
local body, headers = LrHttp.postMultipart(url, content, requestHeaders)
```

Must be called from an async task. Error handling: returns `nil, { error = { errorCode = "..." } }`. Error codes: `cancelled`, `badURL`, `timedOut`, `cannotFindHost`, `cannotConnectToHost`, `resourceUnavailable`, `networkConnectionLost`, `redirectError`, `badServerResponse`, `authenticationError`, `securityError`, `serverCertificateHasBadDate`, `serverCertificateHasUnknownRoot`.

Set `Content-Type = 'skip'` to omit the Content-Type header. Streaming uploads via callback functions (SDK 4.0+).

### LrTasks.execute and LrShell

```lua
-- Shell command (blocks only the calling task)
local exitCode = LrTasks.execute(cmd)

-- Reveal file in Finder/Explorer
LrShell.revealInShell(path)

-- Open with app
LrShell.openFilesInApp({ path1, path2 }, appPath)

-- Open paths via command line
LrShell.openPathsViaCommandLine(paths, appPath)
-- Mac: sequential execution due to fork
-- Windows: command lines >1500 chars are split into multiple calls
```

**Mac App Store sandbox caveat:** `LrTasks.execute` may be restricted in App Store builds.

---

## 7. Develop Parameter Reference

### Complete Parameter List by Panel

The SDK does not publish a single total count; the full global list (sum across panels below) is the canonical enumeration.

```
adjustPanel: Temperature, Tint, Exposure, Highlights, Shadows, Brightness, Contrast,
  Whites, Blacks, Texture, Clarity, Dehaze, Vibrance, Saturation, PresetAmount, ProfileAmount

tonePanel: ParametricDarks, ParametricLights, ParametricShadows, ParametricHighlights,
  ParametricShadowSplit, ParametricMidtoneSplit, ParametricHighlightSplit, ToneCurve,
  ToneCurvePV2012, ToneCurvePV2012Red, ToneCurvePV2012Blue, ToneCurvePV2012Green,
  CurveRefineSaturation

mixerPanel: SaturationAdjustmentRed, SaturationAdjustmentOrange, SaturationAdjustmentYellow,
  SaturationAdjustmentGreen, SaturationAdjustmentAqua, SaturationAdjustmentBlue,
  SaturationAdjustmentPurple, SaturationAdjustmentMagenta,
  HueAdjustmentRed, HueAdjustmentOrange, HueAdjustmentYellow, HueAdjustmentGreen,
  HueAdjustmentAqua, HueAdjustmentBlue, HueAdjustmentPurple, HueAdjustmentMagenta,
  LuminanceAdjustmentRed, LuminanceAdjustmentOrange, LuminanceAdjustmentYellow,
  LuminanceAdjustmentGreen, LuminanceAdjustmentAqua, LuminanceAdjustmentBlue,
  LuminanceAdjustmentPurple, LuminanceAdjustmentMagenta,
  PointColors, GrayMixerRed, GrayMixerOrange, GrayMixerYellow, GrayMixerGreen,
  GrayMixerAqua, GrayMixerBlue, GrayMixerPurple, GrayMixerMagenta

colorGradingPanel: SplitToningShadowHue, SplitToningShadowSaturation, ColorGradeShadowLum,
  SplitToningHighlightHue, SplitToningHighlightSaturation, ColorGradeHighlightLum,
  ColorGradeMidtoneHue, ColorGradeMidtoneSat, ColorGradeMidtoneLum,
  ColorGradeGlobalHue, ColorGradeGlobalSat, ColorGradeGlobalLum,
  SplitToningBalance, ColorGradeBlending

detailPanel: Sharpness, SharpenRadius, SharpenDetail, SharpenEdgeMasking,
  LuminanceSmoothing, LuminanceNoiseReductionDetail, LuminanceNoiseReductionContrast,
  ColorNoiseReduction, ColorNoiseReductionDetail, ColorNoiseReductionSmoothness

effectsPanel: PostCropVignetteAmount, PostCropVignetteMidpoint, PostCropVignetteFeather,
  PostCropVignetteRoundness, PostCropVignetteStyle, PostCropVignetteHighlightContrast,
  GrainAmount, GrainSize, GrainFrequency

lensCorrectionsPanel: AutoLateralCA, LensProfileEnable, LensProfileDistortionScale,
  LensProfileVignettingScale, LensManualDistortionAmount,
  DefringePurpleAmount, DefringePurpleHueLo, DefringePurpleHueHi,
  DefringeGreenAmount, DefringeGreenHueLo, DefringeGreenHueHi,
  VignetteAmount, VignetteMidpoint,
  PerspectiveVertical, PerspectiveHorizontal, PerspectiveRotate, PerspectiveScale,
  PerspectiveAspect, PerspectiveX, PerspectiveY, PerspectiveUpright

calibratePanel: ShadowTint, RedHue, RedSaturation, GreenHue, GreenSaturation,
  BlueHue, BlueSaturation

lensBlurPanel: LensBlurActive, LensBlurAmount, LensBlurCatEye, LensBlurHighlightsBoost,
  LensBlurFocalRange

Other: straightenAngle
```

### Local Adjustment Parameters by Process Version

| Process Version | Parameters |
|---|---|
| v2 (8 params) | local_Exposure, local_Contrast, local_Clarity, local_Saturation, local_Sharpness, local_ToningLuminance, local_ToningHue, local_ToningSaturation |
| v3/v4 (18 params) | local_Temperature, local_Tint, local_Exposure, local_Contrast, local_Highlights, local_Shadows, local_Clarity, local_Saturation, local_ToningHue, local_ToningSaturation, local_Sharpness, local_LuminanceNoise, local_Moire, local_Defringe, local_Blacks, local_Whites, local_Dehaze, local_PointColors |
| v5 (25 params) | v3/v4 + local_Texture, local_Hue, local_Amount, local_Maincurve, local_Redcurve, local_Greencurve, local_Bluecurve |
| v6 (27 params) | v5 + local_Grain, local_RefineSaturation |

### LrDevelopController Functions — Complete (95 functions in SDK 15.3)

**Get/Set (12):**
getValue(param), setValue(param, value), getRange(param), increment(param), decrement(param), resetToDefault(param), resetAllDevelopAdjustments(), startTracking(param), stopTracking(), setTrackingDelay(seconds), setMultipleAdjustmentThreshold(amount), addAdjustmentChangeObserver(context, observer, callback)

**Tools (11):**
selectTool(tool), getSelectedTool(), goToRemove(), goToMasking(), goToEyeCorrection(eyeCorrectionType?), editInPhotoshop(), hasFilterWithName(name). Deprecated: goToHealing(), goToSpotRemoval(), goToDevelopGraduatedFilter(), goToDevelopRadialFilter().
Tool names (selectTool): `loupe`, `crop`, `dust`, `redeye`, `masking`, `upright`, `point_color`, `local_point_color`, `depth_refinement`. Note: `getSelectedTool()` does not return `upright`.
Eye correction types: `"red_eye"`, `"pet_eye"`.

**Masks (SDK 11.0+, 17):**
createNewMask(maskType, maskSubtype?), addToCurrentMask(maskType, maskSubtype?), subtractFromCurrentMask(maskType, maskSubtype?), intersectWithCurrentMask(maskType, maskSubtype?), getAllMasks(), getSelectedMask(), selectMask(id, param), deleteMask(id, param), invertMask(id, param), duplicateAndInvertMask(id, param), toggleHideMask(id, param), toggleOverlay(), selectMaskTool(id, param), getSelectedMaskTool(), deleteMaskTool(id, param), toggleHideMaskTool(id, param), toggleInvertMaskTool(id, param)
Mask types: `brush`, `gradient`, `radialGradient`, `rangeMask`, `aiSelection`
Mask subtypes (only with rangeMask or aiSelection): `color`, `luminance`, `depth`, `subject`, `sky`, `background`, `objects`, `people`, `landscape`

**Spots (SDK 14.1+, 14):**
countAllSpots(), getAllSpots(), getSelectedSpotIndex(), getSelectedSpotParams(), getSelectedSpotType(), setSelectedSpotIndex(idx), setSelectedSpotParams(params), setSelectedSpotType(spotType, useGenAI), moveSelectedSpot(dx, dy), deleteSelectedSpot(), deleteSelectedVariation(), refreshSelectedSpot(), gotoNextVariation(), gotoPreviousVariation()
Spot types: `"heal_patchmatch"`, `"heal"`, `"clone"`

**AI/Enhance (SDK 14.5+, 10):**
setEnhance(paramName, value, denoiseAmount) (SDK 15.3 — `paramName ∈ "denoise"|"rawDetails"|"superRes"`, `value` boolean, `denoiseAmount` optional number 1-100), toggleEnhance(paramName, denoiseAmount, callback, callbackFuncArgTable) (deprecated, use setEnhance), changeDenoiseAmount(delta), getEnhancePanelState(), toggleReflectionRemoval(), changeReflectionRemovalAmount(delta), changeReflectionRemovalQuality(delta), getReflectionRemovalPanelState(), detectDistractingPeople(), applyRemovalOnDetectedDistractingPeople()

**Point Color (SDK 13.2+, 6):**
addPointColorSwatch(), deletePointColorSwatch(index), selectPointColorSwatch(index), updateSelectedPointColorSwatch(params), getSelectedPointColorSwatchIndex(), togglePointColorRangeVisualization()

**Lens Blur (SDK 13.3+, 3):**
setLensBlurBokeh(type), getSelectedLensBlurBokeh(), toggleLensBlurDepthVisualization(). LensBlur params (Active, Amount, CatEye, HighlightsBoost, FocalRange) via getValue/setValue.
Bokeh types: `Circle`, `SoapBubble`, `Blade`, `Ring`, `Anamorphic`

**Color Grading (SDK 10.0+, 2):**
setActiveColorGradingView(view), getActiveColorGradingView()
Views: `3-way`, `shadow`, `midtone`, `highlight`, `global`

**Process Version (2):**
getProcessVersion(), setProcessVersion(version)
Versions: `"Version 1"` through `"Version 6"`

**Remove Panel (2):**
getRemovePanelPreferences(), setRemovePanelPreferences(prefs)

**Navigation and Reset (16):**
revealPanel(paramOrPanelID, subPanelID?), revealPanelIfVisible(panelID), revealAdjustedControls(), showClipping() (no args), setAutoTone(), setAutoWhiteBalance(), resetCrop(), resetTransforms(), resetMasking(), resetRedeye(), resetFilterWithName(name). Deprecated: resetHealing(), resetBrushing(), resetCircularGradient(), resetGradient(), resetSpotRemoval().
Panel IDs: `adjustPanel`, `tonePanel`, `mixerPanel`, `colorGradingPanel`, `detailPanel`, `effectsPanel`, `lensCorrectionsPanel`, `calibratePanel`, `lensBlurPanel`. Mixer sub-panels (`subPanelID` on `revealPanel`, SDK 13.2+): `hslColorPanel`, `pointColorPanel`.

### Enumerated Values Summary

| Category | Values |
|---|---|
| Panel IDs | adjustPanel, tonePanel, mixerPanel, colorGradingPanel, detailPanel, effectsPanel, lensCorrectionsPanel, calibratePanel, lensBlurPanel |
| Mixer sub-panel IDs (revealPanel) | hslColorPanel, pointColorPanel |
| Tool names (selectTool) | loupe, crop, dust, redeye, masking, upright, point_color, local_point_color, depth_refinement |
| Mask types | brush, gradient, radialGradient, rangeMask, aiSelection |
| Mask subtypes | color, luminance, depth, subject, sky, background, objects, people, landscape |
| Spot types | heal_patchmatch, heal, clone |
| Bokeh types | Circle, SoapBubble, Blade, Ring, Anamorphic |
| Color grading views | 3-way, shadow, midtone, highlight, global |
| Process versions | Version 1 through Version 6 |
| Filter names (hasFilterWithName, resetFilterWithName) | enhance, dereflect, remove_people |
| Eye correction types | red_eye, pet_eye |
| Enhance param names (setEnhance) | denoise, rawDetails, superRes |

---

## 8. Module Quick Reference

**LrApplication** — activeCatalog(), versionTable, addHapticObserver(callback) (SDK 15.0), shutdown() (SDK 14.3), developPresetFolders(), filenamePresets(), purchaseSource()

**LrApplicationView** — switchToModule(name), showView(viewName), getCurrentModuleName(), toggleZoom(), zoomToOneToOne() (SDK 10.0), showSecondaryView(viewType)

**LrSelection** — 27 functions: setRating(0-5), setColorLabel(color), flagAsPick(), flagAsReject(), removeFlag(), getRating(), getColorLabel(), getFlag(), selectAll(), selectNone(), selectInverse(), selectFirstPhoto(), selectLastPhoto(), nextPhoto(), previousPhoto(), extendSelection(), removeFromCatalog() (SDK 14.3)

**LrTasks** — startAsyncTask(func), startAsyncTaskWithoutErrorHandler(func), sleep(seconds), execute(cmd), pcall(func), canYield(), yield()

**LrFunctionContext** — Module functions: callWithContext(name, func), postAsyncTaskWithContext(name, func), callWithEmptyEnvironment(func). Instance methods on the `context` object: `context:addCleanupHandler(func)`, `context:addFailureHandler(func)`

**LrBinding** — makePropertyTable(context), keyEquals(key, value), keyIsNil(key), keyIsNotNil(key), keyIsNot(key, value), negativeOfKey(key), andAllKeys(...), orAllKeys(...), kUnsupportedDirection

**LrObservableTable** — addObserver(key, func) (key='ALL' for all changes), removeObserver(key, func), pairs()

**LrExportSession** — countRenditions(), renditions(), doExportOnCurrentTask(), doExportOnNewTask(), recordRemoteCollectionId(id), recordRemoteCollectionUrl(url)

**LrExportRendition** — waitForRender(), recordPublishedPhotoId(id), recordPublishedPhotoUrl(url), renditionIsDone(), skipRender(), uploadFailed(msg). Properties: destinationPath, photo, publishedPhotoId, wasSkipped

**LrExportContext** — configureProgress(opts), renditions(opts), startRendering(). Properties: exportSession, propertyTable, publishService, publishedCollection

**LrExportSettings** — extensionForFormat(format). Formats: ORIGINAL, JPEG, JXL, AVIF, PNG, TIFF, PSD, PSB, DNG. addVideoExportPresets(), supportableVideoExportFormats()

**LrColor** — constructors: LrColor(), LrColor(r,g,b,a), LrColor(name), LrColor(name,a). Named colors: black, white, gray, light gray, dark gray, red, green, blue, cyan, yellow, magenta, orange, purple, brown. Methods: red(), green(), blue(), alpha()

**LrDate** — currentTime(), timeFromComponents(...), timestampToComponents(time). **Cocoa epoch: seconds since Jan 1 2001**, NOT Unix epoch. formatShortDate/MediumDate/LongDate(time), timeToW3CDate(time), timeFromPosixDate(posix), timeToPosixDate(cocoa)

**LrFileUtils** — exists(path) returns `'file'`/`'directory'`/`false`. readFile(path) (SDK 2.2), copy(src,dst), move(src,dst), createAllDirectories(path), delete(path), moveToTrash(path), directoryEntries(path), files(path), recursiveFiles(path). **Do NOT use `break` in directoryEntries/files/recursiveFiles loops.** chooseUniqueFileName(path), fileAttributes(path)

**LrPathUtils** — child(parent, name), parent(path) (returns nil for root since SDK 2.0), extension(path), addExtension/removeExtension/replaceExtension(path, ext), standardizePath(path), getStandardFilePath(key). Keys: `home`, `temp`, `desktop`, `appPrefs`, `pictures`, `documents`, `appData`

**LrHttp** — get(url, headers, timeout), post(url, postBody, headers, method, timeout, totalSize), postMultipart(url, content, headers), openUrlInBrowser(url), parseCookie(headerValue). Error codes: `cancelled`, `badURL`, `timedOut`, `cannotFindHost`, `cannotConnectToHost`, `resourceUnavailable`, `networkConnectionLost`, `redirectError`, `badServerResponse`, `authenticationError`, `securityError`, `serverCertificateHasBadDate`, `serverCertificateHasUnknownRoot`

**LrSocket** — bind(opts) with mode `'send'`/`'receive'`

**LrShell** — revealInShell(path), openFilesInApp(paths, app), openPathsViaCommandLine(paths, app)

**LrLogger** — constructor: LrLogger(name). enable('print'/'logfile'). Levels: trace, debug, info, warn, error, fatal + `f` variants (tracef, debugf, etc.). quickf(level, fmt, ...) for tight loops

**LrPrefs** — prefsForPlugin() returns prefs table. **Deep table gotcha:** reassign the top-level key to trigger save (modifying nested tables does not auto-save). Use `prefs:pairs()` not Lua `pairs(prefs)`.

**LrStringUtils** — trimWhitespace(s), lower(s)/upper(s) (Unicode-aware), encodeBase64(s)/decodeBase64(s), truncate(s, len) (UTF-8-safe), compareStrings(a,b), numberToStringWithSeparators(n)

**LrDigest** — SHA256(data), SHA512(data). Factory pattern: init(), update(chunk), digest(). Deprecated: MD4, MD5, SHA1 since LrC 13.5

**LrMD5** — digest(data). Still works but LrDigest preferred.

**LrXml** — createXmlBuilder(): beginBlock(tag), endBlock(), tag(name, attrs), text(str), serialize(). parseXml(str): childAtIndex(i), childCount(), name(), text(), attributes(), transform(). Platform whitespace differences in output.

**LrErrors** — throwUserError(msg), throwCanceled(), isCanceledError(err)

**LrRecursionGuard** — `LrRecursionGuard(name)` constructor. Instance: `guard:performWithGuard(func, ...)` — runs `func` only if not already active on this guard.

**LrProgressScope** — constructor(opts), setPortionComplete(n, total), isCanceled(), done(), setCaption(str), setIndeterminate() (no args — only works inside showModalProgressDialog)

**LrSystemInfo** — appWindowSize(), numCPUs(), memSize(), is64Bit(), architecture(), osVersion(), ipAddress(), displayInfo()

**LrLocalization** — LOC(zstring, ...) with ^1-^9 substitution. currentLanguage(). ZString escapes: ^r=CR, ^n=LF, ^B=bullet. Languages: de, en, es, fr, it, ja, ko, nl, pt, ru, sv, th, zh_cn, zh_tw

**LrPasswords** — store(service, account, password), retrieve(service, account). Uses OS keychain, scoped by plugin ID.

**LrFtp** — create(opts) with protocol `'ftp'`/`'sftp'`. putFile(localPath, remotePath), exists(remotePath), makeDirectory(remotePath), disconnect(). makeFtpPresetPopup(viewFactory, props)

**LrTether** — startTether() (no args), stopTether(), triggerCapture(), triggerCaptureBlocking(), isTetherActive()

**LrSounds** — getSystemSounds(), playSystemSound(name). Both require async task. **LrUndo** — undo/redo(), canUndo/canRedo(). **LrSlideshow** — startSlideshow/stopSlideshow()

**LrPlugin** — `_PLUGIN` global: enabled, id, path, hasResource(name), resourceId(name)

**LrPhotoInfo** — fileAttributes(path) returns `{ width, height }` for any supported photo file (not necessarily in catalog)

**LrVideoExportPreset** — extension, formatID, name, presetPath

**LrFilterContext** — renditions() alias. Properties: propertyTable, renditionsToSatisfy, sourceExportSession

**LrWebViewFactory** — specialized factory for web-engine plugins: panel_content(), slider_row(), checkbox_row(), labeled_text_input(), identity_plate()

**LrPublishService** — createPublishedCollection(name, parent), getChildCollections(), getPublishSettings(), getPluginId() (may have ".N" suffix)

**LrPublishedCollection** — getPublishedPhotos(), addPhotoByRemoteId(photo, remoteID, remoteUrl, published), publishNow(doneCallback), setRemoteId/Url(id/url)

**LrPublishedPhoto** — getEditedFlag(), getPhoto(), getRemoteId/Url(), setEditedFlag(flag)

**LrCollection** — addPhotos/removePhotos(photos), getPhotos(), getName(), setSearchDescription(desc) (smart collections)

**LrCollectionSet** — getChildCollections/Sets(), delete(). **LrFolder** — getPhotos(opts) (includeChildren), getPath()

**LrKeyword** — getAttributes() (keywordType="person" for face tags, SDK 6.0), getChildren(), getName(), getSynonyms()

---

## 9. LrC vs. Photoshop UXP — Comparison

| Capability | Photoshop UXP | Lightroom Classic |
|---|---|---|
| Language | JavaScript (HTML/CSS/JS) | Lua |
| Pixel Access | Full read/write via Imaging API | None — rendered export only |
| UI Framework | HTML/CSS/JS + Spectrum components | `LrView` declarative widgets |
| Canvas/Drawing | Limited 2D canvas; SVG + generated images work | No drawing primitives at all |
| WebView | Panels since UXP 6.4, local HTML since 8.0 | `LrWebViewFactory` — limited |
| Color Data | Full pixel arrays, color space conversion, profiling | `LrColor` for develop settings only |
| Event System | Rich notification listeners (~219 action events) | `LrCatalog:addPropertyChangeObserver`, limited events |
| Native Code | Hybrid plugins (C++ bridge) | External tool launched via `LrTasks.execute` |
| Develop Control | Direct pixel manipulation, `batchPlay` | `LrDevelopController` for sliders only |
| Document Model | Full DOM (Document, Layer, Channel, Path, Selection, History) | Catalog-centric (LrCatalog, LrPhoto, LrCollection) |
| Plugin Types | Panel, Command | Export, Publish, Metadata, Web Gallery, Filter |
| File Access | Sandboxed with permissions | Full Lua file I/O |

**Key architectural consequence:** Photoshop UXP can directly read pixel data from the active document or any layer via `imaging.getPixels()`. Lightroom Classic SDK has **no equivalent**. Plugins cannot read pixels from Lua. The workaround is: export rendered JPEG/TIFF via `LrExportRendition` or `photo:requestJpegThumbnail`, then decode in an external binary. This makes any pixel-analysis LrC plugin fundamentally a two-process architecture.
