# Adobe Plugin Skill Rewrite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite SKILL.md, uxp-photoshop.md, and lrc-sdk.md with comprehensive API coverage, verified corrections, and practical patterns following the tiered approach described in the design spec.

**Architecture:** Three independent markdown files. SKILL.md is the "90% file" that loads on every skill activation (~550 lines). The two deep reference files (uxp-photoshop.md ~1100 lines, lrc-sdk.md ~1000 lines) are organized by use-case and loaded only for non-trivial work. All content is sourced from research already captured in the design spec at `docs/superpowers/specs/2026-04-22-adobe-plugin-skill-improvement-design.md`.

**Tech Stack:** Markdown files only. No build system, no tests, no dependencies.

**Key reference:** The design spec contains all section-level content plans, corrections, API lists, code snippets, and the content preservation checklist. Every task below references specific sections of that spec. Read the spec before starting any task.

**Parallelism:** Tasks 1, 2, and 3 are fully independent (separate files, no shared state) and can run as parallel subagents. Tasks 4-6 depend on Tasks 1-3 completing first.

---

### Task 1: Write SKILL.md — the "90% file"

**Files:**
- Rewrite: `SKILL.md`
- Reference: `docs/superpowers/specs/2026-04-22-adobe-plugin-skill-improvement-design.md` (sections "SKILL.md — Content Plan" 1-8)

This is the most critical file — it loads on every skill activation. It must be self-contained for ~90% of use cases.

- [ ] **Step 1: Read the current SKILL.md and the design spec**

Read `SKILL.md` (189 lines) to internalize existing content. Read the design spec sections "SKILL.md — Content Plan" 1-8 and "Key Corrections" table. Every existing code snippet, pitfall, and table entry must be preserved (corrected where noted).

- [ ] **Step 2: Write the complete SKILL.md**

Rewrite `SKILL.md` in full (~530-580 lines) following the spec section by section:

**Section structure (in order):**
1. Frontmatter — keep name `adobe-plugin-development` and description unchanged
2. When to Use / Don't Use — preserve current list, add: batchPlay operations, Document/Layer DOM manipulation, Action Recording
3. Core Capability Matrix — update table with new rows: Document DOM model, AI features, Selection API
4. Critical Pitfalls — all UXP pitfalls (preserved + new per spec section 4), all LrC pitfalls (preserved + new per spec section 4). CRITICAL CORRECTIONS: CSS section must note CSSNextSupport feature flag requirement; localStorage/sessionStorage must be noted as available
5. UXP Quick Reference — 5 code snippets (3 preserved + 2 new: Document/Layer ops, batchPlay basics), Spectrum catalog (icon count 40), CSS summary (corrected), Manifest v5 essentials (add featureFlags), Event system, executeAsModal (add timeOut, registerAutoCloseDocument, anti-pattern)
6. LrC Quick Reference — 4 code snippets (preserved + busy-guard pattern added to debounce), Info.lua table (all 20+ fields), data binding patterns, develop parameter list (dense format: params listed per panel), local adjustment params by process version
7. Common Mistakes Table — preserve 11 existing entries + add 5 new per spec
8. Deeper References — updated pointers

**Format rules:**
- Develop parameters: use dense "Panel: param1, param2, ..." format, NOT one-row-per-param tables
- Code snippets: 5-15 lines with comments only for non-obvious behavior
- Version annotations inline: "method(params) — description. (SDK X.Y+)"
- No prose paragraphs — this is agent reference material, not a tutorial

- [ ] **Step 3: Verify content preservation**

Check the written file against the design spec's "Content Preservation Checklist":
- All 7 code snippets from original SKILL.md present (3 UXP + 4 LrC)
- All 11 original common mistakes entries present + 5 new
- All UXP pitfalls present (canvas, imaging API, layout, panel show/hide, label, option)
- All LrC pitfalls present (6 memory leak rules, no develop panel, no photo-changed observer)
- Busy-guard pattern promoted from lrc-sdk.md
- Capability matrix present with new rows
- 6 key corrections applied (CSS, localStorage, WebView, icon count, version, develop params)

- [ ] **Step 4: Verify line count**

Run: `wc -l SKILL.md`
Expected: 500-600 lines. If significantly over 600, compress the develop parameter listing or table formatting. If under 500, check for missing content.

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "Rewrite SKILL.md as comprehensive '90% file'

