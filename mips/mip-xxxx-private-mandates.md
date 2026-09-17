---
MIP: X
Title: Private Mandate Tokens for Autonomous Agent Authority
Authors: Felipe Nunes Oliveira (@devfelipenunes)
Status: Draft
Category: Standards
Created: 2026-07-08
Requires: MIP-0001
Replaces: None
MPS: MPS-0015
Discussions: https://github.com/midnightntwrk/midnight-improvement-proposals/pull/251
---

<!--
 Copyright Midnight Foundation

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

This MIP defines a standard for **Private Mandate Tokens** — a privacy-preserving
delegation primitive for autonomous agents on the Midnight Network. It enables a
sovereign identity (the **Issuer**) to delegate scoped, revocable authority to an
AI agent (the **Agent**) using zero-knowledge proofs, without revealing the
delegation graph, agent identity, or authority bounds to the public ledger.

The standard defines a single normative Compact contract — **Nexus** — that
issues non-transferable, revocable **Private Mandates**. Each mandate is an
off-chain data structure committed on-chain as an opaque `persistentHash`.
The Agent proves authority via a zk-SNARK (`prove_active_mandate`); the Issuer
revokes via an opaque hash (`revoke_mandate`), preserving privacy of the
revocation target.

The MIP also describes a **reference application** — **zPay** — that demonstrates
how a payment contract composes with Nexus. zPay is **not** part of the normative
standard; it shows the integration pattern for any contract that wants to gate
spending by mandate authority.

## Motivation

The emergence of autonomous AI agents creates a fundamental trust problem: how
can a human grant a software agent the ability to act on their behalf without
surrendering privacy or requiring continuous supervision?

Existing delegation mechanisms fall short:

- **Public delegation** (as in Stellar SEP-XXXX) reveals who delegates what to
  whom — creating a permanent, transparent graph of authority relationships.
- **Key sharing** is catastrophic — a compromised hot key grants unlimited
  authority, with no cryptographic circuit-breaker.
- **Off-chain permission files** (JWTs, OAuth scopes) cannot be verified by
  smart contracts in a trustless manner.

Midnight's architecture — client-side proving, the Kachina protocol's hybrid
public/private state model, and the Compact language's zero-knowledge constraint
system — enables a fundamentally different approach. The Agent becomes the
**Prover**: it holds an off-chain mandate, generates a zk-SNARK proving the
requested action falls within scope, and submits only the proof. The verifier
learns **nothing** beyond the boolean result of the authorization check.

