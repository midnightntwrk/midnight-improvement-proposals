---
MIP: "0012"
Title: Native Unshielded Token Standard
Authors:
  - Jay Albert
Status: Draft
Category: Standards
Created: 2026-07-14
Requires: none
Replaces: none
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

This MIP specifies a standard for native unshielded tokens on Midnight: fungible assets that live as protocol-level unshielded UTXOs, publicly valued and owned by a `UserAddress`. The defining property is that the issuing contract is an issuer, not a custodian. A contract is required to bring a token into existence, because minting is a contract circuit and a token's color is bound to the minting contract's address by `tokenType`, and to authorize any later mint or burn. But holding, transfer, receipt, and balance inspection require no contract at all: they are native ledger operations under a Schnorr signature, with no requirement that a recipient has ever interacted with the issuer. A fixed-supply token therefore touches its contract only once, at genesis, and moves peer-to-peer for the rest of its life.

The standard specifies:

- how a conforming token derives its color;
- how new supply is minted and authorized at the single contract-bound point;
- how value moves natively between holders;
- how metadata is bound to a token's color and discovered by wallets;
- how a token is burned; and
- how supply is reported as an honest bound rather than an exact figure.

It is the transparent sibling of MIP-0011 (native shielded tokens) and is designed so that a shielded representation of the same asset can be paired with it by a separate conversion mechanism, such as the UTXO conversion MIP-0004 describes. It deliberately does not cover transfer mediation, post-issuance controls, or the shield/unshield conversion itself, each of which lives elsewhere.

## Motivation

Midnight's token-standards effort is organized around six quadrants (Discussion #142): shielded native, unshielded native, shielded contract, unshielded contract, and two hybrid types. MIP-0011 specifies the native shielded quadrant and MIP-0004 the contract quadrants. This MIP fills the native unshielded quadrant, whose canonical example is NIGHT: the docs classify NIGHT as an unshielded ledger token, transparent and UTXO-based, used for transparent value transfer. NIGHT itself is the protocol's built-in token and is not issued through this standard, but its shape is the shape this standard generalizes for issued assets.

The protocol already supports these tokens directly. `mintUnshieldedToken` mints a coin of a contract-derived color to a `UserAddress`, a holder spends an unshielded UTXO at the ledger layer under a Schnorr signature, `unshieldedBalance` lets a contract read its own balance of a color, and `sendUnshielded`/`receiveUnshielded` let a contract move or absorb these coins when one chooses to involve a contract. What is missing is an agreed convention for how an issued token derives its color, authorizes new supply, and exposes its metadata, so that wallets, indexers, and bridges recognize and handle it uniformly. Without that, every issuer rebuilds the surface differently and the tokens are not portable.

The distinction from the other token standards is not whether a contract is used to mint. All three use a contract to mint, because minting is always a contract circuit. The distinction is how long the contract stays in the loop after that:

- MIP-0004 (contract token): the contract is a permanent custodian. Balances live in its Map, and every transfer is a contract call. The contract is in the loop for the token's whole life.
- MIP-0011 (native shielded token): the contract mints and burns, then steps out. The coin lives as a shielded Zswap UTXO and moves peer-to-peer with no contract.
- This standard (native unshielded token): same shape as MIP-0011, the contract mints then steps out, but the UTXO is publicly valued and owned by a `UserAddress`.

This is the property Discussion #142 asked for directly: value must move to a recipient known only by `UserAddress`, with neither party first transacting with a contract. A standard that required a contract round trip to recognize or transfer a token would just be describing a contract token, which MIP-0004 already covers.

This proposal references its neighbors to reuse settled decisions rather than relitigate them. The bounded-supply model is the same one MIP-0011 establishes for the shielded case, because the underlying limit is identical: out-of-band destruction is invisible to the contract. The mint-authorization requirement is the hash-based pattern MIP-0004 establishes, because the hazard is identical: `ownPublicKey()` is caller-supplied. The shielded-interoperability section below is written so that an issuer who later wants a shielded representation of the same asset can reach for a conversion mechanism, such as MIP-0004's UTXO conversion, without this standard having to define one.

