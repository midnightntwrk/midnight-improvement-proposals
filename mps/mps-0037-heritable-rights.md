---
MPS: "0037"  
Title: Heritable Rights for Self-Replicating Off-Chain Assets  
Authors: Hunter Roberts (hunterincoming)
Status: Proposed  
Category: Standards  
Created: 10-Aug-2026  
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

Existing asset standards, on Midnight and elsewhere, model assets as discrete objects
that are created once, transferred, and eventually destroyed. This holds for tokens,
NFTs, and tokenised real world assets.

It does not hold for assets that replicate. A plant cutting becomes a mother plant
becomes ten thousand clones. A breeding animal produces offspring that produce
offspring. A microbial strain is subcultured indefinitely. In each case the asset is
not a thing but a lineage, and a claim attached to an ancestor is expected to follow
its descendants — a breeder's royalty on offspring, a licence term restricting
propagation, an obligation that survives resale.

No primitive exists for this. An agreement binds the parties who signed it; it cannot
bind an instance that did not exist when it was signed. And the natural approaches
either require disclosing the lineage — which is usually the most commercially
sensitive information the holder has — or grow ledger state without bound.

This MPS identifies the absence of a standard for obligations that inherit through
descent, provable without disclosing ancestry. It defines the problem so that future
MIPs can specify a uniform representation of descent, encumbrance, and clean-descent
verification.

## Vision

A holder registers an asset and declares its parentage. An agreement attaches an
obligation that binds not only the counterparty but everything subsequently derived
from the asset. A prospective buyer, years and several generations later, asks a single
question — *is this free of upstream claims?* — and receives a yes or no.

They learn nothing else. Not which ancestors exist, not who held them, not what was
owed, not how many generations deep the lineage runs. The holder proves the negative
without surrendering the positive.

Registries in different domains — plant varieties, livestock herd books, industrial
strain collections — issue records in the same format and interoperate, because the
representation of descent is common even where the subject matter is not.

## Problem

### 1. Assets that replicate have no representation

Every asset standard assumes a bounded object: mint, transfer, burn. A cultivar that
has been propagated ten thousand times is not ten thousand independent assets, nor is
it one asset with ten thousand holders. It is a lineage in which each instance is
genuinely distinct and genuinely derived.

Modelling each instance as an independent token loses the relationship that matters.
Modelling the lineage as a single asset loses the fact that instances are held
separately and traded separately.

### 2. Obligations cannot outlive the agreement that created them

A breeder licensing a cultivar with a royalty on offspring is describing an obligation
that binds parties who have not yet been identified, over instances that do not yet
exist. Contract law handles this poorly and blockchain standards not at all.

The consequence is that the obligation is enforced socially or not at all. In cannabis
genetics — where high-THC cultivars are excluded from US plant variety protection
entirely — this is the normal state of affairs, and it is why breeders have
historically refused to participate in registries.

### 3. Verifying descent requires disclosing it

The obvious way to prove an asset is unencumbered is to publish its ancestry and show
each ancestor is clean. But the ancestry *is* the secret. A cultivar's parentage is
often the most valuable information about it, and a breeder who discloses it to
complete a sale has given away more than the sale is worth.

The EU's Essentially Derived Varieties doctrine illustrates the cost of this gap: the
legal concept exists across 27 member states, and published legal commentary describes
the difficulty of applying it, because proving derivation is the hard part.

### 4. The natural implementation grows ledger state without bound

Recording each obligation in a ledger map keyed by asset is the obvious approach, and
it scores Tier 3 on Midnight's own contract deployment rubric — unbounded growth,
cheap writes, no expiry — which blocks deployment.

This is not a theoretical concern. It is a design most implementers will reach for and
only discover is unshippable at deployment review.

### 5. Proof cost scales with generations

A lineage check spanning several generations means several inclusion or non-inclusion
proofs. Bundling them into one circuit multiplies constraint count, and in Compact —
which has no loops — every level is unrolled. Without a standard shape for this, each
implementer rediscovers the cost curve independently, and some will discover it after
building on top of the expensive version.

## Use Cases

**Use Case 1: A royalty that survives resale.** A breeder licenses a cultivar to a lab
with an 8% royalty on anything bred from it. The lab propagates and sells clones to a
third party who was never a signatory. The obligation should bind that third party's
material, and the third party should be able to discover this before purchase without
learning who the original breeder was.

**Use Case 2: Clean-descent verification at sale.** A buyer is offered a cultivar and
wants assurance it carries no upstream claim. They should receive a verifiable yes or
no. They should not learn the ancestry, the terms of any upstream agreement, or the
identity of any prior holder.

**Use Case 3: Discharge releases the line.** The royalty above is paid in full. Every
descendant should become clean, immediately, without the breeder having to identify or
contact holders they have never met.

**Use Case 4: Livestock genetics.** A bull's semen is sold with terms restricting
onward breeding. Offspring are traded internationally through herd books that already
record parentage on paper. The same descent representation should serve, without the
herd book operator implementing anything cannabis- or plant-specific.

