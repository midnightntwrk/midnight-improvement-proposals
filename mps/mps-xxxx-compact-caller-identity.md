---
MPS: xxxx
Title: Caller Identity Access in Compact Circuits
Authors: Alejandro Pestchanker (apestchanker)
Status: Proposed
Category: Core
Created: 25-Jun-2026
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

Compact, Midnight's smart contract language, currently provides no mechanism for a circuit to read the cryptographically-enforced identity of the transaction sender. The existing `ownPublicKey()` primitive is a witness function whose return value is free-form data supplied by the caller at proving time — it is not constrained by the ZK proof and is not verified by the on-chain verifier. This creates a critical access-control gap: any contract that uses `ownPublicKey()` for authorization is trivially bypassed by an attacker who supplies an arbitrary key as their witness. The Midnight ledger already computes a cryptographically-enforced `caller` identity (derived from unshielded UTXO input ownership, verified by Schnorr signatures at the protocol level) and stores it in `CallContext`, but Compact provides no stdlib entry point to read it. This MPS describes the gap, its security consequences, and the need for a `kernel.caller()` primitive — or equivalent mechanism — that exposes authenticated caller identity to Compact circuits.

## Vision

A Midnight where Compact contracts can reliably gate actions to specific, cryptographically-authenticated wallet identities. Developers should be able to write `assert(kernel.caller() == stored_admin, "Unauthorized")` with confidence that no attacker can forge the caller identity, just as Ethereum developers rely on `msg.sender`. This capability is foundational for any contract implementing role-based access control, ownership patterns, subscription gating, DID registry management, or multi-party permissioning — all of which are currently impossible to implement securely in Compact alone.

## Problem

### `ownPublicKey()` is not cryptographically bound

The only existing mechanism in Compact for a circuit to learn "who is calling" is `ownPublicKey()`, declared in the compiler as:

```scheme
(declare-native-entry witness ownPublicKey
  "__compactRuntime.ownPublicKey"
  ()
  (TypeRef ZswapCoinPublicKey))
```

The `witness` declaration means this value is supplied externally at proving time and is never constrained inside the ZK circuit. The ZKIR pass for `ownPublicKey` explicitly registers it with `(assert cannot-happen)` — it does not emit any circuit gates. At runtime, the value is simply read from `circuitContext.currentZswapLocalState.coinPublicKey`, which is a plain parameter passed to `createCircuitContext(...)` by JavaScript code. The on-chain `zk_verify` for contract calls takes public inputs `(segment_id, parent.binding_input)` only — `coinPublicKey` is not among them.

Consequence: there is no mechanism in the ZK proof system or the on-chain verifier to distinguish a legitimately-derived `ownPublicKey()` value from an arbitrary one supplied by any caller. A contract using `ownPublicKey()` for access control cannot rely on it as a protocol-level guarantee.

### The authenticated `caller` field exists but is inaccessible

The Midnight ledger does compute a cryptographically-enforced caller identity:

```rust
// midnightntwrk/midnight-ledger: onchain-runtime/src/context.rs
pub struct CallContext<D: DB> {
    pub caller: Option<PublicAddress>,
    ...
}
```

This field is populated at the protocol level by inspecting unshielded UTXO inputs in the transaction:

```rust
// ledger/src/structure.rs
let caller = intent.guaranteed_unshielded_offer.iter()
    .chain(intent.fallible_unshielded_offer.iter())
    .flat_map(|o| o.inputs.iter())
    .map(|i| i.owner.clone())
    ...
```

Spending an unshielded UTXO requires a valid Schnorr signature from the corresponding secret key, verified at the ledger level before the circuit runs. The security guarantee is genuine and already enforced.

This value is passed to the on-chain VM at context index 6 (`CONTEXT_IDX_CALLER`) but Compact provides no stdlib entry point to read it. A comprehensive search of `LFDT-Minokawa/compact` (compiler, stdlib, runtime, documentation) found zero matches for "caller", "PublicAddress", or any function that reads context slot 6. The infrastructure is present; the language API is missing.

### Consequences for real applications

Any Compact contract that needs to restrict actions to specific wallets — including admin registries, DID controllers, token vaults, subscription gates, DAO governance, and multi-sig schemes — faces a choice between:

1. **Using `ownPublicKey()` and accepting that it is not cryptographically enforced**, or
2. **Building complex workarounds** such as coin-spend-based capability tokens, which require every authenticated action to spend and re-mint a coin — adding protocol overhead, coin management complexity, and user friction that are entirely absent from the underlying security requirement.

Neither option is acceptable for production applications handling real user data or assets.

## Use Cases

**DID Registry Access Control.** A W3C-compatible DID registry contract stores DIDs linked to their controller wallet. The contract must ensure that only the controller wallet can update or revoke their DID. When building such a registry on Midnight, the natural approach is to use `ownPublicKey()` to identify the caller and bind it to the stored controller. However, because `ownPublicKey()` is unconstrained witness data, this pattern cannot provide genuine access control — the controller check becomes a convention rather than a protocol guarantee. What is needed is a way to assert `caller == stored_controller` with genuine cryptographic enforcement, so that DID ownership is tied to actual wallet key possession.

**Admin Bootstrap Security.** Contracts commonly implement a one-time admin registration pattern: the first caller after deployment becomes the admin. Building this securely requires that the identity of the "first caller" is cryptographically determined by the protocol, not declared by the caller themselves. With `kernel.caller()`, admin registration would be bound to an unshielded wallet address verified by Schnorr signature, making the bootstrap trustworthy even against mempool-observing attackers.

**Subscription and Capability Gating.** Subscription-based dApps need to gate premium features to paying users. The current workaround requires implementing gating via coin-spend proofs — users must spend and re-mint tokens on every authenticated action. While functional, this adds protocol overhead, coin management complexity, and user friction that would be entirely unnecessary if a simple `assert(is_subscriber(caller))` pattern were available.

