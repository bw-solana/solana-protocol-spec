# Appendix A — Constants, identifiers, and limits

This table gathers the limits most likely to be confused across protocol domains. It
includes every per-slot/block production limit and every shred/FEC limit. Intrinsic
integer ranges and exact field lengths already visible in a byte-layout table are not
repeated here. CRDS retention, QUIC resource, and Turbine topology rows summarize the
informative active liveness profile; they are not block-validity limits.

**Informative table.**

| Domain | Limit | Epoch-`E` value | Governing section |
|---|---|---:|---|
| account state | account data length | `10,485,760` bytes | 4.1 |
| account state | growth of one account in one instruction | `10,240` bytes | 4.2 |
| account state | aggregate positive growth in one transaction | `20,971,520` bytes | 4.2 |
| derived address | seeds / bytes per seed | `16` / `32` | 5.5 |
| UDP / TPU-vote transaction | application datagram or serialized transaction | `1,232` bytes | 6.1, 6.6, 7.2 |
| gossip | combined CRDS values in push/pull response | `1,188` bytes | 6.2 |
| gossip | pull-filter `mask_bits` minimum | `6` | 6.2 |
| gossip | canonical/accepted CRDS Vote index | `<12` / `<32` | 6.3 |
| gossip | EpochSlots / DuplicateShred index | `<255` / `<512` | 6.3 |
| gossip | DuplicateShred proof-consumer assembly | `1..=3` chunks | 6.3 |
| gossip | bounded slot or wallclock field | `< 10^15` | 6.3–6.5 |
| gossip | push/contact wallclock window | `±15,000` milliseconds | 6.2 |
| gossip | pull caller window / random future jitter | `[-15,000,+15,000)` / `0..3,750` milliseconds | 6.2 |
| gossip | default origin timeout / rejected-hash retention | `15,000` / `20,000` milliseconds | 6.2 |
| gossip | prune wallclock window | `±500` milliseconds | 6.2 |
| gossip | slots in one `CompressedSlots` value | `< 16,384` | 6.3 |
| gossip | incremental snapshot hashes | `25` | 6.3 |
| gossip | pruned keys in one Prune message | `32` | 6.4 |
| gossip push | peers per bucket / sends per value / stake buckets | `12` / `9` / `25` | 6.3.1 |
| QUIC/TLS | TLS version / ALPN | TLS `1.3` / `solana-tpu` | 6.6 |
| QUIC/TLS | generated certificate length / public-key bytes | `249` / `32` bytes | 6.6 |
| TPU QUIC | one transaction stream / connection receive window | `1..=4,096` (TPU, TPU-forward) or `1..=1,232` (TPU-vote) / `8,388,608` bytes | 6.6 |
| QUIC | inbound unidirectional streams after stake/RTT scaling | unstaked `128..=896`; staked `128..=3,584` | 6.6 |
| QUIC | idle / handshake / incomplete-stream timeout | `30` / `2` / `2` seconds | 6.6 |
| QUIC | admitted connections | `2,000` staked plus `2,000` unstaked | 6.6 |
| QUIC | per-identity/IP admission | `16` per staked identity; `8` per unstaked IP; `8` new per IP per minute with burst `80` | 6.6 |
| QUIC | aggregate connection rate / burst | `2,500` connections per second / `1,000` | 6.6 |
| QUIC | stream-arrival throttle | `500` streams per millisecond | 6.6 |
| QUIC | application close codes | `1`, `2`, `4`, `5` | 6.6 |
| Alpenglow QUIC | ALPN / path MTU / idle timeout | `alpenglow-v1` / `1,280` bytes / `5` seconds | 6.6.1 |
| Alpenglow QUIC | carriage / streams / inbound connections per peer | DATAGRAM only / `0` / `2` | 6.6.1 |
| repair | signed-request timestamp distance | `600,000` ms | 6.7 |
| repair | Orphan responses / AncestorHashes pairs | `11` / `30` | 6.7 |
| Turbine | fanout / maximum hops in epoch `E` | `200` / `4` | 6.8 |
| transaction | required signatures | `1..=12` | 7.2 |
| transaction | legacy and version-0 / version-1 serialized size | `1,232` / `4,096` bytes | 7.2 |
| transaction | resolved accounts | `128` | 7.3 |
| address table | stored addresses | `256` | 7.4 |
| recent blockhash | queue length / maximum processing age | `300` / `150` | 7.5 |
| transaction | top-level instructions | `64` | 7.7 |
| instruction | account indices / CPI account metas | `255` | 7.7, 12.4 |
| precompile | secp256r1 signature records | `1..=8` | 12.6 |
| compute budget | default small/ordinary top-level request | `3,000` / `200,000` CU | 7.6 |
| compute budget | transaction compute-unit limit | `1,400,000` CU | 7.6 |
| compute budget | heap range and granularity | `32,768..=262,144`, multiple of `1,024` bytes | 7.6, 12.8.4 |
| account loading | loaded account data per transaction | `67,108,864` bytes | 7.6–7.7 |
| account loading | ordinary account overhead / ALT descriptor | `64` / `8,248` bytes | 7.7.1 |
| PoH / block | hashes per tick / bank tick-height interval / serialized tick Entries | absent / `64` / `1` | 8.4 |
| PoH / block | Entry `num_hashes` / target slot duration | `1` / `200,000,000` nanoseconds | 8.4, 16.1 |
| shred FEC | data / coding shreds per set | `32` / `32` | 9.1 |
| shred slot | data-index / coding-index upper bound | each `< 16,384` | 9.2 |
| shred slot | maximum data shreds / coding shreds | `16,384` / `16,384` | 9.2, 10.4.2 |
| shred slot | maximum FEC sets | `512` | 9.2, 10.4.2 |
| shred wire | chained data / coding payload length | `1,203` / `1,228` bytes | 9.3 |
| shred Merkle | proof entries in a 64-shred FEC set | `6` entries of `20` bytes | 9.3–9.4 |
| shred data | ledger capacity without/with retransmitter trailer | `963` / `899` bytes | 9.3 |
| shred recovery | mutually consistent shards required | `32` of `64` | 9.5 |
| ledger framing | maximum DATA_COMPLETE components / serialized Entry bytes | `512` / `10,485,760` bytes | 9.6 |
| block production | aggregate transaction cost | `50,000,000` cost units | 10.4.2 |
| block production | cost attributed to one writable account | `20,000,000` cost units | 10.4.2 |
| block production | statically requested account allocation | `50,000,000` bytes | 10.4.2 |
| block cost model | loaded-account page and charge | `32,768` bytes / `8` cost units | 10.4.1 |
| block cost model | transaction signature / requested write lock | `720` / `300` cost units | 10.4.1 |
| block cost model | secp256k1 / Ed25519 / secp256r1 precompile signature | `6,690` / `2,400` / `4,800` cost units | 10.4.1 |
| block cost model | instruction-data divisor | `4` bytes per cost unit, truncating | 10.4.1 |
| invocation | instruction-stack depth | `5` | 12.4 |
| invocation | complete instruction trace | `64` entries | 12.4 |
| CPI | account metas / account-info records | `255` / `255` | 12.4 |
| CPI | signer-seed groups / instruction data | `16` / `10,240` bytes | 12.4 |
| SBF | text instructions / local call depth | `65,536` / `64` | 12.8.1–12.8.2 |
| SBF | accepted ISA versions | V0 through V3 | 12.8.3 |
| SBF | stack frame / allocated stack | `4,096` / `262,144` bytes | 12.8.1, 12.8.4 |
| SBF | meter checkpoint distance | `10,000` instructions | 12.8.1 |
| syscall | hash slices / Curve25519 MSM points | `20,000` / `512` | 12.8.7–12.8.8 |
| syscall | return data / Poseidon inputs / BLS pairs | `1,024` bytes / `12` / `8` | 12.8.7 |
| syscall | PDA seeds / bytes per seed / signer groups | `16` / `32` bytes / `16` | 5.5, 12.8.7 |
| ZK proof program | batched range commitments | at most `8` | 13.3.9 |
| protocol state | Stake / Vote-v4 / ALT-header bytes | `200` / `3,762` / `56` bytes | 13.3 |
| loader | ProgramData extension minimum / maximum account data | `10,240` / `10,485,760` bytes | 13.3.2 |
| protocol state | ALT stored addresses | `256` | 13.3.7 |
| bounded sysvar | RecentBlockhashes records | `150` | 13.4 |
| bounded sysvar | SlotHashes / StakeHistory records | `512` / `512` | 13.4 |
| bounded sysvar | SlotHistory window | `1,048,576` slots | 13.4 |
| accounts commitment | lattice elements | `1,024` `u16` values | 14.4 |
| accounts commitment | lattice serialization | `2,048` bytes | 14.4 |
| Alpenglow | Notar, Notar-fallback, Skip, and Finalize threshold | `>=60%` admitted stake | 15.3 |
| Alpenglow | Fast-finalize threshold | `>=80%` admitted stake | 15.3 |
| Alpenglow | safe-to-notar / safe-to-skip thresholds | Section 15.5 exact `20%`, `40%`, and `60%` predicates | 15.5 |
| Alpenglow | Notar-fallback targets per validator / certified blocks per slot | `3` / `4` | 15.3, 15.5 |
| Alpenglow | leader window / reward-certificate delay | `4` / `8` slots | 15.4, 16.6.3 |
| Alpenglow | nominal handover / per-slot allowance | `400` / `200` milliseconds | 15.9 |
| epoch | target slot duration / slots per year | `200,000,000` nanoseconds / `157,784,629.968` slots | 16.1, 16.6.2 |
| epoch | normal slots / consecutive leader slots | `432,000` / `4` | 16.1, 16.3 |
| inflation | initial / terminal rate / taper | `8%` / `1.5%` / `15%` | 16.6.2 |
| inflation | state address / epoch-`E` data length | `3vJzniWALu2qGFJ1ZY6JD3Y2ZmA5zWqmVdDX7yBXHgu7` / `49` bytes | 16.6.2 |
| inflation | epoch-1072 / epoch-1071 reward budget | `62,395,745,304,053` / `62,417,544,028,268` lamports | 16.6.2 |
| rewards | admitted vote accounts | `2,000` | 16.6.1 |
| rewards | stake updates per produced block | `2,048` accounts | 16.6.5 |
| rewards | maximum distribution span | `43,200` produced blocks | 16.6.5 |
| feature state | minimum Feature-account data | `9` bytes | 17.1 |
| hard-fork state | registered Mainnet hard forks at `E` | `9` registrations | 17.5 |
