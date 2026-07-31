# Root — Dockerfile

# Dockerfile

Obraz kontenera produkcyjny dla **librefang** — demona w języku Rust z osadzonym panelem React. Jest to trójetapowy, wieloetapowy build, który kompiluje frontend, kompiluje binarkę Rust z osadzonymi w czasie kompilacji zasobami i składa minimalny obraz środowiska uruchomieniowego.

---

## Potok budowania

```mermaid
flowchart LR
    A["Etap 1: dashboard-builder
    node:20_20_2-alpine["20.20.2-alpine"]"] -->|"static/react"| C
    B["Etap 2: builder
    rust:1_94-slim-bookworm["1.94-slim-bookworm"]"] -->|"binarka librefang"| C
    B -->|"packages/"| C
    C["Etap 3: runtime
    node:22.11.0-bookworm-slim"]
    D["deploy/
    docker-entrypoint_sh["docker-entrypoint.sh"]"] --> C
```

### Etap 1 — Builder dashboardu (`dashboard-builder`)

Buduje frontend React, który jest dołączany do katalogu statycznych zasobów binarki Rust.

| Decyzja | Uzasadnienie |
|---|---|
| `node:20.20.2-alpine` (zablokowany minor) | Powtarzalne buildy miesiące po oznaczeniu tagiem. Zmienny `node:20-alpine` mógłby po cichu zmienić wersje patch i wygenerować inny obraz buildera. Zgodny z Node 20 LTS używanym w CI (`.github/workflows/ci.yml`, `.github/workflows/dashboard-build.yml`). |
| `corepack@latest` przed `corepack prepare` | Zestaw kluczy (keyring) dołączony do bazowego obrazu Node dezaktualizuje się, gdy pnpm rotuje klucze podpisywania, co powoduje `"Internal Error: Cannot find matching keyid"`. Odświeżenie corepack najpierw to naprawia. |
| `corepack prepare pnpm@10.33.0 --activate` | Pomija `fetchLatestStableVersion2` (który zawodzi w rejestru npm), aktywując dokładną wersję z `packageManager` w `package.json` bezpośrednio. |
| `pnpm install --frozen-lockfile --ignore-scripts` | Instalacja ze ścisłym lockfile'em z pominięciem skryptów postinstall dla hermetyzności. |
| Node ≥ 20.19 wymagany | Opcjonalne natywne wiązania Vite 8 / rolldown deklarują `engines: ^20.19.0`. Bez tego `pnpm install` po cichu pomija wiązanie `linux-x64-musl`, a `vite build` zawodzi w czasie require. |

**Wynik:** skompilowane zasoby React w `/build/static/react`, konsumowane przez Etap 2.

### Etap 2 — Builder Rust (`builder`)

Kompiluje binarkę release `librefang`.

