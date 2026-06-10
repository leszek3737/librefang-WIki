# Other — librefang-testing-src

# librefang-testing — Example Tests (`tests.rs`)

## Purpose

This file contains example tests that demonstrate how to use the `librefang-testing` test infrastructure. It serves as both a living reference for developers writing new tests and a smoke-test suite verifying that the core API endpoints and mock drivers behave correctly.

## Test Categories

### HTTP Endpoint Tests

These tests instantiate a full `TestAppState`, obtain its Axum router, and fire one-shot HTTP requests through `tower::ServiceExt::oneshot`. They validate status codes and JSON response shapes.

| Test Function | Method & Path | Asserted Behavior |
|---|---|---|
| `test_health_endpoint` | `GET /api/health` | 200, JSON contains `"status": "ok"` or `"degraded"` |
| `test_version_endpoint` | `GET /api/version` | 200, JSON contains a `version` field |
| `test_list_agents` | `GET /api/agents` | 200, JSON contains `items` (array) and `total` (u64) |
| `test_get_agent_invalid_id` | `GET /api/agents/not-a-valid-uuid` | 400, JSON contains `error` field |
| `test_get_agent_not_found` | `GET /api/agents/{random-uuid}` | 404, JSON contains `error` field |
| `test_spawn_agent_post` | `POST /api/agents` | 200 or 201, body has `manifest_toml` |
| `test_delete_nonexistent_agent_is_idempotent` | `DELETE /api/agents/{random-uuid}` | 200, JSON has `"status": "already-deleted"` |
| `test_set_model_not_found` | `PUT /api/agents/{random-uuid}/model` | 4xx/5xx |
| `test_send_message_agent_not_found` | `POST /api/agents/{random-uuid}/message` | 404 or 400 |
| `test_patch_agent_not_found` | `PATCH /api/agents/{random-uuid}` | 404 or 400 |

All endpoint tests follow the same pattern:

```
TestAppState::new() → .router() → test_request() → oneshot → assert_json_ok / assert_json_error
```

### Mock Driver Tests

These tests exercise the mock LLM drivers without touching the HTTP layer, validating driver-level contracts directly.

| Test Function | Driver | What It Validates |
|---|---|---|
| `test_mock_llm_driver_recording` | `MockLlmDriver` | Returns pre-configured responses in sequence, records calls (model, system prompt), and tracks `call_count` |
| `test_mock_llm_driver_custom_tokens_and_stop_reason` | `MockLlmDriver` | Builder methods `with_tokens` and `with_stop_reason` correctly customize `usage` and `stop_reason` in the response |
| `test_failing_llm_driver` | `FailingLlmDriver` | Always returns an error with the configured message; `is_configured()` returns `false` |

### Custom Configuration Test

`test_custom_config_kernel` demonstrates how to build a `TestAppState` with a customized kernel via `MockKernelBuilder::new().with_config()`, then verifies the config was applied by reading `app.state.kernel.config_ref()`.

## Key Helpers Used

All helpers are re-exported from the `librefang-testing` crate root:

- **`TestAppState::new()`** — Creates a test application with default (empty-mock) configuration.
- **`TestAppState::with_builder(builder)`** — Creates a test application from a custom `MockKernelBuilder`.
- **`app.router()`** — Returns a fully-wired Axum `Router` ready for `oneshot`.
- **`test_request(method, path, body)`** — Builds an `axum::http::Request<String>` with the given method, path, and optional JSON body.
- **`assert_json_ok(response)`** — Asserts status 200, deserializes the body into `serde_json::Value`, and returns it.
- **`assert_json_error(response, expected_status)`** — Asserts the given status code, deserializes the error body into `serde_json::Value`, and returns it.
- **`MockLlmDriver`** — A stub `LlmDriver` that returns canned responses, records all calls, and exposes `call_count()` and `recorded_calls()`.
- **`FailingLlmDriver`** — A stub `LlmDriver` that always fails, used for testing error-handling paths.

## Request Flow

```mermaid
flowchart LR
    A[test_request] --> B[router.oneshot]
    B --> C{Status Code}
    C -- 200 --> D[assert_json_ok]
    C -- 4xx_5xx["4xx/5xx"] --> E[assert_json_error]
    D --> F[assert on JSON fields]
    E --> F
```

## Conventions

1. **Tokio flavor**: Most endpoint tests use `#[tokio::test(flavor = "multi_thread")]` because the Axum router requires a multi-threaded runtime. Unit-level mock driver tests use the default single-thread flavor.
2. **No external services**: Everything runs in-process. `MockKernelBuilder` injects mock implementations so no real LLM backend, database, or network is needed.
3. **Idempotency contracts**: The DELETE test explicitly verifies that deleting a nonexistent agent returns `200 OK` with `{"status": "already-deleted"}` rather than `404`. This matches the API's idempotent deletion contract — retries after network blips must not surface phantom errors.

## Adding a New Test

To add a test for a new endpoint:

```rust
#[tokio::test(flavor = "multi_thread")]
async fn test_my_new_endpoint() {
    let app = TestAppState::new();
    let router = app.router();

    let body = serde_json::json!({ "key": "value" }).to_string();
    let req = test_request(Method::POST, "/api/my-endpoint", Some(&body));
    let resp = router.oneshot(req).await.expect("request failed");
    let json = assert_json_ok(resp).await;

    assert!(json.get("expected_field").is_some());
}
```

For mock driver tests that don't need the HTTP layer, skip `TestAppState` and construct the driver directly.