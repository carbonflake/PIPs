---
pip: 56
title: Native Asset Layer
description: Fungible, non-fungible and semi-fungible assets executed off-chain by a bonded operator committee and settled on Pactus through bounded, BLS-attested anchors.
author: CarbonFlake (@carbonflake)
status: Draft
type: Standards Track
category: Core
created: 2026-10-02
requires: 51, 53, 54
---

## Abstract

This PIP adds native assets to Pactus without a virtual machine and without making every validator store or
replay asset balances.

Asset balances and operations live in an **Asset Layer** run by a committee of bonded **operators**. Pactus
keeps one small settlement record: the current asset-state root, a data commitment, a PAC escrow, a deposit
queue and the operator registry. Operators advance it with an **anchor**: a transaction of at most about
2.4 KB, whatever the number of operations it covers, carrying one aggregate BLS signature from a quorum of the
committee.

Three asset shapes exist: fungible, non-fungible and semi-fungible. Minting, burning and mutable parameters are
capabilities, not extra types. A fungible asset may carry a bounded transfer policy (a team share and a burn
share). Users pay execution fees in PAC through an escrowed fee credit.

**Version 1 is a committee-attested design with fraud proofs, and is transitional.** Pactus validators check
that a bonded committee signed a transition, not that it is correct. Anyone who holds the batch data can prove
a single wrong step: an anchor stays pending for a day, a proven fraud reverts it and burns its signers'
bonds, and nothing is paid out on Pactus until the day has passed. Pactus does not check that the data is
available, which is the main thing v1 does not solve. The PAC escrow is capped on a ramp, seats have terms,
and if the committee stops anchoring for 30 days anyone can freeze the layer: users then recover their fee
credit with a Merkle proof, record their native positions in an on-chain registry, and the committee can
resume. The anchor carries an attestation-type byte so that a later PIP can add validity proofs without
changing the settlement record.

## Motivation

Pactus has no smart-contract layer. A token today means off-chain indexers, memos, or another chain. Putting
tokens into consensus does not scale: a block holds at most 1000 transactions every 10 seconds (100 per
second for everything), every validator would hold every balance and replay every transfer, and that state
would grow with token usage instead of PAC usage.

This PIP separates the two jobs. **Pactus** stays small: it orders anchors, verifies one aggregate signature
per anchor, holds escrowed PAC and keeps the operator registry, at a cost independent of the number of
operations. **The Asset Layer** executes operations, keeps balances and retains the data; its capacity depends
on operator hardware. In v1 every operator re-executes every batch, so adding operators adds no capacity
(section 6.3).

Pactus prefers off-chain solutions to off-chain problems, and the Asset Layer is off-chain. Three things only
Pactus consensus can give:

1. **One canonical head with Pactus finality.** The head advances only through a Pactus transaction.
2. **Custody of PAC.** Fee credit is backed by PAC held in escrow under rules every validator enforces: payouts
   never exceed credited deposits, and an uncredited deposit can be reclaimed.
3. **Enforceable accountability.** Only Pactus can lock operator bonds, accept evidence of double-signing and
   burn them.

Everything else stays off-chain. The on-chain footprint is seven payload types, one settlement record, a
deposit queue and, after a halt, two registries of exit claims and receipts (section 4).

The commands are few: create an asset, transfer, mint, burn, update a bounded parameter, request a PAC
withdrawal. They cover a fungible token, an NFT collection and a semi-fungible collection. Exchanges, lending,
oracles, governance, rebasing, freezing and clawback are outside this PIP.

## Goals

* Native fungible, non-fungible and semi-fungible assets with no VM.
* A Pactus cost per anchor that is bounded and independent of operation count.
* PAC-denominated fees with escrowed funds that a user can recover: an uncredited deposit can be reclaimed, and
  fee credit can be recovered with a Merkle proof if the layer stops for 30 days.
* A halt that is neither final nor silent: the committee can resume, and a holder can record a native position
  on Pactus with a proof.
* A PAC escrow cap that starts low and ramps up.
* A proof of one wrong step that stops a false head, makes its signers pay, and keeps Pactus from paying
  anything for it until a challenge window has passed.
* On-chain accountability for conflicting attestations, seat terms against entrenchment, and replacement of
  inactive members.
* Operator revenue from user fees only, with floors that price state growth.
* Byte-exact encodings and test vectors for every consensus object and for the state transition.
* A versioned attestation slot so that validity proofs can replace committee attestation later.

## Non-Goals

* A validity proof in v1.
* A guarantee that batch data is available (sections 1.2 and 3.7).
* Using a native asset on Pactus. After a halt a receipt records who owned what (section 3.5), but nothing
  spends it: a successor layer, defined by a later PIP, would honor it.
* Forced inclusion of operations that operators refuse. Without a validity proof Pactus cannot tell a
  legitimate rejection from censorship.
* A rescue committee. If more than `n - T` operators are gone, the layer cannot resume (section 3.6).
* A general policy language, the off-chain protocol operators use among themselves, or a subsidy for operators.

## Specification

Integers are little-endian and fixed width unless a field is marked *varint*.
`Address` is the 21-byte Pactus address; its first byte is the address type (`0` treasury, `1`
validator, `2` BLS account, `3` Ed25519 account, `4` secp256k1 account). `Hash` is BLAKE2b-256, the hash Pactus
already uses (`crypto/hash`). `||` is byte concatenation. "Zero address" is 21
zero bytes. Amounts on Pactus are `int64` nano PAC and every sum MUST be
rejected before it exceeds `MaxNanoPAC`, as in PIP-54. Quantities inside the
Asset Layer are `uint64` and every operation MUST use checked arithmetic.

### 1. Overview and trust model

#### 1.1 Roles

| Role | Does | Does not |
| --- | --- | --- |
| Pactus validators | Execute the seven new payloads, verify one BLS aggregate per anchor, commit the settlement state | Store balances, download or replay asset operations |
| Operators | Bond PAC, execute and attest batches, retain all batch data, serve state | Mint or move PAC outside the rules of section 3 |
| Users | Sign asset operations with a type-4 secp256k1 key, deposit PAC for fee credit | Need to know operators exist when moving PAC on Pactus |

#### 1.2 Trust model

Let `n` be the committee size of an epoch. The **quorum** is `T = max(floor(2n/3) + 1, n - floor((n - 1) / 6))`:
two thirds, or all but `floor((n - 1) / 6)` operators if that is more. For `n = 21` it is `T = 18`, so 3
operators may be offline; for `n = 51`, `T = 43`; for `n = 7`, `T = 6`. An anchor is accepted with `T`
signatures, and an honest operator signs only a batch that it holds in full and has re-executed (section 5.7).
If fewer than `T` operators are faulty, at least one signer of every head is honest and holds the batch. This
is the rule of the Arbitrum AnyTrust data-availability committee (all but one signature, at least two honest
members), applied to a committee that also validates.

| Property | Holds if |
| --- | --- |
| Pactus consensus, ordinary PAC, validator stake | Independent of the committee. A faulty committee cannot reach them |
| A final head is a correct execution | Fewer than `T` operators signed an incorrect head, **or** a party that holds the batch data submits a fraud proof within `ChallengeWindow` and a validator includes it |
| A wrong head that is challenged in time does no harm on Pactus | Always: it is reverted, nothing is paid for it, and its signers lose their bonds |
| The layer keeps advancing | At least `T` operators are online and honest |
| The data needed to check or rebuild state exists | Fewer than `T` operators are faulty. With `T` or more nothing guarantees it. **Pactus does not check that data is held** |
| A user recovers fee credit, or proves a native position, after a halt | The final head is correct, and some node still holds the state needed for the Merkle proof. A receipt is a record, not a way to spend |
| A frozen layer can resume | At least `T` operators return |
| A conflicting pair of attested heads is punished | Always for the operators who signed both (at least `2T - n` of them) |

A committee of `T` or more colluding operators (18 of 21) can attest an incorrect head. It is pending for
`ChallengeWindow` blocks, and any one party with the batch data can stop it and be paid for it. The attack
therefore needs the committee to **keep the batch from every other party** for the whole window, or to count
on nobody checking. If it succeeds, native assets can be reassigned or destroyed and the PAC credited in escrow
can be drained, up to `MaxEscrow(h)` (50 000 PAC, reaching 500 000 PAC after 180 days). Ordinary PAC balances
are never reachable. The bond is slashed for a double signature in one epoch and for a head proven wrong; it is
not insurance for asset value. A caught committee loses `T` bonds, 180 000 PAC at the launch values.

#### 1.3 Parameters

The values below are initial proposals for community review. They are constants of the protocol version in the
Pactus node, and a later version may change them (section 1.3.1).

| Name | Value | Meaning |
| --- | --- | --- |
| `EpochLength` | 8640 blocks (one day) | The committee is fixed during an epoch. `epoch(h) = floor(h / EpochLength)` |
| `MaxCommittee` | 21 at activation | Maximum seats. A seat is a record with `ActiveFromEpoch != 0` and `ExitEpoch == 0`. At most 64 |
| `MaxOperatorRecords` | 42 | Maximum registry records (seated, waiting, exiting): twice `MaxCommittee` |
| `SeatTerm` | 180 epochs | After a term, a waiting operator can take the seat |
| `MaxRotationsPerEpoch` | 1 | Seats `Seat` may retire for an expired term in one epoch |
| `MaxRotationsPerTerm` | 6 | The same, in one term window `floor(e / SeatTerm)`. Below `T` on purpose |
| `WaitingEpochs` | 30 | Epochs after which a waiting record can be pruned when the registry is full |
| `MinCommittee` | 7 | An anchor is invalid when the epoch committee is smaller |
| `OperatorBond` | 10 000 PAC at activation | Exact bond of a new operator. A record keeps the amount it posted |
| `OperatorUnbondDelay` | 181 440 blocks (21 days) | Delay after leaving before the bond can be withdrawn. The Pactus `UnbondInterval` default |
| `MinDeposit` | 10 PAC | Smallest escrow deposit |
| `MaxEscrow(h)` | 50 000, 150 000, then 500 000 PAC | Maximum PAC in escrow. Steps at 777 600 blocks (90 days) and 1 555 200 blocks (180 days) after `ActivationHeight`. Only a later PIP may raise the final value |
| `HaltTimeout` | 259 200 blocks (30 days) | Blocks without an anchor after which anyone may freeze the layer |
| `DepositExpiryEpochs` | 2 | A deposit made in epoch `e` can be credited until the end of epoch `e + 1`; from `e + 2` it can only be reclaimed |
| `MaxDepositsPerAnchor` | 1 024 | Deposits one anchor may consume. MUST stay at or above `MaxTransactionsPerBlock` (1 000 today) |
| `MaxExitClaimsPerBlock` | 32 | `ExitClaim` transactions a block may contain |
| `MaxExitsPerResume` | 262 144 | PAC exit claims one `AssetResume` may apply. Resumes are serial, one per `ChallengeWindow`; claims can arrive at up to 276 480 per day |
| `MaxPayoutsPerAnchor` | 64 | PAC payouts one anchor may make |
| `AnchorRefund` | 10 000 000 nano PAC (0.01 PAC) | Largest refund of one anchor to its submitter: the default fixed fee of the reference node |
| `MaxBatchBytes` | 33 554 432 | Largest canonical batch (32 MiB) |
| `MaxRefLag` | 30 blocks | Largest lag of a batch `RefHeight` behind an operator's finalized height (sections 3.7 and 5.6) |
| `ChallengeWindow` | 8 640 blocks (one day) | Blocks during which an anchor or resume is pending and can be challenged |
| `MaxPending` | 864 | Records that may be pending at once |
| `MaxSteps` | 1 048 575 (2^20 - 1) | Steps in one batch. `DataRoot` and `TraceRoot` have at most `2^20` leaves, audit paths at most 20 siblings |
| `MaxFraudProofBytes` | 98 304 | Payload size of an `AssetFraud` |
| `MaxFraudProofsPerBlock` | 2 | `AssetFraud` transactions a block may contain |
| `OpenWindow` | 1 440 blocks (four hours) | Time to answer a request to open a leaf (section 3.8) |
| `OpenBond` | 10 PAC | Bond of a request to open a leaf, paid to the responder if answered |
| `MaxOpenRequests` | 4 | Open requests a record may have at once |
| `MaxFinalizePerBlock` | 4 | Records the finalization hook may finalize in one block (section 3.7) |

`ChallengeWindow + OpenWindow` MUST be shorter than `OperatorUnbondDelay`, so that an operator cannot leave
before it can be slashed.

##### 1.3.1 Changing a parameter

A change is a new value in the code of a protocol version, applied from the first epoch that starts after the
version activates (PIP-51). There is no transaction and no vote.

* **`OperatorBond`.** A record keeps the amount it posted (`Bond`, section 2.3); a `Bond` after the change pays
  the new amount. Withdrawals, unbonding and slashes use the record's own amount, so no existing operator is
  affected.
* **`MaxCommittee`.** A raise frees seats at the next epoch, filled by `Bond` and `Seat` as for any vacancy.
  `T` follows the size of each epoch's committee. A version MUST NOT lower `MaxCommittee` or
  `MaxOperatorRecords`.

A version that changes a parameter MUST keep `MaxCommittee <= 64` (the signer bitmap is at most 8 bytes),
`MinCommittee <= MaxCommittee <= MaxOperatorRecords`, and `MaxRotationsPerTerm < T` of a full committee. At 51
seats with `MaxRotationsPerTerm = 6`, taking `T = 43` seats by rotation needs eight windows, so such a version
SHOULD review `MaxRotationsPerTerm`.

#### 1.4 Names used below

`ChainID` is the Pactus genesis hash (`Genesis.Hash()`); a node already knows it
and the sandbox MUST expose it. Every signed message in this PIP starts with a
distinct ASCII domain string and includes `ChainID`, so a signature made for one
purpose or one network is never valid for another.

### 2. Settlement state on Pactus

#### 2.1 Activation and state root

The feature activates with protocol version `V`, the next unassigned version when
this PIP is accepted (`6` if PIP-50 activates first). The payload types below take the seven next
free numbers, 8 to 14, on the assumption that PIP-50 uses type 7. Other Drafts also add payload types
without fixing numbers (PIP-48 adds `SetDelegation`, PIP-33 adds lock and unlock): the version that
activates first takes the next free numbers, and the editors renumber this PIP when assigning. No vector of
this PIP contains a payload type number, so renumbering changes none of them. `V` MUST NOT activate
before epoch 1, so that the epoch value `0` can mean "none" in the fields below.

