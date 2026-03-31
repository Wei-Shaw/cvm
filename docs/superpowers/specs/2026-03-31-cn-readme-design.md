# CN README with Language Switching

## Overview

Add a complete Chinese translation of the README and support language switching between English and Chinese via top-level links.

## Approach: Dual File + Top-Level Language Links

### Files

- **`README.md`** — existing English README, add language switcher at the top
- **`README_CN.md`** — new file, full Chinese translation with identical structure

### Language Switcher

Both files get a top line:

```
[English](README.md) | [中文](README_CN.md)
```

Placed as the first line, before the `# CVM` title, so readers see it immediately on GitHub.

### Translation Conventions

- Commands, code blocks, file paths: keep English
- Descriptive text: translate to Chinese
- Technical terms: keep English or annotate in both languages (shim, patch, symlink, junction, etc.)
- Tables, directory structures: same format as English version
- Sections map 1:1 between the two files for easy maintenance

### Sections to Translate

All sections in README.md will be translated:

1. Title and tagline
2. Why CVM?
3. Quick Start
4. Installation (Prerequisites, From npm, From Source, Verify)
5. Commands (setup, install, uninstall, use, current, list, patch proxy, patch revert, patch status)
6. How It Works (Directory Layout, Shim Mechanism, Proxy Patching)
7. Platform Support
8. Configuration
9. Development (including Project Structure and Design Principles)
10. Contributing
11. License

### Out of Scope

- No automated translation sync tooling
- No i18n framework — these are static Markdown files
- No changes to code, CLI output, or package.json
