---
MPS: "0019" 
Title: Block Production Rewards in NIGHT
Authors: Karmel E - karmoola
Status: Proposed  
Category: Core   
Created: 06-08-2026  
Requires: [none]  
Replaces: [none]  

---
# Block Production -mNight

## **Abstract**

As Midnight decentralizes block production, Cardano SPOs become eligible to produce Midnight blocks and earn NIGHT. There is currently no defined mechanism for an SPO to register, forecast, view, verify, or withdraw these rewards, and no way for delegators to see how rewards split between stake rewards and the SPO's margin fee.

**Vision**

An existing Cardano SPO can estimate expected rewards before committing, watch them accrue, independently verify every payout against the published tokenomics, and withdraw using tooling close to what they already run on Cardano.

## **Problem**

At mainnet launch, blocks are produced by permissioned validators. Reward mechanics become load-bearing only as the network decentralizes and Cardano SPOs earn NIGHT. 

## **The Gaps**

**Forecasting before joining:** An SPO weighing participation wants to estimate likely rewards given their stake and network conditions. No exposed data or method exists, so the decision is blind.

**Monitoring an active pool:** A participating SPO wants to watch rewards accrue per block and epoch, confirm correct linkage, and know when they can withdraw. No public, fresh, complete query interface keyed to stake address exists.

**Getting paid:** An SPO wants to claim rewards into spendable mNIGHT via a CLI/SDK flow close to Cardano's, proving ownership, with fees handled and no dust outputs. No withdrawal path exists, so rewards are stranded.

**Delegator choosing a pool:** A delegator wants to look up any pool by stake address and see the stake-rewards-vs-margin-fee split. No breakdown exists.

**Builder/tooling integration:** A wallet or explorer developer wants a documented, standardized, batch-capable API/SDK to surface SPO rewards.

## **Expected Outcomes**

SPOs can make an informed, low-risk decision to produce Midnight blocks, expanding and decentralizing the producer set. Operators trust payouts because they can verify them, reducing disputes and support burden. Delegators can shop pools on transparent terms, improving stake distribution. Wallets and explorers can surface Midnight rewards natively.

## **Acceptance Criteria:**

- An SPO is able to register and deregister
- An SPO can convert accrued rewards into mNIGHT through a CLI/SDK flow mirroring Cardano's conventions
- An SPO can recompute and confirm any payout against published rewards formula (from whitepaper)
- Protocol automatically initiates the payout to SPOs actively producing blocks