# 12. SVM, loaders, syscalls, and precompiles

## 12.1 Execution boundary

This chapter fixes the consensus-visible execution environment supplied to native
programs and Solana BPF (SBF) programs. Section 12.8 defines the entire active SBF
instruction set and the Solana-specific loader, memory, syscall, input, compute, and
state rules; no external ISA document is normative.

[SOL-SVM-0001] For every executable program and input state accepted by this
specification, an SVM implementation **MUST** produce the same instruction result,
compute-unit consumption, account changes, return data, and consensus-visible error.

[SOL-SVM-0002] The runtime **MUST** select the accepted SBF versions, VM configuration,
loader behavior, syscall registry, precompile set, compute schedule, stack-depth rule,
and feature branches from the bank in which the instruction executes.

## 12.2 Program resolution and loading

A program account is resolved through the transaction key vector. A native builtin is
owned by the Native Loader. An SBF program is owned by a supported BPF loader. Under the
upgradeable loader, executable Program state refers to a distinct ProgramData account
containing deployment metadata and program bytes.

Program selection is the pure protocol relation:

**Normative algorithm.**

```text
ResolveProgram(bank_state, program_id, invoking_slot)
    -> Builtin(program_id, behavior)
     | Sbf(loader_id, program_bytes, deployment_slot,
           ProgramEnvironment(execution, loader_id, deployment_slot,
                              invoking_slot, feature_activation_history))
     | ProgramAccountNotFound
     | InvalidProgramForExecution
     | UnsupportedProgramId
```

`ProgramEnvironment(mode, loader_id, deployment_slot, invoking_slot,
feature_activation_history)` is the pure selection of the accepted SBF versions,
verifier rules, memory configuration, syscall set, and compute schedule. Mode is
`deployment` while validating new bytes and `execution` while invoking stored bytes;
the mode-specific and historically retained syscall differences are stated in Sections
12.8.5 and 12.8.7. No compiled artifact, cache generation, eviction event, or host
execution engine is an input to either relation.

[SOL-SVM-0003] Before invocation, the runtime **MUST** require the program identifier to
be present in the transaction's instruction context and **MUST** reject a missing,
unsupported-owner, invalid-state, or unloadable program without entering its code. The
stored `executable` Boolean is passed through the ABI but is not an epoch-`E` invocation
predicate.

[SOL-SVM-0004] A native-loader-owned program **MUST** dispatch to the builtin registered
for that exact program identifier, while a supported BPF-loader-owned program **MUST**
dispatch through its loader and the bank-selected SBF environment.

[SOL-SVM-0005] `ResolveProgram` **MUST** depend only on the invoking bank's account
state, program identifier, invoking slot, and feature activation history, and **MUST**
return the same selected bytes, loader, environment, and error for equal inputs.

[SOL-SVM-0006] Loading an upgradeable program **MUST** validate the
Program-to-ProgramData relationship and use the ProgramData version visible in bank
state; it **MUST NOT** substitute a later deployment or a host-local artifact. A
deployment, upgrade, or Core-BPF migration at slot `d` **MUST NOT** make the new bytes
invocable until slot `d+1`, and **MUST NOT** use the old bytes during that delay.

A successful deploy or upgrade recorded at slot `d` sets deployment slot `d` and
effective slot `d+1`. During the remainder of `d`, resolution of that program does not
fall back to the previous bytes: an attempted invocation fails with
`InstructionError::UnsupportedProgramId`. At slots at or above `d+1`, resolution uses
the new bytes. A checked addition overflow for `d+1` makes deployment invalid.

## 12.3 Instruction execution environment

[SOL-SVM-0007] The runtime **MUST** execute top-level instructions in message order,
initialize each instruction from transaction account privileges and data, and stop the
transaction at the first failed instruction.

[SOL-SVM-0008] Each invocation **MUST** begin with return data reset to empty for the
invoked program and **MUST** expose only return data produced during the current
invocation chain.

[SOL-SVM-0009] The compute meter **MUST** begin at the transaction's sanitized
compute-unit limit, charge native-program work, VM instructions, heap provisioning,
memory operations, syscalls, CPI, and loader work according to the active schedule, and
fail when a required charge exceeds the remaining units. Epoch-`E` precompile
verification consumes zero execution-meter units; its declared signature count is
charged through transaction fees and the Section 10.4.1 block-admission cost.

[SOL-SVM-0010] VM memory translation **MUST** enforce the address ranges, permissions,
alignment, stack layout, heap size, and input mapping in Section 12.8. An invalid guest
access **MUST** produce the specified VM or instruction error without host undefined
behavior.

[SOL-SVM-0011] Program return status, VM fault, compute exhaustion, and loader/runtime
failure **MUST** map to the protocol instruction-error domain before Chapter 11 applies
transaction rollback.

## 12.4 Invocation stack and CPI

An instruction context contains ordered instruction accounts, program accounts,
instruction data, stack height, and its position in the transaction instruction trace.
CPI uses accounts already present in the transaction context; it does not load an
arbitrary account by public key.

[SOL-SVM-0012] Invocation push **MUST** fail when it would exceed the maximum
instruction-stack depth of 5.

[SOL-SVM-0013] Invocation push **MUST** reject reentrancy when the target program
already appears earlier in the invocation stack, except that a program **MAY** directly
invoke itself when it is also the immediate caller.

[SOL-SVM-0014] A CPI instruction **MUST** resolve every account meta and the callee
program to accounts already present in the caller's transaction instruction context, and
a missing account or callee **MUST** fail the caller instruction.

[SOL-SVM-0015] CPI **MUST NOT** elevate a readonly caller account to writable and **MUST
NOT** elevate a nonsigner caller account to signer unless the caller program derives
that account as a valid program-derived address from supplied signer seeds.

[SOL-SVM-0016] Signer seeds **MUST** be interpreted under the caller program identifier,
not the callee identifier, and invalid seed count, seed length, or derived-address
construction **MUST** fail CPI.

[SOL-SVM-0017] A CPI **MUST** reject more than 255 account metas, more than 255
account-info records, more than 16 signer-seed groups, or instruction data longer than
10,240 bytes. The complete instruction trace, including top-level instructions, **MUST
NOT** exceed 64 entries.

[SOL-SVM-0018] CPI **MUST NOT** directly target the Native Loader, a BPF loader, an
active signature precompile, or an upgradeable-loader management instruction outside the
active allowlist.

[SOL-SVM-0019] Before entering a callee and after returning to its caller, the runtime
**MUST** synchronize every shared account's lamports, data, owner, and permitted length
changes without losing aliases or duplicate-account privileges.

[SOL-SVM-0020] Instruction pop **MUST** fail if an account remains borrowed or if
aggregate lamports differ from the aggregate on instruction entry, and the resulting
error **MUST** trigger transaction rollback.

## 12.5 Syscall dispatch

[SOL-SVM-0021] A syscall **MUST** resolve only when registered in the runtime
environment for the bank and program deployment; an unresolved symbol or static target
**MUST** cause ELF verification or execution failure.

[SOL-SVM-0022] Each syscall **MUST** translate and bounds-check guest pointers and
lengths, enforce required alignment and non-overlap rules, validate selectors and
numeric bounds, and charge its specified compute cost before producing an output or
error.

[SOL-SVM-0023] A sysvar syscall **MUST** return the bank-visible value for the current
invocation and slot and **MUST NOT** read host time or stale process-global state.

[SOL-SVM-0024] Hashing and elliptic-curve syscalls **MUST** operate on the exact guest
byte slices and curve/field encodings selected by the syscall's identifiers, with
deterministic rejection of invalid encodings.

[SOL-SVM-0025] `sol_memcpy_` **MUST** reject overlapping source and destination ranges;
`sol_memmove_` **MUST** permit overlap; `sol_memset_` **MUST** write the low byte of its
fill argument; and `sol_memcmp_` **MUST** write the signed comparison result to its
result pointer.

## 12.6 Signature precompiles

