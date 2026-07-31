# crates — librefang-llm-driver

# `librefang-llm-driver`

Krate traitów warstwy abstrakcji LLM LibreFanga. Definiuje trait `LlmDriver`, typy żądań/odpowiedzi, taksonomię błędów oraz rejestr wyczerpania w pamięci używany przez łańcuchy awaryjne. **Tu nie ma żadnych konkretnych implementacji dostawców** — one znajdują się w siostrzanej krate `librefang-llm-drivers` (zwróć uwagę na końcowe `s`).

## Cel i granice

Ta krate istnieje, aby dać konsumentom (jądro, test harnesses, pętla agenta) stabilną umowę traitów, której mogą polegać *bez* przechodniego ciągnięcia `reqwest`, stosów TLS lub dostarczonych SDK dostawców. Każdy konkretny sterownik (Anthropic, OpenAI, Gemini, Groq, Ollama, Claude Code, Codex CLI, …) jest zaimplementowany w oparciu o ten trait w `librefang-llm-drivers`.

Co jest zarządzane przez tę kratę:

- trait `LlmDriver` oraz wspierające typy `CompletionRequest` / `CompletionResponse` / `StreamEvent`
- enum `LlmError` (jedyny typ błędu dopuszczalny w pozycjach zwrotnych traitów)
- klasyfikacja błędów: `FailoverReason`, `ProviderErrorCode`, `LlmErrorCategory`, `classify_error`
- rejestr wyczerpania dostawców (`ProviderExhaustionStore`) konsultowany przez łańcuchy awaryjne
- `DriverConfig` — niezależna od dostawcy konfiguracja przekazywana do fabryki

Co celowo **nie** jest zarządzane przez tę kratę:

- Żadne okablowanie HTTP, strategia ponawiania ani formatowanie promptów
- Specyficzne dla dostawcy ciała żądań/odpowiedzi
- Wykonywanie narzędzi, pętla agenta, czy problemy jądra

## Architektura

```mermaid
flowchart TD
    subgraph librefang_llm_driver[ta krate]
        Trait[LlmDriver trait]
        Req[CompletionRequest]
        Resp[CompletionResponse]
        Err[LlmError]
        Exhaust[ProviderExhaustionStore]
        Class[classify_error / FailoverReason]
    end
    subgraph librefang_llm_drivers[krate siostrzana]
        Anthropic[AnthropicDriver]
        OpenAi[OpenAiDriver]
        Gemini[GeminiDriver]
        Cli[ClaudeCodeDriver / CodexCliDriver]
        Chain[FallbackChain]
    end
    Kernel[librefang-kernel]
    Tests[librefang-testing]

    Anthropic -.implementuje.-> Trait
    OpenAi -.implementuje.-> Trait
    Gemini -.implementuje.-> Trait
    Cli -.implementuje.-> Trait
    Chain -.implementuje.-> Trait
    Chain --> Exhaust
    Chain --> Class
    Kernel --> Trait
    Tests --> Trait
```

## Typy rdzeniowe

### trait `LlmDriver`

Jedyna abstrakcja implementowana przez konkretnych dostawców:

```rust
#[async_trait]
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest)
        -> Result<CompletionResponse, LlmError>;

    async fn stream(
        &self,
        request: CompletionRequest,
        tx: tokio::sync::mpsc::Sender<StreamEvent>,
    ) -> Result<CompletionResponse, LlmError> { /* domyślna implementacja */ }

    fn is_configured(&self) -> bool { true }
    fn family(&self) -> LlmFamily { LlmFamily::Other }
    fn is_coding_agent(&self) -> bool { false }
}
```

