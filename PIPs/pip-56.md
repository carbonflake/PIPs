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

This PIP adds native assets to Pactus with four payloads, one new state tree, and nothing else: no virtual machine,
no committee, no new cryptography. An asset is a record with a symbol, a number of decimals, a capped supply and a
metadata hash. Balances are kept per account. Assets are signed, paid and replay-protected like every other Pactus
transaction, with the same accounts and keys.

| Payload | Does |
| --- | --- |
| `AssetCreate` | Creates an asset and credits its initial supply to the creator |
| `AssetTransfer` | Moves units to 1 to 8 recipients. A transfer to the zero address burns them |
| `AssetMint` | Lets the issuer create more units, up to the `MaxSupply` fixed at creation |
| `AssetSetRate` | Lets the issuer move the transfer rates of an asset, inside the bounds fixed at creation |

Fungible tokens, NFTs and semi-fungible tokens are the same record with different parameters, not different types.

An asset may carry an optional **transfer policy**: a team rate and a burn rate, in basis points, that every transfer
applies with a few lines of integer arithmetic. The rates are variable but bounded: each has a minimum and a maximum
that are fixed at creation, and only the issuer can move a rate between them.

A block may carry at most `MaxAssetTxPerBlock` (100) asset transactions, so assets can never take more than a tenth
of a block and PAC always keeps the rest. Each asset transaction can carry up to 8 transfers, and the state that an
asset operation creates is paid for once, in PAC.

## Motivation

Pactus has no smart-contract layer. A token today means a memo, an off-chain indexer, or another chain. Putting tokens
into consensus naively has two costs: tokens compete with PAC for the capacity of a block, and every validator stores
every balance for ever.

This PIP keeps the design as small as the problem allows, in the spirit of Pactus: robust, reliable and light.

* **Robust.** Four payloads and one rule per payload. Every amount is bounded and every sum checked before it is
  added ([PIP-54](./pip-54.md)). The only authority an asset can have is a mint capped at creation.
* **Reliable.** Assets carry exactly the trust of PAC: the validators check every rule, and nobody else is trusted.
  Nothing depends on an operator, an indexer or an off-chain data store.
* **Light.** No per-block hook, no signature scheme, no epoch, no registry. The load that assets put on L1 is bounded
  by a constant, not by a market.

The commands are few: create, transfer, mint, and set a rate. Exchanges, allowances, lending, oracles, governance,
freezing, clawback and metadata updates are outside this PIP.

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
| `MaxAssetTxPerBlock` | 100 | Most asset transactions in one block. 10 % of a 1 000-transaction block |
| `MaxRecipients` | 8 | Recipients of one `AssetTransfer`, as `BatchTransfer` |
| `MaxDecimals` | 9 | Same as PAC |
| `MaxAssetSupply` | 2^63 - 1 | Largest quantity of any asset, the largest `int64` |
| `RecordCharge` | 0.001 PAC | Paid to the treasury for each record that a payload can append (section 3.5) |

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
There is at most one `HoldingRecord` for a given `(AssetID, Owner)`, and it is created the first time the owner
receives units. A node keeps a local index from `(AssetID, Owner)` to the record index. The index is not consensus
state and can be rebuilt from the tree.

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
* `Issued` and `Burned` never decrease.

### 3. Payloads

Four payload types are added. They take the next free numbers, 8 to 11, on the assumption that
[PIP-50](./pip-50.md) uses type 7; other Drafts also add payload types, so the editors renumber when assigning. No
test of this PIP depends on a type number.

```go
TypeAssetCreate   = Type(8)
TypeAssetTransfer = Type(9)
TypeAssetMint     = Type(10)
TypeAssetSetRate  = Type(11)
```

#### 3.1 Rules common to all four

* `Signer()` is `From`. Only **account addresses** (BLS, Ed25519, secp256k1) are valid. Treasury and validator
  addresses MUST be rejected as `From`.
* The transaction is signed, expires (`LockTime` / TTL) and is protected against replay like every other
  transaction. No nonce is added. It pays the ordinary fixed fee and is not a free transaction.
