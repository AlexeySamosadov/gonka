# Gonka Code Review Rules

These rules are extracted from real review feedback by the core team (patimen, DimaOrekhovPS, gmorgachev, akup, IgnatovFedor). Every PR must comply before requesting human review.

---

## Go Type Safety

- **NEVER use bare `int`** — always use explicit `int64`, `int32`, or `uint64`
- Use `uint64` for epoch indices, counters, block heights, and similar non-negative values
- Use `math.MaxInt64` from stdlib instead of custom large constants
- Avoid `uint32` for values that may exceed 32-bit range (e.g., validation weights) — use `uint64`
- Be explicit about integer conversions: `int64(x)`, not implicit widening

## DoS & Input Validation

- **Always set upper limits** on user-controlled parameters (epochs, array lengths, iterations)
- **Validate limits BEFORE loops** — especially loops with N*M complexity
- Check both the count of items and their individual values before entering nested loops
- Add request body size limits on HTTP handlers (currently 10 MB max)
- Validate that iterators/denominators are non-zero before division
- Always `strings.TrimSpace()` user-provided strings before comparison or lookup — prevents subtle bugs from whitespace
- Validate all required environment variables and config at startup — fail early with clear messages, not mid-execution with obscure errors

## Error Handling

- **NEVER use `panic()` or `Must*()` in chain code** — these break consensus (enforced by `forbidigo` linter)
- Always return errors, never swallow them — if `SendCoinsFromModuleToAccount` fails, propagate the error
- Use `errorsmod.Wrapf()` with context for error wrapping
- When a function returns `(nil, nil)`, that violates Go conventions — nil error means success
- Log abnormal situations at Error level, even if the code can continue
- If a dependency is required for correct operation, treat its absence as an error — do not silently accept nil dependencies that will cause silent failures downstream

## Logging

- Use the project's standard logging (`logging.Info`, `logging.Error`, etc.) — **never `fmt.Println`**
- Add Info-level logs for significant state changes (e.g., skipping a participant, gate decisions)
- Health check logs that fire every N seconds should be Debug level, not Info — avoid log spam
- Warnings (`Warn`) are for conditions that are no longer errors but still notable
- If a field is intentionally left unpopulated or code is commented out, add a comment explaining why — unexplained empty fields are review flags

## Resource Management & Concurrency

- **Use `context.WithCancel`** for background goroutines — cancel on shutdown to prevent leaks
- If a constructor starts goroutines, add cleanup on error paths (defer cancel or explicit close)
- `math/rand` is NOT thread-safe — guard with mutex or use per-goroutine instances
- When replacing/stopping connections, consider pending queries — don't cancel immediately, use delayed shutdown
- Close resources (gRPC connections, RPC clients) properly — `RPC.Stop()` alone is not enough for full client cleanup
- When shared state has a write lock, **read operations must also hold a read lock** (`RLock`/`RUnlock`) — audit all cached getters for lock consistency
- **Heavy/blocking operations must run in a separate goroutine** — do not block the main processing loop with long-running tasks like historical block verification

## Determinism (Consensus-Critical)

- **NEVER iterate over Go maps directly in consensus code** — map iteration order is non-deterministic and will cause chain forks
- Use sorted key helpers (`sortedKeys`) to ensure deterministic ordering when iterating maps
- Validate uniqueness of IDs before sorting — duplicate keys can cause sorting artifacts and consensus failures
- Sort participants by address before including in seed computation to ensure deterministic ordering across nodes

## Caching

- **Cache must be block-based, not TTL-based** — data doesn't change between blocks, TTL can return stale data across block boundaries
- Clear cache on new block events
- Cache invalidation must happen BEFORE reading new data, not after
- If using gRPC caching, implement it as a gRPC Interceptor tied to connection lifecycle, not as a global cache
- Pass `grpc.CallOptions` through to cached invocations — don't drop them

## Performance

