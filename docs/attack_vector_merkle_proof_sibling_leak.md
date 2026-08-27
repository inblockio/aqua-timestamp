# Attack Vector: Merkle Proof Membership Leak

**Severity:** Low (information disclosure, membership inference)
**Component:** Epoch sealer, witness minter, inclusion proof generation
**Date identified:** 2026-05-20

## Summary

Every Merkle inclusion proof returned to a submitter necessarily contains
sibling hashes along the path from the leaf to the root. At least one of
those siblings is a raw leaf hash submitted by a different user. This is
not a content leak (Aqua hashes are opaque), but a **membership leak**:
an attacker who holds their own proof can confirm whether a suspected
hash was submitted to the same epoch.

## Mechanism

When an epoch seals, all submitted leaf hashes are sorted
lexicographically and assembled into a binary Merkle tree. For each
leaf, the inclusion proof is the list of sibling nodes from the leaf
to the root.

Example with 4 leaves in one epoch:

```
         Root
        /    \
      N_12    N_34
      / \     / \
    H_1 H_2 H_3 H_4
```

- H_1's proof: `[H_2, N_34]`
- H_2's proof: `[H_1, N_34]`
- H_3's proof: `[H_4, N_12]`
- H_4's proof: `[H_3, N_12]`

H_1's owner learns H_2 (the raw hash their direct sibling submitted).
This is the membership leak: they now know H_2 was submitted in this
epoch, even though DID isolation prevents them from learning who
submitted it.

## What leaks per proof

| Property                         | Exposed? | Detail                                                        |
|----------------------------------|----------|---------------------------------------------------------------|
| Direct sibling leaf hash         | Yes      | Exactly one raw leaf hash from another submitter              |
| Internal sibling nodes           | Yes      | `log2(n) - 1` hashes-of-hashes; not raw leaves but deterministic |
| Epoch size                       | Yes      | `batch_tree_size` in every witness payload                    |
| Leaf position in sorted tree     | Yes      | `batch_leaf_index` in every witness payload                   |
| Which DID submitted which leaf   | No       | DID isolation at the API layer prevents cross-user enumeration |
| Content behind the hashed leaf   | No       | Hashes are opaque revision hashes; content is not recoverable  |

## What DID isolation prevents

The API enforces ownership checks on every witness query
(`routes.rs` returns 403 for leaves owned by a different DID). Users
cannot enumerate which leaves belong to whom, cannot fetch another
user's witness, and cannot list leaves in an epoch that are not their
own.

But anyone holding their own proof can extract the sibling leaf hash
directly from the proof array and use it to confirm membership.

## Exploitation scenarios

1. **Membership confirmation.** An attacker who suspects a specific hash
   was submitted to the service can check it against sibling nodes in
   their own proof. If the suspected hash matches a sibling, the
   attacker confirms that hash was submitted in the same epoch.

2. **Correlation across epochs.** If the same attacker submits leaves
   across many epochs, they accumulate sibling hashes over time.
   Combined with `batch_tree_size` and `batch_leaf_index`, this builds
   a partial view of submission activity without knowing the submitters.

3. **Collusion.** Two users who share their proofs from the same epoch
   can reconstruct more of the tree than either could alone. With enough
   colluding participants, the full leaf set is recoverable.

## Mitigating factors

- Leaf hashes are opaque Aqua revision hashes. Knowing a neighbor's hash
  does not reveal the underlying document content.
- The sorted order is lexicographic on the hash bytes, which is
  effectively random. An attacker cannot choose their position in the
  tree.
- DID isolation means the attacker cannot map leaked hashes to specific
  users or DIDs without out-of-band information.

## Approved mitigation: per-leaf membership shielding

The goal is not to protect the submitter's own leaf (that hash is not
secret). The goal is to prevent proof siblings from being recognizable
as other users' raw hashes.

**Approach:** before inserting a leaf into the Merkle tree, compute a
shielded value:

```
S = sha3(leaf || nonce)
```

where `nonce` is a random 32-byte value generated server-side per leaf.
The Merkle tree is built over shielded values `S_1, S_2, ...` instead
of raw hashes. The nonce is returned privately to the submitter and
stored in their witness payload.

**Effect on proofs:**

```
         Root
        /    \
      N_12    N_34
      / \     / \
    S_1 S_2 S_3 S_4       <-- shielded values, not raw hashes
```

H_1's owner receives proof `[S_2, N_34]`. S_2 is `sha3(H_2 || nonce_2)`,
which is useless without `nonce_2` (held only by H_2's owner). The
membership leak is closed: the attacker cannot confirm whether a
suspected hash H_x matches S_2 without knowing nonce_2.

**Verification flow:**

1. Submitter holds: `leaf`, `nonce`, `merkle_proof`, `merkle_root`.
2. Submitter (or any verifier given the witness payload) computes
   `S = sha3(leaf || nonce)`.
3. Walks the proof: `verify(S, index, siblings, root)`.
4. Checks that `root` matches the on-chain anchor.

The witness payload is self-contained: it carries the nonce, so any
holder of the full witness revision can verify independently. But an
observer who only sees another user's shielded sibling cannot reverse it.

**What changes where:**

| Layer | Change |
|-------|--------|
| **aqua-rs-sdk** (upstream) | New template variant (`timestamp_evm_v2` / `timestamp_tsa_v2`) adding an optional `shielding_nonce` field. New templates needed because existing schemas set `additionalProperties: false` and their template hashes are content-addressed. |
| **aqua-rs-sdk** (upstream) | Rust payload structs gain `shielding_nonce: Option<String>`. |
| **aqua-timestamp accumulator** | Generate random 32-byte nonce per leaf at submission time; store alongside `LeafEntry`. |
| **aqua-timestamp sealer** | Build Merkle tree over `sha3(leaf \|\| nonce)` instead of raw `leaf`. |
| **aqua-timestamp witness minter** | Populate `shielding_nonce` in the payload; compute proof against shielded tree. |
| **aqua-timestamp submission response** | Return nonce to submitter on `POST /v1/leaves` (their secret for this leaf). |
| **Verification clients** | Must compute `sha3(leaf \|\| nonce)` before walking the proof. |
| **Anchor layer** | No change. EVM tx and qTSA call still anchor the Merkle root. |
| **DID isolation** | No change. |
| **Existing witnesses** | Unaffected. Old template hash identifies unshielded proofs; new template hash identifies shielded ones. Both are valid. |

## Status

Attack vector documented. Mitigation approach approved. Implementation
requires an upstream SDK template change (new template variant with
`shielding_nonce` field) before the local accumulator/minter changes.