* `Value()` is the PAC charge of section 3.5, `RecordCharge * R`. It is computed from the payload alone, subtracted from
  `From` together with the fee, and credited to the treasury account. Every sum that involves `Value()`, the fee or a
  balance MUST be rejected before it exceeds `MaxNanoPAC` ([PIP-54](./pip-54.md)).
* Every asset quantity is an `int64` with `1 <= quantity <= MaxAssetSupply`. A sum of quantities MUST be checked
  before it is added, in the form `total > MaxAssetSupply - amount`.
* A recipient is an account address. Treasury and validator addresses are rejected, because nothing could move the
  units from them. The only exception is the zero address as a recipient of `AssetTransfer` (section 3.3).
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
`Collector` with `Balance = 0`. A transfer then never has to create the collector's record, which keeps its charge
a function of the payload (section 3.5).

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

`R` is the number of recipients that are not the zero address.

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
holding with `Balance >= total`; `Balance >= Value + fee` in PAC; no entry to an account has `net == 0`.

`Execute`: debit `Value + fee` in PAC and credit `Value` to the treasury; subtract `total` from the holding of `From`
(the holding stays, even at zero); then, for each recipient in the order of the list: if `To` is the zero address, add
`Amount` to the `Burned` of the asset; otherwise add `net` to the holding of `To`, appending a new `HoldingRecord`
first if there is none, add `team` to the holding of `Collector` if `team > 0`, and add `burn` to `Burned`. The holding
of `Collector` already exists (section 3.2). Each addition applies to the live record, so the result is the same when
addresses coincide, and the order makes the index of new records the same on every node.

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

`R` is 1. `Check`: the asset exists; `From == Issuer`; `Amount <= MaxSupply - Issued`; `Balance >= Value + fee`.
`Execute`: debit `Value + fee` and credit `Value` to the treasury; add `Amount` to `Issued`; add `Amount` to the
holding of `To`, appending a `HoldingRecord` first if there is none. A mint pays no transfer rate: it creates units,
it does not move them.

The cap is on units ever issued, not on units alive. A holder can therefore read `MaxSupply` and know the largest
quantity that will ever exist. An asset with `MaxSupply == InitialSupply` rejects every mint.

#### 3.5 Charges

`RecordCharge` pays for state that every node keeps for ever. Every payload has `Value() = RecordCharge * R`, where `R`
is the number of records the payload can append, **counted from the payload alone**: a wallet knows the charge before
it signs, and no charge depends on whether a balance entry already exists.

| Payload | `R` |
| --- | --- |
| `AssetCreate` | 1, plus 1 if `InitialSupply > 0`, plus 1 if `PolicyPresent` is `1` |
| `AssetTransfer` | The number of recipients that are not the zero address |
| `AssetMint` | 1 |
| `AssetSetRate` | 0, so `Value()` is 0 and only the fixed fee is paid |

The charge is paid whether or not a record is in fact appended, so a transfer to an existing holder pays for a record
it did not need. The charge is credited to the treasury, and the fixed fee follows its existing path. The rates of
section 3.3 are a different thing: they are quantities of the asset, set by its issuer, and PAC is not involved.

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

### 4. Block capacity

A block is invalid if it carries more than `MaxAssetTxPerBlock` transactions whose payload is one of the four types
above. There is no other rule about order or mix.

This is the only guarantee that protects PAC, and it is a consensus rule, not a fee market: whatever the demand for
assets, at least 90 % of a 1 000-transaction block stays available to the other payloads. A proposer SHOULD fill the
block with the other transactions first and add asset transactions after them. An asset transaction that does not fit
waits in its pool until its lock time expires, and the signer then signs it again.

### 5. Node changes

* `Sandbox` exposes reads and writes of asset records, and the node adds the tree, its store prefix and the extended
  root. `executeBlock` does not change: there is no per-block hook.
* Block validation counts asset transactions and rejects a block above the cap.
* Transaction pool: a pool is local policy, not consensus. Implementations SHOULD give the four types one pool of
  10 % of `MaxSize`, like `BatchTransfer`, so asset transactions cannot crowd out payments and payments cannot evict
  them. Fee estimation uses the same fixed fee as `Transfer`, plus `Value()`.
