**dApp name:** VeilCore — Proof of Prior Possession for Plant Genetics

**Contract repository:** https://github.com/hunterincoming/veilcore-midnight-testnet

**Brief description:**

VeilCore records who held a plant cultivar and when, without anyone disclosing the
genetics. A record is hashed client-side; only a domain-separated commitment reaches
the chain. The contract exposes twelve circuits across two concerns: provenance
(`anchor`, `anchorBatch`, `proveOwnership`, `pairDna`) and licensing
(`issueLicense`, `countersignLicense`, `proposeTransfer`, `approveTransfer`,
`withdrawTransfer`, `revokeLicense`, `proveLicense`, `licenseStatus`). Genetic
preimages, licence terms and counterparties never leave the holder's device — they are
supplied to circuits as private witnesses.

The contract holds no funds and never has. It stores no genetic data, no personal data
and no plaintext of any kind — only 32-byte commitments.

| Category | Self-assessed score (1–3) | Rationale | Mitigations |
|---|---|---|---|
| Privacy-at-risk | 2 | Disclosed values are domain-separated hashes with no recoverable preimage and no identity linkage. A chain observer can see that an address anchored *a* commitment and when, and can correlate repeat activity by that address — timing and counterparty-shaped metadata rather than identity-level data. High-THC cannabis is a stigmatised market, so we do not claim Tier 1. No genetics, no cultivar name, no breeder identity and no licence terms are ever disclosed. | Commitments are `persistentHash` of a private witness with a domain separator; preimages stay client-side. Holders may use a fresh address per record where correlation is a concern. |
| Value-at-risk | 1 | The contract holds no funds. No tokens are deposited, escrowed, pooled or transferred by any circuit. An exploit could produce an incorrect commitment or licence state, not a loss of principal. | N/A |
| State-space-at-risk | 2 | Anchoring is bounded by design and writes **no per-record state**: `anchor`, `proveOwnership` and `pairDna` disclose into the transaction and touch only fixed single-slot fields. One million records add nothing beyond three slots. Licensing retains state only for **live** agreements — `revokeLicense` removes all four map entries, including any open transfer proposal, so growth is bounded by open business rather than cumulative usage, with clearing initiated by the party who created the entry. | See *Known limitation* below, disclosed rather than omitted. |

---

## Ledger layout

```
export ledger anchorSeq: Counter;                              // fixed
export ledger lastAnchor: Bytes<32>;                           // fixed
export ledger proofSeq: Counter;                               // fixed
export ledger batchSeq: Counter;                               // fixed
export ledger lastBatchRoot: Bytes<32>;                        // fixed
export ledger transferSeq: Counter;                            // fixed
export ledger pendingTransferOf: Map<Bytes<32>, Bytes<32>>;    // open proposals only
export ledger licenseHolderOf: Map<Bytes<32>, Bytes<32>>;      // live licences only
export ledger licenseStatusOf: Map<Bytes<32>, LicenseState>;   // live licences only
export ledger licenseRecordOf: Map<Bytes<32>, Bytes<32>>;      // live licences only
```

Six fixed slots and four maps cleared by the circuits that filled them. A licence
holds at most one open transfer proposal at a time, and `revokeLicense` clears the
proposal along with the licence.

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
- **Circuits:** `commit` (pure) · `anchor` · `anchorBatch` · `proveOwnership` ·
  `pairDna` · `issueLicense` · `countersignLicense` · `proposeTransfer` ·
  `approveTransfer` · `withdrawTransfer` · `revokeLicense` · `proveLicense` ·
  `licenseStatus`
- **Witness:** `localGeneticSecret()`
- **Design notes:** `docs/design.md` in the repository

SHA-256 of compiled artefacts (`contract/src/managed/veilcore/`):