| Decyzja | Uzasadnienie |
|---|---|
| `rust:1.94-slim-bookworm` (zablokowany minor) | Spełnia MSRV workspace'u w `Cargo.toml` (`[workspace.package].rust-version = "1.94.1"`). Zmienny `rust:1-slim-bookworm` mógłby przyjąć nowszy kompilator bez ostrzeżenia. |
| `libdbus-1-dev` | Wymagany przez `libdbus-sys`, transitywną zależność funkcji `sync-secret-service` pakietu `keyring` (dodaną w #3180). Bez tego skrypt builda panikuje z kodem wyjścia 101 — ta sama przyczyna co w #3259, która zablokowała publikację obrazu v2026.4.27-beta6. |
| Mounty pamięci podręcznej buildu | `cargo/registry`, `cargo/git` i `target` są montowane jako mounty pamięci podręcznej BuildKit, aby kompilacja zależności była buforowana między ponownymi buildami. |

#### Osadzone w czasie kompilacji zasoby

Kilka crate'ów używa makr `include_dir!` / `include_str!`, które wymagają obecności konkretnych poddrzew źródłowych w czasie buildu. Dockerfile kopiuje dokładnie te poddrzewa (`.dockerignore` wyklucza resztę):

| Ścieżka źródłowa | Osadzane przez | Skutek braku |
|---|---|---|
| `sdk/python/librefang` | `librefang-channels` przez `include_dir!("$CARGO_MANIFEST_DIR/../../sdk/python/librefang")` w `embedded_sdk.rs` (#5472) | Panika proc macro: `"sdk/python/librefang is not a directory"` |
| `deploy/` | `librefang-api` przez `include_str!("../../../deploy/...")` dla konfiguracji stosu observability (#3062) | `"couldn't read deploy/grafana/..."` |
| `crates/librefang-api/static/react` | Skopiowane z wyniku Etapu 1 | Frontend nie jest serwowany |

#### Polecenie buildu

```bash
cargo build --release --bin librefang --features telemetry
```

Tylko `telemetry` jest włączone — to pełny obraz demona. Adaptery kanałów działają jako boczne procesy (#5408 / #5461), więc stare aliasy funkcji `all-channels` / `core-channels` już nie istnieją. To odpowiada domyślnemu zestawowi funkcji CLI.

Wynikowa binarka jest kopiowana do `/usr/local/bin/librefang`, aby uciec z katalogu `target/` montowanego jako pamięć podręczna.

### Etap 3 — Obraz środowiska uruchomieniowego

| Baza | `node:22.11.0-bookworm-slim` (zablokowany minor Node 22 LTS) |
|---|---|
| Uzasadnienie blokowania | Zmienny `node:lts-bookworm-slim` mógłby po cichu przyjąć nowego majora, gdy alias `lts` się przesunie. |

#### Zależności środowiska uruchomieniowego

| Pakiet | Powód |
|---|---|
| `ca-certificates` | Weryfikacja TLS dla wychodzącego HTTPS |
| `curl` | Używany przez `HEALTHCHECK` |
| `python3`, `python3-venv` | Wsparcie środowiska uruchomieniowego SDK Python |
| `libicu72` | Środowisko uruchomieniowe ICU do przetwarzania tekstu |
| `libdbus-1-3` | Runtime `.so`, z którym linkuje `libdbus-sys`. Ścieżka inicjalizacji keyring uruchamia się wcześnie przy starcie; jeśli `.so` nie może zostać rozwiązany, proces kończy z kodem 101. |
| `gosu` | Narzędzie do porzucania uprawnień, używane przez `docker-entrypoint.sh` |

#### Postawa bezpieczeństwa

- Dedykowany użytkownik `librefang` (uid/gid 1001) jest tworzony przez `addgroup` / `adduser`.
- Zgodność z CIS Docker Benchmark §4.1: powłoka logowania jest ustawiona na `/sbin/nologin` przez `usermod`. Dockerfile celowo unika zbędnego bloku `groupadd -r librefang && useradd -r ...` (wprowadzonego w #3948), który koliduje z już utworzonym użytkownikiem — `groupadd` kończy z kodem 9 ("group already exists"), psując `docker build` na czystych drzewach.
- `/opt/librefang/packages` ma zmienionego właściciela na użytkownika `librefang`, ponieważ `COPY` domyślnie ustawia `root:root`.

---

## Entrypoint i Command

```
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["librefang", "start", "--foreground"]
```

Entrypoint (`deploy/docker-entrypoint.sh`) działa jako **root**, aby mógł zmienić właściciela bind-mountów i zainicjować katalog danych przed porzuceniem uprawnień przez `gosu`. Właściwy demon działa jako użytkownik `librefang`.

Entrypoint przepisuje również `api_listen` na podstawie zmiennej środowiskowej `$PORT` wstrzykiwanej przez Railway, Render lub Fly.

---

## Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s \
  CMD curl -fsS http://127.0.0.1:${PORT:-4545}/api/ready || exit 1
```

| Aspekt | Szczegóły |
|---|---|
| Endpoint | `/api/ready` (nie `/api/health`) — zmieniony w #6633 |
| Dlaczego gotowość, a nie żywotność | Dockerowy `HEALTHCHECK` nie restartuje niezdrowych kontenerów. Jego konsument to bramka `depends_on: condition: service_healthy` w Compose, która ma semantykę gotowości: „czy zależne mogą już wystartować?". Stary endpoint `/api/health` zwraca 200 nawet gdy jego ciało raportuje `status: degraded`, więc nigdy nie mógł zawieść bramki. |
| Forma powłoki (`CMD ...`) nie exec | Wymagana, aby `${PORT:-4545}` rozszerzało się w czasie uruchomienia ze zmiennych środowiskowych. |
| Okres startowy (20s) | Daje demonowi czas na bindowanie, uruchomienie `librefang init` przy pierwszym starcie i uruchomienie serwera axum. |
| Kubernetes | Całkowicie ignoruje `HEALTHCHECK`. Sondy K8s są deklarowane osobno w `deploy/kubernetes/`. |

---

## Środowisko

| Zmienna | Domyślna | Przeznaczenie |
|---|---|---|
| `LIBREFANG_HOME` | `/data` | Katalog danych dla trwałego stanu |
| `PORT` | `4545` (fallback) | Port nasłuchiwania, wstrzykiwany przez dostawców PaaS. Rozszerzany w czasie uruchomienia w healthchecku i przepisywany przez entrypoint. |

---

## Eksponowany port

`EXPOSE 4545` — domyślny port nasłuchiwania API. Nadpisuj w czasie uruchomienia przez `$PORT`.

---

## Kluczowe niezmienniki i pułapki

1. **Nigdy nie odblokowuj tagów obrazów bazowych.** Każda linia `FROM` blokuje konkretnego minora. Zmienne tagi (`node:20-alpine`, `rust:1-slim-bookworm`, `node:lts-*`) psują powtarzalność lub ryzykują ciche przeskoki wersji major.
2. **`sdk/python/librefang` musi być skopiowane.** Jest osadzane w czasie kompilacji. Potrzebne jest tylko to poddrzewo; `.dockerignore` wyklucza resztę `sdk/`.
3. **`deploy/` musi być skopiowane do buildera.** Konfiguracje observability (Prometheus, Tempo, OTEL Collector, Grafana) są osadzane przez `include_str!`. `flake.nix` wymienia te same ścieżki w swoim zestawie plików źródłowych.
4. **Nie dodawaj ponownie bloku `groupadd`/`useradd`.** Patrz #3948 — koliduje z istniejącym użytkownikiem `librefang` i psuje build.
5. **`--ignore-scripts` jest celowe** w instalacji pnpm dashboardu. Usunięcie reintrodukuje niehermetyczne efekty uboczne postinstall.
