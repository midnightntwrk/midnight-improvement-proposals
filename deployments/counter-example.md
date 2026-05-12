**dApp name:** SundaeSwap Capacity Exchange — Counter Demo

**Contract repository:** https://github.com/SundaeSwap-finance/sundae-midnight-examples/tree/main/contracts/counter

**Brief description:**

A minimal counter contract used as a demo for the SundaeSwap Capacity Exchange Service (CES). The contract maintains a single public `Counter` on the ledger and exposes one circuit (`increment`) that increments it by 1. It has no private state, holds no funds, and stores no user data. Its purpose is to demonstrate sponsored transactions — a user calls `increment`, the CES sponsors the transaction's dust cost, and the counter value updates on-chain. This is a reference implementation for third-party dApp developers integrating with the CES.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | The contract has no private state, no witnesses, and no user-identifying data. The only state is a public counter value. There is nothing to leak. | N/A |
| Value-at-risk | 1 | The contract holds no funds. No tokens are deposited, escrowed, or locked. The only on-chain effect is incrementing a counter. An exploit could only increment the counter incorrectly. | N/A |
| State-space-at-risk | 1 | The contract stores a single `Counter` value (fixed size). State does not grow with usage — every `increment` call mutates the same ledger cell. There is no mapping, no per-user state, and no accumulation. | N/A |
