# sdk

# SDK

Wielojęzyczne biblioteki klienckie dla LibreFang Agent OS. Moduł SDK zapewnia typowany dostęp do REST API oraz framework do budowania adapterów kanałów łączących zewnętrzne platformy komunikacyjne z runtime'em agenta.

## Zawartość

| Podmoduł | Język | Auto-generowany? | Zakres |
|---|---|---|---|
| [Go SDK](sdk-go.md) | Go 1.21 | Tak — z `openapi.json` | Tylko klient REST |
| [JavaScript SDK](sdk-javascript.md) | JS/TS (Node ≥ 18) | Tak — z `openapi.json` | Tylko klient REST |
| [Python SDK](sdk-python.md) | Python (tylko stdlib) | Klient REST wygenerowany; sidecar pisany ręcznie | Klient REST + skrypty agenta + framework sidecar |
| [Rust SDK](sdk-rust.md) | Rust (async, tokio) | Klient REST pisany ręcznie; sidecar pisany ręcznie | Klient REST + framework sidecar + adapter Telegram |

## Dwie warstwy, wspólne kontrakty

### Klienci REST API

Klienci Go, JavaScript, Python (`LibreFang`) oraz Rust (`librefang`) udostępniają tę samą powierzchnię zasobów — agentów, umiejętności, modele, dostawców, kanały, poświadczenia, przepływy pracy, wtyczki, audyt i inne — przez LibreFang REST API (domyślnie `http://localhost:4545`). Każdy klient obsługuje strumieniowanie SSE dla odpowiedzi agenta w czasie rzeczywistym.

Klienci Go i JavaScript są w pełni auto-generowani z `openapi.json` przez `scripts/codegen-sdks.py`. Klienci REST Python i Rust obejmują te same punkty końcowe za pomocą idiomatycznych, utrzymywanych ręcznie wrapperów. Wybierz ten, który pasuje do Twojego środowiska uruchomieniowego; kontrakt sieciowy jest identyczny.

### Framework Sidecar

Adaptery kanałów to procesy zewnętrzne, które tłumaczą między platformą komunikacyjną (Telegram, Bluesky, DingTalk, Feishu, Email itd.) a demonem LibreFang. Demon uruchamia każdy adapter jako proces podrzędny i komunikuje się przez stdin/stdout za pomocą JSON rozdzielonego znakami nowej linii.

Zarówno [Python](sdk-python.md), jak i [Rust](sdk-rust.md) zawierają runtime sidecar implementujący ten protokół:

| Wiadomość | Kierunek | Przeznaczenie |
|---|---|---|
| `Ready` | Adapter → Demon | Ogłaszanie możliwości i schematu |
| `Event` | Adapter → Demon | Treść przychodząca (wiadomość, wywołanie zwrotne, odpowiedź z ankiety) |
| `Send` | Demon → Adapter | Treść wychodząca: tekst, media, zawartość interaktywna |
| `Command` | Demon → Adapter | Wskaźniki pisania, reakcje, cykl życia strumieniowania |

Pakiet Python służy jako implementacja referencyjna, z produkcyjnymi adapterami dla Telegrama, Bluesky, DingTalk, Feishu oraz Email. Crate Rust `librefang-sidecar` dostarcza trait `SidecarAdapter` oraz adapter `librefang-sidecar-telegram`, który ma jawną parzystość funkcjonalną z adapterem Python Telegram — ten sam kształt sieciowy, ta sama mapa reakcji emoji, ta sama semantyka kontroli dostępu.

## Parzystość między językami

Adapter Telegram to kanoniczny przykład równoważności między językami: implementacja Rust odzwierciedla `sdk/python/librefang/sidecar/adapters/telegram.py` pole po polu. Gdy nowa funkcjonalność trafi do Python referencyjnego, adapter Rust jest oczekiwany, że podąży za nim.

## Wybór klienta

| Jeśli potrzebujesz... | Użyj |
|---|---|
| Szybkich wywołań REST z funkcji serverless | **JavaScript** lub **Go** — bezproblemowe, generowane |
| Skryptu agenta uruchamianego w piaskownicy (bez zależności) | **Python** — działa wyłącznie na stdlib |
| Wysokowydajnej lub bezpiecznej typowo usługi backendowej | **Rust** — async, zero-kosztowy |
| Nowego adaptera kanału | **Python** (adaptery referencyjne do skopiowania) lub **Rust** (framework oparty na traitach) |

## Potok generowania

```
openapi.json
    │
    ▼
scripts/codegen-sdks.py
    │
    ├──► sdk/go/librefang.go          (nadpisywanie przy regeneracji)
    └──► sdk/javascript/index.js      (nadpisywanie przy regeneracji)
```

Klient REST Python oraz cały kod sidecar są utrzymywane ręcznie. Nigdy nie edytuj wygenerowanych plików — zostaną nadpisane przy następnej regeneracji.
