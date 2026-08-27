# BLAKE3-256 vs SHA3-256: Performance Considerations

**Status:** SHA3-256 is the current protocol-level commitment. A `feat/blake3-256`
branch exists in `aqua-rs-sdk`; switching is a breaking change to all
hash-dependent verification. This note documents the performance rationale
for the transition once that branch lands.

## Throughput comparison (single-threaded, modern x86-64)

| Algorithm | Throughput | Notes |
|-----------|-----------|-------|
| BLAKE3-256 | ~4-6 GB/s | With SIMD (AVX-512 / AVX2 / SSE4.1) |
| SHA3-256 (Keccak) | ~400-600 MB/s | Sponge construction, hard to vectorize |

**Typical speedup: 7-10x on commodity hardware.**

BLAKE3 also scales with parallelism. Its internal Merkle tree structure
enables multi-threaded hashing out of the box, pushing throughput even
higher on large inputs.

## Why SHA3 is slower

Keccak's sponge construction operates on a 1600-bit state that is
inherently difficult to vectorize. BLAKE3 was designed from the ground up
for SIMD and parallel tree hashing.

## Platform-specific notes

**WASM** (relevant to this SDK): the gap narrows because WASM does not
expose native SIMD on all runtimes, but BLAKE3 still wins by ~3-5x in
practice.

**Embedded / constrained targets:** SHA3 can be more competitive because
BLAKE3's speed advantage depends on SIMD, which may not be available.

## Security parity

Both provide 128-bit collision resistance at 256-bit output.

- **BLAKE3** is based on ChaCha (well-analyzed, wide deployment).
- **SHA3** is a NIST standard built on Keccak.

Neither has known weaknesses. The choice is a performance decision, not a
security one.

## Impact on the Aqua timestamping service

Batch timestamping with many Merkle operations (tree construction,
inclusion proofs, root computation) is hash-bound. A 7-10x improvement in
hashing throughput directly reduces seal latency and raises the ceiling on
leaves-per-epoch before the sealer becomes a bottleneck. On the WASM path
(browser-side verification), the 3-5x gain makes client-side proof
verification noticeably faster.

## Transition plan

The `feat/blake3-256` branch in `aqua-rs-sdk` introduces `HashType::Blake3_256`
alongside the existing `HashType::Sha3_256`. Once that branch merges:

1. The protocol gains a new hash type identifier; existing SHA3-256 trees
   remain valid and verifiable.
2. New trees default to BLAKE3-256.
3. Verification must support both hash types for backwards compatibility.
4. All downstream consumers (aqua-timestamp, aqua-node, aqua-state-viewer,
   browser clients) update their SDK dependency to pick up the new type.
