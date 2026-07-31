# skrypty — testy

# skrypty/testy

Testy regresji, dymne i korpusowe dla hooków, instalatorów i skryptów generowania kodu, które znajdują się poza grafem crate'ów Rusta. Te testy strzegą zachowań, do których program uruchamiający testy Cargo nie ma dostępu: hooków powłoki wykonywanych przez Gita, instalatora wczytywanego przez `curl|sh` oraz skryptów narzędziowych w Pythonie wywoływanych z CI lub stacji roboczych deweloperów.

## Co się tu znajduje

| Plik | Testowany obiekt | Język |
|---|---|---|
| `channel_progress_smoke.sh` | Markery postępu wykonywania narzędzi — end-to-end przez demona | Bash |
| `commit-msg-attribution.sh` | `scripts/hooks/commit-msg` — straż atrybucji AI | Bash |
| `install_sh_test.sh` | `web/public/install.sh` — logika instalatora | POSIX sh |
| `pre-commit-sha-fallback.sh` | `scripts/hooks/pre-commit` — synchronizacja linii bazowej SHA openapi | Bash |
| `pre-commit-spaces.sh` | `scripts/hooks/pre-commit` — cytowanie ścieżek rustfmt | Bash |
| `test_check_changelog_attribution.py` | `scripts/check-changelog-attribution.py` | Python |
| `test_codegen_sdks.py` | `scripts/codegen-sdks.py` | Python |

## Testy regresji hooków

### commit-msg-attribution.sh

Test oparty na korpusie dla straży atrybucji AI w hooku commit-msg. Testuje dwie niezależne kontrolki:

1. **Wyrażenie regularne treści komunikatu** — odrzuca commity, których komunikat zawiera ciągi atrybucji (`🤖 Claude Code`, `Co-Authored-By: …`, `Generated with Claude`). Korpus zawiera warianty bez spacji (`ClaudeCode`), pojedynczej spacji, podwójnej spacji i mieszanej wielkości liter, aby zablokować kwantyfikator `[[:space:]]*` i flagę `-i`, którą wcześniejszy błąd osłabił.

2. **Kontrola tożsamości autora** — odrzuca commety autorstwa znanej tożsamości AI nawet wtedy, gdy treść komunikatu jest czysta. Obejmuje nazewnictwo rodzina-wersja (`Claude 3 Opus`) oraz wersja-rodzina (`Claude 3.5 Sonnet`, `claude-3-5-sonnet`), a także skrzynkę `noreply@anthropic.com`. Akceptuje ludzi, którzy przypadkowo mają adres `anthropic.com` lub imiona zaczynające się od tych samych liter (`Claudia`, `Claudio`).

Dwie funkcje pomocnicze napędzają korpus:

- `run <komunikat>` — zapisuje komunikat do pliku tymczasowego, wymusza ludzką tożsamość autora i wywołuje hook. Kod powrotu wskazuje akceptację/odrzucenie.
- `run_as <nazwa> <email>` — zapisuje czysty komunikat i wariuje tożsamość autora, izolując kontrolkę autora od wyrażenia regularnego komunikatu.

### pre-commit-sha-fallback.sh

Test regresji dla zgłoszenia #5664: synchronizacja linii bazowej SHA openapi w hooku pre-commit była cicho pomijana na stacjach roboczych z Linuksem, gdzie istnieje tylko `sha256sum` (brak `shasum`). Test buduje jednorazowe repozytorium gita z układem `openapi.json` + `xtask/baselines/openapi.sha256`, przechowuje zmodyfikowaną specyfikację, maskuje `shasum` z PATH za pomocą shim'a zwracającego kod 127, a następnie weryfikuje:

- Hook kończy z kodem 0 w PATH zawierającym tylko `sha256sum`.
- Plik linii bazowej zawiera prawidłowy skrót przechowywanej specyfikacji.
- Odświeżona linia bazowa została automatycznie przechowana przez hook.

Kopiuje `.gitleaks.toml` repozytorium do repozytorium jednorazowego, aby krok skanowania sekretów nie zawiódł z powodu braku konfiguracji.

### pre-commit-spaces.sh

Test regresji dla zgłoszenia #5664: potok rustfmt w hooku pre-commit dzielił niescytowany `$STAGED_RS` na słowa, więc plik o nazwie `with space.rs` był traktowany jako dwie osobne ścieżki. Test przechowuje celowo źle sformatowany plik `with space.rs`, uruchamia hook i weryfikuje:

- Hook kończy z kodem niezerowym (odrzucenia commita).
- Odrzucenie nastąpiło przez ścieżkę rustfmt (`grep -q "not rustfmt-clean"`).

Pomija test czysto, gdy `rustfmt` nie jest zainstalowany.

## Pakiet testów instalatora

### install_sh_test.sh

Pakiet testów w POSIX-sh dla `web/public/install.sh`, wczytywany z `LIBREFANG_INSTALLER_SOURCE_ONLY=1`, aby funkcje instalatora były dostępne bez wykonywania instalacji. Obejmuje:

