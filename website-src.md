# Website — src

# Website — src Module

The `web/src` module is the frontend application for LibreFang's marketing site and documentation hub. It's a React application built with TypeScript that serves as both a landing page and a dashboard for downloading, deploying, and learning about the LibreFang Agent Operating System.

## Overview

The module renders a single-page application with multiple sections: a hero with animated terminal, architecture overview, capability showcase, performance benchmarks, installation instructions, downloads, documentation links, FAQ, and community section. It supports 7 languages, dark/light themes, and dynamically fetches live data from external APIs.

## Key Files

| File | Purpose |
|------|---------|
| `App.tsx` | Main component with all page sections and routing |
| `i18n.ts` | Translation strings for all 7 supported languages |
| `useRegistry.ts` | React Query hook for fetching registry data |
| `store.ts` | Zustand store for theme and language state |
| `lib/utils.ts` | Utility functions including `cn` className merger |
| `pages/DeployPage.tsx` | Deployment platform selection page |
| `pages/ChangelogPage.tsx` | Release changelog viewer |

## Application Architecture

```mermaid
graph TD
    App[App.tsx] --> Nav
    App --> Hero
    App --> Architecture
    App --> Hands
    App --> Workflows
    App --> Performance
    App --> Install
    App --> Downloads
    App --> Docs
    App --> FAQ
    App --> GitHubStats
    App --> Community
    App --> Footer
    
    Nav --> store[Zustand Store]
    Hero --> useRegistry[useRegistry]
    Hands --> useRegistry
    Architecture --> useRegistry
    Downloads --> GitHubAPI[stats.librefang.ai]
    
    App --> DeployPage[/deploy route]
    App --> ChangelogPage[/changelog route]
```

## State Management

The application uses **Zustand** for global state:

```typescript
interface AppState {
  theme: 'dark' | 'light'
  lang: string
  toggleTheme: () => void
  switchLang: (code: string) => void
}
```

The `store.ts` also handles:
- **CJK font loading** for languages with non-Latin scripts (Chinese, Japanese, Korean)
- **Language persistence** via localStorage

## Internationalization

The `i18n.ts` file exports translations for 7 languages with full type safety via the `Translation` interface:

| Code | Language | URL Pattern |
|------|----------|-------------|
| `en` | English | `/` |
| `zh` | Simplified Chinese | `/zh` |
| `zh-TW` | Traditional Chinese | `/zh-TW` |
| `ja` | Japanese | `/ja` |
| `ko` | Korean | `/ko` |
| `de` | German | `/de` |
| `es` | Spanish | `/es` |

**Language detection flow:**
1. `getCurrentLang()` checks `window.location.pathname` prefix
2. Falls back to `'en'` if no match
3. Zustand persists the selection in localStorage
4. `popstate` listener re-detects on browser back/forward

## Data Fetching

### Registry Data (`useRegistry`)

Fetches capability data from the registry API:

```typescript
// Returns:
interface RegistryData {
  hands: Hand[]      // 15 built-in capability units
  channels: Channel[] // 44 channel adapters
  handsCount: number
  providersCount: number
}
```

The `useRegistry` hook:
- Uses React Query with 5-minute stale time
- Merges detailed descriptions from the API
- Returns localized descriptions based on current language

### GitHub Statistics

```typescript
// Fetches from stats.librefang.ai/api/github
interface GitHubStatsData {
  stars: number
  forks: number
  issues: number
  prs: number
  downloads: number
  lastUpdate: string
  starHistory: { stars: number }[]
}
```

### Downloads

Fetches from `stats.librefang.ai/api/releases` and categorizes assets:

