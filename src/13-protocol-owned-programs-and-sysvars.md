# 13. Protocol-owned programs and sysvars

## 13.1 Identity and deployment model

Protocol-owned behavior is selected by public key plus bank state, not by a
human-readable program name. A program can be a native builtin, a Core-BPF deployment
under an SBF loader, a stateless precompile, or feature-disabled. Migration can retain a
public program identifier while changing its execution owner and code source.

[SOL-PRG-0001] Protocol-owned dispatch **MUST** use the exact program address and form
listed in Section 13.2 for epoch `E`.

[SOL-PRG-0002] For every Core-BPF protocol program, replay **MUST** resolve and
authenticate from the bank the loader, the ProgramData account when applicable, the
executable bytes, the deployment slot, and the upgrade-authority state, then execute
those bytes under Chapter 12.

[SOL-PRG-0003] A validator **MUST** select dispatch from the program account and active
feature state visible in the bank and **MUST NOT** substitute a builtin just because
the public key is historically associated with one.

[SOL-PRG-0004] A builtin-to-Core-BPF migration **MUST** update the program and
ProgramData accounts, capitalization, program resolution, and execution-cost
classification exactly at the activating bank. The migrated bytes **MUST** have
deployment slot `d` and effective slot `d+1`, where `d` is the activating bank's slot.

## 13.2 Known protocol identifiers

**Normative table.**

| Surface | Program ID | Form in epoch `E` |
|---|---|---|
| System program | `11111111111111111111111111111111` | native builtin |
| Vote program | `Vote111111111111111111111111111111111111111` | native builtin |
| Stake program | `Stake11111111111111111111111111111111111111` | Core BPF |
| Config program | `Config1111111111111111111111111111111111111` | Core BPF |
| Feature program | `Feature111111111111111111111111111111111111` | Core BPF |
| Address lookup table | `AddressLookupTab1e1111111111111111111111111` | Core BPF |
| Compute budget | `ComputeBudget111111111111111111111111111111` | native preflight interpreter |
| Native loader | `NativeLoader1111111111111111111111111111111` | reserved builtin owner/dispatcher |
| Deprecated BPF loader | `BPFLoader1111111111111111111111111111111111` | compatibility execution loader |
| BPF loader | `BPFLoader2111111111111111111111111111111111` | compatibility execution loader |
| Upgradeable BPF loader | `BPFLoaderUpgradeab1e11111111111111111111111` | native loader builtin |
| Loader-v4 address | `LoaderV411111111111111111111111111111111111` | reserved and inactive; retained in default-CU and reserved-account classification only |
| ZK ElGamal proof | `ZkE1Gama1Proof11111111111111111111111111111` | native builtin |
| secp256k1 precompile | `KeccakSecp256k11111111111111111111111111111` | precompile |
| Ed25519 precompile | `Ed25519SigVerify111111111111111111111111111` | precompile |
| secp256r1 precompile | `Secp256r1SigVerify1111111111111111111111111` | precompile |
| Incinerator | `1nc1nerator11111111111111111111111111111111` | freeze-time account hook |

[SOL-PRG-0005] A program identifier absent from the epoch-`E` catalog **MUST NOT**
acquire protocol-owned dispatch.

[SOL-PRG-0006] A native builtin **MUST** follow its normative native transition, while a
Core-BPF program **MUST** follow the authenticated deployed bytes selected by
`ResolveProgram`. A public identifier, historical native behavior, or prose summary
**MUST NOT** override the bank-selected form.

## 13.3 Protocol program state

System, vote, stake, config, address-lookup-table, loader, feature, and proof-program
accounts contain consensus state. Their state tags, byte layouts, authorities,
signer/writable requirements, arithmetic, and error codes form part of consensus even
when the program is implemented natively.

[SOL-PRG-0007] An active protocol program **MUST** decode instruction and account-state
bytes with the exact bank-selected format and **MUST** deterministically reject unknown
discriminants, truncation, invalid lengths, invalid authorities, missing signatures, and
prohibited writable or owner combinations.

[SOL-PRG-0008] A protocol program **MUST** apply checked or saturating arithmetic
exactly where its specified transition requires it and **MUST** return the specified
error without committing partial program changes when a precondition fails.

[SOL-PRG-0009] Address-lookup-table resolution **MUST** use only lookup-table state
produced by the active address-lookup-table program and visible under the bank-slot and
SlotHashes rules in Chapter 7.

[SOL-PRG-0010] System durable-nonce initialization, authorization, advancement,
withdrawal, and deallocation **MUST** preserve the nonce state and replay conditions
used by Chapters 7 and 11.

### 13.3.1 System program

System-program instruction data is a Section 5.3 fixed-integer enum. It begins with the
`u32` little-endian discriminant below. A `string` is
`length:u64 || length UTF-8 bytes`, has no terminator, and is limited to 32 bytes for a
derivation seed. Public keys are 32 raw bytes. The decoder permits trailing
instruction-data bytes after a complete variant.

**Normative table.**

| Disc. | Instruction | Exact payload after discriminant | Required instruction accounts |
|---:|---|---|---|
| `0` | CreateAccount | `lamports:u64, space:u64, owner[32]` | `0` funding writable signer; `1` new writable signer |
| `1` | Assign | `owner[32]` | `0` assigned writable signer |
| `2` | Transfer | `lamports:u64` | `0` funding writable signer; `1` recipient writable |
| `3` | CreateAccountWithSeed | `base[32], seed:string, lamports:u64, space:u64, owner[32]` | `0` funding writable signer; `1` created writable; the named base must be an instruction signer |
| `4` | AdvanceNonceAccount | empty | `0` nonce writable; `1` RecentBlockhashes; `2` nonce-authority signer |
| `5` | WithdrawNonceAccount | `lamports:u64` | `0` nonce writable; `1` recipient writable; `2` RecentBlockhashes; `3` Rent; `4` authority signer |
| `6` | InitializeNonceAccount | `authority[32]` | `0` nonce writable; `1` RecentBlockhashes; `2` Rent |
| `7` | AuthorizeNonceAccount | `new_authority[32]` | `0` nonce writable; `1` current-authority signer |
| `8` | Allocate | `space:u64` | `0` allocated writable signer |
| `9` | AllocateWithSeed | `base[32], seed:string, space:u64, owner[32]` | `0` allocated writable; the named base must be an instruction signer |
| `10` | AssignWithSeed | `base[32], seed:string, owner[32]` | `0` assigned writable; the named base must be an instruction signer |
| `11` | TransferWithSeed | `lamports:u64, from_seed:string, from_owner[32]` | `0` funding writable; `1` base signer; `2` recipient writable |
| `12` | UpgradeNonceAccount | empty | `0` nonce writable |
| `13` | CreateAccountAllowPrefund | `lamports:u64, space:u64, owner[32]` | `0` new writable signer; if lamports is nonzero, `1` funding writable signer |

