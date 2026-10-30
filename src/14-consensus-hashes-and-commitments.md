# 14. Consensus hashes and commitments

## 14.1 Hash-function notation

`SHA256(x)` has its single authoritative definition in Section 5.2. `BLAKE3-256(x)` is
the first 32 bytes of the unkeyed BLAKE3 output. `BLAKE3-XOF(x,2048)` is the first 2,048
bytes from the unkeyed BLAKE3 extendable-output function. Hash objects are their 32 raw
bytes; base58 text is display encoding and never part of a preimage unless explicitly
stated.

[SOL-HSH-0001] A component **MUST** hash the exact raw byte preimage named by the
applicable commitment and **MUST NOT** hash a base58, JSON, debug, or host-structure
representation in its place.

## 14.2 Catalog

**Normative table.**

| Commitment | Algorithm and preimage | Ordering/domain | Primary use |
|---|---|---|---|
| Mainnet genesis hash | fixed bytes `45296998a6f8e2a784db5d9f95e18fc23f70441a1039446801089879b08c7ef0` (display form `5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d`) | No runtime serialization step | Chain identity and initial recent blockhash |
| Transaction message hash | `BLAKE3-256("solana-tx-message-v1" \|\| serialized_message)` | Domain literal is 20 ASCII bytes; message bytes are Chapter 7 | Replay-protection key and entry verification support |
| PoH step/tick | `SHA256(previous_hash)` | Repeated sequentially | Time/order chain |
| PoH record | `SHA256(previous_hash \|\| signature_merkle_root)` | Previous first | Commits transaction signatures into entry chain |
| Signature Merkle tree | Section 8.2 | Transaction order, then signature order | PoH record mixin |
| Account lattice element | `BLAKE3-XOF(LE(lamports,8) \|\| data \|\| executable_u8 \|\| owner \|\| address,2048)` | Zero-lamport accounts map to identity | Accounts-state accumulator |
| Accounts lattice hash | 1,024 component-wise sums of `u16` elements modulo `2^16` | Order-independent multiset; each `u16` is little-endian | Bank-hash state commitment |
| Alpenglow block ID | Section 14.5 | FEC roots in ledger order, then parent commitment | Consensus vote, certificate, ancestry, and repair identity |
| Bank hash | Section 14.5 | Parent/order defined below | Vote/state identity |
| Terminal FEC root | Full 32-byte Merkle root of the slot's final FEC set; Chapter 9 | Terminal root of the validated chained sequence | Shred-chain and repair-version identity |
| Shred Merkle nodes and signatures | Chapter 9 | Variant-dependent | Shred authenticity and FEC-set commitment |
| Shred version | Section 14.6 | Genesis plus configured hard-fork list | Network partition identifier |

## 14.3 Transaction and entry commitments

[SOL-HSH-0002] The transaction message hash **MUST** be
`BLAKE3-256(0x736f6c616e612d74782d6d6573736167652d7631 || M)`, where `M` is the
canonical serialized message without its transaction signature-vector prefix.

The flattened-signature order, empty root, leaf domain, parent domain, and odd node rule
are defined only in Section 8.2 and are incorporated here by reference.

[SOL-HSH-0003] A transaction-bearing entry's final PoH step **MUST** mix the signature
Merkle root exactly once as `SHA256(previous || root)`; a tick's final step **MUST** be
`SHA256(previous)` with no mixin.

## 14.4 Accounts lattice hash

For nonzero-lamport account `(address,lamports,data,executable,owner)`, compute 2,048
BLAKE3 XOF bytes over the catalogued preimage, interpret each consecutive pair as a
little-endian `u16`, and obtain vector `L[0..1024)`. A zero-lamport account contributes
the all-zero vector. The full state accumulator is the component-wise wrapping sum of
exactly the latest visible nonzero account version for every address.

[SOL-HSH-0004] Replacing account version `old` with `new` **MUST** update the
accumulator by component-wise `A := A - L(old) + L(new)` modulo `2^16`; absence and a
zero-lamport version both contribute the identity.

[SOL-HSH-0005] The 2,048-byte lattice-hash serialization **MUST** encode accumulator
components in index order as little-endian `u16` values.

## 14.5 Block ID and bank hash

Let `R[0..n)` be the full 32-byte Merkle roots of the block's FEC sets in increasing
`fec_set_index` order, where `n = final_data_shred_index / 32 + 1`. Let `ps` and `pb`
be the header's parent slot and parent block ID. Append this already-hashed parent leaf:

**Normative algorithm.**

```text
R[n] = SHA256(LE(ps,8) || pb || LE(n,4))
```

Reduce the `n+1` 32-byte leaves as a binary Merkle tree. Leaves are used directly; they
do not receive another leaf-domain hash. At each level, duplicate an unpaired final
node, and calculate each parent as:

**Normative algorithm.**

```text
SHA256(0x01 || "SOLANA_MERKLE_SHREDS_NODE" || left[0:20] || right[0:20])
```

The final 32-byte root is the block ID.

[SOL-HSH-0009] Block-ID calculation **MUST** include every FEC root exactly once in
ledger order and the parent leaf exactly once at the end. It **MUST** use `n` as a
little-endian `u32`, the 20-byte child truncation, and odd-node duplication above.

[SOL-HSH-0010] A header parent block ID, repair request, Alpenglow block vote, and
block-bearing certificate **MUST** bind this block ID. A terminal FEC root, bank hash,
or another block's ID **MUST NOT** be substituted.

The bank hash remains the execution-state commitment. Let `P` be the parent bank hash,
`N` the slot's processed signature count as `LE(N,8)`, `H` the slot's final PoH
blockhash, and `A` the 2,048-byte accounts lattice hash.

1. `x := SHA256(P || LE(N,8) || H)`.
2. `x := SHA256(x || A)`.
3. If the configured hard-fork crossing count for `(parent_slot,slot]` is nonzero, let
   `C` be that count encoded `LE(C,8)` and set `x := SHA256(x || C)`.
4. The bank hash is `x`.

The crossing count is the sum of registration multiplicities for configured hard-fork
slots `f` satisfying `parent_slot < f <= slot`. Count addition is saturating in the
configuration structure; conversion to `u64` must be representable in a valid protocol
configuration.

[SOL-HSH-0006] At freeze, the bank hash **MUST** be computed only after deferred fee
distribution, slot-history update, incinerator handling, and all account-lattice updates
for the slot have completed.

[SOL-HSH-0007] The processed signature count in the bank-hash preimage **MUST** be the
protocol-defined count accumulated for processed transactions in the slot, encoded as
unsigned little-endian 64-bit; it is not the number of successful transactions.

## 14.6 Shred version

Start `V` as the 32 raw genesis-hash bytes. For each configured hard-fork registration
in ascending slot order, with registration multiplicity `m`, set
`V := SHA256(V || LE(slot,8) || LE(m,8))`. Initialize bytes `a0=a1=0`; XOR even-position
bytes of `V` into `a0` and odd-position bytes into `a1`. Let `q = (a0 << 8) | a1` as
`u16`; the shred version is saturating `q+1`, so `0xffff` remains `0xffff`.

[SOL-HSH-0008] A validator's emitted shred version **MUST** equal the Section 14.6
result for the specification chain identity and configured hard forks. That result is
`50093` in epoch `E`; shreds with a different version **MUST** be dropped before
reconstruction.

## 14.7 Related commitments

The catalog cross-references the byte-exact shred and entry chapters to avoid duplicate
definitions. Full-accounts snapshot hashes are not vote commitments in the active
lattice-hash bank formula; snapshot verification remains required for trusted-state
restoration but archive layout is outside scope.