Pactus block headers commit to the **pre-state** of the block. For a block whose
version is below `V` the root is unchanged:

```text
stateRoot = HashMerkleBranches(accountMerkle.Root(), validatorMerkle.Root())
```

For a block whose version is `V` or higher:

```text
assetRoot = HashMerkleBranches(Hash(SettlementState),
                               HashMerkleBranches(HashMerkleBranches(depositMerkle.Root(),
                                                                     anchorMerkle.Root()),
                                   HashMerkleBranches(exitMerkle.Root(), exportMerkle.Root())))
stateRoot = HashMerkleBranches(HashMerkleBranches(accountMerkle.Root(),
                                                  validatorMerkle.Root()),
                               assetRoot)
```

`depositMerkle`, `anchorMerkle`, `exitMerkle` and `exportMerkle` are trees of the same kind
as `accountMerkle` (`util/persistentmerkle`): leaf `i` of `depositMerkle` holds
`Hash(DepositRecord_i)`, leaf `Seq` of `anchorMerkle` holds the leaf of the anchor or resume
with that sequence (sections 3.3 and 3.7), leaf `i` of `exitMerkle` holds the leaf of the
`i`-th PAC exit claim and leaf `i` of `exportMerkle` holds the leaf of the `i`-th receipt
(sections 2.4 and 3.5). Before the first anchor the settlement state is the
**initial settlement**: all counters zero, no operators, `Frozen = 0`,
`ExportsVoidBefore = 0`,
`ActivationHeight` equal to the height of the first block of version `V`,
`HeadSeq = 0`, `StateRoot = EmptyStateRoot` (section 5.2), `DataRoot` = 32 zero
bytes, `HeadHash = GenesisHeadHash` (section 2.5), `TipSeq = 0`, `TipHash = GenesisHeadHash`,
`TipStateRoot = EmptyStateRoot`, and four empty trees. The first
block of version `V` therefore commits to the initial settlement in its header
root.

Nodes persist the settlement state, deposit records and the block in the same
store batch that already persists accounts and validators.

#### 2.2 `SettlementState`

`Hash(SettlementState)` is the hash of this encoding:

| Field | Type | Notes |
| --- | --- | --- |
| `Version` | `uint8` | `1` |
| `ActivationHeight` | `uint32` | Height of the first block of version `V` |
| `Frozen` | `uint8` | `0` active, `1` frozen (claims open), `2` resuming (PAC claims closed, exits being applied). Section 3.5 |
| `ExportsVoidBefore` | `uint32` | Height at which the last resume completed, `0` if none. A receipt with `Height` below it is void |
| `HeadSeq` | `uint64` | Sequence of the **final** head. `0` before the first anchor |
| `HeadHash` | `Hash` | Section 2.5, of the final head |
| `StateRoot` | `Hash` | Asset Layer state root of the final head. Exits and receipts are proven against it |
| `DataRoot` | `Hash` | Batch data commitment of the final head |
| `TipSeq` | `uint64` | Sequence of the latest anchor or resume, pending or final. `TipSeq >= HeadSeq`; the records above `HeadSeq` are pending |
| `TipHash` | `Hash` | Head hash of the tip. The next anchor names it as `PrevHead` |
| `TipStateRoot` | `Hash` | State root of the tip. The next batch starts from it |
| `PendingOut` | `int64` | PAC that pending anchors will pay out when they are final |
| `PendingConsumed` | `int64` | PAC of deposits that pending anchors will credit |
| `OpenBonded` | `int64` | PAC held as bonds of open requests (section 3.8) |
| `LastAnchorHeight` | `uint32` | Height of the block that carried the last anchor or resume, or of the last revert. `0` if none |
| `TipFeeShareEpoch` | `uint32` | Epoch of the last fee share of the tip chain (section 3.3). `0` if none |
| `RotationEpoch` | `uint32` | Epoch of the last rotation by expired term. `0` if none |
| `RotationCount` | `uint8` | Rotations by expired term made in `RotationEpoch` |
| `RotationWindow` | `uint32` | Term window of the last rotation by expired term. `0` if none |
| `RotationWindowCount` | `uint8` | Rotations by expired term made in `RotationWindow` |
| `NextOperatorIndex` | `uint32` | Index given to the next operator |
| `DepositCount` | `uint64` | Deposits ever made. Next deposit sequence |
| `DepositCursor` | `uint64` | Cursor of the tip: deposit sequences below this value are consumed, reserved or skipped |
| `ExitCount` | `uint64` | PAC exit claims made (kind 0). Next leaf of `exitMerkle` |
| `ExitCursor` | `uint64` | PAC exit claims below this value have been applied to the Asset Layer state by a resume |
| `ExportCount` | `uint64` | Receipts recorded (kinds 1, 2 and 3). Next leaf of `exportMerkle` |
| `Escrow` | `int64` | PAC held for deposits and fee credit |
| `Consumed` | `int64` | Cumulative PAC of deposits credited by **final** anchors |
| `Paid` | `int64` | Cumulative PAC paid out by **final** anchors and by exit claims |
| `BondedTotal` | `int64` | PAC held as operator bonds |
| `OperatorCount` | `uint8` | At most `MaxOperatorRecords` |
| `Operators` | records | Section 2.3, ascending `Index` |

Invariants after every transaction: `0 <= Paid <= Consumed`,
`Escrow >= Consumed - Paid`, and, at the time a deposit is accepted,
`Escrow <= MaxEscrow(h)`. The value
`Escrow - (Consumed - Paid)` is the PAC of deposits that no final anchor has credited: pending
(`Status 0`) and reserved by a pending anchor (`Status 3`).

#### 2.3 Operator record

| Field | Type | Notes |
| --- | --- | --- |
| `Index` | `uint32` | Never reused |
| `Owner` | `Address` | Account that bonded and receives the bond back |
| `PublicKey` | 96 bytes | BLS12-381 public key, same ciphersuite as Pactus validators |
| `Bond` | `int64` | The amount posted: `OperatorBond` of the version in force at the `Bond` action, or `0` if slashed |
| `ActiveFromEpoch` | `uint32` | `0` while the operator is **waiting** for a seat. Else the first epoch in which it is in the committee |
| `TermEndEpoch` | `uint32` | `0` while waiting. Else `ActiveFromEpoch + SeatTerm` |
| `ExitEpoch` | `uint32` | `0` while not exiting. Else the first epoch without the operator |
| `WithdrawableHeight` | `uint32` | `0` while not exiting |
| `JoinedEpoch` | `uint32` | Epoch in which the record was bonded |
| `Flags` | `uint8` | Bit 0: slashed |

A record is **seated** if `ActiveFromEpoch != 0` and `ExitEpoch == 0`. It is
**waiting** if `ActiveFromEpoch == 0` and `ExitEpoch == 0`. The waiting list is
the waiting records in ascending `Index`.

The committee of epoch `e`, `Committee(e)`, is the records with
`1 <= ActiveFromEpoch <= e` and (`ExitEpoch == 0` or `e < ExitEpoch`), in ascending
`Index`. Membership never changes inside an epoch, because every registry
transaction sets `ActiveFromEpoch` or `ExitEpoch` to a later epoch.

#### 2.4 `DepositRecord`

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | Account that paid. Receives any reclaim |
| `For` | `Address` | Type-4 address credited in the Asset Layer |
| `Amount` | `int64` | Nano PAC |
| `Height` | `uint32` | Block that included the deposit |
| `Status` | `uint8` | `0` pending, `1` reclaimed, `2` credited, `3` reserved by a pending anchor |

A deposit **expires in epoch `e`** when `e >= epoch(Height) + DepositExpiryEpochs`.
It is **creditable in epoch `e`** when `Status == 0` and it has not expired in `e`.
Whether a deposit is creditable in a given epoch depends only on its record and the
epoch, never on later transactions of that epoch.

Its leaf hash is `Hash(From || For || Amount || Height || Status)`. A node MAY
delete the body of a record whose `Status` is `1` or `2`; it MUST keep the leaf hash. The body of a
record in `Status 3` is needed to finalize or to challenge the anchor that reserved it.

#### 2.5 Heads and attestations

```text
GenesisContent = Hash("PAC-ASSET-GENESIS-1" || EmptyStateRoot)
GenesisHeadHash = Hash("PAC-ASSET-HEAD-1" || uint64(0) || 32 zero bytes || GenesisContent)

DepositDigest(oldCursor, newCursor, D) =
    Hash("PAC-ASSET-DEPOSITS-1" || uint64(oldCursor) || uint64(newCursor) ||
         for each deposit d in D, ascending sequence:
             uint64(d.Seq) || d.From || d.For || uint64(d.Amount))

PayoutsDigest(P) =
    Hash("PAC-ASSET-PAYOUTS-1" || uint32(len(P)) ||
         for each payout p in P: uint32(p.StepIndex) || p.To || uint64(p.Amount))

ContentHash = Hash("PAC-ASSET-ANCHOR-1" || StateRoot || DataRoot || TraceRoot || uint32(DataSize) ||
                   uint32(OpCount) || uint64(NewCursor) || DepositDigest || PayoutsDigest ||
                   uint64(FeeShare) || uint64(Refund) || uint32(EvictIndex))

AttestMessage = "PAC-ASSET-ATTEST-1" || ChainID || uint32(Epoch) || uint64(Seq) ||
                PrevHead || ContentHash                         (126 bytes)

HeadHash = Hash("PAC-ASSET-HEAD-1" || uint64(Seq) || PrevHead || ContentHash)

ExitDigest(oldCursor, newCursor, E) =
    Hash("PAC-ASSET-EXITS-1" || uint64(oldCursor) || uint64(newCursor) ||
         for each PAC exit claim e in E, ascending index:
             uint8(e.Kind) || e.Key || uint64(e.Amount))

ResumeContentHash = Hash("PAC-ASSET-RESUME-1" || StateRoot || DataRoot || TraceRoot ||
                         uint32(DataSize) || uint64(NewExitCursor) || ExitDigest)
```

A resume (section 3.6) is attested with the same `AttestMessage` as an anchor, using
`ResumeContentHash` in place of `ContentHash`, and its head is hashed the same way.

`D` is the set of deposit records with sequence in `[oldCursor, newCursor)` that are
creditable in the `Epoch` of the attestation. Records in that range that are not
creditable are skipped and credit nothing. The vectors of the Test Cases section cover these digests.

### 3. Payload types

Seven payload types are added. All seven pay the ordinary fixed fee of Pactus; none
is free. Unknown actions fail decoding.

```go
TypeAssetOperator = Type(8)
TypeAssetEscrow   = Type(9)
TypeAssetAnchor   = Type(10)
TypeAssetEvidence = Type(11)
TypeAssetResume   = Type(12)
TypeAssetFraud    = Type(13)
TypeAssetOpen     = Type(14)
```

#### 3.1 `AssetOperator` (type 8)

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Signer. An account address |
| `Action` | `uint8` | 1 | `0` Bond, `1` Unbond, `2` Withdraw, `3` Seat, `4` Prune |

Action `0` Bond adds:

| Field | Size | Notes |
| --- | --- | --- |
| `PublicKey` | 96 | BLS public key. MUST decode to a non-identity point of the prime-order subgroup, as `crypto/bls` already requires of public keys and signatures |
| `PoP` | 48 | BLS signature over `"PAC-ASSET-POP-1" \|\| ChainID \|\| PublicKey \|\| From` made with that key |

Actions `1`, `2` and `4` add `OperatorIndex` (`uint32`). Action `3` adds nothing.
`Value()` is `OperatorBond` for Bond and `0` otherwise. While `Frozen != 0`
(section 3.5), Bond, Seat and Prune are rejected; Unbond and Withdraw still work.
There is no action that evicts an operator for inactivity: the committee evicts
through the anchor (section 3.3, rule 11).

The proof of possession defeats rogue-key attacks and names the owner, so nobody can replay another party's key.

**Bond.** `Check`: `PoP` is valid; no registry record has this `PublicKey`; the
`PublicKey` is not the key of any Pactus validator (an operator key MUST NOT be a
validator key, so one key never signs both consensus votes and attestations); fewer
than `MaxOperatorRecords` records exist; `NextOperatorIndex < 0xFFFFFFFF` (that value
means "no operator" in an anchor); `Balance >= OperatorBond + fee`.
`Execute`: subtract `OperatorBond + fee` from `Balance`; add `OperatorBond` to
`BondedTotal`; append a record with `Index = NextOperatorIndex` (then increment),
`Owner = From`, `JoinedEpoch = epoch(h)`, `ExitEpoch = 0`, `Flags = 0`. If fewer than
`MaxCommittee` records are seated **and** the waiting list is empty, the record is
seated: `ActiveFromEpoch = epoch(h) + 1` and
`TermEndEpoch = ActiveFromEpoch + SeatTerm`. Otherwise it is waiting: those two
fields are `0`. A newcomer therefore never passes an operator already waiting.

**Unbond.** `From` is the record `Owner`; `ExitEpoch == 0`. A waiting record never
had power: its bond is credited to `Owner` at once and the record is deleted, as in
`Prune`. Otherwise set `ExitEpoch = epoch(h) + 1` and
`WithdrawableHeight = ExitEpoch * EpochLength + OperatorUnbondDelay`. A seated
record whose epoch has not started (`ActiveFromEpoch = epoch(h) + 1`) therefore
never joins a committee. The seat is free for `Seat` as soon as `ExitEpoch` is set.

**Withdraw.** `ExitEpoch != 0` and `h >= WithdrawableHeight`. `From` is any account;
the bond is credited to `Owner`. A slashed record has `Bond = 0`. `BondedTotal` decreases
by the bond and the record is deleted.

**Seat.** `From` is any account. Let `W` be the first record of the waiting list;
it MUST exist. One of two cases applies:

* *Free seat:* fewer than `MaxCommittee` records are seated. Promote `W`.
* *Expired term:* otherwise let `X` be the seated record with
  `TermEndEpoch <= epoch(h)` that has the lowest `(TermEndEpoch, Index)`; it MUST
  exist. Let `e = epoch(h)` and `w = floor(e / SeatTerm)`. If `RotationEpoch != e`,
  set `RotationEpoch = e` and `RotationCount = 0`. If `RotationWindow != w`, set
  `RotationWindow = w` and `RotationWindowCount = 0`. `RotationCount` MUST be below
  `MaxRotationsPerEpoch` and `RotationWindowCount` below `MaxRotationsPerTerm`; then
  increment both. Set `X.ExitEpoch = e + 1` and
  `X.WithdrawableHeight = X.ExitEpoch * EpochLength + OperatorUnbondDelay`
  (no slash), then promote `W`.

