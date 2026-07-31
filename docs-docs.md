# docs — docs

# docs — LibreFang Documentation Site

## Overview

The `docs` module is the source for [docs.librefang.io](https://docs.librefang.io) — a statically generated Next.js site that serves the LibreFang Agent OS documentation. It is written in MDX, styled with Tailwind CSS v4 and the `@tailwindcss/typography` plugin, and deployed to Cloudflare Pages as a fully static export (no server runtime).

The site lives at the `docs/` root in the monorepo and consumes shared design tokens and UI primitives from sibling packages under `packages/`.

---

## Architecture

```mermaid
flowchart TD
    A[MDX pages<br/>src/app/**/*.mdx] --> B[remark plugins<br/>src/mdx/remark.mjs]
    B --> C[rehype plugins<br/>src/mdx/rehype.mjs]
    C --> D[recma plugins<br/>src/mdx/recma.mjs]
    D --> E[Next.js build<br/>next.config.mjs]
    E --> F[withSearch<br/>src/mdx/search.mjs]
    F --> G[Static HTML export<br/>out/]

    H[mdx-components.tsx] -.-> A
    I[typography.ts] -.-> E
    J[packages/react<br/>@web/ui] -.-> A
    K[packages/shared<br/>@web/shared] -.-> A
```

The build is a pipeline of composable wrappers:

1. **`withSearch(nextConfig)`** — injects search-index generation by scanning all MDX files.
2. **`nextMDX(...)`** — registers the MDX loader with the custom remark/rehype/recma plugin chain.
3. **`next build --webpack`** — produces a static export to `out/`.

The application order in `next.config.mjs` is deliberate: `withMDX(withSearch(nextConfig))`. Search is applied first so the MDX processor can see the search-enhanced config.

---

## MDX Content Pipeline

Content is authored as `page.mdx` files inside `src/app/`. Each route corresponds to a directory:

```
src/app/
├── page.mdx                    # / (Chinese, default)
├── en/
│   └── page.mdx                # /en/ (English)
├── cli-profile-rotation/
│   └── page.mdx
└── ...
```

### Plugin chain

| Stage | File | Responsibility |
|-------|------|----------------|
| **remark** (Markdown → mdast) | `src/mdx/remark.mjs` | GFM tables, heading slugs, table-of-contents extraction, frontmatter parsing |
| **rehype** (mdast → hast) | `src/mdx/rehype.mjs` | HTML transformation, syntax highlighting wiring |
| **recma** (hast → JS AST) | `src/mdx/recma.mjs` | Final JS-level transforms before code generation |

### Component mapping

`mdx-components.tsx` is the entry point Next.js looks for. It merges default MDX components with custom overrides from `@/components/mdx`:

```tsx
export function useMDXComponents(components: MDXComponents) {
  return {
    ...components,
    ...mdxComponents,
  };
}
```

This means every heading, code block, table, and link rendered from MDX passes through the custom component layer, enabling features like anchored headings, Shiki syntax highlighting, and styled callouts.

### Page frontmatter

Each MDX page exports a `sections` array used for sidebar navigation:

```mdx
---
title: Some Page
---

Content here...

export const sections = [];
```

---

## Configuration

### `next.config.mjs`

Key settings:

| Setting | Value | Rationale |
|---------|-------|-----------|
| `output` | `"export"` | Fully static HTML — no Node.js server needed |
| `pageExtensions` | `js, jsx, ts, tsx, mdx` | MDX files are first-class pages |
| `images.unoptimized` | `true` | Required for static export (no image optimization server) |
| `outputFileTracingIncludes` | `src/app/**/*.mdx` | Ensures MDX files are bundled for static generation |
| `serverExternalPackages` | `['shiki']` | Shiki uses native WASM; must be externalized from the bundle |

### `tsconfig.json`

Path aliases connect the docs site to both local source and monorepo packages:

```
@/*              → ./src/*
@/components/*   → ./src/components/*
@/lib/*          → ./src/lib/*
@/app/*          → ./src/app/*

@web/ui          → ../../packages/react/src/index.ts
@web/shared      → ../../packages/shared/src/index.ts
@web/config      → ../../packages/config/src/index.ts
```

This means documentation pages can import shared React components (`@web/ui`), shared utilities (`@web/shared`), and configuration (`@web/config`) directly from the monorepo — keeping the docs UI in sync with the product UI.

### `postcss.config.js`

Minimal — delegates everything to Tailwind CSS v4:

```js
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

---

## Styling & Typography System

`typography.ts` defines the complete prose theme for rendered MDX content. It is consumed by `@tailwindcss/typography` and provides:

- **Light and dark mode** via CSS custom properties (`--tw-prose-*` for light, `--tw-prose-invert-*` for dark). The `invert` modifier swaps all variables in one block, enabling `dark:prose-invert` usage.
- **Brand colors**: Emerald (`emerald-500`/`600`) for links and code accents, zinc scale for body text, headings, and borders.
- **Element-level overrides** for spacing, font sizes, list styles, table layouts, blockquote formatting, and code block presentation (inset box-shadow ring + background).
- **Responsive horizontal rules** that extend beyond the prose container padding at different breakpoints.

The config is a plain object exported as `default export` and imported wherever the typography plugin is registered.

---

## Search

The search system is built at build time, not runtime:

| Component | Package | Role |
|-----------|---------|------|
| `src/mdx/search.mjs` | — | `withSearch` wrapper; scans MDX files, generates a search index at build time |
| `flexsearch` | `^0.8.205` | Client-side full-text search engine over the prebuilt index |
| `@algolia/autocomplete-core` | `1.19.9` | Headless autocomplete UI logic for the search bar |
| `react-highlight-words` | `^0.21.0` | Highlights matching terms in search results |

Because the site is statically exported, the entire search index is serialized into static JSON and loaded client-side.

---

## Multi-Language

The site supports two locales with a directory-based routing strategy:

| Route | Language | Source |
|-------|----------|--------|
| `/` | Chinese (default) | Authored directly in `src/app/` |
| `/en/` | English | Synced from the LibreFang repository |

English content lives under `src/app/en/` and mirrors the Chinese route structure.

---

## Content Authoring

### Adding a new documentation page

1. Create a directory under `src/app/`, e.g. `src/app/new-feature/`.
2. Add `page.mdx` with frontmatter and content.
3. Export `sections` at the end of the file for sidebar navigation.

```mdx
---
title: New Feature
description: What this feature does and how to use it.
---

# New Feature

Content written in MDX — full GFM and JSX support.

export const sections = [
  { title: 'Installation', id: 'installation' },
  { title: 'Configuration', id: 'configuration' },
];
```

### Markdown files at the docs root

Standalone reference documents (not rendered as site pages) live at the `docs/` root:

- **`docs/releases.md`** — Release versioning policy (CALVER format, pre-release tag conventions, CI dist-tag behavior). Referenced by the `xtask release` tooling and PR reviewers.
- **`docs/cli-profile-rotation.md`** — User-facing guide for Claude Code CLI account rotation via `TokenRotationDriver`.

---

## Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `next` | `16.2.12` | Static site framework |
| `react` / `react-dom` | `19.2.8` | UI runtime |
| `@mdx-js/loader` + `@mdx-js/react` | `3.1.1` | MDX compilation and component provider |
| `remark` + `remark-gfm` + `remark-mdx` | latest | Markdown processing plugins |
| `shiki` | `^4.3.1` | Server-side syntax highlighting (WASM-based) |
| `prism-react-renderer` | `^2.4.1` | Client-side code rendering |
| `flexsearch` | `^0.8.205` | Build-time indexed search |
| `@algolia/autocomplete-core` | `1.19.9` | Search bar autocomplete logic |
| `@giscus/react` | `^3.1.0` | GitHub Discussions comments |
| `zustand` | `5.0.14` | Lightweight client state (theme, navigation) |
| `motion` | `12.42.2` | Animations |
| `lucide-react` | `^1.27.0` | Icon set |
| `next-themes` | `^0.4.6` | Dark/light mode switching |
| `tailwindcss` | `4.3.3` | Utility-first CSS (v4 with PostCSS plugin) |

---

## Development Workflow

### Prerequisites

- Node.js ≥ 18
- pnpm ≥ 9 (project pins `pnpm@10.11.1`)

### Commands

```bash
pnpm install          # Install dependencies
pnpm dev              # Dev server on port 3001
pnpm build            # Static export to out/
pnpm start            # Serve built output on port 3001
pnpm lint             # Biome check
pnpm lint:fix         # Biome check --write
pnpm typecheck        # tsc --noEmit
pnpm format           # Biome format src --write
```

The dev server runs on port **3001** (not the default 3000) to avoid conflicts with other services in the monorepo.

### Build note

The build script uses `next build --webpack` (not the default Turbopack builder) because the MDX plugin chain and `simple-functional-loader` dependencies are webpack-specific.

---

## Connection to the Monorepo

The docs site is not isolated — it imports from three sibling packages via `tsconfig.json` path aliases:

- **`@web/ui`** (`packages/react/`) — Shared React component library used across LibreFang's web surfaces. Documentation pages can demo real product components.
- **`@web/shared`** (`packages/shared/`) — Shared utilities, types, and constants.
- **`@web/config`** (`packages/config/`) — Shared configuration (site metadata, feature flags, constants).

This means changes to shared packages propagate to documentation automatically — there is no separate copy of components or config in the docs module.