For a seed-derived address, compute `SHA256(base[32] || seed_bytes || owner[32])`; the
result must equal the address being allocated, assigned, created, or debited. Allocation
requires a system-owned account with empty data and `space <= 10,485,760`; assignment
requires empty data; a lamport transfer may debit only a system-owned account with empty
data. CreateAccount additionally requires the destination balance to be zero.
CreateAccountAllowPrefund omits only that zero-balance condition; all allocation,
signer, derivation, funding, and assignment checks remain.

**Normative table. System Program custom-error domain.**

| Value | Exact custom error | First applicable cause |
|---:|---|---|
| `0` | `AccountAlreadyInUse` | a create destination is already allocated or otherwise in use |
| `1` | `ResultWithNegativeLamports` | a checked debit would make the source balance negative |
| `2` | `InvalidProgramId` | assignment/debit ownership or derived owner is not permitted |
| `3` | `InvalidAccountDataLength` | allocation length or existing data length violates the selected instruction |
| `4` | `MaxSeedLengthExceeded` | a seed exceeds 32 bytes |
| `5` | `AddressWithSeedMismatch` | `SHA256(base \|\| seed \|\| owner)` differs from the addressed account |
| `6` | `NonceNoRecentBlockhashes` | nonce initialization/advance lacks the required recent-blockhash state |
| `7` | `NonceBlockhashNotExpired` | the stored durable nonce is still the bank's next durable nonce |
| `8` | `NonceUnexpectedBlockhashValue` | the supplied/stored nonce does not match the transaction marker |

Instruction decode precedes account-list lookup. Thereafter the System Program checks
account count and signer privilege in listed-account order, address/seed derivation,
ownership and state, data length/allocation, then balances. A named custom error takes
precedence at its predicate; otherwise the exact standard `InstructionError` in Section
12.7 applies.

[SOL-PRG-0011] The System program **MUST** decode the table above, enforce the listed
signer and writable privileges, use checked lamport arithmetic, and perform allocation,
assignment, transfer, and durable-nonce transitions atomically. An invalid derivation,
owner, data length, balance, signature, or account list **MUST** fail the instruction
without a partial System-program transition.

### 13.3.2 Upgradeable BPF loader

Upgradeable-loader state begins with a `u32` little-endian discriminant. The fixed
metadata offsets, rather than the serialized length of an absent option, locate
following program bytes.

**Normative table.**

| State disc. | State | Exact metadata and following bytes |
|---:|---|---|
| `0` | Uninitialized | four-byte discriminant `LE(0,4)` followed by zero or more uninterpreted capacity bytes; a `Close` transition truncates the account to the four-byte discriminant |
| `1` | Buffer | byte `4` authority option (`0` absent, `1` present); bytes `[5,37)` hold the authority when present and are ignored otherwise; program bytes begin at `37` |
| `2` | Program | bytes `[4,36)` ProgramData address; metadata length `36` |
| `3` | ProgramData | `[4,12)` last-deployment slot; byte `12` upgrade-authority option; `[13,45)` authority when present and ignored otherwise; SBF bytes begin at `45` |

Management instruction data is another Section 5.3 fixed-integer enum:

**Normative table.**

| Disc. | Instruction | Exact payload after discriminant |
|---:|---|---|
| `0` | InitializeBuffer | empty |
| `1` | Write | `offset:u32, byte_count:u64, bytes[byte_count]` |
| `2` | DeployWithMaxDataLen | `max_data_len:u64` |
| `3` | Upgrade | empty |
| `4` | SetAuthority | empty; optional new-authority account selects Some or None |
| `5` | Close | empty |
| `6` | ExtendProgram | `additional_bytes:u32` |
| `7` | SetAuthorityChecked | empty |

**Normative table. Upgradeable-loader ordered accounts and transitions.**

| Instruction | Ordered instruction accounts | Exact transition |
|---|---|---|
| InitializeBuffer | `0` Buffer `W`; `1` authority | Require Loader-owned Uninitialized state; store Buffer with `Some(key1)`. Epoch-`E` processing requires account 1 even though older construction APIs described it as optional. |
| Write | `0` Buffer `W`; `1` authority `S` | Require matching present Buffer authority; copy payload to `[37+offset,37+offset+byte_count)` without resizing. |
| DeployWithMaxDataLen | `0` payer `W,S`; `1` ProgramData `W`; `2` Program `W`; `3` Buffer `W`; `4` Rent; `5` Clock; `6` System; `7` authority `S` | Derive account 1 from account 2, first transfer every Buffer lamport to payer, then create/fund ProgramData to `45+max_data_len`; validate/copy the Buffer ELF, store ProgramData with current slot and authority, store Program pointing to ProgramData, set Program executable, and truncate Buffer to its 37-byte metadata. |
| Upgrade | `0` ProgramData `W`; `1` Program `W`; `2` Buffer `W`; `3` spill `W`; `4` Rent; `5` Clock; `6` authority `S` | Require matching Program/ProgramData, a deployment slot older than Clock slot, and matching Buffer and ProgramData authority; verify the Buffer ELF, replace ProgramData bytes and slot, zero the unused suffix, retain exactly its rent-exempt lamports, send the excess combined ProgramData/Buffer lamports to spill, and truncate/drain Buffer. |
| SetAuthority | `0` Buffer or ProgramData `W`; `1` current authority `S`; optional `2` new authority | Require the stored present authority to match account 1. Buffer requires account 2; ProgramData may omit it to become immutable. Store the selected authority without changing deployment slot or bytes. |
| Close | `0` Loader-owned state `W`; `1` recipient `W`; optional `2` authority `S`; optional `3` associated Program `W` | Require accounts 0 and 1 distinct. Uninitialized closes without authority. Buffer requires matching authority. ProgramData additionally requires account 3 to point to account 0, a deployment slot older than Clock slot, and matching authority; close ProgramData and make the associated Program unavailable to `ResolveProgram`. Transfer all lamports to account 1 and logically delete account 0 by setting its lamports to zero. |
| ExtendProgram | `0` ProgramData `W`; `1` Program `W`; optional `2` System; optional `3` payer `W,S` | Require matching Program/ProgramData and present upgrade authority. Increase ProgramData length by the payload amount, subject to the 10-KiB/minimum and 10-MiB limits; if its balance is short, transfer the exact rent-exemption shortfall from payer through System. |
| SetAuthorityChecked | `0` Buffer or ProgramData `W`; `1` current authority `S`; `2` new authority `S` | Require both signatures and the matching present authority; store `Some(account2)` without changing slot or bytes. |

ProgramData is derived from the Program address by the Section 5.5 PDA algorithm with
the Program address as the sole seed and the Upgradeable BPF Loader as program ID. Write
ranges must remain within the Buffer data after byte 37. Deploy and upgrade validate the
resulting ELF under Section 12.8.5, copy program bytes into ProgramData after byte 45,
set the deployment slot, and make the Program account executable. An upgrade or
authority change requires the current authority; `SetAuthorityChecked` also requires the
new authority. `ExtendProgram` adds at least 10,240 bytes unless fewer than 10,240 bytes
remain before the 10-MiB account limit, and cannot exceed that limit.

