**dApp name:** USDM

**Contract repository:** Private Moneta and bridge deployment assets; not publicly available

**Brief description:**

USDM on Midnight is a bridged, native ledger unshielded token representing USDM transferred from Cardano. On the Midnight side, users only hold and send unshielded balances, and the asset is minted and burned as bridge transfers settle. On the Cardano side, the corresponding asset flow is lock and unlock. Via Labs coordinates that cross-chain mint/burn and lock/unlock relationship.

| Category | Self-assessed score (1-3) | Rationale | Mitigations (if applicable) |
|---|---:|---|---|
| Privacy-at-risk | 1 | The Midnight-side deployment exposes only ordinary unshielded token ownership and transfer activity. There is no private witness data, shielded note model, or identity-level personal data in scope on Midnight for this asset. | Scope is limited to a native ledger unshielded token. No Midnight-side private memo, shielded balance, or custom privacy-sensitive application state is introduced. |
| Value-at-risk | 1 | The Midnight-side deployment is not a pooled vault, escrow, or TVL-bearing application contract. It represents bridged token issuance and burn on the ledger rather than user funds accumulating inside a long-lived Midnight application contract balance. The cross-chain model is mint and burn on Midnight, with lock and unlock on Cardano. | Midnight-side scope is limited to native unshielded token mint and burn tied to bridge operations. There is no separate user escrow or liquidity pool in the assessed Midnight deployment. |
| State-space-at-risk | 1 | The assessed Midnight-side footprint is limited to native unshielded token ledger state rather than a custom application contract with append-only maps, logs, or per-interaction storage growth. | No custom high-growth Midnight application state is introduced as part of this deployment. The design avoids on-chain registries, order books, note trees, or append-only audit logs on the Midnight side. |

## Risk Rationale

### Privacy-at-risk: 1

USDM is explicitly an unshielded Midnight asset in this deployment. The asset does not introduce a separate private balance model, encrypted memo surface, or shielded transaction flow on Midnight. If a privacy fault occurred in the Midnight proving stack, there is no additional private witness layer for this deployment to expose beyond ordinary unshielded token activity that is already intended to be visible.

### Value-at-risk: 1

This assessment is limited to the Midnight-side token deployment. The Midnight component does not act as a user vault, escrow contract, or pooled capital sink whose severity grows with TVL inside a custom application contract. The economic purpose is bridged token issuance and burn on Midnight coordinated with Via Labs bridge operations, while the corresponding cross-chain asset handling on Cardano is lock and unlock rather than long-term custody inside the assessed Midnight contract surface.

### State-space-at-risk: 1

The Midnight-side deployment does not add a custom application state machine with unbounded per-interaction storage. There is no custom append-only event map, note tree, open-order registry, or user-generated storage sink in scope. Relative to the rubric, this is closer to bounded token metadata and native ledger balance state than to a contract that creates materially growing custom ledger state.

## Ecosystem Significance

USDM is narrower than ShieldUSD from a Midnight-contract-risk perspective, but it is still strategically useful for the ecosystem. A bridged, native ledger stable asset gives Midnight users and applications access to familiar dollar-denominated value without requiring a large custom application-state footprint on the Midnight side.

That makes USDM a useful complement to more privacy-intensive asset models on the network: simple where it should be simple, but still important infrastructure for payments, settlement, and broader stable-asset adoption on Midnight.

## Implementation References

- Midnight-side launch architecture: native ledger unshielded USDM only
- Bridge context: Via Labs coordinates mint and burn on the Midnight side with lock and unlock on the Cardano side
- Public contract repository links are intentionally omitted because the production deployment assets are private

## Testing Summary

The key readiness claim for this self-assessment is architectural: Midnight-side scope is intentionally narrow and limited to native unshielded token behavior. Final deployment review should confirm the production authority wiring used by Via Labs for mint and burn on Midnight, the corresponding lock and unlock flow on Cardano, and that no additional Midnight-side custom application state is introduced beyond this assessed footprint.