- **Avoid loading entire collections into memory** — use `Walk()` on Cosmos SDK collections instead of `GetAll*()`
- Don't iterate all bank balances when you only need participant balances — filter during iteration
- Use `shopspring/decimal` for financial calculations instead of floating-point arithmetic
- Extract HTTP client creation into shared functions — don't create new clients per request
- When querying data you already have in scope, pass it through instead of re-fetching
- `GetParams()` is expensive (deserializes a large struct) — call it **once at the top of a handler** and pass the result through, don't call it multiple times
- Use block height already tracked in `ChainPhaseTracker` rather than making a new ABCI query — avoid unnecessary network round-trips
- When checking if an element exists in a collection, **scan and return early** — don't build an intermediate map for a simple lookup
- Use `url.JoinPath` from Go stdlib for URL construction — extract URL-building logic into a shared reusable function to avoid duplication

## Testing

- Tests must have **actual assertions** — a test that only logs results is not a test
- Don't compare a value to itself (`require.Equal(t, params, params)`) — compare retrieved vs expected
- Use `testify/require` consistently — don't mix manual `if/t.Errorf` with require helpers
- Use `bytes.Repeat([]byte("a"), size)` instead of manual for-loops for generating test data
- Benchmarks with heavy iteration should use `testing.B`, not `testing.T`
- Extract test clients into shared helpers so tests use the same client as production
- Test edge cases: zero values, nil inputs, division by zero, overflow conditions
- **Database migrations must be tested** — apply migration to an old schema and verify the result matches a fresh schema. Migration failures are showstoppers
- When adding new fields to proto/params definitions, also update **Testermint `AppExport.kt`** with corresponding test parameters — missing test params cause governance test serialization failures

## PR Hygiene

- **Only include files directly related to the change** — no unrelated test files or configs
- Keep PRs focused — one concern per PR
- Don't leave unused variables, commented-out code, or TODO placeholders
- **Delete unused functions** — check IDE / static analysis for unreferenced functions, do not leave dead code
- If config defaults in code don't match documentation/PR description, fix the inconsistency
- Protobuf regeneration must match proto file changes exactly
- **No emojis in documentation**
- Do not include unrelated formatting/whitespace changes — they make diffs harder to review
- PRs must be complete and clean before merging — do not merge partial implementations
- Follow canonical **Go import ordering**: stdlib first, then third-party, then internal project imports — do not intermix groups

## Security (Chain-Specific)

- Validate all addresses with `sdk.AccAddressFromBech32`
- Validate coins with `IsValid()` and `IsZero()`
- Check for integer overflow before arithmetic operations on user inputs
- Prevent SSRF: use `NoRedirectClient` for outbound HTTP calls from handlers
- BLS signatures: always validate for duplicate slot indices before threshold checks
- Deterministic selection in consensus code is fine, but don't call it "secure random" — call it "deterministic weighted selection"

## HTTP Client

- Use `NoRedirectClient` (no-redirect HTTP client) for outbound requests from handlers — prevents SSRF via redirects
- Share HTTP client instances — extract creation into a function, don't duplicate
- Add connection pooling for long-lived services to prevent ephemeral port exhaustion

## Cosmos SDK Patterns

- Use epoch from the inference context, not current epoch, for membership checks
- When returning `(nil, nil)` from retry loops — return an error instead, exhausted retries is a failure
- Don't sleep after the last retry attempt — check before sleeping
- For retry-able operations, retry on transient errors (network), not just on empty results
- Use `context.WithCancel` derived contexts for per-client lifecycle management
- Do not use deprecated APIs or queries when newer alternatives exist — prefer in-memory data (e.g., `phaseTracker`) over making new queries
- When gating on settle/reward data, **always verify the epoch index matches** the expected epoch — stale data from older epochs can lead to incorrect gate decisions
- Before sending commands to MLNodes, **verify the node's current state/status** is compatible with the command — do not assume nodes are always in the expected phase
- When a specific value (e.g., `modelId`) is available in scope, pass it explicitly rather than using an empty string and relying on downstream resolution

## Python / MLNode

- In FastAPI code, use **FastAPI `Security` / dependency injection** for auth checks rather than manually calling auth functions in each handler — centralizes auth logic and prevents missed endpoints