Precompiles are processed in top-level message order during instruction execution and
can reference the complete ordered vector of top-level instruction data. They do not run
SBF code and cannot be invoked by CPI. A precompile failure is an executed instruction
error: the transaction fee remains charged and Chapter 11 rolls back program-produced
state.

### 12.6.1 Ed25519 and secp256r1 record layout

Ed25519 and secp256r1 use a two-byte prefix followed by `count` consecutive 14-byte
records. Every integer in a record is an unaligned little-endian `u16`.

**Normative table.**

| Byte range | Field |
|---|---|
| `[0,1)` | signature count (`u8`) |
| `[1,2)` | ignored byte |
| `[2, 2 + 14*count)` | offset records |
| remaining bytes | optional referenced signature, key, and message data |

**Normative table.**

| Record byte range | Field |
|---|---|
| `[0,2)` | signature offset |
| `[2,4)` | signature instruction index |
| `[4,6)` | public-key offset |
| `[6,8)` | public-key instruction index |
| `[8,10)` | message-data offset |
| `[10,12)` | message-data length |
| `[12,14)` | message instruction index |

For either precompile, instruction index `0xffff` selects the current precompile
instruction data; any other value selects that zero-based top-level instruction's data.

**Normative table.**

| Program | Signature bytes | Public key | Message operation | Count rule |
|---|---|---|---|---|
| Ed25519 | 64-byte Ed25519 signature | 32-byte compressed Edwards key | strict Ed25519 verification over referenced message bytes | count `0` is valid only when total data length is exactly 2 |
| secp256r1 | 64 bytes `r \|\| s`, each a 32-byte big-endian integer | 33-byte SEC1 compressed P-256 key | ECDSA with SHA-256; require `1 <= r < n` and `1 <= s <= n/2` | `1..=8` |

### 12.6.2 secp256k1 record layout

The secp256k1 instruction begins with a one-byte count and `count` consecutive 11-byte
records. The `u16` fields are little-endian.

**Normative table.**

| Byte range | Field |
|---|---|
| `[0,1)` | signature count (`u8`) |
| `[1, 1 + 11*count)` | offset records |
| remaining bytes | optional referenced signature, address, and message data |

**Normative table.**

| Record byte range | Field |
|---|---|
| `[0,2)` | signature offset (`u16`) |
| `[2,3)` | signature instruction index (`u8`) |
| `[3,5)` | Ethereum-address offset (`u16`) |
| `[5,6)` | Ethereum-address instruction index (`u8`) |
| `[6,8)` | message-data offset (`u16`) |
| `[8,10)` | message-data length (`u16`) |
| `[10,11)` | message instruction index (`u8`) |

The referenced secp256k1 signature value has this layout:

**Normative table.**

| Relative byte range | Field |
|---|---|
| `[0,64)` | standard compact secp256k1 signature |
| `[64,65)` | recovery identifier |

Verification computes Keccak-256 over the referenced message, recovers the public key,
serializes its 64-byte uncompressed coordinates without the SEC1 prefix, computes
Keccak-256 again, and compares the last 20 bytes with the referenced Ethereum address. A
zero count is valid only when the instruction contains exactly the count byte.

[SOL-SVM-0026] A precompile verifier **MUST** parse the applicable prefix and record
table above, require the complete table to be present, bounds-check every referenced
range with saturating or checked arithmetic, and apply the program-specific verification
rule.

[SOL-SVM-0027] A malformed table, out-of-range instruction reference, invalid key or
signature encoding, failed signature, or invalid count/data combination **MUST** fail
that top-level instruction with the corresponding precompile instruction error, stop
later instructions, and enter the failed-execution rollback path.

[SOL-SVM-0028] Precompile signature counts used for fee and block-admission cost
calculation **MUST** be read from the same active program identifier and instruction
bytes used by verification. Precompile verification itself **MUST NOT** debit the
epoch-`E` transaction execution meter.

[SOL-SVM-0029] A precompile **MUST** observe only the ordered top-level instruction-data
vector; CPI-created instruction data **MUST NOT** be substituted into that vector.

## 12.7 Protocol-visible execution errors

Every instruction failure is returned as `TransactionError::InstructionError(i,e)`,
where `i` is the zero-based top-level message-instruction index even when the failure
arose in CPI and `e` is one exact value from the table below. It is outcome `P2`: the
full fee and any durable-nonce advance commit, replay protection records the message,
account writes roll back, consumed CU remains charged, and the failure is
transaction-level rather than block-fatal. Logs and the final return-data buffer are not
committed protocol outputs.

**Normative table. Complete `InstructionError` domain and first applicable cause.**

| Exact variant | First applicable cause |
|---|---|
| `GenericError` | legacy builtin generic failure |
| `InvalidArgument` | well-formed instruction with invalid value, Boolean, selector, duplicate config key, or incompatible operands |
| `InvalidInstructionData` | instruction bytes cannot decode, have a forbidden discriminant/length, or contain a failed ZK proof |
| `InvalidAccountData` | required account state cannot decode or violates its state invariant |
| `AccountDataTooSmall` | destination account lacks required fixed capacity |
| `InsufficientFunds` | program transition cannot debit requested lamports |
| `IncorrectProgramId` | instruction expected a different program address |
| `MissingRequiredSignature` | first required authority is not an instruction signer |
| `AccountAlreadyInitialized` | initialize targets initialized state |
| `UninitializedAccount` | non-initialize transition targets uninitialized state |
| `UnbalancedInstruction` | invocation-pop lamport sum differs from invocation-push sum |
| `ModifiedProgramId` | post-execution owner changed without owner-assignment authority |
| `ExternalAccountLamportSpend` | program debited an account it does not own |
| `ExternalAccountDataModified` | program changed data it does not own |
| `ReadonlyLamportChange` | first readonly account whose lamports changed |
| `ReadonlyDataModified` | first readonly account whose data changed |
| `DuplicateAccountIndex` | retired compatibility value; not newly produced in epoch `E` |
| `ExecutableModified` | program changed the executable flag |
| `RentEpochModified` | program changed rent epoch |
| `NotEnoughAccountKeys` | a protocol/application program requests more ordered instruction accounts than were supplied |
| `AccountDataSizeChanged` | non-owner changed an account's data length |
| `AccountNotExecutable` | retired executable-bit check; not newly produced under SIMD-0162 behavior |
| `AccountBorrowFailed` | requested borrow conflicts with a live borrow |
| `AccountBorrowOutstanding` | invocation returns while an account borrow remains |
| `DuplicateAccountOutOfSync` | aliased ABI records produced inconsistent mutations |
| `Custom(v)` | program-defined `u32` value `v`; program tables below define protocol-owned values |
| `InvalidError` | nonzero SBF return is neither a defined builtin code nor a `u32` custom value |
| `ExecutableDataModified` | executable account data changed illegally |
| `ExecutableLamportChange` | executable account lamports changed illegally |
| `ExecutableAccountNotRentExempt` | executable account ends below rent exemption |
| `UnsupportedProgramId` | direct invocation/CPI target is forbidden or no longer supported |
| `CallDepth` | invocation stack would exceed 5 |
| `MissingAccount` | first required instruction/CPI account is absent |
| `ReentrancyNotAllowed` | non-direct recursion would re-enter a program already on stack |
| `MaxSeedLengthExceeded` | more than 16 seeds or a seed longer than 32 bytes |
| `InvalidSeeds` | PDA derivation is invalid or does not match the supplied signer |
| `InvalidRealloc` | growth, shrink, or ABI realloc region violates Section 4.2/12.8.6 |
| `ComputationalBudgetExceeded` | next charge exceeds remaining CU |
| `PrivilegeEscalation` | CPI requests signer or writable privilege not held by caller |
| `ProgramEnvironmentSetupFailure` | bank-selected VM environment cannot be constructed |
| `ProgramFailedToComplete` | VM fault, abort, panic, invalid pointer, or unresolved execution path |
| `Immutable` | transition requires an authority but state is immutable |
| `IncorrectAuthority` | supplied signed authority differs from stored authority |
| `BorshIoError` | protocol program's required Borsh decode/encode fails |
| `AccountNotRentExempt` | transition requires rent exemption and balance is too low |
| `InvalidAccountOwner` | account owner differs from the exact required owner |
| `ArithmeticOverflow` | transition requires checked arithmetic and it overflows |
| `UnsupportedSysvar` | requested sysvar is outside the active syscall/cache set |
| `IllegalOwner` | requested new owner is forbidden |
| `MaxAccountsDataAllocationsExceeded` | positive transaction account-data growth exceeds `20,971,520` bytes |
| `MaxAccountsExceeded` | runtime account collection exceeds its protocol bound |
| `MaxInstructionTraceLengthExceeded` | push would exceed 64 trace entries |
| `BuiltinProgramsMustConsumeComputeUnits` | a native dispatch returns without its mandatory charge |

