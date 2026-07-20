---
MPS: "0030"
Title: Decentralization Through Large-Scale Validator Participation  
Authors: Dominik Zajkowski @dzajkowski  
Status: Proposed  
Category: Core  
Created: 11-MAY-2026  
Requires: none  
Replaces: none  

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

Today Midnight is operated by a small, federated committee.
Ideally it should be possible to move to a permissionless model.
To achieve that, the candidate pool must be large and diverse enough — and the committee selection mechanism secure enough — that the probability of an adversary controlling the committee is negligibly small.

Unfortunately, growing the validator set introduces challenges the current architecture does not address: the network topologies that come with a large, distributed committee are in tension with throughput demands.

## Problem

### Permissionless Participation

In a permissionless model, the protocol must handle the committee size dropping below safe thresholds, stake falling below economically secure levels, and degraded states that become significantly more likely when participants are independent operators with no out-of-band coordination channel.

### Network Topology

A larger, geographically distributed committee will naturally operate over complex topologies — more hops, heterogeneous connectivity, variable latency between participants. As throughput targets increase and block sizes grow, these freeform topologies become an obstacle to propagation within the block window. The tension is between supporting the diverse topologies that come with a large committee and the propagation efficiency that high throughput demands. This tension is shared with MPS-xxxx (throughput) and MPS-xxxx (censorship resistance), and a decentralization strategy must account for their constraints.

## Goals

### Primary Goal

1. **Support permissionless validator participation:** The protocol must allow any eligible participant — one who meets the protocol-defined eligibility criteria (e.g., minimum stake, operational requirements) — to enter the candidate pool, and the committee selection mechanism must maintain security guarantees as the candidate pool grows. The implementing MIP must precisely define these criteria before proceeding to implementation.

### Secondary Goals

1. **Resilience under degraded participation:** The network must detect and respond gracefully to declining committee size or stake without relying on out-of-band coordination.
2. **Compatibility with throughput and propagation goals:** The decentralization strategy must not conflict with the goals defined in companion MPSs on throughput and censorship resistance.

## Requirements

* **Safety:** The network must maintain safety guarantees as the committee scales and as validators join and leave without coordination.
* **Liveness:** The protocol must define thresholds and recovery strategies for scenarios where committee size or stake drops below safe levels.
* **Quantitative security thresholds:** The protocol must define and validate the security properties of its committee selection mechanism — whether that involves known committees, anonymous sortition, or a hybrid — expressed as the probability of adversary takeover given stake distribution and adversary resources. These thresholds must be established through research and testing.

## Open Questions

* **How should topology be structured for large committees?** What relay or gossip strategies can reconcile diverse network topologies with high-throughput propagation requirements?
* **What computational resources are required for validators?** CPU cores, memory, persistent storage, and network bandwidth/latency all affect the ability and efficiency of a validator to participate in consensus. Having adequate resources is particularly important in high-throughput environments.
* **How does validator storage interact with history management?** See MPS-xxxx (history management) for the full treatment of archival versus pruned node operation.

## Vision

Midnight operates with a permissionless validator candidate pool.
The security properties of committee selection are well understood as a function of candidate pool size and stake distribution, and the protocol has strategies for handling pool shrinkage.
The network maintains throughput and safety as participants join and leave independently.

## Use Cases

**UC1: Independent Validator**

* **Scenario:** An independent operator joins the candidate pool and participates in consensus without requiring approval or coordination with existing validators.
* **Limitations:** The current federated model does not support permissionless entry.
* **Desired Outcome:** Any eligible participant can join the candidate pool and be selected for committee duty.

**UC2: Degraded Participation**

* **Scenario:** A substantial fraction of validators — the precise threshold to be defined by the implementing MIP — leave the candidate pool or go offline, reducing the pool size and stake below levels defined as safe by the implementing MIP.
* **Limitations:** The protocol does not define thresholds or recovery strategies for shrinking participation.
* **Desired Outcome:** The protocol detects degraded participation and prioritizes safety, maintaining liveness where possible within defined thresholds.

**UC3: Geographically Distributed Committee**

* **Scenario:** Committee members operate across heterogeneous network topologies — varying hop counts, bandwidth, and latency — that are largely outside the protocol's control once participation becomes permissionless. The implementing MIP must define acceptable bounds on topology diversity, including minimum connectivity and maximum tolerable latency.
* **Limitations:** Block propagation within the block window becomes harder as topology complexity grows, creating tension with throughput targets.
* **Desired Outcome:** The network reconciles diverse topologies with high-throughput propagation requirements.

**UC4: Cardano SPO Operator**

**Scenario:** A Cardano stake pool operator extends their existing operation to also run a Midnight validator, using the same hardware and operations team.
**Limitations:** Midnight's resource and operational requirements may exceed what an existing Cardano SPO can absorb without significant new infrastructure or expertise.
**Desired Outcome:** Running a Midnight validator fits within the operating envelope of an existing Cardano stake pool, expanding the candidate pool with operators whose infrastructure and practices are already proven.
## Expected Outcomes

* The protocol supports permissionless validator participation with quantifiable security guarantees.
* Committee selection remains secure across a range of candidate pool sizes and stake distributions.
* The network detects degraded participation and has protocol-level strategies for recovery within defined thresholds, with clear escalation points beyond which out-of-band coordination is required.
* Decentralization goals are achieved without conflicting with throughput and censorship resistance targets.

## Recommended MIPs

To be determined as the problem space is explored further.

## Acknowledgements

* Brian Bush (@bwbush)

## Copyright

This MPS is licensed under CC-BY-4.0.
