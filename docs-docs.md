# docs — docs

# docs — Witryna dokumentacji LibreFang

## Przegląd

Moduł `docs` jest źródłem witryny [docs.librefang.io](https://docs.librefang.io) — statycznie generowanej witryny Next.js, która udostępnia dokumentację LibreFang Agent OS. Jest napisana w MDX, stylizowana za pomocą Tailwind CSS v4 oraz wtyczki `@tailwindcss/typography`, a wdrażana na Cloudflare Pages jako pełny eksport statyczny (bez środowiska serwerowego).

Witryna znajduje się w katalogu głównym `docs/` w monorepo i korzysta ze współdzielonych tokenów projektowych oraz prymitywów UI z pakietów równorzędnych w katalogu `packages/`.

---

## Architektura

```mermaid
flowchart TD
    A[Strony MDX<br/>src/app/**/*.mdx] --> B[wtyczki remark<br/>src/mdx/remark.mjs]
    B --> C[wtyczki rehype<br/>src/mdx/rehype.mjs]
    C --> D[wtyczki recma<br/>src/mdx/recma.mjs]
    D --> E[Budowa Next.js<br/>next.config.mjs]
    E --> F[withSearch<br/>src/mdx/search.mjs]
    F --> G[Eksport statyczny HTML<br/>out/]

    H[mdx-components.tsx] -.-> A
    I[typography.ts] -.-> E
    J[packages/react<br/>@web/ui] -.-> A
    K[packages/shared<br/>@web/shared] -.-> A
```

Budowa to potok kompozycyjnych otoków:

1. **`withSearch(nextConfig)`** — wstrzykuje generowanie indeksu wyszukiwania poprzez skanowanie wszystkich plików MDX.
2. **`nextMDX(...)`** — rejestruje loader MDX z niestandardowym łańcuchem wtyczek remark/rehype/recma.
3. **`next build --webpack`** — generuje eksport statyczny do katalogu `out/`.

Kolejność zastosowań w `next.config.mjs` jest celowa: `withMDX(withSearch(nextConfig))`. Wyszukiwanie jest aplikowane jako pierwsze, aby procesor MDX mógł zobaczyć konfigurację rozszerzoną o wyszukiwanie.

---

## Potok treści MDX

Treść jest tworzona jako pliki `page.mdx` w katalogu `src/app/`. Każda ścieżka odpowiada katalogowi:

```
src/app/
├── page.mdx                    # / (chiński, domyślny)
├── en/
│   └── page.mdx                # /en/ (angielski)
├── cli-profile-rotation/
│   └── page.mdx
└── ...
```

### Łańcuch wtyczek

| Etap | Plik | Odpowiedzialność |
|-------|------|----------------|
| **remark** (Markdown → mdast) | `src/mdx/remark.mjs` | Tabele GFM, kotwice nagłówków, ekstrakcja spisu treści, parsowanie frontmatter |
| **rehype** (mdast → hast) | `src/mdx/rehype.mjs` | Transformacja HTML, podłączenie podświetlania składni |
| **recma** (hast → AST JS) | `src/mdx/recma.mjs` | Końcowe transformacje na poziomie JS przed generowaniem kodu |

### Mapowanie komponentów

`mdx-components.tsx` to punkt wejścia, którego szuka Next.js. Łączy domyślne komponenty MDX z niestandardowymi nadpisami z `@/components/mdx`:

```tsx
export function useMDXComponents(components: MDXComponents) {
  return {
    ...components,
    ...mdxComponents,
  };
}
```

Oznacza to, że każdy nagłówek, blok kodu, tabela i link renderowany z MDX przechodzi przez warstwę niestandardowych komponentów, umożliwiając takie funkcje jak zakotwiczone nagłówki, podświetlanie składni Shiki i stylizowane callouty.

### Frontmatter strony

Każda strona MDX eksportuje tablicę `sections` używaną do nawigacji w panelu bocznym:

```mdx
---
title: Niektóra Strona
---

Treść tutaj...

export const sections = [];
```

---

## Konfiguracja

### `next.config.mjs`

Kluczowe ustawienia:

| Ustawienie | Wartość | Uzasadnienie |
|---------|-------|-----------|
| `output` | `"export"` | Całkowicie statyczny HTML — nie wymaga serwera Node.js |
| `pageExtensions` | `js, jsx, ts, tsx, mdx` | Pliki MDX są pełnoprawnymi stronami |
| `images.unoptimized` | `true` | Wymagane dla eksportu statycznego (brak serwera optymalizacji obrazów) |
| `outputFileTracingIncludes` | `src/app/**/*.mdx` | Zapewnia dołączenie plików MDX do generowania statycznego |
| `serverExternalPackages` | `['shiki']` | Shiki używa natywnego WASM; musi być wyłączone z pakietu |

### `tsconfig.json`

Aliasy ścieżek łączą witrynę dokumentacji zarówno z lokalnym źródłem, jak i pakietami monorepo:

```
@/*              → ./src/*
@/components/*   → ./src/components/*
@/lib/*          → ./src/lib/*
@/app/*          → ./src/app/*

@web/ui          → ../../packages/react/src/index.ts
@web/shared      → ../../packages/shared/src/index.ts
@web/config      → ../../packages/config/src/index.ts
```

Oznacza to, że strony dokumentacji mogą importować współdzielone komponenty React (`@web/ui`), współdzielone narzędzia (`@web/shared`) oraz konfigurację (`@web/config`) bezpośrednio z monorepo — utrzymując spójność interfejsu dokumentacji z interfejsem produktu.

### `postcss.config.js`

Minimalny — deleguje wszystko do Tailwind CSS v4:

```js
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

---

## System stylów i typografii

`typography.ts` definiuje kompletną tematykę prose dla renderowanej treści MDX. Jest konsumowany przez `@tailwindcss/typography` i dostarcza:

- **Tryb jasny i ciemny** za pomocą właściwości niestandardowych CSS (`--tw-prose-*` dla trybu jasnego, `--tw-prose-invert-*` dla trybu ciemnego). Modyfikator `invert` zamienia wszystkie zmienne w jednym bloku, umożliwiając użycie `dark:prose-invert`.
- **Kolory marki**: Szmaragdowy (`emerald-500`/`600`) dla linków i akcentów kodu, skala zinc dla tekstu głównego, nagłówków i obramowań.
- **Nadpisania na poziomie elementów** dla odstępów, rozmiarów fontów, stylów list, układów tabel, formatowania bloków cytatu i prezentacji bloków kodu (wewnętrzny box-shadow ring + tło).
- **Responsywne poziome linie** rozciągające się poza padding kontenera prose na różnych breakpointach.

Konfiguracja to zwykły obiekt eksportowany jako `default export` i importowany tam, gdzie zarejestrowana jest wtyczka typografii.

---

## Wyszukiwanie

System wyszukiwania jest budowany w czasie budowy, a nie w czasie działania:

| Komponent | Pakiet | Rola |
|-----------|---------|------|
| `src/mdx/search.mjs` | — | Otok `withSearch`; skanuje pliki MDX, generuje indeks wyszukiwania w czasie budowy |
| `flexsearch` | `^0.8.205` | Kliencki silnik wyszukiwania pełnotekstowego nad wstępnie zbudowanym indeksem |
| `@algolia/autocomplete-core` | `1.19.9` | Bezgłówna logika UI autouzupełniania dla paska wyszukiwania |
| `react-highlight-words` | `^0.21.0` | Podświetla pasujące terminy w wynikach wyszukiwania |

Ponieważ witryna jest eksportowana statycznie, cały indeks wyszukiwania jest serializowany do statycznego JSON i ładowany po stronie klienta.

---

## Wielojęzyczność

Witryna obsługuje dwie lokalizacje z katalogową strategią routingu:

| Ścieżka | Język | Źródło |
|-------|----------|--------|
| `/` | Chiński (domyślny) | Tworzony bezpośrednio w `src/app/` |
| `/en/` | Angielski | Synchronizowany z repozytorium LibreFang |

Treść angielska znajduje się w `src/app/en/` i odzwierciedla chińską strukturę ścieżek.

---

## Tworzenie treści

### Dodawanie nowej strony dokumentacji

1. Utwórz katalog w `src/app/`, np. `src/app/new-feature/`.
2. Dodaj `page.mdx` z frontmatter i treścią.
3. Wyeksportuj `sections` na końcu pliku dla nawigacji w panelu bocznym.

```mdx
---
title: Nowa Funkcjonalność
description: Co ta funkcja robi i jak jej używać.
---

# Nowa Funkcjonalność

Treść napisana w MDX — pełna obsługa GFM i JSX.

export const sections = [
  { title: 'Instalacja', id: 'installation' },
  { title: 'Konfiguracja', id: 'configuration' },
];
```

### Pliki Markdown w katalogu głównym docs

Samodzielne dokumenty referencyjne (nierenderowane jako strony witryny) znajdują się w katalogu głównym `docs/`:

- **`docs/releases.md`** — Polityka wersjonowania wydań (format CALVER, konwencje tagów pre-release, zachowanie CI dist-tag). Wspierany przez narzędzia `xtask release` i recenzentów PR.
- **`docs/cli-profile-rotation.md`** — Przewodnik użytkownika dotyczący rotacji kont CLI Claude Code za pomocą `TokenRotationDriver`.

---

## Kluczowe zależności

| Zależność | Wersja | Przeznaczenie |
|------------|---------|---------|
| `next` | `16.2.12` | Framework witryny statycznej |
| `react` / `react-dom` | `19.2.8` | Środowisko uruchomieniowe UI |
| `@mdx-js/loader` + `@mdx-js/react` | `3.1.1` | Kompilacja MDX i dostawca komponentów |
| `remark` + `remark-gfm` + `remark-mdx` | latest | Wtyczki przetwarzania Markdown |
| `shiki` | `^4.3.1` | Podświetlanie składni po stronie serwera (oparte na WASM) |
| `prism-react-renderer` | `^2.4.1` | Renderowanie kodu po stronie klienta |
| `flexsearch` | `^0.8.205` | Wyszukiwanie z indeksem budowanym w czasie budowy |
| `@algolia/autocomplete-core` | `1.19.9` | Logika autouzupełniania paska wyszukiwania |
| `@giscus/react` | `^3.1.0` | Komentarze z GitHub Discussions |
| `zustand` | `5.0.14` | Lekki stan klienta (motyw, nawigacja) |
| `motion` | `12.42.2` | Animacje |
| `lucide-react` | `^1.27.0` | Zestaw ikon |
| `next-themes` | `^0.4.6` | Przełączanie trybu ciemnego/jasnego |
| `tailwindcss` | `4.3.3` | CSS utility-first (v4 z wtyczką PostCSS) |

---

## Przepływ pracy deweloperskiej

### Wymagania wstępne

- Node.js ≥ 18
- pnpm ≥ 9 (projekt fixuje `pnpm@10.11.1`)

### Polecenia

```bash
pnpm install          # Instalacja zależności
pnpm dev              # Serwer deweloperski na porcie 3001
pnpm build            # Eksport statyczny do out/
pnpm start            # Serwowanie zbudowanego wyniku na porcie 3001
pnpm lint             # Sprawdzenie Biome
pnpm lint:fix         # Sprawdzenie Biome --write
pnpm typecheck        # tsc --noEmit
pnpm format           # Biome format src --write
```

Serwer deweloperski działa na porcie **3001** (a nie domyślnym 3000), aby uniknąć konfliktów z innymi usługami w monorepo.

### Uwaga dotycząca budowy

Skrypt budowy używa `next build --webpack` (a nie domyślnego buildera Turbopack), ponieważ łańcuch wtyczek MDX i zależności `simple-functional-loader` są specyficzne dla webpacka.

---

## Powiązanie z monorepo

Witryna dokumentacji nie jest odizolowana — importuje z trzech pakietów równorzędnych poprzez aliasy ścieżek w `tsconfig.json`:

- **`@web/ui`** (`packages/react/`) — Współdzielona biblioteka komponentów React używana w interfejsach webowych LibreFang. Strony dokumentacji mogą demonstrować rzeczywiste komponenty produktu.
- **`@web/shared`** (`packages/shared/`) — Współdzielone narzędzia, typy i stałe.
- **`@web/config`** (`packages/config/`) — Współdzielona konfiguracja (metadane witryny, flagi funkcji, stałe).

Oznacza to, że zmiany w pakietach współdzielonych propagują się do dokumentacji automatycznie — nie ma osobnej kopii komponentów ani konfiguracji w module docs.
