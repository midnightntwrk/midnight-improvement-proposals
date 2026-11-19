---
MPS: "xxxx" # to be assigned
Title: Post-Quantum Soundness of Proofs and Shielded Value Balance
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

[MPS-0050](/mps/mps-0050-post-quantum-confidentiality-zswap.md) covers what a cryptographically
relevant quantum computer (CRQC) reveals about shielded notes and leaves the integrity gap to a
separate MPS. [MPS-0051](/mps/mps-0051-post-quantum-spending-authorization.md) covers the signatures
that authorize unshielded spends. This MPS covers the rest of integrity, the checks that decide
whether a transaction is valid at all.

Two of them rest on assumptions a CRQC breaks. Every Midnight proof is a Plonk proof with KZG
commitments over BLS12-381, and KZG binding fails once the setup secret is recovered from the
public parameters. Shielded value balance is a Pedersen check on the embedded JubJub curve, and it
fails once the discrete logarithm between its generators is known. Either failure lets an attacker
create value from nothing, and the second one does so even if the proofs stay sound. Unlike the
confidentiality gap, nothing here is retroactive. Proofs accepted before a CRQC stay valid. The cost
is lead time, because replacing a proof system and a balance mechanism is the longest change in any
post quantum migration.

## Vision

A transaction that Midnight accepts after a CRQC exists is valid for the same reasons it is valid
today. No adversary with a quantum computer can produce an accepting proof for a false statement or
a balanced transaction that creates value. Privacy is kept as it is, since Zswap proofs are already
statistically zero knowledge.

## Problem

**Proof soundness.** `midnight-zk` implements Plonk with KZG polynomial commitments over BLS12-381
(`proofs/src/poly/kzg`). A KZG commitment is binding only while no one knows the secret `τ` behind
the structured reference string. The string publishes `[τ]G1`, so a CRQC running Shor's algorithm
recovers `τ`, and with it opens any commitment to any value. The resulting proofs pass the Plonk
verifier for statements that are false. This covers every proof the ledger checks, Zswap spends and
outputs (`zswap/src/verify.rs`), Dust spends, and every contract operation through its verifier key.

**Value balance.** Shielded amounts are hidden in Pedersen commitments on the embedded curve,
`C = H(type)^v · g^r`, with a generator `H(type)` per token type and segment. `pedersen_check` in
`ledger/src/verify.rs` sums the input, output and binding commitments of a transaction and compares
the sum with the binding randomness times `g` plus the public value deltas. The binding commitment
uses `PureGeneratorPedersen`, which proves knowledge of its exponent with a Fiat Shamir Schnorr proof
on the same curve. All of this is binding only while the discrete logarithm of `H(type)` to base `g`
is unknown. A CRQC computes it, after which an output can commit to a larger value than the inputs
with randomness chosen so that the sum still matches. The proof for that output can be entirely
honest, so this failure does not depend on breaking the proof system.

**Embedded signatures.** Schnorr signatures over the embedded curve
(`transient-crypto/src/schnorr.rs`) rest on the same discrete logarithm and fail the same way.

**No signal.** Forged shielded value looks exactly like real shielded value. Nothing on chain would
show that the supply of a shielded token had grown.

**Lead time.** A replacement has to keep proof size, proving time and memory, verification cost and
recursion within limits that wallets, the proof server and block producers can carry. Every
contract verifier key would need to be regenerated under the new system, and existing contracts
would need a path to move to it. That work has to be done before it is needed, not after.

## Use Cases

- **Counterfeit shielded tokens.** An attacker who knows the discrete logarithm between `g` and
  `H(type)` builds a transaction whose output commits to more value than its inputs, with an honest
  output proof, and passes `pedersen_check`.
- **Forged spends.** An attacker who knows `τ` produces a spend proof for a coin commitment it does
  not own, with a nullifier of its choice. The real owner can still spend the coin later under the
  real nullifier, so the value is duplicated.
- **Bypassed contract rules.** A forged proof satisfies any check a contract circuit enforces on
  private inputs, such as knowledge of an owner secret, so a contract that holds funds behind such a
  check can be drained.

## Goals

1. A complete map of every acceptance check that depends on a discrete logarithm or pairing
   assumption, covering proofs, Pedersen balance, proofs of knowledge, embedded signatures and the
   verifier key lifecycle, in line with midnight-ledger#796.
2. Published measurements of post quantum sound proof systems on representative Zswap and contract
   relations, covering proof size, proving time and memory, verification time and recursion, in
   line with midnight-zk#567.
3. A value balance mechanism whose binding does not depend on a discrete logarithm.
4. A defined path for existing contracts and their verifier keys to move to the new proof system.
5. A way to detect counterfeit shielded supply if soundness ever fails, before the full migration
   lands.

## Expected Outcomes

Midnight keeps its supply and contract integrity once a CRQC exists, without giving up the privacy
Zswap already has. The ledger, proof server, wallet and contract teams know the size and speed of
the replacement early enough to plan for it, and contract authors know how their verifier keys move.

## Open Questions

- Which family fits Midnight's relations and budgets best, hash based systems such as FRI based
  STARKs, lattice based systems, or another option, and at what proof size?
- Can value balance move inside the proof, so that a post quantum proof system carries both checks?
- Can a per token turnstile be enforced where shielded value becomes unshielded, so counterfeit
  shielded value is caught when it leaves the pool?
- Should the network accept both proof systems for a transition period, and how are contracts that
  never migrate treated?
- How does recursion and aggregation in `midnight-zk/aggregation` carry over?
- Which party controls the regeneration of a contract verifier key whose maintenance authority is
  itself classical, given MPS-0051?

## Recommended MIPs

- **Acceptance assumption inventory.** Publishes the map of goal 1 as a living document, so later
  MIPs can show which assumptions they remove.
- **Post quantum proof system and versioned verifier keys.** Selects the proof system, defines a
  versioned verifier key and proof format, and the activation rule. Addresses goals 2 and 4.
- **Post quantum value balance.** Replaces or supplements `pedersen_check` with a check whose binding
  is post quantum. Addresses goal 3.
- **Shielded supply turnstile.** Defines how shielded supply per token is bounded and checked.
  Addresses goal 5.

## References

- [MPS-0050, Post-Quantum Confidentiality for Zswap](/mps/mps-0050-post-quantum-confidentiality-zswap.md)
- [MPS-0051, Post-Quantum Spending Authorization for Unshielded Outputs](/mps/mps-0051-post-quantum-spending-authorization.md)
- [midnight-ledger#796, map ledger proof, balance and binding assumptions](https://github.com/midnightntwrk/midnight-ledger/issues/796)
- [midnight-zk#567, evaluate quantum resistant proof system options](https://github.com/midnightntwrk/midnight-zk/issues/567)
- [midnight-zk](https://github.com/midnightntwrk/midnight-zk) and [midnight-ledger](https://github.com/midnightntwrk/midnight-ledger)
- Kate, Zaverucha, Goldberg. *Constant-Size Commitments to Polynomials and Their Applications*. ASIACRYPT 2010.
- Engelmann, Kerber, Kohlweiss, Volkhov. *Zswap: zk-SNARK Based Non-Interactive Multi-Asset Swaps.*
  IACR ePrint 2022/1002, PoPETs 2022(4). <https://eprint.iacr.org/2022/1002>
- [NIST IR 8547, Transition to Post-Quantum Cryptography Standards](https://csrc.nist.gov/pubs/ir/8547/ipd)

## Acknowledgements

None yet.

## Copyright

This MPS is licensed under CC-BY-4.0.
