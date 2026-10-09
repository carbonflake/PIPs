---
pip: 56
title: Native Assets
description: Fungible, non-fungible and semi-fungible assets as native Pactus state, capped per block to protect PAC.
author: CarbonFlake (@carbonflake)
status: Draft
type: Standards Track
category: Core
created: 2026-10-02
updated: 2026-10-09
requires: 39, 51, 54
---

## Abstract

This PIP adds native assets to Pactus with six payloads, one new state tree, and nothing else: no virtual machine,
no committee, no new cryptography. An asset is a record with a symbol, a number of decimals, a capped supply and a
metadata hash. Balances are kept per account. Assets are signed, paid and replay-protected like every other Pactus
transaction, with the same accounts and keys.

| Payload | Does |
| --- | --- |
| `AssetCreate` | Creates an asset and credits its initial supply to the creator |
| `AssetAccept` | Opts the signer in to an asset against a refundable deposit. Nobody can receive units before accepting |
| `AssetClose` | Closes a balance of the signer, burning what is left if the signer allows it, and returns its deposit |
| `AssetTransfer` | Moves units to 1 to 8 recipients. A transfer to the zero address burns them |
| `AssetMint` | Lets the issuer create more units, up to the `MaxSupply` fixed at creation |
| `AssetSetRate` | Lets the issuer move the transfer rates of an asset, inside the bounds fixed at creation |

Fungible tokens, NFTs and semi-fungible tokens are the same record with different parameters, not different types.

**Nobody can be given an asset against their will.** An account holds an asset only after it has accepted it with its
own signature and locked a deposit for its own record. Transfers and mints never add a record to the state: they only
change a balance that its owner opened. The owner gets the deposit back when it closes a balance, and the freed
place is reused. This is what stops unsolicited airdrops, and it ties the number of balances in use to the PAC that
its owners choose to lock.

An asset may carry an optional **transfer policy**: a team rate and a burn rate, in basis points, that every transfer
applies with a few lines of integer arithmetic. The rates are variable but bounded: each has a minimum and a maximum
that are fixed at creation, and only the issuer can move a rate between them.

Batching happens at two levels. A sender can put up to 8 recipients in one transaction, which costs one signature for
up to 8 transfers. And the block proposer automatically packs the asset transactions of **any** senders, as they
arrive, into **bundles** of up to 8: every bundle of a block but at most one is full. Users send ordinary transactions
and never see a bundle. A bundle takes one slot of the block, so the `MaxAssetTxPerBlock` (200) asset transactions
that a block may carry use at most 25 of its 1 000 slots, and PAC always keeps the rest. A balance is paid for with a
deposit that is returned when it is closed; an asset record, which is permanent, is paid for once.

## Motivation

Pactus has no smart-contract layer. A token today means a memo, an off-chain indexer, or another chain. Putting tokens
into consensus naively has two costs: tokens compete with PAC for the capacity of a block, and every validator stores
every balance, with no way to reuse the room of the ones that nobody uses any more.

This PIP keeps the design as small as the problem allows, in the spirit of Pactus: robust, reliable and light.

* **Robust.** Six payloads and one rule per payload. Every amount is bounded and every sum checked before it is
  added ([PIP-54](./pip-54.md)). The only authorities an asset can have are a mint capped at creation and rates that
  move inside bounds fixed at creation.
* **Reliable.** Assets carry exactly the trust of PAC: the validators check every rule, and nobody else is trusted.
  Nothing depends on an operator, an indexer or an off-chain data store.
* **Light.** No per-block hook, no signature scheme, no epoch, no registry. The load that assets put on L1 is bounded
  by a constant, not by a market.

The commands are few: create, accept, close, transfer, mint, and set a rate. Exchanges, allowances, lending, oracles,
governance, freezing, clawback and metadata updates are outside this PIP.

## Specification

Integers are little-endian and fixed width unless a field is marked *varint*. `Address` is the 21-byte Pactus address,
and the zero address is the treasury address (21 zero bytes). `Hash` is BLAKE2b-256, the hash Pactus already uses, `||`
is byte concatenation, and a quoted string is its ASCII bytes. Fields are encoded in the order listed, using the
encodings of the existing payloads: `Address` as in `Transfer`, and *varint* as the `Amount` of `Transfer` and the
number of recipients of `BatchTransfer` ([PIP-39](./pip-39.md)). Decoding MUST consume exactly the declared fields,
and trailing bytes make a payload invalid. A *varint* MUST use its shortest encoding, and a longer one is invalid, so
that a payload has a single encoding. Every count and length (recipients, symbol, bundle items) MUST be checked
against its bound before anything is allocated or read. Byte-level test vectors come with the reference
implementation.

### 1. Parameters

The values below are initial proposals for community review. They are constants of the protocol version in the node,
and a later version may change them ([PIP-51](./pip-51.md)). A change never alters an existing asset.

| Name | Value | Meaning |
| --- | --- | --- |
| `MaxAssetTxPerBlock` | 200 | Most asset transactions in one block, counted inside the bundles. Bounds the work |
| `MaxBundleItems` | 8 | Asset transactions in one bundle. Bounds the slots, together with the line above |
| `MaxRecipients` | 8 | Recipients of one `AssetTransfer`, as `BatchTransfer` |
| `MaxDecimals` | 9 | Same as PAC |
| `MaxAssetSupply` | 2^63 - 1 | Largest quantity of any asset, the largest `int64` |
| `AssetCharge` | 0.1 PAC | Paid to the treasury, once, by every `AssetCreate`, for a record that is never removed (3.5) |
| `HoldingDeposit` | 0.1 PAC | Locked by every holding that is placed, and returned by `AssetClose` (3.5) |

### 2. State

#### 2.1 The asset tree

A new Merkle tree, `assetMerkle`, is built exactly like `accountMerkle`: leaf `i` is the hash of record `i`, and a new
record is appended at the next index. The tree never shrinks and an `AssetRecord` never moves, so an `AssetID` never
changes meaning. A holding can be replaced, though: a balance that its owner closes becomes a free slot, and the next
balance reuses it. There are four kinds of record: a header at index 0, assets, balances (called holdings) and free
slots. `NoRecord` is the index `0xFFFFFFFF`, and the tree holds at most `0xFFFFFFFF` records, so that `NoRecord` is
never an index: a payload that would append a record to a full tree is invalid. Index 0 is the header, so no
`AssetID` is 0.

`HeaderRecord` (kind `0`), 13 bytes, at index 0 from the first block of version `V`:

| Field | Type | Notes |
| --- | --- | --- |
| `Kind` | `uint8` | `0` |
| `FreeHead` | `uint32` | Index of the free slot freed last, or `NoRecord` |
| `DepositTotal` | `int64` | PAC locked by the deposits of all holdings, in nano PAC |

`AssetRecord` (kind `1`), 82 to 93 bytes, or 123 to 134 with a transfer policy:

| Field | Type | Notes |
| --- | --- | --- |
| `Kind` | `uint8` | `1` |
| `Symbol` | `uint8` length, then bytes | 1 to 12 bytes of `A-Z0-9` |
| `Decimals` | `uint8` | At most `MaxDecimals` |
| `MaxSupply` | `int64` | Cap on `Issued`, fixed at creation |
| `Issued` | `int64` | Units created so far, never decreases |
| `Burned` | `int64` | Units destroyed so far, never decreases |
| `Issuer` | `Address` | The creator. The only account that may mint or set a rate |
| `MetaHash` | 32 bytes | Hash of an off-chain description, fixed at creation. All zero if none |
| `PolicyPresent` | `uint8` | `0` or `1`. Fixed at creation |

If `PolicyPresent` is `1`, the record continues with the transfer policy:

| Field | Type | Notes |
| --- | --- | --- |
| `Collector` | `Address` | An account address that receives the team share. Fixed at creation. Must accept the asset before any transfer, while `TeamRate.Current` is above 0 |
| `TeamRate` | 3 x `uint16` | `Current`, `Min`, `Max`, in basis points. `Min` and `Max` are fixed at creation |
| `BurnRate` | 3 x `uint16` | `Current`, `Min`, `Max`, in basis points. `Min` and `Max` are fixed at creation |
| `TeamCap` | `int64` | Largest team share of one transfer entry. `MaxAssetSupply` means no cap. Fixed at creation |

