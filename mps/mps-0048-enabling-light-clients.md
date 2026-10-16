---
MPS: "0048"
Title: Enabling Light Clients on Midnight
Authors:
  - Bob Blessing-Hartley (bobblessinghartley)
Status: Proposed
Category: Core
Created: 15-Oct-2026
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

Nothing today can hold a trustworthy view of Midnight without running a full node. Joining the
network costs the whole history, every time and for every operator, and anything unwilling to pay
that falls back on taking an operator's word. Starting from a later point means being handed a
committee on faith, and nothing supplies the grounds for that faith. This document asks for the
pieces that would let something start from a recent point and go on from there: a proof of which
committee Midnight seated, the routine signing that feeds it, a way to catch a committee that signs
two incompatible things, a way to reach the present, and a route by which a recent point travels to
whoever needs it.

## Vision

Midnight's protocol carries an enablement layer that gives light clients a nearly trustless
foundation. A client checks a single object, and neither that object nor the work of checking it
grows as the chain does; an ordinary phone does it in seconds. Making the object takes no special
hardware — a machine someone already owns will do — so anyone who wants the proof to exist can
produce one, and nobody has to be paid to. Whoever builds clients, produces proofs, or serves data is
free to do it their own way. And the protocol catches double-signing, the abuse any BFT two-thirds
honest majority is sized against.

## Problem

A node with no history reaches the present by obtaining every block from genesis and executing each
one. The work is the chain's whole history, and it grows with the chain rather than with what the
node needs. Joining therefore costs the whole history every time, for every operator, and the cost
rises with every block Midnight produces.

Starting anywhere other than genesis means trusting a later state without having derived it, so
whoever starts there is given something they take on faith. Two things make such a start safe: the
state at the point of entry, so that execution can go forward from it rather than replay toward it;
and the committee at that point, so that what is offered as the chain can be told from what is not.
The first of these Midnight already produces. The second it does not.

So nothing can hold a trustworthy view of Midnight without running a full node. What is missing is
not evidence that a committee was seated, which the signatures already carry, but the pieces that
would let something start from a recent point and go on from there.

### Why this is needed now

The cost rises with every block, falls on everyone who joins, and is paid again by each of them.
Nothing about it improves by waiting, and what the protocol already signs is most of what a proof
would need.

### Relationship to existing proposals

[MIP-0023: Committee Bridge Consensus Integration](../mips/mip-0023-committee-bridge-consensus.md)
(Proposed) specifies a committee induction for one verifier: a light-client contract on Cardano that
tracks the Midnight BEEFY committee session by session, each committee vouching for its successor
through committee-signed MMR roots, starting from `beefy_activation_block`. Once it is active,
Midnight will produce signed committee handovers. This MPS does not replace it. It states the problem
for verifiers that are not that contract, and sets requirements MIP-0023 deliberately does not meet:

- MIP-0023 verifies one signature list per handover, on Cardano, in sequence. Its committee size is
  capped by `signer_cap` so that a quorum fits one Cardano transaction. This MPS asks for a single
  object a client checks without walking the handovers, at committee sizes well beyond that cap.
  MIP-0023 itself names a succinct quorum proof as "the urgent item after this MIP".
- MIP-0023 is built on secp256k1 ECDSA and Keccak-256 so that Cardano can verify it cheaply. This
  MPS asks for security against a quantum adversary and, where that conflicts with Cardano
  verifiability, keeps the former. A construction answering this MPS may therefore need signatures
  or commitments that differ from MIP-0023's.
- MIP-0023's signed payload is the MMR root alone. This MPS asks for a signed root covering the whole
  post-execution state, which is what bootstrapping the state machine takes.
- MIP-0023 makes equivocation provable and attributable through BEEFY's `DoubleVotingProof` and
  `ForkVotingProof`, but notes that equivocation reporting is a no-op in the runtime's `BeefyApi`,
  that there is no ban list, and that a reporting extrinsic and ban list are future work for a
  separate MIP. This MPS asks for that MIP.
- MIP-0023 depends on a contract, a funding pool and a data pump on Cardano. This MPS asks that a
  client elsewhere need nothing deployed, funded or kept current on any chain.

