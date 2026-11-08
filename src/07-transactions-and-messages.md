# 7. Transactions and messages

## 7.1 Scope and notation

This chapter specifies the three Mainnet message formats in epoch `E`: legacy, version
0, and version 1. It defines their envelopes, sanitation, address resolution, signature
checks, compute-budget interpretation, fee-payer validation, and account loading.

Every `compact-u16` in this chapter is the single authoritative `ShortU16` codec in
Section 5.4; this chapter does not define a second compact-integer encoding.

[SOL-TXN-0001] A transaction decoder **MUST** apply the complete Section 5.4
canonicality and range checks to every `compact-u16` field before using its value as a
count or length.

Unless a rule names a later disposition, failure of envelope parsing or message
sanitation makes the transaction unprocessable: it consumes no fee, executes no
instruction, and creates no replay-cache entry.

## 7.2 Transaction envelope and message encodings

The legacy and version-0 transaction envelope is:

**Normative table.**

| Order | Field | Encoding / length |
|---:|---|---|
| 1 | `signature_count` | canonical `compact-u16` |
| 2 | `signatures` | `signature_count` consecutive 64-byte Ed25519 signatures |
| 3 | `message` | all remaining bytes, in one of the message layouts below |

The bytes from the first byte of `message` through the end of the transaction are the
signed message bytes.

**Normative table.**

| Order | Legacy message field | Encoding / length |
|---:|---|---|
| 1 | `header` | three bytes, detailed below |
| 2 | `static_account_count` | canonical `compact-u16` |
| 3 | `static_account_keys` | `static_account_count` consecutive 32-byte keys |
| 4 | `recent_blockhash` | 32 bytes |
| 5 | `instruction_count` | canonical `compact-u16` |
| 6 | `instructions` | `instruction_count` consecutive compiled-instruction records |

**Normative table.**

| Order | Version-0 message field | Encoding / length |
|---:|---|---|
| 1 | version prefix | one byte `0x80` |
| 2 | legacy fields | every legacy field above, starting with its three-byte header |
| 3 | `lookup_count` | canonical `compact-u16` |
| 4 | `address_table_lookups` | `lookup_count` consecutive lookup records |

Version 1 replaces the preceding envelope and vector codecs with a bounded layout. Its
signatures follow the message rather than preceding it, and it does not carry a
signature-count field. The header's `required_signature_count` determines their count.

**Normative table. Version-1 transaction layout.**

| Order | Field | Encoding / length |
|---:|---|---|
| 1 | version | one byte `0x81` |
| 2 | header | three bytes in the header table below |
| 3 | configuration mask | `u32` little-endian |
| 4 | lifetime blockhash | 32 bytes |
| 5 | instruction count | `u8` |
| 6 | address count | `u8` |
| 7 | static account keys | `address_count` consecutive 32-byte keys |
| 8 | configuration values | fields selected by the mask, in increasing bit order |
| 9 | instruction headers | `instruction_count` records, four bytes each |
| 10 | instruction account indices | concatenated records in instruction order |
| 11 | instruction data | concatenated records in instruction order |
| 12 | signatures | `required_signature_count` consecutive 64-byte signatures |

The signed message is every byte from the `0x81` version byte through the final byte of
the concatenated instruction data. Signatures are not included in the signed message.

**Normative table. Version-1 configuration mask.**

| Bit | Field when set | Encoding | Absent value |
|---:|---|---|---:|
| `0` | priority-fee low word | `u32` little-endian | `0` |
| `1` | priority-fee high word | `u32` little-endian | `0` |
| `2` | compute-unit limit | `u32` little-endian | `0` compute units |
| `3` | loaded-account-data limit | `u32` little-endian | `0` bytes |
| `4` | heap-frame size | `u32` little-endian | `32,768` bytes |

Bits 0 and 1 are one `u64` little-endian priority fee and **MUST** either both be set or
both be clear. Bits 5 through 31 are invalid.