**Role-Based Access Control.** Standard RBAC patterns (admin, issuer, user roles as used in identity registries, token vaults, and DAO contracts) require a trustworthy caller identity as their foundation. Without it, developers must route around the gap with coin-based capability tokens for every role-gated operation, which is workable but significantly increases contract complexity and user friction.

**Multi-Party Contracts.** Contracts requiring agreement from multiple specific parties (escrow, multi-sig, voting) cannot verify that a specific party is the actual caller without a cryptographically-enforced identity primitive.

## Goals

1. Expose the ledger-level `caller` identity to Compact circuits via a new stdlib primitive (e.g., `kernel.caller()`) that returns the authenticated `PublicAddress` or `None` if no unshielded inputs are present.

2. Ensure the returned value is cryptographically bound — i.e., it reflects what the on-chain verifier has already authenticated via Schnorr signature verification, not a witness value the caller can freely choose.

3. Support both the "unshielded caller" case (deterministic `PublicAddress` from UTXO inputs) and the "no caller" case (transaction with shielded inputs only), returning `Option<PublicAddress>` or equivalent.

4. Consider a parallel mechanism for shielded caller identity — potentially via a ZK proof-of-key-ownership that a caller includes in the transaction and which the contract can verify, without requiring an unshielded UTXO spend.

5. Provide clear documentation on the security model of `ownPublicKey()` vs `kernel.caller()` so existing contracts can be audited and migrated.

6. Deprecate the current pattern of using `ownPublicKey()` for access control with appropriate guidance.

## Expected Outcomes

Compact developers will have a trustworthy mechanism for caller identity that matches the security expectations they bring from other smart contract platforms. The common patterns of contract ownership, role-based access control, and permissioned registries will be implementable without complex workarounds. Developers currently routing around the gap with coin-spend-based capability tokens will be able to simplify their designs significantly. The ecosystem of Midnight dApps will be meaningfully more robust and accessible to developers building identity, access control, and permissioning systems.

## Open Questions

1. **Shielded caller identity:** Can a shielded equivalent be designed — i.e., a way for a wallet to prove it controls a specific `ZswapCoinPublicKey` without requiring an unshielded UTXO spend? One candidate: require a coin spend (`createZswapInput`) from a coin whose commitment is stored in the contract, but this binds identity to a specific coin rather than a persistent wallet key.

2. **API surface:** Should the primitive be `kernel.caller()` (returning `Option<PublicAddress>`) consistent with the existing `kernel.*` namespace, or should it live in a different namespace?

3. **Caller vs. initiator:** The current `caller` derivation returns `None` if unshielded UTXO inputs have different owners. Should there be a separate primitive for "the initiating wallet" that relaxes this constraint, accepting any single unshielded input owner?

4. **Migration guidance:** Should the compiler emit a warning when `ownPublicKey()` is used in a context that appears to be access-control (e.g., compared against a ledger value with `assert`)?

5. **Interaction with MPS-0015:** MPS-0015 (Agent Identity Gap, by Zidan/mzf11125) proposes agent identity standards on Midnight. A `kernel.caller()` primitive would directly enable secure agent identity registration without coin-spend workarounds, and would complement the goals described in that proposal.

## Recommended MIPs

**MIP: `kernel.caller()` Compact Stdlib Primitive.**
Add a new native stdlib entry `kernel.caller(): Option<PublicAddress>` (or equivalent type) that reads `CONTEXT_IDX_CALLER` (VM context slot 6) from within a Compact circuit. The value must be the same `caller` field already computed and passed by the on-chain runtime in `CallContext`, ensuring it is derived from verified unshielded UTXO input ownership rather than from witness data. The MIP should specify the return type, behavior when no unshielded inputs are present, documentation requirements, and any deprecation notices for `ownPublicKey()`-only access control patterns.

**MIP: Security Documentation and `ownPublicKey()` Warning.**
Publish clear documentation distinguishing `ownPublicKey()` (witness data, not cryptographically bound) from `kernel.caller()` (protocol-enforced). Add an optional compiler diagnostic warning when `ownPublicKey()` is used in access-control patterns.

## References

- Midnight Compact compiler source — native stdlib declarations: https://github.com/LFDT-Minokawa/compact/blob/main/compiler/midnight-natives.ss
- Midnight Compact runtime — `ownPublicKey` implementation: https://github.com/LFDT-Minokawa/compact/blob/main/runtime/src/zswap.ts
- Midnight Ledger — `CallContext` struct and `caller` derivation: https://github.com/midnightntwrk/midnight-ledger/blob/ledger-8/onchain-runtime/src/context.rs
- Midnight Ledger — `caller` population logic: https://github.com/midnightntwrk/midnight-ledger/blob/ledger-8/ledger/src/structure.rs
- Midnight Ledger — contract call `zk_verify` public inputs: https://github.com/midnightntwrk/midnight-ledger/blob/ledger-8/spec/contracts.md
- Midnight Agent DID Manager — a working W3C-aligned DID registry on Midnight Preprod that motivated this MPS through the practical experience of building identity and access control contracts: https://github.com/apestchanker/midnight-agent-did-manager
- MPS-0015: Agent Identity Gap on Midnight: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md

## Acknowledgements

This MPS emerged from the practical experience of building a W3C-aligned DID registry on Midnight Preprod. While implementing role-based access control and DID ownership patterns, it became clear that no available Compact primitive could provide cryptographically-enforced caller identity. Source-level investigation of the Compact compiler and Midnight ledger confirmed the gap and the presence of an existing `caller` field at the protocol level that is not yet surfaced to contract code.

## Copyright

This MPS is licensed under CC-BY-4.0.
