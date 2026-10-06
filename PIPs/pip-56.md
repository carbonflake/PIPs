---
pip: 56
title: Native Assets
description: Fungible, non-fungible and semi-fungible assets as native Pactus state, capped per block to protect PAC.
author: CarbonFlake (@carbonflake)
status: Draft
type: Standards Track
category: Core
created: 2026-10-02
updated: 2026-10-06
requires: 39, 51, 54
---

## Abstract

This PIP adds native assets to Pactus with five payloads, one new state tree, and nothing else: no virtual machine,
no committee, no new cryptography. An asset is a record with a symbol, a number of decimals, a capped supply and a
metadata hash. Balances are kept per account. Assets are signed, paid and replay-protected like every other Pactus
transaction, with the same accounts and keys.

| Payload | Does |
| --- | --- |
| `AssetCreate` | Creates an asset and credits its initial supply to the creator |
| `AssetAccept` | Opts the signer in to an asset. Nobody can receive units of an asset before accepting it |
| `AssetTransfer` | Moves units to 1 to 8 recipients. A transfer to the zero address burns them |
| `AssetMint` | Lets the issuer create more units, up to the `MaxSupply` fixed at creation |
| `AssetSetRate` | Lets the issuer move the transfer rates of an asset, inside the bounds fixed at creation |

Fungible tokens, NFTs and semi-fungible tokens are the same record with different parameters, not different types.

**Nobody can be given an asset against their will.** An account holds an asset only after it has accepted it with its
own signature and paid for its own record. Transfers and mints never add a record to the state: they only change a
balance that its owner opened. This is what stops unsolicited airdrops, and it makes the growth of the state the
business of whoever causes it.

An asset may carry an optional **transfer policy**: a team rate and a burn rate, in basis points, that every transfer
applies with a few lines of integer arithmetic. The rates are variable but bounded: each has a minimum and a maximum
that are fixed at creation, and only the issuer can move a rate between them.

Batching is automatic and happens at two levels. A sender can put up to 8 recipients in one transaction, which costs one
signature for up to 8 transfers. And the block proposer packs the asset transactions of **any** senders, as they
arrive, into **bundles** of up to 8: every bundle of a block but at most one is full. Users send ordinary transactions
and never see a bundle. A bundle takes one slot of the block, so the `MaxAssetTxPerBlock` (200) asset transactions that
a block may carry use at most 25 of its 1 000 slots, and PAC always keeps the rest. The state that an asset operation
creates is paid for once, in PAC.

## Motivation

Pactus has no smart-contract layer. A token today means a memo, an off-chain indexer, or another chain. Putting tokens
into consensus naively has two costs: tokens compete with PAC for the capacity of a block, and every validator stores
every balance for ever.

This PIP keeps the design as small as the problem allows, in the spirit of Pactus: robust, reliable and light.

* **Robust.** Five payloads and one rule per payload. Every amount is bounded and every sum checked before it is
  added ([PIP-54](./pip-54.md)). The only authority an asset can have is a mint capped at creation.
* **Reliable.** Assets carry exactly the trust of PAC: the validators check every rule, and nobody else is trusted.
  Nothing depends on an operator, an indexer or an off-chain data store.
* **Light.** No per-block hook, no signature scheme, no epoch, no registry. The load that assets put on L1 is bounded
  by a constant, not by a market.

The commands are few: create, accept, transfer, mint, and set a rate. Exchanges, allowances, lending, oracles,
governance, freezing, clawback and metadata updates are outside this PIP.

## Specification

Integers are little-endian and fixed width unless a field is marked *varint*. `Address` is the 21-byte Pactus address,
and the zero address is the treasury address (21 zero bytes). Fields are encoded in the order listed, using the
encodings of the existing payloads: `Address` as in `Transfer`, and *varint* as the `Amount` of `Transfer` and the
number of recipients of `BatchTransfer` ([PIP-39](./pip-39.md)). Decoding MUST consume exactly the declared fields,
and trailing bytes make a payload invalid. Byte-level test vectors come with the reference implementation.

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
| `RecordCharge` | 0.001 PAC | Paid to the treasury for each record that `AssetCreate` or `AssetAccept` appends (section 3.5) |