A basis point is 1/10 000, so a rate of 10 000 is 100 %. Only `Current` ever changes after creation. A rate is
**fixed** when `Min == Max` and **variable** otherwise, so no separate flag is needed. An asset without a policy
behaves as if both rates were 0.

`HoldingRecord` (kind `2`), 42 bytes:

| Field | Type | Notes |
| --- | --- | --- |
| `Kind` | `uint8` | `2` |
| `AssetID` | `uint32` | The index of the `AssetRecord` |
| `Owner` | `Address` | An account address |
| `Balance` | `int64` | Units held, `0 <= Balance <= MaxSupply` of the asset. May be zero |
| `Deposit` | `int64` | The PAC locked when the record was placed. `AssetClose` returns exactly this amount |

`FreeRecord` (kind `3`), 5 bytes, the place of a closed holding:

| Field | Type | Notes |
| --- | --- | --- |
| `Kind` | `uint8` | `3` |
| `Next` | `uint32` | Index of the next free slot, or `NoRecord` |

The `AssetID` of an asset is the index of its `AssetRecord`. A record is hashed over its encoding, `Hash(Bytes)`. No
record has an encoding of 64 bytes, the length of a pair of hashes (a header has 13 bytes, a free slot 5, a
`HoldingRecord` 42 and an `AssetRecord` 82 to 134), so a leaf can never be taken for an inner node of the tree. There
is at most one `HoldingRecord` for a given `(AssetID, Owner)`. Only `AssetCreate` (for its signer, the creator) and
`AssetAccept` (for its signer) place one, so the signature of its owner is always behind a record. `AssetTransfer` and
`AssetMint` never place a record: they need the recipient's record to exist. Only `AssetClose` removes one, and only
for its owner. A node keeps a local index from `(AssetID, Owner)` to the record index. The index is not consensus state
and can be rebuilt from the tree.

**Free slots.** The free slots form a stack: `FreeHead` is the slot freed last, and the `Next` of each slot is the
slot freed before it. When a holding is placed, it takes the slot at `FreeHead` if there is one, and `FreeHead` becomes
the `Next` of that slot; otherwise it is appended at the next index. When a holding is closed, its record becomes a
`FreeRecord` whose `Next` is the old `FreeHead`, and `FreeHead` becomes its index. An `AssetRecord` is always appended
and never takes a free slot, so that an `AssetID` is never an index that has held another record; in an `AssetCreate`,
the `AssetRecord` is appended first and the holding of the creator is placed after it. Every node obtains the same
indexes, because the order of the transactions of a block is fixed. The tree keeps the length of its peak: after many
balances have been closed, their slots stay as 5-byte free records and wait for the next balance.

#### 2.2 State root

For a block below the version `V` that activates this PIP, the state root is unchanged. For a block of version `V` or
higher, the root that the block header commits to, which is the state before the block as today, becomes:

```text
assetRoot = assetMerkle.Root()          (the tree holds at least the header record)
stateRoot = HashMerkleBranches(HashMerkleBranches(accountMerkle.Root(), validatorMerkle.Root()), assetRoot)
```

Asset state MUST be part of the root: a store that the root does not cover would let two nodes commit the same block
with different balances. It is persisted in the same store batch as accounts and validators.

#### 2.3 Invariants

After every transaction, for every asset:

* `0 <= Burned <= Issued <= MaxSupply <= MaxAssetSupply`;
* the sum of its holdings equals `Issued - Burned`;
* `Issued` and `Burned` never decrease.

And for the tree:

* a holding is placed only by `AssetCreate` and `AssetAccept`, and only for the account that signed, and removed only
  by `AssetClose`, for its owner, after burning what is left of its balance, so a transfer or a mint never changes the
  number of records or the PAC locked, and nobody holds a record that it did not sign for;
* the `Deposit` of the holdings add up to `DepositTotal`, and the PAC that accounts have lost to deposits is exactly
  `DepositTotal`: balances, stake, treasury and `DepositTotal` are conserved by every asset transaction, the fee being
  paid where the fee of any other transaction goes;
* the free slots, followed from `FreeHead` by `Next`, are exactly the records of kind 3, each reached once, and the
  chain ends with `NoRecord`; the header is the only record of kind 0, at index 0.

### 3. Payloads

Six payload types are added. They take the next free numbers, 8 to 13, on the assumption that
[PIP-50](./pip-50.md) uses type 7; other Drafts also add payload types, so the editors renumber when assigning. No
test of this PIP depends on a type number.

```go
TypeAssetCreate   = Type(8)
TypeAssetTransfer = Type(9)
TypeAssetMint     = Type(10)
TypeAssetSetRate  = Type(11)
TypeAssetAccept   = Type(12)
TypeAssetClose    = Type(13)
```

#### 3.1 Rules common to all six

* `Signer()` is `From`. Only **account addresses** (BLS, Ed25519, secp256k1) are valid. Treasury and validator
  addresses MUST be rejected as `From`.
* The transaction is signed, expires (`LockTime` / TTL) and is protected against replay like every other
  transaction. No nonce is added. It pays the ordinary fixed fee and is not a free transaction.
* `Value()` is the PAC that the payload takes from `From` besides the fee (section 3.5): a charge that goes to the
  treasury, a deposit that is locked, or both. It is computed from the payload alone and subtracted from `From`
  together with the fee. Every sum that involves `Value()`, the fee, a deposit or a balance MUST be rejected before it
  exceeds `MaxNanoPAC` ([PIP-54](./pip-54.md)). `AssetClose` is the one payload that gives PAC back to `From`.
* Every asset quantity is an `int64` with `1 <= quantity <= MaxAssetSupply`, except the `InitialSupply` of
  `AssetCreate` and the `MaxBurn` of `AssetClose`, which may be 0. A sum of quantities MUST be checked before it is
  added, in the form `total > MaxAssetSupply - amount`.
* An asset transaction is an ordinary transaction, and it travels through the network and the pools as one. In a block
  it is carried only inside a bundle (section 4).
* A recipient is an account address. Treasury and validator addresses are rejected, because nothing could move the
  units from them. The only exception is the zero address as a recipient of `AssetTransfer` (section 3.3). Any other
  recipient MUST already hold a `HoldingRecord` of the asset, possibly with a balance of 0: it MUST have accepted the
  asset (section 3.7).
* `PAC(a)` is the PAC balance of the account `a`, in nano PAC. A holding has its own `Balance`, in units of the asset,
  and the two are never mixed: a check on PAC always says `PAC(From)`.
* `Check` reads the state and changes nothing. `Execute` applies the whole transaction or none of it.

#### 3.2 `AssetCreate` (type 8)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | The creator, who becomes the issuer |
| `Symbol` | `uint8` length, then bytes | 1 to 12 bytes of `A-Z0-9`, and not `PAC` |
| `Decimals` | `uint8` | At most `MaxDecimals` |
| `InitialSupply` | *varint* `int64` | `0 <= InitialSupply <= MaxSupply` |
| `MaxSupply` | *varint* `int64` | `1 <= MaxSupply <= MaxAssetSupply` |
| `MetaHash` | 32 bytes | Opaque to consensus |
| `PolicyPresent` | `uint8` | `0` or `1` |
| `Policy` | as in section 2.1 | Only if `PolicyPresent` is `1`: `Collector`, `TeamRate`, `BurnRate`, `TeamCap` |

The record of the creator is placed if `InitialSupply > 0`, or if the policy is present and `Collector == From`.
`Value()` is `AssetCharge`, plus `HoldingDeposit` if the record of the creator is placed. `BasicCheck` of a policy:
`Collector` is an account address; for each rate, `Min <= Current <= Max <= 10 000`;
`TeamRate.Max + BurnRate.Max <= 10 000`; at least one `Max` is above 0; `1 <= TeamCap <= MaxAssetSupply`. `Check`:
`PAC(From) >= Value + fee`, and the tree has room for the records that are appended (section 2.1).

`Execute`: debit `Value + fee`; credit `AssetCharge` to the treasury; append an `AssetRecord` with `Issuer = From`,
`Issued = InitialSupply`, `Burned = 0` and the policy as given; if the record of the creator is placed, place a
`HoldingRecord` of `From` with `Balance = InitialSupply` and `Deposit = HoldingDeposit`, and add `HoldingDeposit` to
`DepositTotal`. No record is placed for any other account, the collector
included: the collector is named by the creator but has not signed, so it must accept the asset itself (section 3.7).
Until it has, no `AssetTransfer` of this asset is valid as long as `TeamRate.Current` is above 0 (section 3.3). A
creator with `InitialSupply == 0` that is not the collector holds no record yet: to mint to itself, it accepts the asset
like anyone else.