Expand from 189 to ~550 lines. Add 117 develop parameters, complete
Info.lua reference, data binding patterns, new pitfalls (CSSNextSupport
flag, executeAsModal anti-pattern, batchPlay error handling). Correct
CSS support (feature flag required), localStorage (available), WebView
(panels since v6.4). Preserve all existing code snippets and pitfalls."
```

---

### Task 2: Write uxp-photoshop.md — deep UXP reference

**Files:**
- Rewrite: `uxp-photoshop.md`
- Reference: `docs/superpowers/specs/2026-04-22-adobe-plugin-skill-improvement-design.md` (sections "uxp-photoshop.md — Content Plan" 1-12)

This file is loaded when agents need deep UXP API details beyond what SKILL.md covers.

- [ ] **Step 1: Read the current uxp-photoshop.md and the design spec**

Read `uxp-photoshop.md` (969 lines) to internalize all existing content. Read the design spec sections "uxp-photoshop.md — Content Plan" 1-12. Pay special attention to what's PRESERVED vs CORRECTED vs NEW.

- [ ] **Step 2: Write the complete uxp-photoshop.md**

Rewrite `uxp-photoshop.md` in full (~1080-1150 lines) following the spec section by section:

**Section structure (in order):**
1. Header — version baseline UXP 9.2.0 / PS 27.4, "not a browser" warning
2. Building a Panel Plugin (~120 lines) — manifest v5 full spec (preserved JSON + corrected permissions + NEW featureFlags/host.data), panel lifecycle (preserved + PS-57284), panel sizing (preserved), theme awareness (preserved CSS vars), plugin packaging (NEW .ccx)
3. Document & Layer Model (~200 lines) — ALL NEW: Document class (28 props table + 28 methods table with gotchas), Layer class (27 props + 16 non-filter methods + 30 filter methods in compact table), Selection class (5 props + 21 methods), other classes table (LayerComp, ColorSampler, CountItem, PathItem, Guide, HistoryState)
4. Working with Pixels (~100 lines) — PRESERVED with minor additions: imaging API, PhotoshopImageData, component ranges, memory layout, all 8 methods, performance guidelines, display pipeline
5. batchPlay & Action System (~60 lines) — ALL NEW: descriptor structure, 5 reference forms, multiGet, batchPlaySync, Action Recording (v25.0+), descriptor discovery, error handling, action module (7 functions)
6. executeAsModal (~50 lines) — EXPANDED: preserved code + NEW timeOut, registerAutoCloseDocument, exception anti-pattern, modal collision retry, nested scopes, interactive mode
7. Building UIs (~250 lines) — HTML elements (preserved + audio/video additions), CSS (CORRECTED with CSSNextSupport flag), Spectrum (preserved + icon count 40 + SWC overview), React+Spectrum (preserved), DOM limitations (CORRECTED localStorage/sessionStorage), dialog behavior (preserved), Canvas API (preserved)
8. Text & Typography (~40 lines) — ALL NEW: TextItem, CharacterStyle (34 props), ParagraphStyle (14 props), WarpStyle (5 props)
9. Color Management (~25 lines) — ALL NEW: SolidColor, color classes with ranges, convertColor, getColorProfiles
10. External Communication (~50 lines) — preserved + corrected: network APIs (+ FormData), file system (+ createReadStream), WebView (CORRECTED panel support + local HTML + optional domains), hybrid C++ (preserved)
11. App & Core APIs (~40 lines) — ALL NEW: app class (9 props + 7 methods), core module key functions, Preferences API, prototype extensions
12. Known Issues (~40 lines) — preserved 35 items + CSSNextSupport annotations + lockDocumentFocus workaround

**Format rules:**
- Document/Layer property tables: compact with columns Property | Type | R/W | Min Version | Notes
- Filter methods: grouped by category (Blur, Sharpen, Noise, etc.) in a single compact table
- Organize by use-case (the section headers ARE the use-cases), not by API module
- Existing code snippets preserved exactly (Spectrum component examples, React pattern, event listener examples, imaging examples, executeAsModal example)

- [ ] **Step 3: Verify content preservation**

Check against the design spec's content preservation audit (all 50 items unique to uxp-photoshop.md):
- Full manifest JSON example present
- All 7 permission types in table
- Panel lifecycle code snippet present
- Complete HTML element support catalog (structural, form, media, not-supported)
- Complete CSS support/not-supported sections (with corrections)
- Canvas API method catalog and Path2D snippet
- All 8 imaging API method signatures with full options
- Masks & Selection imaging API (getLayerMask, putLayerMask, getSelection, putSelection)
- Point sampling and color profile enumeration
- All 19 Spectrum component examples with attributes/events
- React + Spectrum code snippet
- Action/Core notification listener examples and DOM events catalog
- executeAsModal code with suspendHistory/resumeHistory
- All DOM limitations and available APIs lists
- Dialog behavior (REJECTION_REASON constants)
- Theme awareness (13 CSS vars + 3 font-size vars)
- Hybrid C++ bridge (PSDLLMain, JS↔C++ messaging, build requirements)
- All 35 known issues

- [ ] **Step 4: Verify line count**

Run: `wc -l uxp-photoshop.md`
Expected: 1050-1200 lines. The Document & Layer Model section (~200 lines) is the largest new addition.

- [ ] **Step 5: Commit**

```bash
git add uxp-photoshop.md
git commit -m "Rewrite uxp-photoshop.md with full DOM model and corrections