**Wykrywanie RC powłoki**
- Mapowania `shell_rc_from_shell` dla zsh, bash, fish.
- `choose_shell_rc` cofa się do `$SHELL`, gdy `detect_user_shell` zwraca pusty wynik (scenariusz `curl|sh`).
- Kolejność powrotu według istnienia pliku: `.zshrc` → `.bashrc` → konfiguracja fish.
- Kontrolka „już zainstalowany" dopasowuje `\.librefang/bin` dokładnie, a nie dowolną linię zawierającą „librefang" (co dałoby fałszywy wynik dla użytkownika o nazwie `librefang`).

**Analiza flag i stan sesji**
- `is_enabled` akceptuje `1/true/yes/on` (bez względu na wielkość liter) i odrzuca `0/false/no/off/""`.
- `SESSION_NEEDS_PATH_REFRESH` poprawnie wykrywa, czy katalog instalacji jest w PATH.
- `RESTART_SHELL` preferuje `$SHELL`, cofa się do `USER_SHELL`.

**Wykrywanie powłoki nadrzędnej**
Symuluje `ps` za pomocą fałszywego binarium zwanym stanowym, które zwraca `sh` przy pierwszym zapytaniu `comm` i `zsh` przy drugim, weryfikując, że `detect_user_shell` przechodzi przez drzewo procesów, aby znaleźć rzeczywistą powłokę nadrzędną.

**Wykrywanie dystrybucji (`detect_distro`)**
Oparte na fiksturach przez globalne zmienne `OS_RELEASE_FILE` i `NIXOS_MARKER_FILE`:

| Fikstura | Weryfikuje |
|---|---|
| `os-release-nixos` | `DISTRO=nixos`, brak `DISTRO_LIKE`, nie rodzina debian |
| `os-release-deepin` (`ID=Deepin`) | Małe litery do `deepin`, `DISTRO_LIKE=debian`, rodzina debian |
| `os-release-deepin-bare` | `deepin` dopasowane przez własne ID gdy `ID_LIKE` nieobecne |
| `os-release-ubuntu` | `DISTRO=ubuntu`, `DISTRO_LIKE=debian` |
| `os-release-derivative` (`ID_LIKE="ubuntu debian"`) | Wielowyrazowy `ID_LIKE` niescytowany i dopasowany |
| `os-release-inherited-debian` + znacznik NixOS | Znacznik NixOS ma priorytet nad dziedziczonym `ID=debian` |
| Brakujący os-release | Degradeuje do `unknown`, nigdy nie zawodzi |

**Polityka powrotu platformy (NixOS)**
NixOS nie ma `/lib64/ld-linux-x86-64.so.2`, więc powrót glibc (`gnu`) musi być pominięty przed pobraniem, a nie wykryty po nieudanym sprawdzeniu `--version`.

- `effective_platform_fallback` zwraca puste na NixOS, zachowuje `gnu` na Ubuntu.
- `apply_distro_platform_policy` wyświetla komunikat wymieniający NixOS i brakujący interpreter; pozostaje ciche poza NixOS.
- `should_try_platform_fallback` obejmuje wszystkie trzy stany ponowienia: warto próbować, już utrzymane, brak użytecznego powrotu.
- **Niezależność od kolejności**: uruchomienie `detect_distro` → `detect_platform` → `apply_distro_platform_policy` vs. `detect_distro` → `apply_distro_platform_policy` → `detect_platform` daje tę samą decyzję o powrocie. Reguła NixOS nie może zależeć od kolejności wywołań.

**Rozpoznawanie wersji (`resolve_installable_version`)**
Używa fałszywego `curl` napędzanego przez `MOCK_TAGS`, `MOCK_GOOD_TAGS` i `MOCK_BAD_PLATFORM`:

- Pomija zablokowany najnowszy wydanie (brak zasobów) i cofa się do starszego.
- Cofa się przez warianty platformy (musl → gnu) w ramach wydania.
- Odmawia kompilacji gnu na NixOS.
- Szanuje `LIBREFANG_VERSION` jako twarde przypięcie (bez sondowania zasobów).
- Traktuje `LIBREFANG_PREFERRED_VERSION` jako miękką podpowiedź (cofa się, gdy zablokowane).
- Zawodzi, gdy żadne wydanie nie dostarcza instalowalnego pakietu.

Fałszywy curl wymusza, aby sondy tarball używały żądania zakresu 1-bajtowego (`-r 0-0`), głośno zawodząc, gdy optymalizacja ulegnie regresji.

**Cofanie wersji binarnej (`install_binary_with_rollback`)**
- Działająca aktualizacja: instaluje nowy binarium, usuwa kopię zapasową.
- Zepsuta aktualizacja: cofa do poprzedniego binarium, czyści kopię zapasową.
- Zepsuta świeża instalacja (brak istniejącego binarium): całkowicie usuwa nieuruchamialne binarium.

**Podpowiedź pulpitu (`print_debian_desktop_hint`)**
Symuluje `pkg-config` i `apt-cache`, aby zweryfikować, że podpowiedź nigdy nie przepisuje pakietu niedostępnego w repozytoriach, wymienia poprawną serię webkit podczas sondowania i pozostaje cicha na hostach bez środowiska graficznego lub spoza Debiana.

