---
MPS: "xxxx"
Title: Witness-Supplied Merkle Paths Are Not Bound to the Leaf They Are Checked Against
Authors: Wes Huber @wbaxterh
Status: Proposed
Category: Libraries and Tooling
Created: 09-SEP-2026
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

Merkle membership is Compact's standard way to prove that a caller belongs to an
authorized set. The usual shape is: recompute a commitment from private values, ask a
witness for the path to it, and assert `tree.checkRoot(merkleTreePathRoot(path))`.

`MerkleTreePath` carries its own `leaf` field, and `merkleTreePathRoot` hashes that leaf
rather than the value the circuit computed. The witness runs unconstrained on the prover's
machine, and every leaf and path in the tree is public data. So `checkRoot` alone proves
that *some* leaf is in the tree, not that *the caller's* leaf is. Unless the circuit also
asserts `path.leaf == <recomputed value>`, any member's path satisfies any caller's check.

This is a soundness gap, not a privacy one: the contract admits callers it should reject.
The safe and unsafe forms differ by one line and are visually indistinguishable, the
compiler accepts both, and the type system cannot tell them apart because both are a
well-typed `MerkleTreePath`.

A survey of 122 `checkRoot` call sites across 63 public repositories found 40 sites in 24
repositories that recompute a value, request a witness path for it, and never bind the two.
This MPS describes the gap and its propagation. It does not select a remedy.

## Vision

A developer writing Merkle membership in Compact cannot silently omit the binding step.
Either the language makes the unbound form impossible to express, the standard library
offers a membership primitive that binds by construction, or the toolchain reports the
unbound form where it appears. Whichever route is chosen, a reviewer can determine from the
source whether a membership check is bound, and existing contracts can be audited
mechanically rather than by reading every circuit.

## Problem

### The pattern and the gap

The documented approach to proving membership is:

```compact
export circuit spend(): [] {
  const sk = secretKey();                        // witness
  const commitment = deriveCommitment(sk);       // recomputed in-circuit
  const path = getCommitmentPath(commitment);    // witness, unconstrained
  assert(tokens.checkRoot(merkleTreePathRoot<10, Bytes<32>>(disclose(path))),
         "not a member");
  // ... nullifier, effect
}
```

Three properties combine into the gap:

1. **Witness returns are unconstrained.** A witness executes on the prover's machine. Its
   return value is an input to the proof, not a fact established by it. This is the same
   root cause described in MPS-0029 for `ownPublicKey()`; here the affected value is a
   `MerkleTreePath` rather than a public key.
2. **The path carries its own leaf.** `merkleTreePathRoot` hashes `path.leaf` together with
   the siblings. It never sees `commitment`. The recomputation on line 2 and the check on
   line 4 are not connected by anything.
3. **Paths are public.** Merkle trees on Midnight are public ledger state, and every path
   in the tree can be reconstructed by any observer from that state.

Therefore a prover who is not a member can take any genuinely-inserted leaf, reconstruct its
path from public state, return it from their own witness implementation, and pass
`checkRoot`. The remedy is one assert:

```compact
  assert(path.leaf == commitment, "path is not for this commitment");
```

Nothing in the language, the type system, or the compiler indicates that the assert is
required, and both forms compile without diagnostics.

### The failure is silent in exactly the wrong direction

Omitting the binding does not break the happy path. An honest caller's witness returns their
own path, `path.leaf` equals the recomputed commitment anyway, and every test passes. The
defect is only observable when someone deliberately supplies a path for a leaf that is not
theirs, which is not a case that arises during normal development. A contract can therefore
carry the gap through a full test suite, a demo, and a deployment without a single failing
signal.

### Nullifiers do not compensate

Many affected contracts pair the membership check with a nullifier to prevent reuse. This
does not help. The nullifier is typically derived from the caller's own secret, so an
attacker supplying someone else's path still produces a fresh, unspent nullifier of their
own. The double-spend guard and the membership guard protect different properties, and only
the latter is broken here.

### The knowledge exists but does not propagate

