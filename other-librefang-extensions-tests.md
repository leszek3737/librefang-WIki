# Other — librefang-extensions-tests

# librefang-extensions-tests — Vault Roundtrip Integration Tests

Integration test suite for the `CredentialVault` (`librefang_extensions::vault`). Exercises the full encrypt → persist → reload → decrypt lifecycle and pins several invariants the daemon boot path depends on.

## What These Tests Guard

The vault is the single source of truth for API keys and other secrets. If any of these tests break, the daemon's credential handling is compromised — do not merge without understanding why.

| Invariant | Test |
|---|---|
| Master key must decode to exactly 32 bytes | `decode_master_key_rejects_wrong_byte_length`, `decode_master_key_rejects_literal_32_ascii_chars` |
| A vault initialized under key K can only be reopened with key K | `vault_unlock_with_wrong_key_fails` |
| Credentials survive serialize to disk + deserialize from disk | `vault_roundtrip_encrypt_then_decrypt_with_same_key` |
| `list_keys` hides the `#3651` sentinel | `vault_roundtrip_encrypt_then_decrypt_with_same_key` |
| Sentinel key is immune to `set` / `remove` | `vault_rejects_writes_to_reserved_sentinel_key` |

## Test Overview

### Key Decoding (`decode_master_key`)

These tests pin a critical operator foot-gun: **32 ASCII characters ≠ 32 bytes**. The production recipe is `openssl rand -base64 32`, which produces a 44-character base64 string that decodes to 32 bytes. If an operator pastes a literal 32-character string instead, two failure modes exist:

1. **Valid base64, wrong length** — 32 ASCII chars that are valid base64 decode to 24 bytes. `decode_master_key` must reject this with an "Invalid key length" error.
2. **Invalid base64** — 32 chars outside the base64 alphabet fail at the decode step entirely.

Both paths surface a structured `ExtensionError` rather than silently booting with a truncated key.

### Full Roundtrip

```
init_with_key → set (multiple entries) → drop vault
    ↓
new → unlock_with_key (same key) → get → list_keys
```

The vault writes AES-GCM encrypted data to a temporary file. After the `CredentialVault` is dropped (zeroing in-memory state), a fresh instance reopens the same file and decrypts everything. This validates that serialization, encryption, and key derivation are all consistent across sessions.

### Wrong-Key Rejection

AES-GCM authenticated encryption guarantees that a wrong key produces a decryption failure, not garbage plaintext. The test initializes under one key, then attempts `unlock_with_key` with a different key. The vault must remain locked (`!is_unlocked()`) and return either `ExtensionError::Vault` or `ExtensionError::VaultKeyMismatch`.

### Sentinel Protection

The `SENTINEL_KEY` constant (the `#3651` internal marker) is used by the boot path to verify vault integrity. If external code could overwrite or remove it, the daemon would silently break on restart. Both `set(SENTINEL_KEY, ...)` and `remove(SENTINEL_KEY)` must return `ExtensionError::Vault`.

## Test Helpers

### `fixture_key_b64()`

Returns a deterministic base64-encoded 32-byte key (all zeros). Not cryptographically strong — only used for reproducibility across test runs.

### `fixture_vault_path(tmp: &TempDir)`

Joins `vault.enc` to a temporary directory path. Every test gets its own isolated vault file via `tempfile::tempdir()`.

## Running

```bash
# From the workspace root
cargo test -p librefang-extensions --test vault_roundtrip

# Or run individual tests
cargo test -p librefang-extensions --test vault_roundtrip -- decode_master_key_rejects_literal_32_ascii_chars --exact
```

No OS keyring or environment variables are required. All tests are self-contained.

## Dependencies on Production Code

| Symbol | Source |
|---|---|
| `CredentialVault` | `librefang_extensions::vault` |
| `decode_master_key` | `librefang_extensions::vault` |
| `SENTINEL_KEY` | `librefang_extensions::vault` |
| `ExtensionError` | `librefang_extensions` |
| `Zeroizing<T>` | `zeroize` crate |

## When Adding New Vault Tests

- Use `fixture_key_b64()` and `fixture_vault_path()` to stay consistent with existing tests.
- Always assert on the error variant (`ExtensionError::Vault` vs `VaultKeyMismatch`) — don't just check "it failed."
- If testing a new vault operation, also verify it doesn't leak `SENTINEL_KEY` through the new path.
- Keep tests hermetic: each test creates and owns its own `TempDir`.