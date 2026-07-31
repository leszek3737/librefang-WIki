# docs — supermoce

# Moduł `docs/superpowers`

Dokumentacja planowania i projektowania dla istotnych prac nad funkcjonalnościami. Ten moduł zawiera dwa rodzaje artefaktów — **plany** i **specyfikacje** — które wspólnie określają, jak funkcjonalności są projektowane, zakreskowane i implementowane przed napisaniem kodu.

---

## Struktura modułu

```
docs/superpowers/
├── plans/      # Trasy wdrożenia krok po kroku
└── specs/      # Decyzje architektoniczne i projektowe
```

Każdy dokument jest oznaczony datą (`YYYY-MM-DD-<slug>.md`) i powiązany z gałęzią funkcjonalności. Data łączy plan z odpowiadającą mu specyfikacją i odwrotnie.

---

## Rodzaje dokumentów

### Specyfikacje (`specs/`)

Specyfikacja jest architektonicznym źródłem prawdy dla funkcjonalności. Przechwytuje decyzje przed rozpoczęciem implementacji. Typowa specyfikacja zawiera:

- **Podsumowanie** — jednoparagrafowy opis funkcjonalności
- **Cele / Poza zakresem** — granice zakreskowania
- **Tabela decyzji** — kluczowe wybory projektowe z uzasadnieniem (np. gdzie znajduje się logika, blokujące vs nieblokujące, odkrywanie vs konfiguracja)
- **Architektura** — relacje zależności, nowe/modyfikowane pliki
- **Struktury danych** — definicje cech (traits) w Ruście, typy konfiguracji, enumy
- **Diagramy przepływu** — początkowe połączenie, przepływ uwierzytelniania, odświeżanie tokenów (często jako ASCII art)
- **Kontrakty API** — sygnatury punktów końcowych, kształty żądań/odpowiedzi
- **Strategia testowania** — jednostkowe, integracyjne, ręczne
- **Otwarte pytania** — nierozwiązane kwestie wymagające weryfikacji podczas implementacji

### Plany (`plans/`)

Plan to wykonalny podział specyfikacji na dyskretne zadania. Plany są zaprojektowane do konsumpcji przez agentów wykonujących (podagentów lub przepływy pracy executing-plans) i following a strict format:

- **Dyrektywa nagłówka** — informuje agentów, jakiej umiejętności użyć do implementacji
- **Cel i podsumowanie architektury** — krótkie przypomnienie odsyłające do specyfikacji
- **Mapa plików** — tabele nowych i modyfikowanych plików z ich odpowiedzialnościami
- **Zadania** — numerowane, każde zawiera:
  - Dotyczone pliki (z numerami linii, jeśli ma to zastosowanie)
  - Kroki śledzone checkboxami (`- [ ]`)
  - Wbudowane bloki kodu pokazujące dokładne dodania lub zmiany
  - Polecenia weryfikacji (`cargo build`, `cargo test`, `cargo clippy`)
  - Komunikaty commitów git

Zadania budują się na siebie sekwencyjnie. Zadanie nie jest uznane za zakończone, dopóki jego polecenie weryfikacji nie przejdzie.

---

## Praca z tymi dokumentami

### Dla implementatorów

1. Najpierw przeczytaj specyfikację, aby zrozumieć *dlaczego* podjęto decyzje
2. Podążaj za planem zadanie po zadaniu, odznaczając kroki w miarę postępu
3. Uruchom polecenie weryfikacji po każdym zadaniu przed commitem
4. Użyj komunikatu commitu na końcu każdego zadania dosłownie

Plany odnoszą się do dokładnych numerów linii w istniejącym kodzie. Jeśli numery linii uległy przesunięciu z powodu innych zmian, zlokalizuj referencyjny symbol lub strukturę po nazwie.

### Zadania delta

Plany mogą zawierać **zadanie DELTA** na końcu, które nadpisuje wcześniejsze zadania. Dzieje się tak, gdy implementacja ujawnia, że założenie projektowe było błędne. Zadanie delta jawnie określa, co zastępuje i dlaczego.

Na przykład w planie MCP OAuth Discovery **Zadanie 12 (DELTA)** zmienia przepływ OAuth z inicjowanego przez demona (lokalny listener callback) na inicjowany przez UI (callback kierowany przez serwer API). Nadpisuje logikę przepływu uwierzytelniania z Zadań 5–7, ponieważ efemeryczny lokalny port w oryginalnym projekcie był nieosiągalny w wdrożeniach Docker.