## Specification

### Terminology

- **Native unshielded token**: a class of unshielded UTXOs sharing one color, whose lifecycle is native to the ledger except for the contract-bound supply operations, mint and burn.
- **Color (token type)**: `tokenType(domain, contractAddress)` per the Compact Standard Library. Only the contract at `contractAddress` can mint coins of its colors.
- **Domain separator (`domain`)**: a 32-byte value that, hashed with the contract address, produces one token color. Since the color is `tokenType(domain, contractAddress)`, a single contract yields a distinct color for each distinct domain, so one contract can issue many independent token types.
- **`UserAddress`**: a user public key address. At the Compact layer it is `struct UserAddress { bytes: Bytes<32>; }`, a 32-byte value; wallets present it Bech32m-encoded with the `mn_addr` human-readable prefix and a network identifier (for example `mn_addr_test1...`).
- **Issuer contract**: the contract whose address derives a token's color and which holds its mint and burn authority. Needed to create supply and, for mintable tokens, at each later mint or burn; needed for nothing else.

### Token Lifecycle

A conforming token's lifecycle separates into contract-bound supply operations and a contract-free middle. The contract is an issuer, not a custodian: it is in the loop only to create or retire supply, never to hold or move it between users.

Contract-bound, at supply changes: **mint** and **burn**. Minting is always a contract circuit, `mintUnshieldedToken`, and the color it produces is `tokenType(domain, contractAddress)`, so a color can only originate from the contract whose address derives it. There is no contractless mint. Burn is also a contract operation: it sends coins to an unspendable output for permanent removal and emits a burn event for accounting, mirroring how MIP-0011 burns shielded tokens, and is covered under Burn. A fixed-supply token mints its whole supply once, at genesis; a mintable token calls back for each later mint; and either MAY expose a burn circuit. The contract acts for nothing else.

Contract-free, between supply changes:

- **Transfer.** A holder spends unshielded UTXOs of the color and produces outputs to recipient `UserAddress` values, authorized by a Schnorr signature. No issuer interaction by sender or recipient.
- **Receipt.** A recipient detects an incoming UTXO by scanning the public ledger. No registration, no out-of-band coin delivery.
- **Balance.** Any party reads a color's balance from public ledger state.

A contract MAY also hold these tokens like any user, for example a treasury or a DeFi contract using `sendUnshielded`/`receiveUnshielded`. That is an application choice, never a precondition for another party to hold or move the token.

### Color and the Issuer

A conforming token's color MUST be `tokenType(domain, issuerAddress)`, where `domain` is fixed at the issuer's construction and immutable thereafter. Immutability is load-bearing: a changed domain changes the color and orphans every outstanding UTXO. Wallets and indexers MUST treat color as the token's only identity; `name` and `symbol` are metadata and MUST NOT be used for identity or equality.

`tokenColor` MUST return `tokenType(domain, kernel.self())`, computed at call time. It MUST NOT be precomputed and stored in the constructor: a constructor runs before the contract has a deployed address, so `kernel.self()` there returns a placeholder rather than the real address, and a color derived from it would be wrong.

### Issuance

Issuance is the one contract-bound operation. A conforming issuer mints via:

```compact
export circuit _mint(
  domain: Bytes<32>,
  recipient: Either<ContractAddress, UserAddress>,
  amount: Uint<64>
): Bytes<32>
```

1. MUST revert if `recipient` is a zero address.
2. MUST call `mintUnshieldedToken(domain, amount, recipient)` and return the color.
3. The `Uint<64>` cap is the protocol primitive's; larger issuance uses multiple mints.
4. A consumer composing the supply extension MUST pair each mint with `_addMinted(domain, amount)`.

