**dApp name:** Proof of Spy (Guess Who)

**Contract repository:** https://github.com/SovoInc/guess-who-contract.git

**Brief description:**
Proof of Spy is a browser-based deduction game on the Midnight network. A random spy (culprit) is selected from a 32-character roster and committed on-chain using a ZK hash commitment. The player asks yes/no questions to narrow down suspects, then submits a final accusation. A ZK proof verifies the accusation against the stored commitment without ever revealing the culprit's identity on-chain. The contract is stateful but holds no user funds; all transaction fees are sponsored server-side using DUST tokens.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | The only data disclosed on-chain is a Boolean game result (correct/incorrect), the public game ID, and a one-way hash commitment. The culprit identity and salt are witness-only and never appear in any public output. No real-world identity data is involved. | N/A |
| Value-at-risk | 1 | The contract holds zero user funds at any time. No token transfers, escrow, or vault logic exists in the contract. Transaction fees are paid from a sponsor wallet controlled by the server operator; users bear no financial exposure beyond gas. | N/A |
| State-space-at-risk | 1 | Each game adds one entry to the `games` map. A `delete_game` circuit allows the sponsor server to prune completed (inactive) games after each session ends, gated by ZK proof that the caller knows the original culprit + salt. Steady-state map size is bounded by the number of concurrently active games. | `delete_game` circuit prunes inactive entries; sponsor server calls it automatically on session completion or timeout. |
