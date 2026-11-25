---
MIP: X
Title: 128-bit mint amounts
Authors:
  - David Millar-Durrant (DavidM-D)
Status: Draft
Category: Core
Created: 2026-11-18
Requires: N/A
Replaces: N/A
MPS: none
---

<!--
 Copyright Midnight Foundation

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

Widen mint amounts from `Uint<64>` to `Uint<128>`, the width coins already carry, for shielded and unshielded tokens.

## Motivation

ERC-20 amounts are `uint256`. When a contract mints a token for the outcome of an EVM swap, the amount has to fit the mint's `Uint<64>`, so we round it down. How much is lost depends on the token's decimals. LINK has 18, so a `Uint<64>::MAX` is about 18.4 LINK. So today our contracts can mint at most about 18.44 LINK at a time unless we round amounts down to a coarser unit. Rounding is painful and does not work well, because the amounts that come back from the EVM are not rounded, which leaves a remainder to deal with and is a little messy.

## Specification

- `mintShieldedToken` and `mintUnshieldedToken` take a `Uint<128>` amount.
- `kernel.mintShielded` and `kernel.mintUnshielded` take `Uint<128>`, and the ledger's mint effects carry 128-bit amounts.
- Nothing else changes. Coin values, sends, receives and balances are already `Uint<128>`.

## Rationale

The mint is the only 64-bit step in the token path, so widening it removes the limit without touching the coin format. 128 bits covers every large ERC-20: the largest supplies in base units are around 10^33, about 110 bits, and Uniswap reserves and liquidity are `uint112` and `uint128`.

### Alternatives considered

- Scale decimals at the bridge. This is what we do now, and it is the rounding this proposal removes.
- 248-bit coins and mints. 248 bits is Compact's widest `Uint` and fits one field element, but it means widening the coin value itself, probably a better long term solution, but perhaps we imagine a much more difficult task.
- 256-bit coins and mints, matching `uint256` exactly. Same format change, plus a two-limb integer type in Compact, since 256 bits exceeds the field. Values above 248 bits only occur as sentinels (infinite approvals) and internal share accounting, never as transferable amounts.

## Path to Active

### Acceptance Criteria

A contract mints a coin of value 2^128 - 1 on a public testnet, shielded and unshielded, then sends and burns it.

### Implementation Plan

Ledger change to the mint effect width, then the standard library signatures in the matching compactc release.

## Backwards Compatibility Assessment

Hard fork, since the mint effect encoding changes. The standard library signatures widen, so existing call sites passing `Uint<64>` still type-check on recompile. Deployed contracts keep working if 64-bit mint amounts stay valid as narrower values. Wallets and the indexer already handle 128-bit coin values.

## Security Considerations

A single coin of any `Uint<128>` value can already exist by merging, so coin handling sees no new range. What changes is total supply: with 64-bit mints it takes 2^64 mints for a token's supply to pass 2^128, with 128-bit mints it takes two. We think attacks of this type are limited, since you'd first need to find a way to mint an incredibly large number of tokens, in order to overflow and get a very small number of tokens.

## Implementation

Ledger (mint effect encoding and checks), the kernel operations, compactc and the standard library.

## Testing

Mint at 2^64 - 1, 2^64 and 2^128 - 1, shielded and unshielded, then send, merge and burn the result. Regression tests for existing 64-bit mints.

## References (Optional)

- `compiler/standard-library.compact` in LFDT-Minokawa/compact: `mintShieldedToken`, `mintUnshieldedToken`.

## Acknowledgements

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
