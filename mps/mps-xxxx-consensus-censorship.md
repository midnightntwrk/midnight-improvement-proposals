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
Title: Censorship Vulnerabilities in Deterministic Block Producer Selection
Authors: Bob Blessing-Hartley @bob-blessing-hartley, Dominik Zajkowski @dzajkowski, Jon Rossie @jon-rossie
Status: Proposed
Category: Core
Created: 29-Apr-2026
Requires: none
Replaces: none

---

## Abstract

The deterministic, round-robin nature of Aura exposes upcoming block producers to targeted Layer 3 and Layer 7 DDoS attacks, compromising the network's censorship resistance.
Because the block production schedule is publicly predictable, adversaries can identify and disable the next leader before their slot begins, enabling selective transaction censorship or temporary chain stalls.
This vulnerability is compounded by two structural constraints: a small, publicly known validator committee that is economically viable to DDoS in its entirety, and a block propagation bottleneck that limits committee growth.
This MPS defines the censorship and availability problems arising from predictable block producer selection and publicly enumerable validator sets, and recommends MIPs to address them.

## Vision

Midnight operates with a block production mechanism where no external observer can determine which validator will produce the next block until that block is already published.
The validator set is sufficiently protected at the network layer that coordinated volumetric attacks against the full committee are impractical.
Block production achieves deterministic slot assignment (eliminating fork-inducing collisions) while maintaining bounded finality, ensuring that ZK-proof verification work is never wasted on orphaned chains.

## Problem

### Censorship via Predictable Leader Selection

Aura relies on a predictable, round-robin schedule for block producers.
This lack of proposer anonymity allows malicious actors to execute targeted volumetric (Layer 3) or application-layer (Layer 7) DDoS attacks against upcoming leaders, serving as a direct resource exhaustion attack against that node.
By knocking leaders offline, attackers can effectively censor specific private transactions or momentarily stall the chain.

It is crucial to distinguish this from a Layer 7 mempool attack, which is a chain-wide vulnerability.
In a mempool attack, the cost model (CPU time, data volume, etc.) allows for a cheap external attack that imposes significant resource exhaustion costs on node operators.
This type of chain attack is common against other networks, and slot anonymity mechanisms do not address this specific vulnerability.

Probabilistic approaches to leader hiding (e.g., VRF-based blind assignment) introduce slot collisions where multiple leaders are elected for the same slot, causing high fork rates.
In a network burdened by heavy ZK-proof verification, validating proofs for blocks that are ultimately orphaned due to forking is an unacceptable waste of computational resources.

### Small Committee Vulnerability

Even if the next block producer's identity is hidden, the full list of committee members is publicly known.
If the validator committee is small, a coordinated, sustained volumetric DDoS attack against *all* participants becomes economically viable, potentially leading to a generalized network stall and systemic censorship.
This is a network-layer problem that cannot be solved by consensus protocol changes alone — it requires complementary infrastructure-level mitigations.

### Block Propagation Constraint vs. Committee Size

The network's high throughput target creates a data propagation bottleneck that directly conflicts with the security goal of a large committee.
The data payload for 1,000 transactions is approximately 5 MB (5KB \* 1000tx = approx 5 MB).
Empirical tests show the current network stack cannot sustain blocks greater than 2 MB without significant forking.

* **Direct Connection Scenario (50 members, 2-second block time):**
  Assuming 50 committee members are directly connected to the block author, the required data transfer pressure is 250 MB over 2 seconds, which necessitates a connection speed of approximately 125 MB/s (around 1 Gbps).
* **Hops Scenario (12 nodes, 2 hops):**
  Assuming block distribution across the network must cover the relevant nodes in two 1-second hops, the first hop to 12 nodes would require transferring 60 MB in 1 second, demanding connection speeds of approximately 500 Mbps.

The necessity for high throughput pushes against the requirement for a sufficiently large validator committee to resist a generalized DDoS attack.

## Use Cases

### UC1: Targeted Censorship via Volumetric Attack

* **Scenario:** An adversary wishes to prevent a specific high-value private smart contract transaction from being included in the ledger.
* **Current State:** The adversary checks the deterministic Aura validator schedule and launches a massive UDP flood (Layer 3 DDoS) against the IP address of the scheduled block producer exactly when the transaction is broadcast.
* **Impact:** The block producer drops offline, the slot is missed, and the transaction is delayed or censored until a non-targeted leader steps in.

### UC2: Generalized Committee DDoS

* **Scenario:** An adversary wishes to halt the chain entirely or force prolonged censorship of all transactions.
* **Current State:** With a small, publicly known committee, the adversary launches a sustained volumetric attack against all validator nodes simultaneously.
  The economic cost of attacking the full set is proportional to committee size — a small committee makes this feasible.