Gdy istnieje delta, przeczytaj ją uważnie przed implementacją zadań, które modyfikuje.

---

## Kluczowe konwencje

### Format nagłówka planu

Każdy plan zaczyna się od tego bloku dyrektyw:

```markdown
> **Dla agentów wykonujących:** WYMAGANA POD-UMIEJĘTNOŚĆ: Użyj superpowers:subagent-driven-development
> (zalecana) lub superpowers:executing-plans, aby zaimplementować ten plan zadanie po zadaniu.
> Kroki używają składni checkboxa (`- [ ]`) do śledzenia.
```

To sygnalizuje, że plan jest ustrukturyzowany do automatycznego wykonania, a nie tylko do czytania przez człowieka.

### Komunikaty commitów

Plany określają dokładne polecenia `git commit` na końcu każdego zadania. Te stosują konwencję conventional commit z zakresem crate:

```
feat(runtime): add mcp_oauth module with WWW-Authenticate parser and core types
feat(kernel): implement KernelOAuthProvider with vault storage and PKCE flow
feat(api): add MCP OAuth auth endpoints and auth state in server list
```

### Bramki weryfikacji

Każde zadanie kończy się co najmniej jednym z:

- `cargo build --workspace --lib` — kompilacja
- `cargo test --lib -p <crate>` — testy jednostkowe przechodzą
- `cargo test --workspace` — pełna suita przechodzi
- `cargo clippy --workspace --all-targets -- -D warnings` — brak ostrzeżeń
- `npm run build` — dashboard się buduje

Ostatnie zadanie w planie to zazwyczaj pełne przejście weryfikacji przez build, testy i clippy.

---

## Przykład: MCP OAuth Discovery

Obecne artefakty w tym module (`2026-04-12-mcp-oauth-discovery`) ilustrują pełny wzorzec:

| Artefakt | Zawartość |
|----------|-----------|
| **Specyfikacja** | Projekt wstrzykiwania oparty na cechach (traits) dla OAuth na połączeniach MCP Streamable HTTP. Decyduje, że runtime definiuje cechę `McpOAuthProvider`, a jądro ją implementuje. Trójwarstwowe odkrywanie metadanych (WWW-Authenticate → .well-known → config.toml). Przepływ uwierzytelniania inicjowany przez UI z callback przez port API 4545. |
| **Plan** | 12 zadań (11 + 1 delta) obejmujących 4 craty: `librefang-types`, `librefang-runtime`, `librefang-kernel`, `librefang-api`, plus dashboard React. Zaczyna się od typów konfiguracji, buduje się przez cechę runtime i parser, implementację jądra, punkty końcowe API, a kończy testami integracyjnymi i pełną weryfikacją. |

Plan pokazuje, jak specyfikacja projektowa tłumaczy się na konkretne, uporządkowane, weryfikowalne kroki implementacji.

---

## Wzorce wielocratowe widoczne w tych dokumentach

Specyfikacje i plany w tym module dokumentują kilka wzorców architektonicznych używanych w całej bazie kodu:

- **Wstrzykiwanie przez cechy (trait injection)** — Craty runtime definiują cechy (`McpOAuthProvider`); craty jądra dostarczają implementacje. Pozwala to uniknąć cyklicznych zależności, zachowując testowalność logiki w izolacji.
- **Wzorzec KernelHandle** — Runtime definiuje interfejsy, jądro łączy je z konkretną infrastrukturą (vault, HTTP, rozszerzenia).
- **Trójwarstwowe odkrywanie** — Preferuj odkrywanie natywne dla protokołu (`WWW-Authenticate`, `.well-known`), z fallbackiem do jawnej konfiguracji. Konfiguracja nadpisuje odkrywanie tam, gdzie istnieją oba.
- **Nieblokujący start demona** — Długotrwałe operacje (uwierzytelnianie w przeglądarce) nie blokują uruchamiania. Demon oznacza stan i kontynuuje; ukończenie jest asynchroniczne.
- **Przestrzenie nazw kluczy vault** — Sekrety przechowywane z kluczami z prefiksami (`mcp_oauth:{url}:access_token`), aby uniknąć kolizji między funkcjonalnościami.
