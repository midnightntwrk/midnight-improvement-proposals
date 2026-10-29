---
MPS: "0019"
Title: Block Production Rewards in NIGHT
Authors:
  - Karmel E <karmoola>
  - Bob Blessing-Hartley <bobblessinghartley>
Status: Proposed
Category: Core
Created: 06-08-2026
Requires: [none]
Replaces: [none]
MIP: "Native NIGHT Block Production Rewards: Accrual and Claims; Native NIGHT Block Production Rewards: Cardano Reserve Release (together superseding MIP-0024)"
---

# Block Production Rewards in NIGHT

## Abstract

As Midnight decentralizes block production, participating Cardano stake-pool operators (SPOs) become eligible to produce Midnight blocks and earn NIGHT. The Midnight Tokenomics and Incentives Whitepaper states that rewards are tallied and made available for withdrawal on the Midnight ledger by participating SPOs and the delegators whose stake enables their block production. It also states that the initial trusted permissioned producers do not receive block-production rewards.

There is no complete mechanism yet for an SPO or delegator to register where rewards go, forecast rewards, observe accrual, verify calculations, tell pending amounts from claimable ones, or withdraw spendable NIGHT on Midnight. Nor is there a way for an SPO to set, or a delegator to see, the terms on which a pool's reward is shared between the operator and its delegators.

## Vision

An eligible Cardano SPO can evaluate participation, register for Midnight block production, set the terms on which its pool's rewards are shared, observe the rewards its blocks earn, reproduce every calculation independently, and withdraw its share as spendable NIGHT on Midnight. A delegator can see each pool's terms, reproduce their own share, and withdraw through the same native Midnight mechanism. Wallets, explorers and operator tooling can consume one documented, versioned public interface.

## Constraints

The solution must conform to the approved edition of the Midnight Tokenomics and Incentives Whitepaper (version 1.81, May 2026) and to the notified NIGHT MiCA white paper:

* rewards come from the finite protocol Reserve, not from new minting or NIGHT transaction fees;
* the base distribution rate derives from the initial expansion rate, the Reserve and Treasury allocations, and the expected number of Midnight blocks per year;
* each block's base reward is a fixed proportion of the outstanding reward Reserve;
* the initial fixed subsidy is 95% of the base reward;
* the variable component is divided between the block producer and the on-chain Treasury according to block utilization;
* the initial trusted permissioned producers receive no block-production rewards;
* participating SPOs and their enabling delegators can withdraw rewards as NIGHT on the Midnight ledger;
* while Reserve control remains on Cardano, Midnight must not make reward NIGHT claimable until it has observed enough unused stable release from the Cardano Reserve to cover it; and
* who may change reward parameters, and how, must be consistent with what the published disclosures say about the Reserve's parameters and their governance.

The whitepaper does not define how one pool's producer reward is divided between its SPO and its delegators. As on Cardano, each SPO sets a margin for its own pool and the rest is shared by stake; there is no governed cap, and backers are protected by seeing the margin and being able to move their stake. An ADA SPO's margin is the one in its Cardano pool certificate. The protocol must fix the form of the split and its arithmetic, so that any party can reproduce it.

## Problem

At mainnet launch, blocks are produced by permissioned validators that are not rewarded. Reward mechanics become load-bearing as consensus participation opens to eligible Cardano SPOs. The current system contains ledger primitives for Reserve distribution, claimable reward balances, Treasury credits and signed reward claims, but they are not connected into a complete block-production reward lifecycle.

## Gaps

**Eligibility and registration.** An SPO needs to register and deregister for block production. Every recipient, operator or delegator, needs a reward-specific destination on Midnight, authenticated by its Cardano stake credential and by the Midnight key that will receive the rewards. An existing cNIGHT/DUST registration must not be assumed to authorize reward ownership.

**Recovering control of rewards.** A recipient who loses, or suspects compromise of, the key that receives rewards needs a way to move future and unclaimed rewards to a new key, with bounded and publicly visible exposure while the change takes effect.

**Split terms.** An SPO's margin must be declared where its backers already look (for an ADA SPO, its Cardano pool certificate), and a delegator needs to see it before delegating. A change of terms must not alter a reward already calculated.

**Forecasting before joining.** An SPO considering participation needs an estimate based on the published tokenomics, expected block production, utilization and its own split terms, with assumptions and uncertainty visible.

**Monitoring accrual and availability.** Operators and delegators need to distinguish calculated entitlements, amounts waiting for a Cardano Reserve release, claimable balances and completed claims.

**Getting paid.** Operators and delegators need a supported CLI and SDK flow that turns a claimable balance into spendable unshielded NIGHT on Midnight, without a per-recipient Cardano payout service and without creating many small outputs.