This MIP positions delegation of **authority** as the standard primitive. It
composes with the [Midnight Agent Identity Standard (MAIS, #110)] and its
problem statement ([MPS-0015]) for the identity layer — who an agent is. This
MIP defines what an agent is **allowed to do**. The two compose: MAIS proves
_identity_; Private Mandate Tokens prove _authorized capability_.

## Specification

The normative language for this MIP follows [RFC-2119]: **MUST**, **SHOULD**,
and **MAY** are to be interpreted as described therein.

### Terminology

| Term                | Definition                                                                          |
| ------------------- | ----------------------------------------------------------------------------------- |
| **Issuer**          | The `Bytes<32>` coin public key authorized to create and revoke mandates.           |
| **Agent (Prover)**  | The `Bytes<32>` coin public key authorized to act under a mandate.                  |
| **Private Mandate** | Off-chain authority record, committed on-chain as an opaque hash.                   |
| **mandate_hash**    | `persistentHash([domain_separator, mandate_id, agent, valid_until, caps, max_val])` |
| **Nexus**           | The normative Compact contract for mandate issuance, verification, and revocation.  |
| **zPay**            | A reference application demonstrating mandate-gated payments (non-normative).       |

### Nexus Contract (normative)

Nexus is the only contract normative to this MIP. It stores **no mandate
content** — only opaque hash commitments and a revocation map. This preserves
privacy: the delegation graph, agent identity, and authority bounds never touch
the ledger.

#### Ledger State

```compact
export sealed ledger domain_separator: Bytes<32>;
export sealed ledger issuer: Bytes<32>;
export ledger mandate_commitments: Map<Bytes<32>, Boolean>;
export ledger revoked_mandates: Map<Bytes<32>, Boolean>;
export ledger query_counter: Counter;
```

- **`domain_separator`** — set once in the constructor as
  `persistentHash([issuer_address])`. Binds mandate hashes to this contract
  instance, preventing cross-contract replay. Write-once (`sealed`).
- **`issuer`** — the coin public key authorized to emit and revoke mandates.
  Write-once (`sealed`).
- **`mandate_commitments`** — `mandate_hash → true` for every emitted mandate.
- **`revoked_mandates`** — `mandate_hash → true` for every revoked mandate
  (kill switch). Append-only.
- **`query_counter`** — a `Counter` incremented by each transaction to satisfy
  the Kachina state model's cell requirement. Its value carries no business
  meaning. Note: it is a _runtime requirement_, not a protocol field — see
  [Backwards Compatibility Assessment].

**INV-1 (write-once):** `domain_separator` and `issuer` are set exclusively by
the constructor and cannot be modified by any circuit (`sealed`). Re-initializing
the contract is impossible by construction — there is no `initialize` circuit.

#### Witnesses

```compact
witness msgSender(): Bytes<32>;
witness mandate_agent_pubkey(id: Bytes<32>): Bytes<32>;
witness mandate_valid_until(id: Bytes<32>): Uint<64>;
witness mandate_capabilities(id: Bytes<32>): Uint<64>;
witness mandate_max_value_per_tx(id: Bytes<32>): Uint<64>;
```

Witnesses are private inputs provided by the Agent's local runtime. The
contract treats them as untrusted input; correctness is enforced by the
commitment check in `prove_active_mandate` (the recomputed hash must match an
emitted commitment).

#### Constructor

```compact
constructor(issuer_address: Bytes<32>) {
  domain_separator = disclose(persistentHash<Vector<1, Bytes<32>>>([
    disclose(issuer_address),
  ]));
  issuer = disclose(issuer_address);
  query_counter.increment(1);
}
```

The constructor is invoked once at deployment. There is **no** re-callable
`initialize` circuit — this closes the front-running re-initialization vector
by construction.

#### Circuits

**`emit_mandate(mandate_hash: Bytes<32>)`** — registers a mandate commitment.
The Issuer MUST compute the `mandate_hash` off-chain via
`compute_mandate_hash` (or an equivalent serialization) and pass only the hash.
No mandate field (id, agent, expiry, capabilities, limit) is revealed on-chain.

**INV-2 (emission privacy):** `emit_mandate` reveals only the opaque
`mandate_hash`; no pre-image is stored or disclosed on-chain.

```compact
export circuit emit_mandate(mandate_hash: Bytes<32>): [] {
  assert(disclose(msgSender()) == disclose(issuer), "Nexus: not issuer");
  const d_hash: Bytes<32> = disclose(mandate_hash);
  assert(!disclose(mandate_commitments.member(d_hash)), "Nexus: already emitted");
  mandate_commitments.insert(d_hash, disclose(true));
  query_counter.increment(1);
}
```

The Issuer MUST validate `valid_until > 0` and `capabilities <= 64` off-chain
**before** computing the hash (the contract cannot validate fields it never
sees). The SDK MUST enforce this.

**`prove_active_mandate(mandate_id, action, value): Boolean`** — the core
authorization circuit. The Agent proves it holds an active mandate by
recomputing the hash from witnesses and checking the on-chain commitment and
revocation maps:

1. **Identity**: `msgSender() == mandate_agent_pubkey(id)`.
2. **Freshness**: not expired (`blockTimeGte(valid_until)` is false).
3. **Capability**: `action < capabilities`.
4. **Value limit**: `value <= max_value_per_tx`.
5. **Emission**: the recomputed hash is in `mandate_commitments`.
6. **Non-revocation**: the recomputed hash is not in `revoked_mandates`.

Returns `true` only if all six hold.

**`revoke_mandate(mandate_hash)`** — the kill switch. The Issuer submits the
mandate's opaque hash to `revoked_mandates`. Observers learn only that _some_
mandate was revoked — not which one, by whom, or for what scope.

**Query circuits**: `is_revoked(mandate_hash): Boolean`, `get_issuer(): Bytes<32>`,
`get_domain_separator(): Bytes<32>`, `get_query_count(): Uint<64>`.

**Pure circuits** (run off-chain, no transaction): `compute_mandate_hash(...)`
and `compute_domain_separator(issuer)` — both use the ledger's exact
`persistentHash` serialization, so SDKs MUST use them instead of `sha256` to
avoid hash mismatch.

### Mandate Hash

```text
mandate_hash = persistentHash([
  domain_separator,       // Bytes<32>, binds to this contract
  mandate_id,             // Bytes<32>, unique nonce
  agent,                  // Bytes<32>, authorized agent
  valid_until,            // Uint<64>, Unix timestamp
  capabilities,           // Uint<64>, action count
  max_value_per_tx        // Uint<64>, per-tx limit
])
```

`persistentHash` is the ledger's typed, aligned hash — **not** raw `sha256`.
SDKs MUST compute the hash via the `compute_mandate_hash` pure circuit.

### Reference Application: zPay (non-normative)

zPay demonstrates how a payment contract composes with Nexus. It is **not**
part of the standard — any contract may implement its own mandate-gated logic.

Since Compact does not support cross-contract calls, zPay cannot query Nexus
on-chain. It follows the **ZKCD integration pattern**:

1. **Domain alignment**: zPay computes `domain_separator = persistentHash([admin])`
   in its constructor. When `admin == issuer` (same deployer), this equals the
   Nexus domain separator, so mandate hashes recompute identically.
2. **Mirrored state**: zPay keeps `known_mandates` (commitments registered at
   deposit) and `revoked_vaults` (mirror of revocation, written by
   `revoke_vault` — called by the SDK when Nexus revokes).
3. **Mandate-gated pay**: `pay(mandate_id, amount)` recomputes the hash from
   the same witnesses and requires: known (A3), not revoked (A1), not expired
   (A1), `amount <= max_value` (cap), and `msgSender == agent` (authorization).

zPay's `pay` demonstrates the security invariants any mandate-gated application
SHOULD enforce.

## Rationale

**Authority is the primitive, not payments.** The MIP's title and the ecosystem
gap justify standardizing delegated _authority_ — the "what an agent is allowed
to do" — as a composable primitive. Payments, governance, DeFi, and identity
applications can all gate on mandate authority without inheriting a vault.

**Off-chain mandate + on-chain commitment.** Storing only the hash keeps mandate
content private (front-running resistance, cross-identity privacy) while the
commitment prevents forgery: an Agent cannot invent a mandate whose hash is not
in `mandate_commitments`.

**Hash-only emission.** Passing the pre-computed hash (instead of disclosing
all fields) makes emission privacy real: observers see an opaque 32-byte value,
computationally indistinguishable from random, with no recoverable pre-image.

**`sealed` constructor, not `initialize`.** A re-callable `initialize` circuit
creates a front-running window where an attacker re-initializes the contract
with their own key. `sealed ledger` + constructor makes write-once a
compile-time guarantee.

**`Map.member`, not `Map.lookup`, for existence.** In Compact, `lookup` of an
absent key returns `null`; `disclose(null)` fails. Existence checks MUST use
`member()`. (This was found empirically during validation — see
[Testing].)

**`query_counter` is a runtime cell anchor, not a protocol nonce.** Midnight's
Kachina model requires each transaction to consume or produce an on-chain cell.
`query_counter` exists solely to satisfy this; it is not a business-meaningful
nonce. (The `tx_counter` used in earlier drafts was a misnomer — the protocol
equivalent is the account nonce, managed by the wallet.)

**Composition with identity standards.** This MIP defines the _authority_ layer
— what an agent is allowed to do. It composes with the Midnight Agent Identity
Standard (MAIS, #110) and its problem statement (MPS-0015) for the _identity_
layer — who an agent is, including reputation and validation. Both standards
share Midnight's three-tier disclosure model (public, selective, shielded); the
Disclosure Tier Registry is a natural shared primitive, and this MIP aligns its
disclosure modes with it as informative context, without creating a normative
dependency.

## Path to Active

### Acceptance Criteria

1. The reference implementation compiles with `compactc` (v0.31.1) and passes
   simulator tests on the devnet (21/21, covering C1/C2/A1/A3/A4/INV-1..5).
2. At least one third-party contract integrates mandate authority (the zPay
   reference application demonstrates the pattern).
3. A TypeScript SDK provides `NexusClient` with `pureCircuits.compute_mandate_hash`
   and `compute_domain_separator`, plus agent-side mandate storage.
4. A reference deployment exists on a public network (preprod), with contract
   addresses and transaction examples.

### Implementation Plan

- **Phase 1** (complete): Reference implementation — contracts compiled,
  deployed on devnet, simulator-tested (21/21).
- **Phase 2**: TypeScript SDK (`NexusClient`) with pure-circuit hash helpers.
- **Phase 3** (complete): zPay reference application + MCP server for AI agents.
- **Phase 4**: Preprod deployment and third-party integrations.

## Backwards Compatibility Assessment

This defines a new standard — there are no existing deployments. All identities
use `Bytes<32>` (standard for Compact). The `mandate_hash` pre-image is part of
the standard; changing it invalidates all commitments.

The `query_counter` field is present to satisfy the Kachina cell requirement.
It is deliberately **not** a protocol-level nonce — Midnight manages transaction
nonces at the wallet/account level. Contracts that integrate Nexus MUST NOT
assume `query_counter` has business semantics.

## Security Considerations

### Witness Integrity

The security model rests on witness correctness. `prove_active_mandate` enforces
that the recomputed `mandate_hash` matches an on-chain commitment — binding
witness values to a trusted anchor. A witness that lies about agent/expiry/
limits cannot produce a hash matching a commitment the Issuer registered.

### Emission Privacy (INV-2)

`emit_mandate` reveals only the opaque hash. `persistentHash` is
collision-resistant and, for the mandate pre-image (which includes a
256-bit random `mandate_id`), computationally hides all fields. An observer
cannot recover agent identity, expiry, capabilities, or limits from the ledger.

### Replay Protection

`prove_active_mandate` uses `blockTimeGte` to bind proofs to the block context.
A proof for block N is valid only while the mandate has not expired. For
stronger within-block replay protection, a monotonic per-agent nonce via
witness is a future MAY (see below).

### Revocation Privacy

`revoke_mandate` adds an opaque hash to a public map. Observers see only that
_some_ mandate was revoked. Because the hash is a collision-resistant commitment
with a high-entropy pre-image, the revocation target stays private.

### Vault Authorization (reference application)

zPay's `pay` requires `msgSender == agent` where `agent` is bound to the mandate
via the recomputed hash. This means possession of the `mandate_hash` alone is
**not** sufficient to spend — the caller must be the agent the Issuer
authorized. Since only the Issuer (admin) can register commitments, a revoked
or spoofed mandate cannot drain a funded vault.

### Authorisation Model and Future Direction

The current standard authenticates the caller via the `msgSender` witness,
which the Midnight wallet supplies from the transaction signer. This is correct
for the wallet-agent flow but relies on the runtime, not an in-circuit
signature check. The emerging [MIP-0013] pattern (Schnorr signatures over
JubJub, verified in-circuit) MAY be adopted as an upgrade path for stronger,
non-wallet-dependent authorization. Implementations MAY add such verification
to `prove_active_mandate` / `pay` while keeping the mandate-hash commitment
mechanism unchanged.

### Accounting, not Custody

The zPay vault is an **accounting ledger**: `vault_balances` records authorized
spend, and the actual NIGHT transfer happens in the same transaction envelope
via the wallet. The contract cannot force the transfer (a malicious prover
could deduct without transferring). This is honest by design: the accounting
prevents over-spend beyond the mandate budget, and revocation is the kill
switch. Custodial contracts (e.g., per [MIP-0012]) compose orthogonally.

## Implementation

Repository: `github.com/devfelipenunes/zolvency`

| Component         | Path                                         | Language          |
| ----------------- | -------------------------------------------- | ----------------- |
| Nexus (normative) | `contracts/midnight/nexus/src/nexus.compact` | Compact `>= 0.20` |
| zPay (reference)  | `contracts/midnight/zpay/src/zpay.compact`   | Compact `>= 0.20` |
| TypeScript SDK    | `zpay/zpay-midnight-sdk/`                    | TypeScript        |
| MCP Server        | `zpay/zpay-midnight-mcp/src/index.ts`        | TypeScript        |

Compiler: `compactc` v0.31.1 (language v0.23.0, runtime v0.16.0, ledger 8.0.2).

## Testing

Validation is layered (see [Implementation Plan]):

1. **Simulator tests** (`test/pm-simulator.test.ts`): execute circuits in-memory
   via `createConstructorContext`/`createCircuitContext` — no network, seconds.
   Covers C1 (sealed write-once), C2 (emit hash-only, non-issuer and duplicate
   rejected), A1 (revoke then prove/pay fails), A3 (unknown mandate fails),
   A4 (non-admin deposit fails), INV-4 (over-cap fails), INV-5 (non-negative
   balance). **Result: 21/21 passing.**
2. **Deploy**: `deploy-all.mjs` deploys Nexus + zPay with constructor args and
   validates the constructor executes on-chain.
3. **E2E on-chain** (`e2e-*.mjs`): validates mutations (`emit_mandate`,
   `deposit`) on the devnet. Note: complex circuits (queries, `pay`) may stall
   on a GPU-less local devnet; on a public network or GPU-backed node they
   complete normally. Pure circuits compute values off-chain, avoiding the
   query path.
4. **DUST sponsorship**: multi-wallet fee payment via `balanceFinalizedTransaction`
   (see `e2e-dust-sponsor.mjs`).

## References

- [MIP-0001: MIP Process](https://github.com/midnightntwrk/midnight-improvement-proposals)
- [Midnight Developer Docs](https://docs.midnight.network/)
- [Compact Language Guide](https://docs.midnight.network/development/compact)
- [MPS-0015: Agent Identity](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md)
- [MAIS — Midnight Agent Identity Standard (#110)](https://github.com/midnightntwrk/midnight-improvement-proposals/issues/110)

## Acknowledgements

Thanks to the Midnight Foundation team for the Compact language and Kachina
protocol, and to Zidan (mzf11125) for the MAIS/MPS-0015 alignment that clarified
the authority-vs-identity composition.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under
the Apache License, Version 2.0. Submission requires agreement to the Midnight
Foundation Contributor License Agreement, which includes the assignment of
copyright for your contributions to the Foundation.

[RFC-2119]: https://datatracker.ietf.org/doc/html/rfc2119
[MIP-0013]: https://github.com/midnightntwrk/midnight-improvement-proposals
[Backwards Compatibility Assessment]: #backwards-compatibility-assessment
