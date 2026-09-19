**dApp name:** SundaeSwap Capacity Exchange — Token Mint Demo

**Contract repository:** https://github.com/SundaeSwap-finance/sundae-midnight-examples/tree/main/contracts/token-mint

**Brief description:**

A shielded token vault contract used as a demo for the SundaeSwap Capacity Exchange Service (CES). The contract allows users to mint test tokens, deposit them into a shielded vault, query their balance, and withdraw. It demonstrates the CES's ability to sponsor contract call transactions that involve shielded token transfers.

The `mint_test_tokens` circuit allows **anyone** to mint an unlimited supply of the contract's token. Because the token has a permissionless, uncapped mint, it is economically worthless by design — anyone attempting to create a market for these tokens would see that market immediately drained by newly minted supply.

Internal balances are tracked per user (keyed by a persistent hash of their secret) in a `Map`. The contract holds tokens in a single shielded coin that is consumed and re-created on each deposit/withdrawal. An admin (who proves knowledge of a secret whose `persistentHash` matches the on-chain `admin_key_hash`, so the admin key itself is never stored on the ledger) can prune any entry from the balances map and adjust the depositor cap.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | Private state consists only of a user-generated secret key used to derive an identity hash. The persistent hash is disclosed to the ledger for balance lookup, but the underlying secret never leaves the user's local witness. No real-world identity data is stored. The shielded coin values are protected by Midnight's ZK system — if a ZK fault occurred, it would expose demo token amounts, not identity. Token amounts in a demo contract carry no real-world sensitivity. | N/A |
| Value-at-risk | 1 | The `mint_test_tokens` circuit allows anyone to mint unlimited tokens, making them economically worthless by design. The contract holds shielded tokens in escrow between deposit and withdrawal, but these tokens cannot carry real value — any market for them would be immediately drained by newly minted supply. An exploit could drain the vault, but only of worthless test tokens. | The permissionless mint is intentional and ensures the tokens cannot carry economic value. |
| State-space-at-risk | 2 | The `balances` map grows with the number of unique depositors. A malicious actor could generate ephemeral wallets to create entries. However, the contract enforces a configurable `max_depositors` cap (checked via `balances.size()`), and an admin can prune any entry from the map via `admin_prune` and adjust the cap via `admin_set_max_depositors` (including setting it to 0 to disable new deposits entirely). All other state (token color, tokens held, nonce, counter, contract coin) is fixed-size. | Depositor cap limits map growth. Admin can prune entries and disable deposits. Cap is set at construction and adjustable at runtime. |
