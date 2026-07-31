# crates — librefang-runtime-media

# librefang-runtime-media

Niezależny od dostawcy moduł generowania i rozumienia mediów dla LibreFang. Implementuje syntezę tekstu na mowę, generowanie obrazów/wideo/muzyki, rozpoznawanie mowy na tekst oraz opisywanie obrazów za jednolitym interfejsem trait, z leniwie buforowanymi sterownikami i konfigurowalnym trasowaniem do dostawców.

## Architektura

Kratek zawiera dwa niezależne podsystemy:

1. **Generowanie mediów** — wychodząca synteza/generowanie nowych mediów (TTS, obrazy, wideo, muzyka) przez trait `MediaDriver` i `MediaDriverCache`.
2. **Rozumienie mediów** — wchodząca analiza istniejących mediów (opisywanie obrazów, transkrypcja audio) przez strukturę `MediaEngine` w `media_understanding`.

Oba podsystemy współdzielą tego samego klienta HTTP (`librefang_http::proxied_client`), dyscyplinę obsługi błędów oraz konwencje wykrywania dostawców, ale poza tym są rozdzielone.

```mermaid
graph TD
    subgraph Generation
        Cache["MediaDriverCache"]
        Driver["MediaDriver trait"]
        EL["ElevenLabsMediaDriver<br/>(TTS)"]
        GEM["GeminiMediaDriver<br/>(Image)"]
        GTTS["GoogleTtsMediaDriver<br/>(TTS)"]
        MM["MiniMaxMediaDriver"]
        OA["OpenAIMediaDriver"]
        Cache --> Driver
        Driver --> EL
        Driver --> GEM
        Driver --> GTTS
        Driver --> MM
        Driver --> OA
    end
    subgraph Understanding
        Engine["MediaEngine"]
        Engine --> Vision["Vision LLMs<br/>(Anthropic/OpenAI/Groq/Gemini)"]
        Engine --> STT["Speech-to-Text<br/>(Groq/OpenAI/Gemini/ElevenLabs/...node[")"]"]
        Engine --> FFMPEG["ffmpeg<br/>(.oga / video preprocessing)"]
    end
```

## Trait `MediaDriver`

Każdy dostawca implementuje trait `MediaDriver`, deklarując wspierane możliwości przez `capabilities()`. Metody dla niewspieranych modalności mają domyślne implementacje zwracające `MediaError::NotSupported`, więc dostawca nadpisuje tylko to, co faktycznie potrafi.

```rust
#[async_trait]
pub trait MediaDriver: Send + Sync {
    fn capabilities(&self) -> Vec<MediaCapability>;
    fn is_configured(&self) -> bool { true }
    fn provider_name(&self) -> &str;

    async fn generate_image(&self, _: &MediaImageRequest) -> Result<MediaImageResult, MediaError>;
    async fn synthesize_speech(&self, _: &MediaTtsRequest) -> Result<MediaTtsResult, MediaError>;
    async fn submit_video(&self, _: &MediaVideoRequest) -> Result<MediaVideoSubmitResult, MediaError>;
    async fn poll_video(&self, _: &str) -> Result<MediaTaskStatus, MediaError>;
    async fn get_video_result(&self, _: &str) -> Result<MediaVideoResult, MediaError>;
    async fn generate_music(&self, _: &MediaMusicRequest) -> Result<MediaMusicResult, MediaError>;
}
```

Generowanie wideo jest asynchroniczne: `submit_video` zwraca identyfikator zadania, a następnie wywołujący odpytują przez `poll_video` i pobierają ostateczny wynik przez `get_video_result`. Generowanie obrazów, TTS i muzyki to synchroniczne metody jedno wywołanie.

## `MediaDriverCache` — Leniwa instancjonacja sterowników

Sterowniki są tworzone przy pierwszym użyciu i buforowane jako `Arc<dyn MediaDriver>` w `DashMap`. Klucz bufora to `"{provider}|{base_url}"`, więc ten sam dostawca skonfigurowany z różnymi bazowymi URL-ami daje oddzielne instancje sterowników.

