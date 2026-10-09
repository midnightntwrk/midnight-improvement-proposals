---
MPS: "xxxx" # to be assigned
Title: Post-Quantum Confidentiality for Zswap
Authors:
- David Nevado (davidnevadoc)
Status: Proposed
Category: Core | Standards
Created: 06-Oct-2026
Requires: none
Replaces: none

---

<!--
 Copyright 2026 Midnight Foundation

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

Zswap, Midnight's shielded multi-asset UTXO scheme, provides confidentiality of
shielded transfers with two cryptographic mechanisms. A cryptographically
relevant quantum computer (CRQC) breaks one of these mechanisms but not the
other.
The ownership layer, which links a coin to its owner, relies on the first
mechanism. It is hash-based and hidden by statistical zero-knowledge proofs,
neither of which a quantum computer breaks.
The note contents in transactions rely on the second mechanism. They are
encrypted to the recipient using traditional cryptography. In particular, they
use Diffie-Hellman (DH) key exchange, which can be broken by a CRQC.
The exposure is therefore retroactive and already accruing: every note
ciphertext recorded on-chain today is a harvest-now-decrypt-later target. What a
decryption reveals is bounded to the note's contents (amounts, token types and
counterparties); spend-linkage stays hidden, because it does not depend on the
broken key exchange.
This MPS frames the confidentiality gap; the integrity gap and the concrete
migration belong in a separate MPS and MIP, respectively.

## Vision

Shielded transactions on Midnight stay confidential after a CRQC exists:
amounts, token types and counterparties that were shielded when a transaction
was recorded stay shielded forever.
Midnight acquires cryptographic agility for the confidentiality primitives: the
ability to replace the note key exchange with a post-quantum counterpart without
re-architecting Zswap becomes a documented property of the protocol.

## Problem

Zswap's confidentiality rests on two assumptions. A CRQC breaks one and weakens
the other, and the ledger does not distinguish between them anywhere a reviewer
can see.

**The retroactive threat (harvest-now-decrypt-later).**
Coins are delivered by encrypting the note's contents (value, token type and
nonce) to the recipient address under a DH key exchange. The encrypted note is
stored on-chain and is therefore publicly available, but only decryptable by the
legitimate recipient, who owns the secret key of the address. The symmetric
layer of that construction is quantum-resistant, but the key exchange is not. A
CRQC can recover the recipient's decryption secret from their published address,
after which every note ever sent to that address decrypts. Addresses are often
posted publicly, shared with counterparties and held by exchanges and indexers,
so the exposure is broad.

- **What leaks** is the note content: amounts, token types and, for known
  addresses, counterparties.
- **What does not leak** is spend-linkage: nullifiers are derived by hashing a
  spending key that appears in no ciphertext and on no curve, so even a full
  quantum adversary cannot tell whether or when a decrypted note was spent.

This asymmetry is a direct consequence of the hash-based ownership layer. The
proofs do not weaken this either. Zswap's proofs are statistically
zero-knowledge, so even to an unbounded adversary a historical proof leaks
nothing about the ownership secret. The retroactive exposure is therefore
bounded to the note ciphertexts, and the only defense is to close the key
exchange before the ciphertexts are recorded.
The main consequences of this problem are the following concrete attacks:

- **Retroactive de-anonymization of a shielded holder.** A user receives
  shielded transfers over months; their address is known to a counterparty and
  an exchange. Years later a CRQC solves one discrete log, and every note ever
  sent to that address decrypts, retroactively attaching amounts and token types
  to specific historical outputs. Even if the user tries to mitigate the attack
  at Q-day, they have no way to protect data already on-chain; the only defense
  had to be in place before the ciphertexts were recorded.

- **Reconstruction of a historical counterparty graph.** Once several addresses
  decrypt, an observer holding the full chain history can align senders and
  recipients across transactions and rebuild who paid whom, and how much, over
  the entire life of the shielded pool. The privacy loss is not per-user but
  structural, and it lands on transactions that were confidential when made.

## Goals

**A post-quantum note-delivery system, before a CRQC is credible.** Every note
recorded until the upgrade is deployed stays harvestable forever, so this is the
only confidentiality exposure whose cost accrues while waiting.

## Expected Outcomes

Zswap notes on Midnight stay private after a CRQC arrives, closing the
harvest-now-decrypt-later exposure. Spend-linkage and individual ownership,
already post-quantum, are carried through unchanged.

## Open Questions

- **What CRQC timeline is Midnight planning against?** The deadline for the
  migration is the estimated Q-day minus the required secrecy lifetime of
  shielded data. Both figures are risk-appetite decisions, not technical ones,
  and the goal has no concrete date until they are set.

- **Is the sunk exposure accepted?** Ciphertexts recorded before the migration
  remain harvestable forever; no future upgrade can protect them. Should this be
  explicitly accepted and communicated, or mitigated operationally (e.g. address
  rotation)?

## Recommended MIPs

This section names a solution domain only; the mechanism and staging belong to
the MIP itself.

- **Post-quantum note-delivery system.** Closes the harvest-now-decrypt-later
  exposure while preserving the existing symmetric layer. Urgent: the only cost
  that accrues while waiting.

## References

- Engelmann, Kerber, Kohlweiss, Volkhov. *Zswap: zk-SNARK Based Non-Interactive
  Multi-Asset Swaps.* IACR ePrint 2022/1002; PoPETs 2022(4):507–527.
  <https://eprint.iacr.org/2022/1002>
- Midnight ledger implementation (`midnightntwrk/midnight-ledger`):
  `spec/zswap.md`, `spec/properties.md`, `transient-crypto/src/encryption.rs`,
  `transient-crypto/src/merkle_tree.rs`.
- Related MPSs: **MPS-0011** (native cryptographic primitives).
- **MIP-0001:** Midnight Improvement Proposal Process.

## Copyright

This MPS is licensed under CC-BY-4.0.
