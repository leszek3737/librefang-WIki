# Root — Cross.toml

# Cross.toml

## Przegląd

`Cross.toml` to plik konfiguracyjny narzędzia [`cross`](https://github.com/cross-rs/cross) — narzędzia do cross-kompilacji w języku Rust bez konieczności dodatkowej konfiguracji. Plik ten definiuje dostosowania środowiska kompilacji dla dwóch celów ARM64: natywnego systemu Linux (GNU) oraz Androida. Zapewnia, że przy tworzeniu binariów dla tych architektur używane są poprawne biblioteki systemowe i obrazy Docker.

Ten plik jest odczytywany w czasie kompilacji przez narzędzie `cross` (wywoływane za pomocą `cross build --target <triple>`) i nie zawiera wykonywalnego kodu Rust. Znajduje się w katalogu głównym repozytorium obok pliku `Cargo.toml`.

---

## Konfiguracja celów

### `aarch64-unknown-linux-gnu`

Ten cel tworzy standardowe binaria Linux GNU dla architektury ARM64 (np. dla Raspberry Pi 4/5, serwerów ARM lub runnerów CI opartych na ARM).

Hook `pre-build` wykonuje polecenia powłoki wewnątrz kontenera Docker narzędzia cross **przed** rozpoczęciem etapu kompilacji Rust:

| Krok | Polecenie | Cel |
|------|-----------|-----|
| 1 | `dpkg --add-architecture $CROSS_DEB_ARCH` | Włącza instalację pakietów dla architektury docelowej (np. `arm64`) |
| 2 | `apt-get update` | Odświeża indeks pakietów |
| 3 | `apt-get install --assume-yes libssl-dev:$CROSS_DEB_ARCH` | Instaluje nagłówki/biblioteki deweloperskie OpenSSL dla architektury docelowej |
| 4 | `apt-get install --assume-yes libdbus-1-dev:$CROSS_DEB_ARCH` | Instaluje nagłówki/biblioteki deweloperskie D-Bus dla architektury docelowej |

Zmienna środowiskowa `$CROSS_DEB_ARCH` jest udostępniana przez narzędzie `cross` i przyjmuje wartość poprawnego ciągu architektury Debiana dla danej trójki docelowej (np. `arm64`). Podejście multiarch pozwala bibliotekom architektur hosta i docelowej współistnieć w tym samym kontenerze.

**Dlaczego te biblioteki są potrzebne:**
- **libssl-dev** — Wymagana przez skrzynki Rust łączące się z OpenSSL (np. `openssl-sys`, `native-tls`, `reqwest` z domyślnymi funkcjonalnościami).
- **libdbus-1-dev** — Wymagana przez skrzynki współpracujące z demonem systemowym D-Bus (np. `dbus`, `zbus`).

### `aarch64-linux-android`

Ten cel tworzy binaria kompatybilne z Android NDK dla urządzeń i emulatorów ARM64.

Zamiast wykonywania hooków pre-build, nadpisuje domyślny obraz Docker:

```
ghcr.io/cross-rs/aarch64-linux-android:main
```

To przypina obraz do tagu `main` oficjalnego obrazu Androida `cross-rs`, który zawiera wstępnie skonfigurowane Android NDK i łańcuch narzędzi. Użycie `:main` zapewnia pobranie najnowszego utrzymywanego obrazu, ale może wprowadzić niepowtarzalne kompilacje, jeśli obraz zmieni się w repozytorium źródłowym. Jeśli powtarzalność jest krytyczna, rozważ przypięcie do konkretnego skrótu (digest) lub tagu.

---

## Użycie

Aby kompilować dla skonfigurowanych celów:

```bash
# ARM64 Linux (GNU)
cross build --target aarch64-unknown-linux-gnu

# ARM64 Android
cross build --target aarch64-linux-android
```

Nie są wymagane żadne dodatkowe flagi — narzędzie `cross` odczytuje ten plik automatycznie, gdy znajduje się on w katalogu głównym przestrzeni roboczej.

---

## Powiązanie z bazą kodu

Ta konfiguracja obsługuje scenariusze cross-kompilacji, w których projekt zależy od natywnych bibliotek C (OpenSSL, D-Bus). Bez tych hooków pre-build linker zgłosiłby błąd nierozwiązanych symboli podczas celowania w `aarch64-unknown-linux-gnu`. Nadpisanie obrazu dla celu Android istnieje, ponieważ domyślny obraz `cross` może nie nadążać za wymaganiami projektu w zakresie kompatybilności z NDK lub się od nich różnić.

Plik nie ma wpływu na działanie w czasie rzeczywistym ani bezpośrednich połączeń z innymi modułami źródłowymi — jest to wyłącznie kwestia infrastruktury kompilacji.