[SOL-PRG-0012] The Upgradeable BPF Loader **MUST** preserve the state tags, metadata
offsets, PDA relationship, authority checks, minimum extension, deployment slot,
executable transition, and bank-visible program bytes above. A successful deploy or
upgrade at slot `d` **MUST** set effective slot `d+1`; the new bytes are unavailable for
invocation in `d` and become resolvable in the child slot.

### 13.3.3 Vote program

Vote-account data is a Section 5.3 fixed-integer enum followed by unused account
capacity. Its first four bytes are a little-endian state discriminant: `0`
Uninitialized, `1` V1_14_11, `2` V3, or `3` V4. The decoder permits unused trailing
bytes. Serialization overwrites the encoded prefix but preserves any remaining account
bytes, which therefore remain part of the account hash.

The component encodings used below are:

**Normative table.**

| Component | Exact encoding |
|---|---|
| optional value | tag byte `0`, or tag byte `1` followed by the value |
| Lockout | `slot:u64, confirmation_count:u32` |
| LandedVote | `latency:u8`, then Lockout |
| authorized-voter map | `count:u64`, then `count` records `epoch:u64, voter[32]` in increasing epoch order |
| prior-voter ring | 32 records `voter[32], first_epoch:u64, last_epoch:u64`, then `next_index:u64`, then `is_empty:u8` |
| epoch credits | `count:u64`, then `count` records `epoch:u64, credits:u64, previous_credits:u64` |
| block timestamp | `slot:u64, unix_timestamp:i64` |

Legacy states remain decodable because an untouched account can retain them. Their exact
field sequences after the four-byte state discriminant are:

**Normative table.**

| State | Account size | Exact field sequence |
|---|---:|---|
| V1_14_11 (`1`) | 3,731 bytes | `node[32], withdrawer[32], commission:u8, vote_count:u64, vote_count Lockout records, optional root_slot, authorized-voter map, prior-voter ring, epoch credits, block timestamp` |
| V3 (`2`) | 3,762 bytes | `node[32], withdrawer[32], commission:u8, vote_count:u64, vote_count LandedVote records, optional root_slot, authorized-voter map, prior-voter ring, epoch credits, block timestamp` |

Every successful epoch-`E` Vote-program mutation stores V4 and grows a shorter legacy
account to 3,762 bytes only if its balance is rent-exempt at that length. The V4 prefix
after discriminant `3` is:

**Normative table.**

| Order | V4 field | Exact encoding |
|---:|---|---|
| 1 | node identity | 32 bytes |
| 2 | authorized withdrawer | 32 bytes |
| 3 | inflation-rewards collector | 32 bytes |
| 4 | block-revenue collector | 32 bytes |
| 5 | inflation commission | `u16` basis points, `0..=10000` |
| 6 | block-revenue commission | `u16` basis points, `0..=10000` |
| 7 | pending delegator rewards | `u64` |
| 8 | BLS voter key | option tag and, when present, 48 compressed BLS12-381 G1 bytes |
| 9 | landed votes | `count:u64`, then that many LandedVote records, oldest first; at most 31 |
| 10 | root slot | optional `u64` |
| 11 | authorized voters | authorized-voter map |
| 12 | epoch credits | epoch-credits sequence, at most 64 records |
| 13 | last timestamp | block timestamp |

Vote instruction data also begins with a `u32` little-endian discriminant and permits
trailing bytes. Only the following forms execute in epoch `E`; any other
Vote-instruction discriminant is invalid instruction data.

**Normative table.**

| Disc. | Active instruction | Exact payload after discriminant |
|---:|---|---|
| `0` | InitializeAccount | `node[32], authorized_voter[32], authorized_withdrawer[32], commission:u8` |
| `1` | Authorize | `new_authority[32]`, then authority-kind |
| `3` | Withdraw | `lamports:u64` |
| `4` | UpdateValidatorIdentity | empty |
| `5` | UpdateCommission | `commission_percent:u8` |
| `7` | AuthorizeChecked | authority-kind; new authority is an instruction account signer |
| `10` | AuthorizeWithSeed | authority-kind, `derived_owner[32], seed:string, new_authority[32]` |
| `11` | AuthorizeCheckedWithSeed | authority-kind, `derived_owner[32], seed:string`; new authority is an instruction account signer |
| `14` | TowerSync | compact tower shown below |
| `15` | TowerSyncSwitch | compact tower, then `switch_proof_hash[32]` |
| `17` | UpdateCommissionCollector | `commission_kind:u32`; kind `0` sets the inflation-rewards collector and kind `1` sets the block-revenue collector |
| `18` | UpdateCommissionBps | `commission_bps:u16, commission_kind:u32`; kind `0` is inflation rewards, and other kinds are unavailable in epoch `E` |

Authority-kind is `LE(0,4)` for a voter, `LE(1,4)` for a withdrawer, or
`LE(2,4) || bls_key[48] || proof_of_possession[96]` for a BLS voter. A seed string uses
`length:u64 || UTF-8 bytes` and is at most 32 bytes.

UpdateCommissionCollector requires at least three instruction accounts (`[0]` the vote
account, `[1]` the new collector, `[2]` the authorized withdrawer) and the authorized
withdrawer's signature. If account `1` is the vote account, the vote address is stored.
Otherwise account `1` must be owned by the System Program (else `InvalidAccountOwner`),
rent-exempt at its current data length under the Rent sysvar (else `InsufficientFunds`),
and writable in this instruction (else `InvalidArgument`), checked in that order after the
signature check. Its address is written to the collector field the kind selects; both
kinds are accepted in epoch `E`.

**Normative table.**

| Compact-tower order | Exact encoding |
|---:|---|
| 1 | root `u64`; `u64::MAX` means no root |
| 2 | lockout count as canonical `compact-u16`, at most 31 |
| 3 | for each lockout: canonical unsigned-LEB128 `u64` offset from the root (or zero when no root) for the first, then from the preceding vote; followed by `confirmation_count:u8` |
| 4 | voted-bank hash `[32]` |
| 5 | timestamp option tag and, when present, `unix_timestamp:i64` |
| 6 | terminal FEC root `[32]` |

Offsets are accumulated with checked `u64` addition and must produce strictly increasing
vote slots. The voted-bank hash must equal SlotHashes for the last vote, and the block
identifier field is the Chapter 14 terminal FEC root for the same slot. Vote lockouts, roots,
authorization, timestamp monotonicity, credit accrual, withdrawals, collector fields,
and commission updates are applied to the V4 fields as specified by Chapters 15 and 16.

[SOL-PRG-0013] The Vote program **MUST** accept only the epoch-`E` instruction forms
above, decode legacy account states exactly, store every successful mutation as V4, and
enforce the named authority, tower, SlotHashes, timestamp, rent, credit, withdrawal, and
commission rules without partially updating the vote account on failure.

**Normative table. Vote Program custom-error domain.**