To promote `W`: `ActiveFromEpoch = epoch(h) + 1` and
`TermEndEpoch = ActiveFromEpoch + SeatTerm`. In the second case `X` leaves and `W`
joins at the same epoch boundary, so the committee size does not change. A seat whose term has expired stays with its holder while the waiting list is empty. One call changes one seat.

**Prune.** `From` is any account. Requires a waiting record, a full registry
(`OperatorCount == MaxOperatorRecords`) and `epoch(h) - JoinedEpoch >= WaitingEpochs`.
Effect: `BondedTotal` decreases by the bond, the bond is credited to the `Owner`, and
the record is deleted. A pruned operator may bond again, at the back of the queue.

#### 3.2 `AssetEscrow` (type 9)

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Signer |
| `Action` | `uint8` | 1 | `0` Deposit, `1` Reclaim, `2` Freeze, `3` ExitClaim |

Deposit adds `For` (`Address`, 21) and `Amount` (*varint*); `Value()` is
`Amount`. Reclaim adds `DepositSeq` (`uint64`); `Value()` is `0`. Freeze adds
nothing. ExitClaim is defined in section 3.5. For Freeze and ExitClaim `Value()` is
`0`.

**Deposit.** `Frozen == 0`. `For` MUST be a type-4 secp256k1 address, not the zero
address. `Amount >= MinDeposit`. `Balance >= Amount + fee`;
`Escrow + Amount <= MaxEscrow(h)`; `DepositCount < 2^31 - 1`. Effect: subtract
`Amount + fee`; add `Amount` to `Escrow`; append record `DepositCount` with
`Height = h`; increment `DepositCount`.

**Reclaim.** The record exists, `Status == 0`, `From` equals the record `From`, and
the deposit has expired in `epoch(h)` (section 2.4). Effect: set `Status = 1`;
subtract `Amount` from `Escrow`; credit `Amount` to the record `From`.

A reclaim cannot race with an anchor: an anchor of epoch `e` credits only deposits creditable in `e`, a reclaim in `e` needs a deposit that has expired in `e`, and the two sets are disjoint. Operators SHOULD consume a deposit in the epoch in which it is made.

#### 3.3 `AssetAnchor` (type 10)

An anchor proposes a new head. It is **pending** for `ChallengeWindow` blocks, during which
anyone who holds the batch data can prove that it is wrong (section 3.7). If nobody does, the
head becomes final and the effects of the anchor on Pactus (payouts, refund, fee share,
deposit credit, eviction) are applied (section 3.7, Finalization).

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Submitter. Any account. Pays the Pactus fee, has no authority |
| `AttestationType` | `uint8` | 1 | `0` = committee, with fraud proofs. `1` is reserved for validity proofs, which a later PIP would define. Other values invalid |
| `Epoch` | `uint32` | 4 | |
| `Seq` | `uint64` | 8 | |
| `PrevHead` | `Hash` | 32 | |
| `StateRoot` | `Hash` | 32 | Root after the last step of the batch |
| `DataRoot` | `Hash` | 32 | `ListRoot` of the header and the steps (section 5.6) |
| `TraceRoot` | `Hash` | 32 | `ListRoot` of the state roots before and after each step (section 5.6) |
| `DataSize` | `uint32` | 4 | Bytes of the canonical batch |
| `OpCount` | `uint32` | 4 | Number of operation steps in the batch |
| `NewCursor` | `uint64` | 8 | |
| `DepositDigest` | `Hash` | 32 | |
| `PayoutCount` | `uint8` | 1 | `<= MaxPayoutsPerAnchor`. The number of withdrawal operations of the batch |
| `Payouts` | `PayoutCount` × 33 | | `StepIndex` (`uint32`, 4), `To` (`Address`, 21), `Amount` (`int64`, 8). One for each `REQUEST_WITHDRAWAL` operation, in order |
| `FeeShare` | `int64` | 8 | Nano PAC paid in equal parts to the operators (rule 10). `0` for none |
| `Refund` | `int64` | 8 | Nano PAC paid to `From`, the submitter, to cover its Pactus fee. `0 <= Refund <= AnchorRefund` |
| `EvictIndex` | `uint32` | 4 | `Index` of an operator that the committee evicts, or `0xFFFFFFFF` for none (rule 11) |
| `BitmapLen` | `uint8` | 1 | `ceil(n / 8)` |
| `Signers` | `BitmapLen` | | Bit `i` (LSB first in each byte) is the operator at position `i` of `Committee(Epoch)` |
| `Signature` | 48 | | Aggregate BLS signature over `AttestMessage` |

Maximum payload: `280 + 3 + 33 × 64 = 2395` bytes. `Value()` is `0`.

`BasicCheck`: `From` is an account address; `AttestationType == 0`;
`1 <= DataSize <= MaxBatchBytes`; `OpCount <= MaxSteps`;
`PayoutCount <= MaxPayoutsPerAnchor`; every payout `Amount > 0` and `To` is an
account address (types 2, 3 or 4); the `StepIndex` values strictly increase; `0 <= FeeShare <= MaxNanoPAC`;
`0 <= Refund <= AnchorRefund`; the sum of payout amounts, `FeeShare` and `Refund`
does not overflow; `BitmapLen <= 8`; decoding reads exactly the declared fields.

`Check` (stateful). Each rule below MUST hold, in this order. `Tip` is the latest anchor,
pending or final; `Head` is the latest final one.

1. `Frozen == 0` and `Epoch == epoch(h)`.
2. Let `C = Committee(Epoch)` and `n = |C|`. `n >= MinCommittee`.
3. `BitmapLen == ceil(n / 8)`, no bit at position `>= n` is set, and the number
   of set bits `s` satisfies `s >= T`, with `T` as in section 1.2.
4. `LastAnchorHeight < h` (one anchor or resume per block).
5. `Seq == TipSeq + 1`, `PrevHead == TipHash` and `TipSeq - HeadSeq < MaxPending`.
6. `DepositCursor <= NewCursor <= DepositCount`,
   `NewCursor - DepositCursor <= MaxDepositsPerAnchor`, and if
   `NewCursor > DepositCursor` the record `NewCursor - 1` has `Height < h`.
   `DepositCursor` is the cursor of the tip. Records are appended in block order, so this
   makes every consumed deposit strictly older than the including block.
7. Let `D` be the records in `[DepositCursor, NewCursor)` that are creditable in
   `Epoch`, and `A` the sum of their amounts.
   `DepositDigest == DepositDigest(DepositCursor, NewCursor, D)`. The number of steps of
   the batch is `N = |D| + OpCount + 1`, and `N <= MaxSteps`. Every payout has
   `|D| < StepIndex <= |D| + OpCount`, and `PayoutCount <= OpCount`.
8. With `Pay` the sum of the payout amounts:
   `Paid + PendingOut + Pay + FeeShare + Refund <= Consumed + PendingConsumed + A`.
9. Let `Pk` be the aggregate of the public keys of the signers. `Pk` MUST NOT be the
   identity. `Signature` is valid for `AttestMessage` under `Pk`.
10. If `FeeShare > 0`: let `M` be the records of `C` that are not slashed and
    `m = |M|`. Then `m >= 1`, `TipFeeShareEpoch != Epoch`, `FeeShare mod m == 0` and
    `FeeShare / m >= 1`.
11. If `EvictIndex != 0xFFFFFFFF`: a seated record has that `Index` and
    `ActiveFromEpoch <= Epoch`, and after the eviction at least `MinCommittee`
    records remain seated. The committee cannot evict itself below the size at which
    it can anchor.

`Execute` (submission). Pactus records the anchor, it does not yet pay anything:

* Create the **anchor record**: all fields of the payload, `SignerIndexes` (the `Index` of every operator whose bit is set, decoded against `Committee(Epoch)` as it is in this block), an empty list of open requests, the submission height `h`, the
  submitter `From`, `OldCursor = DepositCursor`, `PrevFeeShareEpoch = TipFeeShareEpoch`,
  `A`, `Pay`, and `RefundPaid = min(Refund, fee of this anchor transaction)`. Its leaf in
  `anchorMerkle`, at index `Seq`, is `Hash(0x50 || record)`.
* Set `TipSeq = Seq`, `TipHash = HeadHash` of the anchor (section 2.5), `TipStateRoot = StateRoot`.
* Set `DepositCursor = NewCursor` and `Status = 3` (reserved) on every record of `D`, so that
  no one can reclaim a deposit that this anchor credits.
* `PendingConsumed += A`; `PendingOut += Pay + FeeShare + RefundPaid`; if `FeeShare > 0`,
  `TipFeeShareEpoch = Epoch`; `LastAnchorHeight = h`.

Eviction is the committee's decision, not the submitter's: the submitter chooses which signatures go into the
bitmap, so a measure of inactivity computed from it could be aimed at one operator. `T` operators can therefore
evict the others, which adds no power beyond section 1.2.

The refund is a fixed amount that the committee attests, because the submitter chooses its fee after the
attestation. Any account that gets a valid anchor into a block receives `RefundPaid`, so a silent operator cannot
keep an anchor from being relayed. The cap by the transaction fee stops a proposer from including its own anchor
at zero fee and keeping the refund; the difference stays in escrow as slack, as does the remainder of a fee share
that does not divide by the number of members. The fee share and the refund are the only ways fee revenue leaves
the Asset Layer.

Verification cost per anchor is one aggregate signature check, at most `MaxCommittee` public-key additions, at
most 1 024 deposit records and one record appended, whatever the number of operations.

#### 3.4 `AssetEvidence` (type 11)

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Reporter. Any account |
| `OperatorIndex` | `uint32` | 4 | |
| `Epoch` | `uint32` | 4 | |
| `Seq` | `uint64` | 8 | |
| `PrevHead` | `Hash` | 32 | |
| `ContentA`, `SigA` | `Hash`, 48 | 80 | First attestation |
| `ContentB`, `SigB` | `Hash`, 48 | 80 | Second attestation |

The payload is 229 bytes. `Value()` is `0`. An attestation made for one epoch cannot
be anchored in another (rule 1 of section 3.3 and the `Epoch` field of the signed
message), so only two attestations of the **same epoch** can conflict; that is the
only case this payload punishes. `Check`: the record exists and is not slashed;
`ContentA != ContentB`; the operator is in `Committee(Epoch)` (using the record's
`ActiveFromEpoch` and `ExitEpoch`); `SigA` verifies under the operator key over
`"PAC-ASSET-ATTEST-1" || ChainID || Epoch || Seq || PrevHead || ContentA`, and
`SigB` likewise over `ContentB`.

`Execute`: set `Flags |= 1`; `ExitEpoch = epoch(h) + 1` if the record has none;
`WithdrawableHeight = ExitEpoch * EpochLength`; `BondedTotal -= Bond`;
credit `floor(Bond / 2)` to `From` and the rest to the treasury account; set
`Bond = 0`. Evidence is admissible while the record still exists, which is at
least `OperatorUnbondDelay` blocks after the operator leaves the committee. It
remains admissible while the layer is frozen or resuming.

#### 3.5 Halt, freeze, exit claims and receipts

If the committee stops anchoring, fee credit would be stranded in escrow and holders of
native assets would depend on a committee that is gone. After `HaltTimeout` blocks
without an anchor or a resume, anyone can freeze the layer. While it is frozen, holders
take back fee credit from Pactus and record their native positions, with Merkle proofs
against the last **final** head, and the committee can resume the layer (section 3.6).

**Freeze** (`AssetEscrow`, action `2`). `Check`: `Frozen` is `0` or `2`, `HeadSeq >= 1`,
`TipSeq == HeadSeq` (nothing is pending) and `h - LastAnchorHeight >= HaltTimeout`.
`Execute`: `Frozen = 1`. A freeze from state `2` (a resume that stalled) reopens PAC claims;
the claims already applied by that resume stay applied.

| `Frozen` | Name | Anchor | PAC claims (kind 0) | Receipts (kinds 1, 2, 3) | Deposit |
| --- | --- | --- | --- | --- | --- |
| `0` | active | accepted | rejected | rejected | accepted |
| `1` | frozen | only `AssetResume` | accepted | accepted | rejected |
| `2` | resuming | only `AssetResume` | rejected | accepted | rejected |

In states `1` and `2`, Bond, Seat and Prune are rejected. Reclaim of an expired pending
deposit, Unbond, Withdraw, Evidence, `AssetFraud`, `AssetOpen` and every ordinary Pactus
transaction work in every state.

**ExitClaim** (`AssetEscrow`, action `3`). It proves one entry of the Asset Layer state
tree against the `StateRoot` of the final head. There are four kinds:

| `Kind` | What it proves | Selector fields | `Value` |
| --- | --- | --- | --- |
| `0` | Fee credit of `From` | none | 16 bytes: `uint64 Nonce \|\| uint64 Credit` |
| `1` | The holding of `From` | `AssetID` (32), `TokenID` `uint64` | 8 bytes: `uint64` quantity |
| `2` | An asset record | `AssetID` (32) | `ValueLen` `uint8` (1 to 225), then the `AssetRecord` bytes |
| `3` | An item record | `AssetID` (32), `TokenID` `uint64` | 49 bytes: `ItemRecord` |

The payload is `From` (21), `Action` (1), `Kind` (1), the selector fields, `Value`,
`ProofBitmap` (32, section 5.2.1) and `Siblings` (32 × popcount of the bitmap, at most
256). The largest payload is 8 505 bytes, a kind 2 with the largest `AssetRecord`. A
block MUST NOT contain more than `MaxExitClaimsPerBlock` ExitClaim transactions of any
kind. Pactus gossip messages are limited to 1 MB; with these caps a block of the new payload types stays near 0.92 MB (section 6).

`BasicCheck`: `Kind` is in `0..3`; `Value` has the length of its kind; for kinds `0` and
`1`, `From` is a type-4 address.

`Check`:

