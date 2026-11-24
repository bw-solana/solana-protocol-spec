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

Codec identifiers are stable names, independent of the specification's reference epoch.
They are not transmitted version fields. Actual wire-version tags and accepted layouts
are defined by the controlling sections and change only with the protocol they describe.

**Normative table. Codec registry.**

| Codec | Surfaces | Controlling rules |
|---|---|---|
| `FIXINT-LE` | Primitive fields and containers in an enclosing codec that explicitly selects canonical fixed-integer serialization | This section's primitive table plus the enclosing field table. Counts are `u64`; enum tags are `u32`; no padding is implicit. |
| `SHORT-U16` | Transaction counts, instruction counts and explicitly named compact sequences | Section 5.4 only. |
| `ULEB128` | ContactInfo version, wallclock, and other explicitly named unsigned compact fields wider than `ShortU16` | Section 5.4. The declared integer width bounds the group count; overflow and nonminimal final zero groups are rejected. |
| `CRDS` | Gossip datagrams, `CrdsValue`, ContactInfo and nested CRDS types | Sections 6.2–6.5. This is a closed schema composing `FIXINT-LE`, `SHORT-U16`, and `ULEB128`; zero-byte deprecated variants and field-specific bounds are exactly as stated there. |
| `REPAIR` | Repair requests, ordinary shred responses, repair ping challenges, ancestor-hash responses, and block-ID repair | Section 6.7. Requests and specialized responses use their stated discriminants and fields; an ordinary response is one `SHRED` followed by a `u32` nonce. |
| `ALPENGLOW-WIRE` | Alpenglow votes and certificates carried in QUIC datagrams | Sections 6.6 and 15.2. The codec is the exact Wincode subset stated there, including explicit enum tags and length encodings. |
| `ALPENGLOW-STATE` | Epoch inflation account state used by Alpenglow vote rewards | Section 16.6.2. This is the exact fixed-integer Wincode record stated there, including the one-byte previous-state option tag. |
| `TX` | Legacy, version-0, and version-1 transactions and messages | Sections 7.1–7.4. Legacy and v0 compose `SHORT-U16`; v1 uses its bespoke fixed-count layout and places signatures after the message. |
| `ENTRY` | Deshredded ledger components, Alpenglow block markers, and Entries | Sections 8.3 and 9.6. |
| `SHRED` | Raw Turbine shreds and shreds embedded in repair responses | Sections 9.1–9.5 and 14.6. This is a fixed-offset, variant-dependent format with exact lengths, padding, Merkle coverage, and Reed–Solomon regions; it is not a general-purpose struct serialization. |
| `SBF-ELF` | Deployed SBF ELF objects and their eight-byte instruction words | Sections 12.8.1, 12.8.3, and 12.8.5. ELF acceptance and instruction decoding depend on the selected active SBF version. |
| `SBF-ABI` | Loader ABI-v0 and ABI-v1 program inputs and outputs, plus syscall guest-memory structures | Sections 12.8.6–12.8.7. The loader selects v0 or v1; syscall structures use their exact field order, widths, padding, alignment, and access rules. |
| `PRECOMPILE` | Ed25519, secp256k1, and secp256r1 signature-verification instruction data | Section 12.6. Prefixes, offset records, referenced byte ranges, and program-specific key and signature encodings are exact. |
| `PROGRAM` | Compute Budget, System, Vote, Stake, Config, Feature, ALT, loader, and ZK ElGamal instructions and account states | Sections 7.6, 13.3, and 17.1. Each program's byte tables control; `FIXINT-LE` applies only where explicitly selected, and bespoke one-byte and POD layouts remain bespoke. |
| `SYSVAR` | Persistent sysvar account data and the synthesized Instructions sysvar | Sections 13.4 and 13.6. Persistent sysvars compose `FIXINT-LE` where stated; the Instructions sysvar uses its own `u16`-offset framing. |

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

## 5.7 Alpenglow BLS signatures and proof of possession

This section defines the BLS variant used by consensus, reward certificates, and Vote
Program BLS authorization. It is separate from the selectable SBF BLS syscall formats
in Section 12.8.7. Public keys are in BLS12-381 G1; signatures and proofs of possession
are in G2. Arithmetic uses these parameters.

**Normative table. BLS parameters.**