* RPC (mandatory for the official node, on every surface it ships): payload type values `8` to `11` in `PayloadType`;
  `GetRaw*Transaction` builders for the four payloads; `GetAsset`, returning the fields of section 2.1, the policy
  with its current rates and bounds, and the live supply `Issued - Burned`; and `GetAssetBalance(AssetID, Address)`.
  A node SHOULD also offer a list of the balances of an address from its local index.
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

## Rationale

**On-chain, because that is the simplest thing that is also fully trusted.** An asset rule that every validator
checks adds no assumption to Pactus. An earlier draft of this PIP ran the assets off-chain under a bonded committee
that checkpointed them on Pactus. That keeps L1 capacity free, but it adds operators, bonds, slashing, epochs and
quorums, and its correctness and its data availability rest on a federation that Pactus cannot check. The hard cap of
section 4 answers the same worry about L1 capacity with one counter.

**A cap, not a market.** A fee market needs a price that nobody knows before the layer is used. A cap on the number
of asset transactions in a block needs no guess, is trivial to validate and gives PAC a floor that no demand can
lower. Its price is that asset users can be delayed when asset demand reaches the cap. That is acceptable, and the
cap is one constant that a later version can raise.

**One record for every kind of asset.** A token-ID dimension (items inside a collection) needs item records, per-item
supply and more rules for each type. Here an NFT is an asset with a supply of one. The cost is that a collection of N
items is N creations, each priced by the fee and `RecordCharge` and paced by the cap. A later PIP can add collections if
there is demand, without changing these records.

**Batching inside `AssetTransfer`.** `BatchTransfer` already shows that up to 8 recipients are a safe size. One
transaction then moves up to 8 transfers, so the cap limits transactions while the layer still serves many users.
The single-recipient transfer is the case of one.

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

**Flat charges known from the payload.** `Value()` is computed from the payload alone, so a wallet never has to guess
a fee from the state, and no fee code depends on whether a balance entry exists, or on whether an asset has a policy:
the record of the collector is appended at creation for that reason. A transfer to an existing holder pays
`RecordCharge` that it did not strictly need. That is cheaper than a second code path.

**PAC pays everything.** No second fee token and no sponsor. A new holder needs a little PAC before moving units.

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
7. **Rates that are fixed, or that have no bounds.** Fixed rates are possible (`Min == Max`) but not the only choice.
   Unbounded rates would let an issuer take any share of any transfer.

## Backwards Compatibility

This is a consensus upgrade. Nodes that do not implement version `V` reject payload types 8 to 11 and cannot compute
the extended state root. Existing payload types, account records and validator records keep their encoding. Accounts
that never use the feature are unaffected.

Activation follows [PIP-51](./pip-51.md): implementations advertise version `V`; when more than 75 % of committee power
supports it, proposers raise the block version; from the first block of version `V` the extended root applies and
the four types are legal. Before that block they MUST be rejected as invalid payload types, and the asset tree does
not exist. State sync and snapshots MUST carry the asset tree from then on. Testnet SHOULD activate first.

## Test Cases

Implementations MUST pass at least these. They add no rule to the Specification.

* **Activation and root.** Types 8 to 11 are rejected below `V`. The first block of `V` has the extended root with
  `assetRoot` equal to 32 zero bytes. The root changes when a record changes, and a restart reloads the same root.
* **Create.** A fixed asset, a minting asset and an asset with `InitialSupply == 0` are created. Rejected: an empty
  symbol, 13 bytes, lowercase, a symbol of `PAC`, `Decimals` 10, `MaxSupply` 0 or above `MaxAssetSupply`,
  `InitialSupply` above `MaxSupply`, insufficient PAC, a treasury or validator `From`. The charge reaches the
  treasury, and the creator's holding is created only if `InitialSupply > 0`. With a policy, rejected: `Current`
  outside `[Min, Max]`, a `Max` above 10 000, `TeamRate.Max + BurnRate.Max` above 10 000, all `Max` at 0, `TeamCap` of
  0, a validator `Collector`. With a policy, the `Collector` holding exists at balance 0 right after creation, and
  is not duplicated when the collector is the creator.