The asset is **minting** if `MaxSupply > InitialSupply`. If they are equal, nobody can ever create another unit and
nobody holds any authority over its supply: the supply is fixed for ever. Symbols are labels, not names: they are not
unique, and the `AssetID` is the identity.

#### 3.3 `AssetTransfer` (type 9)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | The holder |
| `AssetID` | `uint32` | |
| `MaxRate` | `uint16` | Largest `TeamRate.Current + BurnRate.Current` that the sender accepts, in basis points |
| `NumberOfRecipients` | *varint* | `1 .. MaxRecipients` |
| `Recipients[].To` | `Address` | An account address, or the zero address to burn |
| `Recipients[].Amount` | *varint* `int64` | The gross quantity taken from the sender |

`Value()` is 0: a transfer appends no record, so it pays only the fixed fee.

**Rates.** With a policy, each entry whose `To` is not the zero address is split as follows, with `D = 10 000`, the
current rates of the asset, and integer division throughout:

```text
share(a, r) = (a / D) * r + ((a % D) * r) / D
team        = min( share(Amount, TeamRate), TeamCap )
burn        = share(Amount, BurnRate)
net         = Amount - team - burn
```

`share(a, r)` is exactly `floor(a * r / D)`, written so that no intermediate value leaves the `int64` range: since
`r <= D`, `(a / D) * r <= a`, and `(a % D) * r < D * D`. No 128-bit product is needed. Because the two rates add up to
at most `D`, `team + burn <= Amount`, so `net >= 0`. Rounding is down, so a transfer too small to owe one unit of rate
owes nothing. Without a policy, `team` and `burn` are 0 and `net` is `Amount`. An entry to the zero address pays no
rate: the whole `Amount` is burned.

`BasicCheck`: `MaxRate <= 10 000`; the number of recipients is in range; no `To` appears twice; no `To` equals `From`;
every amount is in range; the sum of the amounts, checked as in section 3.1, does not exceed `MaxAssetSupply`.
`Check`: the asset exists; `TeamRate.Current + BurnRate.Current <= MaxRate` (both are 0 without a policy); if the asset
has a policy and `TeamRate.Current` is above 0, `Collector` has a holding of the asset, that is, it has accepted it;
`From` has a holding of at least `total` units; every `To` that is not the zero address has a holding of the asset;
`PAC(From) >= fee`; no entry to an account has `net == 0`. When `TeamRate.Current` is 0, no team share is due and
the collector is not involved.

`Execute`: debit the fee in PAC; subtract `total` from the holding of `From` (the holding stays, even at zero, until
its owner closes it); then, for each recipient in the order of the list: if `To` is the zero address, add `Amount` to
the `Burned` of the asset; otherwise add `net` to the holding of `To`, add `team` to the holding of `Collector` if
`team > 0`, and add `burn` to `Burned`. If `team > 0` then `TeamRate.Current` is above 0, so the holding of
`Collector` exists, by `Check`. Each addition applies to the live record, so the result is the same when addresses
coincide. No record is placed.

For example, with `Amount = 1 000 000`, `TeamRate = 200` (2 %), `BurnRate = 50` (0.5 %) and `TeamCap = 15 000`:
`share(Amount, 200) = 20 000`, so `team = 15 000`; `burn = 5 000`; `net = 980 000`. With `Amount = 99` and the same
rates, `team = 1`, `burn = 0` (0.495 rounds down) and `net = 98`. With `Amount = 49`, both shares round to 0 and the
recipient gets all 49.

`MaxRate` exists because a rate can change between the moment a user signs and the moment the transaction is included
(section 3.6). A wallet MUST set it to the sum of the current rates that it showed, and MUST NOT set it higher without
the explicit consent of the user: an issuer that sees a transaction with a loose `MaxRate` in the pool can raise the
rates just before it. The sender can never pay more than the bounds fixed at creation, and `MaxRate` narrows that
bound for one transaction. A transfer that fails because a rate moved is signed again with the new value.

A burn is a transfer to the zero address. It needs no payload of its own and cannot be undone: the units leave the
supply, and `Issued` is not lowered, so burning never makes room for a new mint.

#### 3.4 `AssetMint` (type 10)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | MUST be the `Issuer` |
| `AssetID` | `uint32` | |
| `To` | `Address` | An account address. Not the zero address |
| `Amount` | *varint* `int64` | |

`Value()` is 0. `Check`: the asset exists; `From == Issuer`; `To` has a holding of the asset; `Amount <= MaxSupply -
Issued`; `PAC(From) >= fee`. `Execute`: debit the fee; add `Amount` to `Issued`; add `Amount` to the holding of `To`. No
record is placed. A mint pays no transfer rate: it creates units, it does not move them.

The cap is on units ever issued, not on units alive. A holder can therefore read `MaxSupply` and know the largest
quantity that will ever exist. An asset with `MaxSupply == InitialSupply` rejects every mint.

#### 3.5 Charges and deposits

Two kinds of record cost PAC, and they are priced differently because they leave the state differently.

* An **asset record** is never removed: every holding refers to it by its index, and the index must keep its meaning.
  It is paid for once, by `AssetCharge`, which goes to the treasury and is not returned.
* A **holding** can be closed by its owner. It is paid for by `HoldingDeposit`, which is locked while the record
  exists and returned as it was posted, less the fee of the close, when `AssetClose` removes the record. The locked
  PAC belongs to no account and is counted in `DepositTotal`.

What each payload takes from `From` besides the fee is computed from the payload alone, so a wallet knows it before it
signs and nothing depends on the state:

| Payload | `Value()` |
| --- | --- |
| `AssetCreate` | `AssetCharge`, plus `HoldingDeposit` if the record of the creator is placed (section 3.2) |
| `AssetAccept` | `HoldingDeposit` |
| `AssetClose` | 0. The deposit comes back, less the fee (section 3.8) |
| `AssetTransfer`, `AssetMint`, `AssetSetRate` | 0 |

Only creation and acceptance place records, so only they take PAC, and each record is paid for by the account that
signed for it. A charge or a deposit is always for a record that is in fact placed.

**Where the PAC goes.** This PIP changes nothing about the fixed fee. Every asset transaction pays it exactly as a
`Transfer` does, and it goes wherever the fee of any other transaction goes, whatever that path is or becomes. Only
`AssetCharge` is explicitly credited to the treasury, and never to the proposer, so that a validator cannot refund the
cost of the assets it creates itself. A deposit is not paid to anyone: it is locked, and it comes back to the account
that posted it, never to a proposer. Explorers that audit the supply MUST count `DepositTotal` next to balances and
stake, as they count the locked deposit of an anchor ([PIP-50](./pip-50.md)); leaving it out looks like a burn. The
rates of section 3.3 are a different thing: they are quantities of the asset, set by its issuer, and PAC is not
involved.

#### 3.6 `AssetSetRate` (type 11)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | MUST be the `Issuer` |
| `AssetID` | `uint32` | |
| `NewTeamRate` | `uint16` | The new `TeamRate.Current`, in basis points |
| `NewBurnRate` | `uint16` | The new `BurnRate.Current`, in basis points |

`Value()` is 0. `Check`: the asset exists and has a policy; `From == Issuer`; `TeamRate.Min <= NewTeamRate <=
TeamRate.Max` and `BurnRate.Min <= NewBurnRate <= BurnRate.Max`; if `NewTeamRate` is above 0, `Collector` has a
holding of the asset, that is, it has accepted it; `PAC(From) >= fee`. `Execute`: debit the fee and set
`TeamRate.Current` and `BurnRate.Current` to the new values. Nothing else about the policy ever changes: not the
bounds, the cap or the collector. The new rates apply to every transfer included after this transaction in the same
block or later. A transaction that sets the current values again does nothing and obeys the same rules. There is no
nonce, so two `AssetSetRate` signed one after the other may be included in either order; an issuer that wants a given
final value waits for the first to be included before it signs the second.

An asset whose bounds satisfy `Min == Max` for both rates has fixed rates and can never be changed; its issuer holds no
authority over them. The bounds are a promise that anyone can read in the state, as `MaxSupply` is.

