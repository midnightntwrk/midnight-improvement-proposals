---
MIP: XXXX
Title: Private Mandate Tokens for Autonomous Agent Authority
Authors: Felipe Nunes Oliveira <@devfelipenunes>
Status: Draft
Category: Standards
Created: 2026-07-08
Requires: MIP-1 (MIP Process)
Replaces: None
---

## Simple Summary

A standardized Compact smart contract interface for issuing non-transferable,
revocable **Private Mandates** that allow a sovereign identity (**Root Anchor**)
to delegate programmable, privacy-preserving authority to AI agents — revealing
only what the proof requires, nothing more.

## Motivation

The emergence of agentic AI systems creates a fundamental trust problem: how can
a human grant an autonomous agent the ability to act on their behalf without
surrendering their privacy?

Existing approaches fall short:

- **Public delegation** (as in Stellar SEP-XXXX) reveals who delegates what to
  whom — creating a permanent, transparent graph of authority relationships that
  may itself be sensitive.
- **Key sharing** is dangerous — a compromised agent gains unlimited authority.
- **Off-chain permission systems** cannot be verified by smart contracts in a
  trustless manner.

Midnight's Kachina protocol, with its public + private state model and
zero-knowledge proofs, enables a new primitive: **Private Mandates** where the
existence, scope, and budget of a delegation can be selectively disclosed.

This MIP defines a standard that any Midnight Compact contract can integrate to
verify whether an agent is authorized to act — and within what bounds — without
revealing the underlying delegation graph, identities, or budget details to the
verifying contract.

## Abstract

This proposal defines a Compact contract interface for hierarchical,
non-transferable authority tokens on the Midnight Network. The standard extends
the concepts from Stellar SEP-XXXX (Hierarchical Mandate Tokens) with
privacy-preserving ZK capabilities unique to Midnight:

1. **Private Authority Graph** — delegation relationships are shielded by ZK
   circuits; a verifier learns only whether authority is valid, not the chain.
2. **Selective Scope Disclosure** — the agent proves compliance with scope
   constraints (TTL, budget, allowlists) without revealing the actual values.
3. **Shielded Budget Consumption** — budget tracking via Zswap shielded coins;
   neither balance nor consumption pattern is public.
4. **Private Epoch Revocation** — epoch increments are public, but which
   mandates are affected remains private to the participants.

## Comparison: SEP-XXXX vs MIP-XXXX

| Aspect                   | Stellar (SEP-XXXX)            | Midnight (MIP-XXXX)                         |
| ------------------------ | ----------------------------- | ------------------------------------------- |
| **Smart contract model** | Soroban (Rust)                | Compact (ZK-native)                         |
| **State model**          | Fully public (Account)        | Public ledger + Private witnesses (Kachina) |
| **Delegation graph**     | Visible on-chain              | Shielded via ZK circuit                     |
| **Scope enforcement**    | On-chain contract reads Scope | ZK circuit proves scope compliance          |
| **Budget tracking**      | Public `spent_budget` counter | Shielded Zswap coins                        |
| **Agent identity**       | Public Address                | Shielded or unshielded (configurable)       |
| **Verify cost**          | O(depth) gas, cached O(1)     | O(1) proof verification                     |
| **Revocation**           | Public epoch increment        | Public epoch + private invalidation proof   |
| **Token standard**       | SEP-41 (Soroban token)        | Zswap shielded tokens                       |

## Terminology

| Term                 | Definition                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------- |
| **Root Anchor**      | The sovereign identity that holds primary authority. May be shielded or public.             |
| **Agent**            | The address that receives a Mandate and may act within its Scope.                           |
| **Private Mandate**  | A non-transferable authority record with shielded details, proven via ZK circuit.           |
| **Scope**            | The rules (TTL, budget, allowlists) attached to a Mandate. Only the commitment is on-chain. |
| **Nexus**            | The central Midnight Compact contract that issues, tracks, and verifies Mandates.           |
| **Scope Commitment** | A `persistentCommit` hash of the Scope, stored on the public ledger.                        |
| **Authority Epoch**  | A monotonic counter on the Root Anchor. Public on ledger.                                   |
| **Shielded Budget**  | Budget allocation tracked as Zswap shielded coins rather than public counters.              |

