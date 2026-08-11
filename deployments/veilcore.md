**dApp name:** VeilCore — Proof of Prior Possession for Plant Genetics

**Contract repository:** https://github.com/hunterincoming/veilcore-midnight-testnet

**Brief description:**

VeilCore records who held a plant cultivar and when, without anyone disclosing the
genetics. A record is hashed client-side; only a domain-separated commitment reaches
the chain. The contract exposes eight circuits across two concerns: provenance
(`anchor`, `proveOwnership`, `pairDna`) and licensing (`issueLicense`,
`countersignLicense`, `revokeLicense`, `proveLicense`, `licenseStatus`). Genetic
preimages, licence terms and counterparties never leave the holder's device — they are
supplied to circuits as private witnesses.

The contract holds no funds and never has. It stores no genetic data, no personal data
and no plaintext of any kind — only 32-byte commitments.

| Category | Self-assessed score (1–3) | Rationale | Mitigations |
|---|---|---|---|
| Privacy-at-risk | 2 | Disclosed values are domain-separated hashes with no recoverable preimage and no identity linkage. A chain observer can see that an address anchored *a* commitment and when, and can correlate repeat activity by that address — timing and counterparty-shaped metadata rather than identity-level data. High-THC cannabis is a stigmatised market, so we do not claim Tier 1. No genetics, no cultivar name, no breeder identity and no licence terms are ever disclosed. | Commitments are `persistentHash` of a private witness with a domain separator; preimages stay client-side. Holders may use a fresh address per record where correlation is a concern. |
| Value-at-risk | 1 | The contract holds no funds. No tokens are deposited, escrowed, pooled or transferred by any circuit. An exploit could produce an incorrect commitment or licence state, not a loss of principal. | N/A |
| State-space-at-risk | 2 | Anchoring is bounded by design and writes **no per-record state**: `anchor`, `proveOwnership` and `pairDna` disclose into the transaction and touch only fixed single-slot fields. One million records add nothing beyond three slots. Licensing retains state only for **live** agreements — `revokeLicense` removes both map entries, so growth is bounded by open business rather than cumulative usage, with clearing initiated by the party who created the entry. | See *Known limitation* below, disclosed rather than omitted. |

---

## Ledger layout

```
export ledger anchorSeq: Counter;                              // fixed
export ledger lastAnchor: Bytes<32>;                           // fixed
export ledger proofSeq: Counter;                               // fixed
export ledger licenseStatusOf: Map<Bytes<32>, LicenseState>;   // live licences only
export ledger licenseRecordOf: Map<Bytes<32>, Bytes<32>>;      // live licences only
```

Three fixed slots and two maps cleared by the circuit that filled them.

## Why anchoring writes no state

An earlier revision stored every anchor in `Map<Bytes<32>, Field>` keyed by commitment,
plus a second map for DNA bindings. That is unbounded growth with no cleanup path, and
we scored it a 3 against this rubric ourselves before submitting.

The fix follows the pattern established in `midnightzk-anchor.md`: the commitment lives
in the transaction, and the chain already retains transaction history. Nothing on-chain
ever read the map back — existence is resolved by querying transaction history through
the indexer, which is where a verifier looks anyway.

`proveOwnership` therefore does not check membership in a ledger map. It emits a dated
transaction disclosing a commitment that only the holder of the preimage can produce. A
verifier compares that proof against the earlier `anchor` transaction carrying the same
commitment; the interval between the two is the evidence.

## Known limitation

A licence that is issued and countersigned but **never revoked** remains in
`licenseStatusOf` and `licenseRecordOf` indefinitely. Terms carry start and end dates
off-chain, but the contract has no notion of expiry, so time alone does not clear an
entry.

This keeps the profile at Tier 2 — bounded per user with a natural ceiling, since a
breeder issues a finite number of agreements and clearing is user-initiated — rather
than Tier 1. We disclose it rather than claiming a bounded-by-design property across
the whole contract.

The mitigation path is an on-chain expiry field permitting a permissionless sweep of
terminal licences. We would rather land that as a reviewed change than assert a Tier 1
we cannot presently substantiate.

## Deployment status