**Normative table. Version-1 instruction header.**

| Offset | Field | Encoding |
|---:|---|---|
| `0` | program-ID index | `u8` |
| `1` | account-index count | `u8` |
| `2` | instruction-data length | `u16` little-endian |

For each instruction, the declared account indices occur in the shared account-index
region and the declared data bytes occur in the shared data region. Both regions
preserve instruction order.

**Normative table.**

| Header offset | Field | Encoding |
|---:|---|---|
| 0 | `required_signature_count` | `u8` |
| 1 | `readonly_signed_count` | `u8` |
| 2 | `readonly_unsigned_count` | `u8` |

**Normative table.**

| Order | Compiled-instruction field | Encoding / length |
|---:|---|---|
| 1 | `program_id_index` | `u8` |
| 2 | `account_index_count` | canonical `compact-u16` |
| 3 | `account_indices` | `account_index_count` bytes |
| 4 | `data_length` | canonical `compact-u16` |
| 5 | `data` | `data_length` bytes |

**Normative table.**

| Order | Address-table-lookup field | Encoding / length |
|---:|---|---|
| 1 | `table_account_key` | 32 bytes |
| 2 | `writable_index_count` | canonical `compact-u16` |
| 3 | `writable_table_indices` | `writable_index_count` bytes |
| 4 | `readonly_index_count` | canonical `compact-u16` |
| 5 | `readonly_table_indices` | `readonly_index_count` bytes |

[SOL-TXN-0002] An accepted legacy or version-0 transaction **MUST** be at most 1,232
bytes, contain between 1 and 12 signatures inclusive, and have
`signature_count == required_signature_count`.

[SOL-TXN-0003] A decoder **MUST** interpret a message first byte with its high bit clear
as the legacy header's `required_signature_count`. In the legacy/version-0 envelope it
**MUST** accept a high-bit-set message byte only when it is `0x80`. An entire transaction
whose first byte is `0x81` selects the version-1 grammar instead.

[SOL-TXN-0004] A decoder **MUST** consume every byte of the transaction according to the
selected grammar and **MUST** reject truncation, integer overflow, a vector extending
past the envelope, or trailing bytes.

[SOL-TXN-0035] A version-1 transaction **MUST** be at most `4,096` bytes, contain
`1..=12` signatures, `1..=64` addresses, and at most `64` top-level instructions. Each
instruction **MUST** reference at most `255` accounts, every account and program index
**MUST** be less than the address count, the program index **MUST** be nonzero, and all
account keys **MUST** be distinct. Version 1 **MUST NOT** contain address-table lookups.

[SOL-TXN-0036] A version-1 heap request **MUST** be a multiple of `1,024` bytes in
`[32,768,262,144]`. The decoder **MUST** reject an unknown configuration bit, only one
priority-fee word, an overflowing concatenated region, or trailing bytes. Compute Budget
program instructions remain executable successful no-ops for version-1 configuration;
the header configuration is authoritative, and those instructions still incur their
ordinary instruction and execution cost.

## 7.3 Header and account-key sanitation

Let `S` be the ordered static account keys. For version 0, let `LW` be addresses
selected by writable lookup indices in lookup-entry order and let `LR` be addresses
selected by readonly lookup indices in lookup-entry order. The complete account-key
vector is `K = S || LW || LR`.

[SOL-TXN-0005] The static-key count **MUST** be at least `required_signature_count`,
`readonly_signed_count` **MUST** be less than `required_signature_count`, and
`readonly_unsigned_count` **MUST** be no greater than `|S| - required_signature_count`.

Static keys are classified in four consecutive ranges: writable signed, readonly signed,
writable unsigned, and readonly unsigned. Version-0 loaded writable keys follow every
static key and are writable unsigned; loaded readonly keys follow them and are readonly
unsigned.

[SOL-TXN-0006] Account-key resolution **MUST** preserve the order `S || LW || LR`, and
every instruction account index **MUST** be less than `|K|`.

