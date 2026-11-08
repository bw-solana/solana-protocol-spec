# 6. Validator wire protocols

## 6.1 Datagram boundary and dispatch

[SOL-WIR-0001] A gossip, repair, TPU-vote, or TVU receiver **MUST** treat one UDP
datagram as one complete application message, enforce the 1,232-byte limit, and reject a
message if its grammar does not consume the datagram exactly.

## 6.2 Gossip envelope and CRDS pull filter

Every gossip datagram begins with a `u32` little-endian `Protocol` discriminant. A
`Vec<T>` below is the Chapter 5 sequence form: an eight-byte little-endian count
followed by that many `T` values.

**Normative table.**

| Discriminant | Variant | Exact payload after discriminant |
|---:|---|---|
| `0` | `PullRequest` | `CrdsFilter`, then one `CrdsValue` |
| `1` | `PullResponse` | sender public key `[32]`, then `Vec<CrdsValue>` |
| `2` | `PushMessage` | sender public key `[32]`, then `Vec<CrdsValue>` |
| `3` | `PruneMessage` | outer public key `[32]`, then `PruneData` |
| `4` | `PingMessage` | `Ping` `[128]` |
| `5` | `PongMessage` | `Pong` `[128]` |

[SOL-WIR-0002] A gossip decoder **MUST** reject a discriminant outside `0..=5`, an
invalid nested value, a payload larger than 1,232 bytes, or trailing bytes. The combined
serialized `CrdsValue` bytes in a push or pull response **MUST NOT** exceed 1,188 bytes.

`CrdsFilter` is serialized in the following order. `BitVec<u64>` bit number `i` is bit
`(i mod 64)`—least significant bit first—of block `floor(i/64)`.

**Normative table.**

| Order | `CrdsFilter` field | Exact encoding |
|---:|---|---|
| 1 | Bloom keys | `u64` count, then that many `u64` little-endian seeds |
| 2 | Bloom blocks | option tag `u8`; `0` for no blocks, or `1`, a `u64` block count, and that many `u64` little-endian blocks |
| 3 | Bloom bit length | `u64` little-endian |
| 4 | Bloom `num_bits_set` | `u64` little-endian |
| 5 | `mask` | `u64` little-endian |
| 6 | `mask_bits` | `u32` little-endian |

The Bloom phantom type has zero bytes. For a CRDS identity hash `h[32]`, let
`hu = LE_decode(h[0..8])`, `low = (!0u64) >> mask_bits` when `mask_bits < 64` and `0`
otherwise, and `canonical_mask = mask | low`. The hash belongs to the filter's partition
exactly when `(hu | low) == canonical_mask`.

For every Bloom seed `k`, calculate a seeded FNV-1a state over all 32 bytes of `h`:
initialize `state = k`; for each byte `b`, set
`state = (state XOR b) * 0x00000100000001b3 mod 2^64`. Its Bloom position is
`state mod bit_length`. Membership is true only if the partition comparison passes, the
keys and bit vector are both nonempty, and every seeded position is set.

[SOL-WIR-0003] An accepted pull filter **MUST** have `mask_bits >= 6`; its bit length
**MUST NOT** exceed the capacity of its serialized blocks. A sender **MUST** set every
low bit of `mask` selected by `low`, while a receiver **MUST** ignore those low bits by
using `canonical_mask` as defined above.

[SOL-WIR-0004] In answering a PullRequest, a validator **MUST** consider only CRDS
values whose identity hash belongs to the requested mask partition and is absent from
the Bloom filter; a Bloom false positive may omit a value but **MUST NOT** make an
out-of-partition value eligible.

## 6.3 CRDS values and payload grammars

A `CrdsValue` is a 64-byte Ed25519 signature followed by a `CrdsData`: a four-byte
little-endian discriminant and its payload. Its cached identity is not serialized.

**Normative table. CRDS discriminants and independent processing status.**

