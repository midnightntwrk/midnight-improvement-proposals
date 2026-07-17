---
MIP: X
Title: Private Mandate Tokens for Autonomous Agent Authority
Authors: Felipe Nunes Oliveira (@devfelipenunes)
Status: Draft
Category: Standards
Created: 2026-07-08
Requires: MIP-0001
Replaces: None
Discussions: https://github.com/midnightntwrk/midnight-improvement-proposals/pull/226
---

## Abstract

A pair of standardized Compact smart contract interfaces — **Nexus** (private mandate
registry) and **zPay** (agentic payment vault) — that enable a sovereign identity
to delegate programmable, privacy-preserving authority to AI agents using
zero-knowledge proofs on the Midnight Network.

The Nexus contract issues non-transferable, revocable **Private Mandates** where
the content (agent, budget, expiry) is proven via ZK witnesses without revealing
the underlying values to the ledger. The zPay contract implements a vault-based
payment gateway that authorizes agents to spend from mandate-linked vaults.

Together, they form a privacy-preserving delegation and payment layer for
autonomous agents — conceptually similar to Stellar SEP-XXXX (Hierarchical
Mandate Tokens) but with full ZK privacy.

## Motivation

The emergence of agentic AI systems creates a fundamental trust problem: how can
a human grant an autonomous agent the ability to act on their behalf without
surrendering their privacy?

Existing approaches fall short:

- **Public delegation** (as in Stellar SEP-XXXX) reveals who delegates what to
  whom — creating a permanent, transparent graph of authority relationships.
- **Key sharing** is dangerous — a compromised agent gains unlimited authority.
- **Off-chain permission systems** cannot be verified by smart contracts in a
  trustless manner.

Midnight's Kachina protocol, with its public + private state model and
zero-knowledge proofs, enables a new primitive: **Private Mandates** where the
existence, scope, and budget of a delegation can be selectively disclosed.

This MIP defines a standard that any Midnight Compact contract can integrate to
verify whether an agent is authorized to act — within what bounds — without
revealing the delegation graph, identities, or budget details.

## Specification

### Terminology

| Term                | Definition                                                                          |
| ------------------- | ----------------------------------------------------------------------------------- |
| **Issuer**          | The `Bytes<32>` public key that creates and revokes mandates.                       |
| **Agent**           | The `Bytes<32>` public key authorized to act under a mandate.                       |
| **Private Mandate** | Non-transferable authority record with shielded details, proven via ZK circuit.     |
| **mandate_hash**    | `persistentHash([domain_separator, mandate_id, agent, valid_until, caps, max_val])` |
| **Nexus**           | Compact contract for mandate issuance and verification.                             |
| **zPay**            | Compact contract for vault balances and payments.                                   |

### Nexus Contract

#### Ledger State

```compact
export ledger domain_separator: Bytes<32>;
export ledger issuer: Bytes<32>;
export ledger mandate_commitments: Map<Bytes<32>, Boolean>;
export ledger revoked_mandates: Map<Bytes<32>, Boolean>;
export ledger query_counter: Counter;
```

#### Witnesses

```compact
witness msgSender(): Bytes<32>;
witness mandate_agent_pubkey(id: Bytes<32>): Bytes<32>;
witness mandate_valid_until(id: Bytes<32>): Uint<64>;
witness mandate_capabilities(id: Bytes<32>): Uint<64>;
witness mandate_max_value_per_tx(id: Bytes<32>): Uint<64>;
```

#### Circuits (8 total)

`initialize(issuer_address)` — one-time setup. Computes domain_separator.

`emit_mandate(mandate_id, agent, valid_until, capabilities, max_value)` — issuer-only.
Computes `mandate_hash = persistentHash([domain_separator, mandate_id, agent, valid_until_bytes, capabilities_bytes, max_value_bytes])` and stores in `mandate_commitments`.

`prove_active_mandate(mandate_id, action, value): Boolean` — verifies 7 conditions:

1. Caller is the authorized agent (witness check)
2. Mandate not expired (`blockTimeGte`)
3. Action index within capability range
4. Value within per-transaction limit
5. Computed hash matches an emitted commitment
6. Hash is in `mandate_commitments`
7. Hash not in `revoked_mandates`

`revoke_mandate(mandate_hash)` — issuer-only. Adds hash to `revoked_mandates`.

Query circuits: `is_revoked`, `get_issuer`, `get_domain_separator`, `get_query_count`.

### zPay Contract

#### Ledger State

```compact
export ledger admin: Bytes<32>;
export ledger vault_balances: Map<Bytes<32>, Uint<64>>;
export ledger tx_counter: Counter;
```

#### Circuits (4 total)

`initialize(admin_address)` — sets admin. `tx_counter++`