An unshielded mint is publicly valued: recipient, color, and amount are visible, and the recipient's wallet detects the UTXO by scanning. A wallet recipient is `right<ContractAddress, UserAddress>(address)`; minting to the issuer itself uses `left<ContractAddress, UserAddress>(kernel.self())`. Because these values become part of the public transcript, the arguments passed to `mintUnshieldedToken` are wrapped in `disclose()`, which is expected here since an unshielded coin is public by construction.

A **fixed-supply** issuer mints its entire supply in the constructor and exposes no further mint circuit. After construction the contract has no role in the token's life; the supply exists as ordinary unshielded UTXOs.

A **mintable** issuer exposes `_mint` under the access control described in Mint Authorization. This is the only case where the contract is touched after genesis, and only by the minter, never by holders or recipients.

Because `_mint` takes `domain` as a parameter and the color is `tokenType(domain, kernel.self())`, one contract can issue several independent tokens, one per domain, each with its own color, supply, and metadata. This is how a single issuer, such as a bridge mirroring assets from several source chains, serves many tokens without deploying a contract per token. A single-token issuer simply fixes one domain (for example in a `sealed` ledger field) and ignores the parameter. Colors from different domains do not interfere: a coin of one never satisfies a spend, receive, or burn of another.

#### Mint authorization

