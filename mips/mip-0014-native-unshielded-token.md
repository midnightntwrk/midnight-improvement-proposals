---
MIP: "0014"
Title: Native Unshielded Token Standard
Authors:
  - Jay Albert
Status: Proposed
Category: Standards
Created: 2026-07-14
Requires: none
Replaces: none
MPS: none
License: Apache-2.0
---

<!--
 This file is part of midnight-improvement-proposals.
 Copyright (C) Midnight Foundation
 SPDX-License-Identifier: Apache-2.0
 Licensed under the Apache License, Version 2.0 (the "License");
 You may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

This MIP specifies a standard for native unshielded tokens on Midnight: fungible assets that live as protocol-level unshielded UTXOs, publicly valued and owned by a `UserAddress`. The defining property is that the issuing contract is an issuer, not a custodian. A contract is required to bring a token into existence, because minting is a contract circuit and a token's color is bound to the minting contract's address by `tokenType`. Holding, transfer, receipt, balance inspection, and burning require no contract at all: they are native ledger operations under a Schnorr signature, with no requirement that a recipient has ever interacted with the issuer. A fixed-supply token therefore touches its contract only at issuance, and moves peer-to-peer for the rest of its life.

The standard specifies:

- how a conforming token derives its color;
- how new supply is minted at the single contract-bound point;
- how value moves natively between holders;
- how metadata is bound to a token's color and discovered by wallets;
- how a token is burned natively; and
- how supply is reported as an honest bound rather than an exact figure.

