# 19. Implementation-specific interoperability profiles

## 19.1 Status and sender behavior

This chapter is informative. It describes implementation-specific behavior that helps
transaction senders and validator peers communicate with deployed receivers. The
profiles use the exact research revisions in [Appendix D](appendix-d-source-references.md).
They are not Mainnet-wide operating parameters or additional validator conformance
requirements. Validators do not need to reproduce another implementation's admission,
flow-control allocation, resource limits, timeouts, or peer-selection policy.

An operator can override defaults, and a receiver can change its allocations as its
load, stake view, or available resources change. These changes do not require a
protocol feature activation. The shared wire, authentication, and application-message
requirements remain in [Chapter 6](06-validator-wire-protocols.md).

A transaction sender uses the receiver's actual QUIC transport parameters and subsequent
`MAX_DATA`, `MAX_STREAM_DATA`, and `MAX_STREAMS` frames, not a locally computed copy of
one of the profiles below. Initial stream credit can be zero and increase after
authentication. Large advertised credit is not a promise of application capacity.
Connection admission, stream completion, and transport acknowledgement do not promise
transaction execution or inclusion in a block.

The two TPU profiles below share X25519 and TLS_AES_128_GCM_SHA256. Offering those
choices allows a sender to negotiate with either profile; support for every cipher
listed by one implementation is not required of another implementation.

On throttling, reset, or connection closure, a sender can back off and retry a stream
or connection while the transaction remains usable under Chapter 7. A sender cannot
assume that every receiver uses the same application close codes or reason strings.
No quota in this chapter changes transaction validity, block accounting, or replay
protection, and none imposes a minimum service rate on a validator.

## 19.2 Agave TPU interoperability profile

### 19.2.1 Transport and admission

The following defaults apply to the TPU services unless an operator changes them.
TPU-forward has different connection defaults from ordinary TPU. TPU-vote uses a
separate simple-QoS governor, not the stake-weighted arrival quotas in Section 19.2.2.

**Informative table. Agave TPU transport and admission defaults.**

| Parameter | Default or behavior |
|---|---|
| TLS key exchange | X25519 |
| TLS 1.3 cipher suites | AES-128-GCM-SHA256, AES-256-GCM-SHA384, and ChaCha20-Poly1305-SHA256 |
| Per-stream receive window | `4,096` bytes on TPU and TPU-forward; `1,232` bytes on TPU-vote; application-message limits remain in Chapter 6 |
| Connection receive window | `8,388,608` bytes |
| Initial bidirectional-stream limit | `0` streams |
| Initial unidirectional-stream limit | `0` streams, raised after handshake/QoS classification |
| Post-handshake unidirectional-stream allocation | Section 19.2.3 stake-and-RTT calculation |
| Idle timeout | `30` seconds |
| Handshake timeout | `2` seconds from receipt of Initial through receipt of Handshake |
| Incomplete-stream chunk timeout | `2` seconds |
| Path migration | disabled |
| QUIC DATAGRAM reception | disabled |
| 0-RTT | server acceptance disabled; application bytes are processed only after a completed handshake |
| Ordinary TPU connection admission | at most `2,000` staked and `3,000` unstaked connections |
| TPU-forward connection admission | at most `5,000` staked and `0` unstaked connections |
| Per-peer connections | at most `16` per staked identity and `8` per unstaked IP |
| Per-source connection rate | `8` new connections per source IP per minute with burst capacity `80` |
| Aggregate connection rate | `2,500` connections/second with burst capacity `1,000` |
| Stream-arrival capacity used by the governor | `500` new streams per millisecond |

### 19.2.2 Arrival quotas and load estimation

Let `m` be the configured stream-arrival capacity in streams/millisecond; its default
is in Section 19.2.1. A throttling window is `100` milliseconds and its capacity is
`W = 100*m` streams. Counters are shared across connections with the same connection-
table key: the peer's public key when known, otherwise its IP address.

**Informative table. Agave stake-weighted stream-arrival quotas.**

| Receiver configuration and peer class | Unthrottled quota per window | Throttled quota per window | Throttling trigger |
|---|---:|---:|---|
| Unstaked connections enabled; unstaked peer | `40` streams | `10` streams | total-load EMA at least `70%` of capacity |
| Unstaked connections enabled; staked peer | `W` streams | `max(21, floor(W*peer_stake/total_stake))` streams | staked-load EMA at least `76%` of capacity |
| Unstaked connections disabled; staked peer | `W` streams | `max(1, floor(W*peer_stake/total_stake))` streams | staked-load EMA at least `95%` of capacity |