## Specification

### I. Contract Structure (Compact)

#### Public Ledger State

Minimal public state — only what is required for trustless verification:

```compact
pragma language_version >= 0.20;
import CompactStandardLibrary;

// ── Admin ───────────────────────────────────────────────────────────

export ledger initialized: Boolean;
export ledger nexus_admin: Address;

// ── Root Anchors ────────────────────────────────────────────────────

export ledger anchors: Set<Address>;

// Maps anchor address → current epoch counter
export ledger epochs: Map<Address, Uint<64>>;

// ── Mandate Registry ────────────────────────────────────────────────

// Maps mandate_id (sequential counter) → scope_commitment (persistentCommit hash)
// The presence of a mandate_id in this map means it was issued.
export ledger scope_commitments: Map<Uint<64>, Field>;

// Revocation flag for each mandate_id
export ledger is_revoked: Map<Uint<64>, Boolean>;

// Counter for issuing sequential mandate IDs
export ledger mandate_counter: Counter;

// ── Shielded Budget Registry ────────────────────────────────────────

// Total shielded budget allocated per mandate (public for supply audit)
// Individual consumption is private (Zswap).
export ledger total_allocated: Map<Uint<64>, Uint<64>>;
```

#### Private Witnesses (Local Component)

Each agent holds their mandate data locally as private witnesses.
These are NEVER stored on the ledger — they are proven via ZK circuits.

```compact
// Private mandate data (witness callbacks provided by the local DApp)
witness mandate_root_anchor(id: Uint<64>): Address;
witness mandate_agent(id: Uint<64>): Address;
witness mandate_issuer(id: Uint<64>): Address;
witness mandate_ttl(id: Uint<64>): Uint<64>;
witness mandate_granted_budget(id: Uint<64>): Uint<64>;
witness mandate_issued_at_epoch(id: Uint<64>): Uint<64>;
witness mandate_contract_allowlist_commitment(id: Uint<64>): Field;
```

#### Constructor

```compact
export circuit constructor(admin: Address): [] {
  nexus_admin.write(admin);
  initialized.write(true);
}
```

### II. Core Interface

#### `register_anchor`

Registers a Root Anchor identity on-chain. This step is REQUIRED for
account-model anchors. (Shielded anchors use an alternative flow.)

```compact
export circuit register_anchor(anchor: Address): [] {
  assert(initialized.read(), "Not initialized");
  assert(!anchors.contains(anchor), "Anchor already registered");
  anchors.insert(anchor);
  epochs.set(anchor, 0);
}
```

#### `issue_mandate`

Issues a new Private Mandate. The Scope is cryptographically committed
to the ledger via `persistentCommit` — the details remain private.

```compact
export circuit issue_mandate(
  issuer: Address,
  agent: Address,
  scope_commitment: Field,
  budget: Uint<64>,
  parent_id: Maybe<Uint<64>>
): Uint<64> {
  // 1. Verify issuer is a known anchor
  assert(anchors.contains(issuer), "Unknown issuer");

  // 2. Generate new mandate ID
  mandate_counter.increment(1);
  const mandate_id = mandate_counter.read();

  // 3. On-chain commitments (zero-knowledge, no plaintext Scope)
  scope_commitments.set(mandate_id, scope_commitment);
  is_revoked.set(mandate_id, false);
  total_allocated.set(mandate_id, budget);

  // 4. Hierarchy enforcement
  // Parent MUST be valid — the off-chain circuit enforces sub-scope ≤ parent scope.
  // On-chain, we only verify parent exists and is not revoked.
  if (disclose(parent_id.isSome())) {
    const pid = parent_id.unwrap();
    assert(scope_commitments.contains(pid), "Parent not found");
    assert(!is_revoked.getOrDefault(pid, false), "Parent revoked");

    // Sum-of-children ≤ parent budget enforced in ZK circuit, not here
  }

  return mandate_id;
}
```