1. Kind `0` needs `Frozen == 1`. Kinds `1`, `2` and `3` need `Frozen` to be `1` or `2`.
2. The state-tree key is: `Hash("PAC-ASSET-K/ACCT" || From)` for kind `0`;
   `Hash("PAC-ASSET-K/HOLD" || AssetID || TokenID || From)` for kind `1`;
   `Hash("PAC-ASSET-K/ASSET" || AssetID)` for kind `2`;
   `Hash("PAC-ASSET-K/ITEM" || AssetID || TokenID)` for kind `3`.
3. The proof verifies `Value` at that key against the `StateRoot` of the final head.
4. Kind `0`: `Amount` is the `Credit` of `Value`, and `1 <= Amount <= MaxNanoPAC`. Kind `1`:
   the quantity is at least `1`. Kinds `2` and `3`: `From` is any account.
5. No PAC claim (kind `0`) that no resume has applied yet (leaf index `ExitCursor` or above), and no receipt (kinds `1`, `2`, `3`) recorded at a height of `ExportsVoidBefore` or above, has the same key. A claim that a resume applied has had its credit removed from the Asset Layer state, and a receipt below `ExportsVoidBefore` is void, so neither may block a later freeze: the account may hold new credit by then.
6. Kind `0`: `Paid + Amount <= Consumed`.

`Execute` for kind `0`: append the leaf `Hash(0x00 || Key || int64(Amount) || uint32(h))` to
`exitMerkle` at index `ExitCount` and increment `ExitCount`; `Paid += Amount`;
`Escrow -= Amount`; credit `Amount` to `From` (creating the account if needed). `Execute` for
kinds `1`, `2` and `3`: append the leaf `Hash(uint8(Kind) || Key || Hash(Value) || uint32(h))`
to `exportMerkle` at index `ExportCount` and increment `ExportCount`. A node MUST keep the
key, amount and height of every PAC claim, and the kind, key and value of every receipt, so
that it can reject a second claim and answer a resume.

**Receipts.** A receipt records one fact of the final head: that an address held a quantity, that an asset has a
definition, or that an item exists. It has no spending power on Pactus; it lets ownership after a halt rest on a
proof against a root that Pactus holds. A holding is recorded only by its owner, a definition or an item by
anyone. A receipt is void if the layer resumes: `ExportsVoidBefore` is set to the height of the resume, and a later
freeze produces valid receipts again. The positions never leave the Asset Layer state.

Claims and receipts are proven against the final head, never against a pending one, so a head
that is still open to challenge cannot be used to take PAC out. The fee pool, which belongs
to the operators, is not claimable and stays in escrow. If the final head overstated credits,
PAC claims are paid in order of inclusion until `Consumed - Paid` is exhausted. A receipt is
only as true as the head it is proven against: if that head is false, so are the receipts
(section 1.2).

#### 3.6 `AssetResume` (type 12)

A frozen layer can be resumed by the committee. A resume is an attested step that applies
to the Asset Layer state the PAC claims that holders have already been paid on Pactus. It
can take several steps. Like an anchor, a resume is pending for `ChallengeWindow` blocks and
can be challenged (section 3.7).

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Submitter. Any account |
| `AttestationType` | `uint8` | 1 | `0` |
| `Epoch` | `uint32` | 4 | |
| `Seq` | `uint64` | 8 | |
| `PrevHead` | `Hash` | 32 | |
| `StateRoot` | `Hash` | 32 | |
| `DataRoot` | `Hash` | 32 | |
| `TraceRoot` | `Hash` | 32 | |
| `DataSize` | `uint32` | 4 | |
| `NewExitCursor` | `uint64` | 8 | |
| `ExitDigest` | `Hash` | 32 | |
| `BitmapLen` | `uint8` | 1 | `ceil(n / 8)` |
| `Signers` | `BitmapLen` | | As in an anchor |
| `Signature` | 48 | | Over `AttestMessage` with `ResumeContentHash` |

The payload is at most 258 bytes. `Value()` is `0`. `BasicCheck`: `From` is an account
address; `AttestationType == 0`; `1 <= DataSize <= MaxBatchBytes`; `BitmapLen <= 8`.

`Check`:

1. `Frozen` is `1` or `2`, and `Epoch == epoch(h)`.
2. Let `C = Committee(Epoch)` and `n = |C|`. `n >= MinCommittee`, and the bitmap and the
   threshold are as in rule 3 of section 3.3.
3. `LastAnchorHeight < h`, and nothing is pending: `TipSeq == HeadSeq`.
4. `Seq == HeadSeq + 1` and `PrevHead == HeadHash`.
5. `ExitCursor <= NewExitCursor <= ExitCount` and
   `NewExitCursor - ExitCursor <= MaxExitsPerResume`. If `Frozen == 1`, also
   `NewExitCursor <= ExitCountBefore(Epoch)`, the number of PAC claims made in an epoch
   before `Epoch`. The number of steps of the batch is `N = NewExitCursor - ExitCursor`.
6. `ExitDigest == ExitDigest(ExitCursor, NewExitCursor, E)`, where `E` is the PAC claims
   in `[ExitCursor, NewExitCursor)`.
7. The aggregate of the signers' keys is not the identity, and `Signature` is valid for
   `AttestMessage` with `ResumeContentHash`.

`Execute` (submission): create the resume record, with `SignerIndexes` as an anchor does (its leaf in `anchorMerkle` is
`Hash(0x50 || record)`), set `TipSeq = Seq`, `TipHash` and `TipStateRoot`, and
`LastAnchorHeight = h`. Nothing else changes until finalization.

At finalization (section 3.7) of a resume: `HeadSeq`, `HeadHash`, `StateRoot` and `DataRoot`
take its values; `ExitCursor = NewExitCursor`; if `Frozen == 1`, set `Frozen = 2`; then if
`ExitCursor == ExitCount`, set `Frozen = 0` and `ExportsVoidBefore` to the height of that
block.

Rule 5 keeps a resume free of races: the first resume covers only claims of earlier epochs, which no later
transaction can change, and it moves the layer to state `2`, which closes PAC claims so that the set to apply
stops growing; later steps apply the rest. Claims made while a resume is pending are not covered by it. If the
committee stops again in state `2`, anyone can freeze again after `HaltTimeout`, which reopens claims. A resume
needs `T` operators; if more than `n - T` are gone the layer stays frozen, and the exits and receipts of
section 3.5 are what protects users.

#### 3.7 Challenge window, finalization and fraud proofs (type 13)

v1 does not make Pactus check that a head is correct. It lets anyone who holds the data
prove, on one step, that it is not, and it makes the signers pay.

**Finalization.** At the start of every block of version `V` or higher, before its
transactions, the node finalizes pending records in order of `Seq`, at most `MaxFinalizePerBlock` of them: while `HeadSeq < TipSeq`
and the record of `HeadSeq + 1` is pending, has no open request (section 3.8) and has
`SubmitHeight + ChallengeWindow <= h`.
Finalizing an anchor:

* `HeadSeq`, `HeadHash`, `StateRoot` and `DataRoot` take the values of the record.
* `Status` of the records of its `D` changes from `3` to `2`; `Consumed += A`;
  `PendingConsumed -= A`.
* Each payout `To` is credited (creating the account if needed), and `RefundPaid` is credited
  to the submitter.
* If `FeeShare > 0`, let `m'` be the number of records of `Committee(Epoch)` that are not
  slashed at that moment. If `m' >= 1`, each of their owners is credited `FeeShare / m'`,
  rounded down; what is not paid stays in escrow as slack.
* `Paid` and `Escrow` change by the amounts actually paid, and `PendingOut -= Pay + FeeShare
  + RefundPaid`.
* If `EvictIndex` is set and rule 11 of section 3.3 still holds, that record gets
  `ExitEpoch = epoch(h) + 1` and `WithdrawableHeight = ExitEpoch * EpochLength +
  OperatorUnbondDelay`, without slashing. Otherwise the eviction is skipped.
* Its leaf in `anchorMerkle` becomes `Hash(0x46 || HeadHash)`.

Finalizing a resume is described in section 3.6. At most one record is submitted per block, but records held back
by an open request (section 3.8) can become due together; `MaxFinalizePerBlock` bounds that burst, and a backlog
drains at `MaxFinalizePerBlock - 1` records per block without changing any record's window.

**`AssetFraud`.**

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Reporter. Any account |
| `Seq` | `uint64` | 8 | The pending record that is challenged |
| `Kind` | `uint8` | 1 | `0` Step, `1` FinalRoot, `2` Header, `3` Start |
| `StepIndex` | `uint32` | 4 | Kind `0`: `1..N`. Other kinds: `0` |
| `Body` | | | By kind, below |

The payload is at most `MaxFraudProofBytes`, and a block holds at most
`MaxFraudProofsPerBlock` of them. `Value()` is `0`. A `Proof` is a length `uint8` followed by
that many 32-byte siblings of a `ListRoot` audit path (section 5.6). A `Witness` is a `Key`
(32), a `ValueLen` `uint16` (`0xFFFF` for an absent entry) and the `Value`, then a
`ProofBitmap` (32) and the `Siblings` of a membership proof (section 5.2.1).

| Kind | Body |
| --- | --- |
| `0` Step | `StepLen` `uint16`, `StepData`, `Proof` against `DataRoot`; `HeaderData` (70 bytes) and its `Proof` against `DataRoot`, present only if the step is an operation; `PreRoot` (32) and its `Proof` against `TraceRoot`; `PostRoot` (32) and its `Proof` against `TraceRoot`; `WitnessCount` `uint8` (at most 8) and the `Witness`es |
| `1` FinalRoot | `FinalRoot` (32) and its `Proof` against `TraceRoot` |
| `2` Header | `HeaderData` (70 bytes) and its `Proof` against `DataRoot` |
| `3` Start | `StartRoot` (32) and its `Proof` against `TraceRoot` |

`Check`:

1. The record `Seq` is pending (`HeadSeq < Seq <= TipSeq`, not reverted) and
   `h < SubmitHeight + ChallengeWindow`.
2. Let `N` be the number of steps of the record (section 3.3 rule 7 or section 3.6 rule 5) and
   `n = N + 1`, the number of leaves of `DataRoot` and of `TraceRoot`. The leaf of the
   header and of `R_0` has index `0`, the leaf of step `s` and of `R_s` has index `s`.
3. Every audit path and membership proof in the payload verifies, with `PreRoot` at index
   `StepIndex - 1` and `PostRoot` at index `StepIndex` of `TraceRoot`, and every witness
   verifies against `PreRoot`. A payload with a proof that does not verify, or without a
   witness that the step needs, is invalid and has no effect.
4. The payload shows fraud:
   * Kind `0`: the step at position `StepIndex` is of the kind that its position requires
     (`StepIndex <= |D|` credit; the next `OpCount` operation; the last one pool; for a resume,
     all are exit steps), **and** at least one of these holds. (a) `StepData` does not decode
     as that kind. (b) For a credit step, it differs from the `StepIndex`-th record of `D`; for
     a pool step, it differs from the `FeeShare` and `Refund` of the record; for an exit step,
     it differs from the PAC claim of the same rank; for an operation step, it is a
     `REQUEST_WITHDRAWAL` and the payouts of the record have no entry with this `StepIndex`
     and the same `To` and `Amount`, or it is not a `REQUEST_WITHDRAWAL` and they have an
     entry with this `StepIndex`. (c)
     Executing the step from the witnesses (section 5.9) breaks a rule of section 5.5, or of
     the step. (d) Executing the step gives a root different from `PostRoot`.
   * Kind `1`: `FinalRoot` differs from `StateRoot` of the record.
   * Kind `2`: the header is wrong: version other than `1`, a batch kind that does not match
     the record, `Epoch` or `Seq` other than those of the record, `PrevStateRoot` other than the
     `StateRoot` of the record before it (final or pending), `RefHeight` above `SubmitHeight` or
     more than `MaxRefLag` blocks before the first block of `Epoch` (for a resume: other than `0`),
     `CreditCount` other than `|D|`, `OpCount` other than that of the record, `Reserved` other
     than `0`, or `NewCursor` other than that of the record. For a resume, `|D|` and `OpCount`
     are `0`, and `NewCursor` is its `NewExitCursor`.
   * Kind `3`: `StartRoot` differs from `PrevStateRoot` of the record.

`Execute` (also run by an `AssetOpen` timeout, section 3.8):

* **Revert.** Records `Seq` to `TipSeq` become reverted (leaf `Hash(0x52 || HeadHash)`).
  For each reverted anchor, `Status` of its `D` goes from `3` back to `0`;
  `PendingConsumed` and `PendingOut` are reduced by its sums. If record `Seq` is an anchor,
  `DepositCursor = OldCursor` and `TipFeeShareEpoch = PrevFeeShareEpoch` of that record (a
  resume changes neither). `TipSeq = Seq - 1`, `TipHash` and `TipStateRoot` take the values of
  the record before it (or of the final head), and `LastAnchorHeight = h`. The bond of every
  open request (section 3.8) of a reverted record is returned to its requester and leaves
  `OpenBonded`. A reverted record pays nothing.
* **Slash.** For each operator in the `SignerIndexes` of record `Seq` (never the bitmap, which can no longer be decoded once a record of an old committee has been deleted) whose operator record still exists and is not
  slashed: `Flags |= 1`, `ExitEpoch = epoch(h) + 1` if it has none,
  `WithdrawableHeight = ExitEpoch * EpochLength`, `BondedTotal -= Bond`, and the bond is
  collected. Of the total `S` collected, `floor(S / 2)` is credited to `From` and the rest to
  the treasury account. Signers of later records that were reverted with it are not slashed:
  they signed a head that is no worse than the one they built on.

`OperatorUnbondDelay` is longer than `ChallengeWindow`, so an operator cannot leave before it
can be slashed for a pending anchor.

**Roots that cannot be opened.** A committee could commit to a root that is not a real tree, so that no path to
the wrong step exists. Section 3.8 lets anyone demand that a leaf be opened on Pactus, and treats a root that
nobody can open as fraud.

**What this does not guarantee.** A proof needs the batch data. If the committee withholds it from everyone,
nobody can prove fraud and a false head becomes final (section 1.2, Security Considerations). What changes is that
the committee must hide the batch from every other party for a day.

#### 3.8 `AssetOpen` (type 14)

A fraud proof opens leaves of `DataRoot` and `TraceRoot`, which works only if they are roots of real trees.
`AssetOpen` lets anyone ask for a leaf to be opened on Pactus; if nobody can open it, the anchor is treated as
proven wrong.