| Value | Exact custom error | Value | Exact custom error |
|---:|---|---:|---|
| `0` | `VoteTooOld` | `11` | `ConfirmationTooLarge` |
| `1` | `SlotsMismatch` | `12` | `RootRollBack` |
| `2` | `SlotHashMismatch` | `13` | `ConfirmationRollBack` |
| `3` | `EmptySlots` | `14` | `SlotSmallerThanRoot` |
| `4` | `TimestampTooOld` | `15` | `TooManyVotes` |
| `5` | `TooSoonToReauthorize` | `16` | `VotesTooOldAllFiltered` |
| `6` | `LockoutConflict` | `17` | `RootOnDifferentFork` |
| `7` | `NewVoteStateLockoutMismatch` | `18` | `ActiveVoteAccountClose` |
| `8` | `SlotsNotOrdered` | `19` | `CommissionUpdateTooLate` |
| `9` | `ConfirmationsNotOrdered` | `20` | `AssertionFailed` |
| `10` | `ZeroConfirmations` |  |  |

Vote instruction decode and account-count/signature checks precede these semantic
errors. For a vote/tower update, test empty/ordered/bounded tower shape, old-slot
filtering, SlotHashes membership and bank hash, proposed-root ancestry, lockout
consistency, then timestamp monotonicity; the first failing predicate supplies the value
above. Authority/owner/account-data failures use the exact standard `InstructionError`
from Section 12.7.

### 13.3.4 Stake Program v5

A Stake account is exactly 200 bytes, owned by the Stake Program, and decoded with
`FIXINT-LE`. Bytes not occupied by the selected state are retained as zero/unused
capacity and remain committed by the account hash.

**Normative table. Stake v5 state byte layout.**

| State tag / range | Field | Exact encoding |
|---|---|---|
| `[0,4)` | state | `u32`: `0` Uninitialized, `1` Initialized, `2` Stake, `3` RewardsPool |
| `[4,12)` | rent-exempt reserve | `u64` lamports; present for tags 1 and 2 |
| `[12,44)`, `[44,76)` | staker, withdrawer | two public keys; tags 1 and 2 |
| `[76,84)`, `[84,92)`, `[92,124)` | lockup | Unix timestamp `i64` seconds, epoch `u64`, custodian `[32]`; tags 1 and 2 |
| `[124,156)` | delegated vote address | public key; tag 2 |
| `[156,164)` | delegated stake | `u64` lamports; tag 2 |
| `[164,172)`, `[172,180)` | activation/deactivation epoch | two `u64`; `u64::MAX` means bootstrap activation or no deactivation; tag 2 |
| `[180,188)` | reserved rate | eight bytes retained for compatibility; tag 2 |
| `[188,196)` | credits observed | vote credits `u64`; tag 2 |
| `[196,197)` | flags | bit 0 means fully activate before deactivation; bits 1–7 zero; tag 2 |
| remaining through byte 200 | unused capacity | preserved/zero-filled by canonical state writes |

Stake instructions use `FIXINT-LE`: a `u32` discriminant followed by the payload below.
`Option<T>` is tag byte `0`, or `1||T`; `StakeAuthorize` is `u32` `0` staker or `1`
withdrawer. `LockupArgs` is optional `i64`, optional `u64`, optional key; the checked
form omits the key and takes a signed new custodian account. A seed string is
`u64 length || UTF-8` and at most 32 bytes.

**Normative table. Stake v5 instructions and ordered accounts.**

| Disc. | Instruction / payload | Ordered instruction accounts and transition |
|---:|---|---|
| 0 | Initialize: Authorized, Lockup | stake `W`; Rent. Require tag 0, 200 bytes, and rent exemption; store tag 1. |
| 1 | Authorize: new key, kind | stake `W`; Clock; current authority `S`; optional lockup custodian `S`. Replace selected authority; an in-force withdrawer lockup additionally requires custodian. |
| 2 | DelegateStake | stake `W`; vote; Clock; StakeHistory; unused StakeConfig; staker `S`. Require tag 1 or eligible tag 2, valid vote state, and delegation at least 1,000,000,000 lamports after reserve; set voter, stake, activation epoch, no-deactivation, and observed credits. |
| 3 | Split: `u64` lamports | source stake `W`; uninitialized destination `W`; staker `S`. Preserve Meta/delegation proportionally; each nonzero delegated remainder satisfies minimum delegation and each account preserves its rent reserve. |
| 4 | Withdraw: `u64` lamports | stake `W`; recipient `W`; Clock; StakeHistory; withdrawer `S`; optional custodian `S`. Withdraw only lamports not effective/activating stake and not required reserve unless closing; enforce lockup. |
| 5 | Deactivate | stake `W`; Clock; staker `S`. Require delegated state and no prior deactivation; set deactivation to current epoch, subject to flag. |
| 6 | SetLockup: LockupArgs | stake `W`; authority `S`. While lockup is in force require current custodian; otherwise require withdrawer; replace each present field. |
| 7 | Merge | destination `W`; source `W`; Clock; StakeHistory; staker `S`. Require identical authorities/lockup and compatible inactive, activating, or active delegations; active voter and credits must match; add balances/delegation checked and drain source. |
| 8 | AuthorizeWithSeed | stake `W`; base `S`; Clock; optional custodian `S`. Derive current authority by Section 5.5 seed form, then apply disc. 1. |
| 9 | InitializeChecked | stake `W`; Rent; staker; withdrawer `S`. Store tag 1 with no lockup. |
| 10 | AuthorizeChecked: kind | stake `W`; Clock; current authority `S`; new authority `S`; optional custodian `S`. Apply disc. 1 using account 3 as new key. |
| 11 | AuthorizeCheckedWithSeed | stake `W`; base `S`; Clock; new authority `S`; optional custodian `S`. Seed payload supplies kind/seed/owner; apply checked authorization. |
| 12 | SetLockupChecked | stake `W`; current authority `S`; optional new custodian `S`. Apply disc. 6 and take a present custodian from account 2. |
| 13 | GetMinimumDelegation | none. Set return data to `LE(1,000,000,000,8)`. |
| 14 | DeactivateDelinquent | stake `W`; delinquent vote; reference vote. Require stake voter match, the reference voted within the last five epochs, and delinquent did not; set current deactivation epoch. |
| 15 | Redelegate | inactive in `E`; return `InvalidInstructionData`. |
| 16 | MoveStake: `u64` lamports | active source `W`; active or inactive destination `W`; staker `S`. Require identical Meta, fully active source, compatible voter/credits, minimum source remainder and destination delegation; move only active stake. |
| 17 | MoveLamports: `u64` lamports | fully active/inactive source `W`; mergeable destination `W`; staker `S`. Require identical Meta and move only lamports above delegated backing and rent reserve. |

Effective, activating, and deactivating amounts are computed from the Delegation fields
and StakeHistory by Section 16.4 before any withdraw, split, merge, delinquent
deactivation, move, or reward operation. A lockup is in force when
`lockup.unix_timestamp > Clock.unix_timestamp` or `lockup.epoch > Clock.epoch`, unless
its custodian signed. Stake mutations other than reward redemption fail with
`Custom(16)` while EpochRewards is active.