The stake quantities are unsigned lamports from the receiver's stake view. For the
stake quotient, multiplication saturates in `u128`; division is integer floor. A zero
denominator or a quotient not representable as `u64` selects the row's minimum quota
(`21` or `1` streams). With default `m`, `W` is `50,000` streams/window. Disabled
unstaked service does not grant an unstaked sender a quota.

Load capacity is `5*m` stream arrivals per 5-ms estimation interval. Staked and unstaked
load have separate integer EMAs. Every update replaces `e` by
`floor((49*r + 975*e)/1024)`, using `u128` intermediate arithmetic, where `r` is that
class's arrival count. After the update carrying the accumulated arrivals, advance
missed intervals with `r=0`. Unstaked throttling tests the sum of the two EMAs; staked
throttling tests only the staked EMA. Thresholds are the floor of the corresponding
binary64 ratio times the interval capacity. These are local scheduling decisions,
not peer-visible transaction validity checks.

### 19.2.3 Concurrent-stream calculation

This allocation is separate from the arrival quotas in Section 19.2.2. Let
`peer_stake` and `total_stake` be unsigned stake lamports in the receiver's view and
`rtt_ms = min(floor(measured_RTT_in_microseconds / 1000),350)`. The receiver computes
the post-handshake unidirectional-stream allocation as follows.

**Informative algorithm. Agave stake-and-RTT stream allocation.**

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

The two stake values are converted separately to IEEE-754 binary64; the quotient
and multiplication use round-to-nearest, ties-to-even, and conversion to `u32`
truncates toward zero before adding `128`. The final multiply saturates in `u32`
before integer division. The resulting allocations are `128..=896` streams for an
unstaked peer and `128..=3,584` streams for a staked peer. A sender still follows the
actual advertised stream credit rather than treating this calculation as an entitlement.

### 19.2.4 Certificate construction

Agave generates one Ed25519 end-entity certificate. Its DER encoding is `249` bytes:
the fixed X.509 template has Ed25519 OID `1.3.101.112`, serial `01` repeated eight
times, issuer CN `Solana node`, validity 1970-01-01 through 4096-01-01, empty subject,
a critical DNS SAN `localhost`, critical empty basic constraints, and a deliberately
invalid 64-byte all-`ff` certificate signature. The variable field is the identity's
32-byte Ed25519 public key.

**Informative table. Agave dummy-certificate byte construction.**

| Byte range | Exact bytes |
|---|---|
| `[0,100)` | `30 81 f6 30 81 a9 a0 03 02 01 02 02 08 01 01 01 01 01 01 01 01 30 05 06 03 2b 65 70 30 16 31 14 30 12 06 03 55 04 03 0c 0b 53 6f 6c 61 6e 61 20 6e 6f 64 65 30 20 17 0d 37 30 30 31 30 31 30 30 30 30 30 30 5a 18 0f 34 30 39 36 30 31 30 31 30 30 30 30 30 30 5a 30 00 30 2a 30 05 06 03 2b 65 70 03 21 00` |
| `[100,132)` | the identity's 32 raw Ed25519 public-key bytes |
| `[132,249)` | `a3 29 30 27 30 17 06 03 55 1d 11 01 01 ff 04 0d 30 0b 82 09 6c 6f 63 61 6c 68 6f 73 74 30 0c 06 03 55 1d 13 01 01 ff 04 02 30 00 30 05 06 03 2b 65 70 03 41 00 ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff` |

The local PKCS#8 private-key encoding is the 16-byte prefix
`30 2e 02 01 00 30 05 06 03 2b 65 70 04 22 04 20` followed by the 32-byte Ed25519
secret seed; it is not transmitted. Neither the certificate's total length nor its
public-key offset is a general receiver requirement. Section 6.6 defines certificate
acceptance, Ed25519 SPKI parsing, and TLS proof of possession independently of this
template. The requester generates SNI `<decimal-IP>.<decimal-port>.sol`; the responder
does not match it to the SAN.

### 19.2.5 Application close codes

These are Agave application close codes, not QUIC transport errors or transaction
execution errors. Another receiver can use different codes or reset a stream instead.

**Informative table. Agave TPU application close codes.**

| Code | UTF-8 reason | Use |
|---:|---|---|
| `1` | `dropped` | an admitted connection-table entry is displaced |
| `2` | `disallowed` | peer or connection classification is not admitted |
| `4` | `too_many` | the applicable connection limit is exceeded |
| `5` | `invalid_stream` | an empty or oversized transaction stream is received |

## 19.3 Firedancer TPU interoperability profile

Firedancer's TPU QUIC receiver uses configurable connection, handshake, and transaction-
reassembly pools rather than the Agave quota and stream-allocation algorithms above.
The following values describe the referenced implementation's defaults and advertised
transport parameters. They are not additional requirements for other receivers.

