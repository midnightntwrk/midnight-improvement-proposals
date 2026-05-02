# **MPS-xxxx: Addressing Censorship Vulnerabilities via Block Producer Anonymity**

**Category:** Consensus/Security

**Created:** 2026-04-29**Abstract**  
**Authors: [Bob Blessing-Hartley](mailto:bob.blessing-hartley@shielded.io),[Dominik Zajkowski](mailto:dominik.zajkowski@shielded.io)[Jon Rossie](mailto:jon.rossie@shielded.io)**

## **Abstract**

The deterministic, round-robin nature of Aura exposes upcoming block producers to targeted Layer 3 and Layer 7 DDoS attacks, compromising the network's censorship resistance. Since Aura is already insufficient for the decentralization roadmap—lacking leader anonymity—any consensus overhaul must also address this vulnerability, either through Secret Leader Election (SLE) or a multi-proposer construction. This MPS defines the need for a next-generation consensus architecture that masks the block producer's identity for censorship resistance and decentralization.

## **Problems**

### **Censorship Vulnerability**

Aura relies on a predictable, round-robin schedule for block producers. This lack of proposer anonymity allows malicious actors to execute targeted volumetric (Layer 3\) or application-layer (Layer 7\) DDoS attacks against upcoming leaders, serving as a direct resource exhaustion attack against that node. By knocking leaders offline, attackers can effectively censor specific private transactions or momentarily stall the chain.

It is crucial to distinguish this from a Layer 7 mempool attack, which is a chain-wide vulnerability. In a mempool attack, the cost model (CPU time, data volume, etc.) allows for a cheap external attack that imposes significant resource exhaustion costs on node operators. This type of chain attack is common against other networks, and slot anonymity mechanisms do not address this specific vulnerability.

Alternative slot-based protocols like BABE introduce Blind Assignment via Verifiable Random Functions (VRFs) to hide leaders, but this probabilistic approach leads to multiple leaders per slot (collisions) and high fork rates. In a network burdened by heavy ZK-proof verification, validating proofs for blocks that are ultimately orphaned due to forking is an unacceptable waste of computational resources.

### **Small Committee Vulnerability**

Even if Secret Leader Election (SLE) is implemented to prevent targeting the *next* proposer, the full list of committee members is publicly known. If the validator committee is small, a coordinated, sustained volumetric DDoS attack against *all* participants becomes viable economically, potentially leading to a generalized network stall and systemic censorship. Addressing this requires a multi-layer approach that includes networking-level recommendations, not just a consensus protocol change.

### **Block Propagation Constraint vs. Committee Size**

The network's high throughput target creates a data propagation bottleneck that directly conflicts with the security goal of a large committee. The data payload for 1,000 transactions is approximately 5 MB (5KB \* 1000tx \= approx 5 MB). We already know based on empirical tests that the current network stack cannot sustain blocks greater than 2MB without significant forking which is further justification for investigating alternative dissemination approaches. 

* **Direct Connection Scenario (50 members, 2-second block time):** Assuming 50 committee members are directly connected to the block author, the required data transfer pressure is 250MB over 2 seconds, which necessitates a connection speed of approximately 125MB/s so around 1 Gbps.  
* **Hops Scenario (12 nodes, 2 hops):** Assuming block distribution across the network must cover the relevant nodes in two 1-second hops, the first hop to 12 nodes would require transferring 60MB in 1second, demanding connection speeds of approximately 500 Mbps.

The necessity for high throughput pushes against the requirement for a sufficiently large validator committee to resist a generalized DDoS attack.

### **UC1: Targeted Censorship via Volumetric Attack**

* **Scenario:** An adversary wishes to prevent a specific high-value private smart contract transaction from being included in the ledger.  
* **Current Approach:** The adversary checks the deterministic Aura validator schedule and launches a massive UDP flood (Layer 3 DDoS) against the IP address of the scheduled block producer exactly when the transaction is broadcast.  
* **Limitations:** The block producer drops offline, the slot is missed, and the transaction is delayed or censored until a non-targeted leader steps in.  
* **Desired Outcome:** The block authoring process relies on a cryptographic mechanism that masks the leader's identity until the block is successfully published, nullifying targeted DDoS vectors.

## **Goals: Primary Goal**

1. **Ensure Robust Censorship Resistance:** Prevent the advanced targeting of block producers by adversaries through leader anonymity—either via Secret Leader Election (SLE) or leaderless/multi-proposer construction.

## **Secondary Goals**

1. **Minimize Fork-Related Wasted Computation:** Reduce or eliminate the probabilistic forks inherent to protocols like BABE to ensure CPU cycles are not wasted verifying ZK-proofs on orphaned chains.  
2. **Maintain Finality Guarantees Under Degraded Conditions:** Ensure the network degrades gracefully if the finality gadget (e.g., GRANDPA) lags, without introducing unbounded fork depth or excessive wasted verification work.

## **Requirements**

* **Security:** The system must maintain the current 2/3-honest-party assumption (Byzantine Fault Tolerance) for safety. It must withstand sustained Layer 3 and Layer 7 DDoS attacks without suffering chain halts.  
* **Privacy:** The architecture must support Midnight's local ZK-proof generation and the concurrent off-chain private state model without introducing new privacy leakage vectors.  
* **Compatibility:** The design must natively integrate with Midnight's dual-token tokenomics, facilitating DUST fee payments and NIGHT validator rewards.  
* **Network-Level Hardening:** The solution must include non-protocol recommendations, such as robust peer selection or traffic obfuscation, to protect the publicly known validator set against coordinated volumetric and application-layer DDoS attacks, regardless of committee size.

## **Success Metrics**

* **Metric 1:** Fork Rate: Reduction of protocol-induced forks to near 0% under normal network conditions.  
* **Metric 2:** Finality Latency: Achieve verifiable hard finality within a bounded timeframe (e.g., \< 5 seconds) following rapid soft finality.

## **Non-Goals**

* Altering the Midnight smart contract model or the core proving system.  
* Changing the fundamental economics of the NIGHT/DUST tokenomic structure.  
* Modifying the consensus protocols of the Cardano L1 mainnet.

## **Open Questions**

* **Sassafras Deanonymization Risks:** If relying on Sassafras for Secret Leader Election, how effectively can we mitigate the risk of deanonymization via traffic analysis when candidate tickets are submitted directly to the chain?  
* **Instant Finality vs. Liveness:** If integrating a strict BFT finality gadget (like GRANDPA) atop a fast-path DAG, what are the precise recovery mechanics when transitioning from an asynchronous network partition back to a synchronous state?
