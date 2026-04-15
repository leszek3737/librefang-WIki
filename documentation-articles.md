# Documentation — articles

# Documentation — Articles

Markdown articles published to external platforms (Dev.to, canonical blog) announcing releases, features, and project milestones.

## Purpose

This module is the content source for all externally published articles about LibreFang. Articles are **not executed at runtime** — they are static Markdown files consumed by publishing pipelines and syndicated to platforms like Dev.to.

The module serves three distinct functions:

1. **Release announcements** — Version-specific changelogs formatted as developer-facing articles
2. **Project introductions** — Evergreen content explaining what LibreFang is and how to get started
3. **Milestone posts** — Announcements for major events (website launches, architecture overhauls)

## File Inventory

| File | Type | Version / Scope |
|------|------|-----------------|
| `hello-librefang.md` | Introduction | Evergreen — project overview, quick start, contributing guide |
| `new-website-launch.md` | Milestone | Website redesign announcement with architecture details |
| `release-0.5.6.md` | Release | v0.5.6 — CLI i18n, Telegram reply-to, SDK publishing |
| `release-0.5.7.md` | Release | v0.5.7 — Multi-token fallback, event webhooks, multi-agent foundations |
| `release-0.6.0.md` | Release | v0.6.0 — Memory/context engine, NVIDIA NIM, Vertex AI, cron workflows |
| `release-0.6.1.md` | Release | v0.6.1 — CLI name resolution, API response consistency, dashboard i18n |
| `release-0.6.2.md` | Release | v0.6.2 — Provider config deduplication, CI build fix (17 compilation errors) |
| `release-0.6.3.md` | Release | v0.6.3 — Vault auto-init, webhook decryption, POSIX installer |
| `release-0.6.4.md` | Release | v0.6.4 — Qwen Code CLI provider, image pipeline, web deployment overhaul |
| `release-0.6.5.md` | Release | v0.6.5 — Security hardening, multi-provider cost tracking, rustls TLS |
| `release-0.6.6.md` | Release | v0.6.6 — Docker `build-push-action@v7` compatibility fix |
| `release-0.6.7.md` | Release | v0.6.7 — HAND manifest routing, raw JSON context hooks, GitHub Discussions |
| `release-0.6.8.md` | Release | v0.6.8 — CLI on npm/PyPI, external DM routing |
| `release-0.7.0.md` | Release | v0.7.0 — LLM intent routing, OpenFang migration, CORS/rate-limit/audit config |
| `release-2026.3.21.md` | Release | CalVer debut — Azure OpenAI, DeepInfra, dashboard redesign, 30+ fixes |
| `release-2026.3.22.md` | Release | CalVer consolidation — 50+ improvements, pipeline runner agents, plugin scoping |

## Front Matter Schema

Every article uses a consistent YAML front matter block:

```yaml
---
title: "LibreFang X.Y.Z Released"          # Required — article headline
published: true                             # Required — controls visibility
description: "One-line summary for SEO"     # Required — appears in link previews
tags: rust, ai, opensource, release         # Required — comma-separated platform tags
canonical_url: https://github.com/...       # Required — original source URL
cover_image: https://raw.githubusercontent.com/librefang/librefang/main/public/assets/logo.png
---
```

### Tag Conventions

| Article Type | Tags |
|-------------|------|
| Release notes | `rust, ai, opensource, release` |
| Introduction | `rust, ai, opensource, agents` |
| Web/milestone | `opensource, rust, ai, webdev` |

## Article Structure Patterns

### Release Notes

All release notes follow a predictable structure:

```markdown
# LibreFang X.Y.Z Released

[2-3 sentence summary of the release theme]

## [Category emojis + headings organized by impact]

[Grouped changes with issue references in parentheses]

## Install / Upgrade

[Standard install block — binary, Rust SDK, JS SDK, Python SDK]

## Links

[Full Changelog, GitHub Release, GitHub, Discord, Contributing Guide]
```

Release notes reference GitHub issues and PRs by number (e.g., `#441`, `#1088`) and credit contributors by username (e.g., `@SenZhangAI`).

### Version Naming Transition

The project switched from SemVer to CalVer starting with `v2026.3.21`:

- **Pre-CalVer**: `release-0.5.6.md`, `release-0.6.0.md`, … `release-0.7.0.md`
- **CalVer**: `release-2026.3.21.md`, `release-2026.3.22.md`

CalVer format is `YYYY.M.DDHH` (e.g., `2026.3.2201`).

The `canonical_url` field maps to the corresponding GitHub release tag:
- SemVer: `.../releases/tag/v0.6.4-20260320`
- CalVer: `.../releases/tag/v2026.3.2201`

## Content Conventions

### Install Block

Every release article includes the same install block:

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

This block must be kept in sync with the actual distribution channels. The install URL `https://get.librefang.ai` and package names (`librefang`, `@librefang/sdk`, `librefang-sdk`) are referenced in the CI/CD release pipeline.

### Links Block

Every release article ends with the same link set:

```markdown
- [Full Changelog](https://github.com/librefang/librefang/blob/main/CHANGELOG.md)
- [GitHub Release](https://github.com/librefang/librefang/releases/tag/v...)
- [GitHub](https://github.com/librefang/librefang)
- [Discord](https://discord.gg/DzTYqAZZmc)
- [Contributing Guide](https://github.com/librefang/librefang/blob/main/docs/CONTRIBUTING.md)
```

The only variable is the GitHub Release URL, which changes per version.

## Publishing Pipeline

Based on release notes referencing "automated release announcements" (#0.5.6), articles are generated or published through a CI workflow:

1. **Trigger** — A GitHub release is published
2. **Generate** — Release notes are converted to article Markdown
3. **Publish** — Articles are posted to Dev.to via API

The workflow was fixed in v0.5.6 (YAML syntax error in Bluesky notification) and v0.6.1 (Dev.to markdown fence wrapper removal in #1167).

## Contributing a New Article

### For a Release

1. Create `articles/release-X.Y.Z.md` (or `release-YYYY.M.DD.md` for CalVer)
2. Copy the front matter from an existing release article
3. Update `title`, `description`, `canonical_url`, and `cover_image`
4. Write the body following the release note structure
5. Reference GitHub issues/PRs by number
6. Include the standard Install and Links blocks
7. Ensure the `canonical_url` matches the actual GitHub release tag

### For a Non-Release Article

1. Choose an appropriate filename (e.g., `new-website-launch.md`, `hello-librefang.md`)
2. Use tags relevant to the content topic
3. Write for an external developer audience — not internal documentation
4. Include practical commands, code blocks, or comparison tables where applicable

### Validation Checklist

- [ ] Front matter is complete (`title`, `published`, `description`, `tags`, `canonical_url`, `cover_image`)
- [ ] No raw HTML in Markdown
- [ ] Code blocks specify their language for syntax highlighting
- [ ] Issue references use `#NNN` format
- [ ] Install block matches current distribution channels
- [ ] Links block points to correct release tag