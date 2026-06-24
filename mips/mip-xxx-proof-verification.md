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

```yaml
MIP: xxxx
Title: Proof Verification in Compact
Authors:
  - Iñigo Querejeta Azurmendi
Status: Draft
Category: Core
Created: 2026-05-15
Requires: none
Replaces: none
License: Apache-2.0
```

# Proof Verification in Compact

## Abstract

This MIP introduces two capabilities to the Midnight stack: (i) a
`midnight-zk` IVC interface for generating recursive proofs, and (ii) two
`verifyProof` primitives in Compact that let circuits verify those proofs
during execution - one that keeps the proof, VK, and public inputs private
(`verifyProofHidden`), and one that exposes them (`verifyProofExposed`).
Together they close the gap.
## Motivation

**No in-circuit proof verification ([MPS 14](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0014-proof-verification-recursion.md)).**
Compact provides no operation that verifies a proof inside a circuit.
Multi-step off-chain computations therefore cannot be expressed as a single
on-chain claim, and every step of an off-chain computation requires a
separate transaction.

Some use cases drive the gap:

- **Multi-round off-chain computation:** a rollup (e.g. a game) runs N
  rounds off-chain, each producing a proof; only the final accumulated proof
  hits the chain.
- **Identity and compliance composition:** an outer proof verifies knowledge
  of a valid inner proof without revealing the inner proof's content.

## Specification

### `midnight-zk` IVC interface

A new `midnight-zk` interface exposes recursive proof generation while keeping
the IVC machinery (folding, accumulator handling, decider) abstracted away
from the application developer. For the common case, the developer specifies
only three things:

- The **genesis state** of the recursive computation.
- The **in-circuit and off-circuit representation** of the state, i.e. how
  it is assigned and constrained inside the circuit, and how it is computed
  natively.
- The **step function** that advances the state by one round, both in-circuit and off-circuit.

From these, `midnight-zk` derives the recursive circuit and instantiates the
prover and verifier.

For experienced users who need more control, it will also be possible to
build **arbitrary proof-carrying data schemes**, not constrained to a linear
IVC chain, composing recursive statements over arbitrary DAG topologies.

### `verifyProof` variants in Compact

The Midnight standard library gains two proof-verification primitives:

```
export circuit verifyProofHidden(proof, vk, public_inputs): Bool;
export circuit verifyProofExposed(proof, vk, public_inputs): Bool;
```

They differ in where verification happens and what appears in the transaction.

**`verifyProofHidden`** - verification happens in-circuit. All three arguments
are private witnesses; the inner proof, VK, and public inputs never appear in
the transaction. The midnight-zk aggregator folds the inner proof into the
outer proof; the transaction carries only the resulting inner proof's accumulator
and the outer proof. This variant is cheaper on the ledger but more expensive to
generate, as the PLONK verifier runs inside the circuit.

**`verifyProofExposed`** - verification happens off-circuit. All three
arguments are public; the proof, VK, and public inputs appear in the
transaction and the node verifies them directly. No in-circuit verification
work is required, making this variant cheaper to generate but more expensive
on the ledger.

#### VK representation in-circuit

A verifying key is represented in-circuit by its `transcript_repr` - a hash
of all VK components computed by midnight-zk. This is the canonical handle 
for reasoning about VKs inside a Compact circuit.

#### Public inputs in-circuit

Public inputs are passed as field elements and can be constrained freely using
Compact's existing type system.
#### Witness generation

The application calls midnight-zk from TypeScript to generate the inner proof.
The resulting proof bytes, the `transcript_repr` of the VK, and the public
inputs are then passed into the Compact witness builder as opaque blobs.
### Transaction extension

The transaction extension format differs between the two variants:

- `verifyProofHidden`: the transaction carries the **aggregator** (accumulator)
  of the inner proof.  The node verifies the outer proof and runs the decider of
  the inner proof.
- `verifyProofExposed`: the transaction carries thje **full inner proof**, the VK, 
  and the public inputs. The node verifies both proofs, inner and outer, directly.

The encoding details are abstracted from the ledger and owned by midnight-zk.
The node verifies the transaction without needing to know how many proofs are
present.

**Design goal.** The multi-proof transaction extension is designed to
generalise. If transaction semantics change in the future - for example,
multiple `verifyProof` calls in a single transaction — the same abstraction
layer should accommodate the change without ledger-level modifications.

## Rationale

### Why express recursion in `midnight-zk` rather than Compact

Writing the recursive statement in `midnight-zk` rather than Compact is the
pragmatic path to a first working version. It avoids the language design and
compiler work that Compact-level recursion would require.

### Why defer the decider off-circuit

The BLS12-381 pairing required to finalise an in-circuit proof verification
is prohibitively expensive in-circuit. Carrying it as a transaction extension
is the only practical path. When `verifyProofHidden` is used, the outer and
inner decider pairings can be batched into a single pairing, reducing ledger 
cost.

## Path to Active

### Acceptance Criteria

- A Compact circuit can accept a proof as input and verify it during
  execution.
- A reference example demonstrates `N` off-chain IVC rounds folded into one
  on-chain submission, using the `midnight-zk` IVC interface.

### Implementation Plan

1. Land the `midnight-zk` IVC interface and validate recursive proof
   generation in isolation.
2. Update `zk-stdlib` to support these instructions.
3. Add the `verifyProofHidden` and `verifyProofExposed` language primitives and corresponding ZKIR support.
4. Extend the ledger transaction format and verification criteria to carry
   the deferred pairing data.
5. Validate end-to-end on a devnet using a reference IVC example folding N
   off-chain rounds into a single on-chain proof.

## Implementation

The work touches `midnight-zk`, the Compact compiler and language, ZKIR, the
ledger, and the proof server.

**`midnight-zk`:**

- New IVC module exposing the `Ivc` trait, `setup`, `IvcProver`, and
  `IvcVerifier`.
- Extend `zk-stdlib` to support verification of IVC proofs. 
- Improvements to the performance of recursion.

**Compact language and compiler:**

- New `verifyProofHidden` and `verifyProofExposed` operations.
- VKs represented in-circuit via their `transcript_repr` hash; public inputs
  as field elements.
- Type-sync mechanism between Compact and `midnight-zk` so the in-circuit
  verification interface is callable directly from Compact. May require type
  extensions and new ZKIR instructions.

**Ledger and verifier:**

- Transaction extension carries either an accumulator (`verifyProofHidden`) or
  a full proof (`verifyProofExposed`); encoding owned by midnight-zk.
- Verification criteria updated to evaluate the deferred check.

## Open Design Questions

**Language design for `verifyProofHidden` and `verifyProofExposed`.** How are
proof and VK witness types surfaced in Compact's type system? The VK is handled
via `transcript_repr` and public inputs as field elements, but the precise
type-level interface and any guarantees the language gives around VK pinning
carry the most language-design uncertainty in this MIP. On the TypeScript side,
the exact API by which the developer calls midnight-zk to generate the inner
proof and threads the result into the Compact witness builder is also
unsettled.

**Performance.** Is the performance of in-circuit verification in BLS12-381
good enough for the intended use cases?

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
