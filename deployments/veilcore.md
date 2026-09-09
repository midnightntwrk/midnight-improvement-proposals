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
| State-space-at-risk | 2 | Anchoring is bounded by design and writes **no per-record state**: `anchor`, `proveOwnership` and `pairDna` disclose into the transaction and touch only fixed single-slot fields. One million records add nothing beyond three slots. Licensing retains state only for **live** agreements — `revokeLicense` removes all three map entries, including any open assignment proposal, so growth is bounded by open business rather than cumulative usage, with clearing initiated by the party who created the entry. An approved assignment is net neutral: one licence is removed and one inserted. | See *Known limitation* below, disclosed rather than omitted. |

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
export ledger licenseStatusOf: Map<Bytes<32>, LicenseState>;   // live licences only
export ledger licenseRecordOf: Map<Bytes<32>, Bytes<32>>;      // live licences only
```

Six fixed slots and three maps cleared by the circuits that filled them. A licence
holds at most one open assignment proposal at a time; `revokeLicense` clears the
proposal along with the licence, and an approved assignment removes the old
licence entirely.

**The map key is the holder.** Only a party who knows the secret behind a licence
commitment can act on that licence, so there is no separate holder field and none
can drift out of step with who actually controls it.

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
| `keys/issueLicense.prover` | `bb6327ed4ac98797f069c42c4f7418238536d5c3a92a7b5a97cb45aec1e613fb` |
| `keys/issueLicense.verifier` | `94e9563f2684222d21fb50727a0e8f0d4a622a061be9d23b50f75166c7b61b6f` |
| `keys/countersignLicense.prover` | `e538299298e05b9920ed6fbcb52a6c3159c0d0ea0a45d4ba3018b8e31d011868` |
| `keys/countersignLicense.verifier` | `fe8ff9309a1c88ec2c24ae0809368690c531dc467053a770b0f6a1aa321d1874` |
| `keys/proposeTransfer.prover` | `0ac26d7531fe20ab3becc2256f88efff7a3369606623bd70ccea22539b02af7e` |
| `keys/proposeTransfer.verifier` | `36f4dd7bbfc70690438af9c7fd604ab20368fc7f140cc485892e376d07d0ee56` |
| `keys/approveTransfer.prover` | `dca1f920cea6197938c9c0b42557af33de3c8755036276b752d04ae50bba83ba` |
| `keys/approveTransfer.verifier` | `5dd49df2634278757e665873917042bc5b279c1376d79b92320ac07fbe715f95` |
| `keys/withdrawTransfer.prover` | `760e2a98d06afa47f51be04270d5e2bdd7d86b461dedda50aff27923931a4ed9` |
| `keys/withdrawTransfer.verifier` | `2cc689a5b9b14dc3ec6738aca655b3dd177b495f0770429d6b5f13cbec6e8207` |
| `keys/revokeLicense.prover` | `21b5e18c10da1b703950fef3e31e6958a044aa5c430d80695dca2077979ce651` |
| `keys/revokeLicense.verifier` | `9133c4ddca10fa4e7930cf5d31a939380e5b525735a832bfa4c4121c9431242c` |
| `keys/proveLicense.prover` | `ee6884c857cbf1403465ef2e41d547977b73cf625c072182bea91ec495db8b16` |
| `keys/proveLicense.verifier` | `8b58154cd6b08e3d865eaa8a1cd76076a61c956e324fa54176ce36923545e805` |
| `keys/licenseStatus.prover` | `314fb3f69e370608db2a2503db31f0c0b553c33bce2f43b7336bcd7d464b2c32` |
| `keys/licenseStatus.verifier` | `21c9dfc7a6e9d17db8d14c1555ac306be9d4d4ab10a1a98188add2bef92aa0a6` |

Reviewers can reproduce these by running the build command above and comparing
fingerprints.

## Revision — 24–25 August 2026

**This document has been corrected after approval, and the subject has grown since
then.** It is recorded here rather than amended silently, because a deployment record
that no longer describes the contract it authorises is worth less than one that says so.

At approval the contract exposed eight circuits. It now exposes twelve. Added since:
`anchorBatch` (batch root anchoring, so one transaction timestamps many records and no
holder needs a wallet), and `proposeTransfer` / `approveTransfer` / `withdrawTransfer`
(licence assignment — a licence is not a bearer instrument, so the holder proposes and
the issuer consents to a named party. See the second correction below: the mechanism as
first written did not achieve this).

**Four authorisation defects were found in the licensing circuits and fixed.** Max Weber
(ODATANO / NIGHTGATE) compiled the contract, deployed it to preprod, ran every circuit
and replayed them as an attacker, reporting each finding with a transaction hash:
[issue #22](https://github.com/hunterincoming/veilcore-midnight-testnet/issues/22).

1. `countersignLicense` took the licence commitment as a public argument. That
   commitment is disclosed by `issueLicense` and is a key in a public map, so any
   observer could activate any pending licence — which removes the bilateral property
   rather than weakening it.
2. `proposeTransfer` was unauthenticated and its write overwrote, so a stranger could
   replace a pending proposal; `approveTransfer` then read whichever proposal was
   pending at execution time, so a proposal the issuer had seen could be swapped before
   approval. Demonstrated in three preprod transactions.
3. `withdrawTransfer` authenticated nobody, so any observer could cancel any pending
   proposal and block an assignment indefinitely.
4. `revokeLicense` left the pending transfer entry behind, contradicting the
   bounded-state claim made above.

The first three now take the licence secret as a private argument and derive the
commitment, which is what `proveLicense` already did. `approveTransfer` names the party
it is approving, and `proposeTransfer` refuses to overwrite a standing proposal.

The four attacks are retained as regression tests in `contract/test-contract.mjs`: each
passed against the contract as reviewed and each must fail against it now.

**The provenance circuits are unchanged.** `anchor`, `proveOwnership` and `pairDna`
carry the same artefact fingerprints as at approval, marked in the table above. Every
changed fingerprint is licensing.

**A second correction, the following morning.** Reading the whole contract after the four
fixes turned up a fifth problem, in the transfer mechanism itself rather than in
its authorisation.

A licence's identity is the secret behind its commitment, and a secret cannot be
un-known. `approveTransfer` reassigned a holder field, which moved nothing: the
outgoing party still knew the secret, so they could still prove the licence and
still propose further transfers, while the incoming party could do nothing unless
the secret was handed over — after which both held it permanently.

The USDA plant variety licence template settles the model. Assignment requires the
licensor's prior written consent, and *the identity of the parties is material to
the formation of this Agreement* with obligations that are *non-delegable*. So the
old licence now ends and a new one begins: the incoming party generates their own
secret and hands over only its commitment, the issuer consents to that specific
commitment, and on approval the old entry is removed and a new one inserted
against the same record. The outgoing party's rights end because the key they hold
is no longer in the map.

`licenseHolderOf` is gone with it. Four regression tests cover the property that
replaced it: the old licence no longer exists, the outgoing party can neither
prove it nor propose another assignment, and the incoming party can prove it.

Also stated rather than enforced: **the licensee generates the licence secret** and
gives the issuer only its commitment. An issuer who generates the secret can also
countersign, which is a licence issued to nobody. The contract cannot check this.

**Nothing has been deployed to mainnet.** The key has not been requested. We would
rather this document, the review, and the deployed bytes agree before it is.

## Notes

- License: Apache-2.0.
- The application layer (record metadata service, browser client) is operated
  separately and is not part of this submission. Only the Compact contract is in scope.