**Podpowiedź instalacji ze źródła (`print_source_install_hint`)**
NixOS otrzymuje adres URL flake (`nix profile install`, `nix run`, `services.librefang.enable`); wszystkie inne dystrybucje otrzymują powrót `cargo install`.

## Testy skryptów Pythona

### test_check_changelog_attribution.py

Testuje `scripts/check-changelog-attribution.py` importując go przez `importlib` (bez wymogu instalacji pakietu). Dwie grupy testów:

**`bullet_block_has_attribution`** — weryfikuje, że atrybucja `(@user)` jest rozpoznawana w dowolnym miejscu bloku kontynuacji punktu, a nie tylko na linii znacznika `- `. Odpowiada to formatowaniu jednego zdania na linię, gdzie długi punkt niesie swoją atrybucję na końcowej linii kontynuacji. Testuje również:

- Atrybucja nie rozchodzi się przez puste linie ani sąsiednie punkty.
- `# pragma: no-attribution` na linii kontynuacji zwalnia punkt z wymogu atrybucji.

**`fragment_tests`** — testuje `check_fragment` i `classify_fragment_path` względem wpisów w `changelog.d/`:

- Poprawnie sformowane fragmenty w rozpoznanych sekcjach (`fixed/`, `added/`, itd.) przechodzą.
- Brakująca atrybucja jest oznaczana na linii 1 jako `MISSING_ATTRIBUTION`.
- Atrybucja oddzielona pustą linią nie jest liczona.
- Puste fragmenty są odrzucane.
- Niezrozpoznalne katalogi sekcji (`fix/` zamiast `fixed/`) są odrzucane nawet z atrybucją, ponieważ montaż nie ma dla nich nagłówka.
- Pliki infrastrukturalne (`README.md`, `.gitkeep`, `.txt`) nigdy nie są skanowane.

### test_codegen_sdks.py

Test dymny dla `scripts/codegen-sdks.py` względem rzeczywistego `openapi.json`. Weryfikuje niezmienniki, które historycznie ulegały regresji:

- **Ładowanie operacji** — `invoke_tool` ma `agent_id` w `query_params` i `has_body=True`; `list_agents` ma pełny zestaw parametrów zapytania; `send_message_stream` jest wykrywane jako operacja strumieniowa.
- **Sygnatury SDK** — sygnatury `invoke_tool` zawierają parametr query/agent_id we wszystkich czterech SDK (Python, JS, Go, Rust).
- **Poprawność strumieni** — Go używa `bufio.NewReaderSize` (a nie gołego `strings.Split`); Rust używa `Vec<u8>` (a nie `from_utf8_lossy` na fragmentach); oba podają status HTTP w zdarzeniach błędu.
- **Limit rozmiaru linii SSE** — `MAX_SSE_LINE` / `maxSSELine` obecne w wyniku Rust/Go.
- **Ucieczka słów zastrzeżonych** — `_py_safe("class")` → `"class_"`, `_rust_safe("type")` → `"type_"`.

## Test dymny

### channel_progress_smoke.sh

Żywy test integracyjny end-to-end, który weryfikuje, że markery postępu wykonywania narzędzi (`🔧 Web Search`) są widoczne przez demona. W przeciwieństwie do innych testów, ten wymaga zewnętrznych warunków wstępnych i nie działa w CI:

- Klucz API LLM w zmiennych środowiskowych (`GROQ_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` lub `MINIMAX_API_KEY`).
- Co najmniej jeden skonfigurowany adapter kanału w `~/.librefang/config.toml`.
- Binarium wydania w `target/release/librefang`.

Skrypt uruchamia demona, ponownie używa istniejącego włączonego agenta lub tworzy efemerycznego (z możliwością narzędzia `web_search`) przez API, wysyła komunikat prawdopodobnie wyzwalający wywołanie narzędzia i sprawdza log sesji pod kątem zdarzeń `tool_use`. Pełna weryfikacja dostarczenia kanałem wymaga skonfigurowanego odbiornika webhooka i jest udokumentowana zewnętrznie.

Czyszczenie uruchamia się przy EXIT przez `trap`, zatrzymując demona i usuwając wszelkich utworzonych agentów tymczasowych.

## Uruchamianie testów

```sh
# Testy powłoki — uruchamiane z katalogu głównego repozytorium
bash scripts/tests/commit-msg-attribution.sh
bash scripts/tests/pre-commit-sha-fallback.sh
bash scripts/tests/pre-commit-spaces.sh
sh scripts/tests/install_sh_test.sh

# Testy Pythona — uruchamiane bezpośrednio
python3 scripts/tests/test_check_changelog_attribution.py
python3 scripts/tests/test_codegen_sdks.py

# Test dymny (wymaga działającego LLM + demona)
bash scripts/tests/channel_progress_smoke.sh
```

Testy regresji hooków pomijają czysto, gdy ich zależność (`sha256sum`, `rustfmt`) jest nieobecna, kończąc z kodem 0 i komunikatem `SKIP:`. Pakiet testów instalatora i testy Pythona nie mają ścieżek opcjonalnego pomijania — wykonują się do końca lub zawodzą.
