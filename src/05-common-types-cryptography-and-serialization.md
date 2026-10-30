# 5. Common types, cryptography, and serialization

## 5.1 Scope and notation

This chapter defines the byte-level primitives used by the validator wire and ledger
chapters.  It is intentionally a small, protocol-specific subset of the libraries from
which the behavior was recovered.  Presentation encodings such as base58 are not wire
encodings unless another chapter explicitly says so.

## 5.2 SHA-256 and Ed25519

`SHA256(x)` below means FIPS 180-4 SHA-256 over the exact byte string `x`.  It does not
include a length, terminator, or serialization wrapper.

[SOL-GEN-0003] A protocol hash written `SHA256(x1 || ... || xn)` **MUST** feed the
listed byte strings to one SHA-256 invocation in the listed order and **MUST NOT** hash
a textual rendering of any operand.

[SOL-GEN-0004] Verification of a signed gossip value, gossip ping or pong, signed repair
request, or leader-signed shred **MUST** reject a malformed public key or signature, an
`S` scalar greater than or equal to the Ed25519 group order, a small-order public key or
`R` point, or a signature that fails the Ed25519 verification equation. A receiver
**MUST NOT** broaden this strict Mainnet rule just because its cryptographic library
accepts additional noncanonical compressed Edwards encodings.

[SOL-GEN-0005] A signature **MUST** be verified against the signer, message bytes, and
domain construction named by the enclosing protocol; a valid signature for a different
field serialization or domain **MUST NOT** authenticate the object.

## 5.3 Codec registry and canonical fixed-integer serialization

A reference to a serialization library, including “bincode” or “wincode,” is not
sufficiently precise for a protocol boundary. This specification therefore assigns a
named codec to every reusable protocol-object family with a byte grammar and defines the
accepted bytes directly. An implementation may use any serializer that produces and
accepts exactly these bytes; a library name, version, default configuration, or
host-language data layout never overrides this section or an enclosing type table.

A codec may compose another codec. For example, a repair response can contain one raw
shred, and a ledger component can contain transactions inside Entries. Transport state
machines, erasure-code mathematics, cryptographic preimage recipes, and raw fixed-size
keys, signatures, or hashes are not separate codecs just because their algorithms
consume bytes. Their enclosing section remains authoritative.

**Normative table. Codec registry.**