### Priorytet rozwiązywania URL

Gdy `get_or_create` jest wywołane z `base_url: None`, bufor rozwiązuje URL w następującej kolejności:

1. **Jawny argument `base_url`** przekazany do `get_or_create` — najwyższy priorytet.
2. **Mapa `provider_urls`** — wypełniana z `[provider_urls]` w konfiguracji, sprawdzana po nazwie dostawcy i kanonicznym aliasie.
3. **Zahardkodowana domyślna wartość sterownika** — najniższy priorytet.

Aliasy dostawców są rozwiązywane przez `canonical_provider_name` (obecnie `"google"` → `"gemini"`), więc wpis `provider_urls` pod `gemini` jest znajdowany, gdy wywołujący pyta o `google`.

### Rejestr dostawców

`load_providers_from_registry` przyjmuje wycinek `ProviderInfo` z katalogu modeli. Dostawcy z niepustymi `media_capabilities` są dodawani do listy preferencji w kolejności rejestru, a wbudowani dostawcy (`openai`, `gemini`, `elevenlabs`, `minimax`, `google_tts`) są dołączani jako rezerwa. Ta lista napędza `detect_for_capability`, które zwraca pierwszego skonfigurowanego dostawcę wspierającego daną możliwość.

### Nieznani dostawcy

Jeśli nazwa dostawcy nie jest rozpoznana, ale skonfigurowany jest `base_url` (jawnie lub przez `provider_urls`), bufor przechodzi na `GenericOpenAICompatMediaDriver` — sterownik kompatybilny z OpenAI, sparametryzowany nazwą dostawcy i URL-em. Klucz API jest odczytywany ze zmiennej `{PROVIDER_UPPER}_API_KEY`. Bez `base_url` żądanie kończy się błędem `InvalidRequest`, kierującym operatora do `config.toml`.

### Hot-Reload

`update_provider_urls` zastępuje mapę URL-i i czyści bufor, więc kolejne wywołania `get_or_create` tworzą nowe sterowniki z nowymi URL-ami. `clear` czyści bufor bez dotykania konfiguracji URL-i.

## Implementacje dostawców

### ElevenLabs (`elevenlabs.rs`)

**Możliwość:** Synteza tekstu na mowę przez `POST /v1/text-to-speech/{voice_id}`.

**Uwierzytelnienie:** Zmienna środowiskowa `ELEVENLABS_API_KEY`.

**Wartości domyślne:**
- Model: `eleven_multilingual_v2`
- Głos: `21m00Tcm4TlvDq8ikWAM` (Rachel)
- Format: `opus_48000_32` (wybrany tak, aby wiadomości głosowe WhatsApp PTT działały bez transkodowania; wywołujący potrzebujący MP3 przekazują `format: Some("mp3_44100_128")`)

**Walidacja identyfikatora głosu:** `voice_id` jest interpolowany bezpośrednio w ścieżkę URL, więc `validate_voice_id` wymusza ścisły format: dokładnie 20 znaków alfanumerycznych ASCII. Zamyka to wektory path-traversal i wstrzykiwania query-string. Walidator działa *przed* sprawdzeniem klucza API, więc gdy oba są błędne, agent widzi działający błąd `InvalidRequest`, a nie ogólny `MissingKey`.

Echa błędów z nieprawidłowych identyfikatorów głosu są ograniczane do `VOICE_ID_ERROR_ECHO_MAX_BYTES` (64 bajty), aby złośliwe wejście o rozmiarze 10 KB nie wyprodukowało 10 KB ciągu błędu.

**Limity odpowiedzi:** Maksymalnie 25 MB (`MAX_AUDIO_RESPONSE_BYTES`), sprawdzane zarówno przez nagłówek `Content-Length`, jak i rzeczywistą liczbę bajtów.