[MPS-0032: History Management for Midnight](mps-0032-storage-management.md) includes, as UC3, a
light client or new validator bootstrapping into current consensus state without syncing full
history. This MPS is a narrower statement of that use case, concerned with how a joining party
learns which committee to trust and how it reaches the present; MPS-0032 remains the home for
pruning, archival, nullifier and commitment integrity, and historical queries.

## Use Cases

**UC1: Light client bootstrap**

* **Scenario:** A light client wants a Midnight state to start from. It obtains a tip — the highest
  block whose finality the committee has signed off on, together with that block's signed state root —
  and it needs to know the committee that signed it was the one Midnight seated.
* **Limitations:** Nothing today tells the client that. It must run a full node or trust whoever
  handed it the tip.
* **Desired Outcome:** A proof, checked against the activation committee and nothing else, tells the
  client the committee was the one Midnight seated. The client then executes forward from the tip.

**UC2: Joining node**

* **Scenario:** A node joining the network wants to participate without replaying the chain.
* **Limitations:** Joining requires obtaining and executing every block from genesis.
* **Desired Outcome:** The node starts the same way as UC1 and keeps going as a node.

**UC3: Reaching the present**

* **Scenario:** A tip is behind the chain by however long it took to publish it, so a client that
  has checked a proof against one still has the blocks since to account for.
* **Limitations:** There is no way to cover those blocks without replaying them as a full node does.
* **Desired Outcome:** From the tip it has checked, the client reaches the present without replaying
  the chain and without running a full node.

**UC4: Obtaining a tip**

* **Scenario:** A client needs a tip to start from.
* **Limitations:** Nobody is appointed to offer tips, no such party is part of the protocol, and none
  exists today.
* **Desired Outcome:** A client obtains a tip without any dedicated source having to exist. Where tips
  come from is one of the things this document asks someone to propose.

**UC5: Committee fact without a root**

* **Scenario:** A party elsewhere — a contract on another chain, or a bridge such as the one
  [MPS-0033](mps-0033-mnight-to-cnight.md) calls for — wants to know which committee Midnight seated
  at an epoch, and does not need a state root.
* **Limitations:** Today it must follow the chain, or rely on a contract that does.
* **Desired Outcome:** It obtains the same proof a light client uses and checks it the same way.

## Goals

At each handover a committee signs a manifest, naming the committee for the next epoch with each
member's weight. Every committee after the first is named this way in a block the committee before
it finalized, so knowing the first and the signatures at each handover determines all of them.
Learning the committee at a far epoch is therefore an induction, and an induction is the kind of
thing a proof can carry whole. A single proof of that induction says what following every handover
would say, and a verifier checks it without holding the chain, without the handovers, and without
being on Cardano or anywhere in particular. What it replaces is the walk, not the trust.

1. **A client checks against the activation committee and nothing else.** No chain to follow, no
   state held anywhere, and no contract on any particular chain standing between it and the answer.
   The activation committee is the committee seated when this scheme begins, which signs the first
   manifest; it is not the committee fixed at genesis, which signed no manifest because there was
   none to sign. Its set is a published constant of the network, held by a client the way the chain
   it is on is held, and not learned from whoever is talking to it.

2. **What a client receives, and the work of checking it, are the same at a far epoch as at a near
   one**, and neither grows as the chain runs. Checking is within an ordinary mobile device, in
   seconds.

3. **Every block's header carries a signed root**, produced as a matter of course rather than on
   request, covering the post-execution state entire: the ledger storage held across every pallet,
   and not midnight-ledger's alone, which it is easy to mistake for the whole. Bootstrapping the
   state machine takes all of it.

4. **What the proof rests on besides the committee is accounted for** by the proposal that implements
   this: the setup the proof system requires and the discarding nothing attests, the circuit
   expressing what was meant, and the code that checks the proof.

5. **The construction is secure against a quantum adversary.** The same object a client checks can be
   checked by a contract on Cardano; where that and quantum security pull against each other, the
   second is kept and the first given up.

6. **Every epoch's committee is reachable, in order, from the activation committee.** A proof covering
   a later epoch covers every one before it, so no epoch is missing from the succession a client is
   shown. It holds at committee sizes well beyond what Midnight seats today. Reaching those sizes is
   Midnight's own business and not an answer's, but an answer that only works at today's sizes does
   not serve.

