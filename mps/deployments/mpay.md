# MPay

## Contract repository

https://github.com/gianalarcon/M-pay

## Brief description

MPay is a privacy-preserving multisig wallet on the Midnight network. Signers are identified by ZK commitments (hash of a browser-local secret, not public keys), so no on-chain linkage between signers and wallet identities. Transfer proposals encrypt the recipient shielded address and amount with a vault key shared out-of-band among signers; on-chain observers see only encrypted ciphertext chunks. The vault holds shielded MPAY tokens (self-minted custom token via `mintShieldedToken`, permissionless mint) and disburses via multisig-approved `sendShielded`.

Primary circuits: `deposit`, `propose` (generic, txType routed), `approveTx`, `executeTransfer/AddSigner/RemoveSigner/SetThreshold`, `stampReady`, `pruneTx`.

## Risk assessment

### Privacy-at-risk: **1**

No information on-chain links activity to any real-world identity. Specifically:

- Signer secrets are browser-local, derived from an arbitrary `signData` message. They never leave the dApp and are not tied to the user's wallet identity.
- On-chain signer commitments are `persistentHash(secret)` — preimage-resistant, not derived from any identifying key.
- Transfer recipient and amount are AES-256-GCM encrypted under a shared vault key, stored in `txData0-3`. Observers see only ciphertext.
- Vault coin values are visible (shielded-token Zswap semantics), but there is no link to the depositor.

A ZK proof fault or witness leak would expose the signer's `secret`, which de-anonymizes that signer within the multisig — but does not expose any real-world identity.

### Value-at-risk: **1**

The MPay vault holds only the custom MPAY token deployed by the same project (`contract/src/token.compact`). The `mint` circuit is **permissionless** — anyone can mint any amount — so the token has no intrinsic economic value. The vault is a demo-scope store of a permissionless token, equivalent to the Sundae `mint-vault-example` case.

No protocol-owned funds, no user escrow of externally-valued assets, no accumulation of third-party tokens.

### State-space-at-risk: **2**

Ledger fields fall into three groups:

**Bounded / fixed-size:** `tokenColor`, `threshold`, `owner`, `finalized`, `txCounter`, `depositCounter`, `signerCount`, `signers` (Set grows/shrinks by explicit signer-governance txs).

**Grows with usage, has explicit cleanup:**

- `vaultCoin` Map — grows per `deposit`, shrinks per `executeTransfer` (`vaultCoin.remove`).
- `txTypes`, `txStatuses`, `txApprovalCounts`, `txData0-3` — grow per `propose`; **`pruneTx(txId)` removes all seven** for any tx that is either `EXECUTED` or `PENDING` with 100+ newer proposals. `pruneTx` is open to anyone, no signer gate.

**Protocol-necessary, intentionally unbounded:**

- `txNullifiers` Set — grows one per approval vote. Cannot be cleaned without breaking anti-replay security (analogous to Ethereum account nonces). Bounded in practice by signer count × lifetime approvals.

Detailed mitigation design and rationale: [`docs/adr/008-state-space-cleanup-pruneTx.md`](https://github.com/gianalarcon/M-pay/blob/main/docs/adr/008-state-space-cleanup-pruneTx.md).
