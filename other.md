# Other

# Supporting Infrastructure

This module contains auxiliary pieces that support the LibreFang documentation site and web presence: URL redirect rules for backward compatibility, the site header component with internationalization, and two Cloudflare Workers for runtime statistics.

---

## URL Redirect Map

**File:** `docs/public/_redirects`

### Purpose

During the 2026-04 docs restructure, all pages moved from flat URLs (e.g., `/agents`) to hierarchical group-based URLs (e.g., `/agent/templates`). This Netlify-format `_redirects` file issues `301` (permanent) redirects so that every existing external link, bookmark, and search index entry continues to resolve.

### URL Grouping Scheme

The restructure organizes pages into five top-level groups, plus a `/zh` locale mirror:

| Group | New Prefix | Representative Redirect |
|---|---|---|
| Getting Started | `/getting-started` | `/librefang` → `/getting-started` |
| Configuration | `/configuration` | `/providers` → `/configuration/providers` |
| Architecture | `/architecture` | `/security` → `/architecture/security` |
| Agent | `/agent` | `/agents` → `/agent/templates` |
| Integrations | `/integrations` | `/api` → `/integrations/api` |
| Operations | `/operations` | `/faq` → `/operations/faq` |

### Wildcard Rules

Several sections use `:splat` wildcards to catch nested paths. These must be listed **after** their exact-match counterpart because Netlify evaluates rules top-to-bottom and stops at the first match:

```
/providers            /configuration/providers         301
/providers/*          /configuration/providers/:splat  301
```

### Locale Handling

The `/zh` (Chinese) section duplicates every rule with the `/zh` prefix preserved on both sides:

```
/zh/providers         /zh/configuration/providers         301
/zh/providers/*       /zh/configuration/providers/:splat  301
```

When adding a new page redirect, update both the English and `/zh` blocks to keep them in sync.

---

## Header Component

**File:** `docs/src/components/Header.tsx`

### Purpose

The site-wide header bar. It provides navigation links, a search trigger, a language toggle between English and Chinese, and a dark/light theme toggle. On mobile it collapses into a hamburger menu.

### Key Behaviors

**Scroll-aware transparency.** The header uses `motion/react`'s `useScroll` and `useTransform` to interpolate background opacity as the user scrolls:

- **Light mode:** opacity transitions from 50% to 90% over the first 72px of scroll.
- **Dark mode:** opacity transitions from 20% to 80%.

This creates a semi-transparent glass effect at the top of the page that solidifies as content scrolls underneath.

**Language switching.** The `LangSwitch` component reads the current pathname via `usePathname()`. If the path starts with `/zh`, it strips that prefix to link to the English version; otherwise it prepends `/zh`. The toggle button displays `中文` when on the English site and `EN` when on the Chinese site.

**Mobile layout.** Below the `lg` breakpoint, the header renders `MobileNavigation` (hamburger menu) and a `Logo` link instead of the desktop nav bar. `MobileSearch` replaces the desktop `Search` component.

### Component Props

`Header` is a `forwardRef` wrapping `motion.div`. It accepts all standard `motion.div` props and forwards the ref, enabling parent layouts to measure or scroll-link the header.

### External Dependencies

| Import | Role |
|---|---|
| `@/components/Button` | Shared button primitive |
| `@/components/Logo` | Brand logo SVG |
| `@/components/MobileNavigation` | Hamburger nav drawer + open-state store |
| `@/components/Search`, `MobileSearch` | Desktop and mobile search dialogs |
| `@/components/ThemeToggle` | Dark/light mode switch |
| `@/lib/utils#withPrefix` | Prepends the configured base path to internal links |

---

## Cloudflare Workers

Two workers run on Cloudflare's edge network to provide lightweight runtime services.

### GitHub Stats Worker

**Directory:** `web/workers/github-stats-worker/`

| Field | Value |
|---|---|
| Worker name | `librefang-github-stats` |
| Entry point | `index.js` |
| KV binding | `KV` (namespace `5d12c668…`) |
| Cron trigger | `0 0 * * *` (daily at UTC midnight) |

This worker fetches repository statistics from the GitHub API on a daily cron schedule and caches them in a Cloudflare KV namespace. The documentation site or other consumers read from KV to display star counts, contributor counts, etc., without hitting the GitHub API on every page load.

### Visit Counter Worker

**Directory:** `web/workers/visit-counter-worker/`

| Field | Value |
|---|---|
| Worker name | `librefang-visit-counter` |
| Entry point | `index.js` |
| KV binding | `VISIT_COUNTER` (namespace `3ddec4f5…`) |

An edge function that increments and reads a page-view counter stored in KV. No cron trigger is configured — it runs on every incoming request to the documentation site.

### Deployment

Both workers use the standard `wrangler.toml` configuration format. To deploy:

```bash
cd web/workers/<worker-name>
npx wrangler deploy
```

Ensure the `CLOUDFLARE_API_TOKEN` environment variable is set with permissions for Workers and KV.

---

## Adding a New Redirect

When a documentation page moves or is renamed:

1. Add an exact-match rule in the appropriate group section (English block).
2. If the page has sub-paths, add a wildcard rule **after** the exact-match rule.
3. Duplicate both rules in the `/zh` block, preserving the `/zh` prefix.
4. Verify the redirect locally with `netlify dev` or by deploying to a preview environment.