**Use Case 5: A registrar in another domain.** An ornamental plant variety rights body
in the EU wants to issue records for its members. It should be able to define its own
subject-matter fields and reuse the descent, encumbrance and verification machinery
unchanged.

## Goals

1. **A standard representation of descent.** A common way to declare that one record
   derives from another, publicly verifiable, without requiring per-record ledger
   state.

2. **Obligations that attach to a record and bind its descendants.** Attachable by the
   holder, dischargeable by the party owed, and effective against instances created
   after attachment.

3. **Clean-descent proof without ancestry disclosure.** A prover should demonstrate
   that neither their record nor any ancestor carries an unmet obligation, disclosing
   neither the ancestors nor the depth of the lineage.

4. **Bounded ledger state.** The representation should not grow with cumulative usage.
   A registry with a million records should occupy no more ledger state than one with
   ten.

5. **Proof cost within browser reach.** Prover keys should be small enough to load in
   a browser, since the parties generating these proofs are breeders and lab
   technicians rather than infrastructure operators.

6. **Resistance to substituted ancestry.** A prover must not be able to satisfy a
   clean-descent check by naming an unrelated clean record as their parent. A
   cryptographic proof of cleanliness is insufficient on its own; the claimed ancestry
   must also be checkable.

7. **Domain independence.** The descent and encumbrance layer should carry no
   subject-matter fields, so registries in unrelated domains can adopt it without
   inheriting another domain's schema.

## Non-Goals

This MPS does not prescribe a tree structure, hash function, or proof system. It does
not define what constitutes an obligation in law, nor how obligations are priced or
settled. It does not specify subject-matter schemas for any domain. It does not address
identity or credentialing of the parties involved. The purpose is to define the problem
so that future MIPs can evaluate competing designs.

## Expected Outcomes

Assets that replicate become representable on Midnight, opening a category that current
standards exclude. Breeders and rights holders gain a mechanism for claims that survive
resale, which is the precondition for participating in any registry at all.

Registries in separate domains interoperate, because descent is common even where
subject matter is not. And implementers stop rediscovering the same two failure modes —
unbounded ledger state and prover keys too large to load.

## Open Questions

1. **Multiple parents.** Sexual reproduction produces two parents; grafting produces a
   scion and a rootstock with different rights. Should descent be a tree, a DAG, or
   typed edges?

2. **Obligation semantics on partial discharge.** If an ancestor carries two
   obligations and one is discharged, what should a descendant's clean-descent check
   report?

3. **Correction and supersession.** Records are anchored and cannot be edited, but
   parentage is sometimes recorded wrongly. How should a correction propagate to
   descendants who were created under the incorrect graph?

4. **Depth bounds.** Any fixed-depth proof imposes a maximum lineage length. What
   depth is appropriate, and how should a lineage exceeding it be handled?

5. **Collision handling.** Deriving a position from a commitment admits collisions at
   practical depths. Should positions be derived, assigned, or something else?

6. **Non-participation.** A holder who never declares parentage does not appear in the
   graph, so a clean-descent proof cannot catch them. Is this acceptable as a
   market-access filter, or does the standard need a stronger notion?

7. **Cross-registry descent.** If a record in one registry descends from a record in
   another, how is the edge represented and verified?

8. **Time-bounded obligations.** Should obligations carry expiry so terminal entries
   can be swept without holder action, and what prevents abuse of that mechanism?

## Recommended MIPs

- **MIP: Descent Declaration Format.** Specifies how a parent link is declared and
  disclosed such that the graph is reconstructable from transaction history without
  per-record ledger state.

- **MIP: Encumbrance Accumulator.** Specifies the representation of the encumbered set,
  how entries are added and cleared, and how a caller proves membership or
  non-membership without the contract trusting caller-supplied state.

- **MIP: Clean-Descent Proof Interface.** Specifies the circuit interface for proving a
  record is not downstream of an encumbered ancestor, including how proof cost scales
  with generations and how claimed ancestry is bound to declared edges.

- **MIP: Domain Profile Registration.** Specifies how a registry declares its
  subject-matter schema against a common envelope, so that domains can interoperate
  without sharing fields.

## References

- Reference implementation and design notes:
  https://github.com/hunterincoming/veilcore-midnight-testnet — includes a deployed
  Preview contract exercising anchoring, prior-possession proof and a licence
  lifecycle, and a lineage contract implementing descent declaration, encumbrance and
  clean-descent proof. Offered as evidence the problem is tractable, not as a proposed
  solution.
- Contract deployment rubric, `deployments/contract-deployment-rubric.md` — the
  state-space criteria that constrain any implementation.
- `deployments/midnightzk-anchor.md` — prior art on bounded-by-design anchoring state.
- Council Regulation (EC) 2100/94 — Community plant variety rights, including the
  Essentially Derived Varieties doctrine.

## Acknowledgements

Thanks to the Midnight Foundation and the Build Club cohort for feedback on the
problem framing, and to the authors of `midnightzk-anchor` whose deployment request
documented the bounded-state pattern this problem statement assumes.

## Copyright

This MPS is licensed under CC-BY-4.0.