- `complete` to jedyna wymagana metoda.
- `stream` ma domyślną implementację, która opakowuje `complete()` i emituje pojedynczy `TextDelta` + `ContentComplete`. Prawdziwe sterowniki strumieniowe nadpisują ją. Domyślna implementacja propaguje błędy `tx.send` jako `LlmError::Http("stream receiver dropped")`, aby rozłączenia klienta ujawniały się jako anulowanie, a nie ciche połykanie (#3543).
- `family()` zwraca wysokopoziomową rodzinę formatu przewodowego (`Anthropic`, `OpenAi`, `Google`, `Local`, `Other`). Celowo szerszą niż tożsamość poszczególnego sterownika — przyszłe polityki przekrojowe opierają się na tej osi.
- `is_coding_agent()` odróżnia agentów kodowania opartych na CLI (Claude Code, Codex CLI, Gemini CLI, …) od dostawców surowego HTTP. Agenty kodowania zarządzają wyborem modelu i mogą wypełniać `CompletionResponse::actual_model`.

### `CompletionRequest`

Wszystkie pola potrzebne sterownikowi do obsłużenia tury. Kilka punktów projektowych warto znać:

- `messages` i `tools` są opakowane w `Arc<Vec<_>>`. Ponowienia, awaryjne przejścia i współdzielenie tur w pętli agenta zwiększają tylko liczniki referencji zamiast głęboko klonować wielo-set-kilobajtową historię wiadomości (#3586, #3766). Kod sterownika czyta przez `&request.messages` / `request.tools.iter()` i otrzymuje automatyczne dereferencjonowanie.
- `extra_body` to `BTreeMap<String, serde_json::Value>`, **nie** `HashMap`. Deterministyczna kolejność kluczy jest wymagana, ponieważ mapa jest spłaszczana do żądań przewodowych; niestabilna kolejność cicho unieważnia pamięć podręczną promptów dostawców (#3298).
- `prompt_caching` / `cache_ttl` / `prompt_cache_strategy` sterują umiejscowieniem punktów przerwania `cache_control` Anthropic. Enum strategii (`PromptCacheStrategy::SystemAndN(n)`) jest przycinany do limitów specyficznych dla dostawcy (4 w Anthropic).
- Pola `agent_id` / `session_id` / `step_id` / `sender_*` propagują tożsamość wywołującego. Sterowniki kompatybilne z HTTP ujawniają je jako nagłówki `x-librefang-{agent,session,step}-id`; sterowniki podprocesowe przekazują je dalej, aby mosty MCP mogły odtworzyć `ToolExecContext` po drugiej stronie.
- `reasoning_echo_policy` niesie metadane katalogu modeli, które mówią sterownikowi OpenAI, jak obsługiwać `reasoning_content` w historycznych turach asystenta (#4842).

`Default` jest zaimplementowane dla wygody — każdy rzeczywisty punkt wywołania musi nadal jawnie ustawić `model` i `messages`; domyślne żądanie nie jest nadające się do użycia jak jest.

### `CompletionResponse`

Niesie bloki treści, powód zatrzymania, wywołania narzędzi, użycie tokenów i dwa pola atrybucji dostawcy:

- `actual_provider` — ustawiane przez opakowania awaryjne (`FallbackChain`, `BudgetGatedDriver`), aby warstwa rozliczeń przypisywała wydatki do gniazda, które wykonało pracę, a nie do nominowanego gniazda. Zawsze `None` w wewnętrznych sterownikach liściowych.
- `actual_model` — ustawiane tylko przez sterowniki agentów kodowania, których uruchomiony CLI może rozwiązać model inny niż żądany. Dostawcy surowi honorują żądany identyfikator i zostawiają to jako `None`.

`text()` konkatenuje warianty `ContentBlock::Text`, pomijając bloki myślowe.

### `StreamEvent`

Enum niekompletny, emitowany przez `mpsc::Sender` przekazany do `stream()`. Wartościowe warianty:

- `TextDelta`, `ThinkingDelta` — przyrostowy tekst/rezonowanie
- `ToolUseStart` / `ToolInputDelta` / `ToolUseEnd` — cykl życia wywołania narzędzia
- `ContentComplete { stop_reason, usage }` — terminalny
- `PhaseChange { phase, detail }` — sygnał cyklu życia UX; kanoniczna stała `PHASE_RESPONSE_COMPLETE` sygnalizuje, że przetwarzanie końcowe (zapis sesji, proaktywna pamięć) zaraz się rozpocznie
- `ToolExecutionResult`, `OwnerNotice` — emitowane przez pętlę agenta, **nie** przez sterowniki LLM; konsumenci mostu kanałowego kierują `OwnerNotice` do DM właściciela

### `LlmFamily`

Pięć wariantów serializowanych jako snake_case (`anthropic`, `open_ai`, `google`, `local`, `other`). Implementacja `Display` odpowiada formie serde, więc logi i JSON się zgadzają.

## Obsługa błędów

Obsługa błędów jest warstwowa: `LlmError` to typ przewodowy zwracany ze sterowników; `FailoverReason` to klasyfikacja strukturalna używana przez łańcuchy awaryjne; `classify_error` / `LlmErrorCategory` to heurystyczny klasyfikator używany do diagnostyki dla użytkownika.

### `LlmError`

Enum `#[non_exhaustive]`. Warianty obejmują `Http`, `Api`, `RateLimited`, `Parse`, `MissingApiKey`, `Overloaded`, `AuthenticationFailed`, `ModelNotFound`, `TimedOut`, `AllProvidersExhausted`. Godne uwagi ograniczenia projektowe wymuszane w całej bazie kodu:

- **Żadnego wariantu-łapacza `String`.** Wszystkie warianty przenoszą uporządkowane pola (#3541, #3711).
- **Żadnego `Box<dyn Error>` w typach zwrotnych traitów.** Używaj `LlmError`.
- `Api` niesie opcjonalny `ProviderErrorCode`, więc klasyfikacja nie zależy od dopasowywania podłańcuchów czytelnych dla człowieka wiadomości (#3745).
- `TimedOut` przechowuje `partial_text: Option<Arc<str>>`, więc klonowanie błędu to inkrementacja licznika referencji O(1) nawet dla częściowych wyników o rozmiarze megabajtów. Implementacja `Display` odwołuje się tylko do `partial_text_len` — większość konsumentów nigdy nie dotyka treści. Dopasowywanie wzorców do wariantu to wspierany sposób przekazywania częściowych wyników (#3552).
- `AllProvidersExhausted` niesie posortowane `details: Vec<ProviderExhaustionDetail>` i opcjonalną `cause: Option<Box<LlmError>>` ujawnianą przez `Error::source()` za pomocą `#[source]` thiserror. Kiedy każde gniazdo zostało z góry pominięte z rejestru wyczerpania, `cause` wynosi `None`.

### `failover_reason()` — klasyfikacja strukturalna

`LlmError::failover_reason()` jest bezalokacyjna, nieomylna i czysto strukturalna. Mapuje każdy wariant na jedną z dziewięciu wartości `FailoverReason`:

| Wariant           | Odtworzenie                          |
|-------------------|--------------------------------------|
| `RateLimit(ms)`   | uśpienie, ponowienie tego samego dostawcy |
| `CreditExhausted` | pominięcie do następnego dostawcy  |
| `ModelUnavailable`| pominięcie do następnego dostawcy  |
| `ContextTooLong`  | propagacja — wywołujący musi skompresować |
| `Timeout`         | pominięcie do następnego dostawcy  |
| `HttpError`       | pominięcie do następnego dostawcy  |
| `AuthError`       | pominięcie do następnego dostawcy  |
| `ChainExhausted`  | terminalne — propagacja do użytkownika |
| `Unknown`         | natychmiastowa propagacja          |

`ChainExhausted` jest różne od `Unknown`: klasyfikacja się powiodła, łańcuch jest po prostu suchy. Ten podział ma znaczenie, ponieważ ich pomieszanie spowodowałoby, że wywołujący zapętlą się na znanym stanie terminalnym.

### `ProviderErrorCode`

Typowany tag dołączany do `LlmError::Api { code, .. }` przez sterowniki parsujące uporządkowane ciało błędu dostawcy. Warianty: `RateLimit`, `CreditExhausted`, `ContextLengthExceeded`, `ModelNotFound`, `AuthError`, `ServerUnavailable`, `ServerError`, `BadRequest`. Kiedy `code` jest obecne, `failover_reason()` klasyfikuje przez typowaną wartość (wyczerpujące, niezależne od języka); w przeciwnym razie cofa się do klasyfikacji tylko na podstawie kodu statusu. Sterowniki potrzebujące precyzyjnego zachowania od niejednoznacznych statusów (403, 404, 400) **muszą** wypełnić `code`.

### Klasyfikacja heurystyczna (`llm_errors.rs`)

`classify_error(message, status)` zwraca `ClassifiedError` z kategorią, możliwość ponowienia, flagą rozliczeń, sugerowanym opóźnieniem, zsanityzowaną wiadomością, surową wiadomością i opcjonalnym kontekstem dostawcy/modelu. `classify_error_with_context` wzbogaca wynik o metadane dostawcy/modelu i akcjęwną `suggestion`.

Priorytet klasyfikacji (od najbardziej specyficznego):

1. **Szybkie ścieżki kodu statusu** — 429 → RateLimit, 402 → Billing, 401 → Auth, 404 → ModelNotFound. Przypadek 403 jest specjalny: sprawdza wzorce rate-limit, rozliczeń, przepełnienia kontekstu, model-nieznaleziony, a następnie `FORBIDDEN_NON_AUTH_PATTERNS` (które kieruje nie-autorskie 403 do ogólnej rurociągu zamiast błędnej klasyfikacji jako błędy autoryzacji — ważne dla chińskich dostawców, które zwracają 403 dla problemów z kwotą/regionem/uprawnieniami modelu).
2. **Dopasowywanie wzorców** w kolejności priorytetu: ContextOverflow → Billing → Auth → RateLimit → ModelNotFound → Format → Overloaded → Timeout. Wzorce to niewrażliwe na wielkość liter sprawdzanie podłańcuchów względem opracowanych tabel; bez zależności od regex.
3. **Wykrywanie stron błędów HTML** (`is_html_error_page`) — wyłapuje odpowiedzi Cloudflare 521–530 maskujące się jako JSON; klasyfikowane jako Overloaded.
4. **Rezerwa** — 5xx → Overloaded, 4xx → Format, tekst brzmiący jak sieciowy → Timeout, w przeciwnym razie Format.

Kluczowe niezmienniki poprawności przypinane przez testy:

- `insufficient_quota` na 403 klasyfikuje jako **Billing**, nie Format. To jednocześnie zgłasza właściwą rzecz operatorom i uruchamia długą przerwę rozliczeń, aby konto bez środków nie było ponawiane w nieskończoność.
- `ssl handshake failure` jest celowo **wykluczone** z przejściowych wzorców SSL — błędy handshake to błędy konfiguracji, które będą zawodzić identycznie przy ponowieniu.
- `FORBIDDEN_NON_AUTH_PATTERNS` celowo nadpisuje ogólny 403 → Auth, gdy ciało wspomina o kwocie, regionie, uprawnieniach modelu, pojemności itp.

### Sanityzacja

`sanitize_for_user(category, raw)` tworzy bezpieczne dla użytkownika wiadomości, ekstrahując pole JSON `.error.message` / `.message` / `.detail` gdy obecne, redagując wszystko, co wygląda jak sekret (`sk-…`, `key-…`, `Bearer …`), usuwając otokę `LLM driver error: API error (NNN):` i ograniczając długość do 200 znaków (300 dla ostatecznej wiadomości dla użytkownika). `cap_message` cofa się do najbliższej granicy znaku UTF-8, aby uniknąć paniki na wejściu CJK/emoji.

`extract_retry_delay` parsuje `retry after N`, `retry-after: N`, `try again in N` z tekstu błędu i zwraca milisekundy (przyrostek `ms` jest honorowany; w przeciwnym razie sekundy są konwertowane).

`is_transient(message)` to szybka heurystyka używana przez wywołujących, którzy nie potrzebują pełnej klasyfikacji — wykonywa operację OR na tabelach wzorców timeout, overloaded, rate-limit i transient-SSL.

## Rejestr wyczerpania dostawców

`ProviderExhaustionStore` to rejestr w pamięci konsultowany przez łańcuchy awaryjne przed każdym wysłaniem i aktualizowany przez warstwę pomiarową, gdy limit budżetu ustawiony przez operatora się uruchamia. Istnieje w tej krate (nie w krate sterowników), aby błąd trait-poziomu `AllProvidersExhausted` i implementacja łańcucha dzieliły jeden typ bez cyklicznego importu.

### Semantyka

- **Lokalny dla procesu.** Restart demona celowo czyści cały stan — utrzymywanie wyczerpania przez restarty groziłoby zablokowaniem gniazda, którego podstawowy problem (rotacja kluczy, doładowanie rozliczeń) został naprawiony poza pasmem.
- **Automatyczne czyszczenie przy odczycie.** `is_exhausted` zwraca `None` i atomowo usuwa wpis po przekroczeniu `until`, więc łańcuch naturalnie ponawia gniazdo bez zewnętrznego narzędzia czyszczącego. Usunięcie używa `remove_if`, więc współbieżne `mark_exhausted` z nowym `until` nigdy nie jest nadpisywane.
- **Wpisy nieograniczone w czasie.** `until: None` parkuje gniazdo, aż operator jawnie je wyczyści. W praktyce każdy wywołujący przekazuje `Some(_)`.
- **Zastępowanie przy oznaczeniu.** Oznaczenie tego samego dostawcy dwa razy zastępuje poprzedni wpis — najnowszy powód jest tym akcyjnym.

### Warianty `ExhaustionReason`

`RateLimited`, `QuotaExceeded`, `BudgetExceeded`, `AuthFailed`. Warianty niczego tu nie napędzają — są rejestrowane dla logów/metryk/uwidocznionych szczegółów błędu. Łańcuch awaryjny traktuje każdy wariant identycznie: pomija aż `until` minie. Każdy wariant ujawnia `as_metric_label()` zwracając stabilny łańcuch kebab-case odpowiedni dla tagów Prometheus.

Powody bez wskazówki resetu raportowanej przez serwer (`QuotaExceeded`, `BudgetExceeded`, `AuthFailed`) używają `DEFAULT_LONG_BACKOFF` (1 godzina) — wystarczająco krótko, aby poprawki operatora automatycznie leczyły łańcuch, wystarczająco długo, aby łańcuch nie marnował próby co minutę.

### Gwarancja determinizmu

Kolejność iteracji `DashMap` jest niedeterministyczna. `snapshot()` zwraca wiersze posortowane rosnąco po `provider_id` (przez `BTreeMap`), aby jakikolwiek zstringowany wynik — wiadomości błędu, logi, tekst wyczerpania włączony do prompta — był bajtowo identyczny między procesami. To zachowuje determinizm pamięci podręcznej promptów (#3298) nawet gdy dane wyczerpania wyciekają do prompta. `live_count()` i `record_skip()` mają charakter wyłącznie obserwacyjny i nigdy nie wpływają na routing.

### Logowanie

`mark_exhausted` i `record_skip` emitują zdarzenia `tracing::info!` z `target: "metering"`. Zdarzenia wyczerpania to sygnał akcjonowalny dla operatora, nie szum debugowania — target pozwala istniejącym filtrom tracing-subscriber kierować zdarzenia pomiarowe do dashboardu bez dodatkowego okablowania.

## `DriverConfig`

Konfiguracja niezależna od dostawcy przekazywana do fabryki sterownika. Dwie właściwości bezpieczeństwa są przypinane przez testy i nie mogą ulec regresji:

- `api_key` i `proxy_url` używają `#[serde(skip_serializing)]`. Jakakolwiek operacja `serde_json::to_*` / `toml::to_*` na `DriverConfig` (zrzut pamięci podręcznej, migawka diagnostyczna, `mcp_config.json`, śledzenie międzyprocesowe) nie może emitować tych pól w czystym tekście. `Deserialize` pozostaje bez zmian — pliki konfiguracyjne nadal je wypełniają przy ładowaniu.
- Ręczna implementacja `Debug` redaguje `api_key`, `proxy_url`, `vertex_ai.credentials_path` jako `<redacted>`.

Inne godne uwagi pola:

- `skip_permissions` domyślnie wynosi `true`, ponieważ LibreFang działa jako daemon bez interaktywnego terminala; monity uprawnień blokowałyby na czas nieokreślony. Własna warstwa zdolności/RBAC LibreFanga jest prawdziwą granicą.
- `max_retries` domyślnie wynosi 3 (cztery próby łącznie). Ustaw na 0, aby wyłączyć ponowienia wewnątrz sterownika i polegać wyłącznie na `FallbackChain`. Dostawcy opartych na CLI ignorują to pole.
- `message_timeout_secs` jest oparty na nieaktywności dla dostawców CLI — podproces jest zabijany po tej liczbie sekund ciszy na stdout, nie po czasie zegarowym.
- `emit_caller_trace_headers` pozwala regulowanym najemcom tłumić nagłówki `x-librefang-{agent,session,step}-id` po stronie przewodowej niezależnie od tego, czy żądanie niesie pola caller-id. Obecnie honorowane tylko przez sterownik kompatybilny z OpenAI.
- `mcp_bridge` niesie `McpBridgeConfig`, więc `DriverConfig` może trzymać okablowanie mostu CLI bez cyklicznej zależności od `librefang-llm-drivers`. Pole to `#[serde(skip)]` — jest wypełniane tylko przez jądro w momencie konstrukcji sterownika.

## Ograniczenia (Tabu)

To są twarde zasady, nie preferencje:

- **Żadnego `reqwest`, żadnych zależności TLS, żadnych dostarczonych SDK klienta.** Czysty trait + typy. Cały powód istnienia tej kraty jako oddzielonej od `librefang-llm-drivers` to utrzymanie lekkich zależności dla buildów testowych.
- **Żadnego importu `librefang-llm-drivers`.** Cykliczność.
- **Żadnych importów `librefang-runtime` / `librefang-kernel`.** Trait sterownika musi istnieć samodzielnie.
- **Żadnych nowych wariantów błędów z typem `String`** w `LlmError`. Użyj uporządkowanego pola enum.
- **Żadnego `Box<dyn Error>` w typach zwrotnych traitów.** Używaj `LlmError`.
- **Nie łącz tej kraty z `librefang-llm-drivers`** „dla prostoty." Krate testowe zależą od traitu samodzielnie, właśnie po to, aby uniknąć ciągnięcia zależności HTTP/TLS.

## Dodawanie nowego sterownika

Nowe sterowniki trafiają do `librefang-llm-drivers`, **nie tutaj**. Implementacja `LlmDriver` nie powinna wymagać dotykania tej kraty, chyba że jeden z poniższych warunków jest prawdziwie spełniony:

- Potrzebna jest nowa metoda w traicie — rzadkie; najpierw omów w issue.
- Potrzebny jest nowy wariant błędu w `LlmError`. Dodaj go jako typowany wariant i zachowaj łańcuch `source()` (#3745).
- Nowy wspólny typ po stronie sterownika jest prawdziwie potrzebny przez wielu dostawców.

## Testowanie

- Zgodność z traitem jest sprawdzana przez mock sterowniki w `librefang-testing` (`MockKernelBuilder`).
- **Nie dodawaj tutaj testów fixture HTTP.** One należą do `librefang-llm-drivers` obok testowanej implementacji.
- Testy jednostkowe w tej krate obejmują: priorytet klasyfikacji błędów i przypadki brzegowe (insufficient_quota, obcinanie CJK, wykluczenie transient SSL), semantykę rejestru wyczerpania (auto-czyszczenie, zastępowanie przy oznaczeniu, porządkowanie snapshot, klonowanie współdzieli stan), zachowanie łańcucha `LlmError` source, redagowanie sekretów `DriverConfig` zarówno w `Debug` jak i `Serialize`, domyślne zachowanie `stream()` włącznie z propagacją błędu rozłączenia odbiorcy, oraz serde round-tripping `LlmFamily`.