| Field | Type | Size | Notes |
| --- | --- | --- | --- |
| `From` | `Address` | 21 | Requester, responder or reporter. Any account |
| `Action` | `uint8` | 1 | `0` Request, `1` Respond, `2` Timeout |
| `Seq` | `uint64` | 8 | The record |
| `Tree` | `uint8` | 1 | `0` `DataRoot`, `1` `TraceRoot` |
| `Index` | `uint32` | 4 | A leaf index, `0..N` |

Action `1` adds `LeafLen` `uint16`, `Leaf` and a `Proof` (a length `uint8` and that many 32-byte
siblings of a `ListRoot` audit path). `Value()` is `OpenBond` for a Request and `0` otherwise.

**Request.** `Check`: the record is pending and `h < SubmitHeight + ChallengeWindow`; `Index <= N`;
no open request of the record has the same `Tree` and `Index`; the record has fewer than
`MaxOpenRequests` open requests; `Balance >= OpenBond + fee`. `Execute`: take `OpenBond` from the
requester, add it to `OpenBonded`, and add the request `(Tree, Index, From, h)` to the record.

**Respond.** `Check`: an open request with this `Tree` and `Index` exists, and `Leaf` is the
leaf at `Index` of the tree under the root of the record (`DataRoot` or `TraceRoot`, with
`n = N + 1` leaves). `Execute`: remove the request; credit `OpenBond` to `From`; `OpenBonded`
decreases by the same amount. The leaf is now public on Pactus, which is how a challenger who
did not have the data learns what the committee committed to.

**Timeout.** `Check`: an open request exists and `h >= RequestHeight + OpenWindow`. `Execute`:
the effects of a proven fraud (section 3.7, Revert and Slash), the requester gets its
`OpenBond` back, and `From`, who may be anyone, is the reporter that receives half of the bond
slashed.

While a record has an open request it is not finalized and the records after it wait, so a request delays
finalization by at most `OpenWindow` blocks beyond the challenge window. Unanswered, it costs the signers their
bonds; answered, it costs the requester `OpenBond`, which goes to the responder. The mechanism does not catch a
committee that opens every leaf that is asked for and nothing else, so it does not remove the need for public
data.

### 4. Pactus node changes

* `Sandbox` exposes `Settlement()` (read and write), deposit records, anchor records, the exit
  and export trees, `ChainID()` and the epoch. Pactus nodes implement a larger part of section 5
  than before: the proof check of section 5.2.1, the audit paths of section 5.6, and the
  execution of a single step of section 5.9 with the rules of sections 5.5 and 5.8 (including
  secp256k1 signature verification). That code runs only to check an `AssetFraud`. The existing
  `Check` / `Execute` split applies to every payload. `executeBlock` changes in one place: before
  the transactions of a block of version `V` or higher it finalizes the pending record that is
  due (section 3.7). That step is a function of the settlement state and the height only, so it
  is the same on every node, and it runs on the commit path as well as on the validation path.
* Transaction pool: each new type gets its own pool of 5 % of `MaxSize`. A pool MUST
  run every check that needs no pairing before it runs one (for an anchor: `Epoch`,
  `Seq`, `PrevHead`, bitmap length and number of set bits; for evidence: the operator
  exists and is not slashed, and the two contents differ; for a fraud proof: the record is
  pending and the payload is within its size limit and its proofs are well formed), and SHOULD
  limit the number of unverified anchors, evidence, bonds and fraud proofs it accepts per
  peer. Anchors are deduplicated by `(PrevHead, ContentHash, Signers, Signature)`, so a node
  pays for one pairing per distinct candidate and an invalid copy of a valid anchor cannot
  hide it.
* Verification budget of a block. The work that needs a pairing is bounded by state: at most one anchor or resume
  (one aggregate verification), at most one evidence per operator that is not yet slashed (two checks each, at
  most 42 records) and at most one check per free registry record for bonds, about 85 signature verifications in
  the worst case. An `ExitClaim` costs at most 256 hashes and 32 fit. An `AssetFraud` costs at most 8 membership
  proofs and one root reconstruction (about 4 000 hashes), three audit paths of 20 hashes and one secp256k1
  verification, and `MaxFraudProofsPerBlock` fit. An `AssetOpen` costs one audit path. Finalization costs at most
  `MaxFinalizePerBlock` anchors per block.
* Block production: a proposer SHOULD place anchors after other transactions. A
  proposer MUST NOT include more than `MaxExitClaimsPerBlock` ExitClaim transactions or more
  than `MaxFraudProofsPerBlock` AssetFraud transactions, and `ValidateBlock` MUST reject a
  block that does. The counts are per-block counters held by the sandbox, not settlement
  state. A proposer SHOULD give `AssetFraud` priority over anchors, because a fraud proof has a
  deadline.
* RPC (mandatory for the official node, on every surface it ships): payload type values `8` to `14` in
  `PayloadType`; `GetRaw*Transaction` builders for the seven payloads; `GetAssetSettlement`, returning the fields
  of the settlement state (section 2.2), the committee of the current epoch, `max_escrow` for the current height
  and the pending sums; `GetAssetDeposit(seq)`; `GetAssetAnchor(seq)` (the record of an anchor or resume, its
  status and the time left in its window); and `ListAssetOperators`, returning the fields of section 2.3.
* Explorers that audit supply MUST add `Escrow`, `BondedTotal` and `OpenBonded` to account
  balances and validator stake. Omitting them looks like a burn.

### 5. The Asset Layer

This part is normative for operators. Pactus validators do not execute batches. They execute
one step of it, and only to check a fraud proof (sections 3.7 and 5.9). All operators MUST
compute identical roots from the same batch, because the attestation of section 2.5 binds that
root, and they MUST agree with that verifier.

#### 5.1 Accounts and signatures

Every sender, recipient, issuer, controller and fixed destination is a type-4
secp256k1 address, except the withdrawal beneficiary of `REQUEST_WITHDRAWAL`,
which is any Pactus account address (types 2, 3 or 4). The zero address means "no
authority" and MUST NOT hold or receive value.

An operation carries its 33-byte compressed public key and 64-byte `r || s`
signature, as a Pactus transaction does. The signature is verified over
`"PAC-ASSET-OP-1" || ChainID || SignedBytes` with the secp256k1 scheme of PIP-53.
Operations are identified by `Hash` of `SignedBytes`, never by signature bytes,
since the scheme does not force a low `s`.

#### 5.2 State tree

State is a sparse Merkle tree with 256 levels and 32-byte keys.

```text
Empty[0]  = 32 zero bytes
Empty[k]  = Hash(0x01 || Empty[k-1] || Empty[k-1])      k = 1..256
EmptyStateRoot = Empty[256]

leaf node   = Hash(0x00 || key || Hash(value))                 (level 0)
inner node  = Hash(0x01 || left || right)
```

At level `j` (counting up from the leaf) bit `255 - j` of the key, most
significant bit of byte 0 first, selects the branch: `0` left, `1` right. An
absent key is `Empty[0]` at level 0. A value of zero is never stored: a holding
of zero, an account record with nonce and credit zero, and a zero fee pool are
removed. A stored value is therefore never all zero bytes. Item records are never removed.

Keys are `Hash(tag || fields)`:

| Entry | Key | Value |
| --- | --- | --- |
| Asset | `"PAC-ASSET-K/ASSET" \|\| AssetID` | `AssetRecord` |
| Item | `"PAC-ASSET-K/ITEM" \|\| AssetID \|\| uint64(TokenID)` | `ItemRecord` |
| Holding | `"PAC-ASSET-K/HOLD" \|\| AssetID \|\| uint64(TokenID) \|\| Address` | `uint64` quantity |
| Account | `"PAC-ASSET-K/ACCT" \|\| Address` | `uint64 Nonce \|\| uint64 Credit` |
| Fee pool | `"PAC-ASSET-K/POOL"` | `uint64` |

Because keys are hashes, a deployment may split the tree across machines by key
prefix. The root does not depend on how it is split.

##### 5.2.1 Membership proofs

A proof that `value` is stored at `key` under `root`, or that `key` is absent, is a 32-byte
`Bitmap` and a list of 32-byte `Siblings`. Bit `j` of the bitmap (byte `j / 8`, least significant
bit first) is set when the sibling at level `j` is explicit; the explicit siblings
are listed in ascending level. All other siblings are `Empty[j]`. To verify:

```text
node = Empty[0] if the entry is absent, else Hash(0x00 || key || Hash(value))
for j in 0..255:
    sibling = next explicit sibling if bit j of Bitmap is set, else Empty[j]
    if bit (255 - j) of key (most significant bit of byte 0 is bit 0) is 0:
        node = Hash(0x01 || node || sibling)
    else:
        node = Hash(0x01 || sibling || node)
accept iff node == root and every explicit sibling was used
```

Pactus nodes run this check for `ExitClaim` (section 3.5) and for the witnesses of an
`AssetFraud` (section 3.7). The cost is at most 256 hashes of 65 bytes.

#### 5.3 Records

`AssetID = Hash("PAC-ASSET-ID-1" || Creator || uint64(CreatorNonce))`, where the
nonce is the nonce of the creating operation.

`AssetRecord`, in this order:
`AssetType uint8`, `MintMode uint8`, `BurnMode uint8`, `Decimals uint8`,
`SymbolLen uint8`, `Symbol`, `MetaHash` (32), `MaxSupply uint64`, `Creator` (21),
`MintController` (21), `ParamController` (21), `Supply uint64`,
`PolicyPresent uint8`, then if present `TeamAddress` (21) and three parameters
(`TeamRate`, `BurnRate`, `TeamCap`), each `Current uint64`, `Min uint64`,
`Max uint64`, `Mutable uint8`.

`ItemRecord`: `MetaHash` (32), `ItemMaxSupply uint64`, `ItemSupply uint64`,
`Burned uint8`.

`AssetType` values: `1` FUNGIBLE, `2` NON_FUNGIBLE, `3` SEMI_FUNGIBLE. These are
the only asset types of v1. Fixed supply, mintable, burnable, taxed and
governance labels are configurations and MUST NOT become types.

#### 5.4 Operation envelope

```text
Version u8 (=1) | Type u8 | Sender (21) | Nonce u64 | Fee u64 | Expiry u32 | Body
SignedBytes = everything above
Operation   = SignedBytes | PublicKey (33) | Signature (64)
```

Types: `1` CREATE_ASSET, `2` TRANSFER, `3` MINT, `4` BURN, `5` UPDATE_PARAMETER,
`6` REQUEST_WITHDRAWAL.

Bodies:

| Type | Body |
| --- | --- |
| CREATE_ASSET | `AssetType u8`, `MintMode u8` (0 none, 1 controller), `BurnMode u8` (0 none, 1 holder), `Decimals u8`, `SymbolLen u8`, `Symbol`, `MetaHash` (32), `MaxSupply u64`, `MintController` (21), `ParamController` (21), `InitialHolder` (21), `InitialSupply u64`, `PolicyPresent u8`, then if present `TeamAddress` (21) and the three parameters as in `AssetRecord` |
| TRANSFER | `AssetID` (32), `TokenID u64`, `Recipient` (21), `Amount u64`, `MinReceive u64` |
| MINT | `AssetID` (32), `TokenID u64`, `Recipient` (21), `Amount u64`, `MetaHash` (32), `ItemMaxSupply u64` |
| BURN | `AssetID` (32), `TokenID u64`, `Amount u64` |
| UPDATE_PARAMETER | `AssetID` (32), `ParamID u8` (0 `TeamRate`, 1 `BurnRate`, 2 `TeamCap`), `NewValue u64` |
| REQUEST_WITHDRAWAL | `Beneficiary` (21), `Amount u64` |

A `FUNGIBLE` asset uses token ID zero everywhere. `NON_FUNGIBLE` and
`SEMI_FUNGIBLE` use nonzero token IDs and zero decimals.

#### 5.5 Execution of one operation

Every operation, in this order:

1. `Version == 1`, `Sender` is a type-4 address and not zero, `PublicKey` derives
   `Sender`, the signature is valid.
2. `Nonce` equals the sender's account `Nonce`.
3. The batch `RefHeight <= Expiry`.
4. The sender's `Credit >= Fee`. Subtract `Fee` from `Credit` and add it to the
   fee pool.
5. Execute the body below.
6. `Fee` is at least the required fee of section 5.8, computed from what step 5
   created.
7. Increment the sender's `Nonce`, which MUST NOT overflow `uint64`.

An operation that fails any step makes the whole batch invalid: a producer drops it before proposing, and
operators refuse to sign a batch that contains one. A batch that does contain one is wrong in a way that a single
step shows (section 3.7).

**CREATE_ASSET.**

* `AssetType` in `{1, 2, 3}`. `Symbol` is 1 to 12 bytes of `A-Z0-9` and is not
  `PAC`.
* `MintMode` and `BurnMode` are `0` or `1`. `MintController` is the zero address
  iff `MintMode == 0`, otherwise a type-4 address.
* `MaxSupply >= 1`.
* FUNGIBLE: `Decimals <= 18`. `InitialSupply <= MaxSupply`. `InitialHolder` is a
  nonzero type-4 address iff `InitialSupply > 0`, else the zero address. If
  `MintMode == 0` then `MaxSupply == InitialSupply`.
* NON_FUNGIBLE and SEMI_FUNGIBLE: `Decimals == 0`, `InitialSupply == 0`,
  `InitialHolder` is the zero address, `PolicyPresent == 0`, and
  `MintMode == 1`. The collection starts empty.
* `PolicyPresent == 1` only for FUNGIBLE. Then `TeamAddress` is a nonzero type-4
  address; for each parameter `Min <= Current <= Max` and `Mutable` is `0` or
  `1`; `TeamRate.Max` and `BurnRate.Max` are at most `10000` and their sum is at
  most `10000`. `ParamController` is a nonzero type-4 address iff some parameter
  is mutable, else the zero address. With `PolicyPresent == 0`, `ParamController`
  is the zero address.
* The asset record `AssetID` MUST NOT already exist. Create it with `Supply = InitialSupply` and, if
  `InitialSupply > 0`, credit `InitialHolder`.

**TRANSFER.** `Amount >= 1`; `Recipient` is a nonzero type-4 address; the asset
exists.

