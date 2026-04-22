# Adobe Plugin Development Skill

An [agent skill](https://agentskills.io) for building Adobe Photoshop UXP plugins and Lightroom Classic plugins. Gives AI coding agents the SDK knowledge, API references, and hard-won pitfall documentation they need to write correct plugin code on the first try.

## What's Covered

### Photoshop UXP (`uxp-photoshop.md`)

- Manifest v5 configuration and permissions
- Imaging API: `getPixels`, `putPixels`, `createImageDataFromBuffer`, `encodeImageData`
- Canvas API limitations (no `drawImage`, `getImageData`, `putImageData`, `toDataURL`)
- Full Spectrum UXP component catalog (`sp-button`, `sp-slider`, `sp-dropdown`, etc.)
- HTML/CSS support matrix and what's missing (no Grid, no transforms, no animations)
- Event system: action notifications, core events (`userIdle`), DOM events
- `executeAsModal` patterns
- Hybrid plugin C++ bridge
- Panel lifecycle, sizing, and theme awareness
- 35-item known-issues list

### Lightroom Classic SDK (`lrc-sdk.md`)

- Plugin architecture: `Info.lua` manifest, entry points, plugin types
- Pixel access workarounds (there is no direct pixel access from Lua)
- `LrView` UI widgets and data binding
- `LrDevelopController` observation and slider control
- Full module catalog (`LrTasks`, `LrDialogs`, `LrFileUtils`, etc.)
- Memory leak prevention rules for long-running floating dialogs
- External binary bridge patterns (Lua to Rust/C++ CLI)
- Debounce and busy-guard patterns for async coroutines

### Quick Reference (`SKILL.md`)

- Side-by-side capability matrix (PS UXP vs LrC)
- Critical pitfalls with code examples
- Common mistake/fix table
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

- **UXP**: v8.1.0, Manifest v5
- **Photoshop**: v23.3.0+
- **Lightroom Classic SDK**: v15.2

Content was compiled from Adobe's official documentation repositories ([AdobeDocs/uxp-photoshop](https://github.com/AdobeDocs/uxp-photoshop), [AdobeDocs/uxp](https://github.com/AdobeDocs/uxp)) and the LrC 15.2 SDK, supplemented with lessons learned from building [Chromascope](https://github.com/kevinkiklee/chromascope), a chrominance vectorscope plugin for both hosts.

## Contributing

Found an error, outdated API, or missing pattern? PRs welcome. The goal is to keep this current with Adobe's SDK releases so agents don't write code against deprecated or non-existent APIs.

## License

[MIT](LICENSE)