* **Transfer.** One recipient and eight succeed; nine fail. Rejected: a repeated `To`, `To == From`, a validator
  recipient, an amount of 0 or below, an unknown asset, a balance that is too small, and 8 recipients of `2^62` each
  (the overflow case of PIP-54), which fails before any addition. A transfer to the zero address raises `Burned` and
  creates no holding. A holding that reaches zero stays. `Value()` is `RecordCharge * R` for every payload and counts
  only the recipients that are not the zero address. New records take the order of the recipient list.
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
  `MaxSupply` in total. An asset with `MaxSupply == InitialSupply` rejects every mint.
* **Block capacity.** A block with 100 asset transactions and 900 other transactions is valid; one with 101 asset
  transactions is invalid.
* **Invariants.** The invariants of section 2.3 hold after every test. Balances, stake and treasury sum to the PAC
  supply after every test: asset transactions only move PAC to the treasury and pay the fee.
* **Determinism.** Two nodes with different local indexes and one restarted from a pruned store obtain the same root
  and the same `GetAsset`. Every payload decodes and re-encodes to the same bytes, and trailing bytes are rejected.

## Reference Implementation

None yet. Before this PIP leaves Draft, a pull request will add the four payloads, the asset tree, the extended root,
the block cap and the RPC to the Pactus node, and report on stated hardware: the time to validate a block with 100
asset transactions of 8 recipients, the cost of a tree update, the size of the store per record, and the effect on
state sync.

### Open items of this draft

* **A reference implementation.** No part of this PIP has run inside a Pactus node.
* **Byte-level vectors** for each payload and for the extended root.
* **Constants.** Every value of section 1 is an initial proposal, not measured.

## Security Considerations

### Arithmetic

[PIP-54](./pip-54.md) came from a sum that wrapped. Every quantity here is an `int64` bounded by `MaxAssetSupply`,
every sum is checked before it is added, and the invariants of section 2.3 are checked in tests. A credit cannot
overflow, because a balance never exceeds the supply of its asset. The test with 8 recipients of `2^62` is mandatory.
The same care applies to PAC: `Value() + fee` and every balance are bounded by `MaxNanoPAC` before the sum.

### State growth

Records are never removed, as for accounts, so a record is paid for once, by `RecordCharge` and the fee. In the
worst case, a sender who fills the cap for ever with 8 new recipients per transaction creates 800 holdings of 34 bytes
per block, about 6.9 million a day and 235 MB of records, and pays about 15 600 PAC a day in charges and fees. Today a
sender can create up to 8 000 accounts per 1 000-transaction block with `BatchTransfer`, so the bound is lower than the
one Pactus already accepts for accounts, at a comparable price per record. Creating an asset adds one asset record of
at most 134 bytes, and costs the fee plus 2 or 3 `RecordCharge`, so asset spam is cheaper per record than holding spam
and is bounded by the same cap. Both the cap and `RecordCharge` are constants that a later version can change. Removal
of empty holdings, with a deposit, is left for a later PIP.

### Block capacity

The cap guarantees PAC at least 90 % of a block, and the proposer's order lets PAC take the rest when it needs it. A
spammer who fills the asset lane (100 transactions, at least 1 PAC a block in fees) delays other asset users and does
not touch PAC. Pools are local policy, and the 10 % pool keeps asset transactions from filling the memory of a node
at the expense of payments.

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

Symbols are not unique and anyone can create one, such as `USDT`. A wallet MUST show the `AssetID` and the issuer
address and not only the symbol, MUST NOT trust a symbol, and SHOULD hide assets that a user did not ask for: a
sender can credit any account with any asset, at no cost to the recipient. The symbol `PAC` is reserved so that no
asset looks like the coin.

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
* **Removal of empty holdings,** with a deposit, to bound state by supply.
* **An atomic swap payload,** so that two parties can exchange assets or an asset and PAC in one transaction.
* **An off-chain layer** with its own accountability, if asset demand outgrows `MaxAssetTxPerBlock`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