* **Impact:** The chain stalls or operates at severely degraded throughput until the attack subsides, with no consensus-level recourse.

## Goals

1. **Censorship Resistance:** Block producers must not be identifiable by external observers prior to block publication, eliminating targeted pre-slot DDoS as a censorship vector.
2. **Minimal Wasted Computation:** The block production mechanism must avoid protocol-induced forks (slot collisions) to prevent wasted ZK-proof verification on orphaned chains.
   Fork rate should approach 0% under normal network conditions.
3. **Bounded Finality:** The network must achieve verifiable hard finality within a bounded timeframe (e.g., < 5 seconds) and degrade gracefully if the finality mechanism lags, without introducing unbounded fork depth.
4. **Network-Layer Hardening:** The publicly known validator set must be protected against coordinated volumetric (Layer 3) DDoS attacks and the attack surface for application-layer (Layer 7) DDoS must be reduced through infrastructure-level hardening, independent of committee size.    
5. **Security:** The system must maintain the current 2/3-honest-party assumption (Byzantine Fault Tolerance) for safety.
   It must withstand sustained Layer 3 and Layer 7 DDoS attacks without suffering chain halts.
6. **Compatibility:** The design must natively integrate with Midnight's dual-token tokenomics, facilitating DUST fee payments and NIGHT validator rewards.
7. **Privacy:** The architecture must support Midnight's local ZK-proof generation and the concurrent off-chain private state model without introducing new privacy leakage vectors.

## Expected Outcomes

- Transaction inclusion is no longer dependent on whether a specific validator is under attack, making censorship of individual transactions significantly more expensive for adversaries.
- Network availability is not contingent on the simultaneous reachability of a small, publicly enumerable set of nodes, reducing the risk of a generalized chain stall from coordinated attacks.
- The tension between block throughput and committee size is explicitly addressed, enabling validators with standard infrastructure (rather than dedicated high-bandwidth links) to participate in block production.
- Validator nodes spend computational resources on ZK-proof verification for blocks that have a high probability to reach finality.

## Open Questions

* **Anonymity vs. Traffic Analysis:** To what degree can leader anonymity be compromised through network-level traffic analysis (e.g., observing which node initiates block propagation), and what mitigations exist at the networking layer?
* **Finality vs. Liveness Tradeoff:** What are the precise recovery mechanics when transitioning from an asynchronous network partition back to a synchronous state under a strict BFT finality mechanism?
* **Committee Size Floor:** What is the minimum committee size required to make a generalized DDoS attack economically impractical, and is this achievable given the block propagation constraints documented above?
* **Mempool Attack Boundary:** How should the boundary between leader-targeted censorship (addressed here) and chain-wide mempool attacks (out of scope) be enforced in solution design to avoid scope creep?

## Recommended MIPs

### MIP: Secret Leader Election for Block Producer Anonymity

Design and implement a consensus mechanism that hides the identity of the next block producer until block publication.
This MIP should evaluate approaches such as Sassafras-style ring-VRF ticket schemes and other Secret Leader Election (SLE) constructions, with specific attention to deanonymization risks when candidate tickets are submitted on-chain.
The mechanism must provide deterministic single-leader-per-slot assignment to eliminate fork-inducing collisions inherent in probabilistic approaches like BABE.
Addresses the core censorship vulnerability (UC1) and the wasted computation goal.

### MIP: BFT Finality Integration with Bounded Latency

Design the finality layer for the post-Aura consensus architecture, ensuring verifiable hard finality within a bounded timeframe.
This MIP should evaluate integration patterns for BFT finality gadgets (e.g., GRANDPA or equivalent) atop the chosen block production mechanism, including behavior during degraded conditions and recovery from network partitions.
Addresses the bounded finality goal and the finality-vs-liveness open question.

### MIP: Network-Layer DDoS Resilience for Validator Sets

Define infrastructure-level recommendations and protocol features to protect the publicly known validator set against coordinated volumetric and application-layer DDoS attacks.
This MIP should evaluate approaches such as traffic obfuscation, robust peer selection strategies, relay network architectures, and sentry node patterns.
Addresses the small committee vulnerability and UC2, complementing the consensus-layer work in the SLE MIP.

## References

* Aura (Authority Round) consensus documentation
* BABE (Blind Assignment for Blockchain Extension) specification
* Sassafras: Semi-Anonymous Sortition for Applicability — W3F Research
* GRANDPA finality gadget specification

## Acknowledgements

Contributors and reviewers to be listed.

## Copyright

This MPS is licensed under CC-BY-4.0.
