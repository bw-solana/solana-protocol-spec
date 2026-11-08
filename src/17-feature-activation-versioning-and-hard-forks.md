# 17. Feature activation, versioning, and hard forks

## 17.1 Feature account representation

The Feature Gate program address is `Feature111111111111111111111111111111111111`. A
recognized feature is identified by its 32-byte account address; a source name or SIMD
number is not its protocol identity.

**Normative table.**

| Byte range | Field | Encoding | Accepted values | Disposition |
|---|---|---|---|---|
| `[0,1)` | `activated_at` tag | `u8` | `0` = pending; `1` = present | Other tag: account does not decode as a feature. |
| `[1,9)` | activation slot | `u64` little-endian | Meaningful when tag is 1 | With tag 0, ignored padding in the nine-byte account allocation. |
| `[9,len)` | trailing account data | bytes | Decoder-compatible trailing data | Does not alter `activated_at`; retained as account data. |

[SOL-FTR-0001] A recognized feature account **MUST** be treated as decodable only when
its owner is the Feature Gate program, its data length is at least 9 bytes, and its data
decodes to tag 0 or tag 1 with the tag-1 `u64` in little-endian order; otherwise that
recognized feature remains inactive.

## 17.2 Active and pending state

For bank slot `s`, begin with the parent's active feature-address set. For every
recognized address not yet active, load its account from the bank's fixed-root account
view and decode it as Section 17.1.

[SOL-FTR-0002] At ordinary bank construction or snapshot restoration, a recognized
feature whose decoded activation slot is `a` **MUST** enter the active set exactly when
`s >= a`; a pending feature (tag 0) **MUST NOT** enter the active set at an ordinary
non-boundary bank.

[SOL-FTR-0003] At the first bank of an epoch, every recognized pending feature **MUST**
enter the active set with activation slot equal to that boundary bank's slot, and its
on-chain feature account **MUST** be rewritten to tag 1 with that slot before the
boundary state is frozen.

On a given fork, a tag-0 feature observed in a bank of epoch `e` activates at the slot
of the first descendant bank whose epoch is greater than `e`. That slot is `first(e+1)`
only when slot `first(e+1)` is present on the fork. If it is skipped, or the fork passes
over whole epochs, the activation slot is the slot of the first bank actually produced
after the boundary (SOL-FTR-0003). A feature whose activation epoch is greater than `E`
is not part of this specification's protocol rules.

[SOL-FTR-0004] An unrecognized feature address **MUST NOT** change executable behavior
just because its account is owned by the Feature Gate program.

## 17.3 Specification resolution

The header identifies reference epoch `E`, and Appendix C lists the feature-linked SIMDs
active by that epoch. Normative chapters state the resulting behavior directly.

[SOL-FTR-0005] Protocol behavior for a block at slot `s` **MUST** be selected from
feature state in that block's ancestor-derived bank state, not from the wall clock, a
validator version string, a compiled feature-set number, a SIMD status, or this
specification's publication time.

[SOL-FTR-0006] A compiled feature-set identifier **MUST NOT** be interpreted as an
on-chain active-feature bitmap; it identifies software capability only.

## 17.4 Activation side effects

Some feature activations alter more than a branch predicate: they may update rent
parameters or inflation, add a builtin or precompile, migrate a builtin to Core BPF, or
upgrade a protocol-owned program from a designated buffer. Such a change is part of the
epoch-boundary state transition.

[SOL-FTR-0007] A Runtime/State-Transition implementation **MUST** apply every side
effect of all newly activated features as one epoch-boundary transition before
processing an ordinary transaction in that bank; no transaction may observe only a
proper subset of those side effects.

## 17.5 Hard forks and restarts

A hard-fork registration is a trusted chain-identity pair
`(slot:u64, multiplicity:u64)`. Chapter 14 incorporates every registration crossed by a
bank into the bank hash and shred version.

**Normative table. Complete Mainnet hard-fork registration list at `E`.**

| Registration order | Slot | Multiplicity |
|---:|---:|---:|
| 0 | `13,334,463` | `1` |
| 1 | `53,180,900` | `1` |
| 2 | `96,542,804` | `1` |
| 3 | `96,542,805` | `1` |
| 4 | `131,973,970` | `1` |
| 5 | `135,986,379` | `1` |
| 6 | `153,139,220` | `1` |
| 7 | `179,526,403` | `1` |
| 8 | `246,464,040` | `1` |

The list is ordered by increasing slot. Shred-version derivation applies the single
`SHA256(V || LE(slot,8) || LE(multiplicity,8))` step in Section 14.6 for each row, in
table order. The LastRestartSlot sysvar contains only the greatest registered slot not
above the bank slot (`246,464,040` for every bank in epoch `E`); it is not a serialized
copy of the list.

[SOL-FTR-0008] A validator **MUST NOT** invent, omit, or reorder a hard-fork
registration for Mainnet; disagreement produces a different bank hash and shred version.

[SOL-FTR-0009] After restart, a Consensus Participant **MUST** restore a trusted root
and its safety-critical vote state before emitting a vote; loss of that state **MUST
NOT** be treated as permission to vote on a conflicting fork.

[SOL-FTR-0010] A forward-conforming implementation **MUST** initialize the hard-fork
configuration with exactly the nine rows above and multiplicity one. It need not
reproduce the historical restart procedure for those slots, but **MUST** preserve their
continuing bank-hash, shred-version, and LastRestartSlot effects.
