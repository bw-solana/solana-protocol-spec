# 9. Shreds, erasure coding, and block reconstruction

## 9.1 Mainnet shred family

[SOL-SHD-0001] A Mainnet receiver **MUST** accept only the structurally valid
chained-Merkle data and coding variants in this chapter and **MUST** reject every other
shred variant byte.

Mainnet FEC sets contain exactly 32 data shreds and 32 coding shreds.

[SOL-SHD-0002] Every FEC set **MUST** contain logical positions 0 through 31 for data
shreds and 32 through 63 for coding shreds; its `fec_set_index` **MUST** be a multiple
of 32 and a data shred's index **MUST** lie in `[fec_set_index,fec_set_index+32)`.

## 9.2 Header layout

Every shred starts with this 83-byte common header:

**Normative table.**

| Byte range | Field |
|---|---|
| `[0,64)` | leader signature |
| `[64,65)` | variant |
| `[65,73)` | slot `u64` LE |
| `[73,77)` | index `u32` LE |
| `[77,79)` | shred version `u16` LE |
| `[79,83)` | FEC-set index `u32` LE |

The data-header suffix is:

**Normative table.**

| Byte range | Field |
|---|---|
| `[83,85)` | `parent_offset:u16` LE |
| `[85,86)` | flags byte |
| `[86,88)` | `size:u16` LE |

The coding-header suffix is:

**Normative table.**

| Byte range | Field |
|---|---|
| `[83,85)` | `data_count:u16` LE |
| `[85,87)` | `code_count:u16` LE |
| `[87,89)` | `position:u16` LE |

[SOL-SHD-0003] A shred encoder and decoder **MUST** use the offsets above; all
multi-byte header integers **MUST** be little-endian.

[SOL-SHD-0004] The active fixed-set coding header **MUST** have `data_count == 32`,
`code_count == 32`, and `position` in `[0,32)`; its logical leaf position **MUST** be
`32 + position`.

The low six data-flag bits are `reference_tick`.  Bit `0x40` is DATA_COMPLETE.  The
LAST_IN_SLOT value is `0xc0` and therefore includes DATA_COMPLETE; `0x80` without `0x40`
is invalid.

[SOL-SHD-0005] A data shred marked LAST_IN_SLOT **MUST** also be marked DATA_COMPLETE,
and a DATA_COMPLETE index **MUST** equal `fec_set_index + 31`; a LAST_IN_SLOT index plus
one **MUST** be divisible by 32.

[SOL-SHD-0006] Except for `(slot,parent) == (0,0)`, a data-shred parent **MUST** equal
`slot - parent_offset`, `parent_offset` **MUST** be nonzero, and the resulting parent
**MUST** be less than the slot and not less than the receiver's rooted slot. In epoch
`E`, both data- and coding-shred indices **MUST** be less than `16,384`; consequently a
slot contains at most `512` 32+32 FEC sets.

## 9.3 Payload and active variants

The fixed 64-leaf tree has six 20-byte proof entries.  Active bytes are:

**Normative table.**

| Variant byte | Meaning | Length |
|---:|---|---:|
| `0x96` | chained data | 1203 |
| `0x66` | chained coding | 1228 |
| `0xb6` | chained data with retransmitter trailer | 1203 |
| `0x76` | chained coding with retransmitter trailer | 1228 |

[SOL-SHD-0007] The variant's low nibble **MUST** equal the Merkle-proof entry count, and
for the 32+32 set it **MUST** equal 6; a Turbine payload **MUST** have the exact length
shown in the table.

For proof count `p` and `r` equal to 64 for a retransmitter trailer or zero otherwise, a
data shred uses these regions:

**Normative table.**

| Region | Byte range / calculation | Length |
|---|---|---:|
| data header | `[0,88)` | 88 |
| ledger-data capacity | `[88, 88 + capacity)` | `capacity = 1203 - 88 - 32 - 20p - r` |
| chained root | `[chain_offset, proof_offset)` where `chain_offset = 88 + capacity` | 32 |
| Merkle proof | `[proof_offset, proof_offset + 20p)` where `proof_offset = chain_offset + 32` | `20p` |
| retransmitter trailer | final region, when present | `r` |

The coding shred uses header size 89 and total size 1228 in the same formulas. At `p=6`,
data capacity is 963 bytes without a trailer and 899 with one; the Reed-Solomon shard
size is respectively 987 or 923 bytes.

[SOL-SHD-0008] A data shred's `size` **MUST** be in `[88,88+capacity]`, and its
ledger-data bytes **MUST** be exactly `[88,size)`; bytes from `size` to the chain root
are padding and **MUST NOT** enter reconstructed ledger data.