This is also the way out when the collector of an asset cannot or does not accept it: if `TeamRate.Min` is 0, the
issuer sets `TeamRate` to 0, no team share is due, and transfers are valid again without the collector (section 3.3).
If `TeamRate.Min` is above 0 there is no way out but the acceptance of the collector.

The rule on `NewTeamRate` above 0 means that `AssetSetRate` can never block an asset: it cannot raise the team rate
while the collector has not accepted, and the collector cannot close its balance while the rate is above 0 (section
3.8). The only state in which transfers are blocked for lack of the collector is the one in which `AssetCreate` left
the asset, with a team rate above 0 and a collector that has not accepted. A collector that closed its balance at a team
rate of 0 must accept again before the rate can be raised.

#### 3.7 `AssetAccept` (type 12)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | The account that opts in. An account address |
| `AssetID` | `uint32` | |

`Value()` is `HoldingDeposit`. `Check`: the asset exists; `From` has no holding of the asset yet;
`PAC(From) >= Value + fee`; if there is no free slot, the tree has room for one more record. `Execute`: debit
`Value + fee`; add `Value` to `DepositTotal`; place a `HoldingRecord` of `From` for the asset with `Balance = 0` and
`Deposit = Value` (section 2.1).

Only the account itself can accept, because only its signature is valid. From then on it can receive transfers and
mints of the asset. There is no way to refuse a transfer once the record exists: an account that does not want an
asset simply never accepts it, and an account that accepted one by mistake closes the balance (section 3.8) and
gets its deposit back. Accepting twice is rejected, so nobody locks a deposit twice for the same record.

A sender cannot open a record for someone else, and cannot pay for it either. This has no exception: the collector that
a policy names accepts like anyone else, and while it has not, no transfer of the asset is valid as long as the team
rate is above 0. Only the issuer who named it is affected, so an account that is named as a collector against its will
is not harmed and is not obliged to do anything. An issuer that wants to distribute an asset publishes the `AssetID`,
lets the recipients accept, and then sends the units, in one batch of up to 8 per transaction. A wallet that wants to
receive an asset asks its user, then signs an `AssetAccept`.

#### 3.8 `AssetClose` (type 13)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | The owner of the holding. An account address |
| `AssetID` | `uint32` | |
| `MaxBurn` | *varint* `int64` | Most units that the owner agrees to burn with the close. `0 <= MaxBurn <= MaxAssetSupply` |

`Value()` is 0. `Check`: the asset exists; `From` has a holding of the asset with at most `MaxBurn` units; the asset
does not have a policy whose `Collector` is `From` while `TeamRate.Current` is above 0; and, with `Deposit` the amount
posted in the record, `PAC(From) + Deposit >= fee` and `PAC(From) <= MaxNanoPAC - Deposit + fee`, which has no
negative step ([PIP-54](./pip-54.md)). `Execute`: add the units of the holding to the `Burned` of the asset, with no
rate, as for a transfer to the zero address; set `PAC(From)` to `PAC(From) + Deposit - fee`; subtract `Deposit` from
`DepositTotal`; turn the record into a `FreeRecord` and push it on the free stack (section 2.1). The units that were
burned are gone with the record, so the sum of the holdings still equals `Issued - Burned`.

The fee is taken from the refund, so that an account that has locked all its PAC can still close, as with the deposit
of an anchor in [PIP-50](./pip-50.md). The refund is the `Deposit` recorded when the holding was placed, not the
current `HoldingDeposit`, so a later change of that constant touches no existing record.

Only the owner can close, because only its signature is valid. `MaxBurn` makes a close safe in both directions. With
`MaxBurn` at 0 the close needs an empty balance, and it fails if units have arrived since the owner signed, so it never
destroys a payment that the owner did not know about. With a larger `MaxBurn` the owner says, in its own signature,
how many units it agrees to lose: that is how it leaves an asset that it no longer wants, or ignores the dust that
someone sent it to stop it from closing, with one transaction and one fee. A holder that wants to keep an owner from
closing must therefore send more units than the `MaxBurn` that the owner chooses, and those units are burned, so the
grief is paid for with the attacker's own units. A wallet MUST show the units that a close burns before it signs.

The collector of an asset cannot close its balance while the team rate is above 0, because every transfer needs that
balance and closing it would block the asset for every holder. When the team rate is 0 it may close, and then
`AssetSetRate` cannot raise the rate again until it has accepted the asset again (section 3.6).

A closed slot is reused by the next holding that is placed, so closing and accepting again does not grow the tree.

### 4. Bundles and block capacity

#### 4.1 Bundles

A transaction cannot be extended by anyone but its signer, because its signature covers the whole payload. So the node
never edits a transaction to add a transfer to it. It packs whole transactions, each with its own signature, into a
**bundle**: an entry of the list of transactions of a block that holds complete asset transactions and has no sender,
fee, lock time or signature of its own. These rules are consensus rules:

1. In a block, asset transactions (the six types above) appear only inside bundles, never as entries of the list on
   their own.
2. A bundle holds 1 to `MaxBundleItems` asset transactions, and nothing else.
3. In a block, at most one bundle holds fewer than `MaxBundleItems` items. A proposer with `n` asset transactions to
   include therefore uses `ceil(n / MaxBundleItems)` slots, and cannot waste slots on half-empty bundles.
4. A block carries at most `MaxAssetTxPerBlock` asset transactions in all.
5. A block with a bundle means the same as the block in which the bundle is replaced, in place, by its items in order.
   Each item is checked and executed as an ordinary transaction at that position: `BasicCheck`, `Check`, `Execute`, its
   signature, its lock time, the replay check on its own transaction ID, and its fee. An invalid item makes the block
   invalid.
6. Items keep their ordinary transaction IDs. The ID of a bundle is
   `Hash("PAC-ASSET-BUNDLE-1" || ID(item 1) || ... || ID(item n))`, and the transaction root of the block is computed
   over the entries of its list, using the bundle ID for a bundle. The root therefore commits to the packing, and
   nobody can repack a block without changing its hash. A proof that an item is in a block is a proof of its bundle
   and the list of at most 8 IDs.
7. A transaction ID appears at most once in a block: not twice in one bundle, not in two bundles, and not both in a
   bundle and as another entry of the list. A block that repeats one is invalid, so a transaction cannot be run twice
   in the same block.

The wire encoding of a bundle belongs to the reference implementation. It MUST be deterministic and carry only a count
and the items.

#### 4.2 Packing

The proposer does the packing, and no user does anything. It takes asset transactions from its asset pool in arrival
order, skips any that do not pass `Check` against the state of the block under construction (and keeps them in its
pool, since an earlier item of the same block can make a later one valid: an `AssetAccept` before a transfer to the
account that signed it, so a proposer SHOULD try the skipped ones again after the others), fills bundles of
`MaxBundleItems`, and stops at `MaxAssetTxPerBlock`. When the first bundle is full it starts a second, and so on; the
last one may be partial. It SHOULD place bundles after the other transactions. What does not fit stays in the pool for
the next block, until its lock time expires, and the signer then signs it again.

#### 4.3 Capacity

Two limits bound two different resources.

* **Slots.** A bundle is one entry of the 1 000-transaction list. Whatever the senders, asset transactions take at most
  `ceil(MaxAssetTxPerBlock / MaxBundleItems)` slots, 25 with the values above, which leaves PAC at least 97.5 % of the
  slots of a block. Rules 1, 3 and 4 make this a guarantee and not a policy.
* **Work.** A bundle saves slots and the header of each transaction, not signature checks: every item is verified,
  executed and written to the state as it would be alone. `MaxAssetTxPerBlock` is what bounds this work and the growth
  of the state, and it is the number that a benchmark must set (Reference Implementation). Any limit that a block has
  in bytes applies to the bytes of the items, so that a bundle cannot be used to get around it.

This is a consensus rule and not a fee market: whatever the demand for assets, it cannot take more than these slots
and this work from a block.

### 5. Node changes

* `Sandbox` exposes reads and writes of asset records, and the node adds the tree, its store prefix and the extended
  root. `executeBlock` does not change: there is no per-block hook.
* The local index from `(AssetID, Owner)` to a record is a cache, and execution is defined by the tree. A node MUST
  rebuild or verify it after an unclean shutdown, and MUST stop, not continue, when a lookup disagrees with the tree:
  two nodes with different indexes would disagree on the validity of a block.
