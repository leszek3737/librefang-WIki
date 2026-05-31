# Kernel Core — librefang-kernel-metering-src

# Kernel Core — Metering Engine (`librefang-kernel-metering`)

Tracks LLM call costs and enforces spending quotas across four scopes: **global**, **per-agent**, **per-provider**, and **per-user**. All persistent data flows through a SQLite-backed `UsageStore`; an in-memory reservation ledger prevents concurrent requests from collectively overshooting caps before their costs settle.

## Architecture

```mermaid
graph TD
    Caller -->|"1. reserve_global_budget()"| ME[MeteringEngine]
    ME -->|estimated_usd| RL[CostReservationLedger]
    RL -->|Mutex| RESERVED["reserved_usd (f64)"]
    ME -->|"2. dispatch LLM call"| LLM[LLM Provider]
    LLM -->|usage| ME
    ME -->|"3. settle() or release()"| RL
    ME -->|"check_all_and_record()"| US[UsageStore (SQLite)]
    ME -->|"flag_provider_budget_exhausted()"| PES[ProviderExhaustionStore]
    PES -->|skip exhausted provider| FC[FallbackChain]
```

## Budget Enforcement Scopes

| Scope | Methods | Time Windows |
|---|---|---|
| Global | `check_global_budget`, `reserve_global_budget`, `check_global_budget_and_record` | hourly, daily, monthly |
| Per-agent | `check_quota`, `check_quota_and_record` | hourly, daily, monthly |
| Per-provider | `check_provider_budget` | hourly, daily, monthly + hourly tokens |
| Per-user | `check_user_budget` | hourly, daily, monthly |

All scopes share the convention that a **zero limit means unlimited** — the corresponding window is skipped during enforcement.

### Atomic vs. Non-Atomic Checks

Non-atomic methods (`check_quota`, `check_global_budget`, `check_provider_budget`, `check_user_budget`) query settled spend from SQLite and return a pass/fail. They are suitable for dashboards and pre-dispatch gating.

Atomic methods (`check_quota_and_record`, `check_global_budget_and_record`, `check_all_and_record`) perform the check and the insert inside a **single SQLite transaction**, closing the TOCTOU race where concurrent requests both pass the check before either writes its cost. Use `check_all_and_record` as the preferred post-call recording path — it validates per-agent, global, and per-provider budgets simultaneously.

## In-Flight Cost Reservation

