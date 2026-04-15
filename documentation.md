# Documentation

# Documentation

The Documentation module encompasses LibreFang's documentation site and external publishing content. It includes the Next.js application that renders the docs site, its source components and build pipeline, and the standalone articles published to external platforms.

## Sub-modules

| Sub-module | Purpose |
|---|---|
| [src](src.md) | Component source, MDX build pipeline (rehype section extraction, search indexing), and UI layer |
| [docs](docs.md) | Top-level site configuration, MDX plugin chain (remark → rehype → recma), Tailwind setup, and Cloudflare Pages deployment |
| [articles](articles.md) | Static Markdown articles for external syndication (Dev.to, blog) — release notes, introductions, and milestones |

## How They Fit Together

```
articles/                 docs/                  src/
(external content)    (site config & deploy)   (components & pipeline)
       │                      │                        │
       │                      │    MDX pages served     │
       │                      │◄───────────────────────┤
       │                      │                        │
       ▼                      ▼                        │
   Dev.to / Blog      Cloudflare Pages                 │
                            ▲                          │
                            └──── static export ───────┘
```

**Content flow:** MDX files under `src/app/` are processed by the three-stage pipeline configured in [docs](docs.md). The rehype plugin (`src/mdx/rehype.mjs`) extracts heading sections for navigation, while the search plugin (`src/mdx/search.mjs`) builds a full-text search index. Components like `Navigation.tsx`, `Search.tsx`, `SectionProvider.tsx`, and `Heading.tsx` consume this data at runtime to provide scroll-tracked navigation and search.

**Separation of concerns:** [src](src.md) owns the interactive layer and build processing; [docs](docs.md) owns configuration, theming, and deployment. The [articles](articles.md) module is entirely independent — it contains no executable code and feeds only external publishing pipelines.

## Key Cross-Module Workflows

1. **Section-aware navigation** — `rehype.mjs` extracts sections during build → `SectionProvider` tracks visible sections at runtime → `Navigation` and `VisibleSectionHighlight` render the sidebar with scroll tracking
2. **Full-text search** — `search.mjs` indexes MDX content at build time → `Search.tsx` and `useSearchProps` query the generated index client-side
3. **Code rendering** — `Code.tsx` and `CodeGroup` handle tabbed code blocks with layout-shift prevention (`usePreventLayoutShift`)