| Codec | Surfaces | Controlling rules |
|---|---|---|
| `FIXINT-LE` | Primitive fields and containers in an enclosing codec that explicitly selects canonical fixed-integer serialization | This section's primitive table plus the enclosing field table. Counts are `u64`; enum tags are `u32`; no padding is implicit. |
| `SHORT-U16` | Transaction counts, instruction counts and explicitly named compact sequences | Section 5.4 only. |
| `ULEB128` | ContactInfo version, wallclock, and other explicitly named unsigned compact fields wider than `ShortU16` | Section 5.4. The declared integer width bounds the group count; overflow and nonminimal final zero groups are rejected. |
| `CRDS-E1072` | Gossip datagrams, `CrdsValue`, ContactInfo and nested CRDS types | Sections 6.2–6.5. This is a closed schema composing `FIXINT-LE`, `SHORT-U16`, and `ULEB128`; zero-byte deprecated variants and field-specific bounds are exactly as stated there. |
| `REPAIR-E1072` | Repair requests, ordinary shred responses, repair ping challenges, ancestor-hash responses, and block-ID repair | Section 6.7. Requests and specialized responses use their stated discriminants and fields; an ordinary response is one `SHRED-E1072` followed by a `u32` nonce. |
| `ALPENGLOW-WIRE-E1072` | Alpenglow votes and certificates carried in QUIC datagrams | Sections 6.6 and 15.2. The codec is the exact Wincode subset stated there, including explicit enum tags and length encodings. |
| `ALPENGLOW-STATE-E1072` | Epoch inflation account state used by Alpenglow vote rewards | Section 16.6.2. This is the exact fixed-integer Wincode record stated there, including the one-byte previous-state option tag. |
| `TX-E1072` | Legacy, version-0, and version-1 transactions and messages | Sections 7.1–7.4. Legacy and v0 compose `SHORT-U16`; v1 uses its bespoke fixed-count layout and places signatures after the message. |
| `ENTRY-E1072` | Deshredded ledger components, Alpenglow block markers, and Entries | Sections 8.3 and 9.6. |
| `SHRED-E1072` | Raw Turbine shreds and shreds embedded in repair responses | Sections 9.1–9.5 and 14.6. This is a fixed-offset, variant-dependent format with exact lengths, padding, Merkle coverage, and Reed–Solomon regions; it is not a general-purpose struct serialization. |
| `SBF-ELF-E1072` | Deployed SBF ELF objects and their eight-byte instruction words | Sections 12.8.1, 12.8.3, and 12.8.5. ELF acceptance and instruction decoding depend on the selected active SBF version. |
| `SBF-ABI-E1072` | Loader ABI-v0 and ABI-v1 program inputs and outputs, plus syscall guest-memory structures | Sections 12.8.6–12.8.7. The loader selects v0 or v1; syscall structures use their exact field order, widths, padding, alignment, and access rules. |
| `PRECOMPILE-E1072` | Ed25519, secp256k1, and secp256r1 signature-verification instruction data | Section 12.6. Prefixes, offset records, referenced byte ranges, and program-specific key and signature encodings are exact. |
| `PROGRAM-E1072` | Compute Budget, System, Vote, Stake, Config, Feature, ALT, loader, and ZK ElGamal instructions and account states | Sections 7.6, 13.3, and 17.1. Each program's byte tables control; `FIXINT-LE` applies only where explicitly selected, and bespoke one-byte and POD layouts remain bespoke. |
| `SYSVAR-E1072` | Persistent sysvar account data and the synthesized Instructions sysvar | Sections 13.4 and 13.6. Persistent sysvars compose `FIXINT-LE` where stated; the Instructions sysvar uses its own `u16`-offset framing. |

[SOL-GEN-0006] A protocol object belonging to a registered surface **MUST** be encoded
with the assigned enclosing codec and every nested codec it names. A receiver
**MUST NOT** retry a failed decode with another serializer configuration, accept a
variable-integer substitute for a fixed integer, or derive field order from reflection
metadata.

[SOL-GEN-0007] A value described as using canonical fixed-integer serialization **MUST**
use the applicable layout below. Fields are serialized in the order in which the
enclosing table lists them, with no alignment, padding, field names, or end marker
unless that table explicitly includes one.

**Normative table. `FIXINT-LE` primitive layouts.**

| Value kind | Exact byte layout |
|---|---|
| `u8`, `u16`, `u32`, `u64`, `u128` | Exactly `1`, `2`, `4`, `8`, or `16` bytes, least-significant byte first |
| `i8`, `i16`, `i32`, `i64`, `i128` | Two's-complement bit pattern of the stated width, least-significant byte first |
| `f32`, `f64` | IEEE-754 binary32 or binary64 bit pattern encoded as a little-endian `u32` or `u64`; no numeric text conversion |
| Boolean | One byte: `00` for false or `01` for true; every other value is invalid |
| unit / zero-sized marker | Zero bytes |
| tuple or struct | Encodings of its fields concatenated in the declared table order; no field count or tags |
| enum variant | Variant discriminant as `u32` little-endian, followed immediately by that variant's fields |
| sequence of `length` elements | `LE(length,8)`, followed by exactly `length` element encodings |
| byte string | `LE(length,8)`, followed by exactly `length` raw bytes |
| UTF-8 string | `LE(length,8)`, followed by exactly `length` bytes of valid UTF-8; no terminator |
| `Option::None` | One tag byte `00` |
| `Option::Some(value)` | One tag byte `01`, followed by the value encoding |
| fixed-size array | Exactly the consecutive element encodings; no length prefix |
| ordered map | `LE(count,8)`, followed by exactly `count` key/value pairs in the order required by the enclosing type |