For an SBF `EXIT` at outer depth, `r0 == 0` means success. A value `v` with upper 32
bits zero maps to `Custom(u32(v))`, except zero is success; `Custom(0)` is encoded by
`1<<32`. Builtin return codes `(n<<32)` map as follows.

**Normative table. SBF builtin return-code mapping.**

| `n` | `InstructionError` | `n` | `InstructionError` |
|---:|---|---:|---|
| 1 | `Custom(0)` | 14 | `InvalidSeeds` |
| 2 | `InvalidArgument` | 15 | `BorshIoError` |
| 3 | `InvalidInstructionData` | 16 | `AccountNotRentExempt` |
| 4 | `InvalidAccountData` | 17 | `UnsupportedSysvar` |
| 5 | `AccountDataTooSmall` | 18 | `IllegalOwner` |
| 6 | `InsufficientFunds` | 19 | `MaxAccountsDataAllocationsExceeded` |
| 7 | `IncorrectProgramId` | 20 | `InvalidRealloc` |
| 8 | `MissingRequiredSignature` | 21 | `MaxInstructionTraceLengthExceeded` |
| 9 | `AccountAlreadyInitialized` | 22 | `BuiltinProgramsMustConsumeComputeUnits` |
| 10 | `UninitializedAccount` | 23 | `InvalidAccountOwner` |
| 11 | `NotEnoughAccountKeys` | 24 | `ArithmeticOverflow` |
| 12 | `AccountBorrowFailed` | 25 | `Immutable` |
| 13 | `MaxSeedLengthExceeded` | 26 | `IncorrectAuthority` |

Any other value with nonzero upper 32 bits maps to `InvalidError`. Runtime
postconditions are checked in transaction-account order; the first failing account and
the first applicable row above determine `e`. Within a protocol program, instruction
decoding precedes account acquisition, which precedes owner/state validation,
signer/authority validation, numeric/rent validation, and mutation. Compute charged on
entry or before a cryptographic operation is not refunded when a later check fails.

[SOL-SVM-0030] An implementation **MUST** preserve the protocol distinction among
transaction sanitation/loading errors, instruction errors, custom program errors, VM
faults, compute exhaustion, privilege escalation, unbalanced lamports, and invalid
account-data or ownership transitions wherever that distinction affects replay or
transaction status.

[SOL-SVM-0031] Diagnostic strings, host exception types, executable-preparation
strategy, and performance metrics **MAY** differ, but they **MUST NOT** alter the error
code, consumed compute, committed state, fee, or replay outcome. An implementation's
inability to prepare an SBF program that passes the normative verifier is a local
failure, not `InstructionError::ProgramFailedToCompile`.

## 12.8 SBF abstract machine and runtime ABI

### 12.8.1 Complete active SBF instruction set

This subsection is the complete normative ISA for SBF V0 through V3 in epoch `E`. Names
from BPF literature are mnemonics only; the byte, arithmetic, and version rules below
are authoritative and an implementation does not need an external BPF specification.

[SOL-SVM-0032] An SBF engine **MUST** implement the SBF opcode, verifier, arithmetic,
call, and VM-fault behavior selected by the ELF's V0–V3 version and **MUST NOT** accept
an instruction rejected by that version's verifier.

Every ordinary instruction occupies eight bytes:

**Normative table. SBF instruction word.**

| Byte range | Field | Interpretation |
|---|---|---|
| `[0,1)` | opcode | instruction class, operation, width, and source mode |
| low nibble of byte 1 | destination register | `r0..r10` as permitted by the opcode |
| high nibble of byte 1 | source register | `r0..r10` as permitted by the opcode |
| `[2,4)` | offset | signed `i16`, little-endian |
| `[4,8)` | immediate | signed `i32`, little-endian |

For versions that permit `LDDW`, it occupies two consecutive eight-byte words: the first
word carries the low 32 immediate bits; the second word has opcode zero and carries the
high 32 bits. No jump may target the second word.

Let `D=r[dst]`, `S=r[src]`, `I=sign_extend_i32(imm)`, `U32(x)` be the low 32 bits
interpreted unsigned, `I32(x)` those bits interpreted signed, and
`EA=r[base]+sign_extend_i16(off)` modulo `2^64`. Loads zero-extend the little-endian
value read at `EA`; stores write the low 1, 2, 4, or 8 bytes. Every memory access is
checked under Section 12.8.4 before it changes a register or byte. In the tables, an
`I/R` opcode pair selects `I` with the first opcode and `S` with the second.

**Normative table. Load and store opcodes.**

| Versions | Mnemonic and width | Opcode(s) | Exact operation |
|---|---|---|---|
| V0, V1, V3 | `ldxb/ldxh/ldxw/ldxdw` | `71/69/61/79` | `r[dst] = load_u{8,16,32,64}(r[src]+off)` |
| V0, V1, V3 | `stb/sth/stw/stdw` immediate | `72/6a/62/7a` | `store_u{8,16,32,64}(r[dst]+off, I)` |
| V0, V1, V3 | `stxb/stxh/stxw/stxdw` register | `73/6b/63/7b` | `store_u{8,16,32,64}(r[dst]+off, S)` |
| V2 | `ldxb/ldxh/ldxw/ldxdw` | `2c/3c/8c/9c` | same load operation |
| V2 | `stb/sth/stw/stdw` immediate | `27/37/87/97` | same immediate-store operation |
| V2 | `stxb/stxh/stxw/stxdw` register | `2f/3f/8f/9f` | same register-store operation |
| V0, V1, V3 | `lddw` | `18`, then continuation opcode `00` | `r[dst]=LE(first.imm,4) + 2^32*LE(second.imm,4)`; consumes two words |
| V2 | `hor64` | `f7` | `r[dst] = D OR (U32(imm) << 32)` |

All arithmetic wraps at its stated width. The 32-bit result is zero-extended except that
in V0, V1, and V3 the results of `add32`, `sub32`, and `mul32` are sign-extended from
bit 31; V2 zero-extends those results. V2 defines immediate subtraction as `I-D`; every
other version and every register subtraction uses `D-X`. A register shift uses
`U32(S) mod 32` or `S mod 64`; an immediate shift must be in `0..31` or `0..63` and
otherwise fails verification.

**Normative table. ALU opcodes and semantics.**

