# Changelog

All notable changes to this skill are documented here. Versioning follows [Semantic Versioning](https://semver.org/).

## [1.0.0] — 2026-04-22

Initial public release.

### Coverage

- **Photoshop UXP** v9.2.0 / Manifest v5 / Photoshop 27.4 (February 2026)
- **Lightroom Classic** SDK 15.3 / Lua 5.1.5

### Files

- `SKILL.md` — agent-facing entry point with capability matrix, critical pitfalls, quick-reference snippets, develop parameter catalog, Info.lua reference, and common mistakes
- `ps-uxp-sdk.md` — deep Photoshop UXP reference (Document/Layer DOM, Imaging API, batchPlay, executeAsModal, Spectrum/SWC, Canvas limits, hybrid C++ bridge, known issues)
- `lrc-sdk.md` — deep Lightroom Classic SDK reference (export/publish pipeline, catalog/photo APIs, LrView, LrDevelopController 95 functions, memory-leak prevention, Controller SDK)

### Audit fixes (2026-04-22)

Audited every API claim against the published Adobe SDK references. Corrections:

- Document property count (30 → 33)
- Mask function signatures (`selectMask`, `deleteMask`, etc.) now show actual `(id, param)` form
- Removed deprecated `permissions.webview.allow` field from manifest examples (removed in UXP 9.1)
- `LrDialogs.presentFloatingDialog` corrected to `(plugin, args)` signature
- `LrDialogs.stopModalWithResult` corrected to `(dialog, result)` signature
- `catalog:withPrivateWriteAccessDo` examples no longer include a bogus action-name argument
- Hybrid C++ bridge messaging API names corrected to `photoshop.messaging.*` (`addSDKMessagingListener`, `removeSDKMessagingListener`, `sendSDKPluginMessage`)
