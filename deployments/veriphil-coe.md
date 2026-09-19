# evidentia — Mainnet Deployment Authorization Application

## 1. Summary

AEMIOU GmbH is a Berlin-registered German company operating **veriphil** — a platform that creates blockchain-anchored cryptographic proofs of existence and integrity for digital files. The applicant is Leonard Seyfarth, Geschäftsführer/CEO, AEMIOU GmbH.

The `evidentia` contract publishes 32-byte SHA-256 content hashes into an on-chain Merkle tree, producing tamper-evident records of existence and time. Users select a file, a SHA-256 fingerprint is computed locally (without uploading the file), combined with the user's identifier to form a content hash, and the operator registers this hash on Midnight. The platform serves use cases including:

- **Intellectual property protection** — timestamped proof that creative works, designs, or source code existed at a specific point in time
- **Document integrity** — verifiable evidence that contracts, specifications, or reports were unaltered since registration
- **Dispute avoidance** — early proof establishment before files are shared, published, or negotiated

We request Mainnet deployment authorization for the design described below.

---

dApp name: veriphil

Contract repository: https://github.com/VincentvdE/veriphil

Brief description:

A minimal contract for blockchain-based proof of existence. The contract stores 32-byte SHA-256 content hashes in a fixed-size on-chain Merkle tree, allowing anyone to verify that a file existed at a specific point in time without revealing the file itself. Only the platform operator can add new hashes. The contract holds no funds, stores no personal data, and is used by the veriphil platform to provide tamper-evident timestamping for digital files.


| Category            | Self-assessed score (1–3) | Rationale                                                                                                                                                                                                | Mitigations (if applicable) |
| ------------------- | ------------------------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| Privacy-at-risk     |                         1 | The contract stores only 32-byte SHA-256 hashes. No file contents, filenames, personal data, or user identifiers are stored on-chain.                                                                    | N/A                         |
| Value-at-risk       |                         1 | The contract holds no funds, tokens, escrow, or other assets. No user value can be lost through contract execution.                                                                                      | N/A                         |
| State-space-at-risk |                         1 | The contract stores a fixed-depth `MerkleTree<32, Bytes<32>>` and a single authority key. Storage is bounded by design, does not grow dynamically, and only the authorized operator can add new entries. | N/A                         |