**Normative table. Stake custom-error domain.**

| Value | Name | Value | Name |
|---:|---|---:|---|
| 0 | NoCreditsToRedeem | 9 | InsufficientReferenceVotes |
| 1 | LockupInForce | 10 | VoteAddressMismatch |
| 2 | AlreadyDeactivated | 11 | MinimumDelinquentEpochsForDeactivationNotMet |
| 3 | TooSoonToRedelegate | 12 | InsufficientDelegation |
| 4 | InsufficientStake | 13 | RedelegateTransientOrInactiveStake |
| 5 | MergeTransientStake | 14 | RedelegateToSameVoteAccount |
| 6 | MergeMismatch | 15 | RedelegatedStakeMustFullyActivateBeforeDeactivationIsPermitted |
| 7 | CustodianMissing | 16 | EpochRewardsActive |
| 8 | CustodianSignatureMissing |  |  |

All other failures use the exact standard `InstructionError` named by the failed
predicate: owner, state bytes, account count, signature, authority, funds, rent, or
checked arithmetic are tested in that order after instruction decode. Stake reward
redemption uses the vote-credit and commission algorithm in Section 16.6. An ordinary
zero-lamport reward does not advance `credits_observed`; only the activation-epoch and
credits-rewind forced updates named there do so without a payout.

### 13.3.5 Config Program

A Config account is owned by the Config Program. Its data begins with a canonical
`ShortU16` key count `N`, then `N` records of public key `[32]` and Boolean signer byte,
followed by opaque configuration bytes. The entire instruction data uses the same layout
and replaces the account prefix; there is no instruction discriminant.

**Normative table. Config transition and exact precedence.**

| Step | Rule / exact error |
|---:|---|
| 1 | Decode incoming key list; malformed input is `InvalidInstructionData`. |
| 2 | Account 0 must be Config-owned (`InvalidAccountOwner`) and current key prefix must decode (`InvalidAccountData`). |
| 3 | If the current signer list is empty, account 0 must sign (`MissingRequiredSignature`). |
| 4 | Visit incoming signer records in order. For each signer other than account 0, consume the next instruction account and require the same key and signer privilege; account-0 records require account 0 to sign. On an initialized config, every incoming signer must occur in the current signer set. First failure is `MissingRequiredSignature`. |
| 5 | Duplicate `(key,is_signer)` records are `InvalidArgument`. If the number of current signer records exceeds the number supplied, return `MissingRequiredSignature`. |
| 6 | If account data length is smaller than instruction-data length, return `InvalidInstructionData`; otherwise copy the complete instruction bytes to offset zero and leave the remaining account bytes unchanged. |

The fixed Core-BPF behavior consumes SBF compute under Chapter 12; the former native
450-CU dispatch constant is not applied. This transition is the only Config instruction
and provides no interpretation of the opaque suffix.

### 13.3.6 Feature Program

The single authoritative Feature-account layout and decoder are in Section 17.1. A
Feature-Program-created account has the nine-byte canonical length; the bank feature
scan applies Section 17.1's explicitly broader accepted domain for already-present
account state.

Feature activation is not a Feature-Program instruction. It is an atomic transaction
sequence that System-transfers the rent-exempt balance, allocates nine bytes, and
assigns the signed feature address to the Feature Program. At the next new-bank feature
scan, a pending account selected for activation is written as `1||LE(current_slot,8)`;
the behavioral side effects in Appendix C and Section 17.4 are applied in that same bank
before transactions.

The sole Feature instruction is byte `00`, RevokePendingActivation, with feature account
0 `W,S`, incinerator account 1 `W`, and System Program account 2 readonly. Require
Feature ownership and a decodable pending state; an active state returns `Custom(0)`
(`FeatureAlreadyActivated`). On success transfer all lamports to the incinerator, resize
the feature account to zero, and assign it to the System Program. Revocation after
activation cannot undo a rule. A Core-BPF migration changes only execution form; it
neither activates nor revokes any Feature account.

### 13.3.7 Address Lookup Table Program

Lookup-table instruction data is `FIXINT-LE`: `u32` discriminants `0` Create
(`recent_slot:u64,bump:u8`), `1` Freeze, `2` Extend (`count:u64` then that many 32-byte
addresses), `3` Deactivate, and `4` Close. State is `u32` tag `0` Uninitialized or tag
`1` LookupTable followed by the metadata below.

**Normative table. Lookup-table account bytes.**

| Range | Field |
|---|---|
| `[0,4)` | state tag `1` |
| `[4,12)` | deactivation slot; `u64::MAX` means active |
| `[12,20)` | last-extended slot |
| `[20,21)` | start index of the addresses appended in that slot |
| `[21,22)` | authority option tag |
| `[22,54)` | authority when present, ignored/zero when absent |
| `[54,56)` | zero `u16` padding |
| `[56,56+32*N)` | `N`, `0..=256`, consecutive addresses |

**Normative table. Lookup-table lifecycle.**

| Instruction | Ordered accounts and exact transition |
|---|---|
| Create | table `W`; authority; payer `W,S`; System. `recent_slot` must occur in SlotHashes and table must equal PDA(`authority\|\|LE(recent_slot,8)`, bump). If already ALT-owned, succeed idempotently. Otherwise fund to `max(1,rent_minimum(56))`, allocate 56, assign ALT owner, and store active metadata with authority. |
| Extend | table `W`; authority `S`; optional payer `W,S` and System when more rent is required. Require owner, present matching authority, active state, at least one new key, and total `<=256`. On the first extension in a slot, store the old `N` as start index; append keys and fund new rent minimum. |
| Freeze | table `W`; authority `S`. Require present matching authority, active state, and `N>0`; set authority absent. A frozen table is immutable and cannot close. |
| Deactivate | table `W`; authority `S`. Require present matching authority and active state; set deactivation slot to current Clock slot. |
| Close | table `W`; authority `S`; recipient `W`. Require distinct table/recipient, present matching authority, and deactivation slot absent from SlotHashes and unequal current slot. Transfer all lamports, set table data length and lamports to zero. |

An active/deactivating table remains usable while its deactivation slot is the current
slot or occurs in SlotHashes. It becomes deactivated when that slot is absent. For
lookup in current slot `s`, if `s == last_extended_slot`, only indices below
`last_extended_slot_start_index` are visible; otherwise all `N` are visible. Invalid
state/owner/index map to the four exact transaction errors in Section 7.8 during message
loading. Program-instruction failures use `InvalidAccountOwner`,
`MissingRequiredSignature`, `Immutable`, `IncorrectAuthority`,
`AccountAlreadyInitialized`, `InvalidArgument`, or `InvalidInstructionData` in the
predicate order shown above.

### 13.3.8 Native Loader and BPF loader execution

The Native Loader address is a reserved dispatch owner, not a general-purpose
instruction program. Directly invoking it returns `UnsupportedProgramId`. A
Native-Loader-owned program account dispatches only when its key is in the bank's
builtin registry; an unregistered key produces
`TransactionError::InvalidProgramForExecution` during loading. The program's own
instruction accounts exclude the loader-owner account.