* Block validation enforces the rules of section 4.1 and computes the transaction root over the entries of the list.
  The block body gains one kind of entry, the bundle. Store indexes (transaction by ID, replay check) work on the
  items, so a transaction is found and replay-protected whether or not it sits in a bundle.
* Block building packs bundles as in section 4.2. The gossip of transactions does not change: nodes relay individual
  asset transactions, and a bundle exists only inside a block.
* Transaction pool: a pool is local policy, not consensus. Implementations SHOULD give the six types one pool of 10 %
  of `MaxSize`, like `BatchTransfer`, so asset transactions cannot crowd out payments and payments cannot evict them.
  Fee estimation uses the same fixed fee as `Transfer`, plus `Value()`. A pool MAY drop an `AssetTransfer` or
  `AssetMint` whose recipient has not accepted the asset, and an `AssetTransfer` of an asset whose team rate is above
  0 and whose collector has not accepted it, but only when no transaction in the pool can make it valid (an
  `AssetAccept` of that recipient or collector, or an `AssetSetRate` to a team rate of 0), since a proposer packs both
  in one block (section 4.2). A pool SHOULD run the checks in the order of their cost: `BasicCheck`, then the PAC
  balance (for an `AssetClose`, the PAC balance plus the deposit that it returns, since it pays its fee from the
  refund), then the signature, then the asset-state `Check`. It SHOULD also limit the number of pending asset
  transactions per sender, because each one can be valid alone against the current state while all together are not
  (several transfers of the same units), and without a limit one funded account can fill the pool with transactions
  that the proposer will skip.
* RPC (mandatory for the official node, on every surface it ships): payload type values `8` to `13` in `PayloadType`;
  `GetRaw*Transaction` builders for the six payloads; `GetAssetState`, returning `DepositTotal`, `FreeHead` and the
  number of records of the tree; `GetAsset`, returning the fields of section 2.1, the policy with its current rates
  and bounds and whether its collector has accepted it (an asset whose team rate is above 0 cannot be transferred
  before), and the live supply `Issued - Burned`; and `GetAssetBalance(AssetID, Address)`, which tells an account that
  has not accepted the asset, or has closed it, from one that holds a balance of 0, so that a sender can check before
  it signs. A node SHOULD also offer a list of the balances of an address from its local index. `GetBlock` and
  `GetTransaction` return the items of a bundle as ordinary transactions, so a wallet or an explorer does not need to
  know about bundles; they MAY also report the bundle of each item.
* Explorers that audit the supply MUST add `DepositTotal` to balances, stake and the treasury: the PAC locked by
  deposits belongs to no account, and leaving it out looks like a burn. `AssetCharge` goes to the treasury, and every
  other PAC amount keeps its place.

### 6. Conventions (informative)

Nothing here is checked by consensus. Wallets and explorers agree on it so that one record can serve several uses.

| Use | `Decimals` | `MaxSupply` | `InitialSupply` | Notes |
| --- | --- | --- | --- | --- |
| Fungible token | 0 to 9 | any | any | `MaxSupply == InitialSupply` for a fixed supply |
| NFT | 0 | 1 | 1 | One unit. Its `AssetID` is its identity |
| Semi-fungible edition | 0 | N | N | N identical units. A smaller `InitialSupply` leaves room for mints |

* `MetaHash` is the hash of a document that gives the name, description, image and attributes. A wallet fetches the
  document from where the issuer published it and checks the hash. The memo of the `AssetCreate` transaction MAY carry
  a pointer to it, within the memo length.
* A collection is a list of `AssetID`s that the issuer publishes off-chain, for example in a document committed by a
  [PIP-50](./pip-50.md) anchor. The protocol has no collection object.
* A wallet shows the **net** amount that a recipient will get, not only the gross `Amount`, and the bounds of the
  rates, not only their current values. A royalty or a tax on units of an asset, paid to a collector, is expressed with
  `TeamRate`; a deflationary asset is expressed with `BurnRate`. Rates are meant for fungible assets: on an asset of
  supply 1 every share rounds to 0. An issuer that wants its team share paid to another account names that account as
  `Collector` and makes it accept the asset before the first transfer (unless the team rate starts at 0), or names
  itself, which needs no extra step, and forwards the share when it wants.
* Receiving an asset takes two steps: the recipient accepts, then the sender sends. A wallet asks its user before it
  signs an `AssetAccept`, and MAY offer the acceptance as a link or a QR code that carries the `AssetID`. A
  distribution to many accounts is a claim: the issuer publishes the `AssetID`, accounts accept, and the issuer sends
  to those that did, 8 at a time. A sale of an NFT starts with the buyer accepting it. A wallet shows the deposits as
  locked PAC, never as spendable PAC, and SHOULD offer to close the balances that the user no longer wants, which
  returns their deposits, showing the units that each close would burn.

## Rationale

**On-chain, because that is the simplest thing that is also fully trusted.** An asset rule that every validator
checks adds no assumption to Pactus. An earlier draft of this PIP ran the assets off-chain under a bonded committee
that checkpointed them on Pactus. That keeps L1 capacity free, but it adds operators, bonds, slashing, epochs and
quorums, and its correctness and its data availability rest on a federation that Pactus cannot check. The hard cap of
section 4 answers the same worry about L1 capacity with one counter.

**A cap, not a market.** A fee market needs a price that nobody knows before the layer is used. A cap on the number
of asset transactions in a block needs no guess, is trivial to validate and gives PAC a floor that no demand can
lower. Its price is that asset users can be delayed when asset demand reaches the cap. That is acceptable, and the
cap is one constant that a later version can raise. With bundles the cap counts the transactions that carry work, and
the slots follow from it.

**One record for every kind of asset.** A token-ID dimension (items inside a collection) needs item records, per-item
supply and more rules for each type. Here an NFT is an asset with a supply of one. The cost is that a collection of N
items is N creations, each priced by the fee and `AssetCharge` and paced by the cap. A later PIP can add collections if
there is demand, without changing these records.

**Two levels of batching.** `BatchTransfer` already shows that up to 8 recipients are a safe size. Inside
`AssetTransfer`, one signature then moves up to 8 transfers, which is the only batching that really lowers the work,
since it saves signatures. The single-recipient transfer is the case of one. Above it, bundles let the proposer put
transactions of different senders into one slot, so that the slots of a block are not the limit of asset traffic.

**The node packs whole transactions.** A node cannot add a recipient to a transaction that someone else signed, and
account signatures do not merge: BLS aggregation exists only for BLS accounts and, for different messages, still costs
one pairing per message. So the bundle holds complete signed transactions and nothing new is trusted: each item is
checked as it would be alone, and a bundle is a compression of the list of transactions. Packing is mandatory and
full, so that a proposer cannot use half-empty bundles to take slots from PAC. The cost is a new kind of entry in the
block body, which is the most delicate part of this PIP for the node authors.

**What a bundle does not save.** Bundles do not reduce signature checks, state writes or, by more than the header of
each transaction, bytes. They free slots. The honest bound on the work of assets is therefore `MaxAssetTxPerBlock`, set
by a benchmark, and not the number of slots.

**Burn is a transfer.** It saves a payload type, a pool and an RPC builder, and the semantics are unambiguous. The
cost is that a transfer to the zero address by mistake cannot be undone, which wallets guard against.

**Mint capped at creation.** Issuers need minting (editions, game items, a stablecoin). Holders need a bound. A cap
that is fixed at creation, and that burning does not reopen, is a promise that anyone can read in the state.
`MaxSupply == InitialSupply` is the strongest form and has no mint authority at all. Transferring or renouncing the
issuer role is left out: it adds a payload and a way to lose control by mistake.

**No allowance.** The approve and spend-from pattern is where most user losses on token chains happen. Leaving it out
keeps the trust model simple: only the holder can move units, and nobody else can ever be allowed to.

**Transfer rates: variable, but inside bounds that cannot move.** Issuers of fungible assets ask for a share of each
transfer (a team or royalty share) and for a burn. A rate that nobody can change is too rigid, since an issuer may need
to lower it, and a rate that its issuer can change freely is a trap. So each rate has a `Min` and a `Max` fixed at
creation, public in the state like `MaxSupply`, and the issuer moves only the current value between them. `Min == Max`
gives a rate that is fixed for ever, with no authority at all. The calculation is three lines of integer arithmetic,
rounds down, never leaves the `int64` range, and a sum of the two rates of at most 100 % means `net` is never negative.
`MaxRate` in the transfer protects the signer against a change that lands before inclusion. The rates act on units
of the asset only and have nothing to do with PAC fees.

