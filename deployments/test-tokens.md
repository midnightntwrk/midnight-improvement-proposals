**dApp name**: Test Tokens

**Repository:** https://github.com/SundaeSwap-finance/midnight-swaps-smart-contracts/blob/main/contracts/test-token/test-token.compact

Note: the above repository is temporary, we will move this to the shieldedtech github org as soon as the repository for it is created. This is a Shielded project, not a Sundae Labs project.

**Brief description:**

A very minimal contract, which mints an arbitrary amount of an arbitrary token color to an arbitrary address (using a witness-provided presumably-random nonce).

Our immediate use case is for testing and demoing Shielded's Midnight Intents project against mainnet. This requires having arbitrary amounts of some shielded token to swap. The contract itself is decoupled from that application, and can ideally be reused for demoing other dApps which interact with arbitrary shielded tokens (such as DEXes).

| Category | Self-assessed score (1-3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | This contract only mints Shielded tokens, and does not enforce anything about those tokens. The only witness is a nonce which is (intended to be) purely random. There is no private information to leak. | N/A |
| Value-at-risk | 1 | This contract only mints Shielded tokens, and does not enforce anything about those tokens. Anyone can invoke the contract to mint as many tokens as they want, so those tokens are valueless. The contract itself holds no funds and produces nothing of value. | N/A |
| State-space-at-risk | 1 | This contract only mints Shielded tokens, and does not enforce anything about those tokens. There is no persisted contract state. Each new Shielded UTxO is a new entry in the ledger's Merkle trees, but this has no more impact on state size than an ordinary zswap. | N/A |