**Independent verification.** Participants need enough finalized public data, parameter history, rounding rules, snapshot identifiers and test vectors to reproduce any payout without trusting an indexer.

**Cross-chain supply safety.** While Cardano controls the Reserve, Midnight needs a specified way to observe stable Cardano Reserve releases, and must never settle more than it has observed released.

**Builder integration.** Wallet and explorer developers need documented, standardized, batch-capable runtime, RPC, indexer, CLI and SDK interfaces.

## Expected outcomes

* More Cardano SPOs can make an informed, low-risk decision to participate in Midnight consensus.
* Operators and delegators receive native NIGHT on Midnight through a predictable, inspectable workflow.
* Reward disputes are reduced, because block attribution, formula inputs, split terms, rounding, Reserve release, settlement and claims can all be reconciled independently.
* Delegators can compare pools by their published split terms.
* Wallets, explorers and operator tooling can expose a consistent reward lifecycle.
* Midnight preserves its internal NIGHT supply invariant and the cross-chain supply invariants throughout reward distribution.

## Acceptance criteria

* An SPO can register and deregister for Midnight block production through the Partner Chains mechanism.
* An operator or delegator can register, rotate, remove and query a reward destination on Midnight, authenticated by the Cardano stake credential and by the receiving key.
* A lost or compromised receiving key can be replaced, with a bounded, publicly visible delay and without losing rewards already credited to the stake address.
* Initial trusted permissioned producers receive zero block-production rewards.
* Every eligible block is attributed from consensus state and its reward is calculated with the published tokenomics formula, exact integer arithmetic and the active parameter version.
* The initial subsidy is 95%, and the non-producer variable share is credited to the on-chain Treasury according to block utilization.
* Each SPO's split terms and the form of the split are publicly queryable, and every pool's split is reproducible from them and the committee-selection stake snapshot.
* Operators and delegators can query their calculated entitlement, release-gated amount, claimable amount and withdrawal history.
* The protocol calculates entitlements automatically for eligible block production. While Cardano controls the Reserve, it makes them claimable automatically, but only up to the stable Cardano Reserve releases it has observed and not yet used.
* An operator or delegator can turn a claimable balance into spendable unshielded NIGHT on Midnight through supported CLI and SDK flows.
* Any payout can be recomputed independently from finalized Midnight and Cardano data, the published formula, the split terms and normative test vectors.
* Settlement is bounded, replay-safe and atomic: a failure cannot consume release credit, advance a settlement cursor, or apply only part of a reward chunk.
* Registration ambiguity, a missing destination, key rotation, rollback, retry, runtime upgrade and delayed Reserve release do not move rewards away from the owning stake address, and do not duplicate, destroy or strand them.
* Documented, batch-capable APIs let wallets and explorers show producer and delegator rewards without treating an indexer response as proof of correctness.
* Before activation, the authority that may change reward parameters is confirmed to be consistent with the published disclosures.
* Every ADA SPO's operator cut is set by the margin in its Cardano pool certificate, with no cap and no fixed amount.
* NIGHT-native SPOs can be added by a later NIGHT staking MIP without migrating ADA SPO rewards, escrow or records.

## Dependencies and scope

The supporting rewards MIP depends on a companion MIP that specifies the Cardano Reserve release and its observation by Midnight. The release follows a published schedule, so Cardano needs no knowledge of Midnight state. The release is part of cross-chain supply safety; it is not a per-recipient payout mechanism. Together the two MIPs supersede MIP-0024, which pays rewards as NIGHT on Cardano.

Block producers will be of two kinds: ADA SPOs, which are Cardano stake pools with ADA delegators, and NIGHT-native SPOs, which hold and are backed by NIGHT on Midnight. An operator is one kind or the other, never both. This MPS covers rewards for both kinds; NIGHT staking itself, including NIGHT-native operator records and stake snapshots, is specified by a separate MPS and MIP. A solution to this MPS must accept NIGHT-native SPOs without migrating ADA SPO rewards, escrow or records.

Moving control of the Reserve from Cardano to Midnight is outside this MPS.

Client-side verification of Midnight is outside this MPS. The reward mechanism must remain independently reproducible from finalized public data, but this MPS does not prescribe how a resource-constrained client obtains or verifies that data.

## References

* [Midnight Tokenomics and Incentives Whitepaper, version 1.81 (May 2026)](https://45047878.fs1.hubspotusercontent-na1.net/hubfs/45047878/Midnight-Tokenomics-And-Incentives-Whitepaper.pdf)
* [NIGHT MiCA White Paper](https://a.storyblok.com/f/176646/x/2de4e159af/midnight-price-night-whitepaper.pdf)
* [MIP-0024: Block Production Rewards Payout on Cardano](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0024-block-production-rewards.md)
* [Discussion #273](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/273)
