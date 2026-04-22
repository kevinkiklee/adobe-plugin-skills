# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

An AI agent skill (following the [agentskills.io](https://agentskills.io) specification) that provides SDK knowledge for building Adobe Photoshop UXP plugins and Lightroom Classic plugins. There is no build system, no source code, and no tests — the deliverable is the markdown files themselves.

## Repository Structure

- **`SKILL.md`** — Skill entry point with YAML frontmatter (~530 lines). The "90% file" that loads on every activation. Contains capability matrix, critical pitfalls, quick-reference code snippets, develop parameter catalog, Info.lua reference, data binding patterns, and common mistakes table.
- **`ps-uxp-sdk.md`** — Deep Photoshop UXP reference (~1150 lines), organized by use-case. Covers Document/Layer DOM model, batchPlay, Imaging API, Spectrum components, CSS support, Text/Color APIs, executeAsModal, hybrid C++ bridge, and 35+ known issues.
- **`lrc-sdk.md`** — Deep Lightroom Classic SDK reference (~1060 lines), organized by use-case. Covers plugin architecture, export/publish pipeline, memory leak prevention, catalog/photo APIs, LrView/LrDialogs, data binding, 95 LrDevelopController functions, and ~40 module quick reference.
- **`README.md`** — Installation instructions and project overview for GitHub.

## SDK Versions Covered

- UXP v9.2.0, Manifest v5
- Photoshop v27.4
- Lightroom Classic SDK v15.3

## Editing Guidelines

- Content was compiled from Adobe's official docs ([AdobeDocs/uxp-photoshop](https://github.com/AdobeDocs/uxp-photoshop), [AdobeDocs/uxp](https://github.com/AdobeDocs/uxp)) and the LrC 15.3 SDK, supplemented with lessons from building [Chromascope](https://github.com/kevinkiklee/chromascope).
- When updating API references, verify against Adobe's current documentation — APIs change between SDK releases.
- `SKILL.md` is the agent-facing "90% file" — it should cover most common use cases without needing the deep references. Deep details and complete API tables belong in `ps-uxp-sdk.md` or `lrc-sdk.md`, which are organized by use-case.
- The YAML frontmatter in `SKILL.md` (`name`, `description`) controls how agent frameworks discover and trigger this skill. Changes there affect activation behavior.

## Installation (for reference)

Installed as a Claude Code skill by cloning/symlinking into `~/.claude/skills/adobe-plugin-development`. No dependencies to install.
