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

---
MPS: xxxx
Title: ZK-Proof Verification Throughput Bottleneck
Authors: Bob Blessing-Hartley <bob.blessing-hartley@shielded.io>, Dominik Zajkowski <dominik.zajkowski@shielded.io>, Jon Rossie <jon.rossie@shielded.io>
Status: Proposed
Category: Core
Created: 29-Apr-2026
Requires: none
Replaces: none

---

## Abstract

Achieving a target of 1,000+ transactions per second (TPS) is physically constrained by sequential ZK-proof verification on a single block leader.
With a 6-second slot (block creation and dissemination time), sustaining 1,000 TPS requires 6,000 transactions per block.
At 6 ms per zk-SNARK verification on the BLS12-381 curve, this requires 36,000 ms of sequential CPU time.
The Substrate runtime allocates only 1,500 ms for user transaction verification within the 6-second slot, sufficient for approximately 250 proofs per block — roughly 42 TPS, which is ~24x short of the 1,000 TPS target.

## Vision

The consensus architecture enables the network to sustain 1,000+ ZK-SNARK verified TPS by distributing proof verification across the validator set rather than concentrating it on a single block leader.
Block creation times remain under 2 seconds, with verifiable hard finality within a bounded timeframe.
Protocol-induced forks are near zero under normal network conditions, and the network degrades gracefully if the finality gadget lags, without introducing unbounded fork depth or excessive wasted verification work.

## Problem

Despite cryptographic optimizations that reduced transaction verification time to 6 ms per proof, the sequential consensus mechanism (Aura for block production, GRANDPA for finality) places all proof verification on a single block leader.
The runtime reserves 1.5 seconds for normal dispatch, which must cover user transactions, proof verification, and state storage operations.

Using the current 6 ms verification time for proofs and a 6-second slot:
* 250 proofs per block = 1,500 ms of CPU time = ~42 TPS (current maximum).
* 1,000 TPS = 6,000 proofs per block = 36,000 ms of CPU time — 24x the dispatch budget.

Achieving 1,000+ TPS is physically impossible under this model.
The bottleneck is architectural, verification is sequential and cannot be parallelized across the validator set under the current single-leader block production model.

## Use Cases

**UC1: High-Throughput Concurrent Smart Contract Execution**

* **Scenario:** A decentralized exchange (DEX) on Midnight experiences a surge in trading volume, processing hundreds of concurrent shielded transactions.
* **Limitations:** The sequential verification caps the maximum throughput well below the network's TPS target, causing mempool congestion, delayed execution, and higher operational latency.

**UC2: Cross-Chain Settlement**

**Scenario:** A bridge moves value between Midnight and another chain, waiting for hard finality on the Midnight side before releasing the corresponding value on the destination chain.
**Limitations:** Finality on Midnight today takes around 15 seconds, which is several times slower than on competing settlement layers, so a bridge built on Midnight cannot offer users a competitive cross-chain settlement experience and that traffic will route elsewhere.
**Desired Outcome:** Hard finality on Midnight is fast enough that a bridge can settle at parity with the fastest production settlement layers, and cross-chain users have a reason to keep their workflows on the chain.

## Goals

1. **Sustain 1,000+ ZK-SNARK verified TPS:** The verification architecture must support processing 1,000+ proofs per second.

## Expected Outcomes

* Transaction Throughput: Sustain > 1,000 ZK-SNARK verified TPS in a live, geographically distributed testnet.
* Degradation: Throughput must not degrade under demand that exceeds capacity: i.e., observed throughput (averaged over a several-block time window) must be a non-decreasing function of transaction demand.

## Non-Goals

* Altering the Midnight smart contract model or the core proving system.
* Changing the fundamental economics of the NIGHT/DUST tokenomic structure.
* Modifying the consensus protocols of the Cardano L1 mainnet.

## Open Questions

* **Hardware Requirements:** If transitioning to an architecture where all nodes verify proofs in parallel, how does this alter the baseline hardware requirements for non-authoring validator nodes?
* **Fork-Related Wasted Computation:** Probabilistic forks inherent to protocols like BABE can cause CPU cycles to be wasted verifying ZK-proofs on orphaned chains.
If the solution to verification throughput involves changes to consensus (e.g., DAG-based protocols), this may also reduce or eliminate protocol-induced forks.
* **Finality Guarantees Under Degraded Conditions:** The current finality gadget (GRANDPA) may lag under high load.
If a solution introduces a different consensus structure, the network must degrade gracefully without introducing unbounded fork depth or excessive wasted verification work.
The precise recovery mechanics when transitioning from an asynchronous network partition back to a synchronous state need further investigation.
* **Block Dissemination Limits:** The current network stack cannot disseminate blocks greater than 2MB without significant forking.
This compounds the verification bottleneck — even if verification throughput is solved, large blocks of verified transactions must still propagate.
* **Block Structure:** If verification and transaction dissemination move out of the block leader's critical path, blocks may no longer need to carry full transactions — they could reference already-verified, already-disseminated transaction bundles.
Whether this changes the block dissemination constraint or introduces new ones depends on the solution architecture.
* **Throughput Target Definition:** The 1,000 TPS target needs further quantification in terms of circuit composition.
Is this for a "typical" workload of smart contracts, for the built-in circuits, or some other mix?
"Typical workload" needs to be defined.
Should this measurement be reworded in terms of kbps instead of nebulous txps?
* **Fork Tolerance Thresholds:** What is the acceptable fork probability under both normal and adversarial conditions?
What does 'near zero' mean? What is the importance of the duration of a fork?
Can relatively frequent one-block forks be considered less harmful than a less frequent 30-block fork?

## Recommended MIPs

The following MIPs are recommended to address the problem domains identified in this MPS:

1. **Parallel Proof Verification via DAG-Based Consensus.**
Evaluate DAG-based protocols (e.g., Narwhal/Bullshark) to decouple mempool dissemination from consensus ordering, distributing the 6,000 ms proof verification workload across *n* validators concurrently.
Should address DAG liveness nuances, including the theoretical liveness bugs related to round-jumping by honest participants as identified in the Mysticeti DAG analysis.

## Acknowledgements

## Copyright

This MPS is licensed under CC-BY-4.0.
