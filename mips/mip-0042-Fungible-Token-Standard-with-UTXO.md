---
MIP: '?'
Title: Fungible Token Standard with UTXO
Authors:
  - Guido De Vita (dvgui)
Status: Proposed
Category: Standards
Created: 2026-03-21
Requires: [MIP-1]
Replaces: none
License: Apache-2.0
---

<!--
 This file is part of midnight-improvement-proposals.
 Copyright (C) 2025-2026 Midnight Foundation
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

This MIP defines a standard interface that extends the [OpenZeppelin `FungibleToken` contract for Midnight](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) — the canonical account-based (Map) token implementation for the Midnight ecosystem — with bidirectional conversion circuits for both shielded and unshielded [UTXOs](https://docs.midnight.network/concepts/utxo). Building directly on the [OpenZeppelin Compact Contracts library](https://github.com/OpenZeppelin/compact-contracts), the proposal introduces four conversion circuits — `shield`, `toUtxo`, `unshield`, and `fromUtxo` — that allow contract token balances to be converted into Midnight-native ledger tokens (UTXOs) and back. This enables contracts built on top of the OpenZeppelin `FungibleToken` to participate in [Zswap](https://docs.midnight.network/concepts/zswap) atomic swaps, payment channels, and other UTXO-native operations, while preserving the familiar account-based programming model for DeFi logic. The conversion is supply-neutral: moving tokens between the Map and UTXO representations does not change the contract's `totalSupply`.

## Motivation

The current [FungibleToken standard](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) on Midnight (as implemented in the [OpenZeppelin Compact Contracts library](https://github.com/OpenZeppelin/compact-contracts)) provides an account-based (Map) token that mirrors the [ERC-20](https://eips.ethereum.org/EIPS/eip-20) interface. While this model is well suited for DeFi protocols, lending, and governance, it cannot directly participate in Midnight's [UTXO](https://docs.midnight.network/concepts/utxo)-native operations such as:

- **[Zswap](https://docs.midnight.network/concepts/zswap) atomic swaps**: Confidential, non-interactive multi-asset exchanges require tokens in UTXO form.
- **Payment channels and streaming payments**: UTXO-based tokens can be transferred peer-to-peer without contract interaction, enabling low-latency payments.
- **Cross-contract and cross-chain bridges**: Bridge protocols often require discrete, individually-owned token units rather than entries in a shared balance map.
- **Privacy-enhanced transfers**: [Shielded UTXOs](https://docs.midnight.network/concepts/utxo) hide the sender, receiver, and amount, providing stronger confidentiality than account-based transfers where balances are visible in the contract's public ledger state.

Multiple projects building on Midnight — including decentralized exchanges, lending protocols, and payment applications — have independently identified the need for tokens that can move fluidly between the [account model and the UTXO model](https://docs.midnight.network/concepts/ledgers). Without a standard interface for this conversion, each project will implement its own incompatible approach, fragmenting the ecosystem and increasing security risk.

This MIP addresses the gap by defining a minimal, composable set of conversion circuits that any fungible token contract can implement alongside the existing [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) interface.

## Specification

### Terminology

- **Map balance**: A token balance stored in the contract's `_balances` ledger map, associated with an account identifier (`Either<ZswapCoinPublicKey, ContractAddress>`). See the [FungibleToken contract](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) for the full ledger state definition.
- **Shielded UTXO**: A [Zswap](https://docs.midnight.network/concepts/zswap) coin whose value and ownership are hidden using zero-knowledge commitments. Managed by the Midnight protocol layer, not by smart contract state.
- **Unshielded UTXO**: A protocol-level [UTXO](https://docs.midnight.network/concepts/utxo) whose value and ownership are publicly visible, signed with Schnorr signatures.
- **Token color**: A deterministic identifier for a class of contract-minted UTXOs, computed as `SHA-256(domainSeparator, contractAddress)` via the `tokenType(domainSep, contract)` function in the [Compact Standard Library](https://docs.midnight.network/compact).
- **Domain separator**: A 32-byte value chosen at deployment that, together with the contract address, uniquely identifies the token type for UTXO operations.

### Required State Extensions

Contracts implementing this standard MUST add the following ledger variables:

```typescript
export ledger domain: Bytes<32>;
export ledger utxoSupply: Uint<128>;
ledger mintCounter: Counter;
ledger mintNonce: Bytes<32>;
```

- `domain`: A domain separator set at construction, used to compute the [token color](https://docs.midnight.network/concepts/utxo) for UTXO operations. MUST be public (`export ledger`) so that any participant can read it and so that the conversion circuits can derive the token color at call time via `tokenType(domain, kernel.self())`.
- `utxoSupply`: A running total of tokens currently held as UTXOs (both shielded and unshielded). Incremented by `shield` and `toUtxo`, decremented by `unshield` and `fromUtxo`. MUST be public (`export ledger`) for off-chain supply accounting.
- `mintCounter`: A monotonically increasing counter used together with `mintNonce` to guarantee uniqueness of shielded coin commitments. SHOULD be private (`ledger`, not `export ledger`) to reduce information leakage.
- `mintNonce`: A nonce that evolves with each `shield` operation to prevent commitment collisions. SHOULD be private (`ledger`, not `export ledger`) to reduce information leakage.

**Why not store `color` in the ledger?** An earlier version stored the token color as `export ledger color: Bytes<32>`, precomputed in the constructor via `tokenType(domainSep, kernel.self())`. This was found to produce a different hash than what `mintShieldedToken` and `mintUnshieldedToken` stamp on minted coins. The Compact runtime resolves `kernel.self()` differently during constructor execution vs circuit execution, causing a mismatch that breaks `unshield` color validation and `fromUtxo` UTXO absorption. The fix is to compute `tokenType(domain, kernel.self())` at call time in each circuit that needs it.

These MUST be initialized in the constructor:

```typescript
constructor(..., domainSep: Bytes<32>, initNonce: Bytes<32>) {
    // ... existing FungibleToken initialization ...
    domain = disclose(domainSep);
    mintNonce = disclose(initNonce);
}
```

### Required Circuits

#### `shield(amount: Uint<64>) → ShieldedCoinInfo`

Converts `amount` tokens from the caller's Map balance into a [shielded UTXO](https://docs.midnight.network/concepts/zswap) (Zswap coin) owned by the caller.

**Behavior:**

1. MUST revert if `amount` is zero.
2. Deduct `amount` from the caller's Map balance. MUST revert if the caller's balance is insufficient.
3. Increment `utxoSupply` by `amount`.
4. MUST NOT modify `totalSupply`.
5. Increment the `mintCounter` and evolve `mintNonce` for commitment uniqueness.
6. Call `mintShieldedToken(domain, amount, mintNonce, caller)` from the [Compact Standard Library](https://docs.midnight.network/compact) to produce a protocol-level shielded coin.
7. Return the `ShieldedCoinInfo` to the caller.

**Note on amount type:** The `amount` parameter is `Uint<64>` rather than `Uint<128>` because Midnight's native `mintShieldedToken` operation accepts only `Uint<64>` values in practice, despite the [Compact Standard Library documentation](https://docs.midnight.network/compact) specifying `Uint<128>`. This limits the maximum amount convertible in a single `shield` operation to `2^64 - 1` base units. See [Amount Type Constraints](#amount-type-constraints-in-outbound-operations) for details.

#### `toUtxo(amount: Uint<64>, recipient: UserAddress) → Bytes<32>`

Converts `amount` tokens from the caller's Map balance into an [unshielded UTXO](https://docs.midnight.network/concepts/utxo) owned by `recipient`.

**Behavior:**

1. MUST revert if `amount` is zero.
2. Deduct `amount` from the caller's Map balance. MUST revert if the caller's balance is insufficient.
3. Increment `utxoSupply` by `amount`.
4. MUST NOT modify `totalSupply`.
5. Call `mintUnshieldedToken(domain, amount, right(recipient))` from the [Compact Standard Library](https://docs.midnight.network/compact) to produce a protocol-level unshielded UTXO. The `domain` value is read directly from the public ledger state.
6. Return the token color (`Bytes<32>`).

**Design note:** The circuit reads `domain` from the public ledger state rather than accepting it as a parameter, eliminating the risk of domain separator misuse and simplifying the caller interface.

#### `unshield(coin: ShieldedCoinInfo) → []`

Converts a [shielded UTXO](https://docs.midnight.network/concepts/zswap) back into Map balance for the caller.

**Behavior:**

1. MUST revert if `coin.value` is zero.
2. Compute the expected token color as `tokenType(domain, kernel.self())`.
3. MUST revert if `coin.color` does not equal the computed color. This prevents crediting this token's Map balance with a shielded coin from a different token contract.
4. Call `receiveShielded(coin)` from the [Compact Standard Library](https://docs.midnight.network/compact) to absorb (nullify) the shielded coin from the transaction.
5. Credit `coin.value` to the caller's Map balance. MUST revert on arithmetic overflow.
6. Decrement `utxoSupply` by `coin.value`.
7. MUST NOT modify `totalSupply`.

**Design note:** The amount to credit is derived directly from `coin.value` (which is `Uint<128>`) rather than accepted as a separate parameter. Since `ShieldedCoinInfo.value` is the committed value of the shielded coin, this eliminates any possibility of an amount mismatch between the UTXO value and the Map balance credit.

**Security note:** The color validation in step 2 is critical. Without it, a user could call `unshield` with a shielded coin from a different token contract and receive a Map balance credit for this token. The `receiveShielded` call itself does not validate the coin's color — it only verifies the coin exists in the ZSwap Merkle tree.

#### `fromUtxo(amount: Uint<128>) → []`

Converts [unshielded UTXOs](https://docs.midnight.network/concepts/utxo) back into Map balance for the caller.

**Behavior:**

1. MUST revert if `amount` is zero.
2. Compute the token color as `tokenType(domain, kernel.self())`.
3. Call `receiveUnshielded(color, amount)` to absorb (nullify) the unshielded UTXOs from the transaction.
4. Credit `amount` to the caller's Map balance. MUST revert on arithmetic overflow.
5. Decrement `utxoSupply` by `amount`.
6. MUST NOT modify `totalSupply`.

**Note on amount type:** The `amount` parameter is `Uint<128>` because `receiveUnshielded` natively accepts `Uint<128>` values, allowing the full range for inbound conversions.

### Optional Circuits

#### `burn(value: Uint<128>) → []`

Implementations MAY include a `burn` circuit that allows any user to burn their own Map balance.

**Behavior:**

1. MUST revert if `value` is zero.
2. Deduct `value` from the caller's Map balance. MUST revert if the caller's balance is insufficient.
3. Decrement `totalSupply` by `value`.

**Design note:** This circuit delegates to the OpenZeppelin `FT__burn` internal function, which handles both the balance deduction and the supply adjustment. It is useful for bridge protocols, redemption mechanisms, and deflationary token models. Unlike conversion circuits, `burn` permanently removes tokens from the supply.

#### `mint(account, value: Uint<128>) → []`

Implementations MAY include a `mint` circuit for post-deployment token issuance. Alternatively, the entire supply may be minted in the constructor.

If a `mint` circuit is provided, it SHOULD be gated by an access control mechanism. The reference implementation uses a lightweight `admin`/`minter` key pattern: the deployer is both `admin` and `minter` by default, and the admin can delegate minting via `setMinter(newMinter: ZswapCoinPublicKey)`. Implementations are free to choose any suitable authorization scheme.

### Optional: Minting Delegation

#### `setMinter(newMinter: ZswapCoinPublicKey) → []`

Admin-only circuit that delegates minting authority to a different key. The deployer starts as both `admin` and `minter`. Calling `setMinter` changes who can call `mint` without transferring admin control.

### Supply Invariant

The fundamental invariant of this standard is:

```math
\texttt{totalSupply} = \sum \text{Map balances} + \sum \text{shielded UTXOs} + \sum \text{unshielded UTXOs}
```

The contract tracks the UTXO portion explicitly via `utxoSupply`:

```math
\texttt{utxoSupply} = \sum \text{shielded UTXOs} + \sum \text{unshielded UTXOs}
```

Therefore, the Map-held supply can be derived off-chain:

```math
\texttt{mapSupply} = \texttt{totalSupply} - \texttt{utxoSupply}
```

Since `totalSupply` in the contract's [ledger](https://docs.midnight.network/concepts/ledgers) only tracks the total tokens ever minted via `_mint` minus any burned via `_burn`, and conversion operations adjust Map balances and `utxoSupply` without touching `totalSupply`, the contract's on-ledger `totalSupply` reflects the total tokens in existence across all representations.

Implementers MUST ensure that:

- Conversion circuits never increment or decrement `totalSupply`. Only `mint` and `burn` operations may alter supply.
- All four conversion circuits correctly maintain `utxoSupply`: increment on `shield`/`toUtxo`, decrement on `unshield`/`fromUtxo`.

### Internal Helpers

Compliant implementations SHOULD provide the following internal circuits:

#### `_deductBalance`

```typescript
circuit _deductBalance(
    account: Either<ZswapCoinPublicKey, ContractAddress>,
    amount: Uint<128>
): []
```

Deducts `amount` from `account`'s Map balance without modifying `totalSupply`. Reverts on insufficient balance.

#### `_creditBalance`

```typescript
circuit _creditBalance(
    account: Either<ZswapCoinPublicKey, ContractAddress>,
    amount: Uint<128>
): []
```

Credits `amount` to `account`'s Map balance without modifying `totalSupply`. Reverts on arithmetic overflow (`MAX_UINT128 - balance < amount`).

These helpers separate the accounting concern (balance adjustment) from the supply concern (the `_update` circuit in the base [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact)), making it explicit that conversions are supply-neutral.

### Compatibility with FungibleToken

This standard is designed as an extension. Contracts MUST implement the core [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) interface (`name`, `symbol`, `decimals`, `totalSupply`, `balanceOf`, `transfer`). The `approve`, `allowance`, and `transferFrom` circuits are deferred — contract-to-contract (C2C) calls are currently disabled on Midnight, making delegated spending unusable. These MAY be added when C2C support ships. The UTXO conversion circuits are additive and MUST NOT alter the behavior of existing FungibleToken circuits.

## Rationale

### Why extend the existing standard rather than create a new token type?

Many DeFi protocols on Midnight are already being built against the [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) interface. By defining UTXO conversion as an extension, existing contracts can adopt the new functionality without breaking compatibility. This also avoids fragmenting the ecosystem between "Map-only" and "UTXO-only" token standards. Midnight's [dual-ledger architecture](https://docs.midnight.network/concepts/ledgers) is explicitly designed to support both models, and this MIP reflects that design intent.

### Why four circuits instead of two?

Midnight distinguishes between [shielded (Zswap)](https://docs.midnight.network/concepts/zswap) and [unshielded (Schnorr-signed)](https://docs.midnight.network/concepts/utxo) UTXOs at the protocol level. They have different privacy properties, different verification mechanisms, and different [Compact Standard Library](https://docs.midnight.network/compact) functions. A unified `toUtxo`/`fromUtxo` interface would hide this distinction and could lead to privacy accidents (e.g., a user intending to create a shielded UTXO accidentally creating an unshielded one). Explicit `shield`/`unshield` and `toUtxo`/`fromUtxo` pairs make the privacy choice unambiguous at the call site.

### Amount type constraints in outbound operations

Midnight's protocol-level UTXO minting functions (`mintShieldedToken`, `mintUnshieldedToken`) accept only `Uint<64>` values in practice, while the Map-based [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) uses `Uint<128>`. The [Compact Standard Library documentation](https://docs.midnight.network/compact) specifies `Uint<128>` for `mintShieldedToken`, but the current compiler silently truncates values to 64 bits. This standard uses `Uint<64>` for outbound operations (`shield`, `toUtxo`) to make the constraint explicit and prevent silent data loss.

Inbound operations (`unshield` and `fromUtxo`) use `Uint<128>` where the protocol supports it, maximizing the range for conversions back to Map balances.

Tokens with balances exceeding `2^64 - 1` base units in a single position will need multiple outbound conversion operations. We recommend that the Midnight protocol team consider upgrading `mintShieldedToken` and `mintUnshieldedToken` to accept `Uint<128>` in a future release, which would allow this standard to be updated accordingly.

### Why read domain from ledger state instead of accepting it as a parameter?

The domain separator determines the token color for unshielded UTXOs. An alternative design would declare `domain` as `sealed` (private) and pass it as an argument to `toUtxo`/`fromUtxo`. However, this creates several problems: users could pass the wrong value (creating non-recoverable UTXOs with a mismatched token color), deployers could selectively share different values with different users, and the value must be publicly known anyway for the contract to be usable. By making `domain` a public ledger variable and reading it directly in `toUtxo`/`fromUtxo`, these problems are eliminated. The correct domain separator is always used, and any participant can read it from the contract's public state.

### Why derive the unshield amount from the coin rather than accepting it as a parameter?

An alternative `unshield(coin, amount)` design would require the caller to specify the amount separately from the shielded coin. Since the coin's value is hidden inside a Pedersen commitment, the contract cannot verify that the claimed `amount` matches the committed value — it would rely entirely on the protocol-level balance equation to catch mismatches. By using `unshield(coin)` and reading `coin.value` directly, the amount is always exactly the committed value, making a mismatch impossible by construction.

### Why supply-neutral conversions?

Tokens moving between Map and UTXO form are not being created or destroyed — they are changing representation. If `totalSupply` were decremented on `shield` and incremented on `unshield`, it would misrepresent the total tokens in existence and could confuse indexers, block explorers, and DeFi protocols that rely on `totalSupply` as a measure of circulating supply. This is consistent with how the [ERC-20](https://eips.ethereum.org/EIPS/eip-20) standard treats wrapping: wrapping [ERC-20](https://eips.ethereum.org/EIPS/eip-20) tokens into a wrapped variant does not change the underlying token supply.

### Why track utxoSupply explicitly?

Without an on-chain `utxoSupply` counter, there is no way for the contract (or off-chain observers) to know how many tokens are currently held as UTXOs versus Map balances. The `totalSupply` alone cannot distinguish between the two. By tracking `utxoSupply` explicitly, indexers and block explorers can compute `mapSupply = totalSupply - utxoSupply` and report accurate circulating supply breakdowns. The cost is minimal: four additional increment/decrement operations across the conversion circuits.

## Path to Active

### Acceptance Criteria

- At least one reference implementation deployed on Midnight testnet.
- Integration demonstrated with at least one [Zswap](https://docs.midnight.network/concepts/zswap)-based DEX or atomic swap protocol.
- Review and endorsement by the Midnight developer community through workshop sessions (as described in [MIP-1](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-Midnight-Improvement-Proposal-Process.md)).
- Security audit of a reference implementation.

### Reference Deployment (Preprod)

A reference implementation has been deployed on Midnight preprod and exercised through all conversion circuits. The deployment demonstrates the full token lifecycle: minting, Map → UTXO conversion (both shielded and unshielded), and UTXO → Map round-trip.

**Contract:** [`ef0ba484ee62fd8e31a7f84732d3b2934b3dcf441c6619b9ba503d8d113a7161`](https://preprod.nightscan.io/contract/?addr=ef0ba484ee62fd8e31a7f84732d3b2934b3dcf441c6619b9ba503d8d113a7161)
**Network:** preprod | **Token:** MIP42 | **Decimals:** 6
**Token color:** `532a67c05b9e3c890f6fe93df3c03644efa708a31a2ae2458d975130651345cf`

| #   | Circuit                | Amount (MIP42) | Transaction                                                                                                                         |
| --- | ---------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `deploy` (constructor) | —              | [`ef0ba484...3a7161`](https://preprod.nightscan.io/contract/?addr=ef0ba484ee62fd8e31a7f84732d3b2934b3dcf441c6619b9ba503d8d113a7161) |
| 2   | `mint`                 | 1,000.000000   | [`0060e08d...9f871d`](https://preprod.nightscan.io/tx/?hash=0060e08de2b5adaaee55c3d888a4e8a64b8678b312f1470f692c01a254bb9f871d)     |
| 3   | `toUtxo`               | 100.000000     | [`0032c6a8...62d0a5`](https://preprod.nightscan.io/tx/?hash=0032c6a87aaedc6e59800d1c3c95486d3f238f246661af46ccfd03875e1662d0a5)     |
| 4   | `toUtxo`               | 50.000000      | [`00ecb30d...197161`](https://preprod.nightscan.io/tx/?hash=00ecb30d9334b4fb687439de304e5b324b28c9739c5523a5ed2ac48ce915197161)     |
| 5   | `shield`               | 100.000000     | [`0036b317...b5952a`](https://preprod.nightscan.io/tx/?hash=0036b31799aba1830775b0711fe0179dfb62e4979255285f9d83f7eb0d28b5952a)     |
| 6   | `shield`               | 50.000000      | [`00207542...bce396`](https://preprod.nightscan.io/tx/?hash=00207542360e240826d9b7efc3cb27ba08eeae5da91e4a7749b3a9d05014bce396)     |
| 7   | `fromUtxo`             | 100.000000     | [`00b1bc05...27e4a9`](https://preprod.nightscan.io/tx/?hash=00b1bc05cfc046c7dfb4517518f099f2830616094f48bf6ba7f029ee0d5a27e4a9)     |
| 8   | `unshield`             | 100.000000     | [`00f10b70...16bed3`](https://preprod.nightscan.io/tx/?hash=00f10b70be48f23541265baf2aada707a81952d5df2965580c71e73ac81116bed3)     |

**Final state:** 800.000000 MIP42 in Map, 50.000000 in unshielded UTXOs, 50.000000 in shielded UTXOs. `totalSupply` = 1,000.000000, `utxoSupply` = 100.000000.

All four conversion circuits (`shield`, `toUtxo`, `unshield`, `fromUtxo`) demonstrated bidirectional round-trips on a live network.

### ZSwap Atomic Swap Demonstration (Preprod)

Two tokens deployed on preprod were used to demonstrate ZSwap atomic swaps in both unshielded and shielded modes between two independent wallets (Alice and Bob).

**Contracts:**

- MIP42: [`ef0ba484...3a7161`](https://preprod.nightscan.io/contract/?addr=ef0ba484ee62fd8e31a7f84732d3b2934b3dcf441c6619b9ba503d8d113a7161)
- SWAP: [`e92d621f...bbf94`](https://preprod.nightscan.io/contract/?addr=e92d621fbdde115ef91c5289a83ab66accabb5f411942e1edb5f2b30748bbf94)

**Setup:** 500 MIP42 minted to Alice, 500 SWAP minted to Bob. Each converted 100 to unshielded UTXOs and 100 to shielded UTXOs.

| #   | Step                     | Transaction                                                                                                                     |
| --- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Mint 500 MIP42 → Alice   | [`00af800b...52e098`](https://preprod.nightscan.io/tx/?hash=00af800b5ca1ca4b6a42c2db500cc3894c0eb4d8d6fda6874fcb38d8ad4852e098) |
| 2   | Mint 500 SWAP → Bob      | [`00e2d84d...10e43b`](https://preprod.nightscan.io/tx/?hash=00e2d84d764861b14ce234be980225d6b8b8232f5a15d57c69dd796943f510e43b) |
| 3   | Alice `toUtxo` 100 MIP42 | [`0025c0f1...3bc901`](https://preprod.nightscan.io/tx/?hash=0025c0f16feda5a4c6ad52bac76bad2a31b55f096e232298b9c7d12571403bc901) |
| 4   | Alice `shield` 100 MIP42 | [`0024419b...8a70dd`](https://preprod.nightscan.io/tx/?hash=0024419b86464f0c7b2ec097b86ec105f52c13468cc2b6ea6f88567004b28a70dd) |
| 5   | Bob `toUtxo` 100 SWAP    | [`00ad0b30...20910`](https://preprod.nightscan.io/tx/?hash=00ad0b303418f177433590292019ee311fd2892953645ff141b5a43338a1020910)  |
| 6   | Bob `shield` 100 SWAP    | [`0033325d...8e8428`](https://preprod.nightscan.io/tx/?hash=0033325d5a353cb2dec6cd306e1b62561743dc741b4ac6146987520f558d8e8428) |

**Unshielded ZSwap** — Alice offers 50 MIP42 for 50 SWAP (unshielded UTXOs):

| #   | Step                     | Transaction                                                                                                                     |
| --- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| 7   | Atomic swap (unshielded) | [`00a8ec41...fae939`](https://preprod.nightscan.io/tx/?hash=00a8ec4169f2a395fd3d38559da1dfceea1b2077bb2caa67cd9469486aadfae939) |

| Wallet         | MIP42 unshielded | SWAP unshielded |
| -------------- | ---------------- | --------------- |
| Alice (before) | 100.000000       | 0               |
| Alice (after)  | 50.000000        | 50.000000       |
| Bob (before)   | 0                | 100.000000      |
| Bob (after)    | 50.000000        | 50.000000       |

**Shielded ZSwap** — Alice offers 50 MIP42 for 50 SWAP (shielded UTXOs):

| #   | Step                   | Transaction                                                                                                                     |
| --- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 8   | Atomic swap (shielded) | [`00fd036f...67ce4d`](https://preprod.nightscan.io/tx/?hash=00fd036f16394ddb21d57312229041ff29d27b85d1c79e56cbea1cfca1ca67ce4d) |

| Wallet         | MIP42 shielded | SWAP shielded |
| -------------- | -------------- | ------------- |
| Alice (before) | 100.000000     | 0             |
| Alice (after)  | 50.000000      | 50.000000     |
| Bob (before)   | 0              | 100.000000    |
| Bob (after)    | 50.000000      | 50.000000     |

Both unshielded and shielded ZSwap atomic swaps completed successfully with correct balance accounting on both sides. The shielded swap preserves privacy: the transaction on-chain does not reveal which tokens were exchanged, the amounts, or the parties involved.

Full reference implementation and CLI tooling available at [midnight-swap](https://github.com/sommetlabs/midnight-swap).

### Implementation Plan

1. Publish a reference implementation as an extension of the [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) `FungibleToken` library.
2. Develop integration tests covering all conversion paths, edge cases, the supply invariant, and `utxoSupply` tracking.
3. Collaborate with DEX and DeFi teams building on Midnight to validate the interface.
4. Submit for formal review through the [MIP process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-Midnight-Improvement-Proposal-Process.md).

## Backwards Compatibility Assessment

This MIP is fully backwards compatible with the existing [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) standard. No changes to existing FungibleToken circuits are required. The new state variables (`domain`, `utxoSupply`, `mintCounter`, `mintNonce`) and circuits (`shield`, `toUtxo`, `unshield`, `fromUtxo`) are purely additive.

Existing contracts that do not implement UTXO conversion will continue to function without modification. Contracts wishing to adopt this standard will need to be deployed with the extended constructor parameters or upgraded if the contract system supports upgrades.

Token balances in the Map are not affected by UTXO operations on other users' balances. The conversion circuits only modify the caller's own balance.

## Security Considerations

### Nonce Uniqueness and Commitment Collisions

The `shield` circuit relies on a monotonically increasing counter and an evolving nonce to produce unique commitments for each [shielded coin](https://docs.midnight.network/concepts/zswap). If two `shield` operations were to produce identical commitments, it could lead to a double-spend vulnerability. Implementations MUST ensure that the counter-nonce evolution scheme produces unique outputs for every invocation. The reference implementation uses `evolveNonce(mintCounter, mintNonce)` from the [Compact Standard Library](https://docs.midnight.network/compact), which is designed for this purpose. Making `mintCounter` and `mintNonce` private ledger state prevents external observation of the nonce evolution sequence, reducing potential information leakage.

### Amount Integrity in unshield

The `unshield` circuit derives the credit amount directly from `coin.value` within the `ShieldedCoinInfo` struct, rather than accepting it as a separate parameter. This design ensures that the amount credited to the caller's Map balance is always exactly the value committed in the shielded UTXO. The protocol-level `receiveShielded` function validates the coin against the [Zswap](https://docs.midnight.network/concepts/zswap) commitment scheme, and the balance equation enforcement ensures that no value is created or destroyed. The combination of these two mechanisms — application-level derivation from `coin.value` and protocol-level commitment validation — provides defense in depth against amount manipulation.

### UTXO Proliferation and State Bloat

Each `shield` or `toUtxo` call creates a new [UTXO](https://docs.midnight.network/concepts/utxo) on the Midnight ledger. If users perform many small conversions, this can lead to UTXO set growth. While Midnight's nullifier-based design handles spent UTXOs efficiently, a large active UTXO set can increase wallet synchronization time and state proof sizes. Applications should be designed to minimize unnecessary fragmentation.

### Domain Separator Integrity

The `domain` value is set once at construction and stored as public ledger state. The `toUtxo` and `fromUtxo` circuits read `domain` directly from the ledger, ensuring that all unshielded UTXO operations use the correct and consistent token color. Since `domain` is immutable after construction (there is no circuit to modify it), the token color for a given contract is fixed for its lifetime. Users and wallets do not need to know or manage the domain separator — it is always read from the contract's public state.

### Role-Based Access Control

For implementations that include a `mint` circuit, access control is critical to prevent unauthorized supply inflation. A single admin key controlling minting authority is a centralization risk: if the key is compromised, an attacker gains unlimited minting power; if the key is lost, no new tokens can ever be minted.

The reference implementation uses a lightweight `admin`/`minter` key pattern with `setMinter` delegation. This keeps the circuit count low (critical for Midnight's current block limits), but provides only single-key authorization. Production deployments with higher block limits SHOULD adopt the [OpenZeppelin AccessControl](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/access/AccessControl.compact) module or an equivalent mechanism for role separation, multi-party governance, revocability, and auditability.

### Zero-Amount Protection

All conversion circuits (`shield`, `toUtxo`, `unshield`, `fromUtxo`) MUST revert on zero-amount inputs. This prevents no-op transactions that consume DUST without performing meaningful work, and eliminates edge cases in supply accounting.

## Implementation

### Components to Modify or Create

1. **New Compact module**: `FungibleTokenUTXO.compact` — extends [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) with the four conversion circuits and associated state.
2. **Constructor extension**: Additional parameters for `domainSep` and `initNonce`. Initialization of `utxoSupply`.
3. **Internal helpers**: `_deductBalance` and `_creditBalance` as supply-neutral balance adjustment circuits.

### Dependencies

- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) `FungibleToken` module (v0.0.1-alpha.1 or later).
- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) `Utils` module.
- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) `AccessControl` module (v0.0.1-alpha.1 or later) — optional, for production deployments that need role-based access control (adds significant circuit count).
- [Compact Standard Library](https://docs.midnight.network/compact): `mintShieldedToken`, `mintUnshieldedToken`, `receiveShielded`, `receiveUnshielded`, `tokenType`, `evolveNonce`, `Counter`.
- [Compact](https://docs.midnight.network/compact) language version >= 0.22.0.

### Reference Implementation

A reference implementation is provided as a [Compact](https://docs.midnight.network/compact) contract that imports and wraps the [FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact) module. The reference implementation includes optional `burn`, `mint` (gated by `admin`/`minter` key pattern with `setMinter` delegation), and 13 circuits total (15 verifier keys — at the local devnet block limit). The full source is available at: [token-with-utxo](https://github.com/sommetlabs/abyss-contracts/blob/main/token-with-utxo).

## Testing

### Unit Tests

- `shield`: Verify Map balance decreases by `amount`, `totalSupply` unchanged, `utxoSupply` incremented, shielded coin returned. Verify revert on zero amount.
- `toUtxo`: Verify Map balance decreases, `totalSupply` unchanged, `utxoSupply` incremented, correct token color returned.
- `unshield`: Verify Map balance increases by `coin.value`, `totalSupply` unchanged, `utxoSupply` decremented.
- `fromUtxo`: Verify Map balance increases, `totalSupply` unchanged, `utxoSupply` decremented.
- Revert on insufficient balance for `shield` and `toUtxo`.
- Revert on zero amount for all conversion circuits.
- Revert on overflow for `_creditBalance`.

### Integration Tests

- Round-trip: `mint` → `shield` → `unshield` → verify balance restored, `utxoSupply` returns to zero.
- Round-trip: `mint` → `toUtxo` → `fromUtxo` → verify balance restored, `utxoSupply` returns to zero.
- Cross-user: User A calls `toUtxo` → UTXO transferred off-chain to User B → User B calls `fromUtxo`.
- [Zswap](https://docs.midnight.network/concepts/zswap) atomic swap between two token contracts both implementing this standard.

### Invariant Tests

- After any sequence of operations, verify: `totalSupply == sum(all Map balances) + utxoSupply`.
- After any sequence of operations, verify: `utxoSupply == sum(all outstanding UTXOs)`.
- Fuzz testing with randomized sequences of mint, burn, shield, unshield, toUtxo, fromUtxo, and transfer operations.

## References (Optional)

- [MIP-1: Midnight Improvement Proposal Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-Midnight-Improvement-Proposal-Process.md)
- [OpenZeppelin Compact Contracts — FungibleToken](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/token/FungibleToken.compact)
- [OpenZeppelin Compact Contracts — AccessControl](https://github.com/OpenZeppelin/compact-contracts/blob/main/contracts/src/access/AccessControl.compact)
- [OpenZeppelin Compact Contracts — Repository](https://github.com/OpenZeppelin/compact-contracts)
- [Midnight Zswap Documentation](https://docs.midnight.network/concepts/zswap)
- [Midnight UTXO Model Documentation](https://docs.midnight.network/concepts/utxo)
- [Midnight Ledgers Documentation](https://docs.midnight.network/concepts/ledgers)
- [The Compact Language](https://docs.midnight.network/compact)
- [ERC-20 Token Standard (Ethereum)](https://eips.ethereum.org/EIPS/eip-20)

## Acknowledgements

This proposal was developed by [Sommet Labs](https://abyss.trade), in collaboration with [OpenZeppelin](https://www.openzeppelin.com) and informed by the existing work in the [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) library. Additional input was provided by ecosystem teams building DeFi and payment applications on Midnight, as well as the [Midnight protocol documentation](https://docs.midnight.network).

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