[SOL-TXN-0007] Every compiled instruction's `program_id_index` **MUST** be nonzero and
less than `|S|`; a program identifier **MUST NOT** be supplied by an address-table
lookup.

[SOL-TXN-0008] Only after every version-0 address lookup has resolved, the runtime
**MUST** first reject `|K| > 128` as `TransactionError::TooManyAccountLocks`, then scan
`K` in order and reject its first duplicate public key as
`TransactionError::AccountLoadedTwice`. A duplicate among static and loaded addresses is
detected by this same scan; the implementation **MUST NOT** report the duplicate before
an excessive resolved lock count.

[SOL-TXN-0009] The runtime **MUST** apply the reserved-key set in Section 13.7 when
deriving writability, and a reserved key **MUST NOT** remain writable when the
transaction reaches account locking or execution.

The transaction signature vector is not part of `K`. The first
`required_signature_count` static keys identify the signature-verification public keys
in order.

## 7.4 Version-0 address-table resolution

An address-lookup-table account has length `56 + 32*N`, where `0 <= N <= 256` for every
table reachable through program execution. Address `i` is the 32 raw bytes at
`[56+32*i, 88+32*i)`. Its first 56 bytes are:

**Normative table.**

| Byte range | Field | Accepted encoding |
|---|---|---|
| `[0,4)` | state discriminant | `LE(1,4)` for initialized LookupTable; `LE(0,4)` is uninitialized and unusable |
| `[4,12)` | `deactivation_slot` | `u64` LE; `u64::MAX` means not deactivating |
| `[12,20)` | `last_extended_slot` | `u64` LE |
| `[20,21)` | `last_extended_slot_start_index` | `u8` |
| `[21,22)` | authority option | `0` for none or `1` for some |
| `[22,54)` | authority | present only for option `1`; 32 raw bytes |
| following 2 bytes | padding | canonically `LE(0,2)` |
| through byte 56 | reserved | canonically zero-filled; for option `0`, bytes `[24,56)` are reserved |

The bytes after the conditional metadata decoder through offset 56 are not addresses. A
nonmultiple-of-32 address region or an unknown state/option discriminant makes the table
invalid. The bound `N <= 256` is an invariant maintained by the Address Lookup Table
program (Section 13.3.7: Create allocates 56 bytes; Extend rejects a total above 256).
Address loading does not check it: an initialized table with `N > 256` deserializes, and
indices `0..=255` resolve normally. Such a table is unreachable through program
execution. Canonical program writes zero the padding and reserved region; resolution does
not interpret those bytes as fields or addresses.

[SOL-TXN-0010] Each version-0 lookup entry **MUST** contain at least one writable or
readonly table index.

[SOL-TXN-0011] A lookup-table account **MUST** exist, be owned by the
address-lookup-table program, deserialize as a valid lookup table, and be usable at the
bank slot under the bank's SlotHashes ancestry; otherwise address loading **MUST** fail
before fees are charged.

[SOL-TXN-0012] Every selected table index **MUST** address an entry visible at the bank
slot, and addresses **MUST** be appended first from all writable selections and then
from all readonly selections while retaining lookup-entry and within-entry index order.

If `deactivation_slot == u64::MAX`, the table is active. Otherwise it remains usable
only at the deactivation slot or while that slot occurs in the current SlotHashes
history. When `bank_slot > last_extended_slot`, all `N` addresses are visible; otherwise
only indices below `last_extended_slot_start_index` are visible.

## 7.5 Signatures, age, and replay checks

A durable-nonce account is system-owned and has exactly 80 data bytes. Its current
initialized form is:

**Normative table.**