| Width | Operation | Immediate/register opcode | Result before the extension rule above | Versions |
|---:|---|---|---|---|
| 32 | add | `04/0c` | `U32(D)+U32(X)` | V0–V3 |
| 32 | subtract | `14/1c` | `U32(D)-U32(X)`, except V2 immediate is `U32(I)-U32(D)` | V0–V3 |
| 32 | multiply | `24/2c` | low 32 bits of `U32(D)*U32(X)` | V0, V1, V3 |
| 32 | unsigned divide | `34/3c` | `U32(D)/U32(X)` | V0, V1, V3 |
| 32 | OR / AND / XOR | `44/4c`, `54/5c`, `a4/ac` | bitwise `D op X` over 32 bits | V0–V3 |
| 32 | left / logical-right / arithmetic-right shift | `64/6c`, `74/7c`, `c4/cc` | `U32(D)<<X`, `U32(D)>>X`, or `I32(D)>>X` | V0–V3 |
| 32 | negate | `84` | low 32 bits of `-I32(D)` | V0, V1, V3 |
| 32 | unsigned remainder | `94/9c` | `U32(D)%U32(X)` | V0, V1, V3 |
| 32 | move | `b4/bc` | `U32(X)`; V2 register form sign-extends `I32(S)` | V0–V3 |
| 32 | byte-order conversion | `d4/dc` | `LE` / `BE` conversion of low `imm` bits, `imm in {16,32,64}` | `LE`: V0,V1,V3; `BE`: V0–V3 |
| 64 | add / subtract | `07/0f`, `17/1f` | `D+X`, `D-X`, except V2 immediate is `I-D` | V0–V3 |
| 64 | multiply / unsigned divide | `27/2f`, `37/3f` | `D*X`, `D/X` | V0, V1, V3 |
| 64 | OR / AND / XOR | `47/4f`, `57/5f`, `a7/af` | bitwise `D op X` | V0–V3 |
| 64 | left / logical-right / arithmetic-right shift | `67/6f`, `77/7f`, `c7/cf` | `D<<X`, `D>>X`, or `I64(D)>>X` | V0–V3 |
| 64 | negate | `87` | two's-complement `-I64(D)` | V0, V1, V3 |
| 64 | unsigned remainder | `97/9f` | `D%X` | V0, V1, V3 |
| 64 | move | `b7/bf` | `X` | V0–V3 |

For an immediate-form row, `X=I`, except V2 unsigned 64-bit PQR immediates use
`U32(imm)`. An immediate zero divisor is rejected by the verifier. A zero register
divisor raises `DivideByZero`; signed `MIN / -1` and `MIN % -1` raise `DivideOverflow`.
No destination is written on either VM fault; externally the transaction records
`InstructionError::ProgramFailedToComplete` at the active instruction index and the
ordinary VM-failure depletion rule in Section 12.8.9 applies.

**Normative table. V2-only product, quotient, and remainder opcodes.**

| Operation | 32-bit immediate/register | 64-bit immediate/register | Exact result |
|---|---|---|---|
| low multiply | `86/8e` | `96/9e` | low 32 or 64 bits of the unsigned product |
| unsigned high multiply | — | `36/3e` | bits `[64,128)` of `u128(D)*u128(X)` |
| signed high multiply | — | `b6/be` | bits `[64,128)` of `i128(I64(D))*i128(I64(X))` |
| unsigned quotient | `46/4e` | `56/5e` | unsigned `D/X` at the stated width |
| unsigned remainder | `66/6e` | `76/7e` | unsigned `D%X` at the stated width |
| signed quotient | `c6/ce` | `d6/de` | signed truncation-toward-zero `D/X` |
| signed remainder | `e6/ee` | `f6/fe` | signed remainder paired with truncation-toward-zero quotient |

For a conditional jump, the taken target is `pc+1+off`; the fall-through is `pc+1`. Both
are instruction-word indices. `JA` always takes its target. Unsigned and signed rows
interpret operands at the width in the table.

**Normative table. Control-flow opcodes.**

| Width | Predicate / operation | Immediate/register opcode | Versions |
|---:|---|---|---|
| 64 | unconditional `JA` | `05` | V0–V3 |
| 64 | `==`, unsigned `>`, unsigned `>=`, bit-set, `!=` | `15/1d`, `25/2d`, `35/3d`, `45/4d`, `55/5d` | V0–V3 |
| 64 | signed `>`, signed `>=` | `65/6d`, `75/7d` | V0–V3 |
| 64 | unsigned `<`, unsigned `<=` | `a5/ad`, `b5/bd` | V0–V3 |
| 64 | signed `<`, signed `<=` | `c5/cd`, `d5/dd` | V0–V3 |
| 32 | `==`, unsigned `>`, unsigned `>=`, bit-set, `!=` | `16/1e`, `26/2e`, `36/3e`, `46/4e`, `56/5e` | V3 |
| 32 | signed `>`, signed `>=` | `66/6e`, `76/7e` | V3 |
| 32 | unsigned `<`, unsigned `<=` | `a6/ae`, `b6/be` | V3 |
| 32 | signed `<`, signed `<=` | `c6/ce`, `d6/de` | V3 |
| — | immediate call | `85` | V0–V3 |
| — | register call (`CALLX`) | `8d` | V0–V3 |
| — | return/exit | `95` | V0–V3 |

For V0–V2 `CALL imm`, `U32(imm)` is the Murmur3-32 symbol key of either a
loader-registered syscall or an internal function. For V3, `src=0` selects a syscall by
the same key, while `src=1` selects the internal target `pc+1+sign_extend_i32(imm)`; any
other `src` is unresolved. `CALLX` takes its virtual target from register number `imm`
in V0/V1, `src` in V2, and `dst` in V3; only `r0..r9` are legal. A call pushes `r6..r9`,
the frame pointer, and the return PC; `EXIT` pops that frame or terminates the program
when depth is zero.

An unknown opcode, illegal version/opcode combination, invalid register, write to `r10`,
invalid branch/call target, branch into an `LDDW` continuation, malformed `LDDW`,
immediate zero divisor, invalid shift/endian immediate, or misaligned V1/V2
`add64 r10,imm` is rejected before execution. V1/V2 alone may write `r10` with
`add64 r10,imm`, and its immediate must be a multiple of 64.

**Normative table. SBF machine limits.**

| Limit / rule | Active value |
|---|---:|
| instruction size | 8 bytes |
| maximum text instructions | 65,536 |
| VM registers | twelve internal `u64` slots; program-visible `r0..r10` |
| local SBF call depth | 64 |
| stack frame | 4,096 bytes |
| instruction-meter checkpoint distance | 10,000 instructions |
| instruction metering | enabled; one compute unit per executed SBF instruction, plus syscall/runtime charges |

### 12.8.2 Registers, calls, and exit

SBF register names use a lowercase `r`: `r0` through `r11`. Indexed notation such as
`r[dst]` denotes the register selected by an instruction field; uppercase symbols such
as `D` and `S` denote values read from registers, not register names.

**Normative table.**

| Register | Meaning |
|---|---|
| `r0` | function return value and program exit status |
| `r1..r5` | call arguments; caller-saved |
| `r6..r9` | callee-saved |
| `r10` | frame pointer; writes are rejected except the version-1/2 dynamic-stack adjustment instruction |
| `r11` | internal program counter, not a program-visible general register |

At program entry, all registers are zero except `r1`, `r2`, `r10`, and the internal
`r11`. `r1` is `0x0000000400000000`, the start of serialized input. `r2` is the virtual
address of instruction data inside that input. `r10` is the version-dependent initial
stack pointer, and `r11` identifies the ELF entry instruction.

[SOL-SVM-0033] Internal calls **MUST** preserve `r6..r9`, enforce call depth 64, and
apply the selected version's static-frame or dynamic-frame rule. A program exit with
`r0 == 0` is success; any other `r0` **MUST** map through the canonical
`u64`-to-`InstructionError` conversion.

### 12.8.3 Active SBF versions

The ELF64 `e_flags` field selects the SBF version: `0`, `1`, `2`, or `3`.

**Normative table.**

| Version | Support in epoch `E` | Solana-specific behavior |
|---|---|---|
| V0 | deployment and execution enabled | fixed 4,096-byte frames with virtual gaps; legacy hashed calls/syscalls; `LDDW` enabled |
| V1 | deployment and execution enabled | dynamic stack frames; permits `add64 r10, imm`; legacy hashed calls/syscalls |
| V2 | deployment and execution enabled | V1 dynamic frames plus PQR arithmetic and instruction-encoding changes; `LDDW`, `NEG`, and `LE` disabled; memory classes moved; `CALLX` register encoded in `src` |
| V3 | deployment and execution enabled | fixed frames; static syscalls; strict ELF program-header format; separate low read-only region; JMP32/eBPF-ISA compatibility; `CALLX` register encoded in `dst` |

