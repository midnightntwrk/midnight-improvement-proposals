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

---
MPS: <Number> # assigned by editors
Title: Multi-key Account Custody for Midnight-Native Assets
Authors: Hector Bulgarini (hbulgarini), Nicolas Di Prima (NicolasDP)
Status: Proposed
Category: Libraries and Tooling | Standards
Created: 04-Jun-2026
Requires: none
Replaces: none

---

## Abstract

Midnight has no ratified model for an on-chain account that custodies and
authorises a user's Midnight-native assets — Night and shielded (Zswap) — under
multi-key, multi-device control while satisfying lost-device recovery,
total-loss recovery, and key non-exfiltration at once. There is no
ratified answer to *where* those assets live or *how* their custody is
authorised and recovered, and the asset classes do not behave uniformly:
user↔contract Night works, contract custody of shielded notes works but leaks
holdings into public ledger state, inter-contract Night is tooling-blocked, and
contract-paid Dust has no v1 surface. Without one ratified model, assets
fragment across asset classes, derivation schemes, and operators, and any asset
the account does not govern forfeits its recovery and revocation guarantees.
This problem surfaced in the design of Midnight Passport, but it confronts any
multi-key account or smart-contract wallet on Midnight. This MPS frames the
problem and the constraints any solution must satisfy, and recommends the MIPs
needed to specify a single, recoverable, privacy-preserving custody model. It
does not select a solution.

## Vision

A single account governs the custody, authorisation, and recovery of every
Midnight-native asset it holds, so that "recovered account" implies "recovered
assets", with no fragmentation across asset classes and no residual seed
dependency. The model holds under multi-key, multi-device control and preserves
the guarantee that keys never leave the party that holds them. It is specified
for adoption by the Midnight Foundation and partner wallets rather than tied to
any single product.

## Problem

**Conflicting requirements.** A custody model must deliver a seedless, fully
recoverable, MPC-backed account *and* multi-device access at once. Left to
defaults, independent device keys and asset paths resolve to distinct accounts
or key roots; coordinating them into one coherent, recoverable account is what
is hard, and no single layer does it today.

**Custody location is unresolved, and asset classes diverge.** There is no
ratified answer to where assets are held or how their custody is authorised:
held by the on-chain account contract, at chain-native addresses derived from a
seed-shaped root, or split between the two. The choice is not uniform across
asset classes either, since Night, shielded (Zswap), and Dust each support a
different set of custody and fee operations, so a single pattern may not serve
all three. Foreign-chain assets sit outside this question entirely; they are
handled by upstream cross-chain vaults, and Passport custodies Midnight-native
assets only.

**Contract custody regresses shielded privacy.** The available recipe for
contract-held shielded notes stores `Map<color, QualifiedShieldedCoinInfo>` in
*public* ledger state, so the
contract's holdings (value and colour) become publicly visible. That is a
regression versus user-held notes, and a linkability vector. Choosing it means
accepting that publicity or paying for mitigations of uncertain sufficiency.

**Assets outside the account forfeit its guarantees.** Value the account does
not custody does not inherit its security, recoverability, or revocation, which
are the very protections Passport exists to provide. That is what demands an
account-wide custody strategy rather than per-feature accretion.

**Threshold-signature MPC does not compose with proving.** Shielded and contract
operations are authorised not by a signed transaction, as Night/unshielded
transfers and external chains are, but by *proof generation*, where the secret
key enters the proof as a witness. Threshold signatures (threshold Schnorr,
FROST), the best-known MPC primitive for splitting key custody so no node holds
the whole key, only compose with paths that *verify a signature*: each node
emits its partial signature and a verifier checks each is valid. The shielded
and contract path verifies a ZK proof, not a signature, so there is no signature
for the nodes to jointly produce and no verification step to consume one. The
standard threshold approach therefore has nothing to attach to, and the witness
must instead be assembled in a single place to prove. That reintroduces a single
point of key exposure, breaking the guarantee that keys never leave the party
that holds them and the MPC property that a key is never exposed in one place,
and it makes custody only as strong as that proving boundary.

**Recovery must follow the assets.** Total-loss recovery must reattach the user
to the *same* account, with the same name and the same balances. Whether this
holds depends on the custody model; a model that restores identity but not asset
access is a recovery failure.

## Use Cases

- **Onboarding** yields one account that immediately custodies the user's
  assets, not a primary wallet plus satellite balances.
- **Multi-device:** any authorised peer device operates on the one account, with
  no per-device asset partition.
- **Lost-device:** revoking a device leaves all custodied assets accessible,
  none stranded behind the revoked key.
- **Total-loss recovery:** recovering the account restores the *same* balances.
- **Receiving / spending shielded value** into and out of the account across
  blocks.

## Goals

1. **One ratified custody model** satisfying multi-device access, lost-device
   recovery, total-loss recovery, and key non-exfiltration *simultaneously*.
2. **Recovery follows assets.** Recovering the account restores access to the
   same balances, verified by an end-to-end recovery test.
3. **No seed re-entry and no key exfiltration** on any asset path, including at
   the proving boundary.
4. **A custody path for Night and shielded, a fee path for Dust,** and a
   specified boundary with upstream cross-chain custody.
5. **No net privacy regression** from custody, or an explicitly scoped and
   accepted trade-off.
6. **Portability.** The model is specified for Foundation and partner-wallet
   adoption, and backed by evidence.

## Expected Outcomes

Onboarding yields a single account that coherently governs all Midnight-native
assets. Developers target one custody surface instead of reconciling per-asset
behaviours; partner wallets get an implementable specification with known
feasibility boundaries; users get funds recoverable and revocable by the same
mechanisms that govern the account, with privacy trade-offs made explicit.
The downstream cryptographic-stack decisions (derivation, signing, recovery,
storage) gain a stable upstream choice to calibrate against.

## Open Questions

- **QSCI privacy:** accept public visibility of contract-held holdings, or
  invest in mitigations of uncertain sufficiency?
- **Compliance:** the shielded model hides the sender, so receiver-side KYC/AML
  cannot be satisfied from receiver data alone. Should sender-side selective
  disclosure live in the custody layer, the proof layer, or both?
- **Asset-class boundary:** one uniform pattern, or hybrid by class (possibly
  forced by the Dust gap)?
- **Per-device keys vs derivation** for authorising custody operations.
- **Recovery semantics:** how the model guarantees that a recovered account
  regains the same balances, and how it relates to any seed-shaped root.
- **Proving boundary / threshold composability:** can authorisation be split
  across MPC nodes whose contributions are each verifiable, or must the witness
  be assembled in one place to prove?
- **Dust fee path:** where Dust balances live and how fees are paid without
  fragmenting custody, absent a contract paymaster.
- **Shielded contract ↔ contract feasibility:** untested; plausibly subject to
  the same tooling gap as inter-contract Night.

## Recommended MIPs

- **Multi-key account contract.** Specifies the custody shape
  (contract, address, or hybrid), and what the account contract holds versus
  what lives at addresses. This is the keystone the others hang from.
- **Recovery paths.** Social recovery (DeRec) plus encrypted-blob backup as
  substitutable alternatives.


## References (Optional)

- OpenZeppelin `compact-contracts@add-multisig` `ShieldedTreasury.compact`: the
  working shielded contract-custody pattern.
- `midnightntwrk/midnight-wallet#293` and `midnightntwrk/midnight-ledger#233`:
  the upstream fixes gating inter-contract Night.

## Acknowledgements

The IOG ARC department and Midnight Foundation.

## Copyright

This MPS is licensed under CC-BY-4.0.