This is not a case of undocumented behaviour. The security guide's "Restricting a circuit to
a group" section states the rule and shows the assert directly:
`assert(path.leaf == derivePublicKey(secretKey()), "path not bound to caller")`. The gap is
that the concept page most developers learn the pattern from, `keeping-data-private`, omits
the assert in both of its Merkle examples and does not link to the guide. A documentation
fix is approved and pending merge (`midnight-docs` PR #1314), but documentation alone does
not reach code that is already written or copied from an example.

The clearest evidence that this is a footgun rather than a knowledge problem is that the same
authors get it right and wrong in adjacent files. In `midnightntwrk/midnight-expert`, the
`compact-privacy-disclosure` skill ships two Merkle examples:

- `NullifierDoubleSpend.compact` binds correctly, and carries a ten-line comment explaining
  precisely why: *"Without this check a prover can return any genuinely-issued token's path
  and spend a token they never held... This is soundness, not privacy: the contract would
  admit people it should reject. The check costs 8 rows and does not change k."*
- `PrivateVoting.compact`, in the same directory, omits the binding in both `commitVote` and
  `revealVote`. Its comments explain the `disclose()` rules on the surrounding lines in
  careful detail, so the omission is clearly not inattention to correctness. It is the one
  rule with no enforcement behind it.

A third example in the same repository, `basic-start/examples/ticket.compact`, binds
correctly. Two out of three teaching examples get it right, and nothing catches the third.

### Survey

Method: GitHub code search for `checkRoot` and `merkleTreePathRoot` in `.compact` files
(2026-09-09), then per-circuit static analysis of each file. For every call site the analysis
resolves the expression passed to `merkleTreePathRoot`, traces it to its defining statement
in the same circuit, determines whether that statement calls a declared `witness`, and
searches the same circuit for a `leaf ==` binding. Sites whose origin could not be resolved
are reported separately rather than counted as unbound. Findings were confirmed by reading
the circuits.

| | Count |
|---|---|
| Repositories with at least one `checkRoot` site | 63 |
| Call sites analyzed | 122 |
| Bound (`leaf ==` present in the same circuit) | 50 |
| Witness-derived path, no binding | **40**, across **24** repositories |
| Origin unresolved or path not witness-derived, needs manual review | 32 |

Affected code is not confined to hackathon submissions. The unbound set includes voting
contracts, credential and attestation systems, salary and payment disclosure contracts, an
RWA issuer authorization check, and developer tooling published by established teams. Of the
24 repositories with an unbound site, only one also contains a bound site, so the pattern
tends to be consistent within a codebase: teams either know the rule or do not.

The survey deliberately reports aggregates. Where a specific deployed contract appears to be
affected, the maintainer is being contacted directly rather than named here.

## Use Cases

**A developer follows the concept page.** They read `keeping-data-private`, copy the
membership example, adapt it to their commitment type, and ship. Their tests pass because
their own witness returns their own path. They have written an authorization check that
admits anyone who can read the ledger.

**An auditor reviews a Compact contract.** There is no mechanical way to answer "are all
membership checks in this codebase bound?" The reviewer must read every circuit that calls
`checkRoot`, resolve each path expression to its origin, and confirm a matching assert. This
survey required writing a bespoke parser to answer that question across 63 repositories.

**A library author exposes a membership helper.** `OpenZeppelin/compact-contracts` gets this
right in `ShieldedAccessControl._validateRole`, with a comment explaining the check. That
correctness is invisible to downstream users, who cannot distinguish a library that binds
from one that does not without reading its source.

**A teaching example propagates the gap.** Example contracts are copied more often than they
are audited. An unbound example in an official skill directory becomes unbound authorization
checks in every project that starts from it.

## Goals

- Make the unbound form either impossible to express, or visible without reading the circuit.
- Preserve the ability to write the check by hand, since not every membership proof binds to
  a single recomputed leaf.
- Allow existing code to be audited mechanically, so the 32 unresolved sites in this survey
  and any future ones can be classified without bespoke tooling.
- Keep the remedy proportionate. The binding assert costs about 8 constraint rows and does
  not change the circuit's `k`, so cost is not the obstacle.
- Align the outcome with the treatment of other unconstrained-witness gaps, so that Compact
  presents one consistent story about which values a proof actually constrains.

## Expected Outcomes

Developers who follow the documented pattern get a sound authorization check by default
rather than by recalling a rule. Reviewers and auditors can determine bindingness from the
source or from a tool. The class of "membership check that admits non-members" stops
recurring in new contracts, and the existing population can be triaged. Teaching material
stops propagating the unbound form, because the unbound form either fails to compile or is
flagged where it is written.

## Open Questions

- Should the remedy live in the language, the standard library, or the toolchain? A
  binding-by-construction membership circuit, a compiler diagnostic on a witness-derived
  `MerkleTreePath` reaching `checkRoot` without a comparison of its `leaf`, and a lint are
  all plausible and are not mutually exclusive.
- Are there legitimate uses of an unbound membership check? Proving that a tree is non-empty,
  or that a given root was historic, may not need to bind a leaf. Any diagnostic needs a
  documented way to express that intent deliberately.
- Should `MerkleTreePath` returned from a witness be a distinct type from one constructed
  in-circuit, so that the type system can carry the distinction?
- How should this interact with MPS-0029? Both describe unconstrained witness values used
  for authorization. A single mechanism for "this value came from a witness and has not been
  constrained" might address both, and several other primitives besides.
- What is the disclosure and remediation path for already-deployed contracts identified as
  affected?

## Recommended MIPs

**Binding membership primitive in the Compact standard library.** A circuit such as
`checkMembership(tree, leaf, path)` that performs the leaf comparison and the root check
together, so that the sound form is the shortest form. Addresses the ergonomics half of the
problem: developers reach for the helper rather than reconstructing the pattern.

**Compiler diagnostic for unbound witness-derived paths.** A warning, or an error under a
strict mode, when a `MerkleTreePath` originating from a witness reaches `merkleTreePathRoot`
in a circuit that never compares its `leaf`. Addresses the existing population and the
copy-from-an-example path, which a new standard library circuit alone will not reach.

**Witness-provenance tracking for authorization-relevant values.** A general mechanism to
mark values that arrived from a witness and have not been constrained, applicable to
`MerkleTreePath`, `ownPublicKey()` per MPS-0029, and similar primitives. Broader and slower
than the two above, and worth considering as the eventual convergent answer.

## References

- MPS-0029, Compact Caller Identity, for the same unconstrained-witness class applied to
  `ownPublicKey()`
- Midnight docs, Guides, Security best practices, "Restricting a circuit to a group"
- Midnight docs, Concepts, "Keeping data private"
- `midnightntwrk/midnight-docs` PR #1314, which adds the binding assert to both Merkle
  examples on the concept page and links them to the security guide
- `midnightntwrk/midnight-expert`, `plugins/compact-core/skills/compact-privacy-disclosure/`,
  for the adjacent bound and unbound teaching examples
- `OpenZeppelin/compact-contracts`, `contracts/src/access/ShieldedAccessControl.compact`, for
  a correctly bound library implementation
- Survey data and the analysis script are available on request

## Acknowledgements

The bound teaching example in `midnight-expert` and the OpenZeppelin implementation both
document the reasoning behind the binding assert clearly, and informed the framing here.
Thanks to @oduameh for reviewing the corresponding documentation change.

## Copyright

This MPS is licensed under CC-BY-4.0.
