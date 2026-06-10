**dApp name:** ShieldUSD

**Contract repository:** Private Moneta deployment assets; not publicly available

**Brief description:**

ShieldUSD is a fiat-backed stablecoin deployment on Midnight with a privacy-preserving shielded note model, public balance support, private-to-public and public-to-private conversion flows, issuer-controlled mint and burn, and compliance controls including pause, freeze, unfreeze, and seizure. This deployment request covers the full current Midnight contract surface.

Strategically, ShieldUSD is exactly the kind of differentiated application Midnight should be able to showcase: a privacy-first stablecoin that demonstrates why confidential programmable assets belong on this network rather than on a conventional transparent chain.

Just as importantly, this contract architecture is not only relevant to ShieldUSD itself. It is a reusable pattern for any token or real-world asset on Midnight that needs privacy, compliance controls, and selective disclosure rather than fully transparent asset behavior.

| Category | Self-assessed score (1-3) | Rationale | Mitigations (if applicable) |
|---|---:|---|---|
| Privacy-at-risk | 2 | ShieldUSD protects sensitive financial activity on Midnight, including private note ownership, transfer relationships, and amounts. A privacy failure could expose financially sensitive metadata and transaction patterns, but the contract is not designed to place identity-level personal data on-chain by default. | The contract uses a shielded note model, encrypted receiver and auditor memo fields, domain-separated commitments, position-bound nullifiers, and authenticated address-pair binding. |
| Value-at-risk | 3 | ShieldUSD represents durable user value that persists on the contract surface across shielded notes and public balances. The blast radius of a critical exploit grows with adoption and circulating value because the deployment covers issuer mint, user transfer, conversion, burn, and compliance flows for the full live asset. | The implementation includes issuer-gated mint and admin mutation paths, pause controls, freeze and unfreeze controls, seizure tooling, owner-bound spending, issuer-bound burn enforcement, and recorded simulator, proof-harness, and live Preview validation. These reduce risk but do not change the underlying value-at-stake classification. |
| State-space-at-risk | 3 | The full contract surface produces substantial persistent state that grows with usage and compliance operations, including the note tree, nullifiers, receiver memos, audit memos, compliance action logs, public audit logs, public balances, allowances, and freeze-related sets. Several of these structures are append-only or steadily increasing without an in-protocol cleanup mechanism. | The implementation avoids loops over unbounded collections in critical paths, uses Merkle proofs and set membership rather than scanning whole histories, and has completed replay and duplicate-note hardening. These measures improve safety and liveness but do not make the overall state footprint low-risk under the rubric. |

## Risk Rationale

### Privacy-at-risk: 2

ShieldUSD is intended to protect user financial privacy on Midnight. If a privacy or proving-system fault surfaced, what would be at stake is not trivial public metadata but private financial amounts, transfer relationships, note ownership patterns, and timing information associated with shielded stablecoin use.

The score is 2 rather than 1 because those exposures are materially sensitive and can support financial surveillance or behavioral inference. The score is 2 rather than 3 because the deployment does not, by default, put identity-grade personal data or direct real-world PII on-chain as part of normal operation.

That same privacy sensitivity is also the reason ShieldUSD is valuable to the ecosystem. A working privacy-preserving stablecoin is a strong proof point for Midnight's core product thesis: confidential value transfer with programmable compliance controls, rather than privacy as a marginal add-on.

It is also a strong proof point for a broader class of Midnight assets. If this contract surface is successful, the lessons and architecture can carry beyond ShieldUSD into other privacy-sensitive tokens and real-world asset deployments that need the same combination of confidentiality, control, and selective disclosure.

### Value-at-risk: 3

ShieldUSD is a live-value stablecoin system, and this request covers the full current contract surface rather than a narrow pilot subset. User value can exist in persistent shielded notes and public balances, while the issuer and compliance surfaces can mutate that value through mint, transfer, conversion, burn, freeze, and seizure-related flows.

