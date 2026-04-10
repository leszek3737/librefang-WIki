# Documentation — articles

# Documentation — Articles Module

## Overview

The `articles/` directory contains the written content for the LibreFang documentation website. These Markdown files serve as the primary source for blog posts, release announcements, announcements, and informational articles that appear on [docs.librefang.ai](https://docs.librefang.ai).

Articles are static content files that complement the generated API documentation and guides. They cover release notes, project announcements, and feature introductions.

## Article Types

The module contains three categories of articles:

### 1. Introduction Articles

General-purpose articles that introduce the project to new users. These provide high-level overviews, quick start guides, and contribution information.

**Example:** `hello-librefang.md`

```
articles/
└── hello-librefang.md   # Main project introduction
```

### 2. Announcement Articles

Articles covering significant project milestones such as website launches, new features, or community updates.

**Example:** `new-website-launch.md`

### 3. Release Notes

Serialized articles documenting each version of LibreFang. These follow a consistent format across versions 0.5.6 through the current CalVer releases.

**Pattern:** `release-{version}.md`

```
articles/
├── release-0.5.6.md
├── release-0.5.7.md
├── release-0.6.0.md
├── release-0.6.1.md
├── release-0.6.2.md
├── release-0.6.3.md
├── release-0.6.4.md
├── release-0.6.5.md
├── release-0.6.6.md
├── release-0.6.7.md
├── release-0.6.8.md
├── release-0.7.0.md
├── release-2026.3.21.md
└── release-2026.3.22.md
```

## Frontmatter Schema

Every article must include valid YAML frontmatter at the top of the file. This metadata powers the documentation site's routing, SEO, and listing features.

```yaml
---
title: "Article Title"
published: true
description: "Short description for SEO and previews"
tags: [tag1, tag2, tag3]
canonical_url: https://github.com/librefang/librefang/releases/tag/vX.X.X
cover_image: https://raw.githubusercontent.com/librefang/librefang/main/public/assets/logo.png
---
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | The article heading and page title |
| `published` | boolean | Whether the article is publicly visible |
| `description` | string | Meta description for SEO (150-160 characters recommended) |
| `tags` | string[] | Categorization tags for filtering and discovery |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `canonical_url` | string | Preferred URL for SEO (typically links to GitHub releases) |
| `cover_image` | string | Hero image URL displayed in listings and social previews |

### Publishing Status

Set `published: false` to draft an article without removing it from the repository. The documentation site filters unpublished articles from public listings.

## Article Structure

### Release Notes Format

Release notes follow a standardized template for consistency across versions:

```markdown
---
# ... frontmatter ...
---

# LibreFang {Version} Released

Opening paragraph describing the release theme and significance.

## What's New

### Category Heading
Description of new features or changes.

### Another Category
Additional changes.

## Bug Fixes

- Individual fix items
- Listed as bullet points

## Install / Upgrade

```bash
# Binary
curl -fsSL https://get.librefang.ai | sh

# Rust SDK
cargo add librefang

# JavaScript SDK
npm install @librefang/sdk

# Python SDK
pip install librefang-sdk
```

## Links

- [Full Changelog](link)
- [GitHub Release](link)
- [GitHub](link)
- [Discord](link)
- [Contributing Guide](link)
```

### Introduction Articles Format

Introduction articles use a more narrative structure with feature highlights:

```markdown
---
# ... frontmatter ...
---

# Article Title

Opening paragraph introducing the topic.

## Section Heading

Content with code examples, tables, or lists.

## Quick Start

```bash
# Terminal commands
```

## Contributing

Contribution guidelines with tables or lists.

## Links

External resources and related documentation.
```

## Common Patterns

### Code Blocks

Install commands are consistently formatted across all release articles:

```bash
# Binary
curl -fsSL https://get.librefang.ai | sh

# Rust SDK
cargo add librefang

# JavaScript SDK
npm install @librefang/sdk

# Python SDK
pip install librefang-sdk
```

### Reference Links

Release notes reference GitHub issues and PRs using `#number` notation (e.g., `#441`, `#1170`). The documentation site transforms these into clickable links to the LibreFang repository.

### Version Numbering

The project switched from semantic versioning (`0.x.x`) to calendar versioning (`YYYY.M.DDHH`) as of version `2026.3.21`:

| Old Format | New Format | Example |
|------------|------------|---------|
| `0.5.6` | `YYYY.M.DDHH` | `2026.3.21` |

### Feature Tables

Feature comparison tables use pipe syntax:

```markdown
| Metric | Others | LibreFang |
|--------|--------|-----------|
| Cold Start | 2.5 ~ 4s | **180ms** |
| Idle Memory | 180 ~ 250MB | **40MB** |
```

## Contributing New Articles

### Writing Release Notes

1. Create a new file named `articles/release-{version}.md`
2. Copy the template from an existing release article
3. Fill in the frontmatter with accurate version information
4. Document changes under appropriate category headings
5. Include the standard install/upgrade section
6. Add links to the full changelog and GitHub release

### Writing Introduction Articles

1. Create a new file named `articles/{slug}.md`
2. Include all required frontmatter fields
3. Structure content with clear headings and examples
4. Add a links section with relevant resources

### Style Guidelines

- **Headings**: Use sentence case for headings (e.g., "What's New" not "What's NEW")
- **Code blocks**: Always specify the language for syntax highlighting
- **Links**: Use descriptive link text, not "click here"
- **Version references**: Use bold formatting for version numbers in prose
- **Breaking changes**: Clearly mark breaking changes in release notes
- **Description length**: Keep descriptions under 160 characters for SEO

## Metadata Conventions

### Tag Taxonomy

Common tags used across articles:

| Tag | Usage |
|-----|-------|
| `rust` | Rust-related content |
| `ai` | AI/ML features |
| `opensource` | Open source announcements |
| `release` | Version release notes |
| `webdev` | Website-related content |

### Cover Images

All articles reference the official LibreFang logo from the repository:

```
https://raw.githubusercontent.com/librefang/librefang/main/public/assets/logo.png
```

## Relationship to Documentation Site

The `articles/` directory integrates with the documentation site's content pipeline:

1. **Frontmatter parsing**: The site reads YAML frontmatter to build article listings
2. **Routing**: File names determine URL slugs (e.g., `release-0.6.0.md` → `/articles/release-0.6.0`)
3. **Tag filtering**: Articles are filterable by tags on listing pages
4. **SEO**: Canonical URLs and descriptions populate meta tags
5. **Social preview**: Cover images appear in Open Graph and Twitter Card metadata

## File Naming

Articles use lowercase kebab-case for filenames:

| Article Type | Naming Convention | Example |
|--------------|-------------------|---------|
| Release notes | `release-{version}.md` | `release-0.6.0.md` |
| Announcements | `{topic}-{topic}.md` | `new-website-launch.md` |
| Introductions | `{topic}.md` | `hello-librefang.md` |

Release note filenames match the GitHub release tag exactly (e.g., `release-2026.3.21.md` for tag `v2026.3.21`).