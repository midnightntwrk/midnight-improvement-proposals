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
Title: History Management for Midnight  
Authors: Dominik Zajkowski @dzajkowski  
Status: Proposed  
Category: Core  
Created: 14-MAY-2026  
Requires: none  
Replaces: none  

---

## Abstract

Midnight's transaction throughput generates a continuous stream of proof data that accumulates over the chain's lifetime.
At target throughput levels, the storage requirements for maintaining full history grow to a point where long-term node operation becomes increasingly expensive, raising barriers to participation and making reliable light clients infeasible.
Without a strategy for managing history size, the network's operational viability and decentralization degrade as the chain ages.
Moving data sharing and availability concerns to the mempool stage of operation does not change this discussion — the data still accumulates and must be managed regardless of when or how it is disseminated.

## Vision

Midnight nodes operate with bounded storage requirements regardless of chain age.
Pruned nodes fully participate in consensus while retaining only a recent window of history.
Archival storage is a voluntary choice, not a protocol requirement.
New participants bootstrap into current consensus state quickly, without syncing full chain history.
Historical data can be verified through efficient retrospective proofs without requiring the full dataset.

## Problem

Every transaction carries proof data, making history growth faster and heavier than in conventional chains.
This is not an archival concern — as history grows, the ability to operate nodes, bootstrap new participants, and verify state at scale degrades.

Managing history size creates several tensions:

Faster finality allows more aggressive history management — once state is final, it can be transformed into more efficient representations: compressed proofs, collapsed state, or discarded entirely.
Slower finality means state remains in flux longer, requiring nodes to retain full data until it settles.
The tension is that faster finality is harder to achieve, particularly with larger committees.

Higher throughput generates data faster, increasing the rate at which history accumulates.
At the throughput levels targeted in MPS-xxxx (throughput), history management becomes a continuous operational concern rather than a long-term planning exercise.

As history accumulates, the cost of operating a full node increases — storage, bandwidth for syncing, and time to bootstrap.
This shrinks the pool of viable committee candidates, working against decentralization goals.
History management directly affects how large the committee can practically grow.

Managing history size creates a tension with the integrity of nullifier sets and commitment pools.
These structures are essential for transaction validity and privacy, and it is not obvious how to compact or transform them without compromising the guarantees they provide.

In a zero knowledge system, history management must not weaken the privacy guarantees of the chain.
Reducing or transforming historical data cannot compromise the privacy-preserving properties that the protocol provides to its users.

## Use Cases

**UC1: Pruned Node Operator**

* **Scenario:** A node operator running with bounded history, continuously shedding old data as new blocks arrive to maintain a stable storage footprint.
* **Limitations:** The current architecture requires full history for consensus participation and block validation. There is no mechanism for a running node to discard historical data while maintaining its ability to participate in consensus, serve peers, and preserve nullifier and commitment pool integrity.
* **Desired Outcome:** Nodes can continuously prune history during normal operation without interruption to consensus participation or degradation of protocol guarantees.

**UC2: Archival Node**

* **Scenario:** A node operator voluntarily storing complete historical data for network redundancy and analysis.
* **Limitations:** The current architecture forces all nodes into this role by default.
* **Desired Outcome:** Archival is a voluntary choice, not a protocol requirement.

**UC3: Light Client Bootstrap**

* **Scenario:** A new validator or light client joining the network and bootstrapping into current consensus state.
* **Limitations:** Joining requires syncing full chain history.
* **Desired Outcome:** New participants can verify current state legitimacy and join consensus without syncing full chain history.

**UC4: Historical Data Verification**

* **Scenario:** Proving that a specific historical data point existed on the live network.
* **Limitations:** Verification requires access to the full historical dataset.
* **Desired Outcome:** Efficient retrospective proofs and audits without storing full history.

**UC5: Indexer and Arbitrary History Queries**

* **Scenario:** An indexer or analytics service ingesting blockchain data to support queries against arbitrary historical state — for example, past balances, historical transaction patterns, or contract state at a given block height.
* **Limitations:** If only archival nodes retain full history, indexers depend on archival node availability for ingestion. A pruned node cannot serve as a data source for queries outside its retention window. The protocol currently does not distinguish between data needed for consensus and data needed for historical queries.
* **Desired Outcome:** The protocol clearly defines which data is available from pruned versus archival nodes, enabling indexers to choose their data source based on query requirements. Pruned nodes can serve recent-history queries, while archival nodes or dedicated history-serving infrastructure support arbitrary historical lookups.

## Goals

### Primary Goal

1. **Enable bounded history operation:** The protocol must allow nodes to participate in consensus, verify current state, and maintain security and privacy guarantees without storing or accessing full chain history.

### Secondary Goals

1. **Support multiple node profiles:** The architecture must accommodate pruned nodes, archival nodes, and light clients as first-class participants with clearly defined capabilities.
2. **Preserve privacy under reduced history:** History management must not introduce new privacy leakage vectors or weaken existing guarantees.
3. **Maintain nullifier and commitment pool integrity:** History management must not compromise the structures essential for transaction validity.

## Expected Outcomes

* Nodes with bounded history can fully participate in consensus and block validation.
* Running nodes can manage storage growth over time without loss of protocol functionality.
* New nodes can bootstrap into current consensus state and verify its legitimacy without syncing full chain history.
* Historical data points can be verified without requiring the full dataset.
* Nullifier and commitment pool integrity is maintained under history management.
* The network maintains BFT safety and liveness guarantees when nodes hold incomplete history.
* History management does not conflict with goals defined in companion MPSs on throughput, censorship resistance, and decentralization.

## Open Questions

* **What influence does the desired finality time have on history management?** The throughput MPS targets hard finality under 5 seconds. Are there facets of history management that will be affected by such a finality window?
* **Should the protocol operate as a standalone layer one, or anchor finality to another blockchain, effectively operating as a sidechain?** Anchoring provides external finality guarantees affecting synchronization and history management story.
* **Should the protocol support general purpose zero knowledge execution, or optimize specifically for layer two rollup verification on top of a post-and-verify layer one mechanism?**
* **How does the protocol prove or verify data availability when history is reduced, and what minimal guarantees from that proof suffice to maintain consensus integrity and safety?**
* **What invariants must hold during live history pruning?** When a running node sheds historical data, what constraints govern the pruning process to ensure uninterrupted consensus participation, peer synchronization, and integrity of nullifier sets and commitment pools?
* **What data availability guarantees should pruned nodes provide to indexers?** Can an indexer operate against a pruned node for recent data, or must it always ingest from an archival node? What protocol-level distinction, if any, should exist between data required for consensus and data required for historical queries?

## Recommended MIPs

To be determined as the problem space is explored further.

## Acknowledgements

* Brian Bush (@bwbush)

## Copyright

This MPS is licensed under CC-BY-4.0.