| Disc. | Payload after discriminant | Decoder | Receiver sanitation | CRDS insertion | Locally originate in `E` | Return in ordinary pull |
|---:|---|---|---|---|---|---|
| `0` | legacy ContactInfo: zero bytes | known tag; payload decode fails | unreachable | unavailable | no | no |
| `1` | `index:u8`, then `Vote` | accept | require `index < 32` and valid Vote | accept if the Section 6.3.1 comparison wins | yes; sender chooses `index < 12` | yes |
| `2` | `index:u8`, then `LowestSlot` | accept only when `index == 0` | require valid LowestSlot | accept if comparison wins | yes | yes |
| `3` | legacy SnapshotHashes: zero bytes | known tag; payload decode fails | unreachable | unavailable | no | no |
| `4` | AccountsHashes: zero bytes | known tag; payload decode fails | unreachable | unavailable | no | no |
| `5` | `index:u8`, then `EpochSlots` | accept | require `index < 255` and valid EpochSlots | accept if comparison wins | yes | yes |
| `6` | legacy Version: zero bytes | known tag; payload decode fails | unreachable | unavailable | no | no |
| `7` | Version: zero bytes | known tag; payload decode fails | unreachable | unavailable | no | no |
| `8` | NodeInstance: zero bytes | known tag; payload decode fails | unreachable | unavailable; no duplicate transition | no | no |
| `9` | `index:u16`, then `DuplicateShred` | accept | require `index < 512` and valid chunk metadata | accept if comparison wins | yes | yes |
| `10` | `SnapshotHashes` | accept | require valid bounded hash lists | accept if comparison wins | yes | yes |
| `11` | `ContactInfo` | accept | require SOL-WIR-0013 whole-value validity; invalid endpoints are treated as absent; gossip-endpoint reachability is local policy | accept if comparison wins | yes | yes |
| `12` | `RestartLastVotedForkSlots` below | accept | require valid bounded fields | accept if comparison wins | only during a coordinated restart | no |
| `13` | `RestartHeaviestFork` below | accept | require valid bounded fields | accept if comparison wins | only during a coordinated restart | no |

For discriminants `0`, `3`, `4`, `6`, `7`, and `8`, “known tag” means only that the enum
discriminant is assigned. The epoch-`E` decoder deliberately returns an error while
decoding its zero-sized deprecated payload, before signature verification, sanitation,
label derivation, or insertion. In particular, deprecated `NodeInstance` values are
never compared as duplicates or retained in the CRDS table.

All `wallclock` and slot values identified as bounded in this section must be strictly
less than `10^15`. The exact non-ContactInfo payload layouts are:

**Normative table.**

| Payload | Exact field sequence |
|---|---|
| `Vote` | `from[32]`, one Chapter 7 transaction, `wallclock:u64` |
| `LowestSlot` | `from[32]`, `root:u64`, `lowest:u64`, `slots:u64 count` followed by ascending unique `u64` slots, `stash:u64 length` followed by bytes, `wallclock:u64` |
| `EpochSlots` | `from[32]`, `u64` count of `CompressedSlots`, those values, `wallclock:u64` |
| `SnapshotHashes` | `from[32]`, full snapshot `(slot:u64,hash[32])`, `u64` incremental count, that many `(slot:u64,hash[32])`, `wallclock:u64` |
| `DuplicateShred` | `from[32]`, `wallclock:u64`, `slot:u64`, unused `u32`, unused `u8`, `num_chunks:u8`, `chunk_index:u8`, `chunk:u64 length` followed by bytes |
| `RestartLastVotedForkSlots` | `from[32]`, `wallclock:u64`, `SlotsOffsets`, `last_voted_slot:u64`, `last_voted_hash[32]`, `shred_version:u16` |
| `RestartHeaviestFork` | `from[32]`, `wallclock:u64`, `last_slot:u64`, `last_slot_hash[32]`, `observed_stake:u64`, `shred_version:u16` |

A `CompressedSlots` starts with a `u32` discriminant and has one of these layouts.
`BitVec<u8>` uses the same option/count/bit-length grammar as the Bloom bit vector,
except its blocks are bytes and bits within a byte are least-significant-bit first.

**Normative table.**

| Discriminant | Form | Exact fields after discriminant |
|---:|---|---|
| `0` | raw DEFLATE | `first_slot:u64`, `num:u64`, byte-string length `u64`, then raw DEFLATE bytes |
| `1` | uncompressed | `first_slot:u64`, `num:u64`, `BitVec<u8>` |

The compressed form is RFC 1951 DEFLATE without a zlib header. Inflation must terminate
successfully; the output bytes form the `BitVec` blocks. For either form,
`first_slot < 10^15`, `num < 16,384`, and selected bit `i` denotes slot `first_slot + i`
for `0 <= i < num`. An uncompressed vector's bit length must be a multiple of eight and
equal its block capacity.

`SlotsOffsets` starts with `u32` discriminant `0` for run-length encoding or `1` for raw
offsets. Run-length encoding is a `u64` element count followed by canonical
unsigned-LEB128 `u16` run lengths; runs alternate selected, unselected, selected,
beginning with selected. Raw offsets is `BitVec<u8>`. In either form selected offset `i`
denotes `last_voted_slot - i`; an underflow does not denote a slot.

