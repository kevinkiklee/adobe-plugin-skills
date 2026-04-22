# Adobe Plugin Development Skill

An [agent skill](https://agentskills.io) for building Adobe Photoshop UXP plugins and Lightroom Classic plugins. Gives AI coding agents the SDK knowledge, API references, and hard-won pitfall documentation they need to write correct plugin code on the first try.

## What's Covered

### Photoshop UXP (`uxp-photoshop.md`)

- Document class (28 properties, 28 methods) and Layer class (27 properties, 46 methods including 30+ filters)
- Selection class, LayerComp, ColorSampler, PathItem, Guide, HistoryState
- Manifest v5 configuration, permissions, and feature flags (`CSSNextSupport`, `enableSWCSupport`)
- Imaging API: `getPixels`, `putPixels`, `createImageDataFromBuffer`, `encodeImageData`, masks, selection
- batchPlay reference forms (ID, Index, Name, Enum, Property), multiGet, Action Recording (v25.0+)
- Canvas API limitations (no `drawImage`, `getImageData`, `putImageData`, `toDataURL`)
- Full Spectrum UXP component catalog (19 `sp-*` widgets, 40 icons) + SWC overview
- HTML/CSS support matrix with CSSNextSupport feature flag details
- Text APIs: TextItem, CharacterStyle (34 properties), ParagraphStyle, WarpStyle
- Color management: SolidColor, color classes, `convertColor`, `getColorProfiles`
- Event system, `executeAsModal` (with `timeOut`, `registerAutoCloseDocument`)
- App/Core module APIs, Preferences (13 groups), prototype extensions
- Hybrid plugin C++ bridge, WebView (panels since v6.4), network APIs
- 35+ known issues catalog

### Lightroom Classic SDK (`lrc-sdk.md`)

- Complete `Info.lua` manifest reference (23 fields), plugin lifecycle, Lua 5.1.5 sandbox details
- 5-stage export pipeline, export/publish service provider callbacks, export filter provider
- Memory leak prevention rules (6 non-negotiable rules with code) for long-running floating dialogs
- `LrCatalog` write gates, batch operations, `findPhotos` with search descriptors
- `LrPhoto` methods, 40+ `getRawMetadata` keys, 70+ `getFormattedMetadata` keys, custom metadata
- `LrView` 25 factory methods, 6 property categories, data binding patterns (basic, transform, multi-key, cross-table)
- `LrDialogs` 14 functions, `LrDevelopController` 94 functions grouped by category
- 117 develop parameters by panel, local adjustment parameters (process versions 2-6)
- Controller SDK architecture (LrSocket, external driver, TCP protocol)
- ~40 module quick reference (LrApplication, LrSelection, LrHttp, LrDigest, etc.)
- External binary bridge patterns with platform detection (Lua to Rust/C++ CLI)

### Quick Reference (`SKILL.md`)

- Side-by-side capability matrix (PS UXP vs LrC) with DOM model, AI features, Selection API
- Critical pitfalls with code examples (CSSNextSupport flag, executeAsModal anti-patterns, memory leaks)
- 117 develop parameters by panel, local adjustment parameters by process version
- Complete `Info.lua` manifest field reference (23 fields)
- Data binding patterns for LrView
- Common mistake/fix table (16 entries)
- Ready-to-use code snippets for the most frequent operations

## Installation

### Claude Code

```bash
# Clone into your skills directory
git clone https://github.com/kevinkiklee/adobe-plugin-skills.git ~/.claude/skills/adobe-plugin-development

# Or symlink if you cloned elsewhere
ln -s /path/to/adobe-plugin-skills ~/.claude/skills/adobe-plugin-development
```

The skill activates automatically when you work on Photoshop UXP or Lightroom Classic plugin code.

### Other Agents

Copy or symlink the skill directory into your agent's skill discovery path. The skill follows the [agentskills.io specification](https://agentskills.io/specification) with standard YAML frontmatter in `SKILL.md`.

## SDK Versions

- **UXP**: v9.2.0, Manifest v5
- **Photoshop**: v27.4
- **Lightroom Classic SDK**: v15.2+

Content was compiled from Adobe's official documentation repositories ([AdobeDocs/uxp-photoshop](https://github.com/AdobeDocs/uxp-photoshop), [AdobeDocs/uxp](https://github.com/AdobeDocs/uxp)) and the LrC 15.2 SDK, supplemented with lessons learned from building [Chromascope](https://github.com/kevinkiklee/chromascope), a chrominance vectorscope plugin for both hosts.

## Contributing

Found an error, outdated API, or missing pattern? PRs welcome. The goal is to keep this current with Adobe's SDK releases so agents don't write code against deprecated or non-existent APIs.

## License

[MIT](LICENSE)