### Gemini (`gemini.rs`)

**Możliwość:** Generowanie obrazów przez Imagen 3 (`POST /v1beta/models/{model}:predict`).

**Uwierzytelnienie:** `GEMINI_API_KEY` lub `GOOGLE_API_KEY`, przekazywane jako parametr zapytania `?key=`.

**Wartości domyślne:**
- Model: `imagen-3.0-generate-002`
- Proporcje: `1:1`, `3:4`, `4:3`, `9:16`, `16:9`

Zwraca obrazy zakodowane w base64. Jeśli wszystkie predykcje są puste, sprawdza `raiFilteredReason` (filtr treści) i zwraca `MediaError::ContentFiltered`. Indywidualne obrazy przekraczające 10 MB są pomijane z ostrzeżeniem `warn!`.

### Google Cloud TTS (`google_tts.rs`)

**Możliwość:** Synteza tekstu na mowę przez `POST /v1/text:synthesize`.

**Uwierzytelnienie:** `GOOGLE_API_KEY` lub `GOOGLE_CLOUD_API_KEY`, przekazywane jako parametr zapytania `?key=`.

**Wartości domyślne:**
- Głos: `en-US-Standard-F`
- Język: `en-US`
- Tempo mówienia: `1.0`

**Obsługa SSML:** `build_input` wykrywa znaczniki SSML i odpowiednio trasuje:
- Tekst zawierający `<speak>` jest wysyłany bez zmian w polu `ssml`.
- Tekst zawierający znaczniki SSML (np. `<break>`, `<prosody>`) bez otoki `<speak>` jest automatycznie owijany.
- Cały pozostały tekst jest wysyłany jako zwykły `text`.

Detektor `is_ssml` używa jednoznacznych znaczników specyficznych dla SSML (`<prosody>`, `<emphasis>`, `<say-as>`, `<phoneme>`, `<par>`, `<seq>`) i wymaga atrybutów specyficznych dla SSML na niejednoznacznych znacznikach (`<sub alias="...">`, `<mark name="...">`, `<audio src="...">`), aby uniknąć fałszywych alarmów na zwykłym HTML.

**Mapowanie kodowania audio** (`map_audio_encoding`):

| Żądany format | Kodowanie Google |
|---|---|
| `opus`, `ogg` | `OGG_OPUS` |
| `wav`, `pcm`, `linear16` | `LINEAR16` |
| wszystko inne | `MP3` |

### MiniMax i OpenAI

Źródło tych modułów jest obcięte w tym zrzucie. Z rejestru bufora wynika, że `MiniMaxMediaDriver` wspiera generowanie wideo (z pomocnikiem `check_base_resp` dla niestandardowej otoki odpowiedzi API MiniMax), a `OpenAIMediaDriver` wspiera generowanie obrazów i TTS. Oba mają również `GenericOpenAICompatMediaDriver` dla zdefiniowanych przez użytkownika punktów końcowych kompatybilnych z OpenAI.

## `MediaError`

Wszystkie awarie przepływają przez pojedynczą enumerację błędów:

| Wariant | Znaczenie |
|---|---|
| `NotSupported` | Sterownik nie implementuje tej modalności. |
| `MissingKey` | Wymagana zmienna środowiskowa klucza API nie jest ustawiona. |
| `Http` | Błąd sieci lub transportu. |
| `Api` | Dostawca zwrócił status spoza 2xx; zawiera `status` i skróconą `message`. |
| `RateLimit` | Dostawca ograniczył żądanie (rate-limit). |
| `ContentFiltered` | Filtr bezpieczeństwa odrzucił żądanie. |
| `InvalidRequest` | Parametry dostarczone przez wywołującego nie przeszły walidacji. |
| `TaskNotFound` | Asynchroniczne ID zadania (wideo) nie rozpoznane. |
| `Other` | Kategoria zbiorcza dla limitów rozmiaru, błędów parsowania itp. |

