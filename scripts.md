# skrypty

# scripts/

Automatyzacja repozytorium: hooki Gita, bramki lintowania CI, narzędzia wydawnicze, generowanie kodu SDK oraz infrastruktura serwowania instalatora. Każdy skrypt jest zaprojektowany tak, aby działać identycznie na laptopie współtwórcy i w CI.

## Moduły podrzędne w pigułce

| Moduł podrzędny | Zakres |
|---|---|
| [scripts](scripts.md) | Skrypty Pythona i powłoki najwyższego poziomu — sprawdzenia atrybucji w changelogu, wymuszanie niezmienników architektonicznych, generowanie kodu SDK, rusztowanie artykułów wydawniczych |
| [docker](docker.md) | Dockerfile sprawdzający instalator (smoke-test) w czystym kontenerze |
| [hooks](hooks.md) | Hooki Gita (`pre-commit`, `pre-push`, `commit-msg`) podłączone przez `core.hooksPath` |
| [tests](tests.md) | Testy regresyjne Bash/POSIX dla hooków, logiki instalatora i znaczników postępu demona |
| [workers](workers.md) | Funkcje Cloudflare Pages przekierowujące `/install.sh` i `/install.ps1` do najnowszego zasobu wydania na GitHubie |

## Jak ze sobą współpracują

### Cykl życia commita

Moduł [hooks](hooks.md) pilnuje każdego commita i pusha lokalnie. `pre-commit` pozostaje szybki (poniżej 2 s), uruchamiając tylko sprawdzenia w zakresie zdiffowanego indeksu; cięższe weryfikacje są odkładane do `pre-push` lub CI. Hook `commit-msg` wymusza zasady atrybucji AI, a `check-changelog-attribution.py` (w [głównych skryptach](scripts.md)) weryfikuje atrybucję `(@user)` na wpisach changeloga i fragmentach `changelog.d/` — uruchamia się zarówno w ścieżce hooków (na zindeksowanych fragmentach), jak i w CI (na pełnym pliku).

Moduł [tests](tests.md) zamyka pętlę: `commit-msg-attribution.sh` testuje hook `commit-msg`, a `pre-commit-sha-fallback.sh` weryfikuje logikę synchronizacji linii bazowej SHA hooka `pre-commit`. Te testy istnieją, ponieważ skrypty powłoki wywoływane przez Gita są niewidoczne dla test runnera Cargo.

### Instalator i potok wydawniczy

Trzy moduły podrzędne współpracują, aby instalator był godny zaufania od końca do końca:

1. **[workers](workers.md)** serwuje przyjazne URL-e (`/install.sh`, `/install.ps1`), pobierając najnowsze metadane wydania z GitHuba i przekierowując do pasującego zasobu dla danej platformy.
2. **[docker](docker.md)** przeprowadza smoke-test `web/public/install.sh` — weryfikacja składni i wykrywanie platformy domyślnie, opcjonalnie pełna instalacja binariów — pełniąc rolę bramki CI dla zmian w instalatorze.
3. **[tests](tests.md)** dostarcza `install_sh_test.sh` — zestaw POSIX sh testujący logikę instalatora bez Dockera.

Po stronie wydawniczej `changelog-to-article.sh` (główny) tworzy rusztowanie artykułu wydawniczego z sekcji changeloga, uzupełniając sprawdzenia atrybucji, które już zweryfikowały treść changeloga.

### Generowanie kodu

`codegen-sdks.py` (główny) generuje powiązania SDK w JavaScript, Pythonie i Ruście na podstawie jądra API. Jest napędzany przez helpery `_tag_pascal` / `_tag_attr` dla spójnej nazewnictwa we wszystkich trzech celach. Zmiany w generowanym wyniku są sprawdzane pod kątem rozjazdu w `pre-push` i CI.

## Wspólne konwencje

- **Brak konieczności czytania kodu źródłowego w czasie wykonania** — hooki i sprawdzenia operują na diffach Gita, zindeksowanej zawartości lub metadanych API, co zapewnia ich przenośność między środowiskami.
- **Atrybucja jest obowiązkowa** — każdy wpis changeloga, fragment changeloga i commit z asystą AI musi zawierać atrybucję `(@user)`, wymuszaną w czasie hooka i w CI przez tę samą logikę bazową (`has_attribution`, `bullet_block_has_attribution`).
- **Testy leżą obok tego, co testują** — gdy zachowanie nie jest osiągalne przez Cargo, test powłoki w module [tests](tests.md) je pokrywa.
