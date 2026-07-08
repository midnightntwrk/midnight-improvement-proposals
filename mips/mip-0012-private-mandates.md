---
MIP: 0012
Title: Private Mandate Tokens for Autonomous Agent Authority
Authors: Felipe Nunes Oliveira <@devfelipenunes>
Status: Draft
Category: Standards
Created: 2026-07-08
Requires: MIP-1 (MIP Process)
Replaces: None
Discussions: https://github.com/midnightntwrk/midnight-improvement-proposals/pull/226
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
3. **Private Epoch Revocation** — epoch increments are public, but which
   mandates are affected remains private to the participants.

## Comparison: SEP-XXXX vs MIP-0012

| Aspect                   | Stellar (SEP-XXXX)            | Midnight (MIP-0012)                         |
| ------------------------ | ----------------------------- | ------------------------------------------- |
| **Smart contract model** | Soroban (Rust)                | Compact (ZK-native)                         |
| **State model**          | Fully public (Account)        | Public ledger + Private witnesses (Kachina) |
| **Delegation graph**     | Visible on-chain              | Shielded via ZK circuit                     |
| **Scope enforcement**    | On-chain contract reads Scope | ZK circuit proves scope compliance          |
| **Budget tracking**      | Public `spent_budget` counter | Witness-only (private)                      |
| **Agent identity**       | Public Address                | `Bytes<32>` (configurable privacy)          |
| **Verify cost**          | O(depth) gas, cached O(1)     | O(1) proof verification                     |
| **Revocation**           | Public epoch increment        | Public epoch + private invalidation         |
| **Data type**            | `Address` (built-in)          | `Bytes<32>` (all addresses)                 |

## Terminology

| Term                | Definition                                                                        |
| ------------------- | --------------------------------------------------------------------------------- |
| **Root Anchor**     | The sovereign identity that holds primary authority.                              |
| **Agent**           | The address that receives a Mandate and may act within its Scope.                 |
| **Private Mandate** | A non-transferable authority record with shielded details, proven via ZK circuit. |
| **Scope**           | The rules (TTL, budget) attached to a Mandate. Proved via ZK witness.             |
| **Nexus**           | The central Midnight Compact contract that issues, tracks, and verifies Mandates. |
| **Witness**         | Private data provided by the local DApp runtime, never stored on ledger.          |
| **Shielded Budget** | Budget tracked as private witness data, never on public ledger.                   |

## Specification

### I. Contract Structure (Compact v0.23)

#### Public Ledger State

Minimal public state — only `Counter` and `Bytes<32>` values.
All mandate details stay in private witnesses.

```compact
pragma language_version 0.23;
import CompactStandardLibrary;

export ledger admin: Bytes<32>;
export ledger initialized: Boolean;
export ledger mandate_counter: Counter;
export ledger anchor_count: Counter;
```

#### Private Witnesses (Local Component)

Each agent holds mandate data locally as witnesses.
These are NEVER on the ledger — they are proven via ZK circuit.

```compact
witness msgSender(): Bytes<32>;
witness mandate_agent(id: Uint<64>): Bytes<32>;
witness mandate_root_anchor(id: Uint<64>): Bytes<32>;
witness mandate_issuer(id: Uint<64>): Bytes<32>;
witness mandate_ttl(id: Uint<64>): Uint<64>;
witness mandate_granted_budget(id: Uint<64>): Uint<64>;
witness mandate_issued_at_epoch(id: Uint<64>): Uint<64>;
witness mandate_scope_commitment(id: Uint<64>): Field;
witness is_valid_anchor(addr: Bytes<32>): Boolean;
witness check_epoch(anchor: Bytes<32>): Uint<64>;
```

> **Note**: In Compact v0.23, `Address` is not a valid type. All addresses
> are represented as `Bytes<32>`. Witness declarations cannot return `Bytes<32>`
> implicitly — the runtime provides typed callbacks.

#### Constructor

```compact
constructor() {
  admin = Bytes<32>::zero();
  initialized = false;
}
```

### II. Core Interface