### 2. State

#### 2.1 The asset tree

A new Merkle tree, `assetMerkle`, is built exactly like `accountMerkle`: leaf `i` is the hash of record `i`, a new
record is appended at the next free index, and records are never removed, so an index never changes meaning. There
are two kinds of record.

`AssetRecord` (kind `1`), 93 bytes, or 134 with a transfer policy:

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
| `Collector` | `Address` | An account address that receives the team share. Fixed at creation |
| `TeamRate` | 3 x `uint16` | `Current`, `Min`, `Max`, in basis points. `Min` and `Max` are fixed at creation |
| `BurnRate` | 3 x `uint16` | `Current`, `Min`, `Max`, in basis points. `Min` and `Max` are fixed at creation |
| `TeamCap` | `int64` | Largest team share of one transfer entry. `MaxAssetSupply` means no cap. Fixed at creation |

A basis point is 1/10 000, so a rate of 10 000 is 100 %. Only `Current` ever changes after creation. A rate is
**fixed** when `Min == Max` and **variable** otherwise, so no separate flag is needed. An asset without a policy
behaves as if both rates were 0.

`HoldingRecord` (kind `2`), 34 bytes:

| Field | Type | Notes |
| --- | --- | --- |
| `Kind` | `uint8` | `2` |
| `AssetID` | `uint32` | The index of the `AssetRecord` |
| `Owner` | `Address` | An account address |
| `Balance` | `int64` | Units held, `0 <= Balance <= MaxSupply` of the asset. May be zero |

The `AssetID` of an asset is the index of its `AssetRecord`. A record is hashed over its encoding, `Hash(Bytes)`.
There is at most one `HoldingRecord` for a given `(AssetID, Owner)`. Only `AssetCreate` (for the creator and for the
collector) and `AssetAccept` (for its signer) append one. `AssetTransfer` and `AssetMint` never append a record: they
need the recipient's record to exist. A node keeps a local index from `(AssetID, Owner)` to the record index. The index
is not consensus state and can be rebuilt from the tree.

#### 2.2 State root

For a block below the version `V` that activates this PIP, the state root is unchanged. For a block of version `V` or
higher, the root that the block header commits to, which is the state before the block as today, becomes:

```text
assetRoot = assetMerkle.Root()          (32 zero bytes if the tree has no record)
stateRoot = HashMerkleBranches(HashMerkleBranches(accountMerkle.Root(), validatorMerkle.Root()), assetRoot)
```

Asset state MUST be part of the root: a store that the root does not cover would let two nodes commit the same block
with different balances. It is persisted in the same store batch as accounts and validators.

#### 2.3 Invariants

After every transaction, for every asset:

* `0 <= Burned <= Issued <= MaxSupply <= MaxAssetSupply`;
* the sum of its holdings equals `Issued - Burned`;
* `Issued` and `Burned` never decrease;
* a holding is appended only by `AssetCreate` and `AssetAccept`, so a transfer or a mint never changes the number of
  records of the tree.

### 3. Payloads

Five payload types are added. They take the next free numbers, 8 to 12, on the assumption that
[PIP-50](./pip-50.md) uses type 7; other Drafts also add payload types, so the editors renumber when assigning. No
test of this PIP depends on a type number.

```go
TypeAssetCreate   = Type(8)
TypeAssetTransfer = Type(9)
TypeAssetMint     = Type(10)
TypeAssetSetRate  = Type(11)
TypeAssetAccept   = Type(12)
```

#### 3.1 Rules common to all five

* `Signer()` is `From`. Only **account addresses** (BLS, Ed25519, secp256k1) are valid. Treasury and validator
  addresses MUST be rejected as `From`.
* The transaction is signed, expires (`LockTime` / TTL) and is protected against replay like every other
  transaction. No nonce is added. It pays the ordinary fixed fee and is not a free transaction.