* Without a policy (including NON_FUNGIBLE and SEMI_FUNGIBLE): subtract `Amount`
  from the sender's holding of `(AssetID, TokenID)`, add it to the recipient's.
  For FUNGIBLE, `TokenID == 0`. For NON_FUNGIBLE, `TokenID != 0` and
  `Amount == 1`. For SEMI_FUNGIBLE, `TokenID != 0` and the item exists.
  `MinReceive` MUST be `0`.
* FUNGIBLE with a policy (`TokenID` is `0` for every FUNGIBLE asset, with or without a policy), using the current parameter values:

```text
team      = min( floor(Amount * TeamRate / 10000), TeamCap )
burn      = floor(Amount * BurnRate / 10000)
recipient = Amount - team - burn
```

  A `TeamCap` of `2^64 - 1` means that there is no cap. `Amount * TeamRate` and
  `Amount * BurnRate` use 128-bit intermediates. Require
  `recipient >= MinReceive`. Apply, in this order and on the live record when
  addresses coincide: subtract `Amount` from the sender; add `recipient` to
  `Recipient`; add `team` to `TeamAddress`; subtract `burn` from `Supply`.
  `MinReceive` lets a signer bound the effect of a parameter change that lands
  before the operation.

**MINT.** `MintMode == 1` and `Sender == MintController`. `Amount >= 1`;
`Recipient` is a nonzero type-4 address.

* FUNGIBLE: `TokenID == 0`, `MetaHash` is 32 zero bytes, `ItemMaxSupply == 0`;
  `Supply + Amount <= MaxSupply`. `Supply` is the live supply: burned units free capacity under `MaxSupply`
  (for a NON_FUNGIBLE collection too, although a burned token ID is never reused).
* NON_FUNGIBLE: `TokenID != 0`, `Amount == 1`, no item record exists for
  `TokenID` (a burned token ID cannot be minted again), `ItemMaxSupply == 1`,
  `Supply + 1 <= MaxSupply`. Create `ItemRecord(MetaHash, 1, 1, 0)`.
* SEMI_FUNGIBLE: `TokenID != 0`. If the item is absent, `1 <= ItemMaxSupply` and
  `Amount <= ItemMaxSupply`; create `ItemRecord(MetaHash, ItemMaxSupply, Amount,
  0)`. If it exists, `MetaHash` and `ItemMaxSupply` MUST equal the stored values
  and `ItemSupply + Amount <= ItemMaxSupply`. In both cases
  `Supply + Amount <= MaxSupply`.

  Credit `Recipient` and increase `Supply` (and `ItemSupply`).

**BURN.** `BurnMode == 1`; `Amount >= 1`; the sender holds at least `Amount` of the holding `(AssetID, TokenID)`,
which does not exist, and so holds nothing, unless `TokenID` is `0` for a FUNGIBLE asset and the ID of an existing item otherwise.
Subtract from the sender's holding and from `Supply` (and `ItemSupply`). For
NON_FUNGIBLE set `Burned = 1`; the item record stays as a tombstone. A
controller can never burn another holder's units with this command.

**UPDATE_PARAMETER.** The asset is FUNGIBLE with `PolicyPresent == 1`;
`Sender == ParamController`; `ParamID <= 2`; that parameter has `Mutable == 1`;
`Min <= NewValue <= Max`. Set `Current = NewValue`. Formulas, destinations and
bounds never change.

**REQUEST_WITHDRAWAL.** `Beneficiary` is an account address of type 2, 3 or 4;
`Amount >= 1`; `Credit >= Amount` after the fee. Subtract `Amount` from `Credit`. The operation
is a payout of the batch: the anchor lists it with the index of its step (section 5.6). A batch
holds at most `MaxPayoutsPerAnchor` of them.

#### 5.6 Batches, steps and traces

A batch advances the state by one sequence number. It is a header followed by steps, and
the state root is recorded after every step.

`ListRoot(L)` of a list of byte strings `L` of length `n >= 1` is the Merkle tree hash of
RFC 9162 with BLAKE2b-256: `ListRoot([d]) = Hash(0x00 || d)`, and for `n > 1`,
`ListRoot(L) = Hash(0x01 || ListRoot(L[0:k]) || ListRoot(L[k:n]))`, where `k` is the
largest power of two smaller than `n`. An audit path is the list of sibling hashes of
RFC 9162 section 2.1.3.1, bottom up, and is checked as in section 2.1.3.2.

```text
Header (70 bytes) =
    Version u8 (=1) | Kind u8 (1 anchor batch, 2 resume batch) | Epoch u32 | Seq u64 |
    PrevStateRoot (32) | RefHeight u32 | CreditCount u32 | OpCount u32 | Reserved u32 (=0) |
    NewCursor u64        (NewExitCursor for a resume batch)

Steps, in this order, each encoded as a leaf:
    CREDIT  0x01 | Seq u64 | For (21) | Amount u64               one per deposit of D
    OP      0x02 | Length u32 | Operation                         OpCount of them
    POOL    0x03 | FeeShare u64 | Refund u64                      exactly one, last
    EXIT    0x04 | Kind u8 | Key (32) | Amount u64               resume batches only, in place of the above;
                                                                  Kind MUST be 0 in v1, any other value is an invalid step

BatchBytes = Header || for each step: Length u32 || step leaf
DataRoot   = ListRoot([Header, Step_1, ..., Step_N])
Trace      = R_0, R_1, ..., R_N      R_0 = PrevStateRoot, R_s = state root after step s
TraceRoot  = ListRoot([R_0, R_1, ..., R_N])
DataSize   = length of BatchBytes
```

`StateRoot` of the anchor is `R_N`. In an anchor batch `N = CreditCount + OpCount + 1`; in a
resume batch `N` is the number of EXIT steps, and `CreditCount = OpCount = 0`.

A `REQUEST_WITHDRAWAL` operation of a batch **is** a payout: the anchor lists, for each, `StepIndex`, `To` and
`Amount` in order (section 3.3), at most `MaxPayoutsPerAnchor` per batch. A producer leaves the others in its
pool, where they change no state.

Execution of an anchor batch:

1. `PrevStateRoot` equals the `StateRoot` of the tip; `Seq` equals `TipSeq + 1`.
   `RefHeight` is not below the `RefHeight` of the previous batch, and at signing time
   it is at most the operator's finalized Pactus height `F` and at least
   `F - MaxRefLag`, with `MaxRefLag = 30` blocks. Pactus can check only the outer bounds: a header is wrong if `RefHeight` is above `SubmitHeight` or more than
   `MaxRefLag` blocks before the first block of `Epoch` (section 3.7, kind `2`), so `Expiry` is enforced against a
   colluding committee only to the granularity of one epoch.
2. Each CREDIT step adds `Amount` to the `Credit` of `For`. The steps MUST be exactly the
   records of `[DepositCursor, NewCursor)` that are creditable in the batch `Epoch`, as the
   operator sees them in finalized Pactus state, in ascending sequence.
3. Each OP step executes the operation (section 5.5, with the state as the previous steps left
   it).
4. The POOL step subtracts `FeeShare + Refund` from the fee pool, which MUST cover it.
   `FeeShare` is `0`, or it satisfies rule 10 of section 3.3 for the batch `Epoch`: no fee
   share was paid earlier in that epoch, and it is a positive multiple of the number of
   non-slashed members of `Committee(Epoch)`. `Refund` is at most `AnchorRefund`.

The anchor's `FeeShare` and `Refund` are those of the POOL step. A batch contains at least one
deposit, operation, fee share or refund.

**Resume batches.** A batch for an `AssetResume` has only EXIT steps, one for each PAC claim in
`[ExitCursor, NewExitCursor)`: `Kind` is `0`, `Key` is the state-tree key of the claim and
`Amount` its amount. Executing an EXIT step removes what Pactus has already paid: it sets the
`Credit` of the account at `Key` to zero, and that credit MUST equal `Amount`. `Out` grows by
the sum of the amounts, because Pactus has already counted them in `Paid`. Receipts have no
effect on the state.

**Conservation.** Let `Cred` be the sum of all `Credit`, `Pool` the fee pool, `Dep` the
cumulative amount of deposits credited and `Out` the cumulative amount paid out by anchors
(withdrawal payouts, fee shares and refunds). At every head, `Cred + Pool = Dep - Out`. Pactus
holds `Consumed = Dep` and `Paid <= Out`, the difference being the slack of sections 3.3 and
3.7, so the PAC in escrow always covers every credit and the fee pool. Operators MUST check
this equality before signing.

#### 5.7 What operators must do

Before signing an attestation an operator MUST: hold the full batch bytes;
re-execute the batch step by step from the previous state, obtain every root of the trace,
and check that `StateRoot` is the last one, that the trace matches `TraceRoot` and the steps
match `DataRoot`; check the deposit list and `DepositDigest` against its own finalized view
of Pactus; check the payouts and their step indices; and check conservation. It SHOULD also
check each step with the verifier of section 5.9, the code that Pactus would run against it.
An operator signs a state it has computed itself, not one it has been sent. An operator MUST sign at most one
`ContentHash` for a given `(Epoch, Seq, PrevHead)`; two are slashable. An operator MUST
write that triple and the `ContentHash` to durable storage before it releases the
signature, and MUST NOT sign for that triple again if it cannot read that record back,
for example after losing its disk. A crash or a restore from backup is the most likely
way for an honest operator to be slashed. The signing key SHOULD live in hardware that
refuses to sign twice for one triple. If an anchor is not included, anyone resubmits the same attestation. After an epoch change the old attestation is
dead and operators sign the batch again for the new epoch; that second signature is not slashable, because the two
attestations differ in `Epoch`.

Operators MUST retain every batch and enough state snapshots to let a new
operator rebuild the state from a snapshot and replay later batches. They SHOULD
serve current state with membership proofs against the Pactus-held `StateRoot`,
so a wallet can check a balance against a root that Pactus finalized.

**Watching.** v1 relies on someone holding the data and checking it, so every operator that
signed MUST publish each batch and its trace, with the witnesses of every step, to anyone who
asks, as soon as it signs. Anyone else who can obtain a batch can check it against
section 5.9 and, if a step is wrong, submit an `AssetFraud` within `ChallengeWindow`. An
operator outside the majority SHOULD do so for every pending anchor: its bond is not at risk
when it challenges, and the reward is half of what the signers lose. Pactus does not verify that
the data is available (section 3.7), so an operator that is given a head without its batch
SHOULD refuse to sign it, and raise the alarm off-chain.

The committee decides evictions off-chain and carries them in the `EvictIndex` of an
anchor. An operator SHOULD vote to evict a peer only after it has been unreachable
for a long time, and SHOULD NOT use a missing signature in a relayed bitmap as
evidence of inactivity, because the relayer chooses the bitmap.

How operators order operations, propose a batch, and exchange signatures is
outside this PIP.

#### 5.8 Fees

Every fee is PAC taken from the sender's credit into the fee pool (section 5.5, step
4). The pool leaves the layer only as the refund and the fee share of an anchor
(section 3.3). Fees are therefore the only revenue of operators in v1.

An operation MUST pay at least the **required fee**:

```text
required = Base + NewHolding * h + NewItem * i + CreateAsset * [type == CREATE_ASSET]
```

where `h` is the number of holding entries the operation creates (a holding entry
is created when its key had no entry and now holds a nonzero quantity: the
recipient, the `TeamAddress` and the `InitialHolder` all count), and `i` is the number
of item records it creates. In nano PAC, 1 PAC being 1 000 000 000:

| Constant | Value | In PAC |
| --- | --- | --- |
| `Base` (every operation type) | 100 000 | 0.0001 |
| `NewHolding` | 1 000 000 | 0.001 |
| `NewItem` | 1 000 000 | 0.001 |
| `CreateAsset` | 1 000 000 000 | 1 |

These values are initial proposals. For scale, the default fixed fee of a Pactus
transaction is 0.01 PAC, so `Base` is one hundredth of it.

The floors are validity rules of the Asset Layer: a batch with an operation below its floor is invalid. They price
state that every operator must keep for ever. A holding that falls to zero is removed, so creation is not charged
twice, and the whole `Fee` is kept: there is no refund of an overpayment. Whether a holding is created can depend
on earlier operations of the same batch, so a wallet SHOULD ask an operator for a quote and add a margin.

#### 5.9 Executing one step, and the verifier on Pactus

To check an `AssetFraud`, a Pactus node executes a single step. The rules of sections 5.2,
5.2.1, 5.5, 5.6, 5.8 and of this section are therefore also code of the Pactus node, run only
for fraud proofs and bounded by the size of one step. A difference between an operator's
library and that code is a way for an honest operator to be slashed, so operators SHOULD run
the same code and check their own trace with it before they sign.

**State needed by a step.** A step reads and writes a small set of entries of the state tree.
Its **witnesses** give the value of each of them before the step (or that it is absent), each
with a membership proof against `R_{s-1}` (section 5.2.1, where an absent entry is proved by
starting from `Empty[0]` instead of a leaf). The verifier needs a witness for every key the
step reads or writes, and rejects a payload with a missing witness or an extra one.

| Step | Keys it reads or writes |
| --- | --- |
| CREDIT | `ACCT(For)` |
| POOL | `POOL` |
| EXIT | The `Key` of the step, which is an `ACCT` key |
| `TRANSFER` | `ACCT(Sender)`, `POOL`, `ASSET(AssetID)`, `HOLD(AssetID, TokenID, Sender)`, `HOLD(AssetID, TokenID, Recipient)`; `HOLD(AssetID, 0, TeamAddress)` if the asset has a policy; `ITEM(AssetID, TokenID)` if `TokenID != 0` |
| `CREATE_ASSET` | `ACCT(Sender)`, `POOL`, `ASSET(new AssetID)`; `HOLD(new AssetID, 0, InitialHolder)` if `InitialSupply > 0` |
| `MINT` | `ACCT(Sender)`, `POOL`, `ASSET(AssetID)`, `HOLD(AssetID, TokenID, Recipient)`; `ITEM(AssetID, TokenID)` if `TokenID != 0` |
| `BURN` | `ACCT(Sender)`, `POOL`, `ASSET(AssetID)`, `HOLD(AssetID, TokenID, Sender)`; `ITEM(AssetID, TokenID)` if `TokenID != 0` |
| `UPDATE_PARAMETER` | `ACCT(Sender)`, `POOL`, `ASSET(AssetID)` |
| `REQUEST_WITHDRAWAL` | `ACCT(Sender)`, `POOL` |

