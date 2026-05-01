# **MPS-xxxx: Addressing Consensus-Related Performance Issues**

**Category:** Performance

**Created:** 2026-04-29  
**Authors: [Bob Blessing-Hartley](mailto:bob.blessing-hartley@shielded.io),[Dominik Zajkowski](mailto:dominik.zajkowski@shielded.io)[Jon Rossie](mailto:jon.rossie@shielded.io)**

## **Abstract**

Achieving a target of 1,000+ transactions per second (TPS) is physically constrained by the current sequential consensus architecture. The sequential verification bottleneck is compounded by the fact that the current network stack cannot disseminate blocks greater than 2MB without significant forking. With zk-SNARK verification at 6 ms per proof on the BLS12-381 curve, processing 1,000 transactions requires 6,000 ms of sequential CPU time. The Substrate runtime allocates only 1,500 ms for user transaction verification within the 6-second block time, sufficient for approximately 250 proofs per block, which is 4x short of the 1,000 TPS target. Some form of parallelization or DAG-based consensus is necessary to decouple verification from the critical path and achieve high throughput.

## **Problem Throughput Bottleneck**

Despite cryptographic optimizations that reduced transaction verification time to 6 ms per proof, the sequential consensus mechanism (Aura for block production, GRANDPA for finality) places all proof verification on a single block leader. The runtime reserves 1.5 seconds for normal dispatch, the class that includes user transactions and their proof verification. Achieving 1,000+ TPS is physically impossible under this model: verifying 1,000 proofs requires 6,000 ms of sequential CPU execution, which is 4x the total compute budget. The bottleneck is architectural, as verification cannot be parallelized across the validator set under the current single-leader block production model.

**UC1: High-Throughput Concurrent Smart Contract Execution**

* **Scenario:** A decentralized exchange (DEX) on Midnight experiences a surge in trading volume, processing hundreds of concurrent shielded transactions.  
* **Limitations:** The sequential verification caps the maximum throughput well below the network's TPS target, causing mempool congestion, delayed execution, and higher operational latency.  
* **Desired Outcome:** The consensus architecture parallelizes ZK-proof verification across the validator set asynchronously, allowing the network to process \> 1,000 TPS without bottlenecking a single leader.

## **Goals**

### **Primary Goal**

1. **Decouple ZK-Verification from Consensus Critical Path:** Restructure transaction ingestion so that the 6 ms cryptographic verification can be parallelized across the network rather than processed sequentially by a single leader.

### **Secondary Goals**

1. **Minimize Fork-Related Wasted Computation:** Reduce or eliminate the probabilistic forks inherent to protocols like BABE to ensure CPU cycles are not wasted verifying ZK-proofs on orphaned chains.  
2. **Maintain Finality Guarantees Under Degraded Conditions:** Ensure the network degrades gracefully if the finality gadget (e.g., GRANDPA) lags, without introducing unbounded fork depth or excessive wasted verification work.

## **Requirements**

* **Performance:** The consensus engine must sustain a minimum throughput of 1,000 TPS. Block times (soft finality) must remain under 2 seconds.  
* **Compatibility:** The design must natively integrate with Midnight's dual-token tokenomics, facilitating DUST fee payments and NIGHT validator rewards.

## **Success Metrics**

* **Metric 1:** Transaction Throughput: Sustain \> 1,000 ZK-SNARK verified TPS in a live, geographically distributed testnet.  
* **Metric 2:** Computational Overhead: Achieve \> 80% CPU utilization across all active validators for parallel proof verification during peak loads, compared to the current single-leader bottleneck.  
* **Metric 3:** Fork Rate: Reduction of protocol-induced forks to near 0% under normal network conditions.  
* **Metric 4:** Finality Latency: Achieve verifiable hard finality within a bounded timeframe (e.g., \< 5 seconds) following rapid soft finality.

## **Non-Goals**

* Altering the Midnight smart contract model or the core proving system.  
* Changing the fundamental economics of the NIGHT/DUST tokenomic structure.  
* Modifying the consensus protocols of the Cardano L1 mainnet.

## **Open Questions**

* **DAG Liveness Nuances:** If adopting a DAG-based protocol like Bullshark and Narwhal, how do we mitigate the theoretical liveness bugs related to round-jumping by honest participants, as recently identified in the Mysticeti DAG analysis?  
* **Hardware Requirements:** If transitioning to an asynchronous DAG where all nodes verify proofs in parallel, how does this alter the baseline hardware requirements for non-authoring validator nodes?  
* **Instant Finality vs. Liveness:** If integrating a strict BFT finality gadget (like GRANDPA) atop a fast-path DAG, what are the precise recovery mechanics when transitioning from an asynchronous network partition back to a synchronous state?

## **Appendices**

### **Appendix A: Throughput vs. Verification Math**

Using the current 6 ms verification time for proofs:

* 100 transactions \= 600 ms of CPU time.  
* 1,000 transactions \= 6,000 ms (6 seconds) of CPU time. This is greater than the total block time.  
  Under a sequential block production model, block times must be strictly greater than 6 seconds to handle 1,000 TPS. Parallelization through a decoupled mempool (Narwhal) distributes this 6,000 ms workload across *n* validators concurrently.