* `Value()` is the PAC charge of section 3.5, `RecordCharge * R`. It is computed from the payload alone, subtracted from
  `From` together with the fee, and credited to the treasury account. Every sum that involves `Value()`, the fee or a
  balance MUST be rejected before it exceeds `MaxNanoPAC` ([PIP-54](./pip-54.md)).
* Every asset quantity is an `int64` with `1 <= quantity <= MaxAssetSupply`. A sum of quantities MUST be checked
  before it is added, in the form `total > MaxAssetSupply - amount`.
* An asset transaction is an ordinary transaction, and it travels through the network and the pools as one. In a block
  it is carried only inside a bundle (section 4).
* A recipient is an account address. Treasury and validator addresses are rejected, because nothing could move the
  units from them. The only exception is the zero address as a recipient of `AssetTransfer` (section 3.3). Any other
  recipient MUST already hold a `HoldingRecord` of the asset, possibly with a balance of 0: it MUST have accepted the
  asset (section 3.7).
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

`R` is 1, plus 1 if `InitialSupply > 0`, plus 1 if `PolicyPresent` is `1`. `BasicCheck` of a policy: `Collector` is an
account address; for each rate, `Min <= Current <= Max <= 10 000`; `TeamRate.Max + BurnRate.Max <= 10 000`; at least
one `Max` is above 0; `1 <= TeamCap <= MaxAssetSupply`. `Check`: `Balance >= Value + fee`, and the tree has room for
`R` more records.

`Execute`: debit `Value + fee` and credit `Value` to the treasury; append an `AssetRecord` with `Issuer = From`,
`Issued = InitialSupply`, `Burned = 0` and the policy as given; if `InitialSupply > 0`, append a `HoldingRecord` of
`From` for it; if the policy is present and `Collector` has no holding of the asset yet, append a `HoldingRecord` of
`Collector` with `Balance = 0`. The collector cannot accept the asset before it exists, and transfers never append a
record, so its record is created here. A creator with `InitialSupply == 0` that is not also the collector holds no
record yet: to mint to itself, it accepts the asset like anyone else.

The asset is **minting** if `MaxSupply > InitialSupply`. If they are equal, nobody can ever create another unit and
nobody holds any authority over the asset: the supply is fixed for ever. Symbols are labels, not names: they are not
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

`BasicCheck`: the number of recipients is in range; no `To` appears twice; no `To` equals `From`; every amount is in
range; the sum of the amounts, checked as in section 3.1, does not exceed `MaxAssetSupply`.
`Check`: the asset exists; `TeamRate.Current + BurnRate.Current <= MaxRate` (both are 0 without a policy); `From` has a
holding with `Balance >= total`; every `To` that is not the zero address has a holding of the asset; `Balance >= fee`
in PAC; no entry to an account has `net == 0`.

`Execute`: debit the fee in PAC; subtract `total` from the holding of `From` (the holding stays, even at zero); then,
for each recipient in the order of the list: if `To` is the zero address, add `Amount` to the `Burned` of the asset;
otherwise add `net` to the holding of `To`, add `team` to the holding of `Collector` if `team > 0`, and add `burn` to
`Burned`. The holding of `Collector` already exists (section 3.2). Each addition applies to the live record, so the
result is the same when addresses coincide. No record is appended.

For example, with `Amount = 1 000 000`, `TeamRate = 200` (2 %), `BurnRate = 50` (0.5 %) and `TeamCap = 15 000`:
`share(Amount, 200) = 20 000`, so `team = 15 000`; `burn = 5 000`; `net = 980 000`. With `Amount = 99` and the same
rates, `team = 1`, `burn = 0` (0.495 rounds down) and `net = 98`. With `Amount = 49`, both shares round to 0 and the
recipient gets all 49.

