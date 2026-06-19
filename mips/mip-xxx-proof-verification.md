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

Compact has no primitive that verifies a proof inside a circuit. Developers
who want to build IVC-style applications, where multiple off-chain computation
steps are folded into a single on-chain claim, cannot verify an aggregated
proof on Midnight. Each step in the computation requires a separate on-chain
transaction, limiting the kinds of applications that can be built.

This MIP introduces two capabilities to the Midnight stack: (i) a
`midnight-zk` IVC interface for generating recursive proofs, and (ii) a
`verifyProof` primitive in Compact that lets circuits verify those proofs
during execution. Together they close the gap.

The recursive statement is expressed at the `midnight-zk` level in this
milestone rather than directly in Compact, deferring the language and
compiler work for Compact-level recursion to a later milestone.

This MIP solves the problem statement described in [MPS-14](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0014-proof-verification-recursion.md).

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
  of a valid inner proof w.r.t. some verifying key from a given set, without 
  revealing which one.

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
- The **step function** that advances the state by one round, both in-circuit
  and off-circuit.

From these, `midnight-zk` derives the recursive circuit, instantiates the
prover and verifier, and handles the decider that finalises verification at 
the end of the chain.

For experienced users who need more control, it will also be possible to
build **arbitrary proof-carrying data schemes**, not constrained to a linear
IVC chain, composing recursive statements over arbitrary DAG topologies.

### `verifyProof` in Compact

The Midnight standard library gains a `verifyProof` function:

```
export circuit verifyProof(proof, vk, public_inputs): Bool;
```

The function runs the PLONK verifier in-circuit and returns a boolean. The
BLS12-381 decider (pairing evaluation) is deferred off-circuit and carried as
an extension to the transaction structure, to be evaluated by the node.

## Rationale

### Why express recursion in `midnight-zk` rather than Compact

Writing the recursive statement in `midnight-zk` rather than Compact is the
pragmatic path to a first working version. It avoids the language design and
compiler work that Compact-level recursion would require.

### Why defer the decider off-circuit

The BLS12-381 pairing required to finalise an in-circuit proof verification
is prohibitively expensive in-circuit. Carrying it as a transaction extension
is the only practical path. Supporting a more advanced scheme where the
in-circuit verification delegates some further work to the decider (making the
recursive prover cheaper) would be desirable, though a complete solution can 
be delivered without it.

## Path to Active

### Acceptance Criteria

- A Compact circuit can accept a proof as input and verify it during
  execution.
- A reference example demonstrates `N` off-chain IVC rounds folded into one
  on-chain submission, using the `midnight-zk` IVC interface.

### Implementation Plan

1. Land the `midnight-zk` IVC interface and validate recursive proof
   generation in isolation.
2. Add BLS12-381 G1 native operations in Compact as a prerequisite for
   in-circuit verification.
3. Add the `verifyProof` language primitive and corresponding ZKIR support.
4. Extend the ledger transaction format and verification criteria to carry
   the deferred pairing data.
5. Validate end-to-end on a devnet using a reference IVC example folding N
   off-chain rounds into a single on-chain proof.

## Backwards Compatibility Assessment

Hard-fork required. The transaction structure changes to carry deferred
pairing data, so blocks produced under the new rules are not validatable by
un-upgraded nodes. 

## Implementation

The work touches `midnight-zk`, the Compact compiler and language, ZKIR, the
ledger, and the proof server.

**`midnight-zk`:**

- New IVC module exposing the `Ivc` trait.
- Improvements to the performance of recursion.

**Compact language and compiler:**

- New `verifyProof` operation, including new witness types for the proof and
  the inner VK.
- Type-sync mechanism between Compact and `midnight-zk` so the in-circuit
  verification interface is callable directly from Compact. Will require type
  extensions and new ZKIR instructions.

**Ledger and verifier:**

- Verification criteria and transaction structure updated to evaluate the 
  deferred check.

## Open Questions

**Exact transaction extension shape.** Which fields are needed to carry the
additional data needed to verify new proofs? This determines the precise
wire-format change and the ledger verification logic.

**Language design for `verifyProof`.** How are proof and VK witness types
surfaced in Compact's type system?

**Performance.** Is the performance of recursion in BLS12-381 good enough
for the intended use cases?

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