[SOL-WIR-0005] A CRDS Vote transaction **MUST** satisfy Chapter 7 transaction sanitation
and decode as a vote instruction; LowestSlot **MUST** have zero `root`, empty `slots`,
empty `stash`, and bounded `lowest` and `wallclock`. Every SnapshotHashes incremental
slot **MUST** be bounded and strictly greater than the full-snapshot slot; a sender
**MUST NOT** publish more than 25 incremental hashes in one value.

[SOL-WIR-0006] A CRDS-valid DuplicateShred chunk **MUST** have `num_chunks > 0` and
`chunk_index < num_chunks`; insertion does not impose the assembly limit. A
duplicate-proof consumer **MUST** discard a chunk from assembly when `num_chunks > 3`.
Assembly state is keyed by `(from,slot)`, not by the outer CRDS index. Receiving an
assembly-eligible chunk replaces the buffered chunk at its `chunk_index`; when the
number of occupied indices equals the received `num_chunks`, every buffered chunk
**MUST** agree on `slot` and `num_chunks`. Concatenating chunk bytes in index order
`0..num_chunks-1` then yields exactly `u64 length || shred1 || u64 length || shred2`.
Both nested shreds **MUST** pass Chapter 9 and constitute a valid same-slot duplicate
proof. The outer index is only the CRDS table label and **MUST NOT** partition assembly
state.

[SOL-WIR-0007] A `CrdsValue` signature **MUST** verify under the `from` or ContactInfo
public key contained in its payload over the exact bytes beginning with the `CrdsData`
discriminant. Its identity **MUST** be `SHA256(signature[64] || serialized_CrdsData)`. A
PullRequest's embedded value **MUST** be discriminant 11 ContactInfo; every pushed or
returned value **MUST** pass its payload constraints and strict signature verification.

### 6.3.1 CRDS replicated-table transition

**Normative table. CRDS label derivation.**

| Value | Table label |
|---|---|
| Vote | `(Vote, index, from)` |
| LowestSlot | `(LowestSlot, from)`; the encoded index is constrained to zero and is not part of the label |
| EpochSlots | `(EpochSlots, index, from)` |
| DuplicateShred | `(DuplicateShred, index, from)` |
| SnapshotHashes | `(SnapshotHashes, from)` |
| ContactInfo | `(ContactInfo, public_key)` |
| RestartLastVotedForkSlots | `(RestartLastVotedForkSlots, from)` |
| RestartHeaviestFork | `(RestartHeaviestFork, from)` |

For a label already present, `new` replaces `old` according to this total order.
ContactInfo first compares `(outset, wallclock)` lexicographically when the public keys
agree. If those tuples differ, the larger tuple wins. For all other cases, and for equal
ContactInfo tuples, larger wallclock wins; equal wallclocks are broken by the
lexicographically larger 32-byte CRDS identity hash. Replacing a value records the
displaced hash in the recently-purged set. An unequal losing value records its own hash
there. An equal hash is a duplicate: it leaves table bytes and insertion order
unchanged; a push-route duplicate increments only its saturating local duplicate count.

[SOL-WIR-0008] A CRDS table **MUST** derive labels and apply replacement in the order
above. An implementation **MUST NOT** merge two values sharing an origin but having
different indexed labels. In particular, EpochSlots is a logical union only when
consumed: decode each surviving indexed value, union selected slots for the origin,
remove duplicates, and order the resulting slots ascending.

Push accepts a signed value only when its wallclock is in the inclusive window
`[now-15,000 ms, now+15,000 ms]`. Pull accepts a caller ContactInfo only when its
wallclock is in the half-open window `[now-15,000 ms, now+15,000 ms)`. One
responder-wide random jitter `j` in `0..3,750 ms` is added to the caller wallclock; a
pull response selects, in table insertion order, nondeprecated values matching the
requested partition, absent from the Bloom filter, satisfying
`value.wallclock <= caller+j`, and fitting the response byte budget. A receiver inserts
an otherwise-new pull value if `now <= value.wallclock + timeout(origin)`; an expired
value may be retained only when ContactInfo for its origin already exists, so it can
assist replacement and purge decisions.

The default origin timeout is `15,000 ms`. The local identity never expires. An identity
with positive epoch stake uses `max(15,000 ms, epoch_duration_ms)`; when every supplied
stake is zero, all identities use that extended timeout. If an origin's ContactInfo has
`min(value.wallclock, local_receive_time)+timeout > now`, every label of that origin is
retained. Otherwise each label expires when the same expression is `<= now`. Displaced
and rejected-outdated hashes remain in pull Bloom filters for `20,000 ms`.

The preceding windows, insertion-order response selection, retention timeouts, and
Bloom-filter retention describe the active network's liveness profile. They are
informative: local time and peer observation are not replicated state, and a different
safe retention or selection policy does not make a bank invalid. Signed-value
constraints, labels, replacement order, filter partitioning, response byte grammar,
and receiver-side revalidation remain normative.