**One issuer for mint and rates.** An earlier draft had a separate parameter controller, a per-parameter mutable flag
and a minimum received per transfer. Here `Min < Max` already says that a rate is variable, one `MaxRate` per
transaction replaces the minimum per recipient, and the issuer is the only authority. A separate controller (a
multi-signature or a vote) can be added later without changing the records.

**Opt-in, so that nobody is given an asset against their will.** Anyone can create an asset with any symbol, and
without opt-in anyone could put it in any wallet for a fee that is small in PAC: look-alike symbols, links in
metadata, and a record added to every victim's state. With opt-in, an asset reaches an account only if that account
signed for it. It also simplifies the rules. A transfer or a mint never places a record, so their execution has no
creation order to fix, their charge is 0, and a holding is placed only when its owner asks and locks a deposit. The
cost is two steps to receive an asset, and a recipient that needs a little PAC for the deposit before it can accept.
To the knowledge of the author, the XRP Ledger, Stellar and Algorand also ask the receiver to opt in. There is no
exception: the creator's own record is placed by the creation it signed, and the collector named by a policy must
accept like anyone else. Creating the collector's record at creation would let an issuer put an asset on any account
by naming it. Instead, a transfer of an asset whose team rate is above 0 is invalid until the collector accepts, which
costs nobody but the issuer that named it. The condition is on the rate and not on the existence of a policy so that
the issuer has a way out: with `TeamRate.Min` at 0, setting the rate to 0 removes the team share, and with it the need
for a collector that may never accept. `AssetSetRate` obeys the same rule, so that the issuer cannot block its own
asset by raising the rate before the collector has accepted.

**A refundable deposit and a free list, so that unused balances are reused.** The XRP Ledger, Stellar and Algorand (to
the knowledge of the author) lock a reserve that the holder gets back when it closes its line, and
[PIP-50](./pip-50.md) locks a deposit for an anchor that its owner gets back when it deletes it. A deposit bounds what
is in use by the PAC that people accept to lock, instead of by what was ever created. The tree of Pactus only appends,
so a closed balance is not cut out: its record becomes a free slot on a stack, the next balance reuses it, and no
index moves. A node never has to shrink the tree or move a record. The tree keeps the length of its peak, with free
slots of 5 bytes in place of the closed balances, so the deposits bound the balances in use and the price of the
length of the tree is the fees, as it is for accounts. The deposit is refunded as it was posted, so a change of the
constant never touches an existing record, and the fee is taken from the refund so that nobody is stuck with all its
PAC locked. Asset records are the exception: they are permanent, since every holding points to one, and they are paid
for once, which is why `AssetCharge` is not small. Their cost is also what an attacker pays to make the permanent part
of the state grow, which is the only part that deposits cannot reclaim.

**Charges known from the payload.** `Value()` is computed from the payload alone, so a wallet never has to guess a fee
from the state, and no fee code depends on the state. Because only creation and acceptance place records, and which
records they place follows from the payload, each charge or deposit is for a record that is in fact placed.

**PAC pays everything.** No second fee token and no sponsor. A new holder needs a little PAC for the deposit and the
fee to accept an asset, and for the fee to move it.

## Alternatives Considered

1. **Off-chain execution checkpointed by a bonded committee** (an earlier draft of this PIP). Rejected as too complex
   and for its trust assumptions. It remains a possible later PIP if asset demand outgrows the cap.
2. **A profile over [PIP-50](./pip-50.md) anchors, computed by indexers.** No consensus change, but Pactus enforces
   nothing: validity is whatever the indexer says.
3. **A smart-contract VM.** Attack surface out of proportion to the tokens this PIP targets.
4. **Item records inside an asset (collections).** Set aside: more state and rules, and the single record covers the
   need at the cost of more creations.
5. **Shrinking the tree when a balance is closed**, by moving the last record into the hole and dropping the last
   leaf. It would give the space back for real, but it needs a tree that can drop its last leaf, which the trees of
   Pactus do not do to the knowledge of the author, and a second tree so that an `AssetID` never moves. The free list
   bounds the records in use, and the length of the tree through its peak, with the tree that exists.
6. **Fees in the assets themselves.** No common unit, and no way for a new holder to pay a first fee. The rates of
   section 3.3 are not fees of this kind: they are a property of the asset that its issuer chose, and they do not pay
   for the transaction.
7. **The node merges transfers of different senders into one transaction with many senders.** Not possible with
   account signatures, which cover one payload each. Bundles keep the signatures and merge only the slot.
8. **Asset transactions flat in the block, with a cap on their number.** Simpler for the block body, but each one
   takes a slot, so 200 of them would take a fifth of a block, and the cap would have to be small. A version can
   still set `MaxBundleItems` to 1 to get one slot per transaction.
9. **Rates that are fixed, or that have no bounds.** Fixed rates are possible (`Min == Max`) but not the only choice.
   Unbounded rates would let an issuer take any share of any transfer.
10. **The sender pays for the recipient's record, with no opt-in** (to the knowledge of the author, the model of
    Cardano and Solana, with a refundable reserve in each). Simpler to use, but anyone can put an asset in any wallet,
    and the recipient has no say in the record that is created for it. Set aside for the opt-in of section 3.7.
11. **Higher charges and wallets that hide unknown assets.** No consensus change, but it prices spam instead of
    stopping it, and the records stay in the state. It remains a rule for wallets in any case.
12. **Balances that are never closed, paid for once.** The simplest rule, but the state only grows and its owners get
    nothing back, which is weaker than the reserves of the chains above.

## Backwards Compatibility

This is a consensus upgrade. Nodes that do not implement version `V` reject payload types 8 to 13 and cannot compute
the extended state root. Existing payload types, account records and validator records keep their encoding. Accounts
that never use the feature are unaffected.

Activation follows [PIP-51](./pip-51.md): implementations advertise version `V`; when more than 75 % of committee power
supports it, proposers raise the block version; from the first block of version `V` the extended root applies, the
asset tree exists with its header record, the six types are legal and a block body may carry bundles, with its
transaction root computed over the entries of its list. Before that block the types MUST be rejected as invalid payload
types, a bundle is an invalid entry, and the asset tree does not exist. Software that reads block bodies (explorers,
indexers, light clients) must learn the bundle. State sync and snapshots MUST carry the asset tree from then on.
Testnet SHOULD activate first.

## Test Cases

Implementations MUST pass at least these. They add no rule to the Specification.

* **Activation and root.** The six asset payload types are rejected below `V`. The first block of `V` has the
  extended root, with an asset tree that holds only the header (`FreeHead` at `NoRecord`, `DepositTotal` 0). The root
  changes when a record changes, and a restart reloads the same root.
* **Create.** A fixed asset, a minting asset and an asset with `InitialSupply == 0` are created. Rejected: an empty
  symbol, 13 bytes, lowercase, a symbol of `PAC`, `Decimals` 10, `MaxSupply` 0 or above `MaxAssetSupply`,
  `InitialSupply` above `MaxSupply`, insufficient PAC, a treasury or validator `From`. `AssetCharge` reaches the
  treasury and is never returned. The creator's holding is placed, with a deposit that raises `DepositTotal`, only if
  `InitialSupply > 0` or the policy names the creator as collector. The asset record is always appended at the end,
  even when there is a free slot. With a policy, rejected: `Current` outside `[Min, Max]`, a `Max` above 10 000,
  `TeamRate.Max + BurnRate.Max` above 10 000, all `Max` at 0, `TeamCap` of 0, a validator `Collector`. With a policy
  that names the creator as collector, the creator's holding exists right after creation, at balance 0 if
  `InitialSupply` is 0, and is placed once. With a policy that names another account as collector, no record is
  placed for that account, and its PAC balance is unchanged.
* **Accept.** An account accepts an asset: a holding with balance 0 is placed, `HoldingDeposit` leaves its PAC balance
  and raises `DepositTotal`, nothing reaches the treasury, and the account can then receive. With no free slot the
  record is appended at the next index; with one, it takes the slot at `FreeHead`. Rejected: an unknown asset, a second
  `AssetAccept` of the same asset by the same account (nothing is locked twice), insufficient PAC, a treasury or
  validator `From`. A creator that holds a record from creation (an initial supply, or itself as collector) cannot
  accept again. An accept and a transfer to that account in one bundle, in that order, both succeed; in the other order
  the transfer makes the block invalid. A collector other than the creator that accepts makes transfers of the asset
  valid.