| File | SHA-256 |
|---|---|
| `keys/anchor.prover` | `a4faa36ae7df32e1d93a7306743c4614417f79ef986ccb59d009a592a91d154f` |  <!-- unchanged since approval -->
| `keys/anchor.verifier` | `ded343e7eb21a4dc4fbf2b0968020a78e6bd3f35e011badfcfd399e5a0930dd8` |  <!-- unchanged since approval -->
| `keys/anchorBatch.prover` | `785faa21fa5b1105554a012e46ddceff34adff95b0c2a941e1bb55f23892bdf4` |
| `keys/anchorBatch.verifier` | `fe662bf56906d169dd03dc5ab21ad8684a57274ab4f162726fa75ad9d7e6a9c9` |
| `keys/proveOwnership.prover` | `ea345c18b31d593ea364bfa625942e969e80072e87ce16bc863d861b2e02ad6a` |  <!-- unchanged since approval -->
| `keys/proveOwnership.verifier` | `f4546dce72170047e0205d2350ad60e1b8d47ce39c021b35c48468823b83d424` |  <!-- unchanged since approval -->
| `keys/pairDna.prover` | `75fdab3affffb26d895743f3944bb61e5af8b8905ab9c075ad815654bf9f7739` |  <!-- unchanged since approval -->
| `keys/pairDna.verifier` | `82239caa00d04357d67aee25d7c961542f4ac4f9e1ae7876ad0e35732649c5fa` |  <!-- unchanged since approval -->
| `keys/issueLicense.prover` | `3b86e79ca846a4d2499f5c495f6b4d6155471c646ef04253b0599b3fcf7e5082` |
| `keys/issueLicense.verifier` | `97a546ab71b2925c8794e366b6712e12002b80f4c05a2e9fbf0ceb5ac063d9e4` |
| `keys/countersignLicense.prover` | `d7901d8ad34e53cf88d9098b55537aef545d673751a28a6ff7ee86c4433b74fc` |
| `keys/countersignLicense.verifier` | `dae83066ccf2781725d41a2edd292f12c178c782f701fb79244fb91f3701506a` |
| `keys/proposeTransfer.prover` | `083a73217d359e4870fd8caf37e9501e0ec8f6837abcc02261cff83958084cc8` |
| `keys/proposeTransfer.verifier` | `c759c60f0fbd0062de980efd559819b5a7f0aebb1123b4ca39f3fd01bb6e8c7a` |
| `keys/approveTransfer.prover` | `40554b1b99996729a3575c8a0e34b0189b518262930b792a65bf270da9ebfe7d` |
| `keys/approveTransfer.verifier` | `761b019e6c355dddb20605488f2392bd087a66a28ddada3488428ec48111758b` |
| `keys/withdrawTransfer.prover` | `760e2a98d06afa47f51be04270d5e2bdd7d86b461dedda50aff27923931a4ed9` |
| `keys/withdrawTransfer.verifier` | `2cc689a5b9b14dc3ec6738aca655b3dd177b495f0770429d6b5f13cbec6e8207` |
| `keys/revokeLicense.prover` | `088c0bf340fd3eb1871a4ba11c4698738d870a51c780697d5c8efe9826293759` |
| `keys/revokeLicense.verifier` | `c8723a5229e869d15164587738f4acd08e26eee523a7ba62b8b74fce82cad983` |
| `keys/proveLicense.prover` | `b8ef539e8c04bf11310c1217e870e9c9cde14dc50bcdea94372f10542a95e832` |
| `keys/proveLicense.verifier` | `24776b152dbc4b37a2092eb77131b35dc7f3fe81bee805f23d75f415d807bdb0` |
| `keys/licenseStatus.prover` | `3642dc89f199b94de9899d9bdb53bb2335aca67d8438a99a4c24b57f0827552b` |
| `keys/licenseStatus.verifier` | `26e6485f52de2868ac087a395553537efbdd496189896e3b6a5a9cd0a5d849e8` |

Reviewers can reproduce these by running the build command above and comparing
fingerprints.

## Revision — 24 August 2026

**This document has been corrected after approval, and the subject has grown since
then.** It is recorded here rather than amended silently, because a deployment record
that no longer describes the contract it authorises is worth less than one that says so.

At approval the contract exposed eight circuits. It now exposes twelve. Added since:
`anchorBatch` (batch root anchoring, so one transaction timestamps many records and no
holder needs a wallet), and `proposeTransfer` / `approveTransfer` / `withdrawTransfer`
(permissioned licence transfer — a licence is not a bearer instrument, so the holder
proposes and the issuer approves).

**Four authorisation defects were found in the licensing circuits and fixed.** Max Weber
(ODATANO / NIGHTGATE) compiled the contract, deployed it to preprod, ran every circuit
and replayed them as an attacker, reporting each finding with a transaction hash:
[issue #22](https://github.com/hunterincoming/veilcore-midnight-testnet/issues/22).

- `countersignLicense`, `withdrawTransfer` and `proposeTransfer` took the licence
  commitment as a public argument. That commitment is disclosed by `issueLicense` and is
  a key in a public map, so any observer could activate, cancel or propose against any
  licence. All three now take the secret as a private argument and derive the
  commitment, which is what `proveLicense` already did.
- `approveTransfer` read whichever proposal was pending at execution time. A proposal
  the issuer had seen could be replaced before approval, moving the licence to a party
  the issuer never agreed to. It now names the expected recipient, and `proposeTransfer`
  refuses to overwrite a standing proposal.
- `revokeLicense` left the pending transfer entry behind, contradicting the
  bounded-state claim made above.

The four attacks are retained as regression tests in `contract/test-contract.mjs`: each
passed against the contract as reviewed and each must fail against it now.

**The provenance circuits are unchanged.** `anchor`, `proveOwnership` and `pairDna`
carry the same artefact fingerprints as at approval, marked in the table above. Every
changed fingerprint is licensing.

**Nothing has been deployed to mainnet.** The key has not been requested. We would
rather this document, the review, and the deployed bytes agree before it is.

## Notes

- License: Apache-2.0.
- The application layer (record metadata service, browser client) is operated
  separately and is not part of this submission. Only the Compact contract is in scope.