[SOL-WIR-0009] Push duplicate suppression, pull selection, expiration, and purge
**MAY** vary without changing protocol state. A pull response **MUST** contain only
individually valid values in the requested partition and **MUST NOT** knowingly include
a value selected by the requester's Bloom filter. The receiver **MUST** independently
apply `SOL-WIR-0007` and `SOL-WIR-0008`. Completed DuplicateShred evidence **MUST**
follow `SOL-WIR-0006`; incomplete assembly may be evicted at any time and has no
consensus effect.

The push active set contains 12 peers per stake bucket and sends each eligible new value
to the first nine unpruned peers. There are 25 buckets. For stake `s` lamports define
`b(s)=min(24, bit_length(floor(s/1,000,000,000)))`; a value owned by `o` uses bucket
`min(b(local_stake),b(stake(o)))`. Peer sampling is without replacement with weight
`(min(peer_bucket,bucket)+1)^2`; zero-weight peers follow positive-weight peers. A valid
Prune applies only at its named destination, only within `500 ms` of local time, and
marks each named origin as suppressed for pushes to the prune sender in the applicable
bucket. A node's own values are never suppressed by a prune false positive.

[SOL-WIR-0010] A sender **MUST** honor authenticated, timely prune state. Active-set
size, stake buckets, sampling, rotation timing, and fanout cap are informative liveness
policy. A sender **MUST NOT** send a value to a peer for which the applicable prune
filter positively contains that origin, except its own origin as stated above.

## 6.4 Ping, pong, and prune

**Normative table.**

| Object | Byte range | Field |
|---|---|---|
| Ping | `[0,32)` | `from` public key |
| Ping | `[32,64)` | 32-byte token |
| Ping | `[64,128)` | signature |
| Pong | `[0,32)` | `from` public key |
| Pong | `[32,64)` | 32-byte token hash |
| Pong | `[64,128)` | signature |

[SOL-WIR-0011] A Ping signature **MUST** verify under `from` over the 32 token bytes.
Its expected Pong hash **MUST** be `SHA256("SOLANA_PING_PONG" || token)`, where the
prefix is the 16 ASCII bytes. A Pong signature **MUST** verify under its `from` key over
the hash, and a Pong used as reachability evidence **MUST** match an outstanding Ping.

**Normative table.**

| Order | `PruneData` field | Exact encoding |
|---:|---|---|
| 1 | `pubkey` | 32 bytes |
| 2 | pruned keys | `u64` count followed by 32 bytes per key |
| 3 | signature | 64 bytes |
| 4 | destination | 32 bytes |
| 5 | wallclock | `u64` little-endian |

[SOL-WIR-0012] A PruneMessage **MUST** have an outer key equal to `PruneData.pubkey`, no
more than 32 pruned keys, and `wallclock < 10^15`. The signature **MUST** verify over
the serialization of `(pubkey, pruned keys, destination, wallclock)`, either directly or
preceded by the byte string `ff 53 4f 4c 41 4e 41 5f 50 52 55 4e 45 5f 44 41 54 41`; the
prefixed form includes `LE(18,8)` before those 18 prefix bytes.

## 6.5 ContactInfo

Mainnet uses shred version `50093` in epoch `E`. ContactInfo discriminant 11 has this
exact field order:

**Normative table.**

| Order | Field | Exact encoding |
|---:|---|---|
| 1 | public key | 32 bytes |
| 2 | wallclock | canonical unsigned-LEB128 `u64`, milliseconds |
| 3 | outset | `u64` little-endian, microseconds |
| 4 | shred version | `u16` little-endian |
| 5 | version major | canonical unsigned-LEB128 `u16` |
| 6 | packed minor | canonical unsigned-LEB128 `u16` |
| 7 | patch or prerelease number | canonical unsigned-LEB128 `u16` |
| 8 | source identifier | `u32` little-endian |
| 9 | feature-set identifier | `u32` little-endian |
| 10 | implementation identifier | canonical unsigned-LEB128 `u16` |
| 11 | addresses | `ShortU16` count followed by `IpAddr` records |
| 12 | sockets | `ShortU16` count followed by `SocketEntry` records |
| 13 | extensions | `ShortU16` count followed by TLV records |

The low 14 bits of packed minor are the minor version. Bits 14–15 select stable (`0`),
release candidate (`1`), beta (`2`), or alpha (`3`). For stable, the next value is the
semantic patch; otherwise the semantic patch is zero and the next value is the
prerelease number.

**Normative table.**