```typescript
// Desktop app patterns:
x64.dmg              // macOS Intel
aarch64.dmg          // macOS Apple Silicon
x64-setup.exe        // Windows x64
arm64-setup.exe      // Windows ARM
amd64.AppImage       // Linux AppImage
amd64.deb            // Linux DEB
x86_64.rpm           // Linux RPM

// CLI patterns:
x86_64-apple-darwin.tar.gz     // macOS Intel
aarch64-apple-darwin.tar.gz    // macOS ARM
x86_64-unknown-linux-gnu.tar.gz  // Linux glibc
aarch64-unknown-linux-gnu.tar.gz // Linux ARM
```

## Component Structure

### Hero Section

Features a typing animation that cycles through use cases:

```typescript
typing: [
  'run autonomous agents 24/7',
  'replace entire workflows',
  'deploy on any hardware',
  'monitor with 16 security layers',
]
```

The `useTyping` hook implements a typewriter effect with:
- Configurable speed (default 60ms per character)
- 2-second pause at end of each phrase
- Backspace deletion at half speed

### Architecture Section

Expandable 5-layer architecture with icons:

| Layer | Icon | Description |
|-------|------|-------------|
| 0 | Globe | 44 channel adapters |
| 1 | Box | 15 hands (capability units) |
| 2 | Cpu | Kernel: lifecycle, workflows, budget, scheduler |
| 3 | Layers | Runtime: Tokio, WASM, Merkle audit |
| 4 | Radio | Hardware: single binary support |

Each layer expands to show technical details and populates channels/hands from the registry API.

### Hands Carousel

Horizontal scrollable grid showing all 15 capability units:
- **Clip** — Video processing (YouTube to shorts)
- **Lead** — Sales prospecting
- **Collector** — OSINT monitoring
- **Predictor** — Forecasting
- **Researcher** — Deep research
- **Trader** — Market intelligence
- + 9 more: Twitter, Browser, Analytics, DevOps, Creator, LinkedIn, Reddit, Strategist, API Tester

Items marked with the `popular` tag in their `tags` array appear first and get a 🔥 indicator.

### Install Section

Detects the user's OS via `navigator.userAgent` and displays the appropriate install command:

| OS | Command |
|----|---------|
| macOS | `curl -fsSL https://librefang.ai/install \| sh` |
| Windows | `irm https://librefang.ai/install.ps1 \| iex` |
| Linux | `curl -fsSL https://librefang.ai/install \| sh` |

## Animation Utilities

### `FadeIn` Component

Wraps Framer Motion's `motion.div` for scroll-triggered entrance animations:

```typescript
<FadeIn delay={200}>
  {children}
</FadeIn>
```

Uses `whileInView` with `viewport={{ once: true, amount: 0.1 }}` to trigger when 10% of the element is visible.

### Accordion Transitions

FAQ and architecture layers use `AnimatePresence` with height/opacity animations for smooth expand/collapse.

## Theme System

Dark/light mode is stored in Zustand and persisted to localStorage. The toggle updates `document.documentElement.classList` and swaps Lucide icons between Sun and Moon variants.

## External Integrations

| Service | Endpoint | Purpose |
|---------|----------|---------|
| Registry API | `stats.librefang.ai/api/registry` | Hands, channels, providers |
| Releases | `stats.librefang.ai/api/releases` | Download binaries |
| GitHub Stats | `stats.librefang.ai/api/github` | Stars, forks, issues |
| Counter | `counter.librefang.ai/api` | Documentation visits |
| contrib.rocks | `contrib.rocks/image` | Contributor avatars |

## Routing

The `App` component checks `window.location.pathname` on mount:

```typescript
'/deploy'      → renders <DeployPage />
'/changelog/'  → renders <ChangelogPage />
otherwise      → renders full landing page
```

Both sub-pages are self-contained React components with their own layouts and styling.

## Performance Considerations

- React Query caches API responses (5-minute stale time for registry, 1-hour for releases)
- Horizontal scroll uses `touch-pan-x` for mobile swipe gestures
- `IntersectionObserver` for scroll-spy navigation only updates active section
- Images use `loading="lazy"` for below-fold content
- Registry fallback values prevent layout shift while loading