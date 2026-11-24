# Appendix D — Source references

These references are informative research material. They do not version this
specification and do not override a numbered protocol requirement.

**Source evidence table. Exact research revisions.**

| Reference | Exact commit | Principal surfaces |
|---|---|---|
| [Agave v4.4.1](https://github.com/anza-xyz/agave/tree/5bb304027fe1a166b02cb26dc8c161593f416fca) | `5bb304027fe1a166b02cb26dc8c161593f416fca` | Bank/SVM/runtime, ProgramData loaded-size floor, durable-nonce replay protection, Alpenglow, transaction v1, wire, TLS identity, and Chapter 19 implementation profiles |
| [Firedancer v26.10.6](https://github.com/firedancer-io/firedancer/tree/46767cf6dad10b05891428a2d853fca546de0ddb) | `46767cf6dad10b05891428a2d853fca546de0ddb` | Independent ProgramData loaded-size floor, nonce replay protection, Alpenglow wire/state, shreds, VM, hashing, and Chapter 19 TPU-profile evidence |
| [SBF implementation v0.24.0](https://github.com/anza-xyz/sbpf/tree/a275924d4cb80e78c16a206f20ece6b3d8a202b9) | `a275924d4cb80e78c16a206f20ece6b3d8a202b9` | Evidence used to inline the V0–V3 ISA, verifier, ELF, call, and memory rules; not a normative dependency |
| [SIMD repository](https://github.com/solana-foundation/solana-improvement-documents/tree/4f80d8a6938f86b9394e792253a2a52e01346314) | `4f80d8a6938f86b9394e792253a2a52e01346314` | Proposal text, activation intent, and feature identifiers linked from Appendix C |
| [Legacy Solana](https://github.com/solana-labs/solana/tree/cca16a815dbc41c806d198ffdd2943e3c38c6a75) | `cca16a815dbc41c806d198ffdd2943e3c38c6a75` | Historical protocol-program and wire behavior still active in `E` |
| [Stake Core-BPF program v5.1.0](https://github.com/solana-program/stake/tree/3b511b63093618bf12493728e416fc647cb19e5b) | `3b511b63093618bf12493728e416fc647cb19e5b` | Stake Program v5.1 executable behavior and fixed-point stake-arithmetic research |
| [RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html) | N/A | Base BPF instruction-set semantics, subject to the SBF modifications in Section 12.8 |
| [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html) | N/A | QUIC transport parameters and connection, stream-data, and stream-count flow control |
| [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html) | N/A | TLS 1.3 integration and handshake authentication for QUIC |

Finalized Mainnet account observations from
[the public Mainnet endpoint](https://api.mainnet-beta.solana.com/) were compared
against Appendix C and the revisions above. Feature activation epochs record fixed
events: for example, `loaderv3_programdata_size_floor` activated at slot `466,128,000`
in epoch `1079`. The inflation account was checked against the byte layout and
boundary calculation in Section 16.6.2; its current/previous epochs and reward budgets
are changing bank state, not protocol constants. LastRestartSlot matched the hard-fork
registration list in Section 17.5. These account observations are research evidence,
not RPC schemas or additional protocol-version selectors.

**Source evidence table. Mainnet Core-BPF deployment metadata.**

| Program from Section 13.2 | ProgramData address | Stored deployment slot | Deployment epoch |
|---|---|---:|---:|
| Stake | `6WU8Nxarf9fudRK5atWwjLY4vFaw5UrrWhL88qz7iCMJ` | `443,232,000` | `1026` |
| Config | `CHKQ74qcDUJbwc4snCEZXL8tV1WQvXqioArLGgUHPZq9` | `325,296,000` | `753` |
| Feature | `H253PPA38ecQvnpSvfWgqBfMuVgZ656CEXLgTHwZQHfs` | `324,864,000` | `752` |
| Address lookup table | `4zSpbk5jGQyMmUrqCSjFZbRKwsrMXBPsyTzjhJEAsefG` | `329,184,000` | `762` |

Each observed ProgramData account had no upgrade authority. Deployment slots identify
the program deployments rather than the specification version. The feature schedule
and version-floor records are supporting research
material: [activation schedule](https://github.com/anza-xyz/agave/wiki/Feature-Gate-Tracker-Schedule)
and [version floor](https://raw.githubusercontent.com/wiki/anza-xyz/agave/version-floor.json).
Testnet/devnet activations and unactivated feature definitions do not extend Appendix C.

## Implementation-profile evidence

The following sources define the implementation defaults documented in Chapter 19.
They do not add validator conformance requirements; actual receiver configuration and
advertised QUIC credit can differ from the defaults.

**Source evidence table. Implementation-specific interoperability profiles.**

| Profile | Source at the exact research revision |
|---|---|
| Agave TPU transport and connection defaults | [Transport configuration](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/streamer/src/quic.rs), [service-specific defaults](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/validator/src/cli.rs), and [TPU/forward/vote configuration](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/validator/src/commands/run/execute.rs) |
| Agave TPU quotas and stream allocation | [Arrival governor](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/streamer/src/nonblocking/stream_throttle.rs) and [stake-and-RTT allocation](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/streamer/src/nonblocking/swqos.rs) |
| Agave TLS and application close behavior | [TLS crypto choices](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/tls-utils/src/crypto_provider.rs), [certificate construction](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/tls-utils/src/tls_certificates.rs), and [stream receiver and close codes](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/streamer/src/nonblocking/quic.rs) |
| Firedancer TPU profile | [Configuration defaults](https://github.com/firedancer-io/firedancer/blob/46767cf6dad10b05891428a2d853fca546de0ddb/src/app/firedancer/config/default.toml), [TPU receiver](https://github.com/firedancer-io/firedancer/blob/46767cf6dad10b05891428a2d853fca546de0ddb/src/disco/quic/fd_quic_tile.c), [advertised QUIC credit](https://github.com/firedancer-io/firedancer/blob/46767cf6dad10b05891428a2d853fca546de0ddb/src/waltz/quic/fd_quic.c), and [TLS crypto support](https://github.com/firedancer-io/firedancer/blob/46767cf6dad10b05891428a2d853fca546de0ddb/src/waltz/tls/fd_tls.h) |
| Agave gossip policy | [Push active set](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/gossip/src/push_active_set.rs), [push admission and prune handling](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/gossip/src/crds_gossip_push.rs), and [pull selection and retention](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/gossip/src/crds_gossip_pull.rs) |
| Agave Alpenglow transport tuning | [Transport configuration](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/votor-transport/src/transport.rs) and [peer connection handling](https://github.com/anza-xyz/agave/blob/5bb304027fe1a166b02cb26dc8c161593f416fca/votor-transport/src/server.rs) |
