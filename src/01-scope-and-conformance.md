# 1. Scope and conformance

## 1.1 Scope

This specification defines the observable protocol rules needed to decode and produce
interoperable Mainnet data, decide validity, reconstruct ordered blocks, execute
deterministic state transitions, calculate consensus commitments, and participate safely
in consensus under the protocol configuration active in reference epoch `E`.

The scope is forward conformance only. An implementation starts from a trusted epoch-`E`
snapshot or another state authenticated to an epoch-`E` root and continues under this
specification. It need not ingest, replay, emit, or preserve encodings and rule branches
that ceased to be accepted before `E`. Activation epochs remain in Appendix C only to
identify how the epoch-`E` configuration was selected; they do not re-enable historical
behavior. A later Mainnet activation requires a revised specification whose affected
normative sections describe the new behavior directly.

The specification applies to banks whose derived epoch is `E`. If the trusted starting
root is the last bank before `E`, child-bank construction applies the epoch-`E` boundary
transition exactly once before that child's transactions. If the trusted starting state
is already within `E`, that state already includes the boundary transition and it must
not be applied again. This specification does not define the first bank of `E+1`.

This specification does not standardize process architecture, stages, channels,
databases, caches, thread scheduling, host execution strategy, or operator interfaces.

## 1.2 Conformance roles

The following codes identify independently referenceable protocol roles and the rules
applicable to each component.

**Normative table.**

| Code | Role | Applicable chapters | Required outcome |
|---|---|---|---|
| `SOL-CNF-0001` | Validator Wire Receiver | 5–6, Appendix B | Decode, authenticate, and disposition required inbound validator messages. |
| `SOL-CNF-0002` | Validator Wire Sender | 5–6, Appendix B | Produce canonical required outbound validator messages. |
| `SOL-CNF-0003` | Block Processor | 4–5, 8–14 | Reconstruct an ordered block and derive its deterministic post-state and commitments. |
| `SOL-CNF-0004` | Transaction Validator | 4–5, 7, 10 | Make the pre-execution acceptance decisions specified by this specification. |
| `SOL-CNF-0005` | SVM Runtime and State Transition | 4, 7, 10–14, 16–17 | Derive the specified post-state and transaction results from the same parent and ordered block. |
| `SOL-CNF-0006` | Consensus Participant | 6, 10, 14–17 | Select, vote, and root according to the consensus rules and preserve safety-critical state. |
| `SOL-CNF-0007` | Validator | All normative chapters | Compose the preceding roles for validator operation. |

When this specification gives an exact byte representation, `SOL-CNF-0002` emits that
canonical representation and `SOL-CNF-0001` applies the specified accepted domain.
Chapter 6 requirements define minimum codec, authentication, and peer-interaction
interoperability. Its explicitly informative active-liveness profiles document deployed
operating parameters but are not additional conformance roles.

## 1.3 Explicit exclusions

JSON-RPC schemas, CLI output, explorers, telemetry, local blockstore and account-index
layouts, snapshot archive compression, bootstrap convenience flows, operating-system
configuration, hardware sizing, performance tuning, peer-scoring heuristics,
transaction-selection policy, and arbitrary application-program semantics are outside
scope except when a specific rule demonstrably changes protocol interoperability,
validity, deterministic state, a consensus commitment, voting, consensus selection,
finality, or necessary liveness.