The deprecated and current BPF loaders may execute already-deployed program accounts
using ABI v0 and v1 respectively, but direct management instructions return
`UnsupportedProgramId` after the fixed loader charge in Section 12.8.8. The Upgradeable
Loader alone accepts the management state machine in Section 13.3.2. CPI directly to any
loader, and CPI to an Upgradeable-Loader Upgrade, SetAuthority, SetAuthorityChecked, or
Close instruction, returns `UnsupportedProgramId` before the nested invocation begins.

### 13.3.9 ZK ElGamal Proof Program

Instruction byte zero is the `u8` discriminant below. For a verify instruction, the
remaining bytes are either the exact proof-data POD value or exactly four bytes
`proof_offset:u32`; the latter reads that POD at the offset in readonly account 0. A
verification with no context needs no more accounts. To store the verified statement,
supply a writable context account followed by the context authority; when proof bytes
came from account 0 those are accounts 1 and 2.

**Normative table. ZK ElGamal proof instructions and complete POD envelopes.**

| Disc. | Instruction | Context POD, in byte order | Context bytes | Proof bytes | Inline proof-data bytes | Stored context-state bytes | CU |
|---:|---|---|---:|---:|---:|---:|---:|
| 0 | CloseContextState | not applicable | — | — | exactly 1 | existing account length | 3,300 |
| 1 | VerifyZeroCiphertext | `pubkey[32], ciphertext[64]` | 96 | 96 | 193 | 129 | 6,000 |
| 2 | VerifyCiphertextCiphertextEquality | `first_pubkey[32], second_pubkey[32], first_ciphertext[64], second_ciphertext[64]` | 192 | 224 | 417 | 225 | 8,000 |
| 3 | VerifyCiphertextCommitmentEquality | `pubkey[32], ciphertext[64], commitment[32]` | 128 | 192 | 321 | 161 | 6,400 |
| 4 | VerifyPubkeyValidity | `pubkey[32]` | 32 | 64 | 97 | 65 | 2,600 |
| 5 | VerifyPercentageWithCap | `percentage_commitment[32], delta_commitment[32], claimed_commitment[32], max_value_le[8]` | 104 | 256 | 361 | 137 | 6,500 |
| 6 | VerifyBatchedRangeProofU64 | `commitments[8][32], bit_lengths[8]` | 264 | 672 | 937 | 297 | 111,000 |
| 7 | VerifyBatchedRangeProofU128 | same | 264 | 736 | 1,001 | 297 | 200,000 |
| 8 | VerifyBatchedRangeProofU256 | same | 264 | 800 | 1,065 | 297 | 368,000 |
| 9 | VerifyGroupedCiphertext2HandlesValidity | `pubkey0[32], pubkey1[32], grouped_ciphertext[96]` | 160 | 160 | 321 | 193 | 6,400 |
| 10 | VerifyBatchedGroupedCiphertext2HandlesValidity | `pubkey0[32], pubkey1[32], grouped_ciphertext_lo[96], grouped_ciphertext_hi[96]` | 256 | 160 | 417 | 289 | 13,000 |
| 11 | VerifyGroupedCiphertext3HandlesValidity | `pubkey0[32], pubkey1[32], pubkey2[32], grouped_ciphertext[128]` | 224 | 192 | 417 | 257 | 8,100 |
| 12 | VerifyBatchedGroupedCiphertext3HandlesValidity | `pubkey0[32], pubkey1[32], pubkey2[32], grouped_ciphertext_lo[128], grouped_ciphertext_hi[128]` | 352 | 192 | 545 | 385 | 16,400 |

Every inline proof envelope is exactly `discriminant:u8 || context_POD || proof_POD`,
with no padding or trailing bytes. The five-byte account form is exactly
`discriminant:u8 || proof_offset:u32_le`; its readonly proof account must contain
`context_POD || proof_POD` at
`[proof_offset, proof_offset + context_bytes + proof_bytes)`. A stored context state is
exactly `authority[32] || proof_type:u8 || context_POD`, which yields the lengths in the
table.

All keys, ciphertext components, commitments, and proof points are canonical compressed
Ristretto encodings; each proof scalar is canonical modulo the Ristretto scalar order.
An ElGamal ciphertext is `commitment[32] || decryption_handle[32]`; a two- or
three-handle grouped ciphertext is one commitment followed by two or three 32-byte
handles. The proof relations are, respectively: encryption of zero; equality of the two
ciphertext plaintexts; equality of a ciphertext plaintext and a Pedersen commitment
opening; knowledge of the secret key corresponding to the supplied public key; either
`percentage == max_value` or `delta == claimed`; membership of every committed value in
its declared unsigned bit range; and validity of every grouped decryption handle for one
shared commitment. Batched grouped proofs check both the low and high grouped ciphertext
under the same supplied keys.

For a batched range context, selected commitments are the prefix before the first
all-zero 32-byte commitment, so there are at most eight. The equally long prefix of
`bit_lengths` supplies the selected unsigned `u8` widths; its sum must be exactly `64`,
`128`, or `256` for discriminants 6, 7, or 8. The remaining commitment and length slots
are not selected, but their raw bytes remain bound into the transcript and are not
required to be zero. Verification uses the named proof's fixed Fiat–Shamir transcript
and rejects a noncanonical scalar, invalid compressed point, identity commitment where
prohibited, malformed range vector, or false relation. Any POD decode or verification
failure is `InvalidInstructionData` after the row's full CU is charged.

A context account is Proof-Program-owned and exactly
`authority[32] || proof_type:u8 || context_POD`. It must contain proof type 0
Uninitialized and have exactly the resulting length; otherwise return
`AccountAlreadyInitialized` or `InvalidAccountData`. Success replaces the entire bytes.
Close takes context `W`, distinct destination `W`, authority `S`; it validates owner,
initialized type, and exact authority in that order, transfers all lamports, truncates
data, and assigns System ownership. Failures are respectively
`MissingRequiredSignature`, `InvalidInstructionData`, `InvalidAccountOwner`,
`UninitializedAccount`, or `InvalidAccountOwner` for an authority mismatch.

### 13.3.10 Core-BPF migration and upgrade side effects

At an activating bank, a protocol-owned migration first authenticates the source
Upgradeable-Loader Buffer state, its configured address and authority, and any
configured verified-build hash; authenticates the target's expected
builtin/stateless/Core-BPF form; derives the target ProgramData PDA; constructs the
Program and ProgramData states from Section 13.3.2; and validates/deploys the source ELF
under the bank's Chapter 12 environment. No bank account is changed if any check fails.

On success it replaces the target Program/ProgramData accounts, logically deletes the
source buffer by setting its lamports to zero, removes any native builtin registration,
and adjusts capitalization by the following checked expression:

**Normative algorithm.**

```text
new_program_lamports + new_programdata_lamports
    - old_target_lamports - old_programdata_lamports - buffer_lamports
```

