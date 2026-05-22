# Audit Finding: Epoch-Seal Thundering Herd

**Severity:** Medium
**Component:** SSE event broadcast (`epoch:sealed`) + witness fetch endpoints (`GET /trees/by-leaf/{leaf}`)
**Status:** Open
**Date:** 2026-05-22

## Description

The aqua-timestamp aggregator collects client-submitted leaf hashes during
an epoch window (default 600 seconds). When the epoch closes, the sealer
task commits all witness revisions to storage and then broadcasts an
`epoch:sealed` SSE event to every connected subscriber via the
`EventBus`. Because the broadcast is instantaneous (a single Tokio
broadcast channel `send`), all subscribed clients discover the seal at
effectively the same wall-clock instant.

Each client then issues `GET /trees/by-leaf/{leaf}?method=evm` (and
optionally `?method=qtsa`) to retrieve its witness pair. With N
concurrent subscribers, this means N requests arrive within a sub-second
window immediately after each seal. The result is a thundering-herd load
spike: request concurrency jumps from near-zero to N in a step function,
saturating the Axum thread pool, the fjall storage read path, and the
outbound bandwidth simultaneously.

The pattern is structural. It is not caused by misbehaving clients; it
is the natural consequence of a single broadcast event triggering
identical behavior across all subscribers.

## Impact

- **Request queuing.** With N clients, the server receives N witness
  fetch requests within seconds of each seal. Each request performs
  two fjall key lookups (`leaf_to_tips`, `tip_to_pair`) plus two
  revision JSON reads (`witness_revisions`). Under contention these
  reads serialize on the fjall LSM compaction lock.
- **Observed timeout at scale.** During stress testing with
  `STRESS_COUNT=13548` and `STRESS_FETCH_SPREAD_SECS=0` (all fetches
  fired concurrently), the server timed out before delivering all
  witnesses. The default `request_timeout` of 20 seconds was
  insufficient.
- **Cascading retry storms.** When witness fetches time out, clients
  retry (the `stress_1000` example retries up to 6 times with
  exponential back-off). Retries overlap with the next epoch's seal
  event, compounding the spike.
- **Resource amplification.** Each seal produces two witness methods
  (EVM + qTSA). A client fetching both methods doubles the request
  count to 2N per seal event.
- **Latency variance.** Even when the server survives the spike, the
  last requests in the queue experience latencies orders of magnitude
  higher than the first, violating any SLA expectation of uniform
  response time.
- **Production projection.** At the design target of 500 active
  wallets (see capacity model), each submitting multiple leaves per
  epoch, the post-seal spike could reach thousands of concurrent
  requests on a 2-core/4GB server.

## Reproduction

The `stress_1000` example in `crates/aqua-timestamp-client/examples/stress_1000.rs`
reproduces the vulnerability directly:

```bash
# Thundering herd: all fetches at once (spread = 0)
STRESS_COUNT=13548 \
STRESS_FETCH_SPREAD_SECS=0 \
STRESS_PARALLEL=32 \
    cargo run -p aqua-timestamp-client --features live-tests --example stress_1000
```

With `STRESS_FETCH_SPREAD_SECS=0`, all 13,548 witness fetches fire
concurrently after the seal event, saturating the server. The test
reports timeouts and non-zero exit.

For comparison, the default `STRESS_FETCH_SPREAD_SECS=60` spreads
fetches over 60 seconds and completes successfully, confirming the
spike (not the total load) is the problem.

## Root Cause

The `EventBus::send` method (in `crates/aqua-timestamp-core/src/events.rs`)
delivers the `EpochSealed` event to all subscribers simultaneously via
`tokio::sync::broadcast::Sender::send`. There is no per-subscriber
jitter, delay, or staggering. The SSE route handler
(`routes::sse_events`) forwards the event immediately to every HTTP
connection. Every well-behaved client reacts identically: parse the
event, issue a witness fetch.

The sealer emits the event from `run_sealer_with_interval` (in
`crates/aqua-timestamp-core/src/sealer.rs`, around line 336) immediately
after `seal_once` returns. There is no window between "witnesses are
readable in storage" and "clients are notified," which is correct for
consistency but creates the spike.

## Recommended Mitigation

### Option A: Server-side per-subscriber fetch delay (preferred)

Add a `fetch_after` field to the `epoch:sealed` SSE event payload. The
server assigns each subscriber a random offset drawn uniformly from
`[0, spread_window)` (e.g., 60 seconds). The SSE event becomes:

```json
{
  "type": "epoch_sealed",
  "epoch_id": 42,
  "leaf_count": 500,
  "merkle_root": "0xabc...",
  "timestamp": 1779100000,
  "fetch_after": 23
}
```

Well-behaved clients delay their witness fetch by `fetch_after` seconds,
converting the step-function spike into a uniform ramp over the spread
window. Non-compliant clients still work (the field is advisory, not
enforced), but the majority of cooperative clients smooth the load.

**Advantages:** simple to implement; no new middleware; backward
compatible (old clients ignore the extra field); spread window is
operator-tunable.

**Implementation notes:**
- `SseEvent::EpochSealed` gains an `Option<u64>` field `fetch_after`.
- The SSE handler assigns the offset per subscriber at event-emission
  time (not per-event, since the broadcast channel clones the same
  event to all receivers; the offset must be injected in the
  `filter_map` stage of `sse_events` where each subscriber has its
  own stream).
- The spread window should be configurable in `[epoch]` config, e.g.
  `fetch_spread_secs = 60`.

### Option B: Server-side token-bucket rate limiting

Add a Tower middleware or Axum layer on the `/trees/by-leaf/{leaf}`
endpoint with a token-bucket rate limiter. When the bucket is exhausted,
return `429 Too Many Requests` with a `Retry-After` header. Clients
back off and retry.

**Advantages:** defends against non-cooperative clients; no protocol
change.

**Disadvantages:** more complex; adds a new failure mode (429s); clients
need retry logic (the stress test already has it, but other clients may
not); does not reduce total server work, only spreads it.

### Option C: Combined approach

Implement Option A for cooperative clients and Option B as a safety net.
The `fetch_after` advisory reduces the baseline load; the token-bucket
catches outliers and attackers.

## Client-Side Workaround (implemented)

The `stress_1000` example already implements a client-side mitigation:
`STRESS_FETCH_SPREAD_SECS` (default 60) spreads witness fetches
uniformly over the configured window. Each fetch is delayed by
`(index * spread * 1000) / (count - 1)` milliseconds, turning the spike
into a linear ramp.

This is effective when the stress test is the only client, but it is a
cooperative-only mitigation. The server remains vulnerable to any client
(or set of clients) that fetches immediately on receiving the SSE event.

## References

- `crates/aqua-timestamp-core/src/events.rs`: `EventBus` and `SseEvent`
  definitions
- `crates/aqua-timestamp-core/src/sealer.rs`: `run_sealer_with_interval`,
  line ~336 where `EpochSealed` is emitted
- `crates/aqua-timestamp/src/routes.rs`: `sse_events` handler (line ~711),
  `get_tree_by_leaf` handler (line ~576)
- `crates/aqua-timestamp-client/examples/stress_1000.rs`:
  `STRESS_FETCH_SPREAD_SECS` client-side spread (line ~176)