| Byte range | Field | Exact encoding |
|---|---|---|
| `[0,4)` | nonce version | `LE(1,4)` for Current; `LE(0,4)` is Legacy and cannot authorize a durable transaction |
| `[4,8)` | state | `LE(1,4)` for Initialized; `LE(0,4)` is Uninitialized |
| `[8,40)` | authority | 32 raw public-key bytes |
| `[40,72)` | durable nonce | 32 raw hash bytes |
| `[72,80)` | fee rate | `lamports_per_signature:u64` LE |

For an uninitialized state, the remaining account-data bytes are zero padding. Given the
bank's latest registered blockhash `H`, its next durable nonce is
`SHA256("DURABLE_NONCE" || H)`, where the domain is the 13 ASCII bytes. Instruction zero
marks a durable transaction only when its program ID is the System program, its data
begins `04 00 00 00`, and its first account index selects the nonce account; bytes after
that four-byte discriminant do not affect marker recognition.

[SOL-TXN-0013] For signature index `i`, the runtime **MUST** verify the 64-byte Ed25519
signature against the exact serialized message bytes using static key `S[i]`, and any
failed required signature **MUST** make the transaction unprocessable.

The transaction message hash used by replay protection is specified in
Chapter 14.

The recent-blockhash queue retains at most 300 registered blockhashes. Its newest value
has age zero, the next-newest age one, and so on. Ordinary transaction processing
accepts an age through 150 inclusive; older retained values remain available for
non-admission lookups but do not satisfy transaction age.

[SOL-TXN-0014] Before charging a fee, the runtime **MUST** require either that
`recent_blockhash` is present in the bank's recent-blockhash queue with age at most
`150`, or that instruction zero and the referenced nonce account form a valid
durable-nonce transaction for that bank.

[SOL-TXN-0015] A durable-nonce transaction **MUST** identify the nonce account through
the instruction-zero marker above, find an exactly 80-byte system-owned
Current/Initialized nonce state whose stored durable nonce equals the transaction's
`recent_blockhash`, require the stored authority among instruction zero's signers, and
require the bank's next durable nonce to differ from the stored value.

[SOL-TXN-0016] The runtime **MUST** reject a transaction already present in the
applicable replay-protection relation for the same message hash and recent blockhash
before account execution.

## 7.6 Compute-budget instructions and fees

The compute-budget program recognizes one setting of each of these parameters: requested
heap bytes, compute-unit limit, compute-unit price in micro-lamports per compute unit,
and loaded-account-data byte limit. Recognition occurs while scanning top-level
instructions before account loading.

Compute-budget instruction data is the following little-endian enum. The decoder reads
the discriminant and payload shown; trailing bytes are ignored. Discriminant `0` is
reserved and is rejected.

**Normative table.**

| Discriminant | Payload bytes | Meaning |
|---:|---|---|
| `1` | `[1,5)` `heap_frame_bytes:u32` | Request heap frame |
| `2` | `[1,5)` `compute_unit_limit:u32` | Set compute-unit limit |
| `3` | `[1,9)` `micro_lamports_per_compute_unit:u64` | Set compute-unit price |
| `4` | `[1,5)` `loaded_account_data_bytes:u32` | Set loaded-account-data limit |

[SOL-TXN-0017] A duplicate recognized compute-budget setting or a malformed recognized
compute-budget instruction **MUST** make the transaction unprocessable.

[SOL-TXN-0018] An explicit heap request **MUST** be a multiple of 1,024 in the inclusive
range 32,768 through 262,144 bytes; the default is 32,768 bytes.

**Normative table. Default requested compute units per top-level instruction.**

| Program classification using the authoritative identifiers in Section 13.2 | Default requested units |
|---|---:|
| System Program | `3,000` CU |
| Compute Budget Program | `3,000` CU |
| Upgradeable BPF Loader | `3,000` CU |
| deprecated BPF Loader | `3,000` CU |
| BPF Loader | `3,000` CU |
| reserved Loader-v4 address | `3,000` CU |
| secp256k1 precompile | `3,000` CU |
| Ed25519 precompile | `3,000` CU |
| Vote Program | `200,000` CU; its SIMD-0387 removal from builtin default-cost classification is active |
| secp256r1 precompile and every other program identifier, including all Core-BPF programs | `200,000` CU |

