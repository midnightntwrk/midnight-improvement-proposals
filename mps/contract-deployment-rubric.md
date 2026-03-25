# Contract Deployment Rubric

## Rubric scoring

Below is a comprehensive rubric designed for scoring the risk level associated with various dimensions of the Phased. This scoring system aims to provide a standardized, objective framework for assessing potential risks, allowing stakeholders to quickly identify areas of concern and prioritize mitigation efforts. The rubric assigns a risk score to each dimension based on predefined criteria, ensuring a consistent evaluation across all launch initiatives.

Each dApp gets scored 1 to 3 on each of the three risk categories. The score reflects what is actually at stake if something goes wrong, not how likely it is.

- Privacy-at-Risk : If a ZK fault surfaces a bug in the proving system that leaks witness data or enables transaction correlation what does a user lose?
- Value-at-Risk: If there's an exploit, how much can users lose and is that loss recoverable?
- State-Space-at-Risk: How much permanent ledger state does this contract generate and can that growth be attacked?

### Privacy-at-Risk Score

The Privacy-at-Risk Score serves as a crucial rubric for evaluating the potential real-world harm associated with data exposure, particularly in the context of decentralized systems. This score is structured into three ascending tiers of risk, each representing a different level of sensitivity and potential for negative impact on users.

**Tier 1: Minimal Risk - Data Intended / Already Public or Devoid of Real-World Harm**

This tier encompasses data points where exposure, even on a public ledger or platform, is expected to result in minimal to no negative consequence for the individuals involved.

**Data Characteristics:**

- The data is already public knowledge and accessible through other means (e.g., publicly listed contract addresses, standard transaction hashes, or non-sensitive, aggregated metrics).
- The data holds no identifying information and is not linked to real-world identities, behaviors, or personal circumstances.
- The exposure of this data carries no real-world harm—meaning it cannot lead to financial loss, reputational damage, harassment, or physical danger.

**Examples:** Aggregated, anonymized statistical data on protocol usage, or data that is inherently uninformative on its own.

**Tier 2: Moderate Risk - Sensitive but Non-Identity-Level Data**

This tier represents data that, while sensitive and potentially valuable to observers, does not directly expose a person's identity or core behavioral patterns. The harm is typically financial or market-related.

**Data Characteristics:**

- The exposed data reveals financial amounts, such as transaction values, portfolio sizes, or contract balances.
- The data includes counterparties, revealing interactions between two or more addresses or entities, which can imply business relationships or network connections.
- The data includes precise timing information (timestamps, block numbers), allowing for granular analysis of market activity, execution speed, or strategic operations.
- This information is sensitive because it can be used for financial surveillance, market front-running, or the deanonymization of financially active entities, but it is not inherently identity-level (it requires significant off-chain correlation to link to a specific person).

**Examples:** Large transaction amounts between two known protocol addresses, or the frequency and volume of trades executed by a single address.

**Tier 3: High Risk - Data Causing Real-World Harm Upon Exposure**

This is the highest-risk tier, assigned to data whose exposure carries a direct and significant potential for real-world harm, linking digital activity to personal identity and vulnerable behaviors. Exposure at this level constitutes a major privacy breach.

**Data Characteristics:**

- Information that directly links an on-chain address or activity to a verifiable real-world identity (e.g., names, physical addresses, government IDs, email addresses).
- Data revealing highly private or vulnerable activities, such as usage of privacy-preserving tools, medical or health-related transactions, political donations, or engagement in markets where stigma or legal risk is present.
- Data that combines financial, timing, and behavioral information to create a comprehensive profile that can be used for targeted attacks, surveillance, or harassment. This includes linking specific, sensitive on-chain behaviors to a known, identifiable individual or group.

Exposure of Tier 3 data can lead to:

- Physical safety risks (harassment, extortion, doxing).
- Severe financial loss or vulnerability to targeted scams.
- Reputational damage, employment risk, or legal consequences.
- Suppression of participation in sensitive or politically active communities.

### Value-at-Risk Score

This scoring mechanism evaluates the financial risk exposure associated with the smart contract, primarily focusing on the amount of user funds that are potentially vulnerable to a security exploit. A higher score indicates a more significant financial risk as the platform grows.

| **Tier** | **Description** | **Financial Risk Profile** | **Confidence During Initial Launch Phase** |
| --- | --- | --- | --- |
| **1** | **No funds locked in contract.** 
The smart contract architecture is designed such that user assets are never directly held within the contract's balance. Transactions may involve temporary, atomic transfers, but the contract does not serve as a long-term custodian or pool for capital. | **Minimal/Zero Risk.** 
An exploit would not result in the direct loss of user principal held in the contract. Risk is limited to gas fees, transaction failures, or minor, non-custodial functional loss. | **Highest level of confidence.** 
Focus shifts entirely to functional correctness and liveness, rather than catastrophic financial loss. |
| **2** | **Funds temporarily escrowed in contract but for a bounded time.** 
The contract holds user funds, but this is limited to short, defined periods (e.g., during a swap, a loan origination, a timelocked withdrawal, or a fixed-duration auction). The time window for funds being vulnerable is limited and predetermined. | **Bounded Risk.** 
The maximum value-at-risk is constrained by the current volume or capacity of the temporary process. An exploit can cause a loss, but the window of opportunity is limited and the total capital at risk is less than the protocol's total value locked (TVL). | **Moderate confidence.** 
Requires thorough review of the escrow mechanism, withdrawal logic, and timeout procedures. A critical bug has a contained blast radius. |
| **3** | **Funds accumulate in the contract over time.** 
The pool grows with usage and an exploit targeting it grows in severity as the platform succeeds. The smart contract acts as a permanent, long-term vault, liquidity pool, or treasury where user funds are intentionally aggregated and stored. The total value locked (TVL) increases directly with platform adoption. | **Exponential/High Risk.** 
The severity of an exploit is directly proportional to the platform's success and TVL. A successful hack represents a catastrophic, systemic loss for the platform, potentially wiping out all accumulated capital. | **Lowest confidence.** 
Demands the highest level of security scrutiny, formal verification, and external audits. Prioritize limiting the initial TVL cap and slowly increasing it over time. |