| Nested type | Exact encoding |
|---|---|
| IPv4 `IpAddr` | discriminant `LE(0,4)`, then four network-order address octets |
| IPv6 `IpAddr` | discriminant `LE(1,4)`, then sixteen network-order address octets; rejected by ContactInfo sanitation |
| `SocketEntry` | `key:u8`, address `index:u8`, canonical unsigned-LEB128 `offset:u16` |
| extension TLV | `type:u8`, `ShortU16` byte length, then exactly that many bytes |

The socket port accumulator begins at zero. For each entry in encoded order, add its
offset with checked `u16` arithmetic; that accumulated port and the indexed address form
the endpoint.

**Normative table. ContactInfo socket-key registry.**

| Key | Defined endpoint | Epoch-`E` protocol use |
|---:|---|---|
| `0` | gossip UDP | active |
| `1` | repair QUIC | recognized; no repair QUIC service, and a placeholder port is permitted |
| `2` | JSON-RPC | discovery only; RPC is outside scope |
| `3` | RPC pubsub | discovery only; RPC is outside scope |
| `4` | repair UDP | active |
| `5` | TPU UDP | reserved and not used; does not authorize UDP transaction ingestion |
| `6` | TPU-forward UDP | reserved and not used |
| `7` | TPU-forward QUIC | active |
| `8` | TPU QUIC | active |
| `9` | TPU-vote UDP | active |
| `10` | TVU UDP | active |
| `11` | TVU QUIC | recognized but inactive |
| `12` | TPU-vote QUIC | active |
| `13` | Alpenglow votor QUIC | active; QUIC DATAGRAM consensus messages |

[SOL-WIR-0013] A valid ContactInfo **MUST** contain only unique IPv4 addresses and
unique socket keys; any IPv6 address in the address list makes it invalid, whether or
not a socket references it. Every socket index **MUST** be in range, every address
**MUST** be referenced, and the running port sum **MUST NOT** overflow `u16`. An
endpoint whose port is zero, or whose address is unspecified or multicast, does not
invalidate the ContactInfo. Such an endpoint **MUST** be treated as absent for every
service lookup, and the ContactInfo **MUST** still be stored and relayed with its
original signed bytes. Push and pull-response ingress **MUST** discard a ContactInfo
whose gossip endpoint (key `0`) is absent or treated as absent. A receiver **MAY** also
discard, as local policy, a ContactInfo whose gossip endpoint lies outside its
configured address space (for example a private or loopback address on a public cluster)
or has not answered a recent Ping (Section 6.4); such a value may be accepted once the
endpoint answers.

[SOL-WIR-0014] ContactInfo `wallclock` **MUST** be less than `10^15`. A peer used for
Turbine or repair **MUST** advertise shred version `50093`. Unknown extension types
**MUST** be skipped by their declared length and unknown socket keys **MUST NOT** be
assigned a known service meaning.

## 6.6 TPU QUIC, TPU-vote UDP, and TLS

[SOL-WIR-0015] Ordinary TPU and TPU-forward ingestion **MUST** use QUIC and **MUST NOT**
accept transaction application data through ContactInfo keys 5 or 6. A TPU-vote UDP
datagram contains exactly one Chapter 7 transaction with no application framing. A TPU,
TPU-forward, or TPU-vote QUIC connection **MUST** negotiate the ten-byte ALPN
`solana-tpu`, and each completed requester-initiated unidirectional stream **MUST**
contain exactly one transaction. The stream is nonempty, consumes all bytes, and is at
most 4,096 bytes. Chapter 7 then applies the format-specific transaction limit: 1,232
bytes for legacy and version 0, or 4,096 bytes for version 1. The UDP form remains
limited to 1,232 bytes by Section 6.1.

[SOL-WIR-0016] A TPU receiver **MUST** pass only QUIC's ordered, reassembled bytes from
one completed unidirectional stream to the transaction decoder. It **MUST** reject a
bidirectional stream, QUIC datagram application data, or bytes concatenated across
streams. A packet gap leaves the stream incomplete until QUIC retransmission or timeout;
it is not an application decode error. Transport certificates do not authorize a
transaction; Chapter 7 signatures do.

**Informative table. Active QUIC liveness and resource profile.**