This table classifies only the default transaction request when no explicit limit
exists. It does not replace the actual native instruction costs or SBF metering in
Section 12.8.8.

[SOL-TXN-0019] An explicit compute-unit limit **MUST** be capped at `1,400,000`. Without
one, assign `3,000` units to each top-level instruction whose program is System, Compute
Budget, Upgradeable BPF Loader, deprecated BPF Loader, BPF Loader, the reserved
Loader-v4 address, secp256k1, or Ed25519; assign `200,000` units to every other
top-level instruction, including Vote; add with saturating `u32` arithmetic and cap the
result at `1,400,000`.

[SOL-TXN-0020] The loaded-account-data byte limit **MUST** be nonzero and capped at
67,108,864 bytes; an absent request selects 67,108,864 bytes.

[SOL-TXN-0021] The prioritization fee in lamports **MUST** be
`ceil(compute_unit_price_micro_lamports * compute_unit_limit / 1,000,000)`, calculated
in a width that cannot overflow the product and saturated to `u64::MAX`.

[SOL-TXN-0022] The signature fee **MUST** count the transaction's required signatures
plus the signature operations declared by each active signature-verification precompile,
and **MUST** multiply that count by the lamports-per-signature value associated with the
accepted recent blockhash, or by the nonce account's stored fee rate for a durable-nonce
transaction, using saturating `u64` arithmetic.

[SOL-TXN-0023] The charged transaction fee **MUST** be the saturating sum of the
signature fee and prioritization fee; its collection and later distribution **MUST**
follow Chapters 11 and 15.

## 7.7 Fee payer and account loading

[SOL-TXN-0024] The fee payer **MUST** be the first static account key, **MUST** be
writable and signed, and **MUST** hold either a system account or an initialized system
nonce account.

[SOL-TXN-0025] Fee validation **MUST** reject a payer that cannot pay the full fee while
retaining the rent-exempt reserve required by its post-fee account state, and it
**MUST** reject a post-fee rent-paying state reached from a rent-exempt pre-fee state.

[SOL-TXN-0026] After fee validation succeeds, the runtime **MUST** deduct the fee before
program execution; an instruction failure **MUST NOT** refund that deduction.

[SOL-TXN-0027] Account loading **MUST** load resolved transaction accounts from the bank
state, synthesize an absent ordinary account as a zero-lamport default account, and
synthesize the instructions sysvar rather than read a persistent instructions-sysvar
account.

[SOL-TXN-0028] The loaded-data meter **MUST** apply the byte-exact algorithm below;
exceeding the transaction's loaded-account-data limit **MUST** stop loading with
`TransactionError::MaxLoadedAccountsDataSizeExceeded`.

[SOL-TXN-0029] A top-level program account **MUST** exist and be owned by an active
native or SBF loader accepted by the protocol configuration. The account's legacy
`executable` Boolean is not an invocation gate in epoch `E` and **MUST NOT** by itself
cause `AccountNotExecutable` or `InvalidProgramForExecution`. A missing program account
is `TransactionError::ProgramAccountNotFound`; an unsupported owner or invalid loader
state is `TransactionError::InvalidProgramForExecution` during loading, or
`TransactionError::InstructionError(i, InstructionError::UnsupportedProgramId)` when
program resolution first occurs for instruction `i`.

[SOL-TXN-0030] If failure occurs only after fee-payer deduction, including a
post-deduction account-loading failure, the transaction **MUST** produce the fee-only
rollback state defined by Chapter 11 and **MUST NOT** expose partial loaded-account
changes.

[SOL-TXN-0031] A legacy or version-0 message **MUST NOT** contain more than 64 top-level
instructions or more than 255 account indices in one compiled instruction.

### 7.7.1 Loaded-account-data meter

