# 18. Security and adversarial considerations

## 18.1 Trust boundary

All network bytes, shreds, transactions, vote data, repair responses, snapshot contents,
and RPC observations are adversarial until verified at their applicable boundary. A
finalized RPC response is evidence about a remote node, not a cryptographic proof of
bank state.

## 18.2 Ambiguous encodings

Decoders are frequent consensus-split surfaces. Length prefixes, enum tags, unused
padding, reserved discriminants, duplicate keys, and integer overflow need an explicit
accepted domain. A sender's canonical encoding does not prove that receivers reject
every noncanonical encoding; byte layouts in this specification separate production
rules from receiver acceptance.

## 18.3 Resource exhaustion

Packet, transaction, account-lock, loaded-data, compute, call-depth, shred-index,
erasure-set, and block limits protect both deterministic accounting and liveness.
Implementations can reject work earlier for locally exhausted resources only where doing
so cannot change ledger validity or voting; a local overload drop is not evidence that
the contained transaction or block is protocol-invalid.

## 18.4 Duplicate data and equivocation

Conflicting shreds at equal `(slot,index,type)` can be leader equivocation evidence.
Prematurely discarding all conflicting bytes can prevent proof construction or
correct-version repair; treating every conflict as replayable can admit mixed blocks.
Chapter 9 owns byte validity and Chapter 15 owns candidate-version state.

## 18.5 Vote safety

Persisted Alpenglow vote history is a safety boundary. Operators must treat lost or
rolled-back first-vote, fallback-vote, finalize-vote, parent-ready, or root state as a
potential equivocation condition, not just an availability incident. Trusted roots,
hard forks, epoch rank maps, BLS keys, and node-identity keys are configuration inputs
whose mismatch can partition a validator or cause unsafe votes.

## 18.6 Cryptographic domains

Solana uses several SHA-256 and BLAKE3 constructions with different prefixes and
preimages. Reusing a digest API without the specified domain bytes, interpreting a
hash's base58 text as bytes, or changing signature strictness can split validity.
Chapter 14 centralizes commitments; format chapters define signature preimages.

## 18.7 Specification updates

Activation of a new protocol feature, adoption of a hard fork, upgrade of a
protocol-owned program, or consensus migration requires a specification revision aligned
with the activation epoch. On-chain state does not silently rewrite this document.