| Parameter | Required value or accepted domain |
|---|---|
| QUIC TLS version | TLS 1.3 only, as required by QUIC v1 |
| Key exchange | X25519 |
| TLS 1.3 cipher suites | accept AES-128-GCM-SHA256, AES-256-GCM-SHA384, and ChaCha20-Poly1305-SHA256 |
| ALPN | exactly `solana-tpu` |
| Per-stream receive window and application hard cap | `4,096` bytes |
| Connection receive window | `8,388,608` bytes |
| Initial bidirectional-stream limit | `0` |
| Initial unidirectional-stream limit | `0`, raised after handshake/QoS classification |
| Concurrent inbound unidirectional streams | after handshake, the exact stake-and-RTT formula below; possible advertised values are unstaked `128..=896` and staked `128..=3,584` streams |
| Idle timeout | `30` seconds |
| Handshake timeout | `2` seconds from receipt of Initial through receipt of Handshake |
| Incomplete-stream chunk timeout | `2` seconds |
| QUIC DATAGRAM | disabled |
| Path migration | disabled |
| 0-RTT | server acceptance disabled; clients may attempt resumption but application bytes are processed only after a completed handshake |
| Default connection admission | at most `2,000` staked and `2,000` unstaked connections; `16` per staked identity and `8` per unstaked IP |
| Per-source connection rate | `8` new connections per source IP per minute with burst capacity `80`; all sources together `2,500` connections/second with burst capacity `1,000` |
| Stream arrival throttle | `500` new streams per millisecond |

Let `peer_stake` and `total_stake` be unsigned epoch-stake lamports and let
`rtt_ms = min(floor(measured_RTT_in_microseconds / 1000),350)`. Compute the
post-handshake unidirectional-stream limit with unsigned floor arithmetic:

**Normative algorithm.**

```text
if peer is unstaked:
    base = 128
else if total_stake == 0 or peer_stake > total_stake:
    base = 128
else:
    base = clamp(floor(binary64(peer_stake) / binary64(total_stake) * 99872)
                 + 128,
                 128, 512)

stream_limit = floor(base * clamp(rtt_ms,50,350) / 50)
```

The two integer stake values are converted separately to IEEE-754 binary64; the quotient
and multiplication are binary64 round-to-nearest-ties-to-even, and conversion to `u32`
truncates toward zero before adding `128`. The final multiply saturates in `u32` before
integer division.

The canonical active sender profile presents one Ed25519 end-entity certificate. Its
DER certificate is 249 bytes: the fixed X.509 template has Ed25519 OID `1.3.101.112`,
serial `01` repeated
eight times, issuer CN `Solana node`, validity 1970-01-01 through 4096-01-01, empty
subject, a critical DNS SAN `localhost`, critical empty basic constraints, and a
deliberately invalid 64-byte all-`ff` certificate signature. The only variable
certificate bytes are the 32-byte Ed25519 SubjectPublicKeyInfo. The corresponding PKCS#8
private key is the 16-byte prefix `30 2e 02 01 00 30 05 06 03 2b 65 70 04 22 04 20`
followed by the 32-byte Ed25519 secret scalar.

**Informative table. Canonical dummy-certificate byte construction.**

| Byte range | Exact bytes |
|---|---|
| `[0,100)` | `30 81 f6 30 81 a9 a0 03 02 01 02 02 08 01 01 01 01 01 01 01 01 30 05 06 03 2b 65 70 30 16 31 14 30 12 06 03 55 04 03 0c 0b 53 6f 6c 61 6e 61 20 6e 6f 64 65 30 20 17 0d 37 30 30 31 30 31 30 30 30 30 30 30 5a 18 0f 34 30 39 36 30 31 30 31 30 30 30 30 30 30 5a 30 00 30 2a 30 05 06 03 2b 65 70 03 21 00` |
| `[100,132)` | the identity's 32 raw Ed25519 public-key bytes |
| `[132,249)` | `a3 29 30 27 30 17 06 03 55 1d 11 01 01 ff 04 0d 30 0b 82 09 6c 6f 63 61 6c 68 6f 73 74 30 0c 06 03 55 1d 13 01 01 ff 04 02 30 00 30 05 06 03 2b 65 70 03 41 00 ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff` |

Certificate-chain, issuer, time, SAN, SNI and certificate-signature validation are
deliberately not trust anchors. The TLS 1.3 CertificateVerify signature under the
certificate public key is mandatory. A server extracts a peer identity for stake QoS
only from a one-certificate chain whose SPKI is exactly 32 Ed25519 bytes; failure to
extract it classifies the peer as unstaked rather than authenticating a transaction. A
requester supplies SNI `<decimal-IP>.<decimal-port>.sol`; the responder does not match
it to the SAN.

[SOL-WIR-0017] Both endpoints **MUST** verify TLS handshake signatures and **MUST**
require client authentication at the handshake, while ignoring PKI chain, name, time,
and dummy-certificate-signature validation as described above. A transport identity
**MUST NOT** replace transaction signatures.