#### `verify_authority`

The core ZK circuit. Any Midnight contract calls this to verify an agent's
authorization — learning ONLY the boolean result, nothing else.

```compact
export circuit verify_authority(
  mandate_id: Uint<64>,
  agent: Address
): Boolean {
  // 1. Mandate must exist (scope commitment on-chain)
  const onchain_commitment = scope_commitments.get(mandate_id);
  if (disclose(onchain_commitment == 0)) {
    return false; // mandate does not exist
  }

  // 2. Not revoked (public check)
  if (disclose(is_revoked.getOrDefault(mandate_id, false))) {
    return false;
  }

  // 3. Verify TTL not expired (private — only the result is disclosed)
  const ttl = mandate_ttl(mandate_id);
  // blockTimeGte(ts) returns true if current block time >= ts
  if (disclose(blockTimeGte(ttl))) {
    return false; // expired
  }

  // 4. Verify agent identity (private witness matches public caller)
  const stored_agent = mandate_agent(mandate_id);
  if (disclose(stored_agent != agent)) {
    return false;
  }

  // 5. Verify scope commitment matches (ZK proof of Scope validity)
  const witness_commitment = mandate_scope_commitment(mandate_id);
  // (witness_commitment is defined as a separate circuit/witness below)
  if (disclose(witness_commitment != onchain_commitment)) {
    return false;
  }

  // All checks passed — agent IS authorized
  return disclose(true);
}

// Helper: prove that the agent's local scope matches the on-chain commitment
circuit mandate_scope_commitment(id: Uint<64>): Field {
  // In a full implementation, this circuit would reconstruct the
  // persistentCommit from individual Scope fields and prove it matches.
  // For the standard interface, we define it as a witness callback.
  return 0; // replaced by local witness
}
```

#### `revoke_mandate`

```compact
export circuit revoke_mandate(
  mandate_id: Uint<64>,
  caller: Address
): [] {
  // Caller must be either:
  // a) A registered anchor AND the root_anchor of this mandate
  // b) The issuer of this mandate

  // For the standard interface, any registered anchor can revoke
  // if they can prove (via ZK) they are the root_anchor.
  assert(anchors.contains(caller), "Caller not an anchor");
  assert(scope_commitments.contains(mandate_id), "Mandate not found");

  is_revoked.set(mandate_id, true);
}
```

#### `increment_epoch`

```compact
export circuit increment_epoch(anchor: Address): Uint<64> {
  assert(anchors.contains(anchor), "Unknown anchor");

  const current = epochs.getOrDefault(anchor, 0);
  const next = current + 1;
  epochs.set(anchor, next);

  return next;
}
```

### III. Delegation Policy & Hierarchy (ZK-Enforced)

Delegation constraints are enforced in the **circuit layer** (off-chain ZK
proof), not on the public ledger. The proving circuit verifies:

1. Child scope ≤ Parent scope in all dimensions (monotonicity)
2. Child budget fraction ≤ Parent's allowed fraction
3. Depth + 1 ≤ `MAX_DELEGATION_DEPTH`
4. Parent is not revoked and epoch is current

```compact
// Pure circuit — no ledger side effects, only proof generation
pure circuit verify_delegation_valid(
  parent_id: Uint<64>,
  parent_root_anchor: Address,
  parent_granted_budget: Uint<64>,
  parent_epoch: Uint<64>,
  parent_ttl: Uint<64>,
  child_scope_commitment: Field,
  child_budget: Uint<64>,
  child_depth: Uint<32>,
  max_depth: Uint<32>,
  child_budget_fraction_pct: Uint<32>,
  anchor_current_epoch: Uint<64>
): Boolean {
  // 1. Parent must be active
  if (disclose(parent_granted_budget == 0)) { return false; }
  if (disclose(blockTimeGte(parent_ttl)))  { return false; }
  if (disclose(parent_epoch != anchor_current_epoch)) { return false; }

  // 2. Depth check
  if (disclose(child_depth > max_depth)) { return false; }

  // 3. Budget fraction enforcement (0 < pct <= 100)
  if (disclose(child_budget_fraction_pct == 0)) { return false; }
  if (disclose(child_budget_fraction_pct > 100)) { return false; }

  // 4. Child budget must not exceed parent's allocated fraction
  const max_child_budget = (parent_granted_budget * child_budget_fraction_pct) / 100;
  if (disclose(child_budget > max_child_budget)) { return false; }

  // 5. Child scope ⊆ parent scope
  // This is proved by the child reconstructing their scope_commitment
  // using values that are individually bounded by parent scope values.
  // The circuit verifies the reconstruction matches the on-chain commitment.

  return disclose(true);
}
```