* **Close and free slots.** An owner closes its holding: its PAC balance rises by the `Deposit` posted in the record,
  less the fee, `DepositTotal` falls by the same `Deposit`, and the record becomes a `FreeRecord` that is the new
  `FreeHead`. With `MaxBurn` at 0 the balance must be 0. With a `MaxBurn` at least the balance, the units are added to
  `Burned`, no rate applies, and the sum of the holdings still equals `Issued - Burned`. Rejected: an unknown asset, an
  account with no holding, a balance above `MaxBurn` (even by one unit), another account's holding, the collector while
  `TeamRate.Current` is above 0, a `MaxBurn` above `MaxAssetSupply`, and a refund that would take the PAC balance above
  `MaxNanoPAC`. An account with 0 PAC and a deposit that covers the fee can close. After two closes, two accepts take
  the slots in the reverse order of the closes, the tree grows by no record, and `FreeHead` ends at `NoRecord`. A
  `HoldingDeposit` that a later version changes does not alter the refund of an old record. A closed account that
  accepts again gets a record that is unrelated to the old one. Units sent to an owner after it signed a close with
  `MaxBurn` at 0 make that close fail and cost the owner only the new signature; the same close with a `MaxBurn` above
  the dust goes through and burns it.
* **Transfer.** One recipient and eight succeed; nine fail. Rejected: a repeated `To`, `To == From`, a validator
  recipient, an amount of 0 or below, a `MaxRate` above 10 000, an unknown asset, a balance that is too small, a
  recipient that has not accepted the asset, an asset whose `TeamRate.Current` is above 0 and whose collector has
  not accepted it (even for a transfer made only of burns), and 8 recipients of `2^62` each (the overflow case of
  PIP-54), which fails before any addition. The same asset with `TeamRate.Current` at 0 accepts transfers although
  the collector has not accepted, and no team share is paid. A transfer to the zero address needs no holding, raises
  `Burned` and creates no record. A holding that reaches zero stays. `Value()` is 0, and the number of records of the
  tree is the same before and after any transfer.
* **Rates.** The example of section 3.3 gives `team = 15 000`, `burn = 5 000` and `net = 980 000`. With the same rates,
  99 gives `team = 1`, `burn = 0`, `net = 98`, and 49 gives `team = burn = 0`. A transfer of `2^63 - 1` with rates
  of 10 000 in total is computed without leaving the `int64` range, and `team + burn + net` equals `Amount` for a
  range of amounts and rates, including values just below and above multiples of 10 000. `TeamCap` limits `team` and
  not `burn`. An entry whose `net` is 0 is rejected. `MaxRate` below the current sum of the rates fails the transfer,
  and an asset without a policy accepts `MaxRate` 0. An entry to the zero address pays no rate. The recipient, the
  collector and `Burned` change as specified when the collector is the recipient or the sender.
* **Set rate.** The issuer sets a rate inside its bounds and the next transfer uses it. A value outside `[Min, Max]`, a
  non-issuer, an asset without a policy and an unknown asset fail. An asset with `Min == Max` accepts only that
  value. Bounds, cap and collector never change. With a collector that has not accepted and `TeamRate.Min` at 0,
  setting `TeamRate` to 0 makes transfers valid again, and setting it back above 0 fails until the collector
  accepts, after which it succeeds. With `TeamRate.Min` above 0, only the acceptance of the collector makes transfers
  valid, and an `AssetSetRate` that keeps `TeamRate` above 0 fails until then. An issuer that is its own collector is
  never blocked. When `TeamRate.Min` is 0, a team rate of 0 is accepted whatever the state of the collector.
* **Mint.** Only the issuer mints. A mint above `MaxSupply - Issued` fails. After a burn, a mint still cannot pass
  `MaxSupply` in total. An asset with `MaxSupply == InitialSupply` rejects every mint. A mint to an account that has not
  accepted the asset fails, including a mint by an issuer to itself when `InitialSupply` was 0 and it holds no record.
  A mint places no record and has `Value()` 0, and it is valid even if the collector has not accepted.
* **Bundles.** A block with 200 asset transactions in 25 full bundles and 975 other transactions is valid; 201 asset
  transactions are invalid. A block with 17 asset transactions in a bundle of 8, a bundle of 8 and a bundle of 1 is
  valid, and the same 17 in a bundle of 8, a bundle of 5 and a bundle of 4 is invalid (two partial bundles). Invalid: an
  empty bundle, a bundle of 9, an asset transaction outside any bundle, a bundle that holds a PAC `Transfer` or another
  bundle, and a bundle in a block below `V`. An item with a bad signature, an expired lock time, an ID that was already
  executed or a balance that is too small makes the block invalid. Two items of one sender in one bundle run in order.
  The state root after a block with bundles equals the one after the same block with the bundles replaced by their
  items. Repacking the same items gives a different bundle ID and a different transaction root. A transaction is found
  by `GetTransaction` with its own ID, and a transaction ID that is already in a bundle is rejected a second time.
* **Invariants.** The invariants of section 2.3 hold after every test, free list included. Balances, stake, treasury and
  `DepositTotal` sum to the PAC supply after every test: asset transactions only move PAC to the treasury, lock it in
  a deposit or give it back, and pay the fee.
* **Determinism.** Two nodes with different local indexes and one restarted from a pruned store obtain the same root
  and the same `GetAsset`. Every payload decodes and re-encodes to the same bytes, and trailing bytes are rejected.
* **Hostile inputs.** Invalid: a varint that is longer than its shortest encoding, 9 recipients, a symbol length of
  13, a bundle count of 0 or 9, each rejected before any allocation of that size. A block that repeats a transaction
  ID, in one bundle, in two, or as a bundle item and a top-level entry, is invalid. A transaction executed in an
  earlier block and included again is invalid. A signature made for a PAC transaction does not validate an asset
  payload. A node whose local index disagrees with the tree stops. A rate raised in the pool after a transfer was
  signed with a tight `MaxRate` makes that transfer fail, and does not charge more. Two `AssetSetRate` included in the
  opposite order of their signing leave the rates of the later inclusion. An `AssetAccept` and a transfer to the
  account that signed it, skipped in the first pass of a proposer, are packed in the same block after a second pass. No
  record encoding has a length of 64 bytes.

## Reference Implementation

None yet. Before this PIP leaves Draft, a pull request will add the six payloads, the asset tree, the extended root,
the bundles, the block cap and the RPC to the Pactus node, and report on stated hardware: the time to validate a block
with `MaxAssetTxPerBlock` asset transactions of 8 recipients each, for BLS, Ed25519 and secp256k1 senders, which is
what sets the cap; the cost of a tree update, the size of the store per record, and the effect on state sync.

### Open items of this draft

* **A reference implementation.** No part of this PIP has run inside a Pactus node.
* **Byte-level vectors** for each payload and for the extended root.
* **The bundle in the block body.** Its wire encoding and the transaction root over the entries of the list are the
  least settled part of this PIP and need the review of the node authors.
* **Constants.** Every value of section 1 is an initial proposal, not measured. `MaxAssetTxPerBlock` is set by the
  benchmark of the Reference Implementation.

## Security Considerations

### Arithmetic

[PIP-54](./pip-54.md) came from a sum that wrapped. Every quantity here is an `int64` bounded by `MaxAssetSupply`,
every sum is checked before it is added, and the invariants of section 2.3 are checked in tests. A credit cannot
overflow, because a balance never exceeds the supply of its asset. The test with 8 recipients of `2^62` is mandatory.
The same care applies to PAC: `Value() + fee` and every balance are bounded by `MaxNanoPAC` before the sum.

### State growth

The state has two parts, and they are bounded differently.

**Balances in use are bounded by the PAC that their owners lock.** A holding exists only while its owner keeps a deposit
of `HoldingDeposit` locked for it, and closing it returns the deposit and frees the slot, so the number of holdings in
use is at most the locked PAC divided by the deposit: with 42 million PAC, at most 420 million records of 42 bytes,
about 17.6 GB, and only if every coin were locked. The deposit is about 2.4 mPAC per byte, the same order as the anchor
of [PIP-50](./pip-50.md) (about 4.5) and a validator record (about 8.3). Transfers and mints never grow the state, and
nobody can make an account hold, or pay for, a record that it did not sign for.