It is the transparent sibling of [MIP-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md) (native shielded tokens) and is designed so that a shielded representation of the same asset can be paired with it by a separate conversion mechanism, such as the UTXO conversion [MIP-0004](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md) describes. It deliberately does not cover authorization, transfer mediation, post-issuance controls, or the shield/unshield conversion itself, each of which lives elsewhere (see [Out of Scope](#out-of-scope)).

## Motivation

Midnight's token-standards effort is organized around six token types ([Discussion #142](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/142)): shielded native, unshielded native, shielded contract, unshielded contract, and two hybrid types. [MIP-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md) specifies the native shielded type and [MIP-0004](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md) the contract types. This MIP fills the native unshielded type, whose canonical example is NIGHT: the [docs](https://docs.midnight.network/tokens) classify NIGHT as an unshielded ledger token, transparent and UTXO-based, used for transparent value transfer. NIGHT itself is the protocol's built-in token and is not issued through this standard, but its shape is the shape this standard generalizes for issued assets.

The protocol already supports these tokens directly. `mintUnshieldedToken` mints a coin of a contract-derived color to a `UserAddress`, and a holder spends an unshielded UTXO at the ledger layer under a Schnorr signature. What is missing is an agreed convention for how an issued token derives its color and exposes its metadata, so that wallets, indexers, and bridges recognize and handle it uniformly. Without that, every issuer rebuilds the surface differently and the tokens are not portable.

The distinction from the other token standards is not whether a contract is used to mint. All three use a contract to mint, because minting is always a contract circuit. The distinction is how long the contract stays in the loop after that:

- MIP-0004 (contract token): the contract is a permanent custodian. Balances live in its Map, and every transfer is a contract call. The contract is in the loop for the token's whole life.
- MIP-0011 (native shielded token): the contract mints and burns, then steps out. The coin lives as a shielded Zswap UTXO and moves peer-to-peer with no contract.
- This standard (native unshielded token): same shape as MIP-0011, the contract mints then steps out, but the UTXO is publicly valued and owned by a `UserAddress`.

This is the property [Discussion #142](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/142) asked for directly: value must move to a recipient known only by `UserAddress`, with neither party first transacting with a contract. A standard that required a contract round trip to recognize or transfer a token would just be describing a contract token, which MIP-0004 already covers.

This proposal reuses decisions from related standards instead of making new ones. It takes the bounded-supply model from MIP-0011, because unshielded tokens face the same limit shielded ones do: the contract cannot see coins destroyed outside it. The interoperability section is written so that an issuer who later wants a shielded version of the same asset can use a separate conversion mechanism, like the one in MIP-0004, without this standard having to define it.

## Specification

### Terminology

- **Native unshielded token**: a class of unshielded UTXOs sharing one color, whose lifecycle is native to the ledger except for minting, the one contract-bound operation.
- **Color (token type)**: `tokenType(domain, contractAddress)` per the [Compact Standard Library](https://docs.midnight.network/compact). Only the contract at `contractAddress` can mint coins of its colors.
- **Domain separator (`domain`)**: a 32-byte value that, hashed with the contract address, produces one token color. Since the color is `tokenType(domain, contractAddress)`, a single contract yields a distinct color for each distinct domain, so one contract can issue many independent token types.
- **`UserAddress`**: a user public key address. At the Compact layer it is `struct UserAddress { bytes: Bytes<32>; }`, a 32-byte value; wallets present it Bech32m-encoded with the `mn_addr` human-readable prefix and a network identifier (for example `mn_addr_test1...`).
- **Issuer contract**: the contract whose address derives a token's color and which mints its supply. Needed to create supply and, for mintable tokens, at each later mint; needed for nothing else.

### Token Lifecycle

A conforming token only touches its contract to mint supply. Minting goes through the contract; everything else, holding, transferring, receiving, checking balances, and burning, happens natively with no contract involved. The contract is an issuer, not a custodian: it never holds or moves tokens between users.

Contract-bound, at issuance: **mint**. Minting is always a contract circuit, `mintUnshieldedToken`, and the color it produces is `tokenType(domain, contractAddress)`, so a color can only originate from the contract whose address derives it. There is no contractless mint. A fixed-supply token mints its whole supply once, at issuance; a mintable token calls back for each later mint. The contract acts for nothing else.

Contract-free, after issuance:

- **Transfer.** A holder spends unshielded UTXOs of the color and produces outputs to recipient `UserAddress` values, authorized by a Schnorr signature. No issuer interaction by sender or recipient.
- **Receipt.** A recipient detects an incoming UTXO by scanning the public ledger. No registration, no out-of-band coin delivery.
- **Balance.** Any party reads a color's balance from public ledger state.
- **Burn.** A holder spends a UTXO to an unspendable address (see Burn); the issuer is not involved.

A contract MAY also hold these tokens like any user, for example a treasury or a DeFi contract. That is an application choice, never a precondition for another party to hold or move the token.

### Color and the Issuer

A conforming token's color MUST be `tokenType(domain, issuerAddress)`, where `domain` is fixed at the issuer's construction and immutable thereafter. Immutability is load-bearing: a changed domain changes the color and orphans every outstanding UTXO. Wallets and indexers MUST treat color as the token's only identity; `name` and `symbol` are metadata and MUST NOT be used for identity or equality.

`tokenColor` MUST return `tokenType(domain, kernel.self())`, computed at call time. It MUST NOT be precomputed and stored in the constructor: a constructor runs before the contract has a deployed address, so `kernel.self()` there returns a placeholder rather than the real address, and a color derived from it would be wrong.

### Issuance

Issuance is the one contract-bound operation. A conforming issuer mints via:

```typescript
export circuit _mint(
  domain: Bytes<32>,
  recipient: Either<ContractAddress, UserAddress>,
  amount: Uint<64>
): Bytes<32>
```

1. MUST revert if `recipient` is a zero address.
2. MUST call `mintUnshieldedToken(domain, amount, recipient)` and return the color.
3. The `Uint<64>` cap is the protocol primitive's; larger issuance uses multiple mints.

An unshielded mint is publicly valued: recipient, color, and amount are visible, and the recipient's wallet detects the UTXO by scanning. A wallet recipient is `right<ContractAddress, UserAddress>(address)`; minting to the issuer itself uses `left<ContractAddress, UserAddress>(kernel.self())`. Because these values become part of the public transcript, the arguments passed to `mintUnshieldedToken` are wrapped in `disclose()`, which is expected here since an unshielded coin is public.

A **fixed-supply** issuer mints its entire supply in a one-shot circuit run once after deployment, then exposes no further mint. The mint runs after deployment rather than in the constructor because a constructor executes before the contract has a deployed address, so `kernel.self()` there returns a placeholder and would derive the wrong color. After that mint the contract has no role in the token's life; the supply exists as ordinary unshielded UTXOs.

A **mintable** issuer exposes `_mint` after deployment. Whether that mint is open to anyone or restricted to specific callers is an authorization decision, out of scope for this standard (see Authorization).

Because `_mint` takes `domain` as a parameter and the color is `tokenType(domain, kernel.self())`, one contract can issue several independent tokens, one per domain, each with its own color, supply, and metadata. This is how a single issuer, such as a bridge mirroring assets from several source chains, serves many tokens without deploying a contract per token. A single-token issuer simply fixes one domain (for example in a `sealed` ledger field) and ignores the parameter. Colors from different domains do not interfere: a coin of one never satisfies a spend or receive of another.

#### Authorization

Whether mint is open to anyone or restricted to specific callers is out of scope for this standard. `_mint` carries no authorization of its own; an issuer that wants to restrict minting composes an access-control policy over it. Open minting is a valid configuration, not an oversight: it suits tokens like an open-contribution collection, a community faucet, or a game piece anyone can create. Implementations that do restrict minting MUST NOT authenticate callers with `ownPublicKey()`, which is a caller-supplied witness not bound to the proof. A standard mechanism for restricting issuance is expected to be its own proposal.

### Transfer

Transfer is a native ledger operation, not a circuit in this standard. A holder spends unshielded UTXOs of the color and produces outputs to one or more recipient `UserAddress` values, authorized by a Schnorr signature. The issuer is not involved and the recipient need not have interacted with it. This mirrors the property MIP-0004 notes for tokens converted into native UTXOs: once native, they move freely, outside any contract's control.

A conforming wallet MUST be able to construct a transfer given only the color and the recipient `UserAddress`. The standard introduces no contract-derived per-user account identifier; doing so would add an address the wallet must track and force both parties through the contract before value could move. Because the transfer is a native UTXO spend, one transaction MAY pay several recipients via multiple outputs. That is a property of the ledger transaction model, not an interface defined here.

### Burn

Burning removes coins from circulation permanently, and it is a native operation, not a contract circuit. A contract can only send coins it holds, so a contract-mediated burn would first take custody of the supply, the custodial behavior this standard avoids. Instead a holder burns by spending a UTXO to an unspendable address: the all-zeroes `UserAddress`, a recipient of 32 zero bytes whose owner has no key, so no Schnorr signature can ever authorize a spend. This is an ordinary transfer whose destination happens to be unspendable; the issuer is not involved.

Burned supply is read from the live UTXO set rather than from a contract counter, since the contract never sees the burn. This mirrors how the standard treats circulating supply generally (see Supply): the on-chain view is a bound, and the indexed UTXO set is the authoritative figure.

One gap is worth noting for review. The shielded side has a blessed `shieldedBurnAddress()` primitive that guarantees an unspendable destination; the standard library has no unshielded equivalent (the name `burnAddress` is already taken in the library for the shielded case). Until one exists, an issuer or wallet constructs the target itself as the all-zeroes `UserAddress`, which works by the same key-absence logic but is not a library-guaranteed constant. This MIP recommends the standard library add an unshielded burn-address primitive mirroring the shielded one. Wallets and indexers must also recognize the burn address, since coins sent to the shielded burn address have in the past been surfaced as phantom spendable balance until special-cased.

### Metadata and Discovery

A token's on-chain footprint is just colored UTXOs; the color is a 32-byte hash, not a name. The question is how a wallet turns a color it sees in someone's balance into "USDC, 6 decimals." There are two paths, matching how Midnight already handles token metadata off-chain (ADR #15):

The first is direct. A DApp frontend that already holds the issuer's compiled contract calls the metadata circuits below and reads the values straight from ledger state. This is the path for a token the DApp itself issued or bundles.

The second is the registry, for the general case where a wallet encounters an unfamiliar color with no prior knowledge of the issuer. Metadata lives in an off-chain registry (PR #104) keyed by the token's color: the wallet looks the color up and gets back the display fields (name, ticker, decimals) and the issuer's `(domain, contract_address)`. Because the color is `tokenType(domain, contract_address)`, the wallet recomputes it from the returned pair and checks it equals the color it looked up. A mismatch means the entry is lying about which token it describes, and the wallet rejects it. This is what lets metadata be off-chain and still trustworthy: the binding is verifiable, not asserted.

```typescript
export circuit name(): Opaque<"string">
export circuit symbol(): Opaque<"string">
export circuit decimals(): Uint<8>
export circuit tokenColor(domain: Bytes<32>): Bytes<32>
```

The derivation check is mandatory in both paths. The registry's own mechanics are out of scope here; this MIP requires only that metadata is keyed by color and that a consumer recomputes and verifies the color before trusting an entry.

### Interoperability With a Shielded Representation

Midnight's dual ledger lets an asset exist in shielded and unshielded form, and the docs state that both NIGHT and custom tokens can move between the two states. What the standard library exposes today are two separate primitive families: `mintShielded`/`sendShielded`/`receiveShielded` over `ZswapCoinPublicKey`, and `mintUnshielded`/`sendUnshielded`/`receiveUnshielded` over `UserAddress`. There is no single primitive in the current library that converts one coin's color between its shielded and unshielded forms, so the two forms are distinct colors under distinct primitive families rather than one color with a privacy flag.

The exact mechanism by which an asset moves between forms is out of scope for this MIP, and this standard neither performs nor specifies it. What matters here is that the standard does not obstruct it: a conforming unshielded token uses the ordinary `tokenType` color space and integer value semantics, so a future conversion mechanism, whether a dedicated conversion contract or a protocol-level path, can pair it with a shielded counterpart without this standard needing to change. An issuer weighing both forms SHOULD note that "the same asset in two forms" is a relationship enforced by whatever conversion mechanism is used, not an intrinsic property of a shared color, and that the transparency guarantees of the two sides differ. NIGHT illustrates the shape of the unshielded side; defining how a shielded representation of such an asset is produced and kept in supply lockstep is a separate problem, appropriate for a conversion-focused proposal rather than this token definition.

### Supply

Exact circulating supply is not knowable for a native token. Coins leave circulation without any contract observing it, whether burned to an unspendable address or stranded by lost keys, so a contract cannot state circulating supply exactly. The authoritative figure is the indexed sum of the live UTXO set for the color, which is possible here because all unshielded values are public.

An issuer MAY additionally expose an OPTIONAL mint counter:

```typescript
export circuit totalMinted(domain: Bytes<32>): Uint<128>
```

`totalMinted(domain)` is exact and monotonic: color derivation guarantees every coin of the color comes from this issuer's mints, and each mint increments it. It is an upper bound on circulating supply (it never decreases as coins are burned), useful as a mint-exact reference for consumers that cannot run an indexer. It is independently verifiable from public mint records, so indexers can flag a non-conforming implementation.

### Conformance

A token conforms if and only if:

1. Its color is `tokenType(domain, issuerAddress)` for a `domain` set immutably at construction.
2. New supply is created only via `mintUnshieldedToken` through that issuer; a fixed-supply token does so only at issuance.
3. It exposes `name`, `symbol`, `decimals`, and `tokenColor`.
4. Holding, transfer, receipt, balance inspection, and burning require no issuer-contract interaction by any party; a transfer is constructible from color and recipient `UserAddress` alone.
5. Any restriction on who may mint uses in-circuit verification, never `ownPublicKey()`. The standard does not require minting to be restricted.
6. If a mint counter is exposed, it is exact and monotonic per the Supply section.

### Versioning

Revisions to this standard follow the normal MIP process; no bespoke versioning scheme is introduced here.

### Out of Scope

The following are deliberately not covered by this standard:

- **Authorization.** Whether minting is open or restricted is left to the composer (see [Authorization](#authorization)). A standard mechanism for restricting issuance is expected to be its own proposal.
- **Transfer mediation and post-issuance controls.** Pause, freeze, clawback, and transfer restrictions are not representable once a user holds a coin; they depend on protocol capabilities under separate discussion ([MPS-0013](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0013-zswap-business-logic.md), [MPS-0021](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0021-phase2-contract-to-contract.md)).
- **Shield/unshield conversion.** Moving an asset between its unshielded and shielded representations spans two primitive families and belongs in a conversion-focused proposal (see [Interoperability With a Shielded Representation](#interoperability-with-a-shielded-representation)).
- **Balances, allowances, and batch transfers.** These belong to the account model that [MIP-0004](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md) covers, not to a native bearer token.

## Rationale

### Why the standard is built around native movement rather than an issuer balance

The protocol binds only supply operations to a contract, through `tokenType`; transfer, receipt, and balance are native. A standard could still route these through the issuer for convenience, but doing so would recreate the account model of MIP-0004 and forfeit the one thing this token type offers: a token that behaves as a bearer instrument, moving between holders with no dependency on the issuer being live, cooperative, or even still deployed. Building on the native operations preserves that property; building on an issuer balance would discard it.

### Why a separate standard from MIP-0011 and MIP-0004

The three target different asset models. MIP-0004 is account-based, with the contract as source of truth and exact supply. MIP-0011 covers assets that exist only as shielded native coins. This covers assets that exist only as unshielded native coins. The shielded and unshielded native cases share a structure but differ in guarantees: unshielded coins are publicly valued, so balances and an indexed circulating supply are observable where the shielded case hides them. Parallel standards let an issuer pick a representation without inheriting the other's properties, consistent with the Discussion #142 decision not to collapse the token types.

### Why issuance is contract-rooted but everything else is not

Color derivation needs an issuer address, which gives every token a verifiable origin at mint time. That value is real once, at creation, and would be pure overhead if charged on every transfer. The standard concentrates the contract at issuance and removes it everywhere else, which is also how the protocol treats NIGHT and other native coins.

### Why supply is bounded, not exact

Coins can leave circulation without the contract observing it, by being burned to an unspendable address or stranded when an owner loses the spending key, so a contract can never know the exact circulating figure. The honest interface is an optional exact `totalMinted` counter as an upper bound, with the indexed live-UTXO sum as the authoritative circulating figure. Nothing about circulating supply needs to live on-chain, because an unshielded issuer can defer it to an indexer summing the public UTXO set, an option the shielded case does not have.

### Why burn is native here but a contract circuit in MIP-0011

[MIP-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md) exposes `_burn` circuits because a shielded issuer commonly custodies coins in a vault and burns coins it holds. This standard takes the opposite stance on purpose: an unshielded issuer is not a custodian and holds no coins to burn, and a contract can only send coins it holds. A contract-mediated burn would therefore have to first receive the holder's coins into the contract, reintroducing exactly the custody this standard avoids. Burning is instead native: the holder spends to an unspendable address directly, with no contract in the loop. The trade-off is that the contract cannot count burns, which is why burned supply is read from the indexed UTXO set rather than an on-chain counter.

### Why metadata is off-chain and color-keyed

Ledger state is public and untrusted for identity, so on-chain `name`/`symbol` would invite impersonation without removing the need for a derivation check. Color-keyed off-chain metadata plus a mandatory `(domain, issuer)` recomputation is lighter and safer. Identity is the color; the name is a label.

### Alternatives considered

- Modeling the token as a MIP-0004 contract-Map balance. Rejected: it charges a contract interaction on every transfer and prevents native movement, defeating the purpose of a native token.
- Defining shield/unshield in this standard. Rejected: conversion between forms is a separate concern spanning two distinct primitive families, and its mechanism is out of scope here; it belongs in a conversion-focused proposal, not this token definition.
- On-chain `name`/`symbol`. Rejected for impersonation and the persistence of the derivation-check requirement.

## Path to Active

### Acceptance Criteria

- A reference implementation covering fixed-supply and mintable issuers, with a test suite that runs against a live network, devnet at a minimum and preferably preview or preprod.
- A Midnight testnet deployment exercising issuance, a later mint on the mintable variant, and a native burn (spend to the all-zeroes `UserAddress`).
- A demonstrated wallet-to-wallet transfer constructed from color and recipient `UserAddress` alone, with neither party interacting with the issuer, and a multi-recipient transfer.
- A wallet or indexer that derives color from `(domain, issuer)`, resolves color-keyed metadata, and reports an indexed circulating supply.
- Review through the MIP process workshops, including the Wallets and DApps Working Group.

### Implementation Plan

1. Land the reference native unshielded token module and the fixed-supply and mintable example issuers, plus an optional mint counter.
2. Add unit and simulator coverage for all behaviors including revert cases, then validate against a live network, devnet at a minimum.
3. Coordinate metadata resolution with PR #104.
4. Promote through the public networks to preprod, then submit for formal MIP review.

## Backwards Compatibility Assessment

Purely additive: a new convention over existing primitives. The only primitive it requires is `mintUnshieldedToken` (plus `tokenType` for color derivation), both already in the Compact Standard Library. It does not conflict with MIP-0004 or MIP-0011; the three target different models and can coexist, including in one hybrid contract. Tokens issued here are ordinary unshielded UTXOs and interoperate with existing wallets and tooling that handles unshielded coins. The `recipient` parameter accepts a `ContractAddress` branch as well as `UserAddress`, so the issuance interface is forward-compatible with contract-to-contract calls when they land.

## Security Considerations

### Open issuance

The core `_mint` carries no authorization, and this is deliberate: permissionless minting is a valid configuration with real uses, an open-contribution collection, a community faucet, a game piece anyone can create. Because there is no restriction baked in, an issuer that does want one composes an access-control policy over `_mint` without changing the token. The one hard rule for those who restrict minting is to verify the caller in-circuit and never on `ownPublicKey()`, which is a caller-supplied witness not bound to the proof and so trivially forgeable. A standard restriction mechanism is out of scope here and expected to be its own proposal. Note that an ungated `_mint` is, by design, an unlimited-supply token; that is the point for the open case, and the caveat for anyone who assumed otherwise.

### Color collisions and mint-first impersonation

Identity rests on `tokenType(domain, contractAddress)`. Distinct issuers cannot produce the same color without a hash collision. The realistic attack is impersonation by metadata: an attacker deploys their own contract and mints a token whose `name`/`symbol` claims to be a known asset. Its color derives from the attacker's address, not the genuine issuer's, so it is a different token, and an attacker cannot mint into the genuine color because only the genuine issuer's address derives it. The defense is the mandatory derivation check: a wallet MUST recompute color from a metadata entry's `(domain, issuer)` and reject any mismatch. Which `(domain, issuer)` is canonical for a brand is the registry's trust problem, not the token's.

### Domain immutability

A mutable domain would change a token's color and strand outstanding UTXOs. The domain MUST be immutable, with no mutating circuit.

### Supply interpretation

`totalMinted`, if exposed, is independently verifiable from public mint records, so indexers can flag a non-conforming implementation. It is an upper bound on circulating supply and SHOULD be presented as such; the indexed live-UTXO sum is the true circulating figure.

### No post-issuance control

Once minted, coins are unconditionally transferable bearer instruments: no pause, freeze, clawback, or transfer restriction at this layer, which is the direct consequence of the contract-free transfer model. A compliance-bound issuer should treat this standard as a phase-one primitive and track the custom-spend-logic work (MPS-0013, MPS-0021) for restrictions that survive user custody.

### Privacy expectations

Native unshielded tokens are public by design: balances, transfers, and ownership by `UserAddress` are visible on-chain. This standard makes no privacy guarantee for this quadrant. Wallets SHOULD make this clear, particularly for assets mirrored from another chain whose users may carry different assumptions, and for any asset that also has a shielded representation, where users may not realize the unshielded side is fully transparent.

## Implementation

### Components

1. Compact modules: a core native unshielded token module (mint, metadata, color derivation) and two example issuers, fixed-supply and mintable, that compose it. An optional mint counter can be added on top.
2. In-memory simulators and unit tests exercising the circuits.
3. No protocol or language changes are required; the standard is a convention over primitives already in the Compact Standard Library.

A reference implementation exists and compiles against the current toolchain (Compact 0.31.x): <https://github.com/JAlbertCode/example-mip-0014>.

### Dependencies

- Compact Standard Library: `mintUnshieldedToken`, `tokenType`, and, for an optional mint counter, `Counter`.
- Off-Chain Token Metadata Registry (PR #104) for third-party metadata resolution.
- A conversion mechanism such as MIP-0004's, only for issuers that also want a shielded representation; not a dependency of this standard itself.

## Testing

### Unit Tests

- Construction: metadata getters return constructor values; the domain is immutable; a fixed-supply issuer exposes no post-issuance mint.
- `_mint`: returns a color equal to `tokenColor(domain)`; mints the correct value to the recipient; reverts on a zero recipient; distinct domains derive distinct colors.
- Mint counter (if exposed): increments by the minted amount; distinct domains accumulate independently; overflow guard.

### Integration Tests

- Contract-free transfer: a wallet-to-wallet transfer constructed from color and recipient `UserAddress` alone, with neither party interacting with the issuer; confirm the issuer is not called.
- Multi-recipient transfer: one transaction pays several `UserAddress` outputs from one holder's UTXOs.
- Burn: a holder spends a UTXO to the all-zeroes `UserAddress`; confirm the coins cannot subsequently be spent and the indexed live-UTXO sum for the color drops.
- Supply bound: strand coins by sending them to an address whose key is discarded, then confirm the indexed live-UTXO sum falls below an exposed `totalMinted`, demonstrating the upper-bound property.
- Metadata resolution: a wallet recomputes color from a registry entry's `(domain, issuer)` and rejects an entry whose derivation does not match the claimed color.

## References

- [MIP-0001: Midnight Improvement Proposal Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-mip-process.md)
- [MIP-0004: Fungible Token Standard with UTXO Conversion Extensions](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md) (freely transferable native UTXOs; the conversion model referenced for shielded interoperability).
- [MIP-0011: Native Shielded Token Standard](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md) (the shielded sibling; the bounded-supply model reused here).
- [MPS-0003: CAIP Support](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md) (for issuers that record a source-chain identifier in metadata).
- [MPS-0027: Domain Separation](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0027-domain-separation.md) (the ecosystem-wide domain-separator standard this MIP's 32-byte `domain` aligns with).
- [Discussion #142: Token Standards](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/142) (the six-token-type taxonomy; the color-and-`UserAddress` transfer requirement).
- [Discussion #136: MIP-0004 review](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/136) (all ledger fields are public; native UTXOs are not contract-controlled).
- [The Compact Language](https://docs.midnight.network/compact) (the standard library and `tokenType`, `mintUnshieldedToken`).
- [Midnight docs, Tokens](https://docs.midnight.network/tokens): NIGHT as an unshielded ledger token; the dual-ledger shielded/unshielded model.

## Acknowledgements

This proposal builds on the token-standards taxonomy in Discussion #142, the native-token structure and bounded-supply model established by MIP-0011, and the ledger-visibility findings in Discussion #136.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