`MaxRate` exists because a rate can change between the moment a user signs and the moment the transaction is included
(section 3.6). A wallet sets it to the rate it showed, or to the `Max` of the asset to accept any change. The sender
can never pay more than the bounds fixed at creation, and `MaxRate` narrows that bound for one transaction.

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
Issued`; `Balance >= fee`. `Execute`: debit the fee; add `Amount` to `Issued`; add `Amount` to the holding of `To`. No
record is appended. A mint pays no transfer rate: it creates units, it does not move them.

The cap is on units ever issued, not on units alive. A holder can therefore read `MaxSupply` and know the largest
quantity that will ever exist. An asset with `MaxSupply == InitialSupply` rejects every mint.

#### 3.5 Charges

`RecordCharge` pays for state that every node keeps for ever. Every payload has `Value() = RecordCharge * R`, where `R`
is the number of records the payload can append, **counted from the payload alone**: a wallet knows the charge before
it signs, and no charge depends on the state.

| Payload | `R` |
| --- | --- |
| `AssetCreate` | 1, plus 1 if `InitialSupply > 0`, plus 1 if `PolicyPresent` is `1` |
| `AssetAccept` | 1 |
| `AssetTransfer` | 0 |
| `AssetMint` | 0 |
| `AssetSetRate` | 0 |

Only creation and acceptance append records, so only they pay a charge, and the account that causes a record is the
account that pays for it. A creator that is also the collector, or whose `InitialSupply` is 0, pays for a record that
is not appended; that is cheaper than a charge that depends on the state. The charge is credited to the treasury, and
the fixed fee follows its existing path. The rates of section 3.3 are a different thing: they are quantities of the
asset, set by its issuer, and PAC is not involved.

#### 3.6 `AssetSetRate` (type 11)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | MUST be the `Issuer` |
| `AssetID` | `uint32` | |
| `TeamRate` | `uint16` | The new `TeamRate.Current`, in basis points |
| `BurnRate` | `uint16` | The new `BurnRate.Current`, in basis points |

`Check`: the asset exists and has a policy; `From == Issuer`; `TeamRate.Min <= TeamRate <= TeamRate.Max` and
`BurnRate.Min <= BurnRate <= BurnRate.Max`; `Balance >= fee`. `Execute`: debit the fee and set the two `Current`
values. Nothing else about the policy ever changes: not the bounds, the cap or the collector. The new rates apply to
every transfer included after this transaction in the same block or later. A transaction that sets the current values
again is valid and does nothing.

An asset whose bounds satisfy `Min == Max` for both rates has fixed rates and can never be changed; its issuer holds no
authority over them. The bounds are a promise that anyone can read in the state, as `MaxSupply` is.

#### 3.7 `AssetAccept` (type 12)

| Field | Type | Notes |
| --- | --- | --- |
| `From` | `Address` | The account that opts in. An account address |
| `AssetID` | `uint32` | |

`Value()` is `RecordCharge`. `Check`: the asset exists; `From` has no holding of the asset yet;
`Balance >= Value + fee`. `Execute`: debit `Value + fee` and credit `Value` to the treasury; append a `HoldingRecord`
of `From` for the asset with `Balance = 0`.

Only the account itself can accept, because only its signature is valid. From then on it can receive transfers and
mints of the asset. There is no way to refuse a transfer once the record exists, and none to remove the record: an
account that does not want an asset simply never accepts it, and an account that accepted one by mistake keeps an
empty record that costs it nothing more. Accepting twice is rejected, so nobody pays twice for the same record.

A sender cannot open a record for someone else, and cannot pay for it either. An issuer that wants to distribute an
asset publishes the `AssetID`, lets the recipients accept, and then sends the units, in one batch of up to 8 per
transaction. A wallet that wants to receive an asset asks its user, then signs an `AssetAccept`.

### 4. Bundles and block capacity

#### 4.1 Bundles

A transaction cannot be extended by anyone but its signer, because its signature covers the whole payload. So the node
never edits a transaction to add a transfer to it. It packs whole transactions, each with its own signature, into a
**bundle**: an entry of the list of transactions of a block that holds complete asset transactions and has no sender,
fee, lock time or signature of its own. These rules are consensus rules:

1. In a block, asset transactions (the five types above) appear only inside bundles, never as entries of the list on
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

The wire encoding of a bundle belongs to the reference implementation. It MUST be deterministic and carry only a count
and the items.

#### 4.2 Packing

The proposer does the packing, and no user does anything. It takes asset transactions from its asset pool in arrival
order, drops any that no longer pass `Check` against the state of the block under construction, fills bundles of
`MaxBundleItems`, and stops at `MaxAssetTxPerBlock`. When the first bundle is full it starts a second, and so on; the
last one may be partial. It SHOULD place bundles after the other transactions. What does not fit stays in the pool for
the next block, until its lock time expires, and the signer then signs it again.

#### 4.3 Capacity

Two limits bound two different resources.

* **Slots.** A bundle is one entry of the 1 000-transaction list. Whatever the senders, asset transactions take at most
  `ceil(MaxAssetTxPerBlock / MaxBundleItems)` slots, 25 with the values above, which leaves PAC at least 97.5 % of the
  slots of a block. Rules 3 and 4 make this a guarantee and not a policy.
* **Work.** A bundle saves slots and the header of each transaction, not signature checks: every item is verified,
  executed and written to the state as it would be alone. `MaxAssetTxPerBlock` is what bounds this work and the growth
  of the state, and it is the number that a benchmark must set (Reference Implementation).

This is a consensus rule and not a fee market: whatever the demand for assets, it cannot take more than these slots
and this work from a block.

### 5. Node changes

* `Sandbox` exposes reads and writes of asset records, and the node adds the tree, its store prefix and the extended
  root. `executeBlock` does not change: there is no per-block hook.
* Block validation enforces the rules of section 4.1 and computes the transaction root over the entries of the list.
  The block body gains one kind of entry, the bundle. Store indexes (transaction by ID, replay check) work on the
  items, so a transaction is found and replay-protected whether or not it sits in a bundle.
* Block building packs bundles as in section 4.2. The gossip of transactions does not change: nodes relay individual
  asset transactions, and a bundle exists only inside a block.
* Transaction pool: a pool is local policy, not consensus. Implementations SHOULD give the five types one pool of
  10 % of `MaxSize`, like `BatchTransfer`, so asset transactions cannot crowd out payments and payments cannot evict
  them. Fee estimation uses the same fixed fee as `Transfer`, plus `Value()`. A pool MAY drop an `AssetTransfer` or
  `AssetMint` whose recipient has not accepted the asset, since it cannot be valid until an `AssetAccept` lands.
* RPC (mandatory for the official node, on every surface it ships): payload type values `8` to `12` in `PayloadType`;
  `GetRaw*Transaction` builders for the five payloads; `GetAsset`, returning the fields of section 2.1, the policy
  with its current rates and bounds, and the live supply `Issued - Burned`; and `GetAssetBalance(AssetID, Address)`,
  which tells an account that has not accepted the asset from one that holds a balance of 0, so that a sender can check
  before it signs. A node SHOULD also offer a list of the balances of an address from its local index. `GetBlock` and
  `GetTransaction` return the items of a bundle as ordinary transactions, so a wallet or an explorer does not need to
  know about bundles; they MAY also report the bundle of each item.
* Explorers need no new term for supply audits: the charges go to the treasury and every other PAC amount keeps its
  place.

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
  supply 1 every share rounds to 0.
* Receiving an asset takes two steps: the recipient accepts, then the sender sends. A wallet asks its user before it
  signs an `AssetAccept`, and MAY offer the acceptance as a link or a QR code that carries the `AssetID`. A distribution
  to many accounts is a claim: the issuer publishes the `AssetID`, accounts accept, and the issuer sends to those that
  did, 8 at a time. A sale of an NFT starts with the buyer accepting it.

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
items is N creations, each priced by the fee and `RecordCharge` and paced by the cap. A later PIP can add collections if
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
`MaxSupply == InitialSupply` is the strongest form and has no authority at all. Transferring or renouncing the issuer
role is left out: it adds a payload and a way to lose control by mistake.

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

**Opt-in, so that nobody is given an asset against their will.** Anyone can create an asset with any symbol, and without
opt-in anyone could put it in any wallet for a fee that is small in PAC: look-alike symbols, links in metadata,
and a record added to every victim's state. With opt-in, an asset reaches an account only if that account signed for
it. It also simplifies the rules. A transfer or a mint never appends a record, so their execution has no creation
order to fix, their charge is 0, and the tree grows only when its owner asks and pays. The cost is two steps to
receive an asset, and a recipient that needs a little PAC before it can accept. To the knowledge of the author, the
XRP Ledger, Stellar and Algorand also ask the receiver to opt in. The records of the creator and of the collector are
the only ones that an account does not accept itself: the creator signs the creation, and the collector is named by it.

**Flat charges known from the payload.** `Value()` is computed from the payload alone, so a wallet never has to guess
a fee from the state, and no fee code depends on the state. A creator that is its own collector, or that creates with
no initial supply, pays for a record that is not appended. That is cheaper than a second code path.

**PAC pays everything.** No second fee token and no sponsor. A new holder needs a little PAC to accept an asset, and
to move it.

## Alternatives Considered

1. **Off-chain execution checkpointed by a bonded committee** (an earlier draft of this PIP). Rejected as too complex
   and for its trust assumptions. It remains a possible later PIP if asset demand outgrows the cap.
2. **A profile over [PIP-50](./pip-50.md) anchors, computed by indexers.** No consensus change, but Pactus enforces
   nothing: validity is whatever the indexer says.
3. **A smart-contract VM.** Attack surface out of proportion to the tokens this PIP targets.
4. **Item records inside an asset (collections).** Set aside: more state and rules, and the single record covers the
   need at the cost of more creations.
5. **Refundable deposits per balance entry.** Would bound state by supply but needs deletion of records, which no
   Pactus tree has, and a rule about whose deposit it is when a sender creates the entry. Not worth it now.
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
10. **The sender pays for the recipient's record, with no opt-in** (an earlier draft of this PIP, and the model of
    Cardano and Solana, with a refundable reserve in each). Simpler to use, but anyone can put an asset in any wallet,
    and without a way to remove records the state grows with the spam. Set aside for the opt-in of section 3.7.
11. **Higher charges and wallets that hide unknown assets.** No consensus change, but it prices spam instead of
    stopping it and leaves the records in the state. It remains a rule for wallets in any case.

## Backwards Compatibility

This is a consensus upgrade. Nodes that do not implement version `V` reject payload types 8 to 12 and cannot compute
the extended state root. Existing payload types, account records and validator records keep their encoding. Accounts
that never use the feature are unaffected.

Activation follows [PIP-51](./pip-51.md): implementations advertise version `V`; when more than 75 % of committee power
supports it, proposers raise the block version; from the first block of version `V` the extended root applies, the
five types are legal and a block body may carry bundles, with its transaction root computed over the entries of its
list. Before that block they MUST be rejected as invalid payload types, a bundle is an invalid entry, and the asset
tree does not exist. Software that reads block bodies (explorers, indexers, light clients) must learn the bundle. State
sync and snapshots MUST carry the asset tree from then on. Testnet SHOULD activate first.

## Test Cases

Implementations MUST pass at least these. They add no rule to the Specification.

* **Activation and root.** Types 8 to 12 are rejected below `V`. The first block of `V` has the extended root with
  `assetRoot` equal to 32 zero bytes. The root changes when a record changes, and a restart reloads the same root.
* **Create.** A fixed asset, a minting asset and an asset with `InitialSupply == 0` are created. Rejected: an empty
  symbol, 13 bytes, lowercase, a symbol of `PAC`, `Decimals` 10, `MaxSupply` 0 or above `MaxAssetSupply`,
  `InitialSupply` above `MaxSupply`, insufficient PAC, a treasury or validator `From`. The charge reaches the
  treasury, and the creator's holding is created only if `InitialSupply > 0`. With a policy, rejected: `Current`
  outside `[Min, Max]`, a `Max` above 10 000, `TeamRate.Max + BurnRate.Max` above 10 000, all `Max` at 0, `TeamCap` of
  0, a validator `Collector`. With a policy, the `Collector` holding exists at balance 0 right after creation, and
  is not duplicated when the collector is the creator.
* **Accept.** An account accepts an asset: a holding with balance 0 is appended at the next index, `RecordCharge`
  reaches the treasury, and the account can then receive. Rejected: an unknown asset, a second `AssetAccept` of the same
  asset by the same account (nothing is charged twice), insufficient PAC, a treasury or validator `From`. The collector
  and a creator with an initial supply hold a record from creation, and cannot accept again. An accept and a transfer
  to that account in one bundle, in that order, both succeed; in the other order the transfer makes the block invalid.
* **Transfer.** One recipient and eight succeed; nine fail. Rejected: a repeated `To`, `To == From`, a validator
  recipient, an amount of 0 or below, an unknown asset, a balance that is too small, a recipient that has not
  accepted the asset, and 8 recipients of `2^62` each (the overflow case of PIP-54), which fails before any addition. A
  transfer to the zero address needs no holding, raises `Burned` and creates no record. A holding that reaches zero
  stays. `Value()` is 0, and the number of records of the tree is the same before and after any transfer.
* **Rates.** The example of section 3.3 gives `team = 15 000`, `burn = 5 000` and `net = 980 000`. With the same rates,
  99 gives `team = 1`, `burn = 0`, `net = 98`, and 49 gives `team = burn = 0`. A transfer of `2^63 - 1` with rates
  of 10 000 in total is computed without leaving the `int64` range, and `team + burn + net` equals `Amount` for a
  range of amounts and rates, including values just below and above multiples of 10 000. `TeamCap` limits `team` and
  not `burn`. An entry whose `net` is 0 is rejected. `MaxRate` below the current sum of the rates fails the transfer,
  and an asset without a policy accepts `MaxRate` 0. An entry to the zero address pays no rate. The recipient, the
  collector and `Burned` change as specified when the collector is the recipient or the sender.
* **Set rate.** The issuer sets a rate inside its bounds and the next transfer uses it. A value outside `[Min, Max]`, a
  non-issuer, an asset without a policy and an unknown asset fail. An asset with `Min == Max` accepts only that
  value. Bounds, cap and collector never change.
* **Mint.** Only the issuer mints. A mint above `MaxSupply - Issued` fails. After a burn, a mint still cannot pass
  `MaxSupply` in total. An asset with `MaxSupply == InitialSupply` rejects every mint. A mint to an account that has not
  accepted the asset fails, including a mint by an issuer to itself when `InitialSupply` was 0 and it is not the
  collector. A mint appends no record and has `Value()` 0.
* **Bundles.** A block with 200 asset transactions in 25 full bundles and 975 other transactions is valid; 201 asset
  transactions are invalid. A block with 17 asset transactions in a bundle of 8, a bundle of 8 and a bundle of 1 is
  valid, and the same 17 in a bundle of 8, a bundle of 5 and a bundle of 4 is invalid (two partial bundles). Invalid: an
  empty bundle, a bundle of 9, an asset transaction outside any bundle, a bundle that holds a `Transfer` or another
  bundle, and a bundle in a block below `V`. An item with a bad signature, an expired lock time, an ID that was already
  executed or a balance that is too small makes the block invalid. Two items of one sender in one bundle run in order.
  The state root after a block with bundles equals the one after the same block with the bundles replaced by their
  items. Repacking the same items gives a different bundle ID and a different transaction root. A transaction is found
  by `GetTransaction` with its own ID, and a transaction ID that is already in a bundle is rejected a second time.
* **Invariants.** The invariants of section 2.3 hold after every test. Balances, stake and treasury sum to the PAC
  supply after every test: asset transactions only move PAC to the treasury and pay the fee.
* **Determinism.** Two nodes with different local indexes and one restarted from a pruned store obtain the same root
  and the same `GetAsset`. Every payload decodes and re-encodes to the same bytes, and trailing bytes are rejected.

## Reference Implementation

None yet. Before this PIP leaves Draft, a pull request will add the five payloads, the asset tree, the extended root,
the bundles, the block cap and the RPC to the Pactus node, and report on stated hardware: the time to validate a block
with `MaxAssetTxPerBlock` asset transactions of 8 recipients each, for BLS, Ed25519 and secp256k1 senders, which is
what sets the cap; the cost of a tree update, the size of the store per record, and the effect on
state sync.

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

Records are never removed, as for accounts, so a record is paid for once, by `RecordCharge` and the fee, and only
`AssetCreate` and `AssetAccept` append one. Transfers and mints never grow the state, and nobody can make an account
pay for a record it did not ask for. In the worst case, a sender who fills the cap for ever with creations that each
append 3 records (an asset of at most 134 bytes and two holdings of 34 bytes) adds 40 KB per block, about 5.2 million
records and 349 MB a day, and pays about 22 500 PAC a day in charges and fees. Accepting only (one 34-byte record per
transaction) adds 59 MB a day for about 19 000 PAC. Today a sender can create up to 8 000 accounts per
1 000-transaction block with `BatchTransfer`, about 830 MB a day, so the bound is lower than the one Pactus already
accepts for accounts. Both the cap and `RecordCharge` are constants that a later version can change. Removal of empty
holdings, with a refund, is left for a later PIP.

An account that accepts an asset by mistake keeps an empty record for ever, and cannot reject later transfers of it,
since anyone who holds the asset may send it. That costs the account nothing more than its own acceptance, and it
never had to accept.

### Block capacity

The bundle rules keep assets to at most 25 slots of a block (section 4.3), so PAC keeps at least 97.5 % of them,
and a proposer cannot take more by packing badly, because only the last bundle may be partial. The **work** of assets is
bounded by `MaxAssetTxPerBlock` alone: a bundle does not lower the signatures to verify or the records to write, so that
number must come from the benchmark and not from the slot arithmetic. A spammer who fills the asset lane (200
transactions, at least 2 PAC a block in fees) delays other asset users and does not touch PAC. Pools are local policy,
and the 10 % pool keeps asset transactions from filling the memory of a node at the expense of payments. The ID of a
bundle commits to its packing, so a relay cannot repack the body of a block without changing its hash, and a block
whose packing breaks section 4.1 is invalid even if every item is valid.

### The issuer

A minting asset gives its issuer the power to create units up to `MaxSupply`. A wallet MUST show whether an asset is
minting and how much room is left, before the user buys it. The issuer key cannot be replaced: if it is lost, no more
units are minted; if it is stolen, the thief can mint up to the cap. An asset with `MaxSupply == InitialSupply` has no
such risk. Nobody can freeze, claw back or change the metadata of an asset.

### Transfer rates

An issuer with a variable rate can move it to its `Max` at any time, and a `Max` of 100 % is a trap: the asset can be
sold while the rate is low and then turned into one that keeps every transfer. The bounds are public and can never be
widened, so the damage is limited to what they say, but only if the user reads them. A wallet MUST show the `Max`
rates, and whether a rate is variable, before the user buys the asset, SHOULD warn when the two `Max` rates are high,
and MUST set `MaxRate` on every transfer it builds. A rate change that lands between signing and inclusion cannot
take more than `MaxRate`. A collector that is the issuer, or a key that the issuer controls, receives the team share
without any further rule, so users judge the asset by the issuer, not by the symbol. The shares round down, so many
small transfers can avoid a rate; this costs the issuer revenue and nobody else anything. A transfer rate does not
apply to a mint or to a burn. A later PIP for exchanges MUST say how an atomic swap treats the rates of the assets
that it moves.

### Identity and spoofing

Symbols are not unique and anyone can create one, such as `USDT`. Opt-in stops an asset from being pushed into a
wallet, but a user can still accept a look-alike by following a link or a QR code. A wallet MUST show the `AssetID` and
the issuer address and not only the symbol, MUST NOT trust a symbol, and MUST ask for confirmation, with those two
values, before it signs an `AssetAccept`. The symbol `PAC` is reserved so that no asset looks like the coin. An
`AssetID` in a link is a request and not an authorization: nothing happens until the user signs.

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
* **Removal of empty holdings,** with a refund of the acceptance charge, to bound state by what is in use.
* **An atomic swap payload,** so that two parties can exchange assets or an asset and PAC in one transaction.
* **An off-chain layer** with its own accountability, if asset demand outgrows `MaxAssetTxPerBlock`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