7. **What a client does to satisfy itself that a given root was signed by the committee Midnight
   seated does not grow with the number of signatures behind it.** Which committee was seated, and
   whether that committee signed a given root, are two questions, and a client needs both answered
   without counting signatures. One construction may answer both, or one may answer each; that choice
   belongs to whoever writes the proposals. What is not open is that both get answered, or that a
   proof of the committee alone stays obtainable and checkable on its own — use cases other than a
   light client want that fact and not a root, and a packaging chosen for the client's sake must not
   cost them.

8. **Producing the signature is part of running a node**, for governance-selected and openly
   registered validators alike, so that failing to produce it is the same kind of failure as failing
   a node's other duties. Anything the system must do on every epoch, forever, is done by software
   participants are likely to run continually as a consequence of running a node, and never by
   anyone's initiative. No participant is asked to hold funds, to spend them, or to give unattended
   software access to them; what the protocol asks of a participant is a signature.

9. **Detecting and reporting equivocation is ordinary node operation.** A client satisfies itself
   about what it is shown by checking signatures against a committee, which it can do because two
   thirds of that committee are assumed honest. That threshold is not arbitrary: any BFT system
   arrives at two thirds because a dishonest member can equivocate — sign two incompatible things and
   show each to a different party. Two thirds is the smallest quorum for which any two quorums share
   an honest member, who having voted once will not vote the other way, so the conflicting quorum
   never forms; and it is small enough that a quorum can still be reached when the remaining third is
   silent. Adopting the number without catching the equivocation it is sized against leaves whoever
   signed both unidentified and still seated, free to do it again, and the client shown the wrong
   branch never learns it was. Detection and reporting are therefore carried out by the software an
   operator already runs rather than by a separate tool or anyone's initiative, and the evidence
   convicts without anyone trusting whoever carries it. What follows is the banning of the offending
   keys; nothing is seized. Midnight does not do this today.

10. **Computing the proof is open to anyone and reserved to no one.** No party is appointed to it,
    none is privileged in it, and a client accepts a proof on what it contains rather than on who
    produced it. Any prover can continue any valid proof another produced, holding nothing private to
    its producer; otherwise every prover begins at the activation committee, the same work is
    repeated for as long as the chain runs, and the cost of becoming one rises without bound.
    Producing the proof has to be within hardware a party already owns for other reasons, and has to
    keep pace with the epoch rate. Nobody is paid to do it, so only what is cheap enough to be worth
    doing will be done.

11. **From the tip it has checked, a client can reach the present** without replaying the chain and
    without running a full node.

An answer having these serves a light client and anything else that wants the same fact; one missing
any of them serves neither well.

## Non-Goals

**No proposal asked for here builds a client**, and none settles what a client is for or who writes
one. What a client does once it holds a trustworthy view, who computes proofs and what they are paid,
and what any flow carrying value decides are all outside this, and each belongs to whoever proposes
that flow.

Nobody is expected to prove and nobody is paid to. Everyone with a client gains when someone does,
little is gained by many doing it at once, and this proposal builds no reward for it. That is a
choice rather than an oversight, and what makes it tolerable is that nothing stops when the proving
does: a joining node starts from an older point or syncs the ordinary way, and a client elsewhere
acts on an older committee or waits, so the absence costs time rather than correctness.

## Expected Outcomes

* A node joins by checking one proof and one signed root and then executing forward, instead of
  replaying years of blocks.
* Anything elsewhere that wants to know which committee Midnight seated at an epoch obtains the same
  object and checks it the same way, with nothing standing between it and the answer that has to be
  deployed, funded, or kept current.
* A committee member that signs two incompatible things is identified and its keys are banned.

## Open Questions

**Long-range attack.** Long-range attack is a known risk for light clients of a proof-of-stake chain,
and nothing here escapes it. A committee past its epoch has nothing left to lose by its keys escaping;
whoever obtains them can sign a successor nobody seated and choose every committee after it, and that
branch satisfies the same rule the proof checks, so a proof of it is sound. Only committees from the
activation onward are exposed, since no earlier one signed a manifest to forge against.