Update from UXP 8.1.0 to 9.2.0 / PS 27.4. Add Document class (28 props,
28 methods), Layer class (27 props, 46 methods incl 30 filters), Selection,
batchPlay reference forms, Action Recording, Text/Color APIs, app/core
modules. Correct CSS (CSSNextSupport flag), localStorage (available),
WebView (panels since v6.4). Reorganize by use-case."
```

---

### Task 3: Write lrc-sdk.md — deep LrC reference

**Files:**
- Rewrite: `lrc-sdk.md`
- Reference: `docs/superpowers/specs/2026-04-22-adobe-plugin-skill-improvement-design.md` (sections "lrc-sdk.md — Content Plan" 1-9)

This file is loaded when agents need deep LrC SDK details beyond what SKILL.md covers.

- [ ] **Step 1: Read the current lrc-sdk.md and the design spec**

Read `lrc-sdk.md` (310 lines) to internalize all existing content. Read the design spec sections "lrc-sdk.md — Content Plan" 1-9. This file grows the most proportionally (310 → ~1000 lines).

- [ ] **Step 2: Write the complete lrc-sdk.md**

Rewrite `lrc-sdk.md` in full (~1000-1060 lines) following the spec section by section:

**Section structure (in order):**
1. Header — LrC 15.2+ SDK, Lua 5.1.5, sandbox constraints
2. Building a Plugin (~100 lines) — Info.lua field reference (all 20+ fields in table), restricted loader environment, plugin lifecycle (loading sequence, shutdown protocol with 10-second timeout, LrDialogs unavailable during shutdown), installation paths per platform, Lua environment (available/blocked globals and namespaces), SDK version history (1.3-6.0+), debugging (LrLogger, log locations, deprecation tracking)
3. Building an Export/Publish Service (~90 lines) — 5-stage pipeline, export service provider callbacks table, export processing loop code snippet, export property keys table, publish service additions, export filter provider pattern
4. Building a Live Develop Visualization (~80 lines) — PRESERVED architecture (5-step process, key challenges) + PRESERVED memory leak prevention (all 6 rules with code snippets including busy-guard pattern) + condensed external binary bridge (keep platform detection code, trim Chromascope-specific CLI flags)
5. Working with the Catalog (~110 lines) — LrCatalog key methods (write gates, batch ops, findPhotos with searchDesc), LrPhoto key methods (getRawMetadata, getFormattedMetadata, setRawMetadata, getDevelopSettings with typo warnings), getRawMetadata key table (all 40+ keys), getFormattedMetadata key table (70+ keys grouped by category), setRawMetadata writable keys, custom metadata
6. Building UIs (~80 lines) — LrView factory methods table (all 25), property categories (6 categories with all properties), data binding patterns with code (basic, transform, multi-key, cross-table, conditional, observers), LrDialogs (all 14 functions with key signatures), view layout patterns (shared widths, overlapping, 'skipped item', repeat...until true)
7. External Communication (~40 lines) — LrSocket (bind, modes, callbacks), controller SDK architecture (two-component, socket protocol, main loop pattern), LrHttp (get/post/postMultipart, error codes, Content-Type='skip'), LrTasks.execute, LrShell
8. Develop Parameter Reference (~100 lines) — complete 117 params by panel (dense format), local adjustment params by process version (v2-v6), LrDevelopController functions grouped by category (Get/Set, Observation, Tools, Masks, Spots, AI/Enhance, Point Color, Lens Blur, Color Grading, Process Version, Other), enumerated values (panel IDs, tool names, mask types/subtypes, spot types, bokeh types, process versions)
9. Module Quick Reference (~60 lines) — ALL modules with purpose and key methods (LrApplication, LrApplicationView, LrSelection, LrTasks, LrFunctionContext, LrBinding, LrObservableTable, LrExportSession/Rendition/Context/Settings, LrColor, LrDate, LrFileUtils, LrPathUtils, LrHttp, LrSocket, LrShell, LrLogger, LrPrefs, LrStringUtils, LrDigest, LrMD5, LrXml, LrErrors, LrRecursionGuard, LrProgressScope, LrSystemInfo, LrLocalization, LrPasswords, LrFtp, LrTether, LrSounds, LrUndo, LrSlideshow, LrPlugin, LrPhotoInfo, LrVideoExportPreset, LrFilterContext, LrWebViewFactory, publish classes, collection classes)

**Format rules:**
- Develop parameters: dense "Panel: param1, param2, ..." format matching SKILL.md
- Metadata key tables: grouped by SDK version, compact format (Key | Type | Notes)
- Module quick reference: inline "Module (key method1, method2, method3)" format — NOT separate tables per module
- LrDevelopController functions: grouped by category, not alphabetical
- Code snippets preserved from original lrc-sdk.md: Info.lua manifest, frame alternation, requestJpegThumbnail guard, debounce pattern (+ busy-guard), temp cleanup, picture live-update, platform detection

- [ ] **Step 3: Verify content preservation**

Check against the design spec's content preservation audit (all 21 items unique to lrc-sdk.md):
- Info.lua manifest code snippet with LrSdkVersion, LrSdkMinimumVersion, LrToolkitIdentifier, VERSION table
- Entry points beyond SKILL.md (LrExportMenuItems, LrExportServiceProvider, LrMetadataProvider)
- photo:getDevelopSettings() and photo:getRawMetadata(key) details
- LrExportSession for rendering
- Full LrView widget catalog (all layout, input, display controls)
- LrView.bind() and LrObservableTable
- LrDialogs.presentModalDialog vs presentFloatingDialog details
- LrDevelopController full API (now 94 functions)
- Full Key Modules table (expanded to ~40 modules)
- Recommended Architecture (5-step process + 4 key challenges)
- Busy-guard + pending flag coalescing pattern code block
- Temp file cleanup code snippet
- External binary bridge (condensed, keep platform detection)
- Comparison table (absorbed into capability notes where relevant)
- "JPEG thumbnails may not reflect current develop settings" caveat
- "External binary adds distribution complexity" caveat
- Plugin types "Web Gallery" and "Library Filter"
- SDK version 15.2

- [ ] **Step 4: Verify line count**

Run: `wc -l lrc-sdk.md`
Expected: 950-1100 lines. The getRawMetadata + getFormattedMetadata key tables (~120 lines combined) and develop parameter reference (~100 lines) are the largest sections.

- [ ] **Step 5: Commit**

```bash
git add lrc-sdk.md
git commit -m "Rewrite lrc-sdk.md with complete API coverage