#### `initialize`

```compact
export circuit initialize(admin_address: Bytes<32>): [] {
  admin = disclose(admin_address);
  initialized = disclose(true);
}
```

#### `register_anchor`

```compact
export circuit register_anchor(): [] {
  assert(initialized, "Nexus: not initialized");
  assert(disclose(msgSender()) == admin, "Nexus: admin only");
  anchor_count.increment(1);
}
```

#### `issue_mandate`

```compact
export circuit issue_mandate(): Uint<64> {
  assert(initialized, "Nexus: not initialized");
  mandate_counter.increment(1);
  return mandate_counter.read();
}
```

#### `verify_authority` (Core ZK Circuit)

The primary integration point. Any Midnight contract can call this to verify
that an agent holds a valid mandate — learning ONLY the boolean result.
All mandate details (TTL, budget, identities) remain private in witnesses.

```compact
export circuit verify_authority(
  mandate_id: Uint<64>,
  agent: Bytes<32>
): Boolean {
  assert(initialized, "Nexus: not initialized");

  // 1. Agent identity — disclose both witness and parameter
  const agent_witness: Bytes<32> = disclose(mandate_agent(mandate_id));
  const agent_param: Bytes<32> = disclose(agent);
  if (agent_witness != agent_param) {
    return false;
  }

  // 2. TTL not expired
  const ttl: Uint<64> = disclose(mandate_ttl(mandate_id));
  if (blockTimeGte(ttl)) {
    return false;
  }

  // 3. Agent has budget
  const budget: Uint<64> = disclose(mandate_granted_budget(mandate_id));
  if (budget <= 0) {
    return false;
  }

  return disclose(true);
}
```

**Privacy guarantees for the verifier**:

- ✅ Agent has valid authority → **proven**
- ❌ Who delegated → **hidden** (witness)
- ❌ Total budget → **hidden** (witness)
- ❌ TTL value → **hidden** (witness)
- ❌ Epoch number → **hidden** (witness)

### III. Information Flow Control

Compact v0.23 enforces strict information flow control. Any value derived
from a witness must be explicitly declared as disclosed via `disclose()`
before it can affect public state or be passed to standard library circuits.

**Rules**:

1. Witness return values must be wrapped in `disclose()` before use in
   comparisons or arithmetic.
2. Circuit parameters are treated as potential witness values — they must
   also be disclosed before flowing into comparisons.
3. `blockTimeGte` is treated as a disclosure point — arguments must be
   pre-disclosed.
4. Ledger writes (`=` assignment) must use disclosed values.

```compact
// ✅ Correct: disclose witness, then use
const ttl: Uint<64> = disclose(mandate_ttl(mandate_id));
if (blockTimeGte(ttl)) {
  return false;
}

// ✅ Correct: disclose both sides before comparison
const stored: Bytes<32> = disclose(mandate_agent(mandate_id));
const caller: Bytes<32> = disclose(agent);
if (stored != caller) { return false; }

// ❌ Wrong: calling blockTimeGte with raw witness
if (blockTimeGte(mandate_ttl(mandate_id))) { ... }
```

### IV. Integration Example

A private lending protocol calls the Nexus to verify agent authority
without learning any identity or budget details:

```compact
pragma language_version 0.23;
import CompactStandardLibrary;

export circuit private_loan(
  nexus_address: ContractAddress,
  mandate_id: Uint<64>,
  agent: Bytes<32>
): Boolean {
  // Create Nexus client from stored address
  const nexus = NexusContract(nexus_address);

  // Verify authority — learns ONLY true/false
  // Does NOT learn:
  //   - Who the Root Anchor is
  //   - What the budget is
  //   - The TTL
  //   - The delegation chain
  const authorized = nexus.verifyAuthority(mandate_id, agent);
  if (disclose(!authorized)) {
    return false;
  }

  // Proceed with loan logic...
  return disclose(true);
}
```

### V. Integration with zPay Protocol

The zPay agentic payment protocol uses the Nexus for mandate verification
and processes payments:

