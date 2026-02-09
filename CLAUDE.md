# CLAUDE.md

This file provides context for Claude Code when working on this repository.

## Project Overview

This is the **Seeed Studio Wiki Platform** -- a large, multilingual documentation website for Seeed Studio's IoT hardware products built with **Docusaurus 3.8.1**. It covers sensors, networking solutions, edge computing devices, SenseCraft cloud services, and more.

**Repository**: `Seeed-Studio/wiki-documents`

## Tech Stack

- **Framework**: Docusaurus 3.8.1 (React 18, TypeScript 4.7)
- **Styling**: SASS/SCSS modules (`.module.scss`)
- **UI Library**: Ant Design 5.x
- **State Management**: Zustand 5.x
- **Search**: Typesense
- **Package Manager**: Yarn
- **Node.js**: >=18 required

## Key Commands

```bash
# Development (single language is faster due to 8,687+ docs)
yarn start:en          # English only dev server
yarn start:cn          # Chinese only dev server
yarn start:en+cn       # Both English and Chinese

# Full dev server (all languages, requires 8GB+ RAM)
yarn start

# Production build (requires 6GB+ RAM)
yarn build

# Type checking
yarn typecheck

# Clear Docusaurus cache
yarn clear

# Generate wiki pages
yarn generatewiki

# Generate language maps
yarn generate-lang-map
```

## Project Structure

```
docs/                  # 8,687+ markdown documentation files
  ├── Cloud_Chain/     # SenseCraft AI, Data Platform
  ├── Contribution/    # Contribution guides
  ├── Edge/            # BeagleBone, EdgeBox, Jetson
  ├── Network/         # LoRa, Meshtastic, SenseCAP
  ├── Sensor/          # Grove, XIAO, SenseCAP sensors
  ├── Solutions/       # Industry solutions
  ├── Topics/          # Home Assistant, TinyML, Edge AI
  ├── zh-CN/           # Chinese translations
  ├── ja/              # Japanese translations
  └── es/              # Spanish translations
src/                   # Frontend source code
  ├── components/      # React components
  ├── pages/           # Page components
  ├── theme/           # Custom Docusaurus theme overrides
  ├── css/             # Global stylesheets
  ├── utils/           # Utility functions
  ├── stores/          # Zustand state stores
  └── types/           # TypeScript type definitions
scripts/               # Build and utility scripts (JS + Python)
static/                # Static assets (images, etc.)
blog/                  # Blog posts
docusaurus.config.js   # Main Docusaurus configuration
sidebars.js            # Navigation sidebar (20,000+ lines)
```

## Documentation Conventions

- **Frontmatter**: All docs use YAML frontmatter for metadata (title, slug, aliases, etc.)
- **Headings**: Start at level 2 (`##`), not level 1
- **Lists**: Use dashes (`-`) for unordered lists
- **Emphasis**: Use asterisks (`*`) for italic and bold, not underscores
- **Tabs**: No hard tabs; use spaces
- **Line length**: No limit enforced (MD013 disabled)
- **Inline HTML**: Allowed (MD033 disabled)
- **Images**: Alt text not required (MD045 disabled)
- **Categories**: Organized via `_category_.yml` files in doc directories
- **i18n**: 4 languages supported -- English (default), Chinese (zh-CN), Japanese (ja), Spanish (es)

## Code Conventions

- **React components**: PascalCase filenames (e.g., `KnowledgebasePage.tsx`)
- **Utility files**: camelCase filenames (e.g., `locale.ts`)
- **CSS modules**: `*.module.scss` pattern for scoped styles
- **Config files**: Language-specific variants (e.g., `config.en.js`, `config.zh.js`)
- **TypeScript**: Uses Node16 module resolution with Docusaurus preset

## CI/CD

- **Build validation**: PRs are tested with `yarn docusaurus build` via GitHub Actions
- **Deployment**: Auto-deploys from `docusaurus-version` branch to GitHub Pages via SSH
- **Translation**: Automated AI translation triggered by `/translate` command on PRs (admin/maintainer only)
- **Link checking**: Markdown URL validation workflow (manual trigger)
- **Memory**: CI workflows allocate 6GB extra swap due to the large doc count

## Important Notes

- This is a very large site. Always use `yarn start:en` (single language) for local development to avoid excessive memory usage.
- The `sidebars.js` file is ~20,000 lines. Avoid reading the entire file; search for specific sections instead.
- The `docusaurus.config.js` extracts frontmatter aliases for redirects during build (not during `start`).
- When adding new documentation, ensure frontmatter includes at minimum: `title`, `slug`, and `description`.
- Translation files live in separate directories (`docs/zh-CN/`, `docs/ja/`, `docs/es/`), not in an i18n folder.