[SOL-SVM-0034] The runtime **MUST** select one version solely from `e_flags`, reject a
version outside V0–V3, and apply that version consistently to ELF parsing, verification,
calls, stack behavior, and execution.

### 12.8.4 Virtual address space

Virtual addresses use 32 low address bits within each 4-GiB region. Address translation,
aligned mapping, version-specific stack-frame gaps, and instruction metering apply.
Account data is part of the serialized-input region rather than a separate direct
mapping.

**Normative table.**

| Region | Start address | Access |
|---|---:|---|
| V3 read-only data | `0x0000000000000000` | read-only; region zero is permitted |
| bytecode and V0–V2 read-only image | `0x0000000100000000` | executable/read-only |
| stack | `0x0000000200000000` | read/write within active frames |
| heap | `0x0000000300000000` | read/write, transaction-selected size |
| serialized input | `0x0000000400000000` | metadata and readonly account fields read-only; permitted account fields/data writable as described below |

The default heap is 32,768 bytes. A compute-budget instruction may select a 1,024-byte
multiple from 32,768 through 262,144 bytes. Stack allocation is `64 * 4096 = 262144`
bytes before version-specific gaps/mapping.

[SOL-SVM-0035] Every load, store, syscall pointer, and indirect call **MUST** translate
entirely within a mapped region with the required access permission. Overflow,
cross-region access, write to read-only memory, and unmapped access **MUST** fault
deterministically.

### 12.8.5 ELF loading and verification

V0–V2 use the lenient parser and relocation path; V3 uses the strict parser. Common
acceptance requires a 64-bit little-endian ELF for machine `EM_BPF`, an entry aligned to
an eight-byte instruction, text length divisible by eight, a supported `e_flags`
version, a resolvable entrypoint, no conflicting function hashes, and bytecode accepted
by the requisite verifier.

The V3 strict image has this program-header contract:

**Normative table.**

| Property | Required value |
|---|---|
| ELF class / data / OS ABI | ELF64 / little-endian / System V (`0`) |
| ELF and header versions | current (`1`) |
| program-header offset | immediately after the ELF64 file header |
| program-header count | one executable header, optionally preceded by one read-only header |
| read-only header | `PT_LOAD`, flags `PF_R`, virtual and physical address `0` |
| bytecode header | `PT_LOAD`, flags `PF_X`, virtual and physical address `0x100000000` |
| segment file offset | contiguous after the program-header table/preceding segment and divisible by 8 |
| file and memory size | equal, in bounds, divisible by 8, and less than 4 GiB |
| entry | inside bytecode segment and divisible by 8 |

The verifier rejects empty or non-eight-byte-multiple text, unknown opcodes, invalid
registers, forbidden writes to `r10`, invalid immediate shift/endian/divisor values,
malformed `LDDW`, jumps outside text or into an `LDDW` continuation, invalid `CALLX`
registers, and immediate calls that do not resolve under the selected version.

[SOL-SVM-0036] Deployment and loading **MUST** reject an ELF that violates its version's
header, relocation, symbol, text, entrypoint, syscall-resolution, or
bytecode-verification rule before it can execute.

### 12.8.6 Program input ABI

The deprecated loader uses ABI v0. The supported BPF Loader and Upgradeable BPF Loader
use ABI v1. In both cases input begins at `0x400000000` with `account_count:u64` in
little-endian order. Duplicate account entries contain an index of the first occurrence
and alias that occurrence.

ABI v1 nonduplicate account record:

**Normative table.**

| Order | Field | Encoding / length | Writable by program |
|---:|---|---|---|
| 1 | duplicate marker | byte `0xff` | no |
| 2 | signer flag | `u8`, `0` or `1` | no |
| 3 | writable flag | `u8`, `0` or `1` | no |
| 4 | executable flag | `u8`, `0` or `1` | no |
| 5 | reserved | four zero bytes | no |
| 6 | account key | 32 bytes | no |
| 7 | owner | 32 bytes | only under owner-assignment rules |
| 8 | lamports | `u64` LE | only when the account is writable and balance rules permit |
| 9 | data length | `u64` LE | only when realloc rules permit |
| 10 | account data | original length bytes | only when program ownership/writability permits |
| 11 | realloc reserve | 10,240 zero bytes | used only for permitted growth |
| 12 | alignment padding | zero bytes through the next 8-byte boundary | no |
| 13 | rent epoch | `u64::MAX` LE | no |

An ABI v1 duplicate record has this layout:

**Normative table.**

| Relative byte range | Field | Encoding |
|---|---|---|
| `[0,1)` | duplicate index | index of the first occurrence |
| `[1,8)` | reserved | seven zero bytes |

After all account records, the input ends with:

**Normative table.**

| Order | Field | Encoding / length |
|---:|---|---|
| 1 | instruction-data length | `u64` LE |
| 2 | instruction data | declared number of bytes; `r2` points here |
| 3 | current program identifier | 32 bytes |

ABI v0 uses this nonduplicate record and a one-byte duplicate record:

**Normative table.**

| Order | Field | Encoding / length |
|---:|---|---|
| 1 | duplicate marker | `0xff`, or earlier index for the entire duplicate record |
| 2 | signer flag | `u8` |
| 3 | writable flag | `u8` |
| 4 | account key | 32 bytes |
| 5 | lamports | `u64` LE |
| 6 | data length | `u64` LE |
| 7 | account data | declared length bytes; no realloc reserve |
| 8 | owner | 32 bytes |
| 9 | executable flag | `u8` |
| 10 | rent epoch | `u64::MAX` LE |

**Normative table.**

| ABI v0 duplicate-record field | Encoding / length |
|---|---|
| duplicate index | one byte containing the earlier account index |

[SOL-SVM-0037] On successful VM exit, the runtime **MUST** deserialize only permitted
lamport, owner, data-length, and data changes. ABI v1 growth **MUST NOT** exceed 10,240
bytes over the entry length or the system maximum account-data length; readonly,
ownership, executable, rent, and lamport-conservation rules **MUST** be enforced before
instruction success.

### 12.8.7 Active syscall ABI

Every SBF syscall receives up to five `u64` arguments in `r1..r5` and returns a `u64` in
`r0`, or raises a VM/syscall error. V0–V2 link external symbols by the 32-bit
Murmur3-x86 hash with seed zero over the UTF-8 symbol bytes; V3 encodes the same hash
statically in `CALL imm` with `src=0`. Every guest pointer and integer in the following
structures is an eight-byte little-endian `u64`; Boolean fields are one byte and accept
only `0` or `1`. Thus the table's hash is both the V0–V2 relocation key and the V3
static index.

**Normative table. Common syscall guest structures.**

| Guest structure | Byte layout |
|---|---|
| byte-slice descriptor | `[0,8)` address, `[8,16)` element count |
| slice-of-slices descriptor | the same 16-byte descriptor, pointing to consecutive 16-byte byte-slice descriptors |
| processed-sibling header | `[0,8)` instruction-data length, `[8,16)` account-meta count |
| C instruction | `[0,8)` program-ID address, `[8,16)` account-meta address, `[16,24)` account-meta count, `[24,32)` data address, `[32,40)` data length |
| C account meta | `[0,8)` public-key address, byte `8` writable, byte `9` signer, bytes `[10,16)` padding |
| C account info | `[0,8)` key address, `[8,16)` lamports address, `[16,24)` data length, `[24,32)` data address, `[32,40)` owner address, `[40,48)` rent epoch, bytes `48`, `49`, `50` signer/writable/executable, bytes `[51,56)` padding |
| signer-seed group | one 16-byte descriptor pointing to consecutive byte-slice descriptors; the outer signer list is another descriptor array |
| Rust stable vector | `[0,8)` address, `[8,16)` capacity, `[16,24)` length |
| Rust CPI instruction | `[0,24)` account-meta stable vector, `[24,48)` instruction-data stable vector, `[48,80)` program ID |
| Rust CPI account meta | `[0,32)` public key, byte `32` signer Boolean, byte `33` writable Boolean; exactly 34 bytes |