### State-Space-at-Risk Score: A Comprehensive Rubric

This scoring rubric evaluates the potential risk and long-term maintenance burden associated with a contract's state-space management. A higher score indicates a greater risk of unbounded growth, increased cost, and potential performance degradation.

**Tier 1: Low Risk (Bounded and Static State)**

The total amount of data (state) the contract can hold is fundamentally limited by its design or a hard-coded maximum. State variables are primarily fixed-size arrays, mapping to a small, predefined set of users/entities, or configuration constants. State is bounded and static (or nearly static); the contract has a defined ceiling on how much state it can produce. The contract itself enforces the maximum state, requiring no external cleanup mechanism.

**Implications:** The contract's fuel consumption and storage cost are highly predictable and do not scale indefinitely with usage. Maintenance overhead is minimal regarding storage management.

**Examples:** A fixed-size governance council registry (e.g., 9 members), a time-locked vault with a single owner, a contract that stores a single, immutable configuration parameter (e.g., a token address or a block timestamp).

**Tier 2: Moderate Risk (Bounded per User, Natural Ceiling)**

The total contract state can grow with the number of users or interactions, but the contribution to state from any single user or a specific, quantifiable set of entities is limited. While the contract doesn't have an *absolute* global ceiling, the nature of the application implies a realistic maximum (e.g., the total number of users or staked assets). State grows with usage but is bounded per user and has a natural ceiling. State-clearing mechanisms are often user-initiated (e.g., withdrawing funds, closing a position), which naturally caps the risk. Developers must monitor user growth relative to storage costs.

**Implications:** State growth is linear with the user base or activity. While manageable, a large user base can lead to significant storage costs. Costs are distributed, and a mechanism (often internal or implicit) prevents *one* user from overwhelming the state.

**Examples:** An ERC-20 token balance mapping (bounded by the number of unique token holders), a decentralized exchange (DEX) where each user can only have one open order per market, a lending protocol where the amount of state is limited by the number of unique depositors/borrowers and their positions.

**Tier 3: High Risk (Unbounded, Cheap Interactions, No Cleanup)**

The contract's state can grow infinitely without limit based on user activity. Individual state-writing transactions are fuel-efficient, incentivizing high frequency. Critically, there is no inherent "expiration date" or mechanism (like self-destruct, cleanup functions, or storage refund incentives) to prune or reclaim storage space. State growth is unbounded, interactions are cheap, state accumulates per interaction and there's no expiry or cleanup mechanism. Requires immediate architectural revision. Solutions include implementing state-pruning functions (incentivized or permissioned), aggregating state (e.g., summarizing logs instead of storing every entry), or introducing an expiry mechanism that forces periodic cleanup.

**Implications:** Storage consumption grows exponentially or rapidly, leading to excessively high network storage costs and potential issues for node synchronization and archival. The contract becomes a permanent data sink, creating a significant long-term financial and technical burden on the network.

**Examples:** A simple public log or message board where anyone can post an unlimited number of entries, a system that tracks *every* historical interaction without aggregation or removal, or a decentralized identifier (DID) system that registers every alias ever created without a deactivation/deletion process.

### Launch Phased Deployment Override Rule:

### This phase represents a critical period for evaluating the stability, performance, and readiness of the product or feature before a full, general release. To ensure maximum stability and minimize potential negative user impact during this controlled rollout, a stringent deployment override rule is enforced.

**Deployment Override Condition:**

A definitive score of 3 on any single category within the rubric will immediately trigger a deployment block. This blocks proceeding with further deployment steps until the underlying issue is fully resolved and the category score is re-evaluated and reduced to an acceptable level.

**Rationale and Implications:**

This high-stakes threshold emphasizes the principle of "fail fast and fix early" within a controlled environment. A score of 3 typically signifies a critical flaw, a severe performance degradation, a significant security vulnerability, or a major operational blocker that poses an unacceptable risk to the pilot user base, system health, or business objectives. This rule ensures that critical issues are identified and mitigated before they impact a wider audience.

If your contract scores a 3 in any category, it is not a permanent block. It is a prompt to rethink the architecture. Join the developer forum or drop into Discord's #dev-chat channel and share your contract structure. The Aliit, the community, and the DevRel team can help you understand what a lower-risk design looks like.