### IV. Shielded Budget (Zswap Integration)

Budget management uses Midnight's Zswap protocol. Each mandate has an
allocated budget that the agent spends via shielded transfers:

```compact
// ── Budget Allocation ───────────────────────────────────────────────

export circuit allocate_shielded_budget(
  mandate_id: Uint<64>,
  amount: Uint<64>,
  recipient: Recipient
): [] {
  // Only the Nexus admin or a registered anchor can allocate
  assert(anchors.contains(msgSender()), "Unauthorized");

  // Update on-chain total
  total_allocated.set(mandate_id, amount);

  // Mint shielded coins into the agent's budget UTXO
  mintShieldedToken(amount, recipient);
}

// ── Payment from Mandate Budget ─────────────────────────────────────

export circuit mandate_payment(
  mandate_id: Uint<64>,
  agent: Address,
  amount: Uint<64>,
  recipient: Recipient,
  shield_input: ShieldedCoinInfo
): [] {
  // 1. Verify the agent holds a valid mandate
  assert(verify_authority(mandate_id, agent), "Unauthorized");

  // 2. Create Zswap input from the agent's shielded budget coin
  const input = createZswapInput(shield_input);

  // 3. Execute shielded transfer (amount hidden, identities hidden)
  sendShielded(amount, recipient);

  // Note: change output is created automatically by the Zswap runtime,
  // returning remaining budget to the agent's shielded wallet.
}
```

### V. Integration Example

A private lending protocol integrates the Nexus to verify agent authority
without learning any identities or budget details:

```compact
pragma language_version >= 0.20;
import CompactStandardLibrary;
import MidnightNexus;

export ledger nexus_address: ContractAddress;

export circuit private_loan(
  mandate_id: Uint<64>,
  agent: Address,
  loan_amount: Uint<64>,
  shield_input: ShieldedCoinInfo
): Boolean {
  // 1. Create Nexus client from stored address
  const nexus = MidnightNexus(nexus_address);

  // 2. Verify authority — learns ONLY true/false
  //    Does NOT learn:
  //    - Who the Root Anchor is
  //    - What other contracts are allowed
  //    - The full budget amount
  //    - The delegation chain
  const authorized = nexus.verifyAuthority(mandate_id, agent);
  if (disclose(!authorized)) {
    return false;
  }

  // 3. Process shielded payment from mandate budget
  nexus.mandatePayment(mandate_id, agent, loan_amount, nexus_address, shield_input);

  return disclose(true);
}
```

**Privacy guarantees for the verifier**:

- ✅ Agent has valid authority → **proven**
- ❌ Who delegated the authority → **hidden**
- ❌ What the total budget is → **hidden**
- ❌ What other mandates exist → **hidden**
- ❌ What other contracts are allowed → **hidden**
- ❌ How much budget was already spent → **hidden**

### VI. Integration with the zPay Protocol

The zPay agentic payment protocol (defined in SEP-XXXX) can be adapted to
Midnight by replacing the public Soroban mandate verification with the
private Midnight circuit:

