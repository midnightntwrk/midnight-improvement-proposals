---
MPS: "xxxx" # to be assigned
Title: Post-Quantum Consensus, Committee and Governance Keys
Authors:
- Quantova Inc (quantova-inc)
Status: Proposed
Category: Core
Created: 19-Nov-2026
Requires: none
Replaces: none
MIP: none
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

[MPS-0050](/mps/mps-0050-post-quantum-confidentiality-zswap.md) and
[MPS-0051](/mps/mps-0051-post-quantum-spending-authorization.md) cover the ledger. This MPS covers
the keys that run the network itself. Block authoring uses sr25519 (AURA, and BABE with its VRF),
finality uses ed25519 (GRANDPA), bridge commitments use ECDSA over secp256k1 (BEEFY), committee
registrations are signed with the stake pool's Cardano ed25519 key and an ECDSA sidechain key, and
governance acts through accounts that sign with sr25519, ed25519 or ECDSA. Each of these public keys
is on chain once its holder registers or first signs. A cryptographically relevant quantum computer
(CRQC) recovers the private key behind any of them. The result is not a loss of one account's funds
but of block production, finality, the bridge or the runtime itself. This MPS records the keys,
where they are exposed and what each one controls, and sets the goals a migration has to meet. It is
solution agnostic.

## Vision

Midnight's consensus, committee membership, bridge and governance stay under the control of the
parties the protocol names after a CRQC exists. A new node or light client can verify the chain
from a trusted point without relying on any key a CRQC can recover.

## Problem

The keys below come from the runtime in `midnight-node` (`runtime/src/lib.rs`) and the partner
chains toolkit it includes.

| Role | Key | Scheme | Public key exposed | What a recovered key controls |
| --- | --- | --- | --- | --- |
| Block authoring | `aura`, `babe` session keys | sr25519, BABE VRF | at registration | blocks and VRF outputs in that validator's slots |
| Finality | `grandpa` session key | ed25519 | at registration | that validator's finality votes |
| Bridge commitments | `beefy` session key | ECDSA secp256k1 | at registration | that validator's signed MMR commitments |
| Committee registration | sidechain key and stake pool key | ECDSA secp256k1, Cardano ed25519 | on Cardano | which session keys a stake pool's delegation backs |
| Cross chain | `crch` key | ECDSA secp256k1 | at registration | the candidate's cross chain signature |
| Governance | Council, Technical Committee and Federated Authority member accounts | sr25519, ed25519 or ECDSA (`MultiSignature`) | in the account ID for sr25519 and ed25519, at first signed call for ECDSA | motions up to root, including runtime upgrades |

**Finality.** GRANDPA finalizes a block once validators holding more than two thirds of the weight
sign it. A CRQC holder who recovers that many grandpa keys can finalize two conflicting chains. Each
consumer that trusts finality, including bridges, exchanges and light clients, accepts whichever chain
it is shown.

**Committee seats.** A candidate registers on Cardano with a message signed by the stake pool's
ed25519 key and the candidate's ECDSA sidechain key (`filter_invalid_candidates.rs`). Both public
keys are published. A forged registration binds a stake pool's delegated stake to session keys the
attacker holds, without the pool operator's involvement. The stake pool key belongs to Cardano and is
outside Midnight's control.

**Bridge.** BEEFY validators sign commitments to the MMR root that bridge light clients follow. Enough
recovered beefy keys let an attacker sign a commitment to a state that never existed.

**Governance.** The Federated Authority dispatches approved motions as root
(`pallets/federated-authority`), and root can authorize a runtime upgrade. A threshold of recovered member keys is enough to install any runtime.

**Leader selection.** Under AURA a recovered aura key lets an attacker author blocks in that
validator's slots. [MIP-0010](/mips/mip-0010-aura-to-babe-migration.md) proposes BABE, where slot
leaders come from a VRF on sr25519 keys. There a recovered key also lets an attacker compute the
victim's VRF outputs and learn its future slots.

**History.** A node or light client that syncs from genesis checks each past authority set's
signatures. Once old keys can be recovered, an attacker can build a different history signed by
those old sets. Running nodes are not fooled, but new ones have nothing to tell the two apart.

