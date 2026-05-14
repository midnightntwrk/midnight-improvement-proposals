# MPS-xxxx: History Management for Midnight

**Category:** Consensus/Storage

**Created:** 2026-05-14
**Authors:** [Dominik Zajkowski](mailto:dominik.zajkowski@shielded.io)

## Abstract

Midnight's transaction throughput generates a continuous stream of proof data that accumulates over the chain's lifetime.
At target throughput levels, the storage requirements for maintaining full history grow to a point where long-term node operation becomes increasingly expensive, raising barriers to participation and making reliable light clients infeasible.
Without a strategy for managing history size, the network's operational viability and decentralization degrade as the chain ages.
Moving data sharing and availability concerns to the mempool stage of operation does not change this discussion — the data still accumulates and must be managed regardless of when or how it is disseminated.

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

**UC1: Pruned Node Operator**

* **Scenario:** A node operator maintaining only recent history, dropping old data to minimize storage footprint.
* **Limitations:** The current architecture requires full history for consensus participation and block validation.
* **Desired Outcome:** Nodes can fully participate in consensus while maintaining only a bounded window of history.

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

## Goals

### Primary Goal

1. **Enable bounded history operation:** The protocol must allow nodes to participate in consensus, verify current state, and maintain security and privacy guarantees without storing or accessing full chain history.

### Secondary Goals

1. **Support multiple node profiles:** The architecture must accommodate pruned nodes, archival nodes, and light clients as first-class participants with clearly defined capabilities.
2. **Preserve privacy under reduced history:** History management must not introduce new privacy leakage vectors or weaken existing guarantees.
3. **Maintain nullifier and commitment pool integrity:** History management must not compromise the structures essential for transaction validity.

## Requirements

* **Safety:** The network must maintain BFT safety guarantees when nodes hold incomplete history.
* **Liveness:** The protocol must remain live even when the majority of nodes have reduced historical state.
* **Privacy:** History management must not weaken the privacy-preserving properties of the chain.
* **Compatibility:** The design must not conflict with goals defined in companion MPSs on throughput, censorship resistance, and decentralization.

## Success Metrics

* **Metric 1:** Nodes with bounded history can fully participate in consensus and block validation.
* **Metric 2:** New nodes can bootstrap into current consensus state and verify its legitimacy without syncing full chain history.
* **Metric 3:** Historical data points can be verified without requiring the full dataset.
* **Metric 4:** Nullifier and commitment pool integrity is maintained under history management.

## Non-Goals

* Altering the Midnight smart contract model or the core proving system.

## Open Questions

* **What influence does the desired finality time have on history management?** The throughput MPS targets hard finality under 5 seconds. Are there facets of history management that will be affected by such a finality window?
* **Should the protocol operate as a standalone layer one, or anchor finality to another blockchain, effectively operating as a sidechain?** Anchoring provides external finality guarantees affecting synchronization and history management story.
* **Should the protocol support general purpose zero knowledge execution, or optimize specifically for layer two rollup verification on top of a post-and-verify layer one mechanism?**
* **How does the protocol prove or verify data availability when history is reduced, and what minimal guarantees from that proof suffice to maintain consensus integrity and safety?**