A Core-BPF upgrade retains the Program account and authority,
replaces only ProgramData, clears the buffer, updates the deployment slot, and adjusts
the same capitalization components. For migration or upgrade slot `d`, the new bytes
have effective slot `d+1`; `ResolveProgram` does not dispatch the old or new form during
`d`. These are bank-creation transitions, not transaction instructions, so a configured
authentication/ELF/hash failure is bank-fatal rather than a transaction error.

For a Core-BPF row, execution follows Chapter 12 using deployed account bytes in the
bank. The public program identifier and migration feature do not substitute for the
Program/ProgramData relationship or executable bytes.

## 13.4 Sysvar account identities

Sysvar accounts use owner `Sysvar1111111111111111111111111111111111111` unless a
specific protocol exception is specified. Known addresses are:

**Normative table.**

| Sysvar | Address | Lifetime/update role |
|---|---|---|
| Clock | `SysvarC1ock11111111111111111111111111111111` | updated for each child bank before transactions |
| EpochRewards | `SysvarEpochRewards1111111111111111111111111` | initialized/updated during partitioned rewards |
| EpochSchedule | `SysvarEpochSchedu1e111111111111111111111111` | genesis-derived schedule |
| Instructions | `Sysvar1nstructions1111111111111111111111111` | transaction-local synthesized account |
| LastRestartSlot | `SysvarLastRestartS1ot1111111111111111111111` | updated before transactions when required |
| RecentBlockhashes | `SysvarRecentB1ockHashes11111111111111111111` | legacy queue view updated at block boundary |
| Rent | `SysvarRent111111111111111111111111111111111` | rent-exempt minimum parameters |
| SlotHashes | `SysvarS1otHashes111111111111111111111111111` | parent-slot/hash ancestry updated before transactions |
| SlotHistory | `SysvarS1otHistory11111111111111111111111111` | updated during freeze |
| StakeHistory | `SysvarStakeHistory1111111111111111111111111` | updated at epoch transition |

Sysvar account data uses the Chapter 5 fixed-width and sequence rules. There is no
host-structure padding. The exact data grammars are:

**Normative table.**

| Sysvar | Exact serialized data |
|---|---|
| Clock | `slot:u64`, `epoch_start_timestamp:i64`, `epoch:u64`, `leader_schedule_epoch:u64`, `unix_timestamp:i64`; exactly 40 bytes |
| EpochSchedule | `slots_per_epoch:u64`, `leader_schedule_slot_offset:u64`, `warmup:u8 Boolean`, `first_normal_epoch:u64`, `first_normal_slot:u64`; exactly 33 bytes |
| EpochRewards | `distribution_starting_block_height:u64`, `num_partitions:u64`, `parent_blockhash[32]`, `total_points:u128`, `total_rewards:u64`, `distributed_rewards:u64`, `active:u8 Boolean`; exactly 81 bytes |
| LastRestartSlot | `last_restart_slot:u64`; exactly 8 bytes |
| RecentBlockhashes | `u64` count, at most 150 records of `blockhash[32], lamports_per_signature:u64` in newest-to-oldest order |
| Rent | `lamports_per_byte:u64`, eight threshold bytes, `burn_percent:u8`; exactly 17 bytes |
| SlotHashes | `u64` count, at most 512 records of `slot:u64, hash[32]` in descending slot order |
| StakeHistory | `u64` count, at most 512 records of `epoch:u64, effective:u64, activating:u64, deactivating:u64` in descending epoch order |
| SlotHistory | option tag `1`, block count `LE(16384,8)`, 16,384 `u64` little-endian blocks, bit length `LE(1048576,8)`, then `next_slot:u64`; exactly 131,097 bytes |

SlotHistory bit `i` is the least-significant-bit-first bit `(i mod 64)` in block
`floor(i/64)`. A slot `s` in `[next_slot.saturating_sub(1048576), next_slot)` is present
exactly when bit `s mod 1048576` is set; a slot at or above `next_slot` is future and an
older slot is too old. Mainnet EpochSchedule bytes are `(432000, 432000, false, 0, 0)`.
Mainnet Rent bytes are `LE(5080,8) || 00 00 00 00 00 00 f0 3f || 32`; the final byte is
decimal 50 (`0x32`) and the middle bytes are the little-endian IEEE-754 representation
of `1.0` retained for compatibility.

[SOL-PRG-0014] A persistent sysvar account **MUST** have its canonical address, Sysvar
owner, a false executable flag, a rent-exempt balance, and the exact serialized data for
the child bank.

[SOL-PRG-0015] A sysvar update **MUST** replace the account data using the canonical
serializer and **MUST** adjust capitalization for any protocol-funded rent-exempt
balance change.

[SOL-PRG-0016] The sysvar cache supplied to builtins, SBF syscalls, and loaders **MUST**
reflect the same values as the child bank's canonical sysvar state at the invocation
point.

## 13.5 Sysvar update phases

[SOL-PRG-0017] Before transaction execution, SlotHashes **MUST** incorporate the parent
slot and parent bank hash according to its bounded history rule, and Clock **MUST**
identify the child slot and epoch with the stake-weighted timestamp result below.

[SOL-PRG-0018] StakeHistory **MUST** be updated exactly when the child crosses an epoch
boundary, and LastRestartSlot **MUST** reflect the greatest qualifying hard-fork or
restart slot visible to the child.

[SOL-PRG-0019] EpochRewards **MUST** be initialized and updated in lockstep with the
partitioned epoch-reward state transition, including its active/inactive status and
distributed amount.

[SOL-PRG-0020] At the slot boundary, RecentBlockhashes **MUST** reflect the active
recent-blockhash queue after terminal hash registration; during freeze, SlotHistory
**MUST** record the child slot before bank hashing.

The bounded histories, timestamp estimator, reward fields, and deprecated-sysvar
availability are selected by the bank feature state and the relevant protocol-program
transition.

For child slot `s`, retain each vote account's most recent timestamp only when its
reported slot `r <= s` and `s-r <= 432000`. Advance that timestamp by the PoH duration
from `r+1` through `s`, group equal advanced estimates, and sum the epoch stake
delegated to each group with saturating `u128` arithmetic. The estimate is the smallest
timestamp whose cumulative stake in ascending timestamp order is strictly greater than
half the total included stake. If no stake is included, retain the parent's timestamp.
Relative to the epoch-start PoH estimate, clamp an estimate that is too far behind to
25% behind and one that is too far ahead to 150% ahead; finally, do not permit the child
Unix timestamp to precede its parent's. On an epoch boundary, the resulting value also
becomes `epoch_start_timestamp`.

## 13.6 Instructions sysvar

The instructions sysvar is transaction-scoped. It contains a serialization of the
top-level message instructions and a mutable current top-level instruction index used
during execution; it is not loaded from the parent account database.

**Normative table.**