**Size.** No post quantum signature aggregates the way BLS does, so a finality justification grows
by one full signature per signer. At 3,309 bytes for ML-DSA-65 against 64 bytes for ed25519, the
signatures in a justification from 67 signers grow from about 4.3 KB to about 222 KB. Gossip, storage and light
client proofs carry that cost.

## Use Cases

- **Hijacked committee seat.** An attacker forges a registration for a large stake pool and
  produces and finalizes blocks with its weight.
- **Conflicting finality.** An attacker who holds enough grandpa keys finalizes two chains and
  double spends across them, including through the bridge.
- **Malicious runtime upgrade.** An attacker who holds a threshold of governance member keys passes a
  motion that replaces the runtime.
- **Forged history.** An attacker serves a new node a chain signed by recovered keys of an old
  authority set.

## Goals

1. A complete inventory of every consensus, committee, bridge and governance key, with its scheme,
   where its public key is exposed, how it is rotated and what it controls, in line with
   midnight-node#2261.
2. A post quantum signature for finality votes, block seals and bridge commitments, measured for
   vote size, justification size, signing and verification time and finality latency at the
   committee sizes Midnight plans to run, in line with midnight-node#2262.
3. Slot leader selection that a CRQC holder cannot steal from the leader.
4. Post quantum authorization for governance motions and runtime upgrades, with an authenticated
   rotation for current members, in line with midnight-node#2263.
5. A committee registration that a CRQC holder cannot forge, including the part that today depends
   on Cardano stake pool keys.
6. Checkpoints that let a new node or light client verify history from a point signed with a post
   quantum key.

## Expected Outcomes

Validators, stake pool operators and governance members know which keys to replace and in what
order. Bridges and light clients know what they need to verify. The network can rotate its own keys
before a CRQC exists, while the current keys can still authenticate the change.

## Open Questions

- Which scheme fits finality votes, ML-DSA for size and speed or SLH-DSA for its hash only
  assumption, and at which parameter set?
- What justification size and finality latency are acceptable at the planned committee size, and
  does that require committee sampling or a threshold scheme?
- Is there a post quantum VRF mature enough for leader selection, or should leader selection rest on
  a different mechanism?
- How should registrations be bound while Cardano stake pool keys remain classical, for example by a
  post quantum key that the pool commits to before a CRQC exists?
- Should a transition period require both the classical and the post quantum signature on votes and
  motions?
- Who signs checkpoints, and how are they distributed to new nodes and light clients?

## Recommended MIPs

- **Consensus key inventory.** Publishes goal 1 for the current release and keeps it current.
- **Post quantum finality and block authoring.** Selects the scheme and defines vote, justification
  and seal formats with an activation rule. Addresses goals 2 and 3.
- **Post quantum governance authorization.** Defines the signature for member accounts and motions,
  and the rotation of current members. Addresses goal 4.
- **Post quantum committee registration.** Defines a registration that binds a post quantum key to a
  stake pool. Addresses goal 5.
- **Signed checkpoints.** Defines checkpoint content, signers and distribution. Addresses goal 6.

## References

- [MPS-0050, Post-Quantum Confidentiality for Zswap](/mps/mps-0050-post-quantum-confidentiality-zswap.md)
- [MPS-0051, Post-Quantum Spending Authorization for Unshielded Outputs](/mps/mps-0051-post-quantum-spending-authorization.md)
- [MIP-0010, AURA to BABE migration](/mips/mip-0010-aura-to-babe-migration.md)
- [midnight-node#2261, map active cryptographic acceptance paths](https://github.com/midnightntwrk/midnight-node/issues/2261)
- [midnight-node#2262, evaluate post quantum consensus authorization](https://github.com/midnightntwrk/midnight-node/issues/2262)
- [midnight-node#2263, design cryptographic migration for governance](https://github.com/midnightntwrk/midnight-node/issues/2263)
- [FIPS 204, ML-DSA](https://csrc.nist.gov/pubs/fips/204/final) and [FIPS 205, SLH-DSA](https://csrc.nist.gov/pubs/fips/205/final)
- [NIST IR 8547, Transition to Post-Quantum Cryptography Standards](https://csrc.nist.gov/pubs/ir/8547/ipd)

## Acknowledgements

None yet.

## Copyright

This MPS is licensed under CC-BY-4.0.