Komunikaty błędów z odpowiedzi API dostawców są skracane do 500 bajtów przez `safe_truncate_str` (pomocnik obcinający z zachowaniem granic UTF-8) przed osadzeniem w `MediaError`.

## Rozumienie mediów (`media_understanding.rs`)

`MediaEngine` obsługuje *wchodzącą* analizę mediów — opisywanie obrazów i transkrypcję audio. W przeciwieństwie do podsystemu generowania, rozumienie używa **wysyłania do jednego dostawcy bez kaskady rezerw**: wybiera jednego dostawcę i bezpośrednio przekazuje jego błąd w przypadku niepowodzenia.

### Wybór dostawcy

Dla każdej modalności silnik rozwiązuje dostawcę w dwóch krokach:

1. Jeśli `[media] image_provider` / `audio_provider` jest jawnie ustawiony w konfiguracji, użyj go.
2. W przeciwnym razie — autodetekcja przez sprawdzenie, która zmienna środowiskowa klucza API jest obecna.

Autodetekcja loguje `warn!` (audio) lub `debug!` (obraz) z zaleceniem jawnej konfiguracji dla powtarzalności.

**Priorytet dostawców wizyjnych:** Anthropic → OpenAI → Groq → Gemini

**Priorytet dostawców audio (STT):** Groq → OpenAI → Gemini → ElevenLabs → MiniMax → Fireworks → Together → SiliconFlow

### Rozwiązywanie modelu

Wybór modelu następuje w trzech poziomach priorytetu:

1. `[media] image_model` / `audio_model` — nadpisanie przez operatora per modalność.
2. `[media.custom_stt] model` — tylko dla dostawców STT niestandardowych/samodzielnych. `custom_stt_model_ref` jawnie zwraca `None` dla każdej wbudowanej nazwy dostawcy, więc to ustawienie nie może przypadkowo nadpisać domyślnych wartości wbudowanego dostawcy.
3. Zahardkodowana wartość domyślna dostawcy z `default_vision_model` / `default_audio_model`.

### Kontrola współbieżności

`MediaEngine` przechowuje `tokio::sync::Semaphore` z maksymalną wartością `max_concurrency` (ograniczonym do 1–8). `process_attachments` uruchamia jedno zadanie na załącznik, z każdym nabywającym pozwolenie. Flagi konfiguracyjne per modalność (`image_description`, `audio_transcription`, `video_description`) decydują, które załączniki są w ogóle przetwarzane.

### Opisywanie obrazów

`describe_image` odczytuje bajty obrazu ze źródła `MediaSource::FilePath` lub `MediaSource::Base64` (źródła URL są odrzucane — wywołujący muszą pobrać najpierw), koduje je w base64 i wysyła do specyficznej dla dostawcy funkcji wizyjnej:

- **Anthropic** — blok treści `image` ze źródłem base64 przez API Messages.
- **OpenAI / Groq** — blok treści `image_url` z URL-em danych przez Chat Completions.
- **Gemini** — część `inline_data` przez `generateContent`.

Każda funkcja dostawcy wyodrębnia odpowiedź tekstową ze swojego specyficznego kształtu JSON.

### Transkrypcja audio

`transcribe_audio` obsługuje bardziej złożoną rurociąg:

1. **Odczyt bajtów** ze źródła `FilePath` lub `Base64`.
2. **Pochodzenie rozszerzenia pliku** z typu MIME lub ścieżki źródła.
3. **Kontenery wideo** (`MediaType::Video`): wyodrębnienie ścieżki audio przez `extract_video_audio_track`, które reenkoduje do Ogg/Opus używając ffmpeg (`-c:a libopus`, mono, 32 kbps, 48 kHz). Strumień wideo jest odrzucany.
4. **Pliki Telegram `.oga`**: repaketyzacja do `.ogg` przez `transcode_oga_to_ogg_opus` używając `-c:a copy` ffmpeg (ładunek jest już Opus; różni się tylko otoka kontenera).
5. **Wysłanie** do wybranego dostawcy STT.