[SOL-WIR-0018] A requester **MUST** finish each transaction stream. EOF after
`1..=4,096` bytes delivers one packet to the Chapter 7 decoder; reset, stop, timeout,
zero bytes, `4,097` bytes, a second transaction in the stream, or a transaction split
between streams discards that stream without an application response. A responder
**MAY** close or reset throttled streams or connections; the sender then retries by
opening a new stream or connection and **MUST NOT** infer transaction acceptance from
transport completion.

**Informative table. QUIC application close codes in the active liveness profile.**

| Code | UTF-8 reason | First use |
|---:|---|---|
| `1` | `dropped` | an admitted connection-table entry is displaced |
| `2` | `disallowed` | peer or connection classification is not admitted |
| `4` | `too_many` | the applicable connection limit is exceeded |
| `5` | `invalid_stream` | an empty or oversized transaction stream is received |

Repair and TVU use UDP in epoch `E`; their recognized QUIC socket keys do not select
this TPU ALPN or transaction stream grammar. TPU, TPU-forward, and TPU-vote QUIC differ
only in advertised endpoint and downstream transaction classification. TPU-vote
additionally accepts the UDP form above.

### 6.6.1 Alpenglow votor QUIC datagrams

The ContactInfo key-13 endpoint carries Alpenglow votes and certificates. This service
is independent of TPU transaction ingestion.

**Normative table. Alpenglow transport profile.**

| Parameter | Active value |
|---|---|
| QUIC version | version 1 |
| TLS version | TLS 1.3 |
| TLS 1.3 cipher suites | AES-128-GCM-SHA256, AES-256-GCM-SHA384, and ChaCha20-Poly1305-SHA256 |
| ALPN | ASCII `alpenglow-v1` |
| application carriage | QUIC DATAGRAM only |
| bidirectional and unidirectional streams | disabled; initial limits are zero |
| path MTU | `1,280` bytes; path-MTU discovery disabled |
| connection migration | disabled |
| idle timeout | `5` seconds |
| keepalive interval | `2` seconds |
| 0-RTT | disabled |
| peer identity | node-identity Ed25519 public key extracted from the mutually authenticated certificate |
| peer admission | identity must belong to the admitted staked-node set for the relevant epoch window |
| inbound connections per admitted peer | at most `2` |

Certificate construction and acceptance use the identity-certificate rules in Section
6.6. The TLS CertificateVerify signature binds possession of the advertised node
identity. A certificate whose Ed25519 public key cannot be extracted, or whose identity
is not in the admitted staked-node set, does not create an unstaked Alpenglow peer; the
connection is rejected. Vote verification maps that node identity to its vote-account
rank, effective stake, and BLS public key for the target slot's epoch.

[SOL-WIR-0019] An Alpenglow endpoint **MUST** negotiate the transport profile above,
**MUST NOT** accept consensus messages on a QUIC stream, and **MUST NOT** accept early
data. It **MUST** discard a datagram that is truncated, has trailing bytes, has another
shred version, has an unknown version or kind, or exceeds the negotiated datagram size.
Transport receipt is not a vote: Chapter 15 signature, rank, and stake checks remain
mandatory.

The datagram payload uses the `ALPENGLOW-WIRE-E1072` codec in Section 15.2. QUIC
connection counts, handshake-rate governors, send retry intervals, and queue capacities
beyond the interoperability values above are local liveness policy rather than consensus
or ledger validity rules.

## 6.7 Repair

Repair requests use a `u32` little-endian top-level discriminant. Variants `0..=6` are
obsolete unsigned forms and are rejected. Variant 7 is a `Pong`; variants `8..=11`
begin with this signed header:

**Normative table.**

| Byte range | Header field |
|---|---|
| `[4,68)` | signature `[64]` |
| `[68,100)` | sender public key `[32]` |
| `[100,132)` | recipient public key `[32]` |
| `[132,140)` | Unix timestamp in milliseconds, `u64` |
| `[140,144)` | nonce, `u32` |

**Normative table.**

| Disc. | Request | Exact fields after header | Total bytes |
|---:|---|---|---:|
| `7` | Pong | no header; `from[32], hash[32], signature[64]` | 132 |
| `8` | WindowIndex | `slot:u64, shred_index:u64` | 160 |
| `9` | HighestWindowIndex | `slot:u64, shred_index:u64` | 160 |
| `10` | Orphan | `slot:u64` | 152 |
| `11` | AncestorHashes | `slot:u64` | 152 |

[SOL-WIR-0020] The signature for variants `8..=11` **MUST** verify under the header
sender over `LE(discriminant,4) || bytes[68..end]`; the signature field itself is
omitted. The recipient **MUST** be the receiving identity and the timestamp **MUST** be
within 600,000 milliseconds of current Unix time.

Ordinary shred responses are one raw Chapter 9 shred followed by the request nonce as
`u32` little-endian. Repair ping challenges use discriminant `0`, then the 128-byte Ping
layout from Section 6.4, for a total of 132 bytes.

