**dApp name:** SundaeSwap Capacity Exchange — Registry

**Contract repository:** https://github.com/SundaeSwap-finance/capacity-exchange/tree/main/packages/registry/contract

**Brief description:**

An on-chain registry of SundaeSwap Capacity Exchange Service (CES) servers. Operators register a domain name under a secret key and post collateral; clients discover available CES servers by reading the public ledger state. No off-chain centralized list is needed.


To register, an operator calls `registerServer` with their domain name and an expiry timestamp, and locks a fixed amount of NIGHT as collateral. Each entry is keyed by the `persistentHash` of an operator-held secret. The contract enforces domain name uniqueness, making two similar entries impossible. An expiry can't be set further out than `maximumRegistrationPeriod`, which is fixed at contract deployment.

Operators can extend their registration via `renewRegistration`. It overwrites the existing entry with a new expiry while preserving the domain name.

Operators can remove their registration and reclaim their collateral via `deregisterServer`. If the entry has not yet expired(`blockTimeLt(entry.expiry)`), the caller's secret key (via the `persistentHash`) is checked against the entry's key. But for expired entires, ownership check is skipped: anyone can call `deregisterServer` and send the collateral to a recipient of their choice. Cleanup is permissionless by design to incentivize operators to renew their registration before expiry; and incentivize anyone for cleaning up lapsed entries.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | The only private state is the operator's 64-byte secret, which is used as a witness. Only its `persistentHash` shows up on-chain as the registry key. All ledger state (`registry`, `takenDomainNames`) is public — domain names and expiry timestamps are disclosed. No identity, financial amount, or sensitive private data is ever revealed. | N/A - public disclosure is intentional. |
| Value-at-risk | 2 | The contract escrows NIGHT collateral from every active registration, so the total at risk grows with the number of operators. Operators lose their collaterals if their secret is compromised. And if an operator simply lets their entry expire without renewing or reclaiming, someone else can claim the collateral. But that's by design rather than a vulnerability. The contract holds no shielded tokens and no end-user funds. | Operators must keep their 64-byte secret confidential. Letting anyone claim expired entries is intentional: collateral is meant to be recoverable once an entry lapses, not an exploit path. The collateral amount is fixed at deployment and visible to every operator. |
| State-space-at-risk | 2 | The `registry` (map) and `takenDomainNames` (set) both grow with the number of distinct active registrations, and there's no hard cap on either. Growth is kept in check two ways: (1) economically, by the NIGHT collateral required for each entry, and (2)temporally, by `maximumRegistrationPeriod`. Everything else in the contract's state is fixed-size and set at construction. | The collateral requirement makes spamming registrations expensive. Cleanup is not a privilege; anyone deregistering expired entries can claim the collateral. This keeps the map from growing unboundedly in practice. |
