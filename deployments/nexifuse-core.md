# Deployment Request - NexiFuse Core

**dApp name:** NexiFuse Core

**Contract repository:** https://github.com/nexifuse/midnight-nexifuse-medical/tree/preprod-migration

**Brief description:**

NexiFuse Core is a privacy-preserving consent registry on the Midnight network. It records, in zero knowledge, whether a given party is currently authorized by a user to access a resource held by an admin-approved third party. No personal data, resource contents, identities, tokens, NIGHT, DUST, or transferable balances are stored on-chain.

| Category | Self-assessed score (1-3) | Rationale | Mitigations (if applicable) |
|---|---:|---|---|
| Privacy-at-risk | 2 | A ZK fault could leak consent-graph metadata, but no protected user data, resource contents, or identities are stored on-chain. Consent metadata can still be regulated personal data, so this is treated as moderate risk. | All identifiers and relationships are domain-separated commitments computed inside ZK circuits. Per-user epoch commitments are identity-bound, revoked permissions are removed from state, and users can rotate epochs to break linkability going forward. |
| Value-at-risk | 1 | The contract holds no tokens, NIGHT, DUST, or transferable balances. There is no `transfer`, `mint`, `burn`, or `withdraw` circuit. The only value represented is access permission. | Users retain unilateral O(1) epoch-level revocation plus targeted per-permission revocation. Admin keys cannot grant permissions on behalf of users. Admin succession is a two-step transfer with cancellation. |
| State-space-at-risk | 2 | Some ledger maps grow over time. Growth is gated by gas costs and ZK-proof requirements, but there is no on-chain rate limit, per-user cap, or expiry sweep. | The third-party allowlist is admin-gated, user-scoped state requires the user's secret to mutate, inserts require fresh ZK proofs and gas, and no circuit iterates over a growing collection. Hot paths use Merkle or set lookups. |

## Risk Rationale

### Privacy-at-risk: 2

No protected user data, resource contents, wallet-balance information, or identities are stored on-chain. Identifiers and relationships are committed to the ledger only as domain-separated hashes computed inside ZK circuits; the underlying values are never disclosed.

The score is 2 rather than 3 because the worst-case leak from a ZK fault is consent-graph metadata, not the underlying medical data itself. Users can rotate keys or epochs to break linkability going forward.

The score is 2 rather than 1 because consent metadata may be regulated personal data in many jurisdictions, so a metadata leak could still create material harm.

### Value-at-risk: 1

The contract custodies no tokens, NIGHT, DUST, or transferable balances. There is no asset transfer, mint, burn, or withdrawal path.

The user holds an always-available recovery path that invalidates outstanding permissions atomically, plus a targeted per-permission revoke that works even if the third party is later delisted. Admin keys cannot grant permissions on behalf of users; admin authority is limited to third-party allowlist management and two-step admin succession.

### State-space-at-risk: 2

The contract has a small number of ledger fields. Growth is constrained because the third-party allowlist is admin-only, user-scoped state requires the user's own secret to mutate, and inserting a new permission requires a fresh ZK proof and gas.

No circuit iterates over a growing collection. Verification and revocation paths use Merkle or set lookups, so adversarial state growth does not degrade verification or revocation cost for honest users.

The score is 2 rather than 1 because there is no on-chain rate limit, per-user cap, or expiry sweep. The score is 2 rather than 3 because growth requires real gas spend per insert, the third-party set is admin-curated, and no operation iterates over the set.

## Implementation References

- Contract: https://github.com/nexifuse/midnight-nexifuse-medical/blob/preprod-migration/contract/src/nexifuse-core.compact
- Witnesses: https://github.com/nexifuse/midnight-nexifuse-medical/blob/preprod-migration/contract/src/witnesses.ts
- Tests: https://github.com/nexifuse/midnight-nexifuse-medical/blob/preprod-migration/contract/src/test/nexifuse-core.test.ts
- CLI and deployment harness: https://github.com/nexifuse/midnight-nexifuse-medical/tree/preprod-migration/cli
- Source deployment request: https://github.com/nexifuse/midnight-nexifuse-medical/blob/preprod-migration/deployments/nexifuse-core.md

The documented toolchain is Compact 0.28.0 and TypeScript 5.8.3.

## Testing Summary

The implementation test suite covers admin transfer, third-party CRUD, epoch rotation, permission grant and revoke, and verification positive and negative cases.

The off-chain witness layer mirrors the contract's hash and Merkle-path logic so on-chain and off-chain commitment derivations do not drift.