**V2 is deployed to Midnight Preview** at
`dc18e54d2f8031dda0eca1970bb1b1639c1686a14303fe057bb46f07bd0a233b`, deployed 10 August
2026 and exercised end to end:

- `anchor` — `anchorSeq` 0 → 1, `lastAnchor` set to the commitment
- `proveOwnership` — `proofSeq` 0 → 1, commitment unchanged
- `issueLicense` → PENDING
- `countersignLicense` → ACTIVE
- `proveLicense` — accepted
- `revokeLicense` — entry cleared; a subsequent `proveLicense` correctly failed with
  *"No such license"*, confirming removal rather than status flagging

A predecessor contract (V1, with the unbounded maps described above) was deployed to
Preview on 22 July 2026 at
`4a457e6d046928e0faa971d80701b8cd48c3a1283713039444b47fedd0a1f3c7`. It is retained as
historical context and is **not** the subject of this request.

## Source and build

- **Compact source:** `contract/src/veilcore.compact`
- **compactc:** 0.31.1 · **language version:** 0.23
- **Build:** `compact compile src/veilcore.compact ./src/managed/veilcore`
- **Circuits:** `commit` (pure) · `anchor` · `proveOwnership` · `pairDna` ·
  `issueLicense` · `countersignLicense` · `revokeLicense` · `proveLicense` ·
  `licenseStatus`
- **Witness:** `localGeneticSecret()`
- **Design notes:** `docs/design.md` in the repository

SHA-256 of compiled artefacts (`contract/src/managed/veilcore/`):

| File | SHA-256 |
|---|---|
| `keys/anchor.prover` | `a4faa36ae7df32e1d93a7306743c4614417f79ef986ccb59d009a592a91d154f` |
| `keys/anchor.verifier` | `ded343e7eb21a4dc4fbf2b0968020a78e6bd3f35e011badfcfd399e5a0930dd8` |
| `keys/proveOwnership.prover` | `ea345c18b31d593ea364bfa625942e969e80072e87ce16bc863d861b2e02ad6a` |
| `keys/proveOwnership.verifier` | `f4546dce72170047e0205d2350ad60e1b8d47ce39c021b35c48468823b83d424` |
| `keys/pairDna.prover` | `75fdab3affffb26d895743f3944bb61e5af8b8905ab9c075ad815654bf9f7739` |
| `keys/pairDna.verifier` | `82239caa00d04357d67aee25d7c961542f4ac4f9e1ae7876ad0e35732649c5fa` |
| `keys/issueLicense.prover` | `43c571c5019e51fe760333484c331120de092b2d0b60bd9802aa27caef685ee0` |
| `keys/issueLicense.verifier` | `f54af8b124a11c6b32b5ef1d80a33657ebc4e283f620f8dfd15e8725efa43ac9` |
| `keys/countersignLicense.prover` | `a6ca0f2bd609f64e563e7bae8baf3ca5ea4f3bc3480e438d95c98b04bb0d11e3` |
| `keys/countersignLicense.verifier` | `15412d166a9fdfeed5e5d6eaf422b6c000f0533f5a97234c72063ade20d7582f` |
| `keys/revokeLicense.prover` | `91ec53c373a6ffc4c9b901020b14a82640ca88e3366653621bf97e340706e3dd` |
| `keys/revokeLicense.verifier` | `272da16855ad5b879fb729cda19f1d37c549237fd55826751ae5bd5bf1a1cf24` |
| `keys/proveLicense.prover` | `0797c0d90097cf39d8c4c064ed279efa2986b81c5f801c42d384a15ab0d2e84a` |
| `keys/proveLicense.verifier` | `b97e34785d555b2ea93e05ab1007acf3f7cd61cd2f410c26a7d048dae348619f` |
| `keys/licenseStatus.prover` | `a7a5b354733108075915375bccf751e07e24dcf78d55ac218e7289ab6e510f7a` |
| `keys/licenseStatus.verifier` | `79682302bbc3f843f22b26aff54efc95a4405f717fe998ea8c4ac9d4e0ef6284` |

Reviewers can reproduce these by running the build command above and comparing
fingerprints.

## Notes

- License: Apache-2.0.
- The application layer (record metadata service, browser client) is operated
  separately and is not part of this submission. Only the Compact contract is in scope.