**Normative table.**

| Specialized response | Discriminant and exact payload before trailing nonce |
|---|---|
| `AncestorHashesResponse::Hashes` | `LE(0,4)`, `u64` count, then at most 30 `(slot:u64,hash[32])` records |
| `AncestorHashesResponse::Ping` | `LE(1,4)`, Ping `[128]` |

[SOL-WIR-0021] A repair response **MUST** end in the outstanding request's nonce and
match its request: WindowIndex requires the exact slot and index; HighestWindowIndex
requires the exact slot and an index not below the request; Orphan permits at most 11
responses with shred slot not above the request; AncestorHashes permits at most 30
pairs. Every returned shred **MUST** pass Chapter 9 before use.

## 6.8 Turbine carriage

[SOL-WIR-0022] A TVU UDP datagram **MUST** carry exactly one raw Chapter 9 shred without
an application header. Before reconstruction, the receiver **MUST** validate the shred's
version, byte layout, Merkle proof, and leader signature.

[SOL-WIR-0023] Forwarding **MUST NOT** alter bytes covered by the leader-signed Merkle
root. A retransmitter signature, when present in its Chapter 9 trailer, is carried
unchanged by subsequent forwarding and does not replace leader authentication.

### 6.8.1 Epoch-`E` Turbine topology

This subsection is the minimum network-liveness profile for a validator wire sender.
Its permutation and forwarding requirements promote interoperable delivery, but a peer's
fanout choice is not ledger data and does not make an otherwise valid shred or block
invalid.

For one shred identity `(slot,index,type)` and scheduled leader `L`, form the 32-byte
seed `SHA256(LE(slot,8) || type_u8 || LE(index,4) || L[32])`, where `type_u8` is `0xa5`
for data and `0x5a` for coding shreds. Initialize ChaCha8 with this seed, counter zero,
stream zero, the RFC 8439 constants, and four column/diagonal double rounds per 64-byte
block. The RNG exposes each block as consecutive little-endian `u32` words; `next_u64`
concatenates the next two words with the first as the low half.

Construct the candidate node vector from every positive-stake identity in the slot's
epoch stake map, every current gossip identity with a valid key-10 TVU UDP address, and
the local identity. Sort by descending `(stake,pubkey)` and deduplicate by pubkey,
preferring the record with ContactInfo. Then traverse that order and retain only the
first usable key-10 endpoint for each exact `(UDP,socket-address)` and only the first
usable key-10 endpoint for each IP address. For a later duplicate, remove its endpoint:
retain the identity if its stake is positive, but drop it when its stake is zero and it
has no other active TVU endpoint. This address deduplication is part of the permutation
input, not just a send-time filter. The leader is removed. Give node `i` weight equal
to its epoch stake in lamports. To sample from total positive weight `T`, set
`zone=(T << leading_zeros(T))-1`; repeatedly draw RNG `x`, compute `p=x*T` as `u128`,
`hi=p>>64`, `lo=p mod 2^64`, and accept `hi` when `lo<=zone`. Select the least index
whose cumulative remaining weight exceeds `hi`, remove that weight, and repeat. After
positive weights are exhausted, select zero-weight indices uniformly by the same
multiply-high method and swap-remove each selected index. This produces the per-shred
permutation `P`.

**Normative algorithm. Turbine fanout tree (`F=200`).**

```text
leader sends the shred to P[0]
for a node at permutation index i:
    offset = saturating_sub(i, 1) mod F
    anchor = i - offset
    step = 1 if i == 0 else F
    child indices are the first F in
        anchor*F + offset + 1, +step, +step, ...
        that are less than len(P)
parent(i):
    none when i == 0
    offset = (i-1) mod F
    q = floor((i-1)/F)
    base = q - saturating_sub(q,1) mod F
    parent index = base if base == 0 else base + offset
```

[SOL-WIR-0024] A leader **MUST** originate each valid shred to `P[0]`. A TVU participant
receiving a newly accepted broadcast shred **MUST** forward it to the available key-10
UDP addresses at its child indices, in tree order, with at most 200 children and at most
four hops. It **MUST NOT** retransmit a shred received as a repair response. Missing,
stale, duplicate-IP, or invalid TVU addresses are skipped; repair under Section 6.7 is
the fallback for the resulting holes.

[SOL-WIR-0025] Turbine membership is evaluated from the participant's current
authenticated gossip view, so two participants may omit different unreachable addresses
without changing shred or block validity. The stake weights, seed, ChaCha8 word stream,
sampling rule, and index-to-child mapping **MUST NOT** be replaced by an
implementation-dependent shuffle.