Let `limit` be the sanitized nonzero requested limit from Section 7.6 and `total=0u32`.
Additions are checked after each item. If an item length cannot convert to `u32`, or if
`total + item` overflows `u32`, set the observable meter to `u32::MAX` and fail.
Otherwise set `total=total+item` and fail when `total > limit`; equality succeeds.

**Normative table. Loaded-account-data byte charges and order.**

| Phase/order | Item | Bytes added |
|---:|---|---:|
| 1 | Each address-table descriptor in message lookup order | `8,248` bytes, once per descriptor, including repeated table addresses |
| 2 | Existing fee-payer account | `64 + data_length` bytes |
| 3 | Remaining resolved keys in `K` order | for each existing nonzero-lamport account, `64 + data_length` bytes |
| immediately after its Program key | Separately loaded upgradeable-loader-v3 ProgramData | `64 + programdata_data_length` bytes when it exists, is not itself in `K`, and has not already been added through another Program |
| any key | Missing or zero-lamport account synthesized as default | `0` bytes |
| instructions sysvar | Synthesized account | `0` bytes |
| builtin/native program account named in `K` | Ordinary existing account | `64 + data_length` bytes; no additional cache or executable charge |
| loaded ALT address | Target account | no address-only charge; its target account is charged by the ordinary `K` rule |
| any synthesized sysvar other than instructions | Ordinary account if present in `K` | `64 + data_length` bytes; synthesis itself has no extra charge |

The `64` bytes are account-loading overhead and are not part of account data. The set
used to deduplicate extra ProgramData addresses is initialized with all of `K`, so an
explicitly named ProgramData account is charged exactly once at its position in `K`.
ProgramData deduplication is across every referencing Program. Failure to find the
referenced ProgramData does not add bytes; the later program-state/owner check returns
the applicable program-loading error.

[SOL-TXN-0032] The meter **MUST** add lookup overhead before the fee payer, then load
the fee payer, then remaining keys in `K` order, adding an eligible extra ProgramData
account immediately after the Program that refers to it. It **MUST** check the limit
after every addition and **MUST NOT** count account data twice just because the key is
referenced by multiple instructions.

[SOL-TXN-0033] A meter failure after successful fee deduction **MUST** be a fee-only
committed transaction. It charges the complete fee; commits durable nonce advancement
when the transaction used one; records the transaction identity in replay protection;
executes no instruction; consumes `0` execution compute units; records loaded bytes as
`min(total,limit)` when `total` is representable and `limit` after overflow; and is a
transaction-level result. During block replay it is not by itself block-fatal.

## 7.8 Error disposition summary

The following four outcome classes are authoritative wherever another section names an
exact error but does not repeat every observable effect.

**Normative table. Error outcome classes.**

| Class | Fee charged | Durable nonce | Replay protection | Execution CU | Production disposition | Replay disposition |
|---|---|---|---|---|---|---|
| `P0` pre-fee rejection | no | unchanged | unchanged | `0` CU | omit the transaction | `BF`: a block containing it is invalid |
| `P1` fee-only load failure | full fee | advancement commits | identity recorded | `0` execution CU | record the failed transaction | committed transaction-level failure |
| `P2` executed failure | full fee | advancement commits | identity recorded | actual CU through failure; full requested CU for a depleting SBF VM failure under Section 12.8.9 | record the failed transaction | committed transaction-level failure |
| `BF` block-fatal | no additional debit beyond effects of earlier transactions already replayed | only effects of earlier committed transactions remain | candidate is not committed | no block result | do not produce the candidate | invalidate the candidate |

For `P1` and `P2`, fee-payer and durable-nonce rollback accounts are committed as
Section 11.4 specifies. Logs, return-data snapshots, diagnostic strings, and persistent
transaction metadata are not consensus state. They may be retained or discarded after
the execution path has used them.

**Normative table. Complete `TransactionError` domain and default outcome.**

