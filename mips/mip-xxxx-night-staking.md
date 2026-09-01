---
MIP: xxxx
Title: NIGHT Staking
Authors:
  - Karmel E (karmoola)
Status: Draft
Category: Core
Created: 01-Sep-2026
Requires: MPS on Native NIGHT Staking, MPS-0019
Replaces: none
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

**An mNIGHT holder delegates to a stake pool run by a block producer and earns a share of that pool's block production rewards. Stake is liquid, nothing is locked or at risk, and stake carries no consensus weight.**

V1 is intentionally simple. It lets token holders earn rewards without new emission and without changing existing network mechanics. The mNIGHT stays in the holder's wallet and stays spendable, so DUST is untouched. Rewards are paid from the Reserve, the same budget that already funds block production, and flow through the producing pool. Pools declare a commission that can only be lowered.

## Motivation

Midnight has no native way for holders to stake NIGHT and share in network rewards. Participation runs entirely through the Cardano SPO path, which needs ADA and stake pool infrastructure rather than NIGHT.

## Specification

### 1. Scope

A stake address and delegation certificate for mNIGHT holders. Stake pools operated by registered block producers. A per-epoch staking reward paid from the Reserve through the producing pool.

*Producer performance scoring and SPO bonding are both separable and both belong in their own MIPs. (Currently pending Block Rewards MIP to confirm phases and sequencing)*

### 2. Staking is non-custodial and transfer-free

Staking MUST NOT move, lock, or transfer mNIGHT. There is no bonding period, no unbonding period, and no lockup. A holder may spend, transfer, or receive mNIGHT at any time while staked.

Liquid staking normally raises the problem of verifying that a staker still holds the stake they are being rewarded for. The epoch mechanism resolves this without locks: stake is captured at epoch boundaries and read with a two-epoch lookback, so a change in a staker's position is reflected after a natural delay rather than requiring a hard lock. This is a common and accepted pattern. A hard lock would be driven by financial-stakeholder preference, not technical necessity, and is not part of V1.

**DUST invariance.** The following MUST hold and MUST be verified by test:

1. A staked address keeps its DUST designation through registration, delegation, redelegation, and deregistration.
2. A staked balance generates DUST at the same rate as an equal unstaked balance.
3. A staked balance has the same DUST cap as an equal unstaked balance.
4. No staking operation causes DUST decay.
5. Unwithdrawn rewards generate DUST like any other balance held by that holder.

### 3. Stake addresses

A holder registers a stake address against a refundable deposit, binds one or more payment addresses to it, and submits a delegation certificate naming a pool. Only bound balances count toward reward weight. An address with no certificate is delegated to the network default.

Stake position is captured at the epoch boundary and applied with the two-epoch lookback the network already uses for its stake distribution, so registration, added stake, and redelegation all take effect after that delay rather than instantly. The delay also removes any benefit from staking immediately before a boundary. On deregistration the deposit is returned and unwithdrawn rewards are paid out in the same transaction.

### 4. Stake pools

A pool is registered by a block producer that is registered and eligible to produce, one pool per producer, declaring an operator reward account and a commission rate. Pools are producer-operated so that the delegation graph built in V1 maps to the producers native stake will back in a later phase.

**Commission ratchet.** A pool MAY lower its commission at any time, effective the next epoch. A pool MUST NOT raise it, and any attempt is rejected. A delegator therefore knows the rate they see is the worst they will ever pay. (Consider if we were to open this: allowed, but only with an extremely lengthend timeline compared to lowering the fee. (~10 epochs) -- this would give stakers ample time to react to the higher fee.. Maybe this can be considered for phase 2)

**Distribution, per pool per epoch.** The pool's reward is the block production reward it earned. The operator retains commission, being that reward multiplied by the commission rate. The remainder is distributed to the pool's stakers in proportion to their reward weight within the pool. automated and not the responsibility of the pool to distribute. This needs to be automated likely via a dedicated claim portal or ideally an airdrop with an option to re-stake rewards on top of existing stake An operator delegating to its own pool earns its delegator share in addition to commission.

Rewards are kept at the pool level. Routing all the way to individual delegators on the Cardano side is deliberately avoided in V1; native NIGHT stakers are paid on Midnight. The exact distribution path is being finalized in the block rewards MIP.

**Network default.** An address with no pool earns its straight share by reward weight and pays no commission. A pool that deregisters moves its delegators to the default without affecting accrued rewards.

### 5. Reward weight

**Base stake** is the staker's mNIGHT captured at the epoch snapshot under section 3.

**Continuity weight** is one, plus the continuity increment multiplied by the continuity epoch count, capped at the maximum. The count is the number of consecutive completed epochs in which the staker's snapshot balance did not fall below the prior epoch's by more than the reduction tolerance. Increases never reset it; a reduction beyond the tolerance or deregistration sets it to zero; redelegating does not.