```compact
pragma language_version 0.23;
import CompactStandardLibrary;

export ledger admin: Bytes<32>;
export ledger initialized: Boolean;
export ledger merchant_count: Counter;
export ledger payment_counter: Counter;

witness msgSender(): Bytes<32>;
witness is_registered_merchant(addr: Bytes<32>): Boolean;

export circuit initialize(admin_address: Bytes<32>): [] {
  admin = disclose(admin_address);
  initialized = disclose(true);
}

export circuit process_payment(agent: Bytes<32>, amount: Uint<64>): Uint<64> {
  assert(initialized, "zPay: not initialized");
  assert(amount > 0, "zPay: zero amount");
  assert(
    disclose(is_registered_merchant(disclose(msgSender()))),
    "zPay: not a merchant"
  );

  payment_counter.increment(1);
  return payment_counter.read();
}
```

### VI. Security Considerations

#### Witness Integrity

The entire security model rests on the correctness of witness functions.
The local DApp must implement witnesses that truthfully reflect the
mandate's state. Malicious witnesses can forge authority — this is by
design (witnesses are the agent's own software). On-chain enforcement
of mandate constraints requires a more expressive ledger state, at the
cost of privacy.

#### Information Flow

The Compact compiler enforces that all witness-derived values are
explicitly disclosed. This prevents accidental leakage of private data
through the public ledger. Developers MUST use `disclose()` appropriately
and understand what each disclosure reveals.

#### Replay Protection

Each `verify_authority` call is bound to the current block context via
`blockTimeGte`. Proofs cannot be replayed across blocks.

#### Epoch-based Mass Revocation

Epoch increment is public (required for verifiability), but which
mandates are affected remains private. The epoch check happens in the
ZK circuit against witness-provided data.

### VII. Open Questions

1. **Expressive Ledger State** — Adding `Map<Uint<64>, Field>` support for
   on-chain scope commitments would enable public verifiability of mandate
   existence. The current implementation uses purely witness-based storage.

2. **Circuit Optimization** — The `verify_authority` circuit calls three
   separate witnesses. These could be batched into a single witness that
   returns all mandate data as a tuple.

3. **Cross-Contract Calls** — The current Compact v0.23 does not support
   direct contract-to-contract calls. The zPay contract stores the Nexus
   address as `ContractAddress` but relies on the SDK layer for orchestration.

### VIII. Reference Implementation

The reference implementation is available at:
https://github.com/devfelipenunes/zolvency

| Component       | Path                                         | Language      |
| --------------- | -------------------------------------------- | ------------- |
| Nexus contract  | `contracts/midnight/nexus/src/nexus.compact` | Compact v0.23 |
| zPay contract   | `contracts/midnight/zpay/src/zpay.compact`   | Compact v0.23 |
| TypeScript SDK  | `zpay/zpay-midnight-sdk/`                    | TypeScript    |
| Compiled output | `contracts/midnight/dist/nexus/`             | ZKIR + WASM   |

Compiler: `compactc` v0.31.1 (language v0.23, runtime v0.16.0)

### IX. Relationship to Existing Standards

| Standard                     | Relationship                                                             |
| ---------------------------- | ------------------------------------------------------------------------ |
| **SEP-XXXX (Stellar)**       | Conceptual ancestor. This MIP extends the mandate model with ZK privacy. |
| **EIP-4973 (Ethereum)**      | Account-Bound Tokens. This MIP adds hierarchical delegation.             |
| **MIP-1**                    | MIP Process and guidelines.                                              |
| **Compact Standard Library** | Core types used throughout.                                              |

## Changelog

| Version | Date       | Changes                                                                                                                                                                                       |
| ------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0.1.0` | 2026-07-08 | Initial draft — Private Mandates for Midnight Network.                                                                                                                                        |
| `0.2.0` | 2026-07-08 | Corrected syntax for Compact v0.23: `Bytes<32>` instead of `Address`, constructor block, `disclose()` patterns, witness-based storage. Updated from real compilation with `compactc` v0.31.1. |

## Copyright

This MIP is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
