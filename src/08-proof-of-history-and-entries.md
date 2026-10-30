# 8. Proof of History and entries

## 8.1 PoH operations

Let `H` be the current 32-byte PoH state and `M` a 32-byte mixin.

[SOL-POH-0001] A PoH append step **MUST** replace `H` with `SHA256(H)`, and `append(n)`
**MUST** apply exactly `n` such sequential steps.

[SOL-POH-0002] A PoH mixin step **MUST** replace `H` with `SHA256(H || M)` and count as
one PoH hash; it **MUST NOT** first hash `M` again.

## 8.2 Transaction-signature mixin

For an entry, flatten transaction signatures in transaction order and, within each
transaction, signature-array order.  Define this Merkle tree:

* leaf `i = SHA256(0x00 || signature[i])`;
* parent `= SHA256(0x01 || left[32] || right[32])`;
* duplicate the last node when a level has an odd node count;
* the empty-list root is 32 zero bytes.

[SOL-POH-0003] A transaction-bearing entry's mixin **MUST** be the Merkle root defined
above over all of its flattened 64-byte signatures.

This signature tree is distinct from the shred Merkle tree: its prefixes are single
bytes, its parent inputs are full 32-byte hashes, and its leaves are signatures only.

## 8.3 Entry bytes and hash transition

An Entry has this canonical fixed-integer layout:

**Normative table.**

| Offset | Field |
|---:|---|
| 0 | `num_hashes:u64` |
| 8 | `hash[32]` |
| 40 | `transactions` sequence count as `u64` |
| 48 | serialized transactions in order |

A batch of entries uses this outer layout; transaction encoding is specified in Chapter
7.

**Normative table.**

| Order | Field | Encoding / length |
|---:|---|---|
| 1 | entry count | `u64` LE |
| 2 | entries | the declared number of consecutive Entry encodings |

[SOL-POH-0004] An Entry sender **MUST** encode the fields and sequence counts above
without alignment bytes, and an Entry receiver **MUST** preserve entry and transaction
order.

Given previous state `P`, count `n`, and transaction vector `T`, the expected entry hash
is:

**Normative algorithm.**

```text
if n == 0 and len(T) == 0: P
otherwise let Q = append(P, saturating_sub(n, 1))
if len(T) == 0:            SHA256(Q)
else:                      SHA256(Q || signature_merkle_root(T))
```

[SOL-POH-0005] An Entry **MUST** be accepted in the PoH chain only when its stored
`hash` equals the expected hash above using the preceding Entry hash, or the parent
block's final Entry hash for the first Entry.

[SOL-POH-0006] An Entry with an empty transaction vector **MUST** be classified as a
tick; an Entry with one or more transactions **MUST** be classified as a record even if
a malformed transaction would later fail validation.

[SOL-POH-0007] Entry PoH verification **MUST** be sequential across every reconstructed
batch in ledger order; parallel computation **MUST NOT** change which preceding hash is
supplied to an Entry.

## 8.4 Tick accounting

Alpenglow uses low-power PoH. An active bank has no `hashes_per_tick` target and a block
contains one terminal tick, called the alpentick. Every Entry advances the hash chain by
one hash regardless of whether it contains transactions. The alpentick is the only empty
transaction Entry permitted as a tick and is the final ledger component after the
footer.

**Normative table. Epoch-`E` PoH and tick parameters.**

| Parameter | Active value |
|---|---:|
| target slot duration | `200,000,000` nanoseconds |
| `hashes_per_tick` | absent |
| inherited bank tick-height interval | `64` ticks/slot |
| initial tick height of a replayed Alpenglow bank | `max_tick_height - 1` |
| tick Entries serialized in one block | exactly `1`, the terminal alpentick |
| Entry `num_hashes` | exactly `1` |

The inherited 64-tick interval remains part of bank tick-height arithmetic. Alpenglow
does not serialize 64 tick Entries: replay initializes the bank one tick below its
maximum, and the single alpentick reaches that maximum.

[SOL-POH-0008] Every Entry in an epoch-`E` block **MUST** have `num_hashes == 1` and
**MUST** satisfy the sequential hash transition in Section 8.3. Absence of a
`hashes_per_tick` target disables only the legacy accumulated-hash-count comparison; it
does not disable Entry hashing.

[SOL-POH-0009] A valid block **MUST** contain exactly one tick Entry. It **MUST** have
an empty transaction vector, follow the BlockFooter component immediately, and be the
last Entry and last component in the block. A transaction Entry, footer, marker, or
additional tick after it is block-fatal.

[SOL-POH-0010] Replay **MUST** increment the bank tick height exactly once for the
terminal alpentick and require the resulting height to equal the bank maximum tick
height. It **MUST NOT** synthesize legacy ticks or substitute a local PoH profile.