**Reward weight** is the base stake multiplied by the continuity weight, summed across all stakers for the network total and across a pool's stakers for its pool total. Continuity depends on time only, so splitting a balance across addresses gains nothing.

The continuity weight is the one addition beyond a plain Cardano-style share. It exists to reward sustained holding over harvest-and-sell, and can be dropped if V1 favors maximum simplicity.

### 6. Funding and emission

Staking rewards are paid from the Reserve, the pool of uncirculated NIGHT that exists specifically to fund block production rewards. All NIGHT exists at genesis; the Reserve pays it out on a diminishing curve, so no staking reward is newly minted and the Reserve's drawdown schedule is unchanged. (source: https://midnight.network/whitepaper)

The per-block reward is the outstanding Reserve balance multiplied by a fixed base distribution rate. Because the outstanding balance shrinks as the Reserve pays out, the per-block reward tapers over time. The incentives paper defines this as a smooth tapering curve and puts the Reserve's life at hundreds of years. The published formulas are, in the paper's notation:

- Base reward per block: Nb = Bo × R, where Bo is the outstanding Reserve balance and R is the base distribution rate.
- Base distribution rate: R = π(1 − B − T) / (B × γ), where π is the initial annual inflation rate, B and T are the Reserve and Treasury allocations, and γ is blocks per year.
- Producer reward: Na = Nb × [S + (1 − S) × U], where S is the subsidy rate (initially 95 percent) and U is block utilization.

The staker reward is a share of the block production reward, routed through the producing pool as in section 4, and is therefore funded from the same Reserve budget that already funds block production. It is not new emission and not a separate Reserve draw.

Open items: **Delegator routing:** how the pool's reward reaches individual stakers is being finalized **Reserve replenishment:** what happens once the Reserve is drawn down

### 7. Claim and interfaces

Rewards are computed at the epoch boundary and credited at the start of the next epoch. Balances accrue without expiry, count toward the staker's snapshot balance so they compound, generate DUST, and become spendable through an explicit withdrawal that MUST use the same CLI and SDK surface as block production reward withdrawal under MPS-0019.

Four batch-capable interfaces are required: **forecast** expected reward for a candidate amount and pool; **position** for a stake address; **pool lookup** returning operator, commission and its history, delegated weight, and the stake reward against commission split, satisfying MPS-0019's delegator requirement; and **epoch accounting** sufficient to reproduce every payout.

### 8. Parameters

| **Parameter** | **Proposed** |
| --- | --- |
| Staker share of pool block production reward | being finalized; see section 6 |
| Stake snapshot | epoch boundary, two-epoch lookback (network default) |
| Maximum continuity weight | 1.50 |
| Continuity increment | set so the maximum is reached at 180 days |
| Reduction tolerance | 5 percent of the prior epoch's snapshot balance |
| Stake address deposit | set to exceed the cost of address spam |
| Maximum commission rate | 1.00 |

### Acceptance Criteria

1. A holder can stake mNIGHT and receive rewards without holding ADA, running a Cardano stake pool, or delegating to one.
2. Registration, binding, delegation, redelegation, and deregistration all work on testnet.
3. Pool registration, commission declaration, and commission lowering work, and any attempt to raise commission is rejected.
4. DUST invariance holds on all five points in section 2, with no measurable difference between staked and unstaked balances.
5. A test stakes an entire mNIGHT balance and then unstakes it, with the staker never lacking the DUST to submit either transaction.
6. Stake is liquid: a staker can change position without a lock, and the change is reflected through the epoch snapshot with the two-epoch lookback rather than immediately.
7. An independent party can reproduce every staking payout, including the commission split, from published formulas and chain data.
8. Total emission over a measured interval matches the published Reserve curve, with no increase attributable to staking.
9. All four interfaces are documented, batch-capable, and in the SDK, and withdrawal uses the same conventions as block production reward withdrawal.
10. A participating SPO's Cardano registration, block production, and rewards are measurably unchanged.
11. Ten consecutive testnet epochs run with no reward accounting discrepancy.

## Implementation

One runtime module and one change to how the block production reward is distributed. A runtime module rather than a contract, because a later phase requires committee selection to read staked amounts, and sourcing a consensus input from contract storage would make consensus depend on contract execution.

The module holds stake addresses, payment bindings, pools with commission history, delegations, reward accounts, and per-epoch snapshot and continuity state. Two points make it scale: stake is read from the epoch snapshot rather than accumulated every block, and rewards are pulled, so the epoch boundary records only per-pool aggregates while each staker claims its own unclaimed epochs, bounded per transaction. The block reward distribution change splits the producing pool's reward into operator commission and the staker remainder, and asserts the parts do not exceed the block production reward.

The staking module is the single build item; the interfaces depend on it. Validation is a ten-epoch testnet run with independent reproduction of payouts, and a parameter review before mainnet activation.