`ACCT`, `POOL`, `ASSET`, `ITEM` and `HOLD` are the keys of section 5.2. A key that depends on
the content of another, such as the `TeamAddress` of the asset, is taken from the witness of
that other key.

**What a step does.** A CREDIT step adds `Amount` to the `Credit` of `For` (creating the account
record with nonce `0` if it is absent; an overflow makes the step invalid). A POOL step subtracts
`FeeShare + Refund` from the pool, which must be at least that large. An EXIT step sets the
`Credit` of the account at `Key` to zero, and the `Credit` read MUST equal `Amount`. An
operation step runs the rules of section 5.5 on the values of the witnesses, including the
signature, the nonce, `Expiry` against the `RefHeight` of the header, the fee floor of section
5.8, and the writes of its body. A write of an all-zero value removes the entry. A step is
**invalid** if it breaks any of these rules.

**New root.** Let `new` map each key of the witnesses to its value after the step, which is
its old value if the step did not write it. `UpdateRoot` computes the root `R_s'` after the
step from the witnesses alone:

```text
node(d, P):                       P is a d-bit prefix
    if no witness key has prefix P: return Sibling(d, P)
    if d == 256: return Empty[0] if new[P] is absent, else Hash(0x00 || P || Hash(new[P]))
    return Hash(0x01 || node(d+1, P||0) || node(d+1, P||1))

Sibling(d, P):                    d >= 1
    take a witness whose key has prefix P[0:d-1] and a different bit at position d-1
    return its sibling at level 256 - d      (explicit if its bit is set, else Empty[256-d])

UpdateRoot = node(0, empty prefix)
```

The set of keys of a step is a function of its decoded bytes and of the `ASSET` entry that it names, not
of whether the step is valid: a step that does not decode needs no witness, since that already shows fraud (kind `0`,
case (a)). Fields such as `Version` are checked when the step is executed, after the keys are known.

Every untouched subtree that the recursion reaches is a sibling on the path of some witness,
so `Sibling` always finds one. The step is accepted by the verifier as correct if it is valid
and `UpdateRoot` equals `PostRoot`.

### 6. Capacity and load on Pactus

This section is informative, except for the caps of sections 3.5 and 3.7. The ceilings follow from the
parameters. **They are not measured throughput.**

#### 6.1 What an asset user costs Pactus

Transfers, mints, burns, creations and parameter updates cost Pactus nothing: they live in the Asset Layer. Funding
fee credit is one `Deposit` transaction. A withdrawal is an entry of an anchor (at most 64). Settling a batch of
any size is one anchor (at most 2 395 bytes, one per block, 864 pending). After a freeze, exiting is one
`ExitClaim` of at most 8 505 bytes (32 per block), and resuming is one `AssetResume` at a time (at most 258 bytes,
at most 262 144 claims).

#### 6.2 Ceilings

* **Pactus alone.** 1 000 transactions per block every 10 seconds: at most 100 per second for every payload type
  together.
* **Asset operations.** A batch is at most 33 554 432 bytes and a signed `TRANSFER` is 221 bytes with its
  length prefix, so a batch holds about 151 800. With at most 864 anchors pending, each for a day, the sustained
  rate is at most `864 x 151 800 / 86 400`, about 1 500 transfers per second.
* **Withdrawals.** At most 64 payouts per anchor, about 0.64 per second; a larger backlog waits in the operators'
  pools. **Deposits:** a block cannot hold more than the next anchor can consume (1 024).
* **Claims after a freeze.** At most 32 per block, about 3.2 per second: 100 000 holders need about 8.7 hours.
  Resumes apply 262 144 claims per day, a little less than the 276 480 that can arrive, and the first resume
  closes PAC claims.
* **Block size.** A block of only new payload types (32 `ExitClaim`, 2 `AssetFraud`, one anchor, 964
  `AssetEvidence`) is about 0.92 MB, under the 1 MB gossip limit. It is a bound on bytes, not a valid block.

#### 6.3 Where the limit moves

In v1 every operator re-executes every batch and stores all data, so adding operators adds no capacity. The
reference implementation MUST report, on stated hardware: operations per second one operator sustains on a mixed
workload; time to execute and sign a 32 MiB batch; cost of a tree update and a membership proof; time to verify
an anchor and an `AssetFraud` of each step type; size of the pending state at `MaxPending`; and the effect of an
anchor per block on block validation time.

#### 6.4 Operator economics

This section is informative and does not claim that running an operator is profitable.

* **Revenue** is the fee pool, paid in equal parts to the owners of the non-slashed seats, at most once per
  epoch, a day late. There is no subsidy.
* **Settlement cost.** Each anchor pays one Pactus fee (0.01 PAC by default), refunded to the submitter up to
  `AnchorRefund`: 8.64 PAC per day for an anchor every 10 blocks. An anchor pays for itself only if its batch
  carries enough fees, about 100 operations at the base fee.
* **Capital.** 10 000 PAC per seat, locked while bonded and for 21 days after: 210 000 PAC for 21 seats, so seats
  may fill slowly. A committee smaller than `MaxCommittee` works, down to `MinCommittee`.
* **Not paid for:** watching other operators' batches, servers, bandwidth and keeping every batch for ever. A
  silent operator earns the same share until it is evicted or its term ends.

The escrow ramp and the fee floors bound usage in the first months, so revenue starts low.

## Rationale

**Committee first, validity proofs later.** A proof system that verifies secp256k1 signatures and executes the
asset rules has no deployed cost data in the Pactus setting, so a PIP that depends on it cannot give byte-exact
encodings today. A committee attestation can, and `AttestationType` leaves the settlement record, the deposit
queue, the payouts and the state root unchanged when a later PIP adds proofs. Single-step fraud proofs are the
bridge: the rules fit in a few pages and need no virtual machine.

**Not the validators as the committee.** It would make every validator a participant in the Asset Layer. Pactus
has no slashing, so validator stake cannot back accountability without touching consensus; a separate bond
cannot. Operators are volunteers who hold state and serve data for a whole term, which a rotating consensus
committee does not do.

**Seats, terms and rotation.** Fixed seats keep aggregation cost constant. A term of 180 epochs with a FIFO
waiting list lets new operators in without auctions, and a holder keeps an expired seat while nobody waits, so a
quiet period cannot shrink the committee. Capping rotations at 6 per window, below `T`, means expiry alone never
gives a party a majority within a window; one per epoch makes them visible. `Prune` stops a party from holding
the registry with waiting records. Eviction is the committee's decision because the only evidence of inactivity
is a bitmap that the submitter chooses. None of this removes the need to trust that fewer than `T` seats belong
to one party.

**A quorum above two thirds.** With two thirds, a committee of 15 can sign among themselves and hide the batch
from the other 6. A higher quorum checks nothing either, but a head with `T` signatures includes an honest
operator unless `T` or more are faulty, and an honest operator signs only what it holds. It leaves 3 of 21
operators free to be offline: a choice between liveness and the attack threshold.

**One aggregate signature, epochs.** Pactus already verifies BLS aggregates, and one pairing check covers any
committee size. Epochs keep the committee stable between signing and inclusion; a signature that crosses an
epoch is re-made, which is cheap because there is no proving step.

**Payouts in the anchor.** Listing at most 64 payouts needs no withdrawal tree, proof per claim or claimed-set,
and a withdrawal that is an operation of the batch needs no provable queue. Claims with proofs exist only after a
freeze, when no committee can list payouts.

**Freeze, exit and resume.** Fee credit is the one thing a user has on Pactus that the committee could strand; a
proof against the last final head returns it without anyone's permission. A rescue committee would let a party
that bonds enough seats attest any state in a halt it may have caused. The exit pays only PAC; native positions
are receipts, because Pactus holds no representation of them. A freeze nobody can undo turns a month's outage
into the end of the layer, so the committee can resume with `T` signatures, which adds no power. Receipts are
void after a resume rather than debited, so a position recorded in a hurry stays usable in the layer. The first
resume covers only claims of earlier epochs and closes PAC claims, so the set it applies is fixed when the
committee signs.

**No forced inclusion.** Without a validity proof Pactus cannot tell a legitimate rejection from censorship; a
forced path would turn silent censorship into a signed rejection that nothing can overturn.

**Fraud proofs on one step, deferred effects.** Pactus cannot run a batch of 150 000 operations, but it can run
one: the rules are a fixed list of commands with checked arithmetic, a sparse Merkle tree and a signature scheme
it already verifies. The trace commits to the root after every step, so a challenger shows the step and its two
roots with no interactive search. Effects are deferred a day because a proven fraud would otherwise arrive too
late; the window must let a watcher fetch, check and get a transaction into a block. A day is a proposal.
`AssetOpen` exists because a committee could commit to a root with no valid path to a wrong step. Only the
signers of the fraudulent anchor are slashed: later signers built on it but did not create it.

**No data-availability guarantee.** Guaranteeing availability means erasure-coding batches across independent
providers and sampling them, a second protocol as large as this one. The cost is that a committee that withholds
a batch from everyone for a day makes fraud impossible to show. The design does what it can: every signer MUST
publish the batch on request, an operator that was not given a batch SHOULD refuse to sign, any one party with
the data is enough, and the reward for a proof is large.

**Deposits and expiry by epoch.** Without reclaim, silent operators would hold deposits hostage. A reclaim that
depended on transaction order inside a block would make a signed anchor fail and force operators to sign other
content for the same head, which looks like equivocation. With expiry by epoch a deposit is credited or
reclaimed, never both, and no honest re-signature looks like equivocation. The price is a credit window of one
to two epochs. `MaxDepositsPerAnchor` is tied to `MaxTransactionsPerBlock` so that an anchor per block always
drains the queue; `MaxExitClaimsPerBlock` keeps a freeze-time crowd from building a block the 1 MB gossip limit
cannot relay.

**Fees.** A subsidy would change Pactus monetary policy for a feature with no users. An equal share to each seat
needs no per-operator counter, and a split weighted by signatures would be steered by the aggregator. The
refund is fixed because the submitter chooses its fee after the attestation, and it goes to whoever gets the
anchor included so any relayer can finish one. Floors with a state surcharge price what operators keep for ever.
A fixed transfer schedule covers the team and burn shares in a page; a policy language needs bytecode, and the
operation envelope has a `Version` byte for it.

**Bond and parameters in code.** A larger bond makes a committee costlier to assemble and attack: 18 attackers
risk 180 000 PAC, more than the 50 000 PAC cap of the first 90 days and less than the 500 000 PAC cap that
follows. The price is that seats fill more slowly. Neither the bond nor the committee size can be validated
before the layer has run, so both are constants a protocol version may change (section 1.3.1). The bond
protects only the PAC; against native assets it is not insurance. The escrow ramp bounds what a compromised
committee can drain and is fixed in the PIP so nobody has to decide later whether to relax it. `MinReceive`
lets a signer bound the effect of a parameter change that lands before the operation.

## Alternatives Considered

1. **Native token payloads in Pactus consensus.** Rejected: every validator stores all balances and replays all
   transfers, and each transfer competes for one of 1 000 slots per block. Putting all operation bytes on Pactus,
   or a forced inbox, recreates it.
2. **A smart-contract VM.** Rejected: attack surface out of proportion to the tokens this PIP targets.
3. **Validity proofs from the start.** Deferred: they need a proof system for secp256k1 signatures and the asset
   rules, and their cost is unknown. Erasure-coded availability with sampling is also deferred: it is a second
   protocol.
4. **Interactive bisection instead of a trace root.** Rejected: several transactions and a timer per dispute,
   against 32 bytes per step.
5. **Slashing validator stake, or a rescue committee after a halt.** Rejected: Pactus has no slashing, and a
   rescue committee is a takeover risk.
6. **A plain two-thirds quorum, or unanimity.** Two thirds (15 of 21) was rejected after the comparison with
   Arbitrum AnyTrust: `T = 18` leaves an honest holder of every final batch unless `T` are faulty, at the price of
   liveness (the layer stops with 4 operators offline, not 7). Unanimity was rejected: two crashed operators
   would stop the layer and could not be evicted, since eviction travels in an anchor.
7. **A flat escrow cap of 500 000 PAC from day one, or seats by auction or stake weight.** Rejected in favor of
   the ramp and the fixed bond.
8. **Eviction by anyone measured on the bitmap, or rotation without a window cap or approved by the committee.**
   Rejected: the submitter chooses the bitmap, all launch terms end together, and the committee would control its
   own composition.
9. **Reclaim of deposits by block height.** Rejected: a stale anchor would force a second signature on different
   content.
10. **A resume that debits receipts or covers all claims with no epoch limit.** Rejected: positions would leave
    the layer unspendable, or a same-block claim would invalidate an attested resume.
11. **A refund of the exact fee, to a named operator, or none; a subsidy from block rewards; a free fee market;
    fees split by signatures.** Rejected: the fee is chosen after the attestation, a named operator could hold up
    anchors, and the others change monetary policy, price no state or are steered by the aggregator.
12. **Slashing every signer of every reverted anchor.** Rejected: one bad batch would ruin the committee.
13. **A shorter or longer window.** Hours would let a committee online at night finalize a bad head unseen; a week
    would make withdrawals slow. A day is a compromise, not a measurement.

## Backwards Compatibility

This is a consensus upgrade. Nodes that do not implement version `V` reject
payload types 8 to 14 and cannot compute the extended state root. Existing payload
types, account records and validator records keep their encoding. Accounts that
never use the feature are unaffected.

Activation follows PIP-51: implementations advertise version `V`; when more than
75 % of committee power supports it, proposers raise the block version; from the
first block of version `V` the extended header root applies, `ActivationHeight` is
set to that block's height and the seven types are legal. Before that block they
MUST be rejected as invalid payload types. As with
every earlier version, the 75 % threshold is a proposer rule, not a validation
rule: a version-`V` block is valid if it carries a certificate from more than 2/3
of committee power, which requires that power to run version-`V` software.

The layer does not start by itself. It needs at least `MinCommittee` bonded
operators whose first epoch has begun. Until then no anchor is valid, and deposits
expire after their epoch window and can be reclaimed. Testnet SHOULD activate first.

## Test Cases

### Vectors