Under the rubric, this is high value-at-risk because the severity of a critical bug scales with adoption and outstanding value represented by the deployment. This is not a bounded-time escrow or a zero-custody metadata contract.

At the same time, high value-at-risk is not accidental here; it reflects that ShieldUSD aims to be economically meaningful infrastructure, not a toy privacy demo. If Midnight is going to prove that serious private financial applications can live on the network, a stablecoin is one of the clearest and most ecosystem-relevant categories to validate.

That validation has spillover value beyond one asset. A privacy-capable stablecoin contract with compliance and disclosure tooling is also a practical foundation for other regulated or commercially sensitive assets that may later need to launch on Midnight.

### State-space-at-risk: 3

ShieldUSD maintains multiple categories of durable state that grow with usage over time. The contract stores a historic note tree, nullifier set, receiver memo map, audit memo map, public audit memo family and counters, compliance action history, public balances, allowances, and freeze-related sets for both private and public address targets.

The score is 3 because this is more than ordinary per-holder token balance state. The design includes append-only or steadily expanding structures per interaction and per compliance event, and there is no general cleanup or pruning mechanism that reduces the long-term ledger footprint of those structures.

Even with that score, the architecture is important for Midnight to evaluate in practice because it exercises exactly the kinds of stateful privacy-preserving asset patterns that simpler example contracts do not: shielded ownership, compliant recovery paths, operational controls, mixed private/public asset movement, and selective disclosure for compliance and audit workflows.

## Ecosystem Significance

ShieldUSD is a strong candidate for demonstrating Midnight's distinctive value to users, builders, and partners. A privacy-preserving stablecoin is easier for the ecosystem to understand than many abstract privacy primitives because the use case is immediate: users want stable-denominated value, but they do not want all balances and transfers fully exposed by default.

It also works as a template category for future Midnight deployments. Many tokens and real-world assets will face the same product requirements: privacy for holders and counterparties, explicit compliance controls, and selective disclosure to authorized operators rather than universal public transparency.

From an ecosystem perspective, supporting a project like ShieldUSD would help show that Midnight can host applications that combine:

- practical financial utility
- meaningful user privacy
- explicit operational and compliance controls
- selective disclosure rather than all-or-nothing transparency
- a contract surface that goes beyond toy examples and exercises real product requirements

For that reason, ShieldUSD should be viewed not only as a single stablecoin launch request, but also as an important reference architecture for how Midnight-native private assets can work in regulated or operationally constrained settings.

That does not reduce the need for careful review under the rubric. It does mean that, if the deployment is accepted with eyes open about the risk profile, the upside for the Midnight ecosystem is unusually clear: ShieldUSD would be a visible example of the network enabling something innovative and difficult to reproduce well on a transparent-chain architecture.

## Implementation References

- Primary Midnight contract surface: private ShieldUSD contract with note tree, nullifiers, public balances, conversion flows, and compliance controls
- Included administrative and compliance functionality: pause, unpause, freeze, unfreeze, seizure, metadata updates, and role rotation
- Validation evidence exists in private Moneta implementation records and operator rehearsals; public repository links are intentionally omitted because the production deployment assets are private

## Testing Summary

Private implementation records for ShieldUSD describe completed simulator coverage, proof-harness validation, and live Preview operator rehearsal across the remediated contract surface, including mint, transfer, burn, pause behavior, freeze and unfreeze, seizure, and compliance logging flows. Those mitigations materially improve confidence in correctness, but they do not change the self-assessed rubric outcome that the full ShieldUSD deployment places meaningful financial privacy, durable user value, and significant persistent ledger state at risk.

The advocacy point is therefore not that ShieldUSD is low-risk. It is that ShieldUSD is a high-value, ecosystem-significant Midnight application whose architecture directly showcases the network's privacy capabilities and can inform future privacy-sensitive token and real-world asset deployments, which makes it worth evaluating seriously even under a strict rubric.
