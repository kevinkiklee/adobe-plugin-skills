# Photoshop UXP Plugin Reference

Comprehensive reference for UXP (Unified Extensibility Platform) plugins targeting Photoshop. Based on UXP 8.1.0 / Manifest v5 / Photoshop 23.3+.

UXP is **not** a browser — it has a custom layout engine, limited DOM, and no WebGL/Workers. Don't assume web platform features work.

---

## Table of Contents

1. [Plugin Architecture](#plugin-architecture)
2. [Manifest v5 Configuration](#manifest-v5-configuration)
3. [Panel Sizing and Layout](#panel-sizing-and-layout)
4. [HTML Element Support](#html-element-support)
5. [CSS Support](#css-support)
6. [Canvas API](#canvas-api)
7. [Imaging API](#imaging-api)
8. [Spectrum UXP Components](#spectrum-uxp-components)
9. [Event System](#event-system)
10. [executeAsModal](#executeasmodal)
11. [DOM Limitations](#dom-limitations)
12. [Theme Awareness](#theme-awareness)
13. [Hybrid Plugins (C++ Bridge)](#hybrid-plugins-c-bridge)
14. [Known Issues](#known-issues)

---

## Plugin Architecture

### Plugin Types

UXP supports two entry point types:

- **Panel**: Persistent UI panel docked in Photoshop's workspace. Lifecycle: `create(rootNode)`, `show(rootNode, data)`, `hide(rootNode, data)`, `destroy(rootNode)`, `invokeMenu(menuId)`. Size constraints: `minimumSize`, `maximumSize`, `preferredDockedSize`, `preferredFloatingSize`.
- **Command**: Headless execution triggered by menu items. Lifecycle: `run()`, `cancel()`. No persistent UI.

A single plugin can declare multiple entry points of mixed types.

### Entry Point & Loading

- `main` field in manifest points to an HTML file (e.g., `index.html`)
- JavaScript loaded via `window.require()` or ES imports
- Photoshop API accessed via `require('photoshop')`
- Plugins loaded/debugged through UXP Developer Tool (UDT)
- UDT supports: Load, Reload, Watch (auto-reload), Unload
- Lifecycle methods have a **300ms timeout** for promise resolution

### Key Modules

| Module | Access | Purpose |
|--------|--------|---------|
| `photoshop.app` | `require('photoshop').app` | App state, active document, color profiles |
| `photoshop.core` | `require('photoshop').core` | executeAsModal, layer tree, menu commands, events |
| `photoshop.action` | `require('photoshop').action` | batchPlay, notification listeners |
| `photoshop.imaging` | `require('photoshop').imaging` | Pixel data read/write |
| `photoshop.constants` | `require('photoshop').constants` | Enumerations |

---

## Manifest v5 Configuration

```json
{
  "manifestVersion": 5,
  "id": "com.example.myplugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "main": "index.html",
  "host": {
    "app": "PS",
    "minVersion": "23.3.0"
  },
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
| `webview` | `{ allow, domains }` | WebView in modal dialogs (UXP 6.0+) |
| `enableUserInfo` | boolean | User GUID access (PS 25.1, UXP 7.3+) |

Manifest v5 requires Photoshop 23.3.0+ and UXP 6.0+.

---

## Panel Sizing and Layout

| Property | Description | Notes |
|----------|-------------|-------|
| `minimumSize` | Smallest allowed dimensions | Host may not honor in all docking configs |
| `maximumSize` | Largest allowed dimensions | Same caveat |
| `preferredDockedSize` | Initial size when docked | Preference, not guaranteed |
| `preferredFloatingSize` | Initial size when floating | Preference, not guaranteed |

### Panel Scrolling

Via CSS `overflow: auto` or `overflow: scroll`. Does NOT auto-scroll to keep focused controls visible on macOS (known issue).

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

### Panel Icons

Required for marketplace submission. 23x23 with optional transparency. Specify theme variants (`dark`, `darkest`, `light`, `lightest`).

---

## HTML Element Support

Subset of standard HTML. Confirmed working elements:

### Structural
`<html>`, `<head>`, `<body>`, `<script>`, `<style>`, `<link>`, `<div>`, `<span>`, `<p>`, `<a>`, `<h1>`–`<h6>`, `<hr>`, `<footer>`

### Form
- `<input>` — types: `text`, `password`, `number`, `range`, `checkbox`, `radio`, `search`. **NOT**: `type="file"`, `type="color"`.
- `<select>` — works but `<option>` requires explicit `value` attribute or `select.value` returns undefined. Use `setAttribute` or the `selected` attribute.
- `<textarea>` — size cannot be set via `rows`/`cols`; use CSS width/height.
- `<button>` — standard.
- `<label>` — **`<label for="id">` is NOT supported**; wrap label around control. Uses `inline-flex` with wrap (disable with `flex-wrap: nowrap`).
- `<form>` — only `method="dialog"`, never URL submission. Without explicit width, block elements inside won't span full width.
- `<progress>` — works but not theme-aware.

### Media
- `<img>` — works. In dialogs needs explicit `width`/`height` or dialog resizes wrong. Ignores EXIF rotation. Grayscale images fail to render but occupy DOM space. No broken-icon placeholder on failure.
- `<video>` — full HTMLVideoElement API (v7.4.0+): `play()`, `pause()`, `stop()`, `load()`, `fastSeek()`, etc.
- `<canvas>` — supported since v7.0.0 with **many limitations** (see Canvas API section).
- `<webview>` — manifest v5 (UXP 6.0+), **modal dialogs only**. Must whitelist domains.

### Other
- `<dialog>` — modal/non-modal. `showModal()` returns a Promise. `show()` accepts positioning options.
- `<template>`, `<slot>` — custom element infrastructure works.
- `<menu>`, `<menuitem>` — basic support.

### NOT Supported (treated as `<div>`)
- `<ul>`, `<ol>`, `<li>` — style divs as lists with CSS
- `<i>`, `<em>` — use `font-style: italic`
- `<title>`
- `<input type="file">`, `<input type="color">`
- `<iframe>` — use `<webview>` in manifest v5 dialogs

### Unsupported Global Attributes
`accesskey`, `aria-*` (all), `autocapitalize`, `contenteditable`, `contextmenu`, `dir`, `draggable`/`dropzone` (full DnD unsupported), `hidden`, `inputmode`, `is`, `item*`, `part`, `spellcheck`, `translate`. `tabindex` is partial.

**Event handler attributes** (`onclick="..."`) work unreliably. Use `addEventListener`.

**Unitless dimensions** don't work in v3.1.0+: write `width="300px"` or use CSS.

---

## CSS Support

### Supported

**Layout (flexbox-based):**
- `display`: `none`, `inline`, `block`, `inline-block`, `flex`, `inline-flex`
- `flex`, `flex-basis`, `flex-direction`, `flex-grow`, `flex-shrink`, `flex-wrap`
- `align-content`, `align-items`, `align-self`, `justify-content`
- `position` with `top`/`bottom`/`left`/`right`
- `overflow`, `overflow-x`, `overflow-y`

**Box model:**
- `width`, `height`, `min-width`, `min-height`, `max-width`, `max-height`
- `margin`, `padding`, `border` (and directional variants)
- `border-radius` (and corner-specific)

**Typography:**
- `font-family`, `font-size`, `font-style`, `font-weight`
- `color`, `letter-spacing`, `text-align`, `text-overflow`, `white-space`

**Background:**
- `background`, `background-color`
- `background-image` supports `url('plugin://assets/filename')`, `linear-gradient()`, `radial-gradient()`
- `background-size`, `background-attachment`
- `background-repeat` is declared but doesn't actually repeat

**Visual:** `opacity`, `visibility`

**Units:** `px`, `em`, `rem`, `vh`, `vw`, `vmin`, `vmax`, `cm`, `mm`, `in`, `pc`, `pt`

**Features:** `calc()`, CSS Variables (incl. `prefers-color-scheme`), `var()`

**Selectors:** type, class, id, `*`, attribute, child `>`, descendant, adjacent sibling `+`, general sibling `~`

**Pseudo-classes:** `active`, `checked`, `defined`, `disabled`, `empty`, `enabled`, `first-child`, `focus`, `hover`, `last-child`, `nth-child`, `nth-last-child`, `nth-of-type`, `only-child`, `root`

**Pseudo-elements:** `::after`, `::before`

**Media queries:** `width`, `height`, `prefers-color-scheme`

### NOT Supported

| Feature | Status |
|---|---|
| `font` shorthand | Use individual `font-*` |
| `text-transform` | No uppercase/lowercase/capitalize |
| `transition`, `@keyframes`, `transform` | No animations or transforms |
| `box-shadow`, `text-shadow` | Not supported |
| `grid` layout | Flexbox only |
| `position: sticky` | Not supported |
| `z-index` | Limited support |
| `float`, `clear` | Not supported |
| `cursor` | Limited; may not revert |
| `background-repeat` | Declared but non-functional |
| `baseline` alignment | Buggy/unreliable |
| `text-decoration` | Limited |

UXP uses a custom layout engine, **not a browser engine**. Flexbox is the primary mechanism.

---

## Canvas API

### HTMLCanvasElement (v7.0.0+)

```javascript
const canvas = document.createElement('canvas');
canvas.width = 400;
canvas.height = 400;
const ctx = canvas.getContext('2d'); // Only '2d' is supported
```

**NOT available:** `toDataURL()`, `toBlob()`, `getContext('webgl')`, `getContext('webgl2')`, `transferControlToOffscreen()`

### CanvasRenderingContext2D

**Style:** `lineWidth`, `lineJoin`, `lineCap`, `globalAlpha`, `fillStyle`, `strokeStyle`

**Paths:** `beginPath`, `closePath`, `moveTo`, `lineTo`, `arc`, `arcTo`, `bezierCurveTo`, `quadraticCurveTo`, `rect`

**Drawing:** `fill`, `stroke`, `fillRect`, `strokeRect`, `clearRect` (⚠ may not work on Windows)

**Gradients:** `createLinearGradient`, `createRadialGradient` (⚠ may not work on Windows)

**NOT available:** `drawImage`, `getImageData`, `putImageData`, `createImageData`, `fillText`, `strokeText`, `measureText`, `save`, `restore`, `translate`, `rotate`, `scale`, `setTransform`, `clip`, `createPattern`, `globalCompositeOperation`, `shadowBlur`/`shadowColor`/`shadowOffsetX`/`shadowOffsetY`, `imageSmoothingEnabled`

### Path2D (v7.0.0+)

```javascript
const path = new Path2D();
path.moveTo(10, 10);
path.lineTo(100, 100);
path.arc(50, 50, 40, 0, Math.PI * 2);
ctx.fill(path);
```

### Bottom Line

UXP canvas is documented as "only basic shapes for now." For pixel-based visualizations, **you cannot use canvas for real work**:

1. No `drawImage` — can't composite images
2. No `getImageData`/`putImageData` — can't read/write pixels
3. No `toDataURL`/`toBlob` — can't export canvas content
4. No text rendering
5. No transforms
6. No compositing operations
7. Windows-specific breakage in gradients and `clearRect`

**Workarounds:**
- Render pixel buffer in JS (software renderer) → `createImageDataFromBuffer` → `encodeImageData({ base64: true })` → `<img src="data:image/jpeg;base64,...">`
- SVG for vector plots
- Embed a WebView (modal dialog only)
- Hybrid plugin with native C++ rendering

---

## Imaging API

```javascript
const imaging = require('photoshop').imaging;
```

Pixel *writes* require `executeAsModal`. Pixel *reads* via `getPixels()` often work outside modal, but the docs suggest modal is needed for "modifications to Photoshop state." Test per operation.

### PhotoshopImageData Object

**Properties:**
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

**Methods:**
- `getData(options?)` — Promise&lt;Uint8Array | Uint16Array | Float32Array&gt;
  - `options.chunky` (default true)
  - `options.fullRange` (default false) — for 16-bit use [0..65535] instead of [0..32768]
- `dispose()` — synchronous, releases native memory. **Always call when done.**

### Component Value Ranges

| Bit Depth | Type | Range | Notes |
|---|---|---|---|
| 8 | Uint8Array | 0-255 | Standard |
| 16 | Uint16Array | 0-32768 by default | Photoshop's non-standard range. Use `fullRange: true` for 0-65535. |
| 32 | Float32Array | 0.0-1.0+ | HDR may exceed 1.0 |

### Memory Layout

- **Chunky (default, interleaved):** `[R0,G0,B0, R1,G1,B1, ...]` (with alpha: `[R0,G0,B0,A0, ...]`)
- **Planar:** `[R0,R1,R2,..., G0,G1,G2,..., B0,B1,B2,...]`

### `getPixels(options)`

```javascript
const result = await imaging.getPixels({
  documentID: doc.id,           // optional, defaults to active
  layerID: layer.id,            // optional, omit for composite
  historyStateID: stateId,      // optional
  sourceBounds: { left: 0, top: 0, right: 300, bottom: 300 },  // optional
  targetSize: { width: 200, height: 200 },    // downsample via pyramid
  colorSpace: "RGB",
  colorProfile: "sRGB IEC61966-2.1",
  componentSize: 8,             // -1 (source), 8, 16, 32
  applyAlpha: false             // premultiply alpha with white
});
// Returns: { imageData: PhotoshopImageData, sourceBounds, level }

const pixels = await result.imageData.getData();
result.imageData.dispose();    // MANDATORY
```

**Performance:** `targetSize` leverages Photoshop's image pyramid cache — pre-computed half-resolution levels, dramatically faster for previews.

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
const w = 100, h = 100, c = 4;  // RGBA
const buffer = new Uint8Array(w * h * c);
// ... fill buffer ...

const imageData = await imaging.createImageDataFromBuffer(buffer, {
  width: w,                    // REQUIRED
  height: h,                   // REQUIRED
  components: c,               // REQUIRED
  colorSpace: "RGB",           // REQUIRED
  colorProfile: "sRGB IEC61966-2.1",
  chunky: true,                // default
  fullRange: false             // for 16-bit only
});
```

Buffer element count must equal `width * height * components`.

### `encodeImageData(options)` — for `<img>` display

```javascript
const jpegBase64 = await imaging.encodeImageData({
  imageData,                    // REQUIRED — must be RGB
  base64: true
});
img.src = "data:image/jpeg;base64," + jpegBase64;
```

**Only RGB** is encodable — not Lab or Grayscale. Without `base64`, returns `Number[]` (raw JPEG bytes).

### Masks & Selection

```javascript
// Read layer mask (single-channel grayscale)
const maskObj = await imaging.getLayerMask({
  layerID: layer.id,
  kind: "user",  // or "vector"
  sourceBounds: { left: 0, top: 0, right: 300, bottom: 300 },
  targetSize: { height: 100 }
});

// Write mask (pixel masks only, not vector)
await imaging.putLayerMask({
  layerID: layer.id,
  kind: "user",
  imageData: grayImageData,
  replace: true,
  commandName: "Edit Mask"
});

// Selection as grayscale pixels
const selObj = await imaging.getSelection({
  documentID: doc.id,
  sourceBounds: { left: 0, top: 0, right: 300, bottom: 300 }
});
await imaging.putSelection({
  documentID: doc.id,
  imageData: grayImageData,
  replace: true,
  commandName: "Modify Selection"
});
```

### Point Sampling

```javascript
const color = await document.sampleColor({ x: 100, y: 100 });
// SolidColor with .rgb.red, .rgb.green, .rgb.blue
```

### Color Profile Enumeration

```javascript
const rgbProfiles = await require('photoshop').app.getColorProfiles('RGB');
const grayProfiles = await require('photoshop').app.getColorProfiles('Gray');
```

### Performance Guidelines

1. Request smallest possible region via `sourceBounds`
2. Use `targetSize` to scale down
3. `dispose()` **immediately** after extracting pixel data
4. Work in document's native `colorSpace`/`colorProfile` when possible
5. All imaging methods are async — always `await`

---

## Spectrum UXP Components

Native Photoshop-styled widgets following Adobe's Spectrum design system. Since UXP v4.1. Components are automatically theme-aware.

### `sp-button`

```html
<!-- Variants: cta (default), primary, secondary, warning, overBackground -->
<sp-button variant="primary">Click Me</sp-button>
<sp-button variant="warning" quiet>Delete</sp-button>
<sp-button disabled>Disabled</sp-button>

<sp-button variant="primary">
  <sp-icon name="ui:Magnifier" size="s" slot="icon"></sp-icon>
  Search
</sp-button>
```

Event: `click`

### `sp-action-button`

```html
<sp-action-button>An Action</sp-action-button>
<sp-action-button quiet>Quiet Action</sp-action-button>

<!-- Icon-only -->
<sp-action-button style="padding: 0; max-width: 32px; max-height: 32px;">
  <sp-icon name="ui:Magnifier" size="s" slot="icon"></sp-icon>
</sp-action-button>
```

### `sp-checkbox`

```html
<sp-checkbox>Include metadata</sp-checkbox>
<sp-checkbox checked>Pre-checked</sp-checkbox>
<sp-checkbox indeterminate>Partial</sp-checkbox>
<sp-checkbox disabled>Disabled</sp-checkbox>
<sp-checkbox invalid>Invalid</sp-checkbox>
```

Events: `change`, `input` — access `evt.target.checked`

### `sp-dropdown`

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

### `sp-slider`

```html
<sp-slider min="0" max="100" value="50">
  <sp-label slot="label">Opacity</sp-label>
</sp-slider>

<!-- Filled variant with rotation -->
<sp-slider min="0" max="360" value="0" variant="filled" show-value>
  <sp-label slot="label">Rotation</sp-label>
</sp-slider>
```

Attributes: `min`, `max`, `value`, `disabled`, `variant="filled"`, `fill-offset` (`"left"`/`"right"`), `value-label` (e.g. `"%"`), `show-value`
Events: `input` (during drag), `change` (on release) — access `evt.target.value`

### `sp-textfield`

```html
<sp-textfield placeholder="Enter text">
  <sp-label isrequired="true" slot="label">Name</sp-label>
</sp-textfield>

<sp-textfield quiet placeholder="Search..."></sp-textfield>
```

Types: `text` (default), `numeric` (-214748.36 to 214748.36), `search`, `password`
**Known issue:** Password value unreadable on macOS. Workaround: toggle type between `password`/`text` on focus/blur.

### `sp-textarea`

```html
<sp-textarea placeholder="Enter description...">
  <sp-label slot="label">Description</sp-label>
</sp-textarea>
```

### `sp-radio-group` / `sp-radio`

```html
<sp-radio-group>
  <sp-label slot="label">Color Space:</sp-label>
  <sp-radio value="ycbcr">YCbCr BT.601</sp-radio>
  <sp-radio value="ycbcr709">YCbCr BT.709</sp-radio>
  <sp-radio value="hsl">HSL</sp-radio>
</sp-radio-group>

<!-- Vertical -->
<sp-radio-group column>
  <sp-label slot="label">Mode:</sp-label>
  <sp-radio value="scatter">Scatter</sp-radio>
  <sp-radio value="density">Density</sp-radio>
</sp-radio-group>
```

Event: `change` — access `evt.target.value`

### `sp-menu` / `sp-menu-item`

```html
<sp-menu>
  <sp-menu-item>Deselect</sp-menu-item>
  <sp-menu-item>Select Inverse</sp-menu-item>
  <sp-menu-divider></sp-menu-divider>
  <sp-menu-item disabled>Feather...</sp-menu-item>
</sp-menu>
```

`sp-menu-divider` only renders visually inside `sp-dropdown`.
Event: `change` — access `evt.target.selectedIndex`

### `sp-icon`

```html
<sp-icon name="ui:Magnifier" size="m"></sp-icon>
```

Sizes: `xxs`, `xs`, `s`, `m`, `l`, `xl`, `xxl`. 36 built-in icons (chevrons, arrows, checkmarks, alerts, magnifier, cross, star, folder, etc.).

### `sp-progressbar`

```html
<sp-progressbar max="100" value="42" show-value>
  <sp-label slot="label">Processing...</sp-label>
</sp-progressbar>

<sp-progressbar max="100" value="75" size="small"></sp-progressbar>
```

### `sp-divider`

```html
<sp-divider size="large"></sp-divider>  <!-- large, medium (default), small -->
```

### `sp-link`

```html
<sp-link href="https://adobe.com">Adobe</sp-link>
<sp-link quiet>Quiet Link</sp-link>
```

Navigation to `href` cannot be prevented. If no `href`, browser will not launch.

### Typography

```html
<sp-heading size="XXS">Heading</sp-heading>   <!-- XXS to XXXL -->
<sp-body size="S">Body text</sp-body>          <!-- XS, S, M, L, XL -->
<sp-detail size="S">Detail text</sp-detail>
<sp-label>Label text</sp-label>
```

### React + Spectrum

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

---

## Event System

### Action Notification Listener

Document-modifying events (layer changes, selections, filters):

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

Since v23.0. Callback gets the event name (string) and an ActionDescriptor.

**Common events:** layer ops (merge, duplicate, group, ungroup), selection changes (select, deselect, inverse, grow, contract), pixel mods (blur, sharpen, levels, curves, invert), layer effects, transforms (rotate, flip, crop, canvas size, image size).

### Core Notification Listener

UI and OS events (v23.3+):

```javascript
const { core } = require('photoshop');

await core.addNotificationListener('UI', [{ event: 'userIdle' }], (event, descriptor) => {
  if (descriptor.idleEnd) console.log('User returned from idle');
});

await core.addNotificationListener('OS', ['activationChanged'], handler);
```

**UI events:** `userIdle`, `minimizeAppWindow`, `panelVisibilityChanged`, workspace activation/dragging/layout/resizing.
**OS events:** `activationChanged`, `displayConfigurationChanged`.

**`userIdle` is the right trigger for deferred heavy re-renders** — fires after the user stops interacting.

**Important:** Event notifications are suppressed during non-interactive `executeAsModal` scope.

### Modal Event Monitoring

```javascript
await action.addNotificationListener(
  ['modalJavaScriptScopeEnter', 'modalJavaScriptScopeExit'],
  handler
);
```

### DOM Events

Standard DOM events work with limits:
Supported: `click`, `change`, `input`, `focus`, `blur`, `mousedown`/`mouseup`/`mousemove`/`mouseenter`/`mouseleave`, `pointerdown`/`pointerup`/`pointermove`, `keydown`/`keyup`, `wheel`, `resize` (via ResizeObserver), `load` (images).

NOT supported: `keypress`, `dragstart`/`drag`/`dragend`/`drop` (DnD fully unsupported). Interactive elements swallow most events — clicks on buttons don't bubble to parents.

### Observers

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

---

## executeAsModal

Required for any operation that modifies Photoshop state.

```javascript
const { core } = require('photoshop');

async function myDocumentEdit(executionContext, descriptor) {
  if (executionContext.isCancelled) throw "User cancelled";
  executionContext.reportProgress({ value: 0.5, commandName: "Processing..." });

  const suspensionID = await executionContext.hostControl.suspendHistory({
    documentID: app.activeDocument.id,
    name: "My Edit"
  });
  // ... do work ...
  await executionContext.hostControl.resumeHistory(suspensionID);
}

try {
  await core.executeAsModal(myDocumentEdit, {
    commandName: "My Edit",
    descriptor: { myParam: "value" },
    interactive: false,
    timeOut: 5000
  });
} catch (e) {
  if (e.number === 9) console.log("Another plugin holds modal state");
}
```

**Key behaviors:**
- Only one plugin holds modal state at a time
- Menu items disabled during modal
- Progress bar appears after 2+ seconds
- Escape key cancels
- Event notifications suppressed during non-interactive modal scope
- Nested modal scopes share global modal state
- `getPixels()` is read-only and often works outside modal; test per operation

---

## DOM Limitations

### Missing APIs

| API | Status |
|---|---|
| `document.createElement('iframe')` | Use `<webview>` in dialogs (v5) |
| `window.devicePixelRatio` | Always returns 1 — cannot detect HiDPI |
| `localStorage` / `sessionStorage` | Use UXP storage APIs |
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

### Available DOM APIs

`Document`, `DocumentFragment`, `Element`, `Node`, `Text`, `Comment`; `querySelector(All)`, `getElementById`; `createElement`, `createTextNode`, `createDocumentFragment`; `appendChild`, `removeChild`, `insertBefore`, `replaceChild`; `setAttribute`, `getAttribute`, `removeAttribute`; `classList`; `innerHTML`, `outerHTML`, `textContent`, `innerText`; `clientWidth`/`Height`, `offsetWidth`/`Height`, `scrollLeft`/`Top`/`Width`/`Height`; `getBoundingClientRect`; `getComputedStyle`; `AbortController`/`AbortSignal`; `alert`/`confirm`/`prompt`.

### Dialog Behavior

- Closed dialogs remain in DOM — must call `element.remove()`
- `showModal()` returns a Promise (not void like browsers)
- Only one dialog at a time
- `HTMLDialogElement.REJECTION_REASON_NOT_ALLOWED` — another dialog already open
- `HTMLDialogElement.REJECTION_REASON_DETACHED` — node removed from DOM

### `innerHTML` Caveat

Inline event handlers (`onclick="..."`) in `innerHTML` are parsed but behavior is unreliable. Always use `addEventListener`.

---

## Theme Awareness

Photoshop exposes CSS variables that update with theme (Darkest, Dark, Light, Lightest).

### Host CSS Variables

```css
/* Colors */
--uxp-host-background-color
--uxp-host-text-color
--uxp-host-border-color
--uxp-host-link-text-color
--uxp-host-text-color-secondary
--uxp-host-link-hover-text-color
--uxp-host-label-text-color
--uxp-host-widget-hover-background-color
--uxp-host-widget-hover-text-color
--uxp-host-widget-hover-border-color

/* Font sizes */
--uxp-host-font-size
--uxp-host-font-size-smaller
--uxp-host-font-size-larger
```

### Usage

```css
body {
  background-color: var(--uxp-host-background-color);
  color: var(--uxp-host-text-color);
  font-size: var(--uxp-host-font-size);
}

@media (prefers-color-scheme: dark) {
  /* Dark and Darkest */
}
@media (prefers-color-scheme: light) {
  /* Light and Lightest */
}
```

Spectrum UXP components are automatically theme-aware — no extra CSS needed.

---

## Hybrid Plugins (C++ Bridge)

Combine JavaScript UXP code with native C++ dynamic libraries (`.dylib` macOS / `.dll` Windows), similar to Node.js C++ addons. C++ runs natively with full system access.

### Use Cases

- High-performance pixel processing (millions of pixels in C++ instead of JS)
- Custom rendering passed back to JS as image data
- Relaxed file system sandbox (no storage API required)
- Access to Photoshop C++ SDK via `PIUXPSuite`

### Architecture

C++ entry point:
```cpp
export SPErr PSDLLMain(const char* selector, SPBasicSuite* basicSuite, PIActionDescriptor descriptor);
```

### JS ↔ C++ Messaging

**C++ → JS:**
```cpp
sPSUXPSuite->SendUXPMessage(pluginRef, "com.example.myplugin", descriptor);
```
```javascript
require('photoshop').core.addSDKMessagingListener((msg) => {
  console.log(msg.pluginId, msg.content);
});
```

**JS → C++:**
```javascript
require('photoshop').core.sendSDKPluginMessage("NativeComponentID", messageData);
```
```cpp
sPSUXPSuite->AddUXPMessageListener(pluginRef, myCallback);
```

### Build Requirements

- Download Hybrid Plugin SDK from Adobe Developer Console
- Separate build/packaging from standard UXP plugins
- macOS: unsigned binaries require manual security approval
- Debugging: JS via UDT, C++ by attaching debugger to Photoshop.exe

---

## Known Issues

### Critical
1. **Canvas APIs broken on Windows:** `createLinearGradient`, `createRadialGradient`, `clearRect` may fail
2. **No `toDataURL`/`toBlob`** — cannot export canvas content directly
3. **No `drawImage`** — cannot composite images on canvas
4. **No `getImageData`/`putImageData`** — cannot read/write individual pixels on canvas
5. **`window.devicePixelRatio` always returns 1** — cannot detect HiDPI
6. **Panel `show`/`hide` unreliable:** `show` fires once only, `hide` never fires (PS-57284)
7. **No CSS transitions/animations** — all visual changes immediate
8. **No CSS Grid** — flexbox only

### UI
9. No element can overlay text-editing widgets; text fields always render above other content
10. Complex SVG may fail or render unexpectedly
11. Grayscale `<img>` fails to render but occupies DOM space
12. `<img>` in dialogs needs explicit width/height or dialog resizes incorrectly
13. `<img>` ignores embedded EXIF rotation
14. `sp-dropdown` needs explicit width
15. `sp-tooltip` `location` attribute controls tip direction, not position
16. Numeric `sp-textfield` triggers validation errors outside -214748.36 to 214748.36
17. DnD fully unsupported
18. Scroll views don't auto-scroll to focused controls (macOS)
19. `<label for="id">` unsupported — wrap label around control
20. `<option>` requires explicit `value`
21. `<option>` doesn't support `disabled`
22. HTML5 input validation unsupported
23. `<textarea>` size cannot be set via `rows`/`cols`
24. `<form>` only supports `method="dialog"`

### Network
25. Self-signed certs unsupported for secure WebSockets on macOS
26. WebSocket extensions unsupported
27. XHR can only send binary via ArrayBuffer, not Blob
28. XHR doesn't support cookies

### Other
29. Closed dialogs remain in DOM — must call `remove()`
30. `entrypoints.setup()` calls delayed more than ~20ms may cause uncatchable errors
31. Clipboard APIs throw in panel-less plugins
32. `keypress` unsupported — use `keydown`/`keyup`
33. Password field values unreadable on macOS
34. `font` shorthand CSS unsupported
35. `text-transform` CSS unsupported