For a Rust stable vector, capacity is not an accepted element count; it must be at least
length. The address points to `length` consecutive elements of the applicable element
size.

**Normative table. Complete epoch-`E` syscall registry and register ABI.**

| Symbol | Hash / static index | `r1, r2, r3, r4, r5` | `r0` and writes |
|---|---:|---|---|
| `abort` | `b6fc1a11` | ignored | always raises `Abort`; no writes |
| `sol_panic_` | `686093bb` | file bytes, length, line, column, ignored | validates UTF-8 then raises `Panic`; no writes |
| `sol_log_` | `207559bd` | UTF-8 bytes, length, ignored×3 | `0`; one program log |
| `sol_log_64_` | `5c2a3178` | five values | `0`; one hexadecimal-value log |
| `sol_log_pubkey` | `7ef088ca` | key `[32]`, ignored×4 | `0`; one base58-key log |
| `sol_log_compute_units_` | `52ba5096` | ignored×5 | `0`; logs remaining CU after its own charge |
| `sol_log_data` | `7317b434` | slice-descriptor array, count, ignored×3 | `0`; one base64 data log |
| `sol_create_program_address` | `9377323c` | seed descriptors, seed count, program ID `[32]`, output `[32]`, ignored | `0` and address on success; `1` and output unchanged for an on-curve/invalid derivation |
| `sol_try_find_program_address` | `48504a38` | seed descriptors, seed count, program ID `[32]`, address output `[32]`, bump output `[1]` | tries bump `255` down through `1`; `0` and both outputs on success, `1` unchanged if none |
| `sol_sha256` | `11f49d86` | slice descriptors, count, digest output `[32]`, ignored×2 | `0` and SHA-256 concatenation digest |
| `sol_keccak256` | `d7793abb` | slice descriptors, count, digest output `[32]`, ignored×2 | `0` and Keccak-256 concatenation digest |
| `sol_secp256k1_recover` | `17e40350` | digest `[32]`, recovery ID, signature `[64]`, output `[64]`, ignored | `0` and uncompressed `x\|\|y`; `1/2/3` for invalid hash/recovery-ID/signature, output unchanged |
| `sol_curve_validate_point` | `aa2607ca` | curve ID, encoded point, ignored×3 | `0` valid, `1` invalid; no writes |
| `sol_curve_group_op` | `dd1c41a6` | curve ID, operation, left, right, output | `0` and point on success; `1` and output unchanged on invalid input |
| `sol_curve_multiscalar_mul` | `60a40880` | curve ID, scalar array, point array, count, output | `0` and point, or `1` unchanged |
| `sol_curve_decompress` | `080c98b0` | curve ID, compressed point, output, ignored×2 | `0` and uncompressed BLS point, or `1` unchanged |
| `sol_curve_pairing_map` | `f111a47e` | curve ID, pair count, G1 array, G2 array, GT output | `0` and 576-byte GT element, or `1` unchanged |
| `sol_get_clock_sysvar` | `d56b5fe9` | output, ignored×4 | `0`; writes 40-byte Clock in field order from Section 13.4 |
| `sol_get_epoch_schedule_sysvar` | `23a29a61` | output, ignored×4 | `0`; writes the five fields from Section 13.4 in a 40-byte C layout, with seven zero padding bytes after the Boolean |
| `sol_get_rent_sysvar` | `bf7188f6` | output, ignored×4 | `0`; writes the 24-byte C layout below |
| `sol_get_last_restart_slot` | `188a0031` | output, ignored×4 | `0`; writes one `u64` slot |
| `sol_get_epoch_rewards_sysvar` | `fdba2b3b` | output, ignored×4 | `0`; writes the 96-byte aligned C layout below |
| `sol_get_sysvar` | `13c1b505` | sysvar ID `[32]`, output, byte offset, byte length, ignored | `0` and requested bytes; `1` if range exceeds data; `2` if absent; outputs unchanged for `1/2` |
| `sol_get_epoch_stake` | `5be92f4a` | null or vote key `[32]`, ignored×4 | total active epoch stake for null, otherwise stake delegated to key or zero |
| `sol_memcpy_` | `717cc4a3` | destination, source, byte length, ignored×2 | `0`; exact copy; overlap raises `CopyOverlapping` |
| `sol_memmove_` | `434371f8` | destination, source, byte length, ignored×2 | `0`; overlap-safe copy |
| `sol_memset_` | `3770fb22` | destination, value, byte length, ignored×2 | `0`; writes low byte of value |
| `sol_memcmp_` | `5fdcde31` | first, second, byte length, result `i32`, ignored | `0`; result is first differing `u8(first)-u8(second)`, or zero |
| `sol_get_processed_sibling_instruction` | `adb8efc8` | reverse sibling index, header, program-ID output, data output, account-meta output | `1` if found, else `0`; when found, if header `data_len` equals the sibling's instruction-data length and header `accounts_len` equals its account-meta count, writes program ID `[32]`, `data_len` data bytes and `accounts_len` 34-byte metas and leaves the header unchanged; otherwise (either field larger or smaller) writes the exact required lengths into the header and no other output |
| `sol_get_stack_height` | `85532d94` | ignored×5 | current instruction stack height, top-level `1` |
| `sol_set_return_data` | `a226d3eb` | bytes, length, ignored×3 | `0`; replaces current return data, maximum 1,024 bytes |
| `sol_get_return_data` | `5d2245e4` | data output, capacity, producer-key output, ignored×2 | full stored length; if copied length is nonzero writes its prefix and producer `[32]`, otherwise writes neither |
| `sol_invoke_signed_c` | `a22b9c85` | C instruction, account-info array, count, signer groups, count | `0` on successful CPI; otherwise raises the callee/CPI error |
| `sol_invoke_signed_rust` | `d7449092` | Rust instruction, account-info array, count, signer groups, count | same |
| `sol_alt_bn128_group_op` | `ae0c318b` | selector, input, input length, output, ignored | `0` and result; `1` unchanged for invalid curve input |
| `sol_alt_bn128_compression` | `334fd5ed` | selector, input, input length, output, ignored | `0` and result; `1` unchanged for invalid curve input |
| `sol_poseidon` | `c4947c21` | parameter ID, endian ID, slice descriptors, count, output `[32]` | `0` and digest; invalid field input returns `1` unchanged; invalid selector/count raises before charging |

Every row in this registry is available to a newly deployed V0, V1, V2, or V3 program in
epoch `E`; the SBF version changes only whether the hash is resolved through an ELF
relocation or a static call. All feature gates that introduce these rows are active at
`E`, so no row has an additional per-program feature condition. Availability for an
already deployed program is derived by `ProgramEnvironment` from its loader and
deployment slot, the invoking slot, and feature activation history, as required by
`SOL-SVM-0039`.

`sol_blake3`, `sol_sha512`, `sol_big_mod_exp`, `sol_remaining_compute_units`, and
`sol_get_fees_sysvar` are not registered in the epoch-`E` environment. `sol_alloc_free_`
is not accepted in a new deployment; a program deployed under a retained historical
environment may resolve it only as described below.

Before any dereference, `address+length` and descriptor-array multiplication are checked
for `u64`/host-size overflow. An output must lie wholly in writable guest memory and
every input wholly in readable guest memory. Typed `u64`, `i32`, descriptor, point,
scalar, CPI, and sysvar pointers have their natural alignment; byte and public-key
buffers have alignment one. With the active parameter-address restriction, fixed/generic
sysvar destinations may not begin in the serialized-input region. With the active
output-overlap rule, cryptographic, PDA, hash, and return-data output intervals must not
overlap an input interval to the same call. CPI additionally pins every supplied key,
owner, lamport, and data pointer to that account's ABI address and requires its mutable
data-length word to be outside serialized account memory.

The direct fixed-sysvar syscalls copy their in-memory `repr(C)` objects, not their
compact account serializations.