**Ramię wysyłania STT:**

| Dostawca | Protokół |
|---|---|
| Groq, OpenAI, MiniMax, Fireworks, Together, SiliconFlow | Multipart OpenAI Whisper (`whisper_transcribe`) |
| Gemini | Multimodalny `generateContent` z `inline_data` |
| ElevenLabs | API Speech-to-Text (`/v1/speech-to-text`) |
| Custom / samodzielny | Multipart OpenAI Whisper przez `custom_stt_config` |

`whisper_transcribe` wysyła pola formularza `language` i `prompt` tylko wtedy, gdy są ustawione (z argumentu wywołania lub domyślnych wartości konfiguracji `[media]`), produkując bajtowo identyczne żądania do zachowania sprzed dodania funkcji, gdy żadne nie jest skonfigurowane. Niestandardowe punkty końcowe z `key_required = false` pomijają nagłówek `Authorization` całkowicie zamiast wysyłać pusty token bearer.

### Integracja z ffmpeg

Całe użycie ffmpeg przepływa przez `run_ffmpeg_pipe`, współdzielony pomocnik, który:

- Karmi wejście przez stdin, zbiera stdout — bez plików tymczasowych na dysku.
- Uruchamia zapisy stdin i odczyty stdout/stderr jako współbieżne zadania, aby uniknąć zakleszczeń bufora rur.
- Wymusza 30-sekundowy limit czasu zegara ściennego, zabijając i zbierając proces potomny po upływie.
- Zgłasza czytelny dla człowieka komunikat błędu "zainstaluj ffmpeg" sparametryzowany przez `install_hint`.

### Obserwowalność

`record_media_understanding_failure` emituje licznik `librefang_media_understanding_failures_total` z etykietami `kind` (obraz/audio), `provider` i `model`, plus strukturalny `warn!`. To jest sygnał detekcji dla cicho wygaszanych modeli hostowanych przez dostawców — awaria pojawia się jako mierzalna metryka zamiast degradować do bezpośredniego przekazywania mediów tylko z wpisem w logach. Licznik jest emitowany w punkcie awarii wewnątrz silnika, niezależnie od tego, który wywołujący ją wywołał.

Kardynalność jest ograniczona: `kind` ma dwie wartości, a `provider`/`model` są pobierane ze skonfigurowanego lub wbudowanego zestawu domyślnego, zgodnie z dyscypliną etykiet innych liczników LibreFang.

### Sanityzacja błędów

Błędy dostawców zwracane do podpowiedzi agenta (przez `Err(String)`) są sanityzowane tak, aby zawierały tylko kod statusu HTTP, nie treść odpowiedzi. Pełne treści odpowiedzi są zachowywane w `tracing::warn!` do diagnostyki przez operatora. Zapobiega to wyciekowi kluczy API lub wewnętrznych elementów żądań z odpowiedzi błędów dostawców do kontekstu LLM. URL-e Gemini zawierają klucz API jako `?key=…`, więc wyświetlenia błędów reqwest nigdy nie są przekazywane wywołującemu.

## Integracja z kratekiem nadrzędnym

`librefang-runtime` reeksportuje ten kratek pod jego historycznymi ścieżkami (`runtime::media`, `runtime::media_understanding`) za flagą funkcji `media` domyślnie włączonej, więc podrzędne punkty wywołań nie wymagają zmian importów. Kratek został wydzielony z `librefang-runtime` jako część podziału god-crate #3710.

Kratek typów `librefang-types` dostarcza wszystkie współdzielone typy żądań/odpowiedzi (`MediaTtsRequest`, `MediaImageRequest`, `MediaConfig`, `MediaAttachment`, `MediaCapability` itd.), a `librefang-http` dostarcza klienta HTTP świadomego proxy, używanego przez wszystkich dostawców.
