# Internationalization

# Internationalization (i18n)

The `i18n/` directory manages all multi-language content for LibreFang. It contains two distinct systems that serve different purposes:

1. **Document translations** — Full markdown translations of project documentation (README, getting-started guide, skill development guide) for human readers.
2. **Runtime error catalogs** — TOML key-value files that catalog translatable error messages, used by the application at runtime via Fluent (`.ftl`) locale files.

There is no executable code in this module. It is static content consumed by GitHub rendering, the build system, and the runtime locale loader.

## Directory Layout

```
i18n/
├── README.md                 # Contributor guide for translations
├── README.de.md              # German README translation
├── README.es.md              # Spanish README translation
├── README.ja.md              # Japanese README translation
├── README.ko.md              # Korean README translation
├── README.pl.md              # Polish README translation
├── README.zh.md              # Chinese (Simplified) README translation
├── en.toml                   # English error message catalog
├── zh.toml                   # Chinese error message catalog
├── getting-started.fr.md     # French getting-started guide
└── skill-development.zh.md   # Chinese skill development guide

packages/cli-npm/             # npm binary distribution wrapper
├── README.md
├── package.json
└── bin/librefang.js
```

## Document Translations

### README Translations

Each `README.<lang>.md` is a complete translation of the root `README.md`. They follow identical section structure and preserve all HTML markup, badge URLs, image paths, and relative links.

**Current languages:**

| File | Language | Code |
|------|----------|------|
| `README.de.md` | Deutsch (German) | `de` |
| `README.es.md` | Español (Spanish) | `es` |
| `README.ja.md` | 日本語 (Japanese) | `ja` |
| `README.ko.md` | 한국어 (Korean) | `ko` |
| `README.pl.md` | Polski (Polish) | `pl` |
| `README.zh.md` | 中文 (Chinese Simplified) | `zh` |

All translated READMEs contain a multi-language navigation bar near the top:

```html
<a href="../README.md">English</a> | <a href="README.zh.md">中文</a> | ...
```

This bar must be updated in **every** translation file (including the root `README.md`) whenever a new language is added.

### Additional Document Translations

Beyond the README, some longer-form docs have been translated:

- `getting-started.fr.md` — Full French translation of the getting-started guide
- `skill-development.zh.md` — Full Chinese translation of the skill development guide

These follow the same conventions: preserve all relative links pointing to English originals, keep formatting identical, and leave brand names and technical terms untranslated.

### Link Conventions

All document translations use `../` relative paths to reference files outside `i18n/`:

- `../docs/CONTRIBUTING.md` — Contribution guide (English original)
- `../docs/GOVERNANCE.md` — Governance docs (English original)
- `../public/assets/logo.png` — Shared assets

Docs files are **never duplicated** into `i18n/`. Translation files point to the single English original. This avoids drift and maintenance burden.

## Runtime Error Message Catalogs

### Purpose

The TOML files (`en.toml`, `zh.toml`) catalog all translatable error messages used by the LibreFang API and runtime. They are organized by subsystem into TOML tables:

- `[agent]` — Agent lifecycle errors (spawn, not-found, invalid-id)
- `[message]` — Message handling errors (too-large, delivery-failed)
- `[template]` — Template parsing and lookup errors
- `[manifest]` — Manifest validation and signature errors
- `[auth]` — Authentication errors
- `[session]` — Session management errors
- `[workflow]` — Workflow execution errors
- `[trigger]` — Event trigger errors
- `[budget]` — Budget management errors
- `[config]` — Configuration read/write errors
- `[profile]` — Profile lookup errors
- `[cron]` — Scheduled job errors
- `[general]` — Catch-all errors (not-found, internal, bad-request, rate-limited)

### Relationship to Fluent Files

The TOML files are **reference catalogs**, not the files loaded at runtime. The actual translation lookup uses Fluent (`.ftl`) files located at:

```
crates/librefang-types/locales/
```

The TOML catalogs exist for:
- Human-readable reference of all error keys and their English source text
- External tooling that may need to parse the message catalog
- Future TOML-based i18n backends

### Message Format

Messages use ICU-style interpolation with curly braces:

```toml
delivery-failed  = "Message delivery failed: {reason}"
not-found        = "Template '{name}' not found"
parse-failed     = "Failed to parse template: {error}"
bad-request      = "Bad request: {reason}"
```

Placeholders like `{reason}`, `{name}`, `{error}` are filled at runtime by the calling code.

### Adding a New Error Language

1. Copy `en.toml` to `<lang-code>.toml` (e.g., `ja.toml`).
2. Translate all message values. Keep all interpolation placeholders (`{reason}`, `{name}`, etc.) unchanged.
3. Create corresponding Fluent files under `crates/librefang-types/locales/<lang-code>/`.
4. Register the locale in the runtime's locale loader.

## NPM CLI Package

The `packages/cli-npm/` directory wraps the pre-built LibreFang CLI binary for npm distribution. It is not related to translations but lives in the i18n/packages tree.

### Platform Binaries

The package uses `optionalDependencies` to install the correct platform-specific binary:

| Package | Platform |
|---------|----------|
| `@librefang/cli-darwin-arm64` | macOS Apple Silicon |
| `@librefang/cli-darwin-x64` | macOS Intel |
| `@librefang/cli-linux-x64` | Linux x64 (glibc) |
| `@librefang/cli-linux-arm64` | Linux arm64 (glibc) |
| `@librefang/cli-linux-x64-musl` | Linux x64 (musl/Alpine) |
| `@librefang/cli-linux-arm64-musl` | Linux arm64 (musl/Alpine) |
| `@librefang/cli-win32-x64` | Windows x64 |
| `@librefang/cli-win32-arm64` | Windows arm64 |

npm automatically selects the correct optional dependency at install time based on the user's platform.

## Contributing Translations

### Adding a New README Language

1. Copy the root `README.md` into `i18n/` as `README.<lang>.md`, where `<lang>` is an [ISO 639-1](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) code.
2. Translate all prose. Leave unchanged: HTML tags, badge URLs, image paths, code blocks, brand names (LibreFang, Hands, Hand, FangHub), and well-known technical terms (Rust, crate, CLI, API, WebAssembly).
3. Add your language link to the navigation bar in **every** existing translation file and the root `README.md`.
4. Verify all relative links resolve correctly from the `i18n/` directory.
5. Open a PR.

### Updating Translations After English Changes

When the root `README.md` changes:

1. Review the diff to identify new or modified sections.
2. Add corresponding translated sections to each language file in the same position.
3. If you cannot translate to all languages, update what you can and open issues for the rest.

### Style Guidelines

- Match the tone and length of the English original — avoid adding commentary.
- Use idiomatic phrasing over literal word-for-word translation.
- Use consistent terminology throughout (e.g., always translate "agent" the same way).
- Keep well-known technical terms in English.
- Brand names stay untranslated.