**The length of the tree is priced by the fees, as it is for accounts.** The tree keeps the length of its peak, and the
deposit is returned, so a sender can fill the cap with `AssetAccept` (one 42-byte record per transaction), close all of
them, and leave a longer tree of free slots. At the cap that is 8 KB of leaves per block, about 73 MB a day, for about
17 000 PAC a day in fees and nothing burned. That is about 2.3 times the price per byte, and 8 times the price per
record, of the accounts that `BatchTransfer` creates today (0.01 PAC for 8 accounts of 12 bytes), which Pactus already
accepts. The deposit does not lower this price; it prices the time during which the records are in use, and it lets
honest users reuse the room that they free.

**Asset records are permanent, and they are paid for.** A sender that fills the cap with `AssetCreate` and no initial
supply adds 27 KB per block, about 1.7 million records and 232 MB a day, and spends about 190 000 PAC a day on
`AssetCharge` and fees; one gigabyte of asset records costs about 820 000 PAC. Today a sender can create up to 8 000
accounts per 1 000-transaction block with `BatchTransfer`, about 830 MB a day, so the permanent part is bounded below
what Pactus already accepts for accounts. `AssetCharge`, `HoldingDeposit` and the cap are constants that a later
version can change.

A proposer is the one attacker that this PIP cannot price out with the fixed fee: if that fee goes to the proposer
(the path is not changed here), a proposer can fill its own blocks with asset transactions at no net cost for the fee.
It still pays `AssetCharge` for every asset, which goes to the treasury and never back to it, it still locks a deposit
for every balance, and it still cannot exceed `MaxAssetTxPerBlock` per block, so the figures above are the bound for a
proposer too.

An account that accepts an asset by mistake closes the balance and gets its deposit back, less a fee. It cannot refuse
units that a holder sends it, but a close with a `MaxBurn` above those units goes through anyway, so dust cannot keep
an owner from recovering its deposit; the sender of the dust loses the units, and the owner loses nothing.

### Block capacity

The bundle rules keep assets to at most 25 slots of a block (section 4.3), so PAC keeps at least 97.5 % of them,
and a proposer cannot take more by packing badly, because at most one bundle may be partial. The **work** of assets is
bounded by `MaxAssetTxPerBlock` alone: a bundle does not lower the signatures to verify or the records to write, so that
number must come from the benchmark and not from the slot arithmetic. A spammer who fills the asset lane (200
transactions, at least 2 PAC a block in fees) delays other asset users and does not touch PAC. Pools are local policy,
and the 10 % pool keeps asset transactions from filling the memory of a node at the expense of payments. The ID of a
bundle commits to its packing, so a relay cannot repack the body of a block without changing its hash, and a block
whose packing breaks section 4.1 is invalid even if every item is valid.

### The issuer

A minting asset gives its issuer the power to create units up to `MaxSupply`. A wallet MUST show whether an asset is
minting and how much room is left, before the user buys it. The issuer key cannot be replaced: if it is lost, no more
units are minted and the rates stay where they were; if it is stolen, the thief can mint up to the cap and move the
rates inside their bounds. An asset with `MaxSupply == InitialSupply` and fixed rates has no such risk. Nobody can
freeze, claw back or change the metadata of an asset.

### Transfer rates

An issuer with a variable rate can move it to its `Max` at any time, and a `Max` of 100 % is a trap: the asset can be
sold while the rate is low and then turned into one that keeps every transfer. The bounds are public and can never be
widened, so the damage is limited to what they say, but only if the user reads them. A wallet MUST show the `Max`
rates, and whether a rate is variable, before the user buys the asset, SHOULD warn when the two `Max` rates are high,
and MUST set `MaxRate` on every transfer it builds to the rate it showed, never to the `Max` of the asset unless the
user asks for it. A rate change that lands between signing and inclusion cannot take more than `MaxRate`. A collector
that is the issuer, or a key that the issuer controls, receives the team share without any further rule, so users
judge the asset by the issuer, not by the symbol. The shares round down, so many small transfers can avoid a rate; this
costs the issuer revenue and nobody else anything. A transfer rate does not apply to a mint or to a burn. A later PIP
for exchanges MUST say how an atomic swap treats the rates of the assets that it moves.

An issuer that names a collector that cannot accept, for instance a mistyped address, makes the asset untransferable for
every holder, burns included, as long as `TeamRate.Current` is above 0 and that collector has not accepted, and the
collector of an asset can never be changed. That state can only come from `AssetCreate`: `AssetSetRate` cannot raise
the team rate above 0 before the collector has accepted. If `TeamRate.Min` is 0, the issuer releases the asset by
setting the team rate to 0, at the price of the team share. If `TeamRate.Min` is above 0, nothing releases it but the
collector's acceptance, so such an asset with a collector other than the issuer has no way out if the address is
wrong. A wallet MUST have the issuer confirm the collector address before it signs an `AssetCreate` with a policy,
SHOULD warn when `TeamRate.Min` is above 0 and the collector is not the issuer, and SHOULD show an asset whose
collector has not accepted as not yet transferable. The check costs the issuer one `AssetAccept` from the collector,
or none if it names itself.

### Identity and spoofing

Symbols are not unique and anyone can create one, such as `USDT`. Opt-in stops an asset from being pushed into a
wallet, but a user can still accept a look-alike by following a link or a QR code. A wallet MUST show the `AssetID` and
the issuer address and not only the symbol, MUST NOT trust a symbol, and MUST ask for confirmation, with those two
values, before it signs an `AssetAccept`. The symbol `PAC` is reserved so that no asset looks like the coin. An
`AssetID` in a link is a request and not an authorization: nothing happens until the user signs. Symbols can also
mislead by their digits (`0` and `O`, `1` and `I`), and so can `Decimals`: two assets with one symbol, one with 9
decimals and one with none, show quantities that differ by a factor of a billion. A wallet shows the decimals, and
amounts in the units of the asset that it reads from the state.

### Hostile inputs and what the user signs

* **An `AssetID` is known only after inclusion.** It is the position of the record in the tree, assigned when
  `AssetCreate` executes. Another creation can land first, so a wallet MUST NOT announce, accept or send an `AssetID`
  that it has not read from an included `AssetCreate`. Otherwise someone who sees a pending creation can take the ID,
  give it the same symbol, and have the announced ID accepted.
* **Metadata is attacker-chosen.** The document behind `MetaHash` can say anything. Wallets and explorers MUST check
  the hash before they use it, cap its size, accept only fixed content types, run no active content from it, and
  fetch no address that it embeds without the user asking.
* **Blind signing.** A wallet MUST show the decoded payload before it signs: the type, the `AssetID`, the symbol, the
  issuer, each recipient with its gross and net amount, the rates and `MaxRate`, and for a close the units that it
  burns. It MUST NOT sign a payload that it cannot decode, and a signing device that cannot show these values SHOULD
  refuse asset payloads.
* **Pool flooding.** Funded accounts can flood the asset pool with transactions that are valid alone and invalid
  together. The per-sender limit and the cost-ordered checks of section 5 bound this; the pool is local policy, so a
  node that does not apply them loses its own memory and nothing else.
* **Index corruption.** A local index that disagrees with the tree is a consensus fault, not a cache miss, which is
  why section 5 makes a node stop.

### Keys, replay and burns

Assets use ordinary accounts. Replay protection is that of every transaction: the lock time, and the rejection of
a transaction ID that was already executed. There is no allowance, so a signature can never authorize someone else
to spend later. A transfer to the zero address destroys units for good, and a wallet SHOULD ask for confirmation.
Loss of a key is the loss of its holdings, as for PAC.

### Privacy

Balances and transfers are public, as for PAC. This PIP adds no privacy.

## Future Extensions (informative)

* **Collections.** A token-ID dimension if one asset per item proves too costly.
* **A separate rate controller,** for example a multi-signature or a vote, and a delay before a new rate applies.
* **Removal of assets that nobody holds,** which needs a count of the holdings of each asset, to bound the permanent
  part of the state too.
* **An atomic swap payload,** so that two parties can exchange assets or an asset and PAC in one transaction.
* **An off-chain layer** with its own accountability, if asset demand outgrows `MaxAssetTxPerBlock`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