| Region | Exact encoding |
|---|---|
| header | instruction count `N:u16`, followed by `N` little-endian `u16` offsets from byte zero to each instruction record |
| instruction record | account count `A:u16`; `A` account-meta records; program ID `[32]`; data length `D:u16`; `D` data bytes |
| account-meta record | flags `u8` then public key `[32]`; flag bit 0 is signer, bit 1 writable, and bits 2–7 are zero |
| final field | current top-level instruction index `u16` in the last two data bytes |

Every offset points at the corresponding record's account count. Records are stored in
top-level message order and account metas retain instruction order. All counts, offsets,
and total size must fit `u16`.

[SOL-PRG-0021] When the instructions-sysvar key is present in a transaction, account
loading **MUST** synthesize its data from that transaction's complete top-level
instruction list and **MUST NOT** use any persistent account value at the same address.

[SOL-PRG-0022] Immediately before each top-level instruction is invoked, the runtime
**MUST** set the synthesized instructions sysvar's current index to that instruction's
zero-based message index.

[SOL-PRG-0023] CPI **MUST NOT** append callee instructions to the synthesized top-level
instruction list or change the current index away from the enclosing top-level
instruction.

## 13.7 Reserved protocol accounts and incineration

The epoch-`E` set has no pending or conditionally writable entries. Every row is an
always-readonly base entry; no feature adds or removes it in `E`.

**Normative table. Complete reserved-account set.**

| Reserved surface | Exact address | Feature transition | Writability |
|---|---|---|---|
| Address Lookup Table Program | `AddressLookupTab1e1111111111111111111111111` | base | always readonly |
| BPF Loader | `BPFLoader2111111111111111111111111111111111` | base | always readonly |
| deprecated BPF Loader | `BPFLoader1111111111111111111111111111111111` | base | always readonly |
| Upgradeable BPF Loader | `BPFLoaderUpgradeab1e11111111111111111111111` | base | always readonly |
| Compute Budget Program | `ComputeBudget111111111111111111111111111111` | base | always readonly |
| Config Program | `Config1111111111111111111111111111111111111` | base | always readonly |
| Ed25519 precompile | `Ed25519SigVerify111111111111111111111111111` | base | always readonly |
| Feature Program | `Feature111111111111111111111111111111111111` | base | always readonly |
| abandoned Loader-v4 address | `LoaderV411111111111111111111111111111111111` | base; retained permanently | always readonly |
| secp256k1 precompile | `KeccakSecp256k11111111111111111111111111111` | base | always readonly |
| secp256r1 precompile | `Secp256r1SigVerify1111111111111111111111111` | base | always readonly |
| Stake Config | `StakeConfig11111111111111111111111111111111` | base | always readonly |
| Stake Program | `Stake11111111111111111111111111111111111111` | base | always readonly |
| System Program | `11111111111111111111111111111111` | base | always readonly |
| Vote Program | `Vote111111111111111111111111111111111111111` | base | always readonly |
| ZK ElGamal Proof Program | `ZkE1Gama1Proof11111111111111111111111111111` | base | always readonly |
| ZK Token Proof Program | `ZkTokenProof1111111111111111111111111111111` | base | always readonly |
| Clock sysvar | `SysvarC1ock11111111111111111111111111111111` | base | always readonly |
| EpochRewards sysvar | `SysvarEpochRewards1111111111111111111111111` | base | always readonly |
| EpochSchedule sysvar | `SysvarEpochSchedu1e111111111111111111111111` | base | always readonly |
| Fees sysvar | `SysvarFees111111111111111111111111111111111` | base, deprecated | always readonly |
| Instructions sysvar | `Sysvar1nstructions1111111111111111111111111` | base | always readonly |
| LastRestartSlot sysvar | `SysvarLastRestartS1ot1111111111111111111111` | base | always readonly |
| RecentBlockhashes sysvar | `SysvarRecentB1ockHashes11111111111111111111` | base, deprecated | always readonly |
| Rent sysvar | `SysvarRent111111111111111111111111111111111` | base | always readonly |
| Rewards sysvar | `SysvarRewards111111111111111111111111111111` | base, deprecated | always readonly |
| SlotHashes sysvar | `SysvarS1otHashes111111111111111111111111111` | base | always readonly |
| SlotHistory sysvar | `SysvarS1otHistory11111111111111111111111111` | base | always readonly |
| StakeHistory sysvar | `SysvarStakeHistory1111111111111111111111111` | base | always readonly |
| Native Loader | `NativeLoader1111111111111111111111111111111` | base | always readonly |
| Sysvar owner | `Sysvar1111111111111111111111111111111111111` | base | always readonly |

[SOL-PRG-0024] Immediately after address resolution, sanitation **MUST** demote every
row above to readonly. Demotion changes only the effective writable bitmap: the signed
message bytes and its three header counts remain unchanged. A reserved address requested
writable still contributes one requested-writable-account cost in Section 10.4.1, but
it is not included in effective-writable account cost attribution and does not change
fee calculation.

[SOL-PRG-0025] At freeze, a present incinerator account **MUST** be replaced with the
default account and its lamports **MUST** be removed from capitalization before the
accounts commitment and bank hash are finalized.

[SOL-PRG-0026] Lamports sent to the incinerator after its processing point **MUST NOT**
be retroactively burned in an earlier bank.

[SOL-PRG-0027] A transaction that attempts a state change through a demoted reserved
account **MUST** fail at instruction validation with the applicable
`ReadonlyLamportChange` or `ReadonlyDataModified`; the message is not rejected merely
for having requested that reserved key writable.

[SOL-PRG-0028] A Core-BPF program **MUST** execute the authenticated deployed bytes
selected by `ResolveProgram`; prose in this chapter is not authority to substitute a
native implementation. The Stake, Config, Feature, and Address Lookup Table state
decoders and cross-program rules in Sections 13.3.4 through 13.3.10 remain normative
where the rest of the protocol consumes those accounts. Any mismatch between the prose
and authenticated execution is a defect in this specification, not an alternative
state transition.

## 13.8 Core-BPF programs

The following subsections are an index, not duplicate definitions. Each program's single
authoritative state machine is the referenced Section 13.3 subsection; Chapter 12
supplies its SBF execution semantics and the bank's authenticated Program/ProgramData
bytes supply its code.

### 13.8.1 Stake Program v5

The complete active instruction set, 200-byte state, transitions, and custom error
domain are defined once in Section 13.3.4.

### 13.8.2 Config Program

The key-prefix and opaque-suffix state, sole update transition, and exact error
precedence are defined once in Section 13.3.5.

### 13.8.3 Feature Program

Pending/active state, activation, revocation, and migration behavior are defined once in
Sections 13.3.6 and 17.1–17.4.

### 13.8.4 Address Lookup Table Program

Instruction bytes, 56-byte metadata, address visibility, and the complete lifecycle are
defined once in Sections 13.3.7 and 7.4.

## 13.9 Deployment dependence

Core-BPF executable bytes and upgrade-authority state are ordinary bank state. An
implementation restoring or replaying a bank authenticates those accounts through the
state commitment and executes the version visible at that bank, as required by
`SOL-PRG-0002`; this document does not embed mutable deployed bytes.
