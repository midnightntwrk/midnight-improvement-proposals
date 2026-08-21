---
MPS: <Number> # assigned by editors
Title: Native NIGHT Staking on the Midnight Network
Authors: Karmel E <karmoola>
Status: Proposed
Category: Core
Created: 20-Aug-2026
Requires: MPS-0019
Replaces: none
MIP: none
---

<!--
 Copyright 2026 Midnight Foundation

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

Midnight offers no native mechanism for ordinary holders to stake NIGHT on the Midnight Network and share in network rewards. Participation in block production and its rewards runs entirely through the Cardano SPO path, which requires operating or delegating to a Cardano stake pool and holding ADA rather than NIGHT.

This gap weakens the incentive to hold NIGHT, concentrates security participation in a narrow operator set, and defers a core tokenomics question about how staked NIGHT, block production rewards, and DUST generation interact.

## Vision

NIGHT holders have something to do with NIGHT besides hold it and wait. A holder can commit NIGHT on Midnight, see what it earns, and withdraw it through tooling that matches the conventions they already use, without operating a Cardano stake pool or acquiring ADA.

This is also the first piece of network sovereignty. Today every path into Midnight block production and its rewards begins on Cardano. Native NIGHT staking gives Midnight one participation and security parameter that it defines and adjusts itself.

DUST is untouched. Committing NIGHT never costs a holder the ability to transact.

## Problem

### The gap

There is no way to stake NIGHT on Midnight. NIGHT is held, and holding generates DUST. It is never bonded, locked, delegated, or committed to any network role, and no mechanism pays a holder for holding it.

The only route into block production and its rewards is the Cardano SPO path. An SPO registers through a Cardano contract, is selected in proportion to delegated ADA, produces blocks, and is paid in NIGHT. MPS-0019 covers the registration, forecasting, monitoring, and withdrawal tooling for that flow.

That path is a payout mechanism for operators. It is not a way for a holder to participate with the asset they hold.

### Affected actors

**NIGHT holders.** Retail and treasury holders who want a use for NIGHT beyond price exposure. They have no participation option today.

**Delegators.** Today the only delegators in the system are ADA delegators on Cardano, who earn NIGHT through an SPO’s margin. There is no NIGHT delegator role, so a holder cannot back an operator with NIGHT or share in what that operator earns.

**Block producers.** SPOs who may want to attract Midnight-side backing, signal alignment, or differentiate from operators who registered opportunistically. They have nothing to accept backing into.

**Contract-held NIGHT.** NIGHT held by a contract has no defined answer for who designates the DUST recipient, who generation is attributed to, or who any staking reward accrues to. This is unresolved before any staking contract can exist.

### Considerations

**DUST generation for staked balances.** DUST accrues to a designated address up to a cap proportional to the NIGHT balance. Severing the designation causes linear decay to zero, and transferring NIGHT zeroes the DUST at the origin.

**cNIGHT versus mNIGHT.** NIGHT exists as Cardano-native cNIGHT, which is what exchanges list and DEX liquidity trades, and Midnight-native mNIGHT, which is canonical on Midnight and the only form that generates DUST.

**Withdrawal expectations.** Claiming and withdrawing should follow the conventions already established for block production rewards, close to what operators and holders use on Cardano, through a documented CLI and SDK path.

**Consensus scope.** Block production is Aura with GRANDPA finality, with committee selection intended to weight registered SPO candidates by delegated ADA. The solution must state whether native NIGHT staking affects producer eligibility or selection weight, or is a rewards and participation mechanism only.

## Use Cases

**A holder wants a reason to keep holding.** They hold NIGHT, see no yield, no role, and no vote, and the only action available is to hold or sell. Nothing distinguishes a holder of three years from one of three days.

**A staking contract is written.** A team building a staking product cannot determine who designates the DUST recipient for contract-held NIGHT or how generation and rewards are attributed, so the contract cannot be specified.

## Goals

1. **Give holders a native participation option.** A NIGHT holder should be able to commit NIGHT on Midnight and share in network rewards without operating or delegating to a Cardano stake pool.
2. **Broaden security participation.** Participation should extend beyond the registered SPO set without displacing it.
3. **Make holding NIGHT rational.** Holding should carry a defined return, role, or both, so the incentive to hold does not rest on price expectation alone.

### Success criteria

A future MIP satisfies this MPS if all of the following are testable and true.

1. A holder can stake NIGHT and receive rewards without holding ADA, operating a Cardano stake pool, or delegating to one.
2. The stakeable form of NIGHT is mNIGHT.
3. Rewards are claimed through a documented CLI and SDK path consistent with the block production reward conventions.
4. Total emission does not exceed the published Reserve curve, and the funding source for staking rewards is named.

## Copyright

This MPS is licensed under CC-BY-4.0.