| Exact variant | First-producing phase / precedence | Outcome |
|---|---|---|
| `AccountLoadedTwice` | phase 4, after `TooManyAccountLocks` | `P0` admission; `BF` replay |
| `AccountNotFound` | phase 8 fee debit or phase 9 required debit account | `P0` admission and `BF` replay before deduction; `P1` after deduction |
| `ProgramAccountNotFound` | phase 9 first missing invoked program | `P1` |
| `InsufficientFundsForFee` | phase 8 | `P0` admission; `BF` replay |
| `InvalidAccountForFee` | phase 8 | `P0` admission; `BF` replay |
| `AlreadyProcessed` | phase 7 | `P0` admission; `BF` replay |
| `BlockhashNotFound` | phase 6 | `P0` admission; `BF` replay |
| `InstructionError(i,e)` | phase 10 or post-instruction validation; exact `e` is Section 12.7 | `P2` |
| `InvalidAccountIndex` | phase 1 static message check | `P0` admission; `BF` replay |
| `SignatureFailure` | phase 2 | `P0` admission; `BF` replay |
| `InvalidProgramForExecution` | phase 9 owner/state check | `P1` |
| `SanitizeFailure` | phase 1 | `P0` admission; `BF` replay |
| `WouldExceedMaxBlockCostLimit` | producer reservation or replay cost check | `P0` in production; `BF` in replay |
| `UnsupportedVersion` | phase 1 | `P0` admission; `BF` replay |
| `WouldExceedMaxAccountCostLimit` | producer reservation or replay cost check | `P0` in production; `BF` in replay |
| `WouldExceedAccountDataBlockLimit` | producer reservation or replay allocation check | `P0` in production; `BF` in replay |
| `TooManyAccountLocks` | phase 4 before duplicate scan | `P0` admission; `BF` replay |
| `AddressLookupTableNotFound` | phase 3 | `P0` admission; `BF` replay |
| `InvalidAddressLookupTableOwner` | phase 3 | `P0` admission; `BF` replay |
| `InvalidAddressLookupTableData` | phase 3 | `P0` admission; `BF` replay |
| `InvalidAddressLookupTableIndex` | phase 3 | `P0` admission; `BF` replay |
| `InvalidRentPayingAccount` | phase 8 or phase 11 | `P0` admission and `BF` replay at fee validation; `P2` after execution |
| `DuplicateInstruction(i)` | phase 5 first repeated singleton setting at index `i` | `P0` admission; `BF` replay |
| `InsufficientFundsForRent{account_index}` | phase 11, lowest failing account index | `P2` |
| `MaxLoadedAccountsDataSizeExceeded` | phase 9 meter order | `P1` |
| `InvalidLoadedAccountsDataSizeLimit` | phase 5 | `P0` admission; `BF` replay |
| `UnbalancedTransaction` | phase 11 aggregate lamport check | `P2` |

[SOL-TXN-0034] A failure **MUST** expose the exact `TransactionError` variant and
outcome in the table above. When multiple conditions are false, the phase order in
Section 11.3 and the within-phase order in this table select one error; an
implementation **MUST NOT** choose a later or more convenient error.

Account contention, maintenance mode, a stale locally prepared transaction,
executable-preparation capacity, and scheduling cancellation are local admission or
retry conditions. They are not protocol transaction results, consume no protocol
resources, and cannot be committed as the status of a transaction in an accepted
block. Serializable `TransactionError` compatibility variants such as `AccountInUse`,
`CallChainTooDeep`, `MissingSignatureForFee`, `ClusterMaintenance`,
`AccountBorrowOutstanding`, `InvalidWritableAccount`,
`WouldExceedAccountDataTotalLimit`, `ResanitizationNeeded`,
`ProgramExecutionTemporarilyRestricted`, `ProgramCacheHitMaxLimit`, and
`CommitCancelled` therefore do not belong to the epoch-`E` protocol error domain.

The signature-verification precompile byte formats and counting rules are specified in
Section 12.6.
