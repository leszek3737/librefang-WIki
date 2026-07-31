# crates — librefang-runtime-media

# librefang-runtime-media

Provider-agnostic media generation and understanding for LibreFang. Implements text-to-speech, image/video/music generation, speech-to-text, and image description behind a uniform trait interface, with lazy-cached drivers and configurable provider routing.

## Architecture

The crate has two independent subsystems:

1. **Media generation** — outbound synthesis/generation of new media (TTS, images, video, music) via the `MediaDriver` trait and `MediaDriverCache`.
2. **Media understanding** — inbound analysis of existing media (image description, audio transcription) via the `MediaEngine` struct in `media_understanding`.

Both subsystems share the same HTTP client (`librefang_http::proxied_client`), error discipline, and provider-detection conventions, but are otherwise decoupled.

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

## The `MediaDriver` Trait

Every provider implements `MediaDriver`, declaring which capabilities it supports via `capabilities()`. Methods for unsupported modalities have default implementations that return `MediaError::NotSupported`, so a provider only overrides what it can actually do.

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

Video generation is asynchronous: `submit_video` returns a task ID, then callers poll with `poll_video` and retrieve the final result with `get_video_result`. Image, TTS, and music are synchronous single-call methods.

## `MediaDriverCache` — Lazy Driver Instantiation

Drivers are created on first use and cached as `Arc<dyn MediaDriver>` in a `DashMap`. The cache key is `"{provider}|{base_url}"`, so the same provider configured with different base URLs yields separate driver instances.

### URL Resolution Priority

When `get_or_create` is called with `base_url: None`, the cache resolves the URL in this order:

1. **Explicit `base_url` argument** passed to `get_or_create` — highest priority.
2. **`provider_urls` map** — populated from `[provider_urls]` in config, checked by provider name and canonical alias.
3. **Driver's hardcoded default** — lowest priority.

Provider aliases are resolved by `canonical_provider_name` (currently `"google"` → `"gemini"`), so a `provider_urls` entry under `gemini` is found when a caller asks for `google`.

### Provider Registry

`load_providers_from_registry` accepts a slice of `ProviderInfo` from the model catalog. Providers with non-empty `media_capabilities` are added to the preference list in registry order, with built-in providers (`openai`, `gemini`, `elevenlabs`, `minimax`, `google_tts`) appended as fallback. This list drives `detect_for_capability`, which returns the first configured provider supporting a given capability.

### Unknown Providers

If a provider name is not recognized but a `base_url` is configured (either explicitly or via `provider_urls`), the cache falls back to `GenericOpenAICompatMediaDriver` — an OpenAI-compatible driver parameterized by provider name and URL. The API key is read from `{PROVIDER_UPPER}_API_KEY`. Without a `base_url`, the request fails with an `InvalidRequest` error guiding the operator to `config.toml`.

### Hot-Reload

`update_provider_urls` replaces the URL map and clears the cache, so subsequent `get_or_create` calls build fresh drivers with the new URLs. `clear` empties the cache without touching URL configuration.

## Provider Implementations

### ElevenLabs (`elevenlabs.rs`)

**Capability:** Text-to-speech via `POST /v1/text-to-speech/{voice_id}`.

**Auth:** `ELEVENLABS_API_KEY` environment variable.

**Defaults:**
- Model: `eleven_multilingual_v2`
- Voice: `21m00Tcm4TlvDq8ikWAM` (Rachel)
- Format: `opus_48000_32` (chosen so WhatsApp PTT voice notes work without transcoding; callers needing MP3 pass `format: Some("mp3_44100_128")`)

**Voice ID validation:** The `voice_id` is interpolated directly into the URL path, so `validate_voice_id` enforces a strict shape: exactly 20 ASCII-alphanumeric characters. This closes path-traversal and query-string injection vectors. The validator runs *before* the API key check, so when both are wrong the agent sees the actionable `InvalidRequest` error rather than a generic `MissingKey`.

Error echoes from invalid voice IDs are capped at `VOICE_ID_ERROR_ECHO_MAX_BYTES` (64 bytes) so a 10 KB malicious input doesn't produce a 10 KB error string.

**Response limits:** 25 MB max (`MAX_AUDIO_RESPONSE_BYTES`), checked both via `Content-Length` header and actual byte count.

### Gemini (`gemini.rs`)

