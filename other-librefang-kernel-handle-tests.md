# Other — librefang-kernel-handle-tests

# librefang-kernel-handle — Integration Tests

## Purpose

This test module validates the **default trait method implementations** provided by `librefang-kernel-handle`. The crate defines a large surface of kernel capability traits (`AgentControl`, `MemoryAccess`, `TaskQueue`, etc.) where most methods carry default implementations so that downstream kernel adapters only override the capabilities they actually support. These tests ensure those defaults behave correctly and that higher-level convenience methods correctly delegate to their lower-level counterparts.

## Test Files

| File | Focus |
|------|-------|
| `defaults_approval.rs` | Default gating decisions — approval and tool denial |
| `defaults_delegation.rs` | Convenience methods delegate to base trait methods |
| `defaults_returns.rs` | Default return values, error variants, and configuration constants |
| `send_channel_file_data_zero_copy.rs` | Regression test for zero-copy `Bytes` semantics (#3553) |

## Test Strategy

Every test file follows the same pattern:

1. **Define a stub struct** (e.g., `NoopKernelHandle`, `TrackingSendHandle`) that implements all required kernel traits.
2. Most trait methods are given no-op or error-returning bodies — the tests never call these.
3. Methods that *are* under test are either left as the **default implementation** (to verify its behavior) or instrumented with an `AtomicBool` (to verify delegation).
4. The test asserts on the return value or on whether the base method was invoked.

This approach avoids depending on a real kernel runtime while exercising the trait dispatch layer that production code uses.

## Default Behavior Verified

### Approval and Tool Policy (`defaults_approval.rs`)

The default implementations create a **permissive** execution environment:

| Method | Default | Rationale |
|--------|---------|-----------|
| `request_approval` | `ApprovalDecision::Approved` | Auto-approve when no approval backend is configured |
| `is_tool_denied_with_context` | `false` | No tools are denied in the absence of a policy |
| `requires_approval` | `false` | No tools require human sign-off by default |

### Delegation Chains (`defaults_delegation.rs`)

Several trait methods exist as convenience wrappers that delegate to a simpler base method. Each test uses an `AtomicBool` to confirm the base method was actually called:

```
send_to_agent_as(agent, msg, parent)
  └─► send_to_agent(agent, msg)

send_to_agent_as_with_key(agent, msg, parent, key)
  └─► send_to_agent_as(agent, msg, parent)
       └─► send_to_agent(agent, msg)

send_to_agent_with_key(agent, msg, key)
  └─► send_to_agent(agent, msg)

spawn_agent_checked(toml, parent, capabilities)
  └─► spawn_agent(toml, parent)

requires_approval_with_context(tool, sender, channel)
  └─► requires_approval(tool)
```

The `TrackingSendHandle`, `TrackingSpawnHandle`, and `TrackingApprovalHandle` stubs each embed a single `AtomicBool` to detect whether the leaf method fires. The test passes only if the flag is set **and** the return value propagates correctly.

### Return Values and Error Variants (`defaults_returns.rs`)

Default implementations for traits that require a kernel backend return structured errors rather than panicking:

| Trait | Method | Default Return |
|-------|--------|---------------|
| `ToolPolicy` | `resolve_user_tool_decision` | `UserToolGate::Allow` |
| `MemoryAccess` | `memory_acl_for_sender` | `None` |
| `ToolPolicy` | `tool_timeout_secs` / `tool_timeout_secs_for` | `120` (seconds) |
| `ToolPolicy` | `max_agent_call_depth` | `5` |
| `ToolPolicy` | `readonly_workspace_prefixes` / `named_workspace_prefixes` | empty `Vec` |
| `CronControl` | `cron_create`, `cron_list`, `cron_cancel` | `Err(KernelOpError::Unavailable("Cron scheduler"))` |
| `WikiAccess` | `wiki_get` | `Err(KernelOpError::Unavailable("wiki_get"))` |
| `WikiAccess` | `wiki_search` | `Err(KernelOpError::Unavailable("wiki_search"))` |
| `WikiAccess` | `wiki_write` | `Err(KernelOpError::Unavailable("wiki_write"))` |

The `KernelOpError::Unavailable` variant carries a **capability name** string, enabling callers to match on the specific unavailable subsystem rather than parsing error messages. The wiki methods each report their own verb (`"wiki_get"`, `"wiki_search"`, `"wiki_write"`) so audit logs can identify which entry point was invoked while the vault was disabled.

### Zero-Copy File Transfer (`send_channel_file_data_zero_copy.rs`)

Regression test for **#3553**. The `ChannelSender::send_channel_file_data` method accepts `bytes::Bytes` instead of `Vec<u8>`. This matters because channel adapters (retry wrappers, metering, fan-out) clone the buffer, and `Bytes::clone()` is a reference-count bump — not a heap allocation.

Three properties are verified:

1. **Cloning shares the allocation** — Multiple `Bytes::clone()` calls produce handles pointing at the same underlying memory address.
2. **The kernel sees the caller's allocation** — `CapturingFileKernel` records the pointer passed into `send_channel_file_data`. The test confirms it matches the address the caller held (after `.clone()`), proving no silent copy occurred at the trait boundary.
3. **`Vec ↔ Bytes` round-trip is O(1)** — When `Bytes` uniquely owns its allocation, `Vec::from(bytes)` reuses the same pointer. This pins the `bytes` crate vtable behavior documented for 1.x.

```rust
// The critical assertion — clone at the call site, kernel sees the same address
kernel
    .send_channel_file_data("telegram", "user@example", original.clone(), ...)
    .await;

assert_eq!(seen_addr, expected_addr);
```

## How to Add a New Default-Method Test

1. Create or reuse a stub struct in the appropriate test file.
2. Implement all required traits. Boilerplate-heavy traits (`MemoryAccess`, `TaskQueue`, `KnowledgeGraph`, etc.) can return `Err("not used".into())` since they won't be called.
3. If testing delegation, add an `AtomicBool` field to your stub and set it in the base method.
4. If testing a return value, leave the method as the default impl and assert on the result.
5. For `Unavailable` error variants, pattern-match on `KernelOpError::Unavailable(capability_name)`.