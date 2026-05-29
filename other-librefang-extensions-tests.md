# Other — librefang-extensions-tests

# Vault Roundtrip Integration Tests

## Purpose

This test module validates the **credential vault** (`librefang_extensions::vault`) across its critical security invariants. It exercises the full encrypt → persist → reload → decrypt lifecycle without depending on OS keyrings or environment variables, using an explicit master key instead.

The tests guard against several historical foot-guns and regression risks that the daemon's boot path depends on.

## Invariants Under Test

| Invariant | Test | Why It Matters |
|---|---|---|
| 32-byte key requirement | `decode_master_key_rejects_wrong_byte_length` | Prevents silently booting with a truncated key |
| Base64 decoding is mandatory | `decode_master_key_rejects_literal_32_ascii_chars` | 32 ASCII chars decode to 24 bytes, not 32 — catches the `LIBREFANG_VAULT_KEY` operator mistake |
| Wrong-key detection | `vault_unlock_with_wrong_key_fails` | AES-GCM authentication must reject mismatched keys rather than produce garbage plaintext |
| Sentinel key isolation | `vault_rejects_writes_to_reserved_sentinel_key` | The `#3651` sentinel must not be writable or removable via the public API |
| Full round-trip fidelity | `vault_roundtrip_encrypt_then_decrypt_with_same_key` | Data survives encryption, disk persistence, process restart (simulated via drop/reopen), and decryption |
| Sentinel key hidden from enumeration | `vault_roundtrip_encrypt_then_decrypt_with_same_key` | `list_keys` must not surface the sentinel alongside user keys |

## Test Design

### Key Handling

```
┌─────────────────────────────────────────────┐
│  Master Key Lifecycle                        │
│                                              │
│  fixture_key_b64()                           │
│       │                                      │
│       ▼                                      │
│  decode_master_key(b64_string)               │
│       │                                      │
│       ├──► Ok(Zeroizing<[u8;32]>)            │
│       │        used by init_with_key /       │
│       │        unlock_with_key               │
│       │                                      │
│       └──► Err(ExtensionError)               │
│              • wrong byte length             │
│              • invalid base64                │
└─────────────────────────────────────────────┘
```

The helper `fixture_key_b64()` produces a deterministic all-zeros 32-byte key, base64-encoded to a 44-character string. This mirrors the production recipe (`openssl rand -base64 32`) without requiring cryptographic strength — tests only need the key to round-trip correctly.

### Vault Lifecycle

Each vault test follows the same pattern:

1. **Create** a `CredentialVault` pointing at a temporary file path.
2. **Initialize** with `init_with_key`, which creates the encrypted store on disk.
3. **Write** credentials via `set`.
4. **Drop** the vault (simulates process exit; memory is zeroed).
5. **Reopen** a new `CredentialVault` at the same path.
6. **Unlock** with `unlock_with_key`.
7. **Verify** credentials via `get` / `list_keys`.

```
┌──────────┐     ┌───────────┐     ┌──────────┐     ┌───────────┐
│ new(path)│────►│init_with  │────►│ set(k,v) │────►│  drop()   │
└──────────┘     │ _key(key) │     │ (repeat) │     │ (persist) │
                 └───────────┘     └──────────┘     └─────┬─────┘
                                                          │
                       ┌──────────────────────────────────┘
                       ▼
                ┌──────────────┐     ┌──────────────────┐
                │unlock_with   │────►│ get(k) / list_   │
                │ _key(same_k) │     │ keys() verify    │
                └──────────────┘     └──────────────────┘
```

### Sentinel Key Protection

The constant `SENTINEL_KEY` (defined in `librefang_extensions::vault`) is an internal marker used by the boot-path verification branch (`#3651`). The vault must reject any attempt to `set` or `remove` this key through the public API, and `list_keys` must exclude it from results.

## Error Contracts

The tests assert against `ExtensionError` variants from `librefang_extensions`:

- **`ExtensionError::Vault(_)`** — general vault operation failures (sentinel writes, sentinel removals, decode errors mentioning "Invalid key length").
- **`ExtensionError::VaultKeyMismatch { .. }`** — wrong-key unlock. The tests accept either this variant or `Vault(_)` because the underlying AES-GCM failure message routing has historically varied between format versions. The contract is simply "non-Ok".

## Running

```sh
cargo test --package librefang-extensions --test vault_roundtrip
```

All tests are self-contained — each creates its own `tempfile::TempDir` and cleans up on completion. No external state, environment variables, or OS keyring access is required.