**Normative table. Direct-sysvar syscall C layouts.**

| Sysvar / byte range | Field and exact encoding |
|---|---|
| Clock `[0,8)` | `slot:u64` little-endian |
| Clock `[8,16)` | `epoch_start_timestamp:i64` little-endian |
| Clock `[16,24)`, `[24,32)`, `[32,40)` | `epoch:u64`, `leader_schedule_epoch:u64`, `unix_timestamp:i64`; exactly 40 bytes total |
| EpochSchedule `[0,8)`, `[8,16)` | `slots_per_epoch:u64`, `leader_schedule_slot_offset:u64` |
| EpochSchedule `[16,17)`, `[17,24)` | `warmup:u8` Boolean, then seven zero padding bytes |
| EpochSchedule `[24,32)`, `[32,40)` | `first_normal_epoch:u64`, `first_normal_slot:u64`; exactly 40 bytes total |
| Rent `[0,8)`, `[8,16)` | `lamports_per_byte_year:u64`, then the IEEE-754 binary64 bits of `exemption_threshold` |
| Rent `[16,17)`, `[17,24)` | `burn_percent:u8`, then seven zero padding bytes; exactly 24 bytes total |
| LastRestartSlot `[0,8)` | `last_restart_slot:u64`; exactly 8 bytes total |
| EpochRewards `[0,8)`, `[8,16)`, `[16,48)` | `distribution_starting_block_height:u64`, `num_partitions:u64`, `parent_blockhash[32]` |
| EpochRewards `[48,64)`, `[64,72)`, `[72,80)` | 16-byte-aligned `total_points:u128`, `total_rewards:u64`, `distributed_rewards:u64` |
| EpochRewards `[80,81)`, `[81,96)` | `active:u8` Boolean, then fifteen zero padding bytes; exactly 96 bytes total |

These writes require natural alignment and are all-or-nothing.

Except for the documented sibling-header length update and operations whose purpose is
memory copying, no syscall makes a partial output: it charges at the point specified in
Section 12.8.8, validates selectors/counts and all input/output mappings, computes into
temporary host state, then writes the complete result. An invalid
pointer/alignment/overlap or arithmetic overflow raises a syscall-originated instruction
error; an ordinary nonzero result in the table does not raise an instruction error.

The curve selectors and fixed operand representations are:

**Normative table. Curve selectors and operands.**

| Curve ID | Representation | Point/scalar bytes | Permitted operation |
|---:|---|---|---|
| `0` | Curve25519 Edwards | point `32`, scalar `32` | validate; group add `0`, subtract `1`, multiply `2`; MSM |
| `1` | Curve25519 Ristretto | point `32`, scalar `32` | validate; group add `0`, subtract `1`, multiply `2`; MSM |
| `4` / `132` | BLS12-381 pairing, little-/big-endian | G1 `96`, G2 `192`, GT `576` | pairing map over `0..=8` pairs |
| `5` / `133` | BLS12-381 G1, little-/big-endian | uncompressed point `96`, compressed point `48`, scalar `32` | validate; group add `0`, subtract `1`, multiply `2`; decompress |
| `6` / `134` | BLS12-381 G2, little-/big-endian | uncompressed point `192`, compressed point `96`, scalar `32` | validate; group add `0`, subtract `1`, multiply `2`; decompress |

For group add and subtract, the left and right pointers each address one point. For
multiply, the left pointer addresses one scalar and the right pointer one point.
Curve25519 MSM addresses consecutive 32-byte scalars and points and accepts `0..=512`
pairs. A successful curve operation returns `0`; an invalid point or scalar returns `1`;
an unsupported selector is an invalid attribute. BLS little-endian encodes each 48-byte
base-field element least significant byte first and orders each G2 extension element as
`c0,c1`; big-endian reverses each base-field element and orders it as `c1,c0`.

BN254 `sol_alt_bn128_group_op` receives
`(selector,input_address,input_length,result_address)`. A BN254 G1 point is
`x[32] || y[32]`; a G2 point is four 32-byte base-field elements. Big-endian G2 order is
`x.c1 || x.c0 || y.c1 || y.c0` as in EIP-197; little-endian order is
`x.c0 || x.c1 || y.c0 || y.c1`. Each coordinate and scalar uses the selector's byte
order.

**Normative table.**

| BN254 selector | Operation | Accepted input and output |
|---:|---|---|
| `0` | big-endian G1 add | input at most `128` bytes, right-padded with zeros to `P[64] \|\| Q[64]`; output `64` bytes |
| `128` | little-endian G1 add | input exactly `128` bytes `P[64] \|\| Q[64]`; output `64` bytes |
| `2` | big-endian G1 multiply | input at most `96` bytes, right-padded with zeros to `P[64] \|\| scalar[32]`; output `64` bytes |
| `130` | little-endian G1 multiply | input exactly `96` bytes `P[64] \|\| scalar[32]`; output `64` bytes |
| `3` / `131` | big-/little-endian pairing | input length a multiple of `192`, each record `G1[64] \|\| G2[128]`; output `32` bytes encoding zero or one |
| `4` / `132` | big-/little-endian G2 add | input exactly `256` bytes `P[128] \|\| Q[128]`; output `128` bytes |
| `6` / `134` | big-/little-endian G2 multiply | input exactly `160` bytes `P[128] \|\| scalar[32]`; output `128` bytes |

BN254 compression uses the same endian flag and independent base selectors:

**Normative table.**

| Selector | Operation | Input | Output |
|---:|---|---:|---:|
| `0` / `128` | big-/little-endian G1 compress | `64` | `32` |
| `1` / `129` | big-/little-endian G1 decompress | `32` | `64` |
| `2` / `130` | big-/little-endian G2 compress | `128` | `64` |
| `3` / `131` | big-/little-endian G2 decompress | `64` | `128` |

Poseidon accepts only parameter ID `0` (BN254, x-to-the-fifth S-box), endian ID `0` for
big-endian or `1` for little-endian, and at most 12 input descriptors. Every nonempty
input is exactly 32 bytes and encodes a field element strictly below the BN254
scalar-field modulus; the result is exactly 32 bytes in the selected byte order. A
zero-input or invalid-field invocation is charged and returns `1` with output unchanged.

[SOL-SVM-0038] Every active cryptographic syscall **MUST** enforce the selector, operand
length, byte order, point/scalar validity, count limit, and output length in the tables
above; it **MUST NOT** truncate an oversized operand or accept a noncanonical field
encoding.

Programs deployed before the active `disable_deploy_of_alloc_free_syscall` boundary may
retain `sol_alloc_free_`; its arguments are allocation size and free address, and a zero
free address allocates from the 32-KiB bump heap.

[SOL-SVM-0039] `ProgramEnvironment` **MUST** select the syscall surface above for new
deployments and derive the historically compatible surface for an already deployed
program from protocol inputs. Only a program deployed before the active
`disable_deploy_of_alloc_free_syscall` boundary may resolve `sol_alloc_free_`. Any
other imported symbol **MUST** fail loading or resolution.

### 12.8.8 Compute schedule

The following active cost parameters are in compute units. A syscall consumes its charge
with checked depletion before returning its result; any overrun is
`ComputationalBudgetExceeded`. Variable-length operations use saturating arithmetic and
the formulas in the syscall contract.

**Normative table.**