**Capability:** Image generation via Imagen 3 (`POST /v1beta/models/{model}:predict`).

**Auth:** `GEMINI_API_KEY` or `GOOGLE_API_KEY`, passed as `?key=` query parameter.

**Defaults:**
- Model: `imagen-3.0-generate-002`
- Aspect ratios: `1:1`, `3:4`, `4:3`, `9:16`, `16:9`

Returns base64-encoded images. If all predictions are empty, checks for `raiFilteredReason` (content filter) and returns `MediaError::ContentFiltered`. Individual images exceeding 10 MB are skipped with a `warn!`.

### Google Cloud TTS (`google_tts.rs`)

**Capability:** Text-to-speech via `POST /v1/text:synthesize`.

**Auth:** `GOOGLE_API_KEY` or `GOOGLE_CLOUD_API_KEY`, passed as `?key=` query parameter.

**Defaults:**
- Voice: `en-US-Standard-F`
- Language: `en-US`
- Speaking rate: `1.0`

**SSML handling:** `build_input` detects SSML markup and routes accordingly:
- Text containing `<speak>` is sent as-is under the `ssml` field.
- Text containing SSML tags (e.g. `<break>`, `<prosody>`) without a `<speak>` wrapper is auto-wrapped.
- All other text is sent as plain `text`.

The `is_ssml` detector uses unambiguous SSML-only tags (`<prosody>`, `<emphasis>`, `<say-as>`, `<phoneme>`, `<par>`, `<seq>`) and requires SSML-specific attributes on ambiguous tags (`<sub alias="...">`, `<mark name="...">`, `<audio src="...">`) to avoid false positives on ordinary HTML.

**Audio encoding mapping** (`map_audio_encoding`):

| Requested format | Google encoding |
|---|---|
| `opus`, `ogg` | `OGG_OPUS` |
| `wav`, `pcm`, `linear16` | `LINEAR16` |
| anything else | `MP3` |

### MiniMax and OpenAI

