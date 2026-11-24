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
| [BLS signatures v4.0.0](https://github.com/anza-xyz/solana-sdk/tree/6744904ade8b5ca176c6ceb828cc78eb3907569f/bls-signatures) | `6744904ade8b5ca176c6ceb828cc78eb3907569f` | G1/G2 encodings, signature/PoP domains, identity/subgroup checks, and aggregate verification in Section 5.7 |
| [BLS curve wrapper](https://github.com/anza-xyz/blstrs/tree/a1faffa3420bdaf4102425c43f00cdd815d4f145) | `a1faffa3420bdaf4102425c43f00cdd815d4f145` | Point conversion and subgroup checks in the pinned BLS dependency |
| [BLST v0.3.17](https://github.com/supranational/blst/tree/54e6e55674722fc2797ebb4bbb71b26d881eb4b8) | `54e6e55674722fc2797ebb4bbb71b26d881eb4b8` | G2 deserialization accepts a compressed prefix in the fixed 192-byte signature field; `src/e2.c`, `POINTonE2_Deserialize_Z` |
| [ZK SDK v8.0.1](https://github.com/solana-program/zk-elgamal-proof/tree/509d67bb656a61f1be121a66afebb4338401ffb0/zk-sdk) | `509d67bb656a61f1be121a66afebb4338401ffb0` | Section 5.8 group/generator definitions and Section 13.3.9 POD fields, transcript labels, sigma equations, range-proof equation, and zero-tail checks |
| [Merlin v3.0.0](https://github.com/zkcrypto/merlin/tree/a94892f457202399661573480bb7250b4fe7c0af) | `a94892f457202399661573480bb7250b4fe7c0af` | Evidence for the inlined Merlin-v1.0 / STROBE-v1.0.2 subset in Section 5.8 |
| [Message v5.0.0](https://github.com/anza-xyz/solana-sdk/tree/7b0c6692a317d9bedfe957a44d48e60a86ed192c/message) | `7b0c6692a317d9bedfe957a44d48e60a86ed192c` | Complete-key-list program-ID write demotion in Section 7.3 |
| [SVM transaction v6.0.0](https://github.com/anza-xyz/solana-sdk/tree/bc4b3d14471333d6349b88ac988fabcf222f8918/svm-transaction) | `bc4b3d14471333d6349b88ac988fabcf222f8918` | Static/effectively-writable durable-nonce marker eligibility in Section 7.5 |
| [RFC 9669](https://www.rfc-editor.org/rfc/rfc9669.html) | N/A | Base BPF instruction-set semantics, subject to the SBF modifications in Section 12.8 |
| [RFC 9380](https://www.rfc-editor.org/rfc/rfc9380.html) | N/A | BLS12-381 G2 hash-to-curve primitive with Section 5.7's exact DSTs |
| [RFC 9496](https://www.rfc-editor.org/rfc/rfc9496.html) | N/A | Section 5.8 Ristretto point encoding, scalar field, and uniform-byte element derivation |
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

## Runtime and consensus correction evidence

Paths below are relative to the exact repository revisions in the research table.
They identify the checked behavior; the controlling protocol definitions remain the
numbered sections, not the source files.

**Source evidence table. Cross-implementation checks.**

| Specification surface | Agave evidence | Firedancer evidence |
|---|---|---|
| footer Clock and nanosecond-clock writes | `runtime/src/bank.rs`: `update_clock_slot_for_alpenglow`, `update_clock_from_footer`; `runtime/src/block_component_processor.rs`: `update_bank_with_footer_fields` | `src/flamenco/runtime/sysvar/fd_sysvar_clock.c`; `src/flamenco/runtime/fd_runtime.c`: `apply_footer` |
| consensus ranks and denominator | `runtime/src/epoch_stakes.rs`: `BLSPubkeyToRankMap::new`; `runtime/src/bank.rs`: `verify_certificate` | `src/flamenco/stakes/fd_vote_stakes.c`: `fd_vote_stakes_finalize` |
| loader CPI allowlist and failure conversion | `program-runtime/src/cpi.rs`: `check_authorized_program`; `program-runtime/src/vm.rs`; `program-runtime/src/invoke_context.rs` | `src/flamenco/vm/syscall/fd_vm_syscall_cpi.c` |
| native allocation and signed resize meter | `programs/system/src/system_processor.rs`; `transaction-context/src/transaction_accounts.rs` | `src/flamenco/runtime/program/fd_system_program.c`; `src/flamenco/runtime/fd_borrowed_account.c` |
| copied-input memory and writeback | `program-runtime/src/serialization.rs` | `src/flamenco/runtime/program/fd_bpf_loader_serialization.c` |
| hash aliasing and staged syscall compute | `syscalls/src/lib.rs`; `syscalls/src/logging.rs`; `program-runtime/src/cpi.rs` | `src/flamenco/vm/syscall/fd_vm_syscall_hash.c`; `fd_vm_syscall_util.c`; `fd_vm_syscall_cpi.c` in the same directory |
| nonce eligibility and program-ID demotion | pinned SDK dependencies above; `runtime/src/bank/check_transactions.rs` | `src/flamenco/runtime/program/fd_system_program_nonce.c`; `src/flamenco/runtime/fd_runtime.c` |
| Vote instruction 17 | `programs/vote/src/vote_processor.rs`; `programs/vote/src/vote_state/mod.rs`: `update_commission_collector` | `src/flamenco/runtime/program/vote/fd_vote_program.c` |
| block-ID repair | `core/src/repair/serve_repair.rs`; `core/src/repair/standard_repair_handler.rs` | `src/discof/repair/fd_repair.h` |
| ContactInfo endpoint filtering | `gossip/src/contact_info.rs`: `sanitize_entries`, `sanitize_socket`, `TryFrom<ContactInfoLite>` | `src/flamenco/gossip/fd_gossip_message.c` |
| fee-payer rent-failure disposition | `svm/src/account_loader.rs`: `validate_fee_payer`; `svm/src/rent_calculator.rs` | `src/flamenco/runtime/fd_executor.c` |
| ZK sigma/range verification | pinned ZK SDK above, called by `programs/zk-elgamal-proof/src/lib.rs` | `src/ballet/zksdk/instructions/`; `src/ballet/zksdk/rangeproofs/`; `src/ballet/zksdk/transcript/` |

The SBF verifier at the pinned revision enforces nonempty eight-byte-multiple text,
not a 65,536-instruction ceiling (`src/verifier.rs`). The independent VM verifier in
`src/flamenco/vm/fd_vm.c` likewise has no such instruction-count ceiling.

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