**Informative table. Firedancer TPU transport and admission profile.**

| Parameter | Default or behavior |
|---|---|
| TLS key exchange | X25519 |
| TLS 1.3 cipher suite | AES-128-GCM-SHA256 |
| Connection capacity | `131,072` connections per QUIC tile |
| Concurrent-handshake capacity | `4,096` handshakes per QUIC tile |
| Transaction-reassembly configuration | `131,072` simultaneous reassemblies per QUIC tile |
| Idle timeout | `10,000` milliseconds |
| ACK scheduling delay configuration | `10` milliseconds; advertised maximum ACK delay `20` milliseconds |
| QUIC Retry | enabled |
| Initial connection receive credit | `2^62 - 1` bytes |
| Initial unidirectional-stream credit | `2^60` streams |
| Per-stream receive credit | `4,096` bytes; application-message limits remain in Chapter 6 |
| Initial bidirectional-stream credit | `0` streams |
| Active path migration | disabled |

The advertised connection and stream credits are intentionally much larger than the
receiver's finite reassembly resources. They do not reserve those resources. A sender
can encounter local drops or connection rejection despite having sufficient QUIC
credit. Neither Agave's stream-allocation formula nor its staked/unstaked quota values
predicts this receiver's available capacity.

## 19.4 Agave gossip policy

The CRDS encodings, signature checks, labels, replacement order, Bloom partition
semantics, and prune meaning remain in Sections 6.2–6.4. The following wallclock,
selection, retention, and peer-sampling policies are local behavior in the referenced
Agave implementation; another validator does not need to reproduce them.

**Informative table. Agave gossip timing and selection profile.**

| Operation | Policy |
|---|---|
| Push admission | signed value wallclock in inclusive `[now-15,000 ms, now+15,000 ms]` |
| Pull-caller admission | caller ContactInfo wallclock in half-open `[now-15,000 ms, now+15,000 ms)` |
| Pull-response future allowance | one responder-wide random jitter `j` in `0..3,750 ms`; select values with `value.wallclock <= caller.wallclock+j` |
| Pull-response order | table insertion order, subject to Chapter 6 partition, Bloom-membership, value-validity, and response-byte constraints |
| Default origin timeout | `15,000` milliseconds |
| Local identity timeout | never expires |
| Positive-stake origin timeout | `max(15,000 ms, epoch_duration_ms)`; the same extended timeout applies to all identities when every supplied stake is zero |
| Displaced/rejected-outdated hash retention in pull Bloom filters | `20,000` milliseconds |
| Prune freshness | within `500` milliseconds of the receiver's local time |

A receiver inserts an otherwise-new pull value when
`now <= value.wallclock + timeout(origin)`. An expired value can be retained when
ContactInfo for its origin already exists, to assist replacement and purge decisions.
If an origin's ContactInfo has
`min(value.wallclock, local_receive_time)+timeout > now`, every label of that origin
is retained. Otherwise each label expires when its own corresponding expression is
`<= now`. Local time and receipt history are not replicated bank state.

The push active set has `12` peers per stake bucket, `25` buckets, and sends an eligible
new value to the first `9` unpruned peers. For stake `s` lamports, define
`b(s)=min(24, bit_length(floor(s/1,000,000,000)))`; a value owned by `o` uses bucket
`min(b(local_stake),b(stake(o)))`. Peer sampling is without replacement with weight
`(min(peer_bucket,bucket)+1)^2`; zero-weight peers follow positive-weight peers. An
accepted prune suppresses its named origins for pushes to the prune sender in the
applicable bucket, except that a node's own values are not suppressed by a prune
false positive.

## 19.5 Agave Alpenglow transport tuning

The Alpenglow datagram codec, authentication, peer eligibility, and no-early-data rule
remain in Section 6.6.1. Transport tuning does not change vote or certificate validity.

**Informative table. Agave Alpenglow transport tuning.**

| Parameter | Value or behavior |
|---|---|
| TLS key exchange and cipher suites | Section 19.2.1 crypto choices |
| Initial and minimum path MTU | `1,280` bytes; path-MTU discovery disabled |
| Idle timeout | `5` seconds |
| Keepalive interval | `2` seconds |
| Active path migration | disabled |
| Inbound connections per admitted peer | at most `2` |
| Initial bidirectional- and unidirectional-stream credit | `0` streams |

Senders respect the peer's negotiated QUIC DATAGRAM size rather than treating a path
MTU as an application-payload budget. A different validator can choose other transport
tuning while retaining the shared Chapter 6.6.1 application protocol.