Expand from 310 to ~1000 lines. Add 94 LrDevelopController functions,
117 develop parameters, 40+ getRawMetadata keys, 70+ getFormattedMetadata
keys, complete Info.lua field reference, 5-stage export pipeline, LrView
25 factory methods, LrDialogs 14 functions, data binding patterns,
LrSelection/LrApplicationView APIs, controller SDK architecture.
Reorganize by use-case."
```

---

### Task 4: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read the current README.md**

Read `README.md` (70 lines).

- [ ] **Step 2: Update version references and coverage summary**

Update:
- SDK Versions section: UXP baseline from v8.1.0 to v9.2.0, Photoshop from v23.3.0+ to v27.4
- "What's Covered" section: update bullet points to reflect expanded coverage (e.g., "94 LrDevelopController functions", "Document/Layer DOM model", "batchPlay reference forms", "117 develop parameters")
- Keep installation instructions, contributing section, and license unchanged

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Update README with expanded coverage and version info"
```

---

### Task 5: Cross-file consistency check

**Files:**
- Read: `SKILL.md`, `uxp-photoshop.md`, `lrc-sdk.md`

- [ ] **Step 1: Verify cross-file consistency**

Check that:
- SKILL.md's develop parameter list matches lrc-sdk.md section 8 exactly (same parameter names, same panel groupings)
- SKILL.md's "Deeper References" section accurately describes what's in each deep reference file
- Version baselines are consistent: UXP 9.2.0 / PS 27.4 in both SKILL.md and uxp-photoshop.md; LrC SDK 15.2 in both SKILL.md and lrc-sdk.md
- Code snippets that appear in both SKILL.md and a deep reference file are identical
- The 6 key corrections are consistently applied across all files (CSS feature flag, localStorage, WebView, icon count, develop params, versions)
- No broken internal references ("see section X" pointing to sections that don't exist)

- [ ] **Step 2: Fix any inconsistencies found**

If any inconsistencies are found, fix them in the file that deviates from the spec. The spec is the source of truth.

- [ ] **Step 3: Commit fixes (if any)**

```bash
git add SKILL.md uxp-photoshop.md lrc-sdk.md
git commit -m "Fix cross-file consistency issues"
```

Only commit if changes were made.

---

### Task 6: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Read the current CLAUDE.md**

Read `CLAUDE.md`.

- [ ] **Step 2: Update SDK versions and file descriptions**

Update the "SDK Versions Covered" section to match the new version baselines. Update file descriptions if the restructuring changed what each file covers (e.g., note that files are now "organized by use-case").

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "Update CLAUDE.md with new SDK versions and file structure"
```