| Parameter | Exact value |
|---|---|
| base-field prime `p` | `0x1a0111ea397fe69a4b1ba7b6434bacd764774b84f38512bf6730d2a0f6b0f6241eabfffeb153ffffb9feffffffffaaab` |
| subgroup order `q` | `0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001` |
| G1 curve | `y^2 = x^3 + 4` over `Fp` |
| G2 curve | `y^2 = x^3 + 4*(1+u)` over `Fp2`, where `u^2 = -1` |
| G1 generator `g1`, compressed | `97f1d3a73197d7942695638c4fa9ac0fc3688c4f9774b905a14e3a3f171bac586c55e83ff97a1aeffb3af00adb22c6bb` |
| message hash-to-G2 | RFC 9380 Section 8.8.2 `BLS12381G2_XMD:SHA-256_SSWU_RO_`, with the signature DST below and no augmentation |
| signature DST | ASCII `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |
| proof-of-possession DST | ASCII `BLS_POP_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` |

The hash-to-curve primitive is the random-oracle construction in
[RFC 9380](https://www.rfc-editor.org/rfc/rfc9380.html#section-8.8.2): expand the exact
message with XMD/SHA-256 and the selected DST, hash to two Fp2 elements, map with the
specified simplified-SWU/isogeny map, add, and clear the cofactor. A raw SHA-256 digest,
encode-to-curve variant, omitted `_RO_`, or different DST is not interchangeable.

All coordinate limbs below are canonical 48-byte **big-endian** integers smaller than
`p`. For `a = a0 + a1*u`, the wire order is `a1 || a0`. The first byte reserves bit 7
for compression, bit 6 for infinity, and bit 5 for the compressed root-selection sign.
Remove these flags before reading that coordinate limb. For finite compressed points,
the sign selects the lexicographically larger `y` when set; Fp2 comparison considers
`y1` before `y0`. In the uncompressed coordinate form, compression and sign bits are
zero. The fixed 192-byte signature envelope also accepts the compressed-prefix form
specified below; its length alone does not select uncompressed coordinate decoding.

**Normative table. BLS encodings, with no length prefix or padding.**

| Value | Exact layout | Size |
|---|---|---:|
| compressed G1 public key | `x`, with compression bit set and sign bit selecting `y` | 48 bytes |
| compressed G2 proof of possession | `x1 || x0`, with compression/sign flags in `x1[0]` | 96 bytes |
| G2 vote or certificate signature, usual coordinate form | `x1 || x0 || y1 || y0` | 192 bytes |
| G2 vote or certificate signature, accepted compressed-prefix form | compressed G2 point in bytes `[0,96)`; bytes `[96,192)` are ignored by point decoding but remain part of the fixed wire field | 192 bytes |
| infinity, compressed | first byte `c0`, all remaining bytes zero | group's compressed size |
| infinity, uncompressed | first byte `40`, all remaining bytes zero | group's uncompressed size |

To decode a 192-byte signature, inspect byte zero: if bit 7 is set, decode only its
first 96 bytes as compressed G2; otherwise decode the complete uncompressed form.
This includes compressed infinity with an ignored 96-byte suffix. Producers normally
serialize uncompressed coordinates, but receivers must accept both forms; treating
the suffix as zero padding or rejecting a valid compressed prefix would narrow the
active verification behavior.

Reject noncanonical field limbs/flags in the selected point representation, failed
point decompression, points off the curve, and points outside the order-`q` subgroup.
Public keys must additionally be nonidentity.
The signature decoder permits the canonical identity encoding; acceptance still
requires the pairing equation below. An implementation must not replace subgroup
validation with mere curve membership or reject extra valid encodings based on its
internal point representation.

**Normative algorithm. Signature and aggregate verification.**

1. For one message `m`, decode public key `P` and signature `sigma`; require
   `e(P, HashToG2(m, signature_DST)) = e(g1, sigma)` in the order-`q` target group.
2. For a certificate with one payload, add the public keys selected by its signer map
   in G1 and apply the same equation. The signer set must be nonempty and its sum must
   be nonidentity. The keys must come from the PoP-validated Section 15.2 rank map.
3. For a base-3 certificate, partition signers into the primary and fallback payload
   groups specified in Section 15.2. Require a nonempty union and a nonidentity sum for
   each nonempty group. Verify
   `e(P_primary, HashToG2(m_primary)) * e(P_fallback, HashToG2(m_fallback))
   = e(g1, sigma)`, omitting an empty group. Verify the aggregate actually supplied;
   no individual constituent signatures are recoverable from it. Apply the certificate's
   stake threshold separately after signature verification.

Here `e` is the BLS12-381 pairing with final exponentiation; bilinearly equivalent
pairing implementations give the same equality. Aggregate signatures are sums in G2,
not concatenations or hashes. Consensus payload bytes are defined only in Section 15.2.

**Normative algorithm. Vote-account proof of possession.**

For vote account address `V[32]`, stored compressed key `K[48]`, and proof `pi[96]`,
hash the exact 89-byte message `ASCII("ALPENGLOW") || V || K` to G2 with the PoP DST.
Decode `K` and `pi` with the compressed formats above and verify
`e(K, HashToG2(message, PoP_DST)) = e(g1, pi)`. The Vote Program charges its additional
34,500 CU before verification and reports `InvalidArgument` for a malformed key/proof
or false equation. The account address binding is required; a PoP for another vote
account does not authorize the same key here. Once stored through this transition,
epoch rank construction validates the key encoding but does not rerun the PoP.

## 5.8 Ristretto proof arithmetic and transcripts

The ZK ElGamal program uses the ristretto255 group and canonical 32-byte point
encoding in [RFC 9496 Section 4](https://www.rfc-editor.org/rfc/rfc9496.html#section-4).
Decode with its canonical field, sign, square-root, and identity rules, not Edwards
point decoding. Scalars are 32-byte little-endian integers strictly less than
`l = 2^252 + 27742317777372353535851937790883648493`. Scalar arithmetic below is
modulo `l`; `0` denotes the group identity when used as a point.

**Normative table. ZK group parameters.**

| Name | Definition |
|---|---|
| `G` | canonical Ristretto generator, compressed hex `e2f2ae0a6abc4e71a884a961c500515f58e30b6aa582dd8db6a65945e08d2d76` |
| `Map64(b)` | ristretto255 element derivation from exactly 64 uniform bytes, RFC 9496 Section 4.3.4 |
| `H` | `Map64(SHA3-512(compress(G)))`; SHA3-512 is FIPS 202 SHA3, not Keccak-512 |
| Pedersen commitment | `C = x*G + r*H`, for value scalar `x` and opening scalar `r` |
| ElGamal key and ciphertext | for nonzero secret `s`, `P = s^-1*H`; ciphertext `(C,D)` has `D=r*P`, so `C-s*D=x*G` |
| range generators `G_i`, `H_i` | separately squeeze SHAKE256 streams seeded by ASCII `GeneratorsChainG` and `GeneratorsChainH`; map each consecutive 64-byte block with `Map64`, starting at index zero |

Proof transcripts use Merlin v1.0 framing over STROBE-128 v1.0.2. These are algorithm
identifiers, not a choice of serializer or dependency version. The following defines
the exact subset required here; all text labels are ASCII, with no terminator.

**Normative table. STROBE state and operations.**

| Item | Definition |
|---|---|
| state | 200 bytes `S`, byte positions `pos`, `begin`, and current operation flags; rate `R=166` bytes |
| permutation | Keccak-f[1600], interpreting each consecutive eight state bytes as a little-endian lane, standard 24 rounds |
| initialization | zero `S`; set `S[0..6] = 01 a8 01 00 01 60`, `S[6..18] = ASCII("STROBEv1.0.2")`; permute; set positions/flags to zero; perform meta-AD of `Merlin v1.0` |
| `F()` | XOR `begin` into `S[pos]`, `04` into `S[pos+1]`, and `80` into `S[167]`; permute; reset `pos=begin=0` |
| absorb bytes | XOR each byte into `S[pos]`, increment `pos`; run `F()` whenever `pos==166` |
| begin operation `(flags, more)` | if `more`, require the same flags and continue without a header. Otherwise save `old=begin`, set `begin=pos+1` and current flags, absorb `[old,flags]`; if flag `04` is set and `pos!=0`, run `F()` |
| meta-AD | begin with flags `12` hex, then absorb the supplied bytes |
| AD | begin with flags `02` hex, then absorb the supplied bytes |
| PRF | begin with flags `07` hex; output `S[pos]`, zero that state byte, increment `pos`, and run `F()` at the rate boundary for each requested byte |

**Normative algorithm. Merlin operations used by ZK proofs.**

```text
append(label, bytes):
    meta-AD(label, more=false)
    meta-AD(LE(length(bytes),4), more=true)
    AD(bytes, more=false)

challenge(label):
    meta-AD(label, more=false)
    meta-AD(LE(64,4), more=true)
    b = PRF(64 bytes, more=false)
    return integer_little_endian(b) mod l

new_proof(instruction_label):
    initialize STROBE as above
    append("dom-sep", "solana-zk-elgamal-proof-program-v1")
    append("dom-sep", instruction_label)
```

Appending a scalar uses its canonical 32 bytes; appending a point uses its canonical
compressed 32 bytes; appending `u64` uses eight little-endian bytes. Each append and
each challenge mutates the transcript. Challenges are consumed even when a subsequent
equation does not use their value. No transcript operation includes Rust type names,
instruction discriminants, account addresses, JSON, or host padding unless a proof's
explicit input table includes those bytes. Section 13.3.9 specifies the complete
instruction-specific transcript sequences and equations.