`vectors.md`, submitted in the assets of this PIP next to `gen_vectors.py` (Python 3, standard library only), lists
test values computed with plain BLAKE2b-256 from the layouts of sections 2.5, 3.5, 3.7, 5.2, 5.2.1 and 5.6, and
the script checks that a tampered proof or audit path fails. They cover: the empty state root and the genesis
head; an asset ID, a holding key and a one-entry state root; the transfer policy table below; `ListRoot` and
audit paths; a batch of one pool step with its correct and fraudulent traces and the `AssetFraud` that proves the
latter; the digests, `ContentHash`, `HeadHash` and the 126-byte attestation message of an anchor; a membership
proof with one explicit sibling and an exit-claim leaf; the resume digest and head; and a receipt leaf.

| Name | Value |
| --- | --- |
| `EmptyStateRoot` | `933f40aae7a4ce7f2705c889bd9417d6360695ec0852fb3e1326b1c03dc6da13` |
| `GenesisHeadHash` | `66bd737f4f983634f07fc435ab1efd50c84b62dbf03372cdd14cf847904d413d` |
| `AssetID` for `Creator` = `0x04` followed by twenty `0x11` bytes, nonce `0` | `ce4429f3e1ee9bbda5a0f75adb9b350dd6beaa761174a3fc9cc4c511e3610622` |

Transfer policy (`TeamRate`, `BurnRate` in basis points):

| Amount | `TeamRate` | `BurnRate` | `TeamCap` | team | burn | recipient |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | 200 | 100 | none | 2 | 1 | 97 |
| 99 | 200 | 100 | none | 1 | 0 | 98 |
| 1 | 200 | 100 | none | 0 | 0 | 1 |
| 1 000 000 | 200 | 100 | 5 000 | 5 000 | 10 000 | 985 000 |

### Consensus tests (Pactus node)

Implementations MUST pass at least these. They add no rule to the Specification.

* **Activation and root.** Payload types 8 to 14 are rejected below version `V`; the first block of `V` carries
  the extended root of the initial settlement; the root changes with `Escrow`, an operator record or a deposit
  leaf; a restart reloads the same root.
* **Operators.** Bond with a valid `PoP` is seated when seats are free and nobody waits, waiting otherwise; a
  wrong `PoP`, an identity or out-of-subgroup key, a duplicate key, a validator's key and a full registry are
  rejected. Unbond, Withdraw, `EvictIndex`, `Seat` (free seat, expired term, both rotation limits) and `Prune`
  follow section 3.1, including every refusal listed there. Committee membership never changes inside an epoch.
* **Escrow.** Deposits below `MinDeposit`, to a non-type-4 address or beyond `MaxEscrow(h)` are rejected, with
  the ramp at 777 600 and 1 555 200 blocks. Reclaim is refused before expiry, for another account, for a credited
  deposit and twice. An expired deposit is never in `D`. PIP-54 overflow cases fail before any addition.
* **Anchor.** A correct anchor is recorded as pending and changes none of `HeadSeq`, `Consumed`, `Paid`, `Escrow`
  or any balance. Each rule of section 3.3 has a failing case, among them one signature fewer than `T`, a bit past
  `n`, a wrong bitmap, a second anchor in a block, a wrong `PrevHead`, a full pending list, a cursor out of range,
  a consumed deposit from the same block, payouts above the escrow, an identity aggregate. An anchor that credits
  a deposit made in the same block fails, and the same attestation succeeds in the next block. Finalization at
  `SubmitHeight + ChallengeWindow` pays payouts, `min(Refund, fee)` and the fee share, and finalizes at most
  `MaxFinalizePerBlock` records.
* **Evidence.** Two contents for one `(Epoch, Seq, PrevHead)` slash the operator; equal contents, another key, a
  non-member epoch or different epochs fail; evidence works after `ExitEpoch` and while frozen.
* **Challenge.** With the vectors, an `AssetFraud` of kind `0` on the fraudulent trace reverts a pending anchor
  and slashes its signers, half to the reporter; the same proof against a correct trace, a changed sibling, a
  missing or extra witness or a wrong path has no effect. Reverts restore the cursors and sums, release deposits
  and slash nobody but the signers of the challenged record. Proofs are refused for a final record and from
  `SubmitHeight + ChallengeWindow`. Each kind and each case of kind `0` is accepted when wrong and rejected when
  correct. A block with 3 `AssetFraud` is invalid. `AssetOpen` request, response and timeout behave as in
  section 3.8, and a record with a random `TraceRoot` is reverted.
* **Freeze, exit, receipts.** Freeze is refused one block early, with nothing final, with something pending or
  when frozen. `ExitClaim` kind `0` with the vector proof pays `7 000 000 000` and appends the vector leaf; every
  tampered field, a duplicate key, `Paid + Amount > Consumed` and a pending head are refused; a block with 33
  claims is invalid. Receipts follow the vectors, change no balance and are refused in state `0`.
* **Resume.** A resume that applies every claim of earlier epochs returns the layer to `0` and sets
  `ExportsVoidBefore`; one that leaves claims moves it to `2`. A claim in the epoch of the first resume does not
  invalidate it. A stalled state `2` can be frozen again.
* **Accounting.** Balances, stake, `Escrow`, `BondedTotal`, the treasury and pending amounts sum to the supply
  after every test. A pruned restart returns the same `GetAssetSettlement`.
* **Regressions.** A fraud proof slashes exactly `SignerIndexes`, even after a signer was slashed and withdrawn. An
  account can claim again in a second freeze. A backlog finalizes at most `MaxFinalizePerBlock` per block. A
  header with a bad `RefHeight` is proven wrong by kind `2`.
* **Parameters.** With `MaxCommittee = 51` an anchor needs 43 signatures and a 7-byte bitmap; 42 signatures or
  9 bytes are refused. After `OperatorBond` changes, a record keeps the amount it posted.

### Asset Layer tests (operator libraries)

The vectors reproduce byte for byte. Each asset type with each capability is created, and each rule violation of
section 5.5 and each fee below a floor is rejected. A policy transfer follows the table above, also when sender,
recipient and `TeamAddress` coincide. `MinReceive`, `UPDATE_PARAMETER` bounds, NFT and SFT mint and burn rules,
replays and another `ChainID` fail as specified. Conservation holds after a batch of every operation. A batch
with one invalid operation, or a bad `RefHeight`, is invalid as a whole. Different tree partitionings give the
same root. `ListRoot`, audit paths, membership proofs and `UpdateRoot` (shared ancestors, writes, removals) match
the vectors. Executing each step from its witnesses gives the state of the operator's full execution.

## Reference Implementation

None yet. Before this PIP leaves Draft, a pull request will add the seven payloads,
the settlement state, the exit and export trees, the proof check and the extended root to the
Pactus node, and a separate library will implement section 5 and pass the Asset Layer tests. The vector generator and its output are attached to this PIP.

### Open items of this draft

This draft is for review. What it does not have yet:

* **A reference implementation**, as above. No part of this PIP has run inside a Pactus node.
* **Vectors for whole payloads.** The vectors cover hashes, digests, roots, proofs and attestation messages. They
  do not include a fully serialized payload, the hash of a `SettlementState`, or an extended header root.
* **Constants.** Every value of section 1.3 is an initial proposal. The capacity figures of section 6 are derived
  from those values, not measured on an operator.
* **Data availability** is not guaranteed (Security Considerations).
* **Validity proofs** are only a reserved value of `AttestationType`.
* **Size.** The text may be split into several PIPs (operator registry and bonds; settlement and anchors; the
  Asset Layer; fraud proofs, halt and resume) if reviewers prefer.

## Security Considerations

### Committee compromise

`T` colluding operators can attest an incorrect state. What follows depends on whether anyone checks it.

* **If a party with the data challenges within `ChallengeWindow`**, the head is reverted, nothing is paid for it,
  its reserved deposits are released, and every signer loses its bond (180 000 PAC for `T = 18` at the launch
  bond), half to the reporter. Nothing is lost on Pactus.
* **If nobody challenges**, because nobody had the data, nobody checked, or the proof could not be included in
  time, the head becomes final. Native assets can be reassigned or destroyed, and up to `Consumed - Paid` PAC can
  be paid to chosen accounts. Uncredited deposits stay reclaimable.

Ordinary PAC, validator stake, other operators' bonds and consensus are not reachable. Users SHOULD limit what
they entrust to the layer and keep fee credit small. `MaxEscrow(h)` bounds the PAC loss; the value of native
assets has no bound. At the launch values the bonds of `T` operators exceed the PAC the layer may hold in the
first 90 days (50 000), not the final cap (500 000), and never bound the value of native assets.

A caught committee is replaced by whoever bonds first in the next epoch, with no limit per epoch: `T` new
operators that bond together hold `T` of 21 seats. Until `MinCommittee` members exist no anchor is possible. The
honest minority and the community SHOULD be ready to bond in that epoch. A limit on seatings per epoch would give
them time, at the price of a slower launch; v1 has none.

Slashing covers conflicting attestations of one epoch and an anchor proven wrong, nothing else. Operators who
all sign the same wrong head and are never challenged are not punished. At 21 seats a conflicting pair of heads
implicates at least 15 operators (`2T - n`); a wrong anchor implicates every signer.

### Watching, and what can go wrong

The protection of v1 is only as good as the watching, and Pactus does not watch. Operators outside the majority
receive the batches and have reason to check them.

* *Nobody checks in time.* A watcher offline for a day is a risk. Operators SHOULD run watchers for each other's
  anchors and publish what they check.
* *A proof is not included.* Censoring a proof for a day takes control of every proposer (Pactus has 51 in
  rotation). `AssetFraud` is bounded to two per block, so a flood of slow proofs could fill the quota; a proposer
  SHOULD give priority to proofs that verify.
* *The verifier and an operator's library differ.* Sections 5.5, 5.8 and 5.9 are consensus code and also
  implemented by operators; a disagreement can slash an honest operator. Operators SHOULD run the Pactus code on
  their own trace before signing.
* *A reverted head delays honest operations* until the committee anchors again. A false proof costs its sender
  the fee.

### Data availability and withholding

Pactus checks signatures, not data, and v1 does not make data available. A committee that wants a false head
through can withhold the batch and state from everyone else: nobody can then build a fraud proof, and the head
becomes final after the window. This is the main thing v1 does not solve.

v1 requires every signer to publish each batch, with its trace and witnesses, to anyone who asks, and an
operator that was not given the batch SHOULD refuse to sign. Operators outside a colluding majority receive the
batches and any one is enough; a user whose balance changes without an operation of theirs is a second watcher,
archives a third. v1 does not verify that any of this happens: a committee of `T` can sign among themselves and
serve nobody, and the other `n - T` can raise an alarm only off-chain. With a validity proof (a later PIP) a
withheld batch would no longer let a false head become final: the layer would freeze and holders would exit
against the last proven head.

### Halting

If fewer than `T` operators are online no anchor can be produced and nobody can be evicted, since eviction
travels in an anchor; the same holds below `MinCommittee`. After `HaltTimeout` anyone can freeze the layer;
holders exit PAC with a Merkle proof and record native positions as receipts, and `T` returning operators can
resume it (section 3.6).

This does not solve everything. A receipt is evidence for a successor layer that a later PIP would define, not a
way to move the asset. If more than `n - T` operators are gone for good the layer never resumes, and `n - T + 1`
operators (4 of 21) can keep it frozen by staying away, with nothing to slash them: the bond does not protect
liveness. Claims and receipts are proven against the last final head: if it overstated credits, claimants are
paid first come, first served until `Consumed - Paid` runs out, and if it misstated positions so do the
receipts. A proof for an old head does not verify against the last one, so wallets cannot prepare claims in
advance. An attacker that records dust claims can add only a few days to a freeze.

### Other risks

* *Censorship.* A user cannot force an operation in v1. A censored user can stop depositing and reclaim
  uncredited deposits; PAC on Pactus is never affected.
* *Denial of service on Pactus.* Verification per anchor is constant, one anchor per block, `MaxPending` pending,
  finalization bounded by `MaxFinalizePerBlock`. A deposit costs the fixed fee and leaves a 32-byte leaf in the
  deposit tree for ever, like any transaction that creates state.
* *Eviction and rotation.* `T` operators can evict honest minority members and replace them from the waiting
  list; they can already attest any state, but a takeover becomes quieter. A well-funded party that bonds early
  can fill the waiting list. The defenses (6 rotations per window, 1 per epoch, `ceil(T / 6)` windows to take
  `T` seats: three at 21 seats, six months or more) slow a takeover and make it visible; they do not stop a party
  that holds `T` seats. A party that fills the waiting list also fills the registry; `Prune` removes records that
  waited 30 epochs, at the cost of their place in the queue. Operators SHOULD join in different epochs at launch.
* *Keys.* An operator's signing key is as valuable as its bond, and an honest operator can be slashed by a
  restore from backup that forgets what the key signed, or by a stolen key used to frame it. Section 5.7
  requires a durable record before signing. An operator key is never a validator key.
* *Sequencers.* The sequencer orders operations and can reorder, delay or omit them. Pactus enforces only the
  outer bounds of `RefHeight`, so against a committee that signs it `Expiry` holds to one epoch. An honest
  operator signs the first of two batches for the same head and refuses the second.
* *Capacity squatting.* A party can deposit up to `MaxEscrow(h)` and leave it, preventing others from
  depositing, at the cost of the capital. It is a denial of service on funding, not a theft.
* *Wallets.* A wallet MUST display spendable PAC separately from fee credit, MUST show the asset ID and not only
  the symbol, SHOULD hide unsolicited assets and SHOULD set `MinReceive` for fungible transfers with a mutable
  policy. A transfer small enough that its team share rounds to zero pays none; splitting costs one fee per piece.
* *Privacy.* Operations and balances are visible to operators and whoever they serve. This PIP adds no privacy.

## Future Extensions (informative)

* **Validity proofs.** A later PIP can define `AttestationType = 1`: an anchor with a succinct proof of correct
  execution, verified by Pactus and final at inclusion, with the committee attesting only that it holds the data.
  A withheld batch then costs liveness (a freeze and exits) instead of funds. It must first settle the proof
  system, the program, the setup and the cost of proving, which depends on the state tree of section 5.2.
* **Data availability sampling** across independent providers, which would remove the main limit of v1.
* **Forced inclusion**, which needs validity proofs so that a rejection can be checked.
* **A successor layer** that honors the receipts and names a committee after a halt that cannot resume.
* **A policy language, paid storage, and fee economics** (a weighted split, a fee market, a refund that follows
  the real fee).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
