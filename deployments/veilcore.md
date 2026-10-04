**dApp name:** VeilCore — Proof of Prior Possession for Plant Genetics

**Contract repository:** https://github.com/hunterincoming/veilcore-midnight-testnet

**Brief description:**

VeilCore records who held a plant cultivar and when, without anyone disclosing the
genetics. A record is hashed client-side; only a domain-separated commitment reaches
the chain. The contract exposes thirteen circuits across two concerns: provenance
(`anchor`, `anchorBatch`, `proveOwnership`, `pairDna`, `rotateRecordSecret`) and licensing
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
| `keys/proveOwnership.prover` | `f5a47297aa9ed9d6336e0c6e93295491a87ebb6ac512414e2fe7f900d4a6b9f9` |  <!-- CHANGED, third revision -->
| `keys/proveOwnership.verifier` | `6e767c5c0fe99d0a2a1b1183869d811374eb4e622ba08196eaef99a1604e2f24` |  <!-- CHANGED, third revision -->
| `keys/pairDna.prover` | `f4faf7f469b15e659b3ac562f1e2599592db4f2636e3b8eeefd975e28f347746` |  <!-- CHANGED, third revision -->
| `keys/pairDna.verifier` | `11639f2a848f3948b3696bf16217abacc57ddaef021b87fce03f9ff2474f1f9a` |  <!-- CHANGED, third revision -->
| `keys/rotateRecordSecret.prover` | `01ac3c8e9c1f2136b436d0983a1cb72bcef325a0bae26341036d7aced105822f` |  <!-- NEW, third revision -->
| `keys/rotateRecordSecret.verifier` | `45632e5a11adcaacb2218203f9c86dfe9b809e712e5509d79c31f94740475861` |  <!-- NEW, third revision -->
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

## Revision — 13 September 2026

A third revision, and the first that touches the provenance circuits. The two
prior revisions could say that `anchor`, `proveOwnership` and `pairDna` carried
the same artefact fingerprints as at approval. That is no longer true of two of
them, and the table above marks which.

**What was wrong.** Working through Midnight's own security guide before mainnet,
`proveOwnership` computed `disclose(commit(localGeneticSecret()))` into a local
and never let it reach a public position. The guide is explicit that `disclose()`
clears the compiler's private-data check and does not publish: a value becomes
visible only when it crosses a public boundary, through a ledger write, a return
from an exported circuit, or a contract-to-contract call. The commitment did none
of those.

> **Corrected 16 September 2026.** A return from an exported circuit is *not* a
> public boundary. See *Correction — 16 September 2026* at the end of this
> document.

The public transcript of a `proveOwnership` call was, in full:

```
[ idx path[2], addi 1, ins ]
```

A counter increment. An observer could see that somebody proved knowledge of some
secret and could not tell which record it concerned, which is the entire
evidentiary claim the circuit exists to support. The document approved in August
described it as disclosing the commitment in a dated transaction. It did not.

`pairDna` had the same defect on one side: it wrote the DNA commitment to
`lastAnchor` and dropped the record commitment, publishing a fingerprint attached
to nothing.

**What changed.** Both circuits now return their commitment, which is a public
position. The API and CLI surface it with the transaction hash and block height,
so a verifier receives something to compare against the earlier anchor rather
than a transaction hash and a claim.

> **Corrected 16 September 2026.** This paragraph is wrong and the revision it
> describes did not fix the defect it reported closed. See *Correction — 16
> September 2026*.

**What was added.** `rotateRecordSecret`, in response to the checklist item that a
role must not be permanently lockable by a single lost secret. A witness secret
cannot be recovered from the chain, and a holder who lost theirs previously lost
every record keyed to it with no remedy. The holder proves control of the current
secret and publishes a new commitment; the old one is returned, so the rotation is
a checkable link between two identities rather than an unexplained new anchor.

> **Corrected 16 September 2026.** Returning it published nothing, so under the
> build recorded here a rotation is an unexplained new anchor. See *Correction —
> 16 September 2026*.

Licences issued against the old record are deliberately not re-keyed by this
circuit. Rewriting every agreement attached to a record would move other parties'
rights without their knowledge, so they go through the existing transfer path
where the issuer approves.

**What did not change.** `anchor` and `anchorBatch` carry the same fingerprints as
at approval. Every licensing artefact is byte-identical to the second revision:
sixteen prover and verifier keys across eight circuits, unchanged.

**Two claims in the contract were corrected rather than the code.** The header
described state as bounded by design; the bound is economic. `issueLicense`
requires owning a record, and a record is owned by committing a secret the caller
chooses, so a party holding no material can issue licences to themselves and grow
the maps at the cost of a fee per entry. And `anchorBatch` is unauthenticated by
design, so `lastBatchRoot` holds whichever root anyone wrote most recently. An
inclusion proof is checked against the root in the transaction a holder cites,
never against that slot. Both are now stated in the source.

**Regression coverage.** Three attack cases were added for the new circuit: a
stranger's rotation cannot land on another holder's record, a holder's rotation
returns the identity it replaces, and a rotation to the commitment already held is
refused. Eight adversarial cases now pass in `contract/test-contract.mjs`.