| Parameter | CU | Parameter | CU |
|---|---:|---|---:|
| syscall base | 100 | memory-operation base | 10 |
| log five `u64` values | 100 | log public key | 100 |
| create/try-find program address per attempt | 1,500 | CPI invocation base | 946 |
| bytes per CPI CU | 250 | sysvar base | 100 |
| SHA-256/Keccak-256 base | 85 | hash byte coefficient | 1 |
| secp256k1 recover | 25,000 | heap page cost per additional 32 KiB | 8 |
| Edwards validate/add/subtract/multiply | 159 / 473 / 475 / 2,177 | Ristretto validate/add/subtract/multiply | 169 / 521 / 519 / 2,208 |
| Edwards MSM base/increment | 2,273 / 758 | Ristretto MSM base/increment | 2,303 / 788 |
| BN254 G1 add/multiply | 334 / 3,840 | BN254 G2 add/multiply | 535 / 15,670 |
| BN254 pairing first/additional pair | 36,364 / 12,121 | Poseidon coefficient `a` / constant `c` | 61 / 542 |
| BN254 G1 compress/decompress | 30 / 398 | BN254 G2 compress/decompress | 86 / 13,610 |
| BLS G1 add/subtract/multiply | 128 / 129 / 4,627 | BLS G2 add/subtract/multiply | 203 / 204 / 8,255 |
| BLS G1 decompress/validate | 2,100 / 1,565 | BLS G2 decompress/validate | 3,050 / 1,968 |
| BLS pairing first/additional pair | 25,445 / 13,023 | maximum SHA-256 slices | 20,000 |

Native dispatch contributes to the same execution term `X` in Section 10.4.1. The fixed
charge is consumed before the native instruction body returns, so an instruction error
does not refund it. Any nested CPI, SBF execution, syscall, or explicitly additive
operation is charged in addition:

**Normative table.**

| Native dispatch or operation | Exact compute charge |
|---|---:|
| System-program instruction | `150` CU |
| Compute-budget-program instruction | `150` CU |
| Vote-program instruction | `2,100` CU |
| each Vote-program BLS proof-of-possession verification | additional `34,500` CU |
| direct BPF-loader management instruction | `570` CU, then fail as unsupported |
| direct deprecated-BPF-loader management instruction | `1,140` CU, then fail as unsupported |
| upgradeable-BPF-loader management instruction | `2,370` CU |

The ZK ElGamal proof program has zero additional dispatch base and consumes the
following instruction charge before proof verification or context closure:

**Normative table.**

| ZK ElGamal proof instruction | Exact compute charge |
|---|---:|
| CloseContextState | `3,300` CU |
| VerifyZeroCiphertext | `6,000` CU |
| VerifyCiphertextCiphertextEquality | `8,000` CU |
| VerifyCiphertextCommitmentEquality | `6,400` CU |
| VerifyPubkeyValidity | `2,600` CU |
| VerifyPercentageWithCap | `6,500` CU |
| VerifyBatchedRangeProofU64 | `111,000` CU |
| VerifyBatchedRangeProofU128 | `200,000` CU |
| VerifyBatchedRangeProofU256 | `368,000` CU |
| VerifyGroupedCiphertext2HandlesValidity | `6,400` CU |
| VerifyBatchedGroupedCiphertext2HandlesValidity | `13,000` CU |
| VerifyGroupedCiphertext3HandlesValidity | `8,100` CU |
| VerifyBatchedGroupedCiphertext3HandlesValidity | `16,400` CU |

Poseidon cost for `n` inputs is `61*n^2 + 542`. MSM cost for positive length `n` is
`base + (n-1)*increment`. BLS pairing costs `25,445 + 13,023*max(n-1,0)`. BN254 pairing,
where `n=floor(input_length/192)`, costs
`36,364 + 12,121*max(n-1,0) + 85 + input_length + 32`; a malformed nonmultiple length
is still charged by that formula before returning failure. Each generic hash syscall
accepts at most 20,000 slices, charges 85 once, then for each slice of length `L`
charges `max(10, floor(L/2))`. BN254 compression costs `100` plus the applicable 30,
398, 86, or 13,610 row constant.

The byte-dependent and invocation charges are:

**Normative table.**

| Operation | Exact compute charge |
|---|---:|
| executed SBF instruction | `1` CU |
| provision one SBF VM heap of `H` bytes | `8 * (ceil(H / 32768) - 1)` CU |
| `sol_log_` with `L` bytes | `max(100,L)` CU |
| `sol_panic_` with file-name length `L` | `L` CU |
| `sol_log_64_`, `sol_log_pubkey`, or compute-unit log | `100` CU |
| `sol_log_data` with `n` slices of lengths `L_i` | `100 + 100*n + sum(L_i)` CU |
| memory copy, move, set, or compare of `L` bytes | `max(10,floor(L/250))` CU |
| generic hash over slices of lengths `L_i` | `85 + sum(max(10,floor(L_i/2)))` CU |
| create program address | `1,500` CU |
| try-find program address returning bump `b` | `1,500 * (256-b)` CU; bumps are tried `255` down through `1` |
| try-find program address with no successful bump | `1,500 * 256` CU |
| set return data of length `L` | `100 + floor(L/250)` CU |
| get and copy `L` return-data bytes plus producer key | `100 + floor((L+32)/250)` CU; just `100` when `L=0` |
| direct Clock / EpochSchedule / Rent / LastRestartSlot / EpochRewards syscall | `140 / 140 / 124 / 108 / 196` CU |
| generic sysvar copy of `L` bytes | `100 + floor(32/250) + max(10,floor(L/250))` CU |
| epoch stake, null key | `100` CU |
| epoch stake, vote key supplied | `110` CU |
| CPI fixed entry | `946` CU, before argument translation and callee execution |

For one CPI with `m` account metas, `i` account-info records, instruction-data length
`D`, and unique callee account-data lengths `A_j`, argument translation adds:

**Normative algorithm.**

```text
floor(D / 250)
+ floor(34*m / 250)
+ floor(80*i / 250)
+ sum(floor(A_j / 250))
```

The 80-byte account-info term is the active accounting size, independent of whether the
guest used the C or Rust descriptor. Duplicate instruction accounts do not repeat the
`A_j` term. Callee instructions, heap provisioning, syscalls, and nested CPI continue to
draw from the same meter. All divisions in this subsection are unsigned integer floor;
all sums and products saturate unless a named operation requires checked failure.

**Normative table. Syscall charge point and failure charging.**

| Syscall family | Charge ordering |
|---|---|
| panic, logging, PDA, secp recovery, memory, return-data, sibling/stack, fixed/generic sysvar, epoch stake, and CPI | consume the listed base or length charge before translating the first guest pointer; later validation failure retains it |
| generic SHA-256/Keccak-256 | reject a count above 20,000 without charge; otherwise consume 85, validate the output and descriptor array, then consume each slice's charge immediately before hashing that slice |
| Curve25519/BLS group operations | reject an unsupported selector without charge; for a supported selector consume the complete operation/count charge before translating points, scalars, or output |
| BN254 group operation or compression | reject an unsupported/inactive selector without charge; otherwise compute and consume the full selected charge before pointer and curve-input validation |
| Poseidon | reject an invalid parameter ID, endian ID, count above 12, or cost overflow without charge; otherwise consume `61*n^2+542` before pointer, field, and output validation |

No validation failure refunds compute already consumed. A syscall that returns an
ordinary nonzero status instead of raising an instruction error consumes the same charge
as the corresponding successful selected operation.

[SOL-SVM-0040] The runtime **MUST** calculate and consume every native dispatch, SBF
instruction, heap, syscall, memory, cryptographic, return-data, sysvar, and CPI charge
exactly by the tables and formulas above. All such charges **MUST** draw from the same
transaction compute meter, and the remaining units at every protocol-visible boundary
**MUST** follow the same charge sequence.

### 12.8.9 Completion, deserialization, and failure

On an ordinary VM exception other than a syscall-originated instruction error, the
active `deplete_cu_meter_on_vm_failure` rule consumes all remaining transaction compute.
A syscall-originated instruction error retains the compute already charged by that path.
Successful VM exit is not instruction success until parameter deserialization,
account-change validation, lamport balance, borrow checks, and invocation-pop checks all
succeed.

[SOL-SVM-0041] SBF execution **MUST** follow the abstract machine independently of the
host execution strategy. An access fault, arithmetic fault, invalid call, call-depth
failure, verifier escape, or compute exhaustion **MUST NOT** commit program writes; a
nonzero exit status or post-execution account-validation failure **MUST** enter the same
Chapter 11 rollback path.
