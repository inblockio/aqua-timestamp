# Handover: Merkle Proof Membership Shielding

**Date:** 2026-05-20
**From:** Tim + Claude session
**Scope:** aqua-rs-sdk (template change) + aqua-timestamp (accumulator/minter change)

## Context

aqua-timestamp is a batching timestamping service. Users submit revision
hashes, which accumulate into per-epoch Merkle trees. The Merkle root is
dual-anchored (Sepolia EVM + eIDAS qTSA). Each submitter receives an
inclusion proof linking their hash to the anchored root.

The service is live at `https://timestamp.inblock.io` with M0 through M5
shipped. See `docs/success-criteria.md` for milestone definitions and
the project `CLAUDE.md` for full architecture context.

## The problem

Standard Merkle inclusion proofs leak sibling hashes. At least one
sibling in every proof is a **raw leaf hash submitted by a different
user**. This creates a membership inference attack: if an attacker
suspects hash H_x was submitted, they can check it against the sibling
nodes in their own proof to confirm or deny membership in the same epoch.

This is not a content leak (Aqua revision hashes are opaque and
non-reversible). It is a **membership leak**: the ability to confirm
that a specific hash was timestamped in a specific epoch.

Full analysis with exploitation scenarios is in
`docs/attack_vector_merkle_proof_sibling_leak.md`.

## The solution: per-leaf shielding nonces

Before building the Merkle tree, compute a shielded value for each leaf:

```
S = sha3(leaf || nonce)
```

where `nonce` is a random 32-byte value generated server-side. The tree
is built over shielded values. Proofs contain shielded siblings that
cannot be reversed without the per-leaf nonce, which only the
corresponding submitter holds.

Verification stays self-contained: the witness payload carries
`(leaf_via_previous_revision, shielding_nonce, merkle_proof, merkle_root)`.
A verifier computes `S = sha3(leaf || nonce)`, walks the proof, and
checks the root against the on-chain anchor. No external secret needed.

## Implementation plan

### Step 1: SDK template change (aqua-rs-sdk)

**Why a new template:** The existing `timestamp_evm.json` and
`timestamp_tsa.json` schemas set `additionalProperties: false`. Their
template hashes are content-addressed identifiers already anchored
on-chain in existing witnesses. Adding a field changes the hash,
breaking the identity of the template. Clean path is new template
variants.

**What to do:**

1. Copy `src/schema/templates/timestamp_evm.json` to
   `timestamp_evm_v2.json`. Add:
   ```json
   "shielding_nonce": {
     "type": "string",
     "maxLength": 256,
     "description": "Per-leaf random nonce for membership shielding. Hex-encoded, 0x-prefixed."
   }
   ```
   Add `"shielding_nonce"` to the `required` array. Keep
   `additionalProperties: false`.

2. Same for `timestamp_tsa.json` to `timestamp_tsa_v2.json`.

3. Add Rust structs `EvmTimestampPayloadV2` / `TsaTimestampPayloadV2`
   with `shielding_nonce: String`. Derive `TEMPLATE_LINK` from the new
   schema hash.

4. Existing `EvmTimestampPayload` / `TsaTimestampPayload` stay
   untouched. Old witnesses remain valid under the old template hash.

### Step 2: Accumulator change (aqua-timestamp)

**File:** `crates/aqua-timestamp-core/src/accumulator.rs`
(or wherever `LeafEntry` is defined)

- Generate a random 32-byte nonce per leaf at acceptance time.
- Store `nonce` alongside `leaf` and `submitter_did` in `LeafEntry`.
- Return the nonce to the submitter in the `POST /v1/leaves` response
  body (this is the submitter's secret; they need it to verify later).
- Persist the nonce in the fjall `epoch_leaves` partition so it
  survives restarts.

### Step 3: Sealer change (aqua-timestamp-core)

**File:** `crates/aqua-timestamp-core/src/sealer.rs`

In `build_record_and_sorted_leaves`:
- Instead of sorting raw leaf hashes, compute
  `shielded = sha3(leaf || nonce)` for each `LeafEntry`.
- Sort and build the Merkle tree over shielded values.
- The `EpochRecord.merkle_root` is now the root of the shielded tree.
- The anchor layer (EVM/qTSA) anchors this root; no change needed there.

### Step 4: Witness minter change (aqua-timestamp-core)

**File:** `crates/aqua-timestamp-core/src/witness.rs`

- Use the V2 payload structs instead of V1.
- Populate `shielding_nonce` with the hex-encoded nonce for that leaf.
- Compute the inclusion proof against the shielded tree (the leaf's
  position is its shielded value's position in the sorted order).
- The `merkle_proof` field now contains shielded siblings.

### Step 5: Verification client update (aqua-timestamp-e2e)

**File:** `crates/aqua-timestamp-e2e/src/flow.rs`

- After fetching the witness, extract `shielding_nonce` from the
  payload.
- Compute `S = sha3(leaf || nonce)`.
- Verify: `inclusion_verify(S, index, proof, root)`.
- The e2e test in `tests/e2e/live_roundtrip.sh` must exercise this path.

### What does NOT change

- Anchor layer (EVM tx, qTSA call): still anchors the Merkle root.
- `/trees/*` response shape: still `{revisions, file_index}`.
- DID isolation logic: unchanged.
- Signature over the witness revision: still EIP-191 over canonical JSON.
- Old witnesses minted before this change: valid forever under old
  template hash. Consumers route on template link.

## Key constraint

**SDK is authoritative over spec.** The project rule (see `CLAUDE.md`
"Hard requirements") says `aqua-rs-sdk` wins where it disagrees with
`aqua-spec`. The template change must land in the SDK first. Do not
attempt to bypass `additionalProperties: false` with local hacks.

## Files to read before starting

1. `docs/attack_vector_merkle_proof_sibling_leak.md` (this finding)
2. `docs/success-criteria.md` (project contract, hard requirements)
3. `crates/aqua-timestamp-core/src/witness.rs` (current minter)
4. `crates/aqua-timestamp-core/src/sealer.rs` (current sealer)
5. `~/aqua-rs-sdk/src/schema/templates/timestamp_evm.json` (current schema)
6. `~/aqua-rs-sdk/src/schema/templates/timestamp_evm.rs` (current Rust struct)

## Test strategy

- Unit test: build a 4-leaf shielded tree, verify each proof, confirm no
  sibling matches any raw leaf hash.
- Integration test: full round-trip (submit, seal, fetch witness, verify
  shielded proof).
- Regression: existing unshielded witnesses must still deserialize and
  verify under the old template link.
- Negative: an attacker who knows H_other but not nonce_other cannot
  match it against any sibling in their own proof.

## Risk

Low. The change is additive (new template variant, not modification of
existing). Old witnesses are unaffected. The only coordination point is
the SDK template change landing before the service-side changes.