[SOL-SHD-0009] The 32 bytes at `chain_offset` **MUST** be the preceding FEC set's full
Merkle root in the chain selected for the slot; the first set's incoming root **MUST**
be the terminal chained root accepted for the parent block. Conflicting roots identify
distinct candidate versions and **MUST NOT** be combined.

## 9.4 Shred Merkle authentication

For each shred, define:

**Normative algorithm.**

```text
leaf = SHA256(0x00 || "SOLANA_MERKLE_SHREDS_LEAF" || payload[64:proof_offset])
node = SHA256(0x01 || "SOLANA_MERKLE_SHREDS_NODE" || left[0:20] || right[0:20])
```

The ASCII suffixes contain 25 bytes after their one-byte domain prefix.  At an odd tree
level the last node is paired with itself.  A proof entry is the first 20 bytes of the
sibling hash, from leaf level upward.

[SOL-SHD-0010] A receiver **MUST** derive the leaf index as `data.index - fec_set_index`
for data or `32 + coding.position` for coding, fold the six proof entries according to
the index bits, and reject unless the fold ends at one root with no remaining high index
bits.

[SOL-SHD-0011] The 64-byte common-header signature **MUST** verify under the scheduled
leader key for the shred's slot over the 32 raw bytes of the reconstructed full Merkle
root.

[SOL-SHD-0012] All shreds combined into one FEC set **MUST** agree on slot, FEC-set
index, leader signature, variant family, counts, proof depth, chain root, and
reconstructed Merkle root; a receiver **MUST NOT** combine shards from inconsistent
sets.

[SOL-SHD-0013] In a resigned variant, the final 64 bytes **MUST** be excluded from the
leader Merkle leaf and Reed-Solomon shard and carried as the retransmitter-signature
trailer. The trailer is not an acceptance predicate in epoch `E` and **MUST NOT**
replace the leader-signature check.

## 9.5 Reed-Solomon recovery

The erasure inputs are 64 equal-length byte columns over `GF(2^8)`. Field addition is
XOR. Multiplication is polynomial multiplication modulo `x^8 + x^4 + x^3 + x^2 + 1`
(`0x11d`), with `2` as the generator of the 255 nonzero elements. Data shard `i` is
payload `[64,chain_offset)` from data shred `fec_set_index+i`.  Coding shard `j` is
payload `[89,chain_offset)` from coding position `j`.  Thus data common/data headers
after the leader signature are protected; coding headers and all Merkle proofs are not.

Define a `64 x 32` Vandermonde matrix `V` by `V[r,c] = r^c` in that field, including
`0^0 = 1`. Let `T = V[0..32,0..32]` and let the systematic generator matrix be
`G = V * inverse(T)`. Thus rows `0..31` of `G` are the identity and row `32+j` produces
coding shard `j`. Each output byte position is the field dot product of its generator
row with the 32 data bytes at that position.

[SOL-SHD-0014] Encoding and recovery **MUST** use generator matrix `G`, with data
positions followed by coding positions in the order above. Recovery **MUST** fail with
fewer than 32 valid, mutually consistent shards; otherwise it **MUST** invert the 32-row
submatrix selected by any 32 available logical positions to recover the original 32 data
shards. Shard ordering **MUST NOT** vary.

[SOL-SHD-0015] After Reed-Solomon recovery, each recovered data header **MUST** be
decoded and sanitized, and every recovered shard's leaf, proof, root, and leader
signature **MUST** be regenerated or verified before its ledger bytes are used.

## 9.6 Deshredding

[SOL-SHD-0016] To deshred one ledger component, a receiver **MUST** order data shreds by
increasing index, require consecutive indices, append only each shred's `[88,size)`
bytes, stop at the first DATA_COMPLETE shred, and reject a missing DATA_COMPLETE marker
or any trailing shred in that component.

[SOL-SHD-0017] A slot **MUST** be considered shred-complete only when data indices begin
at zero, are consecutive through a unique LAST_IN_SLOT shred, and every component
boundary needed to decode the concatenated Entry batches is present. A zero-entry
component or unconsumed bytes after a decoded entry batch **MUST** invalidate the
candidate; padding outside each data shred's declared `size` never enters the deshredded
component.

An ordinary DATA_COMPLETE component is independently framed as exactly one
`ENTRY-E1072` outer sequence:

**Normative table. Deshredded component framing.**