> **Corrected 16 September 2026.** The second of those asserted on the return
> value and passed against a circuit that published nothing — the test carried the
> same defect as this document. Nine adversarial cases pass as of the correction.
> See *Correction — 16 September 2026*.

### The updatability decision

The mainnet readiness checklist asks for this to be settled before the contract
holds value, so it is recorded here rather than made at the console.

The preprod deployment has a maintenance authority nobody chose. `deployContract`
installs a single-signature authority when none is supplied, sampling a key and
storing it in the private state provider, and that is what happened. The store was
encrypted with a password inherited from the example this repository was forked
from — published, therefore not a password. Both are now fixed in the source: the
deploy path requires an explicit signing key or an explicit null, and the password
is read from the environment with no fallback.

For mainnet the authority will be named at deploy time and held jointly, not by
one person and not in a file on a laptop.

It will not be relinquished at deployment. Circuits are bound to the proof system
that compiled them, and an un-upgradable contract cannot be repaired when that
changes — only replaced, leaving every record naming its address pointing at a
contract that can no longer be called. Relinquishing later, once the proving stack
has settled, remains open and is the intended end state: a registry able to
rewrite its own rules is not the neutral thing this format claims to be, and the
transaction that gives up that power is worth more as a public act than the power
is worth holding.

The reason this is a choice rather than a risk is that anchoring is optional by
design. Verification is SHA-256 over a canonical serialisation and requires
nothing from any chain; §3.2 of the specification carries `contractAddress` per
record, so a record names its own deployment and an unanchored record verifies
while stating plainly that its date rests on whoever holds it. A contract that
becomes uncallable degrades new anchoring. It does not invalidate evidence.

**Nothing is deployed to mainnet, and the deploy key issued on 8 September has not
been used.** The preprod deployment at
`fb9c55944908c466dcea7b9807f00ea727b37cebec13870080016ddc5a9d721d` predates every
change in this revision.

---

## Correction — 16 September 2026

The third revision reported a defect and reported it fixed. The defect was real.
**The fix was not a fix, and this document asserted the reasoning that made it look
like one.** Recorded here rather than edited into the section above, on the same
principle that section was written under: a deployment record that no longer
describes what it authorises is worth less than one that says so.

**What this document got wrong.** It listed "a return from an exported circuit" as
one of the boundaries across which a disclosed value becomes public. It is not. A
return travels in the call's communication commitment, which is blinded with
randomness: it reaches the caller's own DApp and nobody reading the ledger. The
third revision then applied that rule — it made `proveOwnership` and `pairDna`
return their commitments and recorded the defect as closed.

Max Weber (ODATANO / NIGHTGATE) compiled the artefact recorded above and searched
each call's `proofData.publicTranscript` — what a `ContractCall` actually carries —
against its `input`/`output`, which only the communication commitment covers:

| Call | Value | Public transcript | Input/output only |
|---|---|---|---|
| `anchor(rec)` | rec | yes | |
| `proveOwnership()` | rec | **no** | yes |
| `pairDna(rec, dna)` | rec | **no** | yes |
| `pairDna(rec, dna)` | dna | yes | |
| `rotateRecordSecret(new)` | old | **no** | yes |
| `rotateRecordSecret(new)` | new | yes | |

So of the build whose fingerprints are recorded above:

- a prior-possession proof is **not** checkable by a third party — the chain shows
  `proofSeq + 1` and nothing else;
- a DNA pairing publishes the fingerprint and **not** the record it binds;
- a rotation publishes the new commitment and **nothing** linking it to the old.

Those are the three properties the third revision reported as restored. The API and
CLI did surface the returned values, which is what made it look right in testing —
but the API and CLI *are* the caller's DApp, and that is the one place a return is
visible.

**The actual fix** is a ledger write per value a verifier needs: four cells,
`lastOwnershipProof`, `lastPairedRecord`, `lastRotatedFrom` and `lastRotatedTo`, in
separate slots rather than reusing `lastAnchor` — because `anchor` proves the
preimage of what it writes there and these circuits do not, so a reader treating one
cell's history as dated possession would collect claims nobody established.

**Status, plainly.** The fix exists in the implementation repository at commit
`63a178e` and is **not** in the fingerprints recorded above. Nor was that build
ever deployed: the preprod deployment named in this document predates every change
in the third revision, nothing is on mainnet, and the deploy key issued on 8
September remains unused. **No deployed contract is affected by this correction.
What is affected is this document.**

**A fourth revision will follow**, carrying new fingerprints for `proveOwnership`,
`pairDna` and `rotateRecordSecret`, and it will be filed before the deploy key is
used and before any mainnet deployment is requested. We would rather file a
correction that says a fix is pending than leave an approved record asserting an
evidentiary property nothing has.

**The rule that catches this class**, and that this document should have applied:
assert the value appears in `out.proofData.publicTranscript`, not in `out.result`.
Applying it found the same defect in our own test suite — the regression case named
*"the rotation names the identity it replaces"* asserted on the return value and
passed against a circuit that published nothing. It now asserts on the ledger cell,
with a separate case recording that the return agrees with it and is not the proof
of it. **Nine adversarial cases** pass in `contract/test-contract.mjs` as of this
correction, up from eight.