Source for these modules is truncated in this snapshot. From the cache registry, `MiniMaxMediaDriver` supports video generation (with a `check_base_resp` helper for the MiniMax API's non-standard response envelope), and `OpenAIMediaDriver` supports image generation and TTS. Both also have `GenericOpenAICompatMediaDriver` for user-defined OpenAI-compatible endpoints.

## `MediaError`

All failures flow through a single error enum:

| Variant | Meaning |
|---|---|
| `NotSupported` | The driver doesn't implement this modality. |
| `MissingKey` | Required API key environment variable is not set. |
| `Http` | Network or transport error. |
| `Api` | Provider returned non-2xx; carries `status` and truncated `message`. |
| `RateLimit` | Provider rate-limited the request. |
| `ContentFiltered` | Safety filter rejected the request. |
| `InvalidRequest` | Caller-supplied parameters failed validation. |
| `TaskNotFound` | Async task ID (video) not recognized. |
| `Other` | Catch-all for size limits, parse failures, etc. |

Error messages from provider API responses are truncated to 500 bytes via `safe_truncate_str` (a UTF-8-boundary-safe truncation helper) before being embedded in `MediaError`.

## Media Understanding (`media_understanding.rs`)

The `MediaEngine` handles *inbound* media analysis — describing images and transcribing audio. Unlike the generation subsystem, understanding uses **single-provider dispatch with no fallback cascade**: it picks one provider and surfaces its error directly if it fails.

### Provider Selection

For each modality, the engine resolves a provider in two steps:

1. If `[media] image_provider` / `audio_provider` is explicitly set in config, use it.
2. Otherwise, auto-detect by checking which API key environment variable is present.

Auto-detection logs a `warn!` (audio) or `debug!` (image) recommending explicit configuration for reproducibility.

**Vision provider priority:** Anthropic → OpenAI → Groq → Gemini

**Audio (STT) provider priority:** Groq → OpenAI → Gemini → ElevenLabs → MiniMax → Fireworks → Together → SiliconFlow

### Model Resolution

Model selection follows a three-tier precedence:

1. `[media] image_model` / `audio_model` — operator-set per-modality override.
2. `[media.custom_stt] model` — for custom/self-hosted STT providers *only*. `custom_stt_model_ref` explicitly returns `None` for every built-in provider name, so this setting cannot accidentally override a built-in provider's default.
3. Hardcoded provider default from `default_vision_model` / `default_audio_model`.

### Concurrency Control

`MediaEngine` holds a `tokio::sync::Semaphore` capped at `max_concurrency` (clamped to 1–8). `process_attachments` spawns one task per attachment, each acquiring a permit. Per-modality config flags (`image_description`, `audio_transcription`, `video_description`) gate which attachments are processed at all.

### Image Description

`describe_image` reads image bytes from `MediaSource::FilePath` or `MediaSource::Base64` (URL sources are rejected — callers must download first), base64-encodes them, and dispatches to the provider-specific vision function:

- **Anthropic** — `image` content block with base64 source via the Messages API.
- **OpenAI / Groq** — `image_url` content block with a data URL via Chat Completions.
- **Gemini** — `inline_data` part via `generateContent`.

Each provider function extracts the text response from its provider-specific JSON shape.

### Audio Transcription

`transcribe_audio` handles a more complex pipeline:

1. **Read bytes** from `FilePath` or `Base64` source.
2. **Derive file extension** from MIME type or source path.
3. **Video containers** (`MediaType::Video`): extract the audio track via `extract_video_audio_track`, which re-encodes to Ogg/Opus using ffmpeg (`-c:a libopus`, mono, 32 kbps, 48 kHz). The video stream is dropped.
4. **Telegram `.oga` files**: re-packetize to `.ogg` via `transcode_oga_to_ogg_opus` using ffmpeg's `-c:a copy` (the payload is already Opus; only the container wrapper differs).
5. **Dispatch** to the selected STT provider.

**STT dispatch arms:**

| Provider | Protocol |
|---|---|
| Groq, OpenAI, MiniMax, Fireworks, Together, SiliconFlow | OpenAI Whisper multipart (`whisper_transcribe`) |
| Gemini | Multimodal `generateContent` with `inline_data` |
| ElevenLabs | Speech-to-Text API (`/v1/speech-to-text`) |
| Custom / self-hosted | OpenAI Whisper multipart via `custom_stt_config` |

`whisper_transcribe` sends `language` and `prompt` form fields only when they are set (from the per-call argument or `[media]` config defaults), producing byte-identical requests to pre-feature behavior when neither is configured. Custom endpoints with `key_required = false` omit the `Authorization` header entirely rather than sending an empty bearer token.

### ffmpeg Integration

All ffmpeg usage flows through `run_ffmpeg_pipe`, a shared helper that:

- Feeds input via stdin, collects stdout — no scratch files on disk.
- Runs stdin writes and stdout/stderr reads as concurrent tasks to avoid pipe-buffer deadlocks.
- Enforces a 30-second wall-clock timeout, killing and reaping the child on expiry.
- Reports a human-readable "install ffmpeg" error message parameterized by `install_hint`.

### Observability

`record_media_understanding_failure` emits the `librefang_media_understanding_failures_total` counter with labels `kind` (image/audio), `provider`, and `model`, plus a structured `warn!`. This is the detection signal for hosted models being silently retired by providers — the failure surfaces as an actionable metric rather than degrading to raw media passthrough with only a log line. The counter is emitted at the point of failure inside the engine, regardless of which caller triggered it.

Cardinality is bounded: `kind` has two values, and `provider`/`model` are drawn from the configured or built-in default set, matching the label discipline of other LibreFang counters.

### Error Sanitization

Provider errors returned to the agent prompt (via `Err(String)`) are sanitized to carry only the HTTP status code, not the response body. Full response bodies are kept in `tracing::warn!` for operator diagnosis. This prevents provider error responses from leaking API keys or request internals into the LLM context. Gemini URLs embed the API key as `?key=…`, so reqwest error displays are never surfaced to the caller.

## Integration with the Parent Crate

`librefang-runtime` re-exports this crate at its historical paths (`runtime::media`, `runtime::media_understanding`) behind the default-on `media` feature flag, so downstream call sites require no import changes. The crate was extracted from `librefang-runtime` as part of the #3710 god-crate split.

The types crate `librefang-types` provides all shared request/response types (`MediaTtsRequest`, `MediaImageRequest`, `MediaConfig`, `MediaAttachment`, `MediaCapability`, etc.), and `librefang-http` provides the proxy-aware HTTP client used by all providers.