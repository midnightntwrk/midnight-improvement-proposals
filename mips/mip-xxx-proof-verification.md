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
`midnight-zk` IVC interface for generating recursive proofs, and (ii) a
`verifyProof` primitive in Compact that lets circuits verify those proofs
during execution against a verifying key fixed at circuit compile time.
## Motivation

Currently, compact does not support **in-circuit proof verification ([MPS 14](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0014-proof-verification-recursion.md)).**
Multi-step off-chain computations therefore cannot be expressed as a single
on-chain claim, and every step of an off-chain computation requires a
separate transaction.

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

### `verifyProof` in Compact

The Midnight standard library gains a proof-verification primitive:

```
export circuit verifyProof(proof, vk, public_inputs): Bool;
```

Verification happens in-circuit. The verifying key is public and fixed at
circuit compile time: it appears in the signature so the developer names
which VK the circuit verifies against, but it is not a witness that can
vary at proving time. The proof and public inputs are witnesses. The
midnight-zk aggregator folds the inner proof into the outer proof; the
transaction carries the resulting accumulator and the outer proof.

#### VK representation in-circuit

A verifying key is uniquely represented in-circuit by the hash of its `transcript_repr`  
together with the commitment to all its fixed columns. This is the canonical handle 
for reasoning about VKs inside a Compact circuit.

#### Public inputs in-circuit

Public inputs are passed as field elements and can be constrained freely using
Compact's existing type system.

#### Witness generation

The application calls midnight-zk from TypeScript to generate the inner proof.
The resulting proof bytes and the public inputs are then passed into the
Compact witness builder as opaque blobs. The VK is not passed at witness time;
it is already fixed in the compiled circuit.

### Transaction extension

In order to support proof verification in compact, we need to extend the 
transaction format. The transaction carries the **aggregator** (accumulator)
of the inner proof. At transaction validation time the node verifies the outer
proof and finalises the inner proof's deferred pairing in the same step, so
no separate decider pass is needed.

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
is the only practical path. The outer and inner decider pairings are batched
into a single pairing at validation time, reducing ledger cost.

## Path to Active

### Acceptance Criteria

- A Compact circuit can accept a proof as input and verify it during
  execution against a fixed verifying key.
- A reference example demonstrates `N` off-chain IVC rounds folded into one
  on-chain submission, using the `midnight-zk` IVC interface. 

### Implementation Plan

1. Land the `midnight-zk` IVC interface and validate recursive proof
   generation in isolation.
2. Update `zk-stdlib` to support these instructions. 
3. Add the `verifyProof` language primitive and corresponding ZKIR support.
4. Extend the ledger transaction format and verification criteria to carry
   the deferred pairing data.
5. Validate end-to-end on a devnet using a reference IVC example folding N
   off-chain rounds into a single on-chain proof.

## Open Design Questions

**Language design for `verifyProof`.** How are the proof and VK types
surfaced in Compact's type system? The VK is fixed at compile time and
public inputs are field elements, but the precise type-level interface and
any guarantees the language gives around VK pinning carry the most
language-design uncertainty in this MIP. On the TypeScript side, the exact
API by which the developer calls midnight-zk to generate the inner proof and
threads the result into the Compact witness builder is also unsettled.

**VK format specification, versioning, and interop.** The current sketch of
the in-circuit VK representation (hash of `transcript_repr` together with
commitments to the fixed columns) is thin. A formal wire-format specification
is needed, since Compact tooling, wallets, and the node all consume it and a
byte-for-byte agreement is required if pinning by hash is to remain sound.
As midnight-zk evolves the format is likely to change; whether we support
more than one active version at a time, and whether proofs produced against
one version can be verified by a circuit compiled against another, are open.

**Performance.** Is the performance of in-circuit verification in BLS12-381
good enough for the intended use cases?

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