An enclosing byte-layout table may deliberately specify a different tag, length width,
compact integer, conditional field, padding region, or trailing capacity. That more
specific table controls that type. A map without an enclosing key order has no canonical
protocol encoding; hash-table iteration order is never an encoding rule.

[SOL-GEN-0008] A decoder for a datagram or a complete QUIC application message **MUST**
consume the entire application byte string and **MUST** reject a truncated field, an
impossible length, integer overflow, or trailing bytes not assigned by that message
format.

The exception to exact consumption is an enclosing format that gives an explicit length
or expressly permits unused capacity or trailing bytes, such as a data shred's `size`
field, the four-byte repair nonce after a shred, a Vote account, or a program
instruction whose section says that its decoder permits a suffix.

## 5.4 `ShortU16` and compact integers

`ShortU16` is an unsigned, least-significant-group-first base-128 integer.  The low
seven bits of each byte carry data and bit 7 indicates another byte.

**Normative table. `SHORT-U16` canonical ranges.**

| Encoded length | Canonical value range | Byte constraints |
|---:|---:|---|
| 1 byte | `0..127` | byte 0 continuation bit is clear |
| 2 bytes | `128..16383` | byte 0 continuation bit is set; byte 1 continuation bit is clear and its value group is nonzero |
| 3 bytes | `16384..65535` | bytes 0 and 1 continuation bits are set; byte 2 continuation bit is clear, its value group is nonzero, and only its low two bits may be set |

[SOL-GEN-0009] A canonical `ShortU16` **MUST** encode values 0 through 127 in one byte,
128 through 16383 in two bytes, and 16384 through 65535 in three bytes; a sender
**MUST** use the shortest form.

[SOL-GEN-0010] A `ShortU16` receiver **MUST** reject a fourth byte, continuation on the
third byte, nonzero bits above bit 15, and an overlong encoding whose last seven-bit
group is zero.

The ContactInfo version and wallclock fields use the same unsigned base-128 rule at
their stated `u16` or `u64` width rather than the three-byte `ShortU16` cap.

[SOL-GEN-0011] An unsigned LEB128 field **MUST** terminate within the number of
seven-bit groups required by its declared width and **MUST** reject overflow and a
non-minimal final zero group.

## 5.5 Program-derived addresses

Although program-derived addresses are most visible during execution, they are a common
32-byte identity primitive and are fixed here once.

[SOL-GEN-0012] `create_program_address(seeds, program_id)` **MUST** reject more than 16
seeds or any seed longer than 32 bytes, then compute
`SHA256(seed[0] || ... || seed[n-1] || program_id || "ProgramDerivedAddress")`, where
the marker is the 21 ASCII bytes and `program_id` is 32 raw bytes.

[SOL-GEN-0013] The digest from `create_program_address` **MUST** be rejected if it
decompresses as an Ed25519 curve point and otherwise **MUST** be returned as the 32-byte
address.

[SOL-GEN-0014] `find_program_address` **MUST** append a one-byte bump seed to the
caller's seed list, try bump values from 255 down through 1, and return the first
address accepted by `create_program_address` together with that bump. If none of those
255 values succeeds, the search has no result; bump zero is not tested by the active
search operation. Because the appended bump counts as a seed, a caller list of 16 seeds
cannot produce a result under the 16-seed `create_program_address` ceiling.

## 5.6 Resource-safe decoding

[SOL-GEN-0015] Before allocating or advancing by a peer-supplied count or length, a
receiver **MUST** prove that the operation fits both the remaining input and the
protocol-specific bound; allocation failure or host-width conversion **MUST NOT** change
acceptance into partial decoding.