*A non-normative sketch.* A long-range forgery cannot fool anyone who watched the history happen. An
actor tracking Midnight continuously knows which derivation is the real one and, unless compromised
itself, will offer only that one. A client could take the current committee from several such actors
picked in advance: its wallet provider, an exchange, a node of its own. There is one right answer, so
if they disagree with each other, or with a proof the client has been handed, something is wrong and
nothing the client holds says what it is; the client stops and puts it to the user rather than
deciding for itself.

The more such actors there are, and the more of them a client checks, the harder the attack becomes:
the retired keys are no longer enough, and every one of those actors has to be compromised as well.
What is trusted of them is narrow. The proof they serve is verifiable either way, so what they are
trusted for is having weeded out the false histories. Gossip will not do, because a client relying on
it is open to an eclipse attack: an adversary that comes to occupy all of a client's connections
chooses everything that client hears. Actors picked in advance are not reached that way. Who would run
them is open.

**Retroactive hard fork.** If Midnight were forced to restart a stalled chain by a retroactive hard
fork the community supported, a finalized block would be replaced by one from which a different
committee is then chosen in the ordinary way. That would not break the induction — the new committee
is named by a predecessor exactly as any other is — but it would leave a second history, since the
block replaced was finalized and so were the blocks after it on the original basis. Two branches would
exist whose committees are each organically named and each provable. The abandoned branch stops
growing at the fork and the live one does not, which is the only thing distinguishing them and is not
something the proof says. MIP-0023 names a related case: a BEEFY restart by `set_new_genesis`, which
starts voting again from a new genesis block.

*A second non-normative sketch, and a costly one.* If a block were not treated as truly final until
the epoch after its own had run, a stall would be discovered while neither epoch was final, the rewind
would replace nothing a client could have been shown as settled, and the second history would not
arise. What that costs is a full epoch of added finality latency, paid by everything else that depends
on finality, and it runs directly against the bounded, fast hard finality that
[MPS-0044](mps-0044-consensus-performance.md) asks for. Whether the trade is worth making has not been
examined.

Proposals addressing either problem are welcome.

**Committee selection and epoch structure.** The Goals assume a committee that is named, with
weights, at each handover. [MIP-0010](../mips/mip-0010-aura-to-babe-migration.md) changes how
committees are staged ahead of each epoch, and [MPS-0030](mps-0030-consensus-decentralization.md)
leaves open whether committees are known or chosen by anonymous sortition. How the requirements here
carry over to either change is open.

## Recommended MIPs

1. **Succinct committee and root proof.** The construction, or constructions, that let a client
   satisfy itself about a committee and the root it signed without counting signatures, and what they
   rest on. Addresses Goals 1, 2, 4, 5, 6, 7 and 10. Relates to the succinct quorum proof MIP-0023
   names as its follow-up, subject to the quantum requirement in Goal 5.
2. **Whole-state signing duty.** The signing duty that gives it its input, carried in every block's
   header and covering the whole of a block's state. Addresses Goals 3 and 8.
3. **Equivocation reporting and key banning.** Equivocation reporting, which Midnight does not do
   today, turned on and made ordinary node operation, with offending keys banned. Addresses Goal 9.
   This is the separate MIP that MIP-0023 defers.
4. **Reaching the present.** A way for a client to reach the present from the tip it has checked.
   Addresses Goal 11 and UC3.
5. **Tip gossip.** An extension to the node network's gossip so that tips travel it, so that a client
   can obtain one without any dedicated source having to exist. Addresses UC4. Such an extension does
   not by itself resist eclipse attack; see Open Questions.

## References

* [MIP-0023: Committee Bridge Consensus Integration](../mips/mip-0023-committee-bridge-consensus.md)
* [MPS-0032: History Management for Midnight](mps-0032-storage-management.md)
* [MPS-0033: mNIGHT to cNIGHT](mps-0033-mnight-to-cnight.md)
* [MPS-0044: ZK-Proof Verification Throughput Bottleneck](mps-0044-consensus-performance.md)
* [MPS-0030: Decentralization Through Large-Scale Validator Participation](mps-0030-consensus-decentralization.md)
* [MIP-0010: Consensus Migration from Aura to BABE](../mips/mip-0010-aura-to-babe-migration.md)

## Acknowledgements

TBD.

## Copyright

This MPS is licensed under CC-BY-4.0.
