# MPS-xxxx: Decentralization Through Large-Scale Validator Participation

**Category:** Decentralization

**Created:** 2026-05-11
**Authors:** [Dominik Zajkowski](mailto:dominik.zajkowski@shielded.io)

## Abstract

Today Midnight is operated by a small, federated committee.
Ideally it should be possible to move to an unfederated model.
To achieve that two things need to be true:
- The committee candidates cohort is large enough that it's statistically unlikely to have an adversary take over the chain.
- The committee size is large enough that it's statistically unlikely to select a malicious committee.

Unfortunately, growing the validator set introduces challenges the current architecture does not address: the network topologies that come with a large, distributed committee are in tension with throughput demands.

## Problem

### Unfederated Participation isn't sufficiently refined

In an unfederated model, the protocol must handle the committee size dropping below safe thresholds, stake falling below economically secure levels, and degraded states that become significantly more likely when participants are independent operators with no out-of-band coordination channel.

### Network Topology

A larger, geographically distributed committee will naturally operate over complex topologies — more hops, heterogeneous connectivity, variable latency between participants. As throughput targets increase and block sizes grow, these freeform topologies become an obstacle to propagation within the block window. The tension is between supporting the diverse topologies that come with a large committee and the propagation efficiency that high throughput demands.

### Interaction with Throughput and Propagation Constraints

Growing the committee does not happen in isolation. As described in MPS-xxxx (throughput) and MPS-xxxx (censorship resistance), the network's throughput targets and block propagation limits create tensions with committee size. A decentralization strategy must account for these constraints rather than treat committee growth as an independent variable.

## Goals

### Primary Goal

1. **Support a large, unfederated validator set:** The consensus architecture must support a committee size sufficient for meaningful decentralization, with protocol-level mechanisms for operating without central coordination.

### Secondary Goals

1. **Resilience under degraded participation:** The network must detect and respond gracefully to declining committee size or stake without relying on out-of-band coordination.
2. **Compatibility with throughput and propagation goals:** The decentralization strategy must not conflict with the goals defined in companion MPSs on throughput and censorship resistance.

## Requirements

* **Safety:** The network must maintain safety guarantees as the committee scales and as validators join and leave without coordination.
* **Liveness:** The protocol must define thresholds and recovery strategies for scenarios where committee size or stake drops below safe levels.

## Success Metrics

* **Metric 1:** Defined and tested protocol-level thresholds and responses for degraded committee participation.
* **Metric 2:** Consensus validated at target committee size on a geographically distributed testnet.

## Open Questions

* **What is the target committee size for credible decentralization?** This determines the scale at which the consensus protocol must be validated.
* **What are the safe thresholds for committee size and stake?** Below what levels does the network's security model break down, and what should the protocol do when approaching them?
* **How should topology be structured for large committees?** What relay or gossip strategies can reconcile diverse network topologies with high-throughput propagation requirements?
* **What computational resources are required for validators?** CPU cores, memory, persistent storage, and network bandwith/latency all affect the ability of efficiency of a validator to participate in consensus. Having adequate resources is particularly important in high-throughput environments.
* **Are validators required to archive the full history of blocks?** Or is that role delegated to special-purpose archival nodes?