A `_mint` or `_burn` circuit MUST be gated. Implementations MUST NOT authenticate callers with `ownPublicKey()`: it is a caller-supplied witness, not bound to the proof. The conforming pattern stores the authorized identity as a `Bytes<32>` commitment and verifies the caller in-circuit against a secret. The secret MAY be supplied via a witness or as a private circuit input; both become private inputs and are equivalent in security (Discussion #136), so the requirement is in-circuit verification against a committed secret, not a delivery mechanism.

```compact
witness secretKey(): Bytes<32>;

export ledger minter: Bytes<32>;

circuit authPublicKey(sk: Bytes<32>): Bytes<32> {
  return persistentHash<Vector<2, Bytes<32>>>(
    [pad(32, "midnight:native-unshielded:minter"), sk]
  );
}

circuit assertMinter(): [] {
  assert(minter == authPublicKey(secretKey()), "only minter");
}
```

The domain-separator string prevents a secret from being replayed across unrelated contracts.

### Transfer

Transfer is a native ledger operation, not a circuit in this standard. A holder spends unshielded UTXOs of the color and produces outputs to one or more recipient `UserAddress` values, authorized by a Schnorr signature. The issuer is not involved and the recipient need not have interacted with it. This mirrors the property MIP-0004 notes for tokens converted into native UTXOs: once native, they move freely, outside any contract's control.

A conforming wallet MUST be able to construct a transfer given only the color and the recipient `UserAddress`. The standard introduces no contract-derived per-user account identifier; doing so would add an address the wallet must track and force both parties through the contract before value could move. Because the transfer is a native UTXO spend, one transaction MAY pay several recipients via multiple outputs. That is a property of the ledger transaction model, not an interface defined here.

### Burn

A burn should remove coins from circulation permanently. The ledger enforces balance preservation, so coins cannot be destroyed by leaving a transaction short; they must go somewhere. The only destination that guarantees permanent removal is an output nobody can ever spend: an all-zeroes address whose owner has no key, so no signature can authorize a spend. This is the model the sibling standard MIP-0011 uses for shielded tokens, where its burn sends coins to `shieldedBurnAddress()`, and this standard follows it so that both token types burn the same way.

A conforming burn therefore sends the coins to an unspendable unshielded output and emits the standard `UnshieldedBurn` contract event (MIP-0002), which records the token type and amount:

```compact
export circuit _burn(domain: Bytes<32>, amount: Uint<128>): []
```

Accounting comes from the event, not from any held balance: an indexer sums `UnshieldedBurn` events to track burned supply, and under the optional supply extension the circuit also increments `_addBurned(domain, amount)`. This matches how MIP-0011 accounts shielded burns through `emitShieldedBurn`. Because the coins land on an unspendable output rather than in the contract, they are gone irreversibly, the same guarantee an EVM `burn()` or a Cardano negative-mint gives, and there is no held balance that could be re-sent.

There is one gap this standard cannot close on its own, and it is the central open question for review. The shielded side has a blessed `shieldedBurnAddress()` primitive that guarantees an unspendable destination; the current standard library has no `unshieldedBurnAddress()` equivalent. Until one exists, an unshielded issuer must construct the unspendable output itself, typically the all-zeroes `UserAddress`, which works by the same key-absence logic but is not a library-guaranteed constant. This MIP RECOMMENDS that the standard library add an `unshieldedBurnAddress()` primitive mirroring the shielded one, so that the unshielded burn is as safe and as canonical as the shielded burn it parallels. Wallets and indexers must also recognize the burn address, since coins sent to the shielded burn address have in the past been surfaced as phantom spendable balance until special-cased.

Retiring coins into the issuer via `receiveUnshielded` is NOT a burn under this standard. Those coins remain in the contract's balance and can be re-sent with `sendUnshielded`, so they have not left circulation; that pattern is a treasury or escrow operation, not destruction. An issuer that wants a held reserve MAY use it, but MUST NOT report the held amount as burned.

Even with an irreversible burn, an issuer's counter is a lower bound on total destruction, because coins can also leave circulation by key loss, which no counter observes. This is one reason supply is reported as a bound rather than an exact figure.

### Metadata and Discovery

A token's on-chain footprint is just colored UTXOs; the color is a 32-byte hash, not a name. So the question is how a wallet turns a color it sees in someone's balance into "USDC, 6 decimals." Two paths, matching how Midnight already handles token metadata off-chain (ADR #15):

The first is direct. A DApp frontend that already holds the issuer's compiled contract calls the metadata circuits below and reads the values straight from ledger state. This is the path for a token the DApp itself issued or bundles.

The second is the registry, for the general case where a wallet encounters an unfamiliar color with no prior knowledge of the issuer. Metadata lives in an off-chain registry (PR #104) keyed by the token's color: the wallet looks the color up and gets back the display fields (name, ticker, decimals) and the issuer's `(domain, contract_address)`. Because the color is `tokenType(domain, contract_address)`, the wallet recomputes it from the returned pair and checks it equals the color it looked up. A mismatch means the entry is lying about which token it describes, and the wallet rejects it. This is what lets metadata be off-chain and still trustworthy: the binding is verifiable, not asserted.

```compact
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

Exact circulating supply is not knowable for a native token. Coins can leave circulation without a counter observing it, whether burned to an unspendable output or stranded by lost keys, so a contract can only ever bound the supply, not state it exactly. An OPTIONAL supply extension tracks the honest bounds:

```compact
export circuit totalMinted(domain: Bytes<32>): Uint<128>
export circuit totalBurned(domain: Bytes<32>): Uint<128>
export circuit totalSupply(domain: Bytes<32>): Uint<128>
```

- `totalMinted(domain)` is exact: color derivation guarantees every coin of the color comes from this issuer's mints, and each mint MUST increment it.
- `totalBurned(domain)` is a lower bound: it counts burns that went through the contract's burn circuit, not coins stranded by key loss.
- `totalSupply(domain)` MUST equal `totalMinted` minus `totalBurned`, which is an upper bound on circulating supply. Actual circulating supply is at most this figure and may be less, since lost or otherwise stranded coins are not counted as burned.

For the unshielded case an indexer can compute a tighter figure than the contract can, by summing the live UTXO set for the color, since all values are public. Integrators SHOULD treat the contract's `totalSupply` as an upper bound and the indexed UTXO-set sum as the authoritative circulating figure. The on-chain counter remains useful as a mint-exact reference for consumers that cannot run an indexer.

### Conformance

A token conforms if and only if:

1. Its color is `tokenType(domain, issuerAddress)` for a `domain` set immutably at construction.
2. New supply is created only via `mintUnshieldedToken` through that issuer; a fixed-supply token does so only at genesis.
3. Any mint or burn circuit is gated by in-circuit verification against a committed secret, never `ownPublicKey()`.
4. It exposes `name`, `symbol`, `decimals`, and `tokenColor`.
5. Holding, transfer, receipt, and balance inspection require no issuer-contract interaction by any party; a transfer is constructible from color and recipient `UserAddress` alone.
6. If supply is tracked, it follows the bounded model and is presented as a bound.

### Versioning

Revisions to this standard follow the normal MIP process; no bespoke versioning scheme is introduced here. The authorization commitment uses a domain-separated string, `midnight:native-unshielded:minter`, purely to scope the secret to this use and prevent it from being replayed against another contract's commitment, following the plain-string convention used by Midnight's in-contract hash-authentication examples. It carries no version suffix: a contract's auth commitment has no chain-wide compatibility to preserve, and a future change to the scheme would ship as a new module with its own separator.

## Rationale

### Why the standard is built around native movement rather than an issuer balance

The protocol binds only supply operations to a contract, through `tokenType`; transfer, receipt, and balance are native. A standard could still route these through the issuer for convenience, but doing so would recreate the account model of MIP-0004 and forfeit the one thing this quadrant offers: a token that behaves as a bearer instrument, moving between holders with no dependency on the issuer being live, cooperative, or even still deployed. Building on the native operations preserves that property; building on an issuer balance would discard it.

### Why a separate standard from MIP-0011 and MIP-0004

The three target different asset models. MIP-0004 is account-based, with the contract as source of truth and exact supply. MIP-0011 covers assets that exist only as shielded native coins. This covers assets that exist only as unshielded native coins. The shielded and unshielded native cases share a structure but differ in guarantees: unshielded coins are publicly valued, so balances and an indexed circulating supply are observable where the shielded case hides them. Parallel standards let an issuer pick a representation without inheriting the other's properties, consistent with the Discussion #142 decision not to collapse quadrants.

### Why issuance is contract-rooted but everything else is not

Color derivation needs an issuer address, which gives every token a verifiable origin at mint time. That value is real once, at creation, and would be pure overhead if charged on every transfer. The standard concentrates the contract at issuance and removes it everywhere else, which is also how the protocol treats NIGHT and other native coins.

### Why supply is bounded, not exact

Coins can leave circulation without the contract observing it, most simply by being stranded when their owner loses the spending key, so the contract can only ever know an upper bound. Naming `totalMinted`, `totalBurned`, and an upper-bound `totalSupply` is the honest interface, and is the same model MIP-0011 adopts. Supply tracking is an extension, not core, because an unshielded issuer can defer the authoritative circulating figure to an indexer summing the public UTXO set.

### Why metadata is off-chain and color-keyed

Ledger state is public and untrusted for identity, so on-chain `name`/`symbol` would invite impersonation without removing the need for a derivation check. Color-keyed off-chain metadata plus a mandatory `(domain, issuer)` recomputation is lighter and safer. Identity is the color; the name is a label.

### Alternatives considered

- Modeling the token as a MIP-0004 contract-Map balance. Rejected: it charges a contract interaction on every transfer and prevents native movement, defeating the quadrant.
- Defining shield/unshield in this standard. Rejected: conversion between forms is a separate concern spanning two distinct primitive families, and its mechanism is out of scope here; it belongs in a conversion-focused proposal, not this token definition.
- On-chain `name`/`symbol`. Rejected for impersonation and the persistence of the derivation-check requirement.

## Path to Active

### Acceptance Criteria

- A reference implementation, ideally an unshielded counterpart to the MIP-0011 modules in the OpenZeppelin Compact Contracts library, with a simulator-based test suite, covering fixed-supply and mintable issuers.
- A Midnight testnet deployment exercising genesis issuance, a later mint on the mintable variant, the bounded supply getters, and a burn to an unspendable output with its `UnshieldedBurn` event.
- A demonstrated wallet-to-wallet transfer constructed from color and recipient `UserAddress` alone, with neither party interacting with the issuer, and a multi-recipient transfer.
- A wallet or indexer that derives color from `(domain, issuer)`, resolves color-keyed metadata, and reports an indexed circulating supply.
- Review through the MIP process workshops, including the Wallets and DApps Working Group.

### Implementation Plan

1. Land reference native unshielded token modules (fixed-supply and mintable, single-color and multi-color issuers) plus an optional supply extension.
2. Add simulator and unit coverage for all behaviors, including revert cases.
3. Coordinate metadata resolution with PR #104.
4. Deploy to testnet, then submit for formal MIP review.

## Backwards Compatibility Assessment

Purely additive: a new convention over existing primitives. Every primitive it uses exists in the current Compact Standard Library: `mintUnshieldedToken`, `sendUnshielded`, `receiveUnshielded`, `unshieldedBalance`, and `tokenType`. It does not conflict with MIP-0004 or MIP-0011; the three target different models and can coexist, including in one hybrid contract. Tokens issued here are ordinary unshielded UTXOs and interoperate with existing wallets and tooling that handles unshielded coins. The `recipient` parameter accepts a `ContractAddress` branch as well as `UserAddress`, so the issuance interface is forward-compatible with contract-to-contract calls when they land.

## Security Considerations

### Unrestricted issuance

The core module is deliberately unrestricted: `_mint` and `_burn` carry no authorization of their own, the same way MIP-0011's module leaves access control to the composer. This keeps the token logic and the authorization policy separate, so an issuer can choose single-owner, multisig, role-based, or constructor-only issuance without the standard prescribing one.

The consequence is that authorization is the consumer's responsibility, and an ungated `_mint` is an infinitely mintable token. A consumer that exposes mint or burn after genesis MUST gate it, and MUST NOT gate it on `ownPublicKey()`, which is a caller-supplied witness not bound to the proof. The reference implementation SHOULD ship the hash-commitment gate shown under Mint Authorization as a ready-to-compose access-control extension rather than as part of the core token, so issuers apply it by composition. Secret leakage equals minter compromise; treat the minter secret like a wallet signing key. A fixed-supply issuer avoids this surface entirely by minting only in the constructor and exposing no mint circuit.

### Color collisions and mint-first impersonation

Identity rests on `tokenType(domain, contractAddress)`. Distinct issuers cannot produce the same color without a hash collision. The realistic attack is impersonation by metadata: an attacker deploys their own contract and mints a token whose `name`/`symbol` claims to be a known asset. Its color derives from the attacker's address, not the genuine issuer's, so it is a different token, and an attacker cannot mint into the genuine color because only the genuine issuer's address derives it. The defense is the mandatory derivation check: a wallet MUST recompute color from a metadata entry's `(domain, issuer)` and reject any mismatch. Which `(domain, issuer)` is canonical for a brand is the registry's trust problem, not the token's.

### Domain immutability

A mutable domain would change a token's color and strand outstanding UTXOs. The domain MUST be immutable, with no mutating circuit.

### Supply interpretation

`totalMinted` is independently verifiable from public mint records, so indexers can flag non-conforming implementations. `totalSupply` is an upper bound and SHOULD be presented as such; the indexed UTXO-set sum is the tighter circulating figure.

### No post-issuance control

Once minted, coins are unconditionally transferable bearer instruments: no pause, freeze, clawback, or transfer restriction at this layer, which is the direct consequence of the contract-free transfer model. A compliance-bound issuer should treat this standard as a phase-one primitive and track the custom-spend-logic work (MPS-0013, MPS-0021) for restrictions that survive user custody.

### Privacy expectations

Native unshielded tokens are public by design: balances, transfers, and ownership by `UserAddress` are visible on-chain. This standard makes no privacy guarantee for this quadrant. Wallets SHOULD make this clear, particularly for assets mirrored from another chain whose users may carry different assumptions, and for any asset that also has a shielded representation, where users may not realize the unshielded side is fully transparent.

## Implementation

### Components

1. Compact modules: fixed-supply and mintable native unshielded token issuers (single-color and multi-color) composed with the library's `Utils` module, plus an optional supply extension. The intent is to add these alongside the MIP-0011 native-token modules in the OpenZeppelin Compact Contracts library, reusing their structure.
2. Mocks, simulators, and unit tests exposing the circuits.
3. No protocol or language changes are required; the standard is a convention over primitives already in the Compact Standard Library.

### Dependencies

- Compact Standard Library: `mintUnshieldedToken`, `sendUnshielded`, `receiveUnshielded`, `unshieldedBalance`, `tokenType`, `persistentHash`, `pad`, `Counter`.
- OpenZeppelin Compact Contracts: the `Utils` module and the MIP-0011 native-token modules as the structural precedent.
- Off-Chain Token Metadata Registry (PR #104) for third-party metadata resolution.
- A MIP-0004-style conversion contract, only for issuers that also want a shielded representation; not a dependency of this standard itself.

## Testing

### Unit Tests

- Construction: metadata getters return constructor values; the domain is immutable; a fixed-supply issuer exposes no post-genesis mint.
- `_mint`: returns a color equal to `tokenColor(domain)`; mints the correct value to the recipient; reverts on a zero recipient; distinct domains accumulate independent supplies under the extension.
- `_burn`: reverts when provided inputs do not cover `amount`; increments `totalBurned` under the extension.
- Supply extension: `totalSupply == totalMinted - totalBurned` after arbitrary sequences; unknown domains read 0; overflow guard on `_addMinted`.
- Authorization: a caller proving the committed secret succeeds; a wrong or absent secret fails the in-circuit assertion; an `ownPublicKey()`-based gate is rejected by review as non-conforming.

### Integration Tests

- Contract-free transfer: a wallet-to-wallet transfer constructed from color and recipient `UserAddress` alone, with neither party interacting with the issuer; confirm the issuer is not called.
- Multi-recipient transfer: one transaction pays several `UserAddress` outputs from one holder's UTXOs.
- Burn: a holder calls `_burn`; confirm the coins move to an unspendable output, the indexed live-UTXO sum drops, an `UnshieldedBurn` event is emitted, and `totalBurned` increments under the extension. Confirm the burned coins cannot subsequently be spent.
- Supply bound: strand coins by sending them to an address whose key is discarded, then confirm the indexed live-UTXO sum falls below the contract's `totalSupply`, demonstrating the upper-bound property.
- Metadata resolution: a wallet recomputes color from a registry entry's `(domain, issuer)` and rejects an entry whose derivation does not match the claimed color.

## References

- MIP-0001: Midnight Improvement Proposal Process.
- MIP-0004: Fungible Token Standard with UTXO Conversion Extensions (hash-based mint authorization; freely transferable native UTXOs; the conversion model referenced for shielded interoperability).
- MIP-0011: Native Shielded Token Standard (the shielded sibling; conformance-profile and bounded-supply model reused here).
- MPS-0003: CAIP-2 Compliant Network Identifiers (for issuers that record a source-chain identifier in metadata).
- Discussion #142: Token Standards (quadrant taxonomy; the color-and-`UserAddress` transfer requirement).
- Discussion #136: MIP-0004 review (all ledger fields are public; witness vs circuit-input secrets are equivalent; native UTXOs are not contract-controlled).
- MIP-0002: Contract events, including the standard `UnshieldedBurn` event emitted by the canonical burn path.
- PR #104: Off-Chain Token Metadata Registry.
- Midnight docs, Tokens: NIGHT as an unshielded ledger token; the dual-ledger shielded/unshielded model.

## Acknowledgements

This proposal builds on the token-standards taxonomy in Discussion #142, the native-token structure and bounded-supply model established by MIP-0011, and the authorization and ledger-visibility findings in Discussion #136.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.