The core concurrency problem (#3616): when N triggers fire simultaneously, they all read the same pre-call total from SQLite, all pass the budget gate, and all commit — producing multi-x overshoots.

`CostReservationLedger` solves this by holding an estimated USD cost for every in-flight LLM call in a single `Mutex<f64>`. The critical section in `check_and_add` performs the limit check and the addition under the same lock acquisition, so the second concurrent caller sees the first's reservation.

### Reservation Lifecycle

1. **Reserve** — call `MeteringEngine::reserve_global_budget(&budget, estimated_usd)` before dispatching the LLM request. Returns a `MeteringReservation`.
2. **Dispatch** — send the request to the LLM provider.
3. **Settle or release**:
   - `reservation.settle()` — the call completed and actual usage was recorded. Releases the estimated hold so the ledger doesn't double-count against the settled SQLite row.
   - `reservation.release()` — the call failed before incurring cost. Releases the hold immediately.
   - **Drop guard** — if the reservation is dropped without calling `settle()` or `release()` (e.g. due to a panic), `Drop::drop` releases the hold as a safety net.

`MeteringReservation` is annotated `#[must_use]` so the compiler warns if a reservation is silently discarded.

### Intentional Asymmetry in Comparison Operators

- `reserve_global_budget` uses `>` (strictly greater than) when projecting spend — a single call that exactly reaches the cap is still allowed through.
- `check_global_budget` (post-call gate) uses `>=` (greater than or equal) — once the limit is fully consumed, no further calls are dispatched.

This ensures a single call at the boundary succeeds while subsequent calls are blocked.

## Provider Exhaustion Integration (#4807)

When a per-provider budget gate refuses a provider, the engine can flag it in a shared `ProviderExhaustionStore` so the LLM fallback chain skips that slot for `DEFAULT_LONG_BACKOFF` without re-dispatching a request that would only be denied again.

```rust
let engine = MeteringEngine::new(store)
    .with_exhaustion_store(exhaustion_store.clone());
```

Wire the **same** `ProviderExhaustionStore` instance into both the metering engine and the `FallbackChain` so both layers observe a coherent view. Retrieve the store later via `engine.exhaustion_store()` to pass to downstream clients.

The exhaustion flag is set by the private method `flag_provider_budget_exhausted`, which:
- Logs at `info` level with target `metering`.
- Calls `store.mark_exhausted(provider, ExhaustionReason::BudgetExceeded, Some(instant + DEFAULT_LONG_BACKOFF))`.
- Is a no-op when no exhaustion store is attached (backward compatible).

## Cost Estimation

### Token-to-USD Calculation

The private function `estimate_cost_from_rates` handles all pricing math:

| Token Type | Pricing |
|---|---|
| Regular input | `(tokens / 1M) × input_per_m` |
| Cache-read input | `(tokens / 1M) × input_per_m × 0.10` |
| Cache-creation input | `(tokens / 1M) × input_per_m × 1.25` |
| Output | `(tokens / 1M) × output_per_m` |

Regular input tokens are derived as `input_tokens.saturating_sub(cache_read + cache_creation)` where the cache sum uses `saturating_add` to prevent overflow from malicious provider responses.

### Two Estimation Paths

- **`estimate_cost`** — static method, uses hardcoded default rates ($1/$3 per million tokens). Independent of any catalog. Use in unit tests or when no catalog is available.
- **`estimate_cost_with_catalog`** — reads pricing from the `ModelCatalog`. Falls back to default rates when the model is unknown. Special-cases ChatGPT session-auth models (provider `"chatgpt"`) that report zero catalog pricing — these use the legacy default rates so budgets still get a conservative non-zero estimate. Subscription-only providers like `alibaba-coding-plan` intentionally report $0 cost.

## Public API Reference

### `MeteringEngine`

Constructed via `new(store)` with an optional `with_exhaustion_store(store)` builder call.

**Pre-call reservation:**
- `reserve_global_budget(&budget, estimated_usd) → Result<MeteringReservation>`
- `pending_reserved_usd() → f64`

**Non-atomic quota checks:**
- `check_quota(agent_id, &quota)`
- `check_global_budget(&budget)` — includes in-flight pending cost
- `check_provider_budget(provider, &provider_budget)`
- `check_user_budget(user_id, &user_budget)`

**Atomic check-and-record:**
- `check_quota_and_record(&record, &quota)`
- `check_global_budget_and_record(&record, &budget)`
- `check_all_and_record(&record, &quota, &budget)` — preferred post-call path

**Cost estimation (static):**
- `estimate_cost(model, input_tokens, output_tokens, cache_read, cache_creation) → f64`
- `estimate_cost_with_catalog(&catalog, model, input_tokens, output_tokens, cache_read, cache_creation) → f64`

**Query and maintenance:**
- `record(&record)`
- `budget_status(&budget) → BudgetStatus`
- `get_summary(agent_id: Option<AgentId>) → UsageSummary`
- `get_by_model() → Vec<ModelUsage>`
- `cleanup(days) → usize`
- `exhaustion_store() → Option<ProviderExhaustionStore>`

### `BudgetStatus`

Serialized snapshot returned by `budget_status` containing current spend, limits, and utilization percentages for hourly/daily/monthly windows plus the alert threshold and default token limit.

## Concurrency Safety Notes

- The in-process `CostReservationLedger` only synchronizes callers within a single process. Multi-process deployments can still race at the SQLite level; the post-call `check_all_and_record` transaction is the authoritative gate for that scenario.
- SQLite reads in `reserve_global_budget` happen outside the in-memory lock because they can be slow and may fail. Settled spend is monotonic within a time window, so the just-observed value cannot under-count the gate.
- All arithmetic on token counts from provider wire data uses `saturating_add` and `saturating_sub` to prevent overflow from `u64::MAX/2 + 1` style inputs that previously caused debug panics and silent release-mode wrapping.