| Offset/order | Field | Exact encoding and constraint |
|---:|---|---|
| 0 | `entry_count` | `u64` little-endian; `1..=component_bytes/48` in epoch `E` |
| 8 | Entries | exactly `entry_count` consecutive Section 8.3 Entry records |
| end | trailing bytes | none |

The active fixed-FEC rule permits DATA_COMPLETE only on the final data shred of each
32-data-shred FEC set; LAST_IN_SLOT implies DATA_COMPLETE. Therefore a slot has at most
`512` components. Entry and transaction fields cannot continue into another component.
The combined serialized Entry batches in a slot **MUST NOT** exceed `10,485,760` bytes.

An eight-byte zero count selects a block marker rather than an empty Entry batch. Every
marker has the following exact preamble:

**Normative table. Block-marker preamble.**

| Byte range | Field | Accepted value |
|---|---|---|
| `[0,8)` | Entry count | `LE(0,8)` |
| `[8,10)` | marker version | `LE(1,2)` |
| `[10,11)` | marker kind | `0` footer, `1` header, `2` update-parent, `3` genesis certificate |
| `[11,13)` | body byte length | `u16` little-endian |
| `[13,13+length)` | marker body | exactly `length` bytes |

The first body byte is the selected marker's version and is exactly `1`. Header and
update-parent bodies are each 41 bytes: the version byte, a `u64` little-endian parent
slot, and a 32-byte parent block ID. The footer body is:

**Normative table. BlockFooterV1 body.**

| Order | Field | Exact encoding |
|---:|---|---|
| 1 | footer version | `u8 = 1` |
| 2 | bank hash | 32 bytes |
| 3 | block-producer time | `u64` little-endian nanoseconds |
| 4 | user-agent length | `u8` |
| 5 | user-agent | the declared `0..=255` bytes |
| 6 | finalization-certificate option | `u8` tag `0` or `1`, followed by the value when `1` |
| 7 | skip-reward-certificate option | `u8` tag `0` or `1`, followed by the value when `1` |
| 8 | notar-reward-certificate option | `u8` tag `0` or `1`, followed by the value when `1` |

**Normative table. Footer certificate records.**

| Record | Exact fields in order |
|---|---|
| votes aggregate | compressed BLS signature, 96 bytes; bitmap byte length, `u16` LE; signer bitmap |
| finalization certificate | slot, `u64` LE; block ID, 32 bytes; final votes aggregate; notar aggregate option tag `u8`, then aggregate when `1` |
| skip reward certificate | slot, `u64` LE; compressed BLS signature, 96 bytes; `ShortU16` bitmap byte length; signer bitmap |
| notar reward certificate | slot, `u64` LE; block ID, 32 bytes; compressed BLS signature, 96 bytes; `ShortU16` bitmap byte length; signer bitmap |

A signer bitmap is version byte `0`, bit count as `u16` little-endian, then exactly
`ceil(bit_count/8)` bytes. Bits are packed least-significant bit first and identify
validators by the epoch rank map. Unused high bits in the last byte are zero.

[SOL-SHD-0018] The component sequence **MUST** be one header, zero or more nonempty
Entry batches, one footer, and one terminal alpentick Entry batch, in that order. The
header is the first component; the footer is followed immediately by a batch containing
exactly the one tick described in Section 8.4; and that batch is the last component.
No component may follow it. Each component **MUST** consume its bytes exactly.

[SOL-SHD-0019] The header parent slot **MUST** equal the selected bank parent and its
parent block ID **MUST** equal that parent's Section 14.5 block ID. The footer timestamp
must fit `i64` and lie inclusively in
`[parent_time_nanos + 1, parent_time_nanos + 2*elapsed_slot_duration_nanos]`, using
saturating arithmetic. The footer bank hash **MUST** equal the hash of the frozen bank.
An update-parent or genesis-certificate marker is structurally decodable but invalid in
the epoch-`E` steady-state profile.

[SOL-SHD-0020] Framing precedence **MUST** validate shred/FEC structure and a
DATA_COMPLETE boundary first. At a present boundary, decode the `u64` count; zero enters
the marker grammar and nonzero enters the Entry grammar. Then require exact consumption.
A truncated count, field crossing the boundary, mismatched marker length, empty ordinary
batch, or leftover byte is block-fatal. A receiver **MUST NOT** concatenate components
to repair a truncated field. Any data shred after LAST_IN_SLOT, more than `512`
components, or more than `10,485,760` Entry bytes is block-fatal.

The `InvalidBlock::*` names in this section are specification disposition codes. They
are not ledger bytes, network error codes, or names imported from any validator
implementation.