`deposit(mandate_hash, amount)` — admin-only. Sets `vault_balances[hash] = amount`. Overwrites on re-deposit.

`pay(mandate_hash, amount, recipient)` — anyone with the hash. Deducts from `vault_balances[hash]`. Requires sufficient balance.

`get_vault_balance(mandate_hash): Uint<64>` — returns vault balance.

### Mandate Hash

```text
mandate_hash = persistentHash([
  domain_separator,
  mandate_id,        // Bytes<32>, unique nonce
  agent,             // Bytes<32>, authorized agent
  valid_until,       // Uint<64>, Unix timestamp
  capabilities,      // Uint<64>, action count
  max_value_per_tx   // Uint<64>, per-tx limit
])
```

### Integration Flow

```
Issuer (admin):
  1. initialize → deploy contracts
  2. emit_mandate(id, agent, caps, limit)
  3. deposit(hash, amount) → fund vault

Agent (MCP/AI):
  4. prove_active_mandate(id, action, value) → ZK proof
  5. pay(hash, amount, recipient) → authorize spend
  6. wallet.transferTransaction() → real NIGHT movement
```

## Rationale

**Separation of concerns**: Nexus handles only authority verification (never value). zPay handles only vault balances (never authority). This follows least-privilege design.

**Hash-based commitments**: storing `persistentHash(...)` instead of mandate content keeps all details private. The agent proves possession via ZK witnesses.

**`Bytes<32>` addressing**: Compact has no native `Address` type. All identities are Zswap coin public keys (`Bytes<32>`).

**Deposit overwrite**: a design trade-off to avoid Kachina cell creation costs. The admin manages total allocation off-chain. A future version may add accumulation.

## Path to Active

### Acceptance Criteria

1. Reference implementation compiles with `compactc` and passes E2E tests on devnet.
2. At least one third-party contract integrates `prove_active_mandate`.
3. TypeScript SDK and MCP server documentation exists.
4. DUST sponsorship pattern documented for multi-wallet scenarios.

### Implementation Plan

- **Phase 1** (complete): Reference implementation — contracts compiled, deployed, tested.
- **Phase 2**: TypeScript SDK (NexusClient, ZPayClient).
- **Phase 3** (complete): MCP server for AI agents — 5 tools.
- **Phase 4**: Mainnet deployment and third-party integrations.

## Backwards Compatibility Assessment

This defines a new standard — no existing deployments. All contracts use `Bytes<32>` for identities (standard for Midnight Compact). The `mandate_hash` pre-image is part of the standard and changing it would invalidate all commitments.

## Security Considerations

### Witness Integrity

The security model rests on witness correctness. The Nexus enforces that computed `mandate_hash` matches a stored commitment — binding witness values to an on-chain anchor.

### Hash Preimage Resistance

`persistentHash` (Poseidon-based) provides ZK-friendly collision resistance. Without all pre-images, observers cannot determine mandate content.

### Replay Protection

`prove_active_mandate` uses `blockTimeGte`. Proofs are bound to the current block and cannot be replayed.

### Revocation Privacy

`revoke_mandate` adds an opaque hash to a public map. Observers see only that SOME mandate was revoked, not which one.

## Implementation

Repository: `github.com/devfelipenunes/zolvency`

| Component      | Path                                         | Language          |
| -------------- | -------------------------------------------- | ----------------- |
| Nexus contract | `contracts/midnight/nexus/src/nexus.compact` | Compact `>= 0.20` |
| zPay contract  | `contracts/midnight/zpay/src/zpay.compact`   | Compact `>= 0.20` |
| TypeScript SDK | `zpay/zpay-midnight-sdk/`                    | TypeScript        |
| MCP Server     | `zpay/zpay-midnight-mcp/src/index.ts`        | TypeScript        |

Compiler: `compactc` v0.31.1 (runtime v0.16.0)

## Testing

- `deploy-all.mjs`: deploys + initializes + tests deposit/balance.
- `e2e-simple.mjs`: emit → deposit → pay → balance.
- `e2e-dust-sponsor.mjs`: DUST sponsorship via `balanceFinalizedTransaction`.
- MCP Server: 5 tools validated against devnet.
- Wallet transfer: `signRecipe` + `finalizeRecipe` for real NIGHT movement.

All tests run on Midnight devnet (node 0.22.5, indexer 4.0.2, proof-server 8.1.0).

## References

- [MIP-0001: MIP Process](https://github.com/midnightntwrk/midnight-improvement-proposals)
- [Midnight Developer Docs](https://docs.midnight.network/)
- [Compact Language Guide](https://docs.midnight.network/development/compact)

## Acknowledgements

Thanks to the Midnight Foundation team for the Compact language and Kachina protocol.

## Copyright

This MIP is licensed under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