```
┌─────────────────────────────────────────────────────────────────┐
│                    zPay-Midnight Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐     ┌────────────────┐     ┌──────────────┐   │
│  │  Root Anchor  │     │   Agent (AI)   │     │  Verifier    │   │
│  │  (Sovereign)  │     │  (Prover)      │     │ (Contract)   │   │
│  └──────┬───────┘     └───────┬────────┘     └──────┬───────┘   │
│         │                     │                      │           │
│         │  1. Issue Mandate   │                      │           │
│         │  (scope_commitment) │                      │           │
│         │────────────────────>│                      │           │
│         │                     │                      │           │
│         │                     │  2. verifyAuthority  │           │
│         │                     │  (ZK proof, O(1))   │           │
│         │                     │─────────────────────>│           │
│         │                     │                      │           │
│         │                     │  3. mandatePayment   │           │
│         │                     │  (shielded transfer) │           │
│         │                     │─────────────────────>│           │
│         │                     │                      │           │
│         │  4. Epoch increment │                      │           │
│         │  (mass revocation)  │                      │           │
│         │────────────────────>│                      │           │
│         │                     │                      │           │
└─────────────────────────────────────────────────────────────────┘
```

The key differences from the Stellar zPay:

| Aspect             | zPay (Stellar/Soroban)   | zPay-Midnight           |
| ------------------ | ------------------------ | ----------------------- |
| Mandate visibility | Public on Nexus          | Scope commitment only   |
| Agent identity     | Public Address           | Private in ZK circuit   |
| Payment type       | Public XLM/USDC transfer | Shielded Zswap transfer |
| Budget tracking    | Public `spent_budget`    | Zswap shielded coins    |
| Verification       | Soroban contract call    | ZK circuit proof        |
| Privacy level      | None (transparent)       | Selective disclosure    |

### VII. Security Considerations

#### Scope Monotonicity via ZK

The circuit MUST enforce that a child scope commitment is strictly more
restrictive than its parent without revealing either. This is achieved
through cryptographic commitments and range proofs — not by comparing
plaintext values on-chain.

#### Replay Protection

Each circuit invocation is bound to the current block context. The
`blockTimeGte` and `blockTimeLt` functions prevent proof replay across
blocks.

#### Budget Exhaustion via Shielded Coins

Because budget is tracked as Zswap shielded coins rather than a public
counter, the risk of front-running budget consumption is eliminated.
Only the agent holding the mandate's private key can spend its shielded
budget.

#### Epoch-based Mass Revocation

Incrementing the epoch is public (required for verifiability), but which
mandates are affected by the epoch change remains private. Old mandates'
scope commitments remain on the ledger but fail the epoch check in the
circuit — a fact only the agent and the verifier learn during a
`verify_authority` call.

#### ZK Circuit Soundness

The security of the entire system rests on the soundness of the ZK
circuits. Implementations MUST use the official Compact compiler and
SHOULD have their circuits formally verified or audited before mainnet
deployment.

### VIII. Open Questions

1. **Shielded Root Anchors** — Should the standard support fully-shielded
   anchors (no public address on ledger)? This would require a different
   registration flow via ZK proof of key possession, without the `anchors`
   public Set.

2. **Merkle-based Allowlist Proofs** — Contract allowlist verification
   via Merkle inclusion proofs. The specific tree depth and hash function
   should be standardized.

3. **Cross-Mandate Budget Aggregation** — Should an agent be able to prove
   combined budget across multiple mandates from different root anchors?

4. **DUST Economics** — The cost of proof generation and submission on
   Midnight must be analyzed for typical mandate chains (depth 1-8).

5. **Witness Implementation** — The standard defines the on-chain interface
   but not the local component implementation. A reference implementation
   in TypeScript should accompany any production deployment.

### IX. Relationship to Existing Standards

| Standard                     | Relationship                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------- |
| **SEP-XXXX (Stellar)**       | Conceptual ancestor. This MIP extends the mandate model with ZK privacy.        |
| **EIP-4973 (Ethereum)**      | Account-Bound Tokens. This MIP adds hierarchical delegation + shielded budgets. |
| **MIP-1**                    | MIP Process and guidelines.                                                     |
| **Compact Standard Library** | Core types and Zswap primitives used throughout.                                |

## Changelog

| Version | Date       | Changes                                                |
| ------- | ---------- | ------------------------------------------------------ |
| `0.1.0` | 2026-07-08 | Initial draft — Private Mandates for Midnight Network. |

## Copyright

This MIP is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
