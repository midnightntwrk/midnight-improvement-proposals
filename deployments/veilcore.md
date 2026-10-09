**dApp name:** VeilCore — Proof of Prior Possession for Plant Genetics

**Contract repository:** https://github.com/hunterincoming/veilcore-midnight-testnet

> **Revision 4.** This header describes the contract as it is now: one contract, 24
> circuits, source at `ceb3a16`. It replaces the header approved in August and
> revised on 13 September. That header is kept as filed, with headings moved down one
> level and notes added, under *Superseded: the contract to 13 September 2026*. What changed, what was found, and what is still open are in
> *Revision 4* at the end of this document. Revision 4 also adds a second, separate
> contract, the claims contract, described under *The claims contract* in Revision 4.

**Brief description:**

VeilCore records who held a genetic record and when, the licences granted against it,
and the parentage and obligations its holders agreed to. It is for plant and animal
genetics; cannabis is the first market. A record is the commitment to a 32-byte random
secret its holder keeps. The contract never receives genetic data, names, licence terms
or amounts. Parentage and obligations are keyed by the origin. A licence is keyed by the
commitment that issued it and controlled by whichever commitment is that identity's
head. All three survive the holder rotating or recovering their secret.

One contract exposes 24 circuits:

- **Records and identity (7):** `anchor`, `anchorBatch`, `proveOwnership`, `pairDna`,
  `rotateRecordSecret`, `recoverRecordSecret`, `replaceRecoveryCommitment`
- **Licences (8):** `issueLicense`, `countersignLicense`, `proposeTransfer`,
  `approveTransfer`, `withdrawTransfer`, `revokeLicense`, `sealRevocations`,
  `proveLicense`
- **Descent (3):** `proposeParent`, `confirmParent`, `withdrawParent`
- **Obligations (6):** `proposeObligation`, `encumberOwnRecord`, `withdrawObligation`,
  `acceptObligation`, `rejectObligation`, `discharge`

Six pure circuits compute the published hashes and carry no keys: `commit`,
`recoveryCommit`, `licenseCommit`, `licenseKey`, `presentationTag`, `obligationKey`.

The contract holds no funds and never has. It stores commitments, counters, three flags, a
seal time and a protocol version. It also publishes links between commitments: which record descends from which, which record owes something to which,
which record issued a licence, which commitment replaced which.

| Category | Self-assessed score (1–3) | Rationale | Mitigations |
|---|---|---|---|
| Privacy-at-risk | 2 | No genetic data, names, terms or amounts reach the chain. The witnesses are secrets, a verifier's challenge, a record commitment and a tree path. What is public is a graph between pseudonymous identities, with timing: confirmed parentage, obligations and obligation proposals and their beneficiaries, licence issue, activation, transfer and revocation, ownership proofs, rotations and recoveries. That is counterparty and timing data, Tier 2. It is one step from identity-level: holders show their records to buyers and verifiers by design, and anyone who learns who holds one identity can read its whole public history. Cannabis is a stigmatised market, and the rubric lists such markets under Tier 3. We claim 2 because nothing on chain names a party: linking an identity needs its holder to disclose it off chain. Once disclosed, that identity's public history reads as Tier 3. A ZK fault that leaked witnesses would expose record, recovery or licence secrets: control of a record until its holder recovers it, and which licensee stands behind a presentation. | Commitments are domain-separated SHA-256 of random secrets. Obligation terms are hashed with a random salt off chain. A presentation publishes a tag only the verifier can recognise and hides the licence and the licensee. Ownership proofs and presentations answer a verifier's challenge, which verifier rules 5 and 8 require be used once. Holders can split activity across records; rotation does not unlink, and we say so. |
| Value-at-risk | 1 | The contract holds no funds. No circuit receives, holds or sends tokens. Obligations record royalties; they do not move money. An exploit could produce wrong licence, lineage or obligation state, or let someone act as a record's holder until recovery. That can cost money off chain. It cannot drain a balance, because there is none. | N/A |
| State-space-at-risk | 2 | Bounded per identity, growing with the number of identities. Every entry is overwritten, cleared by the party who created it, or capped per anchored identity: at most 16 rotations (reset by recovery) and 16 recoveries, so at most 288 `originOf` entries; 2 parents; 16 obligations in force per record, plus 16 per recovery (at most 272); 8 waiting proposals per proposer; 32 pending and 1024 active licences per issuer. Each anchor writes seven entries that are never removed, and an identity can hold at most 299 permanent entries; fees are paid in DUST, which regenerates, so creating identities is rate-limited, not priced. State grows with the number of anchored records, not with how often anyone calls. Why we still score 2, and why reviewers may read it as 3: *State* in Revision 4. There is no global ceiling on identities, so not Tier 1. Two things are not bounded per identity: the licence tree's root history, cleared only when someone seals (no more often than every 600 s; up to 900 s after the previous seal if its sealer set the bound 300 s ahead), and the total one party can create with many anchors, one fee each. | The caps are enforced in the contract. Waiting proposals, licences and obligations in force each have a clearing move that frees a place; identity and parentage entries are permanent and capped. There is no automated sealer; the operator seals by hand (CLI option 15) after licence activations. Bound table: *State* in Revision 4; full table in `docs/design.md`, *State bounds*. |

---

## Ledger layout

```
export sealed ledger protocolVersion: Uint<16>;            // fixed, set to 1 by the constructor

export ledger anchorSeq: Counter;                          // fixed
export ledger proofSeq: Counter;                           // fixed
export ledger batchSeq: Counter;                           // fixed
export ledger pairSeq: Counter;                            // fixed
export ledger rotationSeq: Counter;                        // fixed
export ledger transferSeq: Counter;                        // fixed
export ledger presentationSeq: Counter;                    // fixed
export ledger sealSeq: Counter;                            // fixed
export ledger descentSeq: Counter;                         // fixed
export ledger obligationSeq: Counter;                      // fixed

export ledger lastAnchor: Bytes<32>;                       // fixed event cell
export ledger lastBatchRoot: Bytes<32>;                    // fixed event cell
export ledger lastOwnershipProof: Bytes<32>;               // fixed event cell
export ledger lastOwnershipChallenge: Bytes<32>;           // fixed event cell
export ledger lastPairedRecord: Bytes<32>;                 // fixed event cell
export ledger lastPairedDna: Bytes<32>;                    // fixed event cell
export ledger lastRotatedFrom: Bytes<32>;                  // fixed event cell
export ledger lastRotatedTo: Bytes<32>;                    // fixed event cell
export ledger lastRecoveredOrigin: Bytes<32>;              // fixed event cell
export ledger lastPresentation: Bytes<32>;                 // fixed event cell
export ledger lastPresentationRoot: MerkleTreeDigest;      // fixed event cell
export ledger lastPresentationUnsealed: Boolean;           // fixed event cell
export ledger lastIssuedLicense: Bytes<32>;                // fixed event cell
export ledger lastActivatedLicense: Bytes<32>;             // fixed event cell
export ledger lastTransferredLicense: Bytes<32>;           // fixed event cell
export ledger lastActivatedRecord: Bytes<32>;              // fixed event cell
export ledger lastDescentChild: Bytes<32>;                 // fixed event cell
export ledger lastDescentParent: Bytes<32>;                // fixed event cell
export ledger lastObligationRecord: Bytes<32>;             // fixed event cell
export ledger lastObligation: Bytes<32>;                   // fixed event cell
export ledger lastBeneficiary: Bytes<32>;                  // fixed event cell
export ledger lastProposedObligation: Bytes<32>;           // fixed event cell
export ledger lastProposedAgainst: Bytes<32>;              // fixed event cell
export ledger lastProposedBy: Bytes<32>;                   // fixed event cell

export ledger originOf: Map<Bytes<32>, Bytes<32>>;         // +1 per rotation or recovery; never removed; at most 288 per identity
export ledger headOf: Map<Bytes<32>, Bytes<32>>;           // 1 per identity that has moved; overwritten; never removed
export ledger recoveryOf: Map<Bytes<32>, Bytes<32>>;       // 1 per anchor; never removed

export ledger licenseStatusOf: Map<Bytes<32>, LicenseState>;   // per issuer at most 32 PENDING + 1024 ACTIVE; removed by revoke
export ledger pendingTransferOf: Map<Bytes<32>, Bytes<32>>;    // at most 1 per active licence; removed by approve, withdraw, revoke
export ledger activeLicenses: HistoricMerkleTree<24, Bytes<32>>; // at most 1024 leaves per issuer, 2^24 in all; roots kept until the next seal
export ledger licenseSlotOf: Map<Bytes<32>, Uint<64>>;         // 1 per active licence; removed by revoke
export ledger licenseAtSlot: Map<Uint<64>, Bytes<32>>;         // 1 per active licence; removed by revoke
export ledger lastSealTime: Uint<64>;                          // fixed
export ledger unsealedChanges: Boolean;                        // fixed

export ledger pendingParentOf: Map<Bytes<32>, Bytes<32>>;      // at most 1 per child; removed by confirm, withdraw
export ledger parentsOf: Map<Bytes<32>, Set<Bytes<32>>>;       // at most 2 per child; never removed
export ledger hasOffspring: Set<Bytes<32>>;                    // at most 1 per identity; never removed
export ledger pendingObligations: Set<Bytes<32>>;              // at most 8 per proposer; removed by withdraw, reject, accept
export ledger openObligations: Set<Bytes<32>>;                 // 16 per record, plus 16 per recovery (at most 272); removed by discharge
export ledger obligationCountOf: Map<Bytes<32>, Counter>;      // 1 per anchor; never removed

export ledger rotationsOf: Map<Bytes<32>, Counter>;            // 1 per anchor; never removed; caps rotations at 16, reset by recovery
export ledger recoveriesOf: Map<Bytes<32>, Counter>;           // 1 per anchor; never removed; caps recoveries at 16
export ledger pendingObligationsBy: Map<Bytes<32>, Counter>;   // 1 per anchor; never removed; caps waiting proposals at 8
export ledger pendingLicensesBy: Map<Bytes<32>, Counter>;      // 1 per anchor; never removed; caps PENDING licences at 32
export ledger activeLicensesBy: Map<Bytes<32>, Counter>;       // 1 per anchor; never removed; caps ACTIVE licences at 1024
export ledger rootsSinceSeal: Boolean;                         // fixed
```

38 fixed slots: the protocol version, ten counters, 24 event cells, the seal time and two
flags (`unsealedChanges`, `rootsSinceSeal`). Nineteen containers. Eight are cleared by the
circuit that ends what filled them. Eleven are never cleared, and each is capped per
anchored identity: `originOf`, `headOf`, `recoveryOf`, `parentsOf`, `hasOffspring`,
`obligationCountOf` and the five counter maps added for the state bounds.

**Event cells are per transaction.** A circuit's return value reaches only its caller, so
every value a verifier needs is written to a cell. Each cell holds the value from the most
recent transaction that wrote it. Verifiers read them per transaction from the indexer,
together with which circuit that transaction called (verifier rule 6 in
`docs/design.md`).

## Source and build

- **Compact source:** `contract/src/veilcore.compact` at `ceb3a16`.
- **compactc:** 0.31.1 · **language version:** 0.23. CI builds with 0.31.1.
- **Build:** `cd contract && npm run compact`, which runs
  `compact compile src/veilcore.compact ./src/managed/veilcore`
- **Fingerprints:** `npm run fingerprints` writes `docs/fingerprints.md`
- **Circuits:** the 24 listed above, plus the six pure hash circuits
- **Witnesses:** `localGeneticSecret()`, `incomingGeneticSecret()`, `recoverySecret()`,
  `licenseSecret()`, `licenseRecord()`, `licensePath()`, `presentationChallenge()`
- **Design notes:** `docs/design.md` (verifier rules, trust model, known limits,
  deployment, governance). Attack history: `docs/security-pass-30sep.md`.

SHA-256 of compiled artefacts (`contract/src/managed/veilcore/`), copied from
`docs/fingerprints.md`: a prover and a verifier key for each of the 24 circuits, the ZKIR
of each circuit in two forms, and the compiled contract code (97 rows).

Built from commit `ceb3a16` with compactc 0.31.1 on the founder's machine (fingerprints committed in `e89a387`). A second build with compactc 0.31.1, without key generation, reproduced the 24 `.zkir` files and `contract/index.js` byte for byte. They were reproduced again on 6 October 2026 from `a3d1884`, with the compiler download checked against the SHA-256 pinned in CI. The proving and verifying keys and `.bzkir` files were built once, on the founder's machine.

| Artefact | SHA-256 |
|---|---|
| `keys/acceptObligation.prover` | `987fbe45b55acc56658b8f0b0c915e7834eb09696af8033a4d54311a8f7c591e` |
| `keys/acceptObligation.verifier` | `ef9db27c4691f23d744cc7fdab7bedc99c8a218f15b59636c9e5622790f4b5a3` |
| `keys/anchor.prover` | `aa537acec7d6d18dbd5d355a8e770b202f6e3551c5a7a533f7a272d1c00a1206` |
| `keys/anchor.verifier` | `198c0740486feba610b9e1c7ae3037399f95d4f157c930b120748d66db5e748b` |
| `keys/anchorBatch.prover` | `530a62ad31a19e93df7e58efd684db79a72742ae6362694e65e62b26a0eab954` |
| `keys/anchorBatch.verifier` | `8ae5b4c15c3a20993e8d34b4022d180d95ed2d88927fca184527ca6857f06104` |
| `keys/approveTransfer.prover` | `33fb0354d3d92071ad977aa12d1318d02d203e1146d719b9b524ad40467e29fa` |
| `keys/approveTransfer.verifier` | `881a484ee1db8b1308da672e921d7035908b7a7e7be321cb34266afc3b09ae1b` |
| `keys/confirmParent.prover` | `0a6bd6a0b39a2b3597bfc6eadf1e0d710f4f052b91b72aa2d475565f343a7265` |
| `keys/confirmParent.verifier` | `39f8c5e68b5b73b6df5a4842c0a9d856b405412f49067b8df062baf18b33e7a1` |
| `keys/countersignLicense.prover` | `951fd5a73d674e2021f96e3d4da411853d0af943840a41fdf3747c7adbbc6a15` |
| `keys/countersignLicense.verifier` | `71149411b93b5f7dea286a4a651d5bc35f2a28f3e07f23ffc0c31cbb4779df68` |
| `keys/discharge.prover` | `4fe5207ccff22c23ee9792b3bfa4409a265ecbf35aca4a7f9a0814fdbd00e516` |
| `keys/discharge.verifier` | `6e5c082a07834d71bdcef4b00209761da598674625df29209cdb34589c1f2bfe` |
| `keys/encumberOwnRecord.prover` | `0a631517bb08a8b3eec793246d10cd948a7bbf689b4f703077efec428bf17aca` |
| `keys/encumberOwnRecord.verifier` | `64450fe400ff5b17ae07d5a5f619f7aa3598f6601a7a8e27d10c1fd8a1d4c743` |
| `keys/issueLicense.prover` | `1cc2311835932c6428810fb88b73aa3165e6ec1de037ef7c7754b2b4bd82e449` |
| `keys/issueLicense.verifier` | `7fd3f48b02763c7a99f7cb7571a3fe5923e37b08c1bc57f8fe6ec78ca16b952e` |
| `keys/pairDna.prover` | `bcc2b1cde706a038d6aee96704929d6e0c812b7abac11f5b426cc983576f1090` |
| `keys/pairDna.verifier` | `be1f3fb649332b832548a23b6c873a28a2fa21e1227c8501eec8b594466fbe4c` |
| `keys/proposeObligation.prover` | `4aba5096018525521b7522ab584256a361c7cbd7c85670292751f176fa068793` |
| `keys/proposeObligation.verifier` | `587f78f979b38bd46e9d0a2e210f54b819ca28253b04e4a97ec0f5632b879dfd` |
| `keys/proposeParent.prover` | `963e52a4b2227ea079f1a40fe517391dd994f59c18614e340fcf64c31d43afc4` |
| `keys/proposeParent.verifier` | `4eeb3cc9f91246278c7ba5f6a99148da06eec56ec5094cd2bd6c8211661c7a64` |
| `keys/proposeTransfer.prover` | `2f1efaac8919ab4c213a12d07c073486aa4dc87cae854e4b66c76a9c84a4c8a6` |
| `keys/proposeTransfer.verifier` | `b74d32cd0cbc9d9aa0ada29badca4dd315796dbb43c4120d1d8d40adb58ab686` |
| `keys/proveLicense.prover` | `9e191bb84a5ca5a771da0265297c734e09be3fae46a303fc3a76dbe64394d11a` |
| `keys/proveLicense.verifier` | `b338dda3501ae2dec26b241b353495580debcb359455b0671b6619197f29d59b` |
| `keys/proveOwnership.prover` | `bcaabf722cf456ee06d362b88b50bcf2dc8e90f16659ed17268afab507fad329` |
| `keys/proveOwnership.verifier` | `348ea64c7c532fa9ab4d1ad9097de2d2f0f3e4dd4b9369bda4d0fcdc7ae38462` |
| `keys/recoverRecordSecret.prover` | `b70e052751d77e607ff7a536a5e6d3fd618e0c528cb196ce658091fc69579e8b` |
| `keys/recoverRecordSecret.verifier` | `d31abbf23545e823fcbac5211812e4e8102083055a120da590769d63f5ee89ed` |
| `keys/rejectObligation.prover` | `c9a94cd60339368c3559a5cdf119384532da65758ab705780285c42ab3576f1c` |
| `keys/rejectObligation.verifier` | `61b2301586c3c9bf4a283a0774d6b7ace2782916854e1d4fcfeb534beb08b282` |
| `keys/replaceRecoveryCommitment.prover` | `30a84ee352fd383026907272a42afa4f366b6c9bb2185238f26344e53958445c` |
| `keys/replaceRecoveryCommitment.verifier` | `8e694ea480f2d9227c5a89c8d75ae570119124296819350e85207fb3d77483f6` |
| `keys/revokeLicense.prover` | `4bf25562b50ae3f9b95281849a4c14bba5bd3a2d5a23fd86f86bedf13d00284f` |
| `keys/revokeLicense.verifier` | `56b8ab4be3548195a8eb2dc56a8f678a2f29ef0ffd8f4f97793593eac3333c83` |
| `keys/rotateRecordSecret.prover` | `c57c6146dfe5f4319026704157726f42eb1c9da8bdeddb422e6680d2d5f4a0a8` |
| `keys/rotateRecordSecret.verifier` | `e078ccbafad46cd43a3157eb41b715dc0cfcdefbe3a46761ce38b214996e5d71` |
| `keys/sealRevocations.prover` | `4ebaceaad233bebb5cfb0763de213d205a36ad1c35a00cb93120911501f14d1a` |
| `keys/sealRevocations.verifier` | `694d4cbcc5d865550f01af1d9b03ade91690a5e5a41b221a713f1f87b02d7d06` |
| `keys/withdrawObligation.prover` | `09111b2983933b67102ec9f85952661748bd5128818b11ff16b161f987e552bd` |
| `keys/withdrawObligation.verifier` | `9757f321082f0239b499c22004d1b66a6b9051607a549175f249c9dbc404a27a` |
| `keys/withdrawParent.prover` | `6df81f4c4df374efa3efb010a0ef27d3f915d426fcf9e200897c834f307b15fc` |
| `keys/withdrawParent.verifier` | `7cb48b4cf19a9f2a1aa8890678f499c3ab5a0dc592799a71c1a3d3181aca1068` |
| `keys/withdrawTransfer.prover` | `df817d4568d0bf186f9c4483690090660c0a064be02e94322c7c786ae0aeb728` |
| `keys/withdrawTransfer.verifier` | `8ac9fb09ff1f7e74a663faa7b4369fbd7ee5011e3709015e28e0567f9954803f` |
| `zkir/acceptObligation.bzkir` | `f2a767fb36ff57908e721e519fff520ae0fb997175be749d4f0c36daa1692ee5` |
| `zkir/acceptObligation.zkir` | `a7e7d862fa96d68055efffafa831e7e545600742cd3d405b76815c86985c4ec0` |
| `zkir/anchor.bzkir` | `eda2c09ca15e01855e1a017c3d200988043a250b02f2335d63d2b9613582c76b` |
| `zkir/anchor.zkir` | `bac1066f7a62ce2c7e560068e41b00319e62b85741b4750aee11f2934d5a1d4e` |
| `zkir/anchorBatch.bzkir` | `a0e18c980c17127ae64aeb12149f8ef4d78af37333cb4e9e100cbad732b70006` |
| `zkir/anchorBatch.zkir` | `5c5b7cb86dffa017b359bccb05ab67c67c85b9e84249d9bfea8162d96e559eaf` |
| `zkir/approveTransfer.bzkir` | `b42b5bf329d2f056eacf9f0207357a255defb1a1d8656f2f73e08e81727d6daf` |
| `zkir/approveTransfer.zkir` | `de731f03a94b1f3de76dfb45a1a8e0b00bcf46719c104d85a4e4e281275306cd` |
| `zkir/confirmParent.bzkir` | `e36fb986a0e9c123a089cdfb72202cf287cb95a6d6650e12a656f194bbfe87a6` |
| `zkir/confirmParent.zkir` | `b06a315cddef0b71c47814cbfb6f24fc8be945826deb9f87b57d8dfe06905990` |
| `zkir/countersignLicense.bzkir` | `482a7c264537602fd18dd66af1a1ee53e1a06206fe4e2401e3398bd2705aac59` |
| `zkir/countersignLicense.zkir` | `26ffe4d5a07829a43ada1fd55b0b47572e250cd9b02289df222fe9d7e528f4a0` |
| `zkir/discharge.bzkir` | `58a6bd0cd2eda514a5fa56b63918aded099f50483f0f3e1c7f0b73a341e8242b` |
| `zkir/discharge.zkir` | `37d4b2c0ac0f3f51fdf279b051ca2a13b96cbd5aa3a7b5e82947a57c96db1a3e` |
| `zkir/encumberOwnRecord.bzkir` | `64fe84b4dd663fad815794e1a2e1df17b8f678196bfe2d41693cd3c1d26378ac` |
| `zkir/encumberOwnRecord.zkir` | `94cdfcbf81987bcae61492737a3586838a05fd51ae5d360e805ab892b5a6a749` |
| `zkir/issueLicense.bzkir` | `ba0949cfaedd41dd02f98f7ae367b234e71afddb0737200c9b61902252661768` |
| `zkir/issueLicense.zkir` | `4f3baed4be5154a5e5a3c65c1208570c33c21b325f328a72edf3f35e99cf06f1` |
| `zkir/pairDna.bzkir` | `25c249d3fc744988ff76669731db293bf80a88c0cba0c9bd09927f827ea67676` |
| `zkir/pairDna.zkir` | `0dcd742a2efee30a9b9eaae451179e0af0fa935c6493fb4c919fdb6e3e0293f4` |
| `zkir/proposeObligation.bzkir` | `3abc7f4ba2a3dea1a23b1bf6cc409418a3ac14a4f2ee961fd32dcf8dcb5c5f01` |
| `zkir/proposeObligation.zkir` | `2273be490bf9ec7d9a615f80c319fb0d0873d3678c99f85a45b438f16d1f3a05` |
| `zkir/proposeParent.bzkir` | `3364503895ef390606c8fd9fffee95ec8d1200a5fad6c04ef370e7406f5712d9` |
| `zkir/proposeParent.zkir` | `fc7f235c1af5a2e1bb32c38260b07a87f9301dd6cb8807c747c6584d6c2d1d4a` |
| `zkir/proposeTransfer.bzkir` | `b5813fec8bec1ce5babadd56a61060a5d71dbe9cb2d7e64e04daf7d5d61383b7` |
| `zkir/proposeTransfer.zkir` | `a5cfd2b2cb5f11cfaad4e89c25ffea9f2218f0113bcd4c223bbeef5988c4a1b5` |
| `zkir/proveLicense.bzkir` | `50428b86cecdb5d759a9a1af52d5b3998d95fd091e231ebf716226631954e0a8` |
| `zkir/proveLicense.zkir` | `bc5ae6bf3cceabcbe9a51bc6b4337ee724997461a7ac067ec5e6bdb050c36d6a` |
| `zkir/proveOwnership.bzkir` | `27400218fe6aa22ae3e7c22a7f97ecb8122f9a2ded9d426140b790fe2c2cb71e` |
| `zkir/proveOwnership.zkir` | `36526d88747e918c88cb0f664954268c384fca2165b50a3e714b21c2338391d3` |
| `zkir/recoverRecordSecret.bzkir` | `a37ff42c633bd2b659198b946e79ac6cf0e4bbe654b9e6f6a0544e8be86e4767` |
| `zkir/recoverRecordSecret.zkir` | `e1c1292e1adc75469284301467bc4f6aa550bc246e9a499d8b761ea137480983` |
| `zkir/rejectObligation.bzkir` | `965216e42fc22bde45b9a2d0b95c8ba2224d71a6eb456c5c001366613912564e` |
| `zkir/rejectObligation.zkir` | `7f918dd6362d3dbc8328fb9672a6cd61aaf42a26c7dcc54665888bb2600c31e9` |
| `zkir/replaceRecoveryCommitment.bzkir` | `d7c0ce21b113a5bbb188e2b46496d5e062b490a5f784d1663e7037f93377cde4` |
| `zkir/replaceRecoveryCommitment.zkir` | `11177e8f616fd4f1256420df9f272ae6e535f2328d15936b13150419289a77ae` |
| `zkir/revokeLicense.bzkir` | `0e860a2acac01b277c9c840c2d508dd454904cbd7c13cd487afef437026adb08` |
| `zkir/revokeLicense.zkir` | `a24609ff839e4f4cf43851d6c77afaf9465d60582369e3739cb42177fd73b9c0` |
| `zkir/rotateRecordSecret.bzkir` | `534cb3558d749b993673e0f9f1aed970dcfd369fa4db729f2c328cc656c9993d` |
| `zkir/rotateRecordSecret.zkir` | `76d217a6a7587e85467ff1905272682eaf54dbd7969f676d99f306a401066fb7` |
| `zkir/sealRevocations.bzkir` | `36dfee86566f1569f10f1464e06ce976436dc8a90c0b5c56aeedd1eb97a14cd1` |
| `zkir/sealRevocations.zkir` | `734fb049584993d6b4ec6e97b6ab0c399a0ed0c27613372c91a7cea483b1b78a` |
| `zkir/withdrawObligation.bzkir` | `f735f8f4cd3e226693e7b0ec8045e294655c2737f5a0ca1b74c4d9339c688d0a` |
| `zkir/withdrawObligation.zkir` | `a0550dc303b29b5e3111e108548c8faf5a84b581b6fa8c92d6fcc6763e9b8f7e` |
| `zkir/withdrawParent.bzkir` | `59c1fd185a89fdfca9ffde3646bd086e6fad19fa137a0e2a8c28a2937becfbfc` |
| `zkir/withdrawParent.zkir` | `b8ef43b8ad128c9b05c71cbe313fbc1e34ca7bd7427393b4a21e0618e8c72a35` |
| `zkir/withdrawTransfer.bzkir` | `a6cc18da40ece2ee34c8cfb93c87683ff010e320dfc8a4f9c57ead589f6d23b7` |
| `zkir/withdrawTransfer.zkir` | `64cc7b9bb7b3f13cb206126548394e3156ae5ee88b9877943411e200879d9a99` |
| `contract/index.js` | `4e23ffc28f3de3ce670d9cea2896301dfeec8b4593d1b65332eb18f74ba5d286` |

Reviewers can reproduce the table by building at `ceb3a16` with compactc
0.31.1, running `npm run fingerprints`, and comparing. A mainnet deploy or join from the
CLI is refused unless the local build matches `docs/fingerprints.md` as committed
(`bboard-cli/src/keys-check.ts`).

---

## Superseded: the contract to 13 September 2026

> **Superseded by Revision 4.** Kept as filed, with headings moved down one level and
> notes added. It describes the 13-circuit contract
> (eight circuits at approval) and its fingerprints. None of it is the build Revision 4
> asks to approve. Where a statement below is now false, a note says so.

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

> **Superseded, Revision 4.** These scores describe the 13-circuit contract. Revision 4
> re-scores the current one: privacy 2, value 1, state-space 2 (3 before the state bounds
> were added; see *State* in Revision 4).

---

### Ledger layout

> **Superseded, Revision 4.** This is the 13-circuit layout. The current one, with
> nineteen containers, is at the top of this document.

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

### Why anchoring writes no state

> **No longer true, Revision 4.** The current `anchor` writes seven entries per record
> (`recoveryOf` and six counters) that nothing removes, and rotations, recoveries and
> parentage add more, each capped per identity. See *State* in Revision 4.

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

### Known limitation

> **Still true, Revision 4**, now capped: an issuer can hold at most 1024 active
> licences. See *State* in Revision 4.

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

### Deployment status

> **Revision 4.** None of these deployments is the build Revision 4 asks to approve.

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

### Source and build

> **Superseded, Revision 4.** These fingerprints are not the build being approved. The
> current table is at the top of this document.

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

> **Changed in Revision 4. Approved by Hunter Roberts on 6 October 2026; Makoto Steiner to
> confirm.** The implementation installs one signing key, so the authority is not held
> jointly at launch. It is held on paper, two copies, one per founder: see *The
> maintenance authority* in Revision 4.

It will not be relinquished at deployment. Circuits are bound to the proof system
that compiled them, and an un-upgradable contract cannot be repaired when that
changes — only replaced, leaving every record naming its address pointing at a
contract that can no longer be called. Relinquishing later, once the proving stack
has settled, remains open and is the intended end state: a registry able to
rewrite its own rules is not the neutral thing this format claims to be, and the
transaction that gives up that power is worth more as a public act than the power
is worth holding.

> **Changed in Revision 4. Approved by Hunter Roberts on 6 October 2026; Makoto Steiner to
> confirm.** The maintenance policy (`docs/maintenance-policy.md`) drops relinquishment as
> the intended end state and sets no retirement date: the end state becomes custody by
> independent parties under published rules. See *The maintenance authority* in Revision 4.

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

> **Revision 4.** The fix is in the build Revision 4 describes. Commit `63a178e` is not
> in the history of `main`, which now begins at `f652ae2` on 27 September.

**A fourth revision will follow**, carrying new fingerprints for `proveOwnership`,
`pairDna` and `rotateRecordSecret`, and it will be filed before the deploy key is
used and before any mainnet deployment is requested. We would rather file a
correction that says a fix is pending than leave an approved record asserting an
evidentiary property nothing has.

> **Filed as Revision 4**, at the end of this document. Every artefact's fingerprint
> changed, not only those of these three circuits.

**The rule that catches this class**, and that this document should have applied:
assert the value appears in `out.proofData.publicTranscript`, not in `out.result`.
Applying it found the same defect in our own test suite — the regression case named
*"the rotation names the identity it replaces"* asserted on the return value and
passed against a circuit that published nothing. It now asserts on the ledger cell,
with a separate case recording that the return agrees with it and is not the proof
of it. **Nine adversarial cases** pass in `contract/test-contract.mjs` as of this
correction, up from eight.

> **Revision 4.** `contract/test-contract.mjs` was replaced by the Vitest suites in
> `contract/src/test/`. How the rule is applied there is in Revision 4.

---

## Revision 4 — 6 October 2026

Prepared 6 October 2026. Filed when both founders have signed off (*Founder sign-off*,
at the end of this revision), before any mainnet deployment.

This is the fourth revision the 16 September correction promised. It is filed before the
deploy key is used and before any mainnet deployment is requested, as that correction
said it would be.

It describes a different contract from the one approved. At approval the contract had
eight circuits; on 13 September it had thirteen. It now has 24, in one contract that
covers records, licences and lineage. Every artefact fingerprint has changed. The header
at the top of this document now describes this contract. What it replaced is kept under
*Superseded*.

### What is being approved

- `contract/src/veilcore.compact` at `ceb3a16`, built with compactc 0.31.1.
- The 97 artefacts in the fingerprint table at the top: 48 keys, 48 ZKIR files and
  `contract/index.js`.
- Deployed in fragments, as described below, with a maintenance authority kept at
  launch.
- A second contract, `contract/src/veilcore-claims.compact` (5 circuits), built at
  `c75c155` with compactc 0.31.1, with its own 21-row fingerprint table (committed in
  `765cab1`), deployed with **no** maintenance authority (an empty committee). See *The
  claims contract* below.

Neither contract's source has changed since: `veilcore.compact` was last changed in
`ceb3a16`, `veilcore-claims.compact` and `schnorr.compact` in `cd30c11` and `74e529c`, and
all three are the same on `main` at `a3d1884` (6 October 2026).

### The 16 September defect is fixed

The correction found that `proveOwnership`, `pairDna` and `rotateRecordSecret` returned
the values a verifier needs, and a return is not public. The fix it described is in this
build:

- `proveOwnership` writes the caller's live record to `lastOwnershipProof`.
- `pairDna` writes the record to `lastPairedRecord` and the DNA commitment to
  `lastPairedDna`.
- `rotateRecordSecret` writes the old commitment to `lastRotatedFrom` and the new one to
  `lastRotatedTo`. `recoverRecordSecret` writes the new one to `lastRotatedTo`, the
  origin to `lastRecoveredOrigin`, and zeroes `lastRotatedFrom`.

These are separate cells from `lastAnchor`, as the correction required. `pairDna` no
longer writes `lastAnchor` at all.

One more change to `proveOwnership` came out of review (round 9). It now takes the
verifier's challenge and writes it to `lastOwnershipChallenge`. Without it, anyone could
point a verifier at the real holder's proof and present it as their own. Verifier rule 8
in `docs/design.md` says how to check one.

**Is the rule applied?** The correction's rule was: assert the value is public, not that
it came back. The tests now check these values in ledger state after the call, never in
the return value (`records.test.ts`, `attack-round3.test.ts`,
`rules-coverage-round11.test.ts`). A ledger cell changes only through the call's public
transcript, so this checks what a third party sees. The smoke test reads
`lastOwnershipProof`, `lastPairedRecord` and `lastPairedDna` back from the chain through
the indexer. It does not read the rotation cells back from the chain. The literal form of
the rule, searching a call's public transcript, is used for `proveLicense`, to check
what must *not* appear there (`attack-round4.test.ts`), and for `issueLicense`, to check
that the licence commitment *does* appear (`rules-coverage-round11.test.ts`,
`attack-bounds.test.ts`).

Two references in the correction no longer resolve. Commit `63a178e` is not in the
history of `main`, which now begins at `f652ae2` on 27 September. And
`contract/test-contract.mjs` was replaced by the Vitest suites in `contract/src/test/`.

### What changed since 13 September

- **One contract.** Lineage was a separate contract. It could not see rotations or
  recoveries in the provenance contract, so a beneficiary who rotated could never
  release what they were owed. It was merged in and keyed by identity.
- **Identity.** A record is an origin (anchored) or a successor (moved to by rotation or
  recovery). `originOf` maps a successor to its origin; `headOf` maps a moved origin to
  its live head. Only the head can act. Parentage and obligations are keyed by the
  origin. A licence is keyed by the commitment that issued it and controlled by
  whichever commitment is that identity's head. All survive every move. Moves are capped (see *State*).
- **Recovery.** `anchor` now takes a recovery commitment. `recoverRecordSecret` moves the
  identity with the recovery secret, whoever holds the head, including a thief. It
  installs a new recovery commitment in the same call, so a recovery secret works once.
  `replaceRecoveryCommitment` swaps a recovery secret that may have leaked.
- **Licences.** An entry is keyed by `licenseKey(licence, issuing record)`, so a licence
  names its issuer. Active licences are leaves of a `HistoricMerkleTree<24>`. Revocation
  and transfer clear a leaf at once; older roots stop verifying at the next
  `sealRevocations`, which anyone may call no more often than every 600 s; up to 900 s
  after the previous seal if its sealer set the bound 300 s ahead. `licenseRecordOf` is gone, and so is the `licenseStatus` circuit, which leaked
  what was looked up.
- **Presentations.** `proveLicense` takes the licence secret, its record, its tree path and
  the verifier's challenge as witnesses. It publishes a tag only the verifier can recognise,
  the root it proved against, and whether a revocation was waiting.
- **Descent.** `proposeParent` by the child's holder, `confirmParent` by the parent's.
  Once a record has confirmed offspring, its own parents are fixed, so no cycle can form.
- **Obligations.** The beneficiary proposes, the record's holder accepts or rejects, and
  only the beneficiary releases. A holder can encumber their own record in one step.
- **Every caller is derived from a secret.** No circuit takes the caller's record, or
  any secret, as an argument. Seven witnesses replace the one in the 13 September build.
- **Versioned hashes.** Every tag reads `veilcore:v1:...`. The contract publishes
  `protocolVersion = 1`. Test vectors are in `contract/vectors/v1.json`.
- **State bounds (1 October, evening).** Caps per anchored identity on rotations,
  recoveries, parents, obligations, waiting proposals and pending and active licences,
  kept in five counter maps created at anchor. `issueLicense` and `proposeObligation`
  now publish what a recovered owner needs to clear a thief's entries. A seal is allowed
  when only activations changed the licence tree. See *State*.

### State

At approval this document scored state at 2 because anchoring wrote no per-record state
(*Why anchoring writes no state*, under *Superseded*). That is not true of this contract.

**How the score went from 3 to 2.** We first scored the previous build against the
rubric and it scored 3: anchors, rotations, recoveries and confirmed parents added
entries nothing removed, an identity could rotate or take parents without limit, and
pending licences and obligation proposals had no limit either. Under the override rule
that blocks deployment. Rather than file at 3, we changed the contract so that every
entry is overwritten, cleared by the party who created it, or capped per anchored
identity. The bounds, summarised (the full table is in `docs/design.md`, *State
bounds*):

| Field | Bound | Cleared by |
|---|---|---|
| 38 fixed slots (version, 10 counters, 24 event cells, seal time, 2 flags) | one value each | overwritten |
| `recoveryOf` and six counters (`obligationCountOf`, `rotationsOf`, `recoveriesOf`, `pendingObligationsBy`, `pendingLicensesBy`, `activeLicensesBy`) | 1 each per identity, made at anchor | nothing |
| `originOf` | at most 288 per identity: 16 rotations (`MAX_ROTATIONS`), reset by each of at most 16 recoveries (`MAX_RECOVERIES`) | nothing |
| `headOf` | 1 per identity that has moved; overwritten | nothing |
| `parentsOf` | at most 2 per child (`MAX_PARENTS`) | nothing |
| `hasOffspring` | at most 1 per identity | nothing |
| `pendingParentOf` | at most 1 per child | `confirmParent`, `withdrawParent` |
| `licenseStatusOf` | per issuer, at most 32 PENDING (`MAX_PENDING_LICENSES`) and 1024 ACTIVE (`MAX_ACTIVE_LICENSES`) | `revokeLicense` (issuer); a transfer swaps one for one |
| `pendingTransferOf` | at most 1 per active licence | `approveTransfer`, `withdrawTransfer`, `revokeLicense` |
| `licenseSlotOf`, `licenseAtSlot`, tree leaves | 1 per active licence; at most 2^24 in all | `revokeLicense` |
| `activeLicenses` root history | one root per tree change since the last seal | `sealRevocations` (anyone, no more often than every 600 s; up to 900 s after the previous seal if its sealer set the bound 300 s ahead) |
| `pendingObligations` | at most 8 per proposer (`MAX_PENDING_OBLIGATIONS`) | `withdrawObligation` (proposer), `acceptObligation` or `rejectObligation` (holder) |
| `openObligations` | 16 per record, plus 16 per recovery, at most 272 (`MAX_OPEN_OBLIGATIONS`) | `discharge` (beneficiary only) |

**Why that is a 2.** Each identity's share of state is capped, and each identity costs an
`anchor` transaction and its fee. State grows with the number of anchored records, not
with how often anyone calls: a repeated call by one identity overwrites a cell, stops at
a cap, or clears what it created (`contract/src/test/state-bounds.test.ts`). That is the
shape of the rubric's Tier 2 examples, such as an ERC-20 balance map that grows with the
number of holders. It is not a 1: nothing caps the number of identities.

**The August self-score of 3, faced directly.** In August we scored V1 a 3 ourselves,
because it stored an entry per anchor that nothing removed (*Why anchoring writes no
state*, under *Superseded*). This contract again writes permanent entries per anchor:
`anchor` writes `recoveryOf` and six counters (`obligationCountOf`, `rotationsOf`,
`recoveriesOf`, `pendingObligationsBy`, `pendingLicensesBy`, `activeLicensesBy`), and
nothing removes them. The worst case of permanent entries per anchored identity:

- 7 written at anchor (`recoveryOf` and the six counters);
- at most 288 in `originOf` (16 rotations, then up to 16 recoveries each followed by 16
  more rotations);
- 1 in `headOf`;
- at most 2 parents in `parentsOf`;
- 1 in `hasOffspring`.

That is at most 299 per identity. Fees are paid in DUST, which regenerates, so the cost
of creating identities is a rate limit, not a price: a party with NIGHT can keep
anchoring as its DUST comes back. We nonetheless score 2. Every per-identity quantity is
capped. The identity is the unit of use, as a token holder is in the rubric's Tier 2
example. Everything else is cleared by its creator or overwritten. Reviewers may read
this as Tier 3; we would rather say so than have it found.

**Where the bound is weaker, stated plainly.**

- **Root history depends on someone sealing.** Every activation, approved transfer and
  revocation adds a root to the licence tree's history, and only a seal clears it. The
  contract never seals on its own. A seal is now allowed when only activations changed
  the tree. There is no automated sealer; the operator seals by hand (CLI option 15)
  after licence activations. If nobody
  seals, the history grows with every tree change.
- **Many anchors get many caps.** The caps are per anchored identity. One party who pays
  for many anchors gets a fresh set with each, for example 8 waiting proposals per
  anchor against one record. The holder can find each from the proposal cells and
  reject it, a transaction each.
- **The caps can refuse legitimate use** at the limits: a third parent, a 17th
  obligation in force on a record that has never been recovered, a 33rd licence waiting for countersignature, a
  1025th active licence from one issuer, or a move after an identity has used its 16
  recoveries and the 16 rotations after the last one.

The permanent entries are permanent on purpose. Keeping old successors in `originOf` is
how a retired secret stops working, and a pedigree an ancestor could edit would not be
evidence. They are now capped per identity.

**We score state-space at 2.** The bounds are new and were reviewed twice, by an attack on
them and an attack on the fixes, described under *What was found and fixed* (seven
findings, then two more; six fixed in the contract, three documented). Whether they are enough is the reviewers'
call.

### Privacy

The approved header said genetic preimages, licence terms *and counterparties* never
leave the holder's device, and that disclosed values have no identity linkage.
Preimages and licence terms still stay off chain. The claims that counterparties stay
off chain and that disclosed values have no identity linkage are no longer true. This contract publishes, as
commitments:

- **Parentage** between identities, readable from state (`parentsOf`) and per
  transaction (`lastDescentChild`, `lastDescentParent`).
- **Obligations:** the record, the obligation commitment and the beneficiary, per
  transaction (`lastObligationRecord`, `lastObligation`, `lastBeneficiary`). A proposal
  publishes the same three before the holder answers (`lastProposedObligation`,
  `lastProposedAgainst`, `lastProposedBy`), so a rejected proposal still shows it was
  made. Terms stay off chain, hashed with a random salt.
- **Licences:** issue, countersign, approve and revoke publish the licence key and the
  issuing record; a transfer proposal and its withdrawal publish the licence key; a seal
  publishes neither. The licence key links these calls to each other. Issue, countersign
  and an approved transfer publish a licence commitment (`lastIssuedLicense`,
  `lastActivatedLicense`, `lastTransferredLicense`); issue and approve do so that a
  recovered owner can revoke licences a thief issued or transferred. A transfer proposal
  publishes the incoming commitment. Revoke and approve publish the caller's record.
- **Presentations** publish a tag and a tree root. The root narrows the issuer to those
  with live licences at that root. At launch that can be one issuer.
- **Ownership proofs** publish the record and the verifier's challenge.
- **Rotations and recoveries** publish the old and new commitments. Rotation does not
  unlink.

Whether fee payments can link a presentation to its countersign is an open question for
the Midnight wallet, not this contract. We keep the score at 2: counterparty and timing
data between pseudonyms, with no identity data on chain. It is the score we are least
sure of after state. Cannabis is a stigmatised market, and the rubric lists such
markets under Tier 3. We claim 2 because nothing on chain names a party: linking an
identity needs its holder to disclose it off chain. Once disclosed, that identity's
public history reads as Tier 3.

### Value

Unchanged at 1. The contract holds no funds and no circuit moves tokens.

### What was found and fixed

**Outside review.** Max Weber (ODATANO / NIGHTGATE) attacked the licensing circuits in
August ([issue #22](https://github.com/hunterincoming/veilcore-midnight-testnet/issues/22);
*Revision — 24–25 August 2026* above). In September he checked the third revision's
build against its public transcripts, which found the defect in *Correction — 16
September 2026*. The repository records no review by him of this build.

**Twelve rounds of adversarial review** on 30 September and 1 October 2026, recorded in
`docs/security-pass-30sep.md`. Round 1 was our own. Rounds 2 to 12 were by AI reviewers
in separate sessions that had not seen the fixes, directed by the founders (Max Weber's
August and September passes, above, are the human reviews). Three rounds (8, 11 and 12)
were followed by an independent re-attack of their fixes. Contract findings were
demonstrated against the build before being fixed or documented. Contract attacks are kept
as tests in `contract/src/test/`. Not every finding has a test. These were adversarial
reviews, not a formal security audit.

Contract findings, as rated there:

- **CRITICAL.** A transfer could forge a licence from another issuer, because the tree
  leaf was the bare licence commitment. The leaf is now `licenseKey(licence, issuer)`.
- **HIGH.** Licences became unrevocable after two rotations. Recovery could not beat a
  thief who rotated first, and a thief could block it by rotating again; recovery now
  writes the new head without reading the old one. `proveLicense` proved nothing a
  verifier could tie to an issuer or a request; it now answers the verifier's challenge.
  Revocation could be starved by a licensee toggling a transfer proposal. Anyone could
  encumber any record, and only the attacker could release it. A
  revoked licensee could bundle a stale presentation with a seal in one transaction and
  pass; a presentation now records at proof time whether a revocation was waiting.
- **Others, rated MEDIUM or lower, or not rated.** Lineage could not survive a rotation
  (see *What changed*). A recovery secret stayed valid after use. An ancestor could
  rewrite the pedigree of material already descended from it. A retired secret could
  still prove ownership. Anyone could cancel every presentation in flight once a block.
  `issueLicense` could be blocked by front-running. `proveOwnership` named no verifier.
- **Client and tooling.** The CLI wrote record secrets, recovery secrets, the wallet seed
  and the maintenance key to plain-text log files. The maintenance key stayed in the
  local store after deploy. "Deploy with no maintenance authority" deployed one with a
  random key. All fixed.

  > **Round D (4 October).** The key "removed" from the local store after deploy was still
  > readable in the store's files (LevelDB keeps deleted values until compaction). Since
  > round D it is never written there. See *Attack round D* below.

In rounds 10 and 11 the contract held: no HIGH or MEDIUM finding on chain, and the
contract did not change. It then changed once more, for the state bounds (see *State*).

**Round 12: an attack on the state bounds.** An independent attacker went after the new
caps (`contract/src/test/attack-bounds.test.ts`) and found seven issues. Four were fixed
in the contract:

- F1: a thief holding the head key could fill the owner's 32 pending-licence places with
  licences the owner could not revoke, because issue did not publish the licence
  commitment. Issue now publishes it (`lastIssuedLicense`).
- F2: the same for the 8 waiting-proposal places. A proposal now publishes what, against
  whom and by whom (`lastProposed*`), so the recovered owner can withdraw them.
- F3: a thief could spend all 16 rotations. Recovery now resets the rotation count.
- F5: active licences were not counted per issuer. They are now capped at 1024
  (`activeLicensesBy`); revoking an active licence frees a place, a transfer keeps the
  count.

Three are documented, not fixed: F4, a thief can fill both parent slots, and confirmed
edges are permanent; F6, the root history is cleared only when someone seals; F7, the
caps are per anchored identity, so a party with many anchors gets more. All three are
under *Known and not fixed*.

A second attacker found R1 and R2 (both fixed in `ceb3a16`); P2 documented. R1: a thief
could fill a record's 16 obligation places with obligations only he can release; a record
now gets 16 more places per recovery (at most 272). R2: a thief could transfer a licence
to a commitment no event cell showed; `approveTransfer` now publishes the new commitment
(`lastTransferredLicense`). P2: a proposal publishes the obligation commitment even if
rejected, harmless when salted, as the CLI does.

**Attack round D, 4 October 2026** (`docs/security-pass-4oct-roundD.md`). Six independent
reviews: the website, the fee-paying demo service, the SDK (TypeScript, Python, Rust), the
registry, this contract with its operator tool and client API, and the supply chain.

- **This contract:** no Critical, High or Medium finding in `veilcore.compact`. It did not
  change: the source is still `ceb3a16` and the fingerprints are still those in `e89a387`.
- **Operator tool and client API: three Mediums and six Lows, all fixed.** The ones that
  change how this contract is deployed and joined:
  - The maintenance key is never written to the deploying computer. It was "removed"
    from the local store after deploy, but the store's files kept it. Signing keys are now
    held in memory only (`api/src/memory-overlays.ts`), and finishing a deploy asks for the
    key from paper.
  - A contract with this build's circuits but a forged starting state passed `join`.
    `join` now reads the contract's deploy transaction and compares its starting state with
    this build's constructor (`api/src/starting-state.ts`). On mainnet, `join` accepts only
    the address pinned in the code (`MAINNET_VEILCORE_ADDRESS`, `api/src/deploy-guard.ts`),
    which is empty until the deploy and is then set to the address recorded here. The
    deploy prints its transaction id, recorded under *Mainnet deployment* below.
  - A recovery-secret replacement could land while reported as failed. It is now
    confirmed by two reads 30 seconds apart, as rotation and recovery are.
  - Retiring the authority now installs an empty committee, which anyone can read on
    chain, instead of a key nobody keeps (see *The maintenance authority*).
- **Re-check.** A seventh reviewer checked the first fixes by removing each guard and
  confirming a test fails. It found one new Medium here: the first version of the
  starting-state check could refuse the genuine contract after any key change, because
  midnight-js returns the current state, not the deploy state, when the latest action is a
  maintenance update. Fixed in `8de6f2a` by reading the deploy transaction itself.
- **Not yet independently reviewed:** that fix (`8de6f2a`), and the claims contract's
  mainnet gate (`5a980b3`: deploy guard, address pin, fingerprint check). The branches that
  only run on mainnet (fingerprint refusals, empty-pin refusals, joining at the pin) cannot
  run on preprod and are covered by unit tests only.

These were adversarial reviews by AI reviewers in separate sessions, directed by the
founders, not a formal security audit.

### Known and not fixed

From `docs/design.md`, *Known limits* and *Trust model*. Each is stated there.

- **What a thief does before recovery stands.** Revocations, accepted obligations and
  confirmed parentage are not undone. No one can remove a confirmed edge.
- **Licences a thief issued** stay PENDING under the identity until the owner revokes
  them. Each issue publishes its commitment, so after recovering the owner can find and
  revoke them; no client does that lookup yet. Proposals a thief made in the identity's
  name are the same.
- **A thief can fill both parent slots** with records whose holders confirm (F4). The
  edges are permanent, so the true parent can never be recorded.
- **A parent proposal a thief filed survives recovery.** If the named parent confirms,
  the edge is permanent. The owner should withdraw it (`withdrawParent`).
- **Root history is bounded only while someone seals** (F6). Nothing seals on its own.
  There is no automated sealer; the operator seals by hand (CLI option 15) after licence
  activations.
- **The caps are per anchored identity** (F7). Each anchor costs one fee and brings a
  fresh set; a party with many anchors can, for example, keep 8 proposals per anchor
  waiting against one record, which the holder rejects one transaction at a time.
- **The caps can refuse legitimate use:** a third parent, a 17th obligation in force on
  a record that has never been recovered, a 33rd pending or 1025th active licence from one issuer, or any move after
  16 recoveries and the 16 rotations after the last one.
- **The revocation window.** A revoked licence's old path verifies on chain until the
  next seal: up to 600 s plus 300 s plus the time until someone seals. A verifier
  following rule 5 does not accept it. A griefer can make honest licensees re-prove, at a
  fee to the griefer too.
- **No list of revoked licences.** A revoked licensee who makes a new licence secret
  cannot be recognised. An issuer must know who it is issuing to.
- **The licensee should make the licence secret.** Stated, not enforced, as in August.
- **A presentation names the issuer, not the licence.** Any live licence from that
  record passes, including one the issuer granted itself.
- **No expiry on chain.** Licence terms and end dates live off chain.
- **Record commitments are stable pseudonyms.** Actions under one record link.
- **A record made from the all-zero secret** can be anchored, and then anyone can act
  as it.
- **An edge whose parent has no holder** can never be confirmed.
- **A thief holding a beneficiary's secret** can release what is owed to it before
  recovery.
- **The indexer is trusted for what it reports.** On mainnet that is Blockfrost:
  Midnight's hosted mainnet indexer endpoint was retired on 30 September. For a decision that matters,
  compare a second indexer or your own node.
- **A licence presentation shows the licence was live when it landed, not later.** A
  later revocation cannot be tied to it. Since round D, `verify.ts` refuses a presentation
  that landed before its challenge was issued or more than an hour ago
  (`acceptPresentationAt`, `MAX_PRESENTATION_AGE_MS`); the CLI's option 27 uses it.
  Earlier drafts of this revision said `verify.ts` did not check the first of these; it
  does.
- **The VeilCore registry service is not the source of truth.** It is out of scope here.

### Deployment in fragments

A preprod smoke test of an earlier 24-circuit build failed at deploy with "exceeded
block limit in transaction fee computation". The deploy transaction carries a verifier
key per circuit, and all 24 do not fit.

`VeilcoreAPI.deploy` (`api/src/veilcore-api.ts`) now deploys with the keys of the first
8 circuits, by name order: `acceptObligation`, `anchor`, `anchorBatch`,
`approveTransfer`, `confirmParent`, `countersignLicense`, `discharge`,
`encumberOwnRecord`. On a block-limit refusal it halves to 4, 2, then 1. The maintenance
authority then adds each remaining key in its own transaction, waiting for the indexer to
show each before the next. CLI deploy option 4 finishes an interrupted deploy. The ledger
and every circuit are the compiled ones; only which keys ride the first transaction
differs. On preprod (2 October 2026) 8 fitted at the first try: the deploy transaction
carried 8 keys (block 2807919) and the 16 maintenance transactions that followed took
under five minutes, all 24 keys on chain at 14:27 EDT. Whether 8 fits on mainnet is
known only at the deploy; if not, the tool halves as above.

During the deploy the contract is on chain with some keys and not others. Circuits
already keyed can be called. Only the authority can add keys (checked in round 9).

`join` checks every key on chain against the local build, and refuses a contract that
carries a circuit the build does not have. On mainnet, deploy (option 1), join
(option 2) and finish (option 4) first refuse unless the local build matches the
committed `docs/fingerprints.md`.

`api/src/deploy-guard.ts` refuses every network except undeployed, preview and preprod
unless `VEILCORE_DEPLOYMENT_RECORD_REVISION` is set to 4 or more
(`REQUIRED_RECORD_REVISION = 4`). It checks a declared number. It cannot see whether this
revision was filed or approved.

### The maintenance authority

**This changes what the 13 September revision said, in two ways. Both changes need the
approval of both founders before this revision is filed. Hunter Roberts approved both on
6 October 2026, with the maintenance policy (`docs/maintenance-policy.md`); Makoto
Steiner's confirmation is marked below.** That revision said the mainnet authority would
be "held jointly, not by one person and not in a file on a laptop", and that relinquishing
it "remains open and is the intended end state".

1. **Not held jointly at launch.** The implementation does not hold it jointly.
   `VeilcoreAPI.deploy` installs one signing key as the authority. The midnight-js calls
   it uses, `deployContract` and `replaceAuthority`, take a single key. There is no second
   signer and no threshold. The ledger supports a committee with a threshold
   (`ContractMaintenanceAuthority`); using it needs our own deploy and maintenance code,
   which does not exist yet.
2. **No retirement date, and a different end state** (see *How it ends*).

How it is handled. The CLI generates the key, or takes one typed in. A typed-in key is
hidden as it is typed and is not shown again. A generated key is shown on screen only,
in groups of eight characters; nothing is sent until the operator types WRITTEN and then
types the key back from paper, hidden, and it matches. **The key is never written to the
deploying computer's disk.** The CLI holds signing keys in memory only
(`api/src/memory-overlays.ts`, since round D on 4 October 2026; before that, a key
"removed" after the deploy could still be read from the local store's files). The deploy
transaction is built first and the contract address is logged before it is sent. To
finish a deploy that stopped partway, CLI deploy option 4 asks for the key again from
paper, uses it for that run only, and drops it. After the deploy the computer keeps no
copy. VeilCore's copy is on paper (who holds it: below).

**Who holds the main contract's maintenance key.** Proposed on 3 October 2026 and approved
by Hunter Roberts on 6 October 2026 (`docs/maintenance-policy.md`): one key at launch, on
paper only, two copies, one held by Hunter Roberts and one by Makoto Steiner, kept
separately; then a two-of-three committee with an independent holder, built and tested
with the move to midnight-js 5 that the ledger v8 to v9 upgrade requires anyway, and
installed by one published `replaceAuthority` transaction. Two paper copies of one key
guard against losing it; they do not stop one founder acting alone, because either copy can
sign. The alternative was real joint control (two signatures required) before launch,
which means writing, attacking and preprod-testing new deploy and maintenance code first,
and so a later mainnet date. The deploy tool checks one copy, the one typed back; the
runbook has the operator write the second sheet while the key is on screen and check it
against the screen group by group, and Makoto Steiner's copy goes to him by hand, never
photographed, scanned, emailed, messaged or typed anywhere (`docs/runbook.md`, step 18).

**Decided.** The two-copy key above and the maintenance policy in
`docs/maintenance-policy.md` were approved by Hunter Roberts on 6 October 2026 and confirmed
by Makoto Steiner on 7 October 2026.

**VeilCore-run does not touch the authority.** VeilCore also offers a managed service,
VeilCore-run (`docs/MANAGED.md`), for partners with no developers: VeilCore holds a
partner's record, licence and claim secrets in a store of that partner's own and sends
the partner's transactions for them. To the contracts it is one more client. It is built
only on the partner kit's public exports (`@veilcore/contracts`; every import is checked by
`veilcore-run/test/surface.test.ts`), which include no deploy, circuit-key or maintenance
operation, and the wallet it pays fees from is not the maintenance key. It changes nothing
in either contract, and nothing about who holds the main contract's authority or whether
the claims contract has one. What a partner hands VeilCore by using it is set out in
`docs/MANAGED.md`.

What it can do. It can add and remove verifier keys, so it can repair or disable any
circuit, and a key for a new circuit could rewrite state. Whoever holds it controls the
contract. It cannot change the ledger layout; that needs a new contract. Until it is
retired, holders should treat the circuit set as changeable by VeilCore.

Why it is kept at launch. The reason given on 13 September stands: circuits are bound
to the proof system that compiled them, and a contract nobody can maintain can only be
replaced. Midnight's maintainers state that circuits which compile differently after a
ledger upgrade need "a maintenance verifier-key update" (midnight-node #1969). It is also
how a wrong circuit is fixed or switched off without abandoning every record anchored on
the contract.

How it ends. `retireMaintenanceAuthority` (`api/src/veilcore-api.ts`; CLI main menu option
33, which asks for the key from paper) replaces the authority with an empty committee and
threshold 1 (`retireMaintenanceAuthorityProvably`, `api/src/maintenance.ts`). No signature
can satisfy it, no replacement key is made or stored, and anyone can read it from the
contract's state. The claims contract is locked the same way at the end of its deploy.

**No retirement date for the main contract.** Proposed on 3 October 2026 and approved by
Hunter Roberts on 6 October 2026 (`docs/maintenance-policy.md`); confirmed by Makoto
Steiner on 7 October 2026, with the key above. A retired authority
could not make the verifier-key update a ledger upgrade may require. The end state named on
13 September, relinquishment, changes to custody by independent parties under published
rules. Retirement remains possible if Midnight stops requiring maintenance across
upgrades. Every use of the authority is limited to a network upgrade, a security fix, a
correctness fix, or a change both founders approved in writing; announced at least 14
days ahead (an urgent security fix within 72 hours after); and published in a new
revision of this record.

**A retirement is visible on chain; a live authority's custody is not.** A retired
authority is an empty committee, which anyone can read from the contract state. While the
authority is live, the chain cannot show who holds its key or how many copies exist;
outsiders rely on this record and the maintenance policy. Any use is public: maintenance
transactions and the authority's counter are on chain, and anyone can compare the circuits
and keys on chain with a build of `ceb3a16`, as `join` does. That detects a change after it
happens. It does not prevent one.

### The claims contract

**No maintenance authority on the claims contract, enforced by the operator tool.** Off a
test network, mainnet included, the tool refuses any claims deploy that would end with an
authority (`assertClaimsDeployAllowed`, `api/src/claims-api.ts`): the only deploy it builds
ends with an empty committee. Keeping an authority would mean changing that code and
another review first (`docs/mainnet-completeness.md`). Both founders agreed (Hunter
Roberts 6 October, Makoto Steiner 7 October 2026). Its address and transaction ids are filled in after the deploy (*Mainnet
deployment* below).

A second contract, deployed separately from the main one, which stays unchanged. A holder
uses it to prove one fact about a record sealed with `sha256/fields/v1` (SPEC 4.5) without
showing the rest. Design, limits and measurements: `docs/claims-design.md`.

- **Circuits (5):** `proveValue`, `proveRange`, `proveDistinct`, `proveUnchanged`,
  `proveAttested`. Every one at most k=17 (`contract/scripts/circuit-sizes.sh`), so a
  holder proves on an ordinary laptop.
- **State:** fixed event cells (`lastClaimKind`, `lastClaimRecord`, `lastClaimOther`,
  `lastClaimSchema`, `lastClaimSlot`, `lastClaimParam`, `lastClaimOp`,
  `lastClaimAttesterX`, `lastClaimAttesterY`) and one counter (`claimSeq`). Nothing grows
  with the number of claims. Verifiers read each claim per transaction through the
  indexer.
- **What a claim publishes:** the record commitment(s), the schema id, and per claim kind
  a slot, a bound and its direction, a mask, or a laboratory's public key. A value claim
  publishes the value itself, by the holder's explicit choice. No other value, salt or
  field secret reaches the chain.
- **Funds:** none. No circuit receives, holds or sends tokens.

| Category | Self-assessed score (1–3) | Rationale |
|---|---|---|
| Privacy-at-risk | 2 | As for the main contract: pseudonymous record commitments and their timing. A value claim publishes one value by the holder's choice; repeated range claims narrow a hidden number, and the tool shows what was already published. |
| Value-at-risk | 1 | Holds no funds. A false claim accepted would mislead a verifier off chain; it cannot drain anything. |
| State-space-at-risk | 1 | Fixed cells and one counter; nothing is added per claim or per user. |

**Maintenance authority: none.** The deploy adds the five circuit keys, then replaces the
authority with an empty committee and threshold 1 (`api/src/maintenance.ts`,
`retireMaintenanceAuthorityProvably`), which no signature can satisfy and which anyone can
read from the contract's state. On mainnet the operator tool refuses any claims deploy that
would end otherwise (`assertClaimsDeployAllowed`, `api/src/claims-api.ts`). Until that last
step lands, the authority is a temporary single key that the deploy generated and never
shows to anyone. The CLI keeps it in the deploying computer's encrypted private-state store (unlike
the main contract's key, which is held in memory only), so that an interrupted claims
deploy can be finished (CLI option 36), and deletes it once the empty committee is
confirmed on chain; from then on it controls nothing. A change, including one forced by a
Midnight network upgrade, is a new deployment at a new address, with a new pin in the code
and a new revision of this record; verifiers have to be told. Claims made on the old one
stay in the chain's history; reading them depends on the indexer decoding old transactions
(see *Open design question* in `docs/mainnet-completeness.md`).

**Source and build.** `contract/src/veilcore-claims.compact` (with `schnorr.compact`), last
changed in `cd30c11` and unchanged since, built at `c75c155` with compactc 0.31.1 by
`cd contract && npm run compact`, which compiles both contracts. Fingerprints: `npm run fingerprints:claims` writes the claims table of
`docs/fingerprints.md` (it refuses unless the same build of the main contract still matches
the main table above). On mainnet the CLI refuses to deploy, join or finish the claims
contract unless the local build matches that table as committed
(`bboard-cli/src/keys-check.ts`).

SHA-256 of compiled artefacts (`contract/src/managed/veilcore-claims/`), copied from
`docs/fingerprints.md`: a prover and a verifier key for each of the 5 circuits, the ZKIR of
each circuit in two forms, and the compiled contract code (21 rows).

Built at `c75c155` with compactc 0.31.1 on the founder's machine (fingerprints committed
in `765cab1`). A second build with compactc 0.31.1, without key generation (4 October, from
the same claims source), reproduced the 5 `.zkir` files and `contract/index.js` byte for
byte, and so did another on 6 October from `a3d1884`. The proving and verifying keys and
`.bzkir` files were built once, on the founder's machine.

| Artefact | SHA-256 |
|---|---|
| `keys/proveAttested.prover` | `2abb918cd2a749716e61346e82d69550ff3015e5892a540c69d2b2512310e451` |
| `keys/proveAttested.verifier` | `3a26f2f34082eccb6a1c96fc9ec20243247d2c723666f218bd61a479432d9f38` |
| `keys/proveDistinct.prover` | `79c10171a6438316f79fb9d3c0c0b262c82514d51009b0696cfde6867a156ae1` |
| `keys/proveDistinct.verifier` | `6d1b1166d5ca9e8e3e2e2dcfb83079bfa192f8fdd8a617a74bb15166e921cf24` |
| `keys/proveRange.prover` | `f13517970ac4f81537d766f1d48e3567c5d3525420e9071df5c01096f2119397` |
| `keys/proveRange.verifier` | `54c8245f19f8296a60dc1110fade6e1b52ba6a3e72ba5be832206c86c145c1ae` |
| `keys/proveUnchanged.prover` | `f01607396d5d01a3fb3756dea3fddcb98f35ab730fab316fd6629b40ff432fb2` |
| `keys/proveUnchanged.verifier` | `cb7d9e9cdbaa82f7b9cc9046405174b40b5b070037d8e79d4d54859b4150b103` |
| `keys/proveValue.prover` | `e1f159dbaa4649f8d5622f8d2703770125bc16f54c0bc92a78b8af1002423ab7` |
| `keys/proveValue.verifier` | `af639a1af8e84cde7bf44c4df401db73ecd8ec46074e34e2d9e8a4679834c2ef` |
| `zkir/proveAttested.bzkir` | `ed892e54325bce35ff5d3e17e843b413f3d813e4b7dc4639a74931f37008d762` |
| `zkir/proveAttested.zkir` | `b932d410ae925ba8f951d680d8e8aaf821733f27d6e1b7940fce40c99d881c70` |
| `zkir/proveDistinct.bzkir` | `5dcf734ae7015016b77e3faf7ea4e06d9426cef9dd63d500f51a4320a64d1605` |
| `zkir/proveDistinct.zkir` | `3894b23a38800483eb7aa6e926e654caee836a0e5299def9a5211c6c5b3e558d` |
| `zkir/proveRange.bzkir` | `cb4cacefd457dfb627338733fd449998b33a5580ef14c103b26dc30289947206` |
| `zkir/proveRange.zkir` | `60db0e31b6f4b52d9ce040b073a05904665718786ff9e5b6c2c648c517b11c56` |
| `zkir/proveUnchanged.bzkir` | `3cd33cbcfd6e3797f96f4919428bcec2d12d8b5ca40f99888249d228aaf24175` |
| `zkir/proveUnchanged.zkir` | `74b4bb9c579dc31adbbdb639db11a2115bae29ec18edf9296537c0fcc18bd622` |
| `zkir/proveValue.bzkir` | `efb54476e612e61d0bda9cba3517da16eae508366f33f32b207f0fd2c1e334e1` |
| `zkir/proveValue.zkir` | `d7e2e4309c0e79fc39a09f6c35350ae0d30c98f5d62b872eb40de3f7505dc34a` |
| `contract/index.js` | `b549c631fa3e3f2e4adf554434519e447844d7f83748cec44f8145ae1f6ecd65` |

**Deployment on mainnet.** Deployed on deploy day right after the main contract, from the
same operator run (CLI main menu option 34). Joining it on mainnet accepts only the
address pinned in the code (`MAINNET_CLAIMS_ADDRESS`, `api/src/deploy-guard.ts`), the one
recorded under *Mainnet deployment* below.

**Preprod.** 4 October 2026, smoke test with the claims phase, PASSED 37 of 37 on the
founder's MacBook Air (16 GB): claims contract
`175f23573c3d9c9dd20d8bee159df07fc3b739e6a2a895f14ae4d20de5d2a4af`, its authority read
back from the chain as an empty committee (check 28), then every claim kind proved on the
laptop and landed (`docs/preprod-run-4oct.md`). 5 October 2026, on the operator tool after
round D (`d9d563f`), PASSED 37 of 37 again: claims contract
`29d3ea80e121518f8fd8bd72533d856cf29cdbddbda1b6f322a661aa4f2484b6`, deploy transaction
`0060fbb06e1483161bf0bee8204491ca09b698be70f59e9cc0e18e635cc8fbedcc`, authority read back
as an empty committee (check 28) (`docs/preprod-run-5oct.md`).

### Mainnet deployment

This revision was filed on 7 October 2026 (midnight-improvement-proposals pull request
#373), before either contract was deployed, as the 16 September correction promised. The
details below were added on 8 October 2026, after the deploy, as an addendum to this
revision, together with the commit that pins both addresses in `api/src/deploy-guard.ts`
(`MAINNET_VEILCORE_ADDRESS` and `MAINNET_CLAIMS_ADDRESS`). Joining either contract on
mainnet accepts only these addresses.

**Main contract (`veilcore`, source `ceb3a16`, fingerprints in `e89a387`):**

- **Contract address:** `a04de0a2684f3713276325649540c7278ffd07cba8b014e7489844f319a02347`
- **Deploy transaction id:** `007553ba3c32d93d305ec8b828bb3481d3f9c79d9e6644fa12f19e623557141c70`
  (block 2922687, 8 October 2026, 06:44 EDT)
- **Circuit keys in the deploy transaction:** 8; the rest added in 16 maintenance
  transactions; all 24 on chain at 06:50 EDT, 8 October 2026. The tool then checked that the
  deploy transaction carries the VeilCore constructor's starting state.
- **Maintenance authority:** one signing key, on paper, two copies, as under *The
  maintenance authority*

**Claims contract (`veilcore-claims`, source `cd30c11`, built at `c75c155`, fingerprints in
`765cab1`):**

- **Contract address:** `ef763eb4ad1846b716dbfa90c00560a9a943ffb4d1fc638b0707df5adefd070d`
- **Deploy transaction id:** `0023acc5850b4fc87f97e12cee605d22e80570f40731e38d8574083984e7868f80`
  (all 5 circuit keys in the deploy transaction; block 2922871)
- **Retirement (empty committee) transaction:**
  `0091979012d160ff9e4cb1162b8e29f18dfccdeb00068357a40913fccfe658335f` (block 2922874)
- **Deployed:** 8 October 2026, 07:03 EDT

**Both:**

- **Pin commit** (`MAINNET_VEILCORE_ADDRESS` and `MAINNET_CLAIMS_ADDRESS` in
  `api/src/deploy-guard.ts`): `5722fdb` (8 October 2026)

### Testing and deployment status

- **Contract tests on this build:** `Test Files 18 passed; Tests 266 passed | 9
  expected fail | 1 skipped (276)` (`cd contract && npm test`, run on the founder's Mac
  at `ceb3a16` with `FUZZ_RUNS` unset, 1 October 2026). The 9 expected failures are
  attacks that must fail. The suites now include `state-bounds.test.ts`,
  `attack-bounds.test.ts` and `reattack-bounds.test.ts`. The skipped test is the
  1024-active-licence cap (in `attack-bounds.test.ts`), which runs only with
  `SLOW_TESTS=1`. It has since been run to completion: on 3 October, after a fix to the
  test simulator's slot search (`docs/self-audit-3oct.md`, section 2), and on 6 October at
  `a3d1884`, where `SLOW_TESTS=1 npx vitest run src/test/attack-bounds.test.ts` passed 16
  of 16. (Earlier drafts of this revision said it had not.)
- **Contract tests on 6 October 2026, both contracts, at `a3d1884`:** `Test Files 32
  passed (32); Tests 528 passed | 9 expected fail | 1 skipped (538)` (`cd contract &&
  npm test`, built with the CI-pinned compactc 0.31.1, `FUZZ_RUNS` unset). The extra
  suites since 1 October are the claims contract's, round D's and the mutation-testing
  follow-ups. `cd api && npm test` (deploy guard, presentation lookup, the empty-committee
  retirement on a real ledger-v8 state) also passed.
- **Smoke test on a local Midnight chain, this build: PASSED 26 of 26** on 1 October
  2026 at 20:56 EDT (node 0.22.3, indexer 4.0.1, proof server 8.0.3; local contract
  `84cca6f12d5035eeda3b9277872b38ce87e3614c9410ba8f86fed04fce7c13a8`). Run again on 2 October 2026 at 05:26 EDT with the final operator tool (`db1cd3e`, which changed the deploy order): PASSED 26 of 26, local contract `0b784aadba3507eeb522fbe27684849927ded8d4412c1a1a6c95ce5bd04cc2ee`. It deploys
  a fresh contract and calls 16 of the 24 circuits with real proofs. It does not call
  `anchorBatch`, `replaceRecoveryCommitment`, `withdrawTransfer`, `withdrawParent`,
  `proposeObligation`, `acceptObligation`, `rejectObligation` or `withdrawObligation`.
  Run by the founder on his MacBook; the closing line was `SMOKE TEST PASSED: 26 checks
  passed. Contract 84cca6f1…`.
- **Smoke test on preprod, this build: PASSED 26 of 26** on 2 October 2026, closing at
  14:35 EDT (operator tool `455cf05`; last transaction, `discharge`, at preprod block
  2808047). Contract address:
  `9c7b69275e53acc38fcbebff93c53febe46a3898580c11fd2c4b923fc5efb7a3`. Run by the founder
  on his MacBook; the closing line was `SMOKE TEST PASSED: 26 checks passed. Contract
  9c7b6927…`. An earlier attempt the same morning (`b0008e3`) was refused before any
  contract was created with `1010: Invalid Transaction: Custom error: 171`
  (OutOfDustValidityWindow) while the preprod indexer lagged the chain; the operator tool
  then misread that refusal as a block limit, fixed in `455cf05`
  (`docs/security-pass-30sep.md`).
  After this run an independent review changed the operator tool's failure paths only
  (messages, a forced stop, terminal scrubbing; `docs/security-pass-30sep.md`); the
  successful path and the contract are as run here. The local smoke test was re-run on
  the final tool (`c0647dc`) on 2 October 2026, closing at 19:46 EDT: PASSED 26 of 26,
  local contract `88ef3d861c043f4d48be4d2aacd63d1118266ccfed32be5ac160ee7f0563d428`.
  The operator runbook makes a passed preprod run a precondition of the mainnet deploy.
  The code does not check it.
- **Smoke test on preprod with the claims phase: PASSED 37 of 37** on 4 October 2026, on
  the `claims-contract` branch (this contract's source unchanged), on the founder's MacBook
  Air (16 GB): main contract
  `f239e680f1f60c38990b7066c59c9538a42c2ae414e584e21155350a7e93306a`, claims contract
  `175f23573c3d9c9dd20d8bee159df07fc3b739e6a2a895f14ae4d20de5d2a4af`
  (`docs/preprod-run-4oct.md`). This was before the round D changes to the operator tool.
- **Smoke test on preprod after round D: PASSED 37 of 37** on 5 October 2026, 19:01 to
  19:13 EDT, on `main` at `d9d563f` (the operator tool with the round D fixes: private
  state in `~/.veilcore/preprod/`, maintenance key in memory only, the join check that
  reads the deploy transaction, the claims mainnet gate). Main contract
  `93c062e10863ee8d4d72694a42908aa6c55036645fcc327fafc533bc827dc294`, claims contract
  `29d3ea80e121518f8fd8bd72533d856cf29cdbddbda1b6f322a661aa4f2484b6`
  (`docs/preprod-run-5oct.md`). Since then, up to `a3d1884` (6 October), only one message
  line in the smoke test and one test file (`bboard-cli/src/claims-mainnet.test.ts`) have
  changed in `bboard-cli`, `api` or `contract`. The smoke test deploys the main contract
  through the API with a key it passes in directly; deploy option 1's paper-key prompts,
  finishing with option 4 from paper, and option 33 on the main contract have not been
  run on a live network.
- **Earlier deployments, none of them this build:** V1 on Preview at
  `4a457e6d046928e0faa971d80701b8cd48c3a1283713039444b47fedd0a1f3c7` (22 July); V2 on
  Preview at `dc18e54d2f8031dda0eca1970bb1b1639c1686a14303fe057bb46f07bd0a233b` (10
  August); preprod at
  `fb9c55944908c466dcea7b9807f00ea727b37cebec13870080016ddc5a9d721d`, named on 13
  September; the provenance contract on Preview at
  `f75d42dc1e4ec5a2cdcc50509f2d432ad60fb5c64b5da921a0ec22a0e287f939` (27 September,
  before the merge).
- **Both contracts were deployed to mainnet on 8 October 2026** (*Mainnet deployment*
  above). The steps before it are listed here as they stood at filing.
- **Zero-spend mainnet rehearsal (7 October 2026):** passed. The operator tool on mainnet
  through Blockfrost: indexer, node and local proof server connected; the deploy wallet
  restored from its recovery phrase, its DUST address matching the wallet app's; synced in
  about 10 minutes with DUST available for fees; then exit at the deploy menu. Nothing was
  sent.
- **Paper-key deploy path on preprod (7 October 2026):** passed. Practice contract
  `72fe33436d424fcf247919c8e2f0de224175cc55061739be0ebffb2d650f2f73`; a generated key
  written on two sheets and typed back from the second; finished from paper (option 4);
  retired provably from paper (option 33), transaction
  `0081bdbc410b9713dd8235ba20ad78ad6849552ff9a2d9470e0c43ca9ddc77a54c`
  (`docs/preprod-paper-key-practice.md`).

### What this revision changes in this document

- A second contract, the claims contract, is added under *The claims contract*, with its
  own fingerprint table, deployment details and self-assessment.
- The header now describes this contract: the brief description, the scores, the ledger
  layout, source and build, and the fingerprint table.
- The header as it stood on 13 September is kept as filed, with headings moved down one
  level and notes added, under *Superseded*.
- Short notes were added to the 13 September revision and the 16 September correction
  where Revision 4 changes or answers them. Nothing in them was deleted.
- **The state bounds came after a first draft of this revision.** That draft scored the
  previous build 3 on State-Space-at-Risk against the rubric, which blocks deployment.
  Instead of filing at 3, the contract was changed to cap state per anchored identity
  (*State*), and the score is now 2. An independent attack on the bounds found 7 issues:
  4 fixed in the contract (F1, F2, F3, F5) and 3 documented (F4, F6, F7). A second
  attack on the fixes found 2 more, both fixed in the contract (R1: 16 more obligation
  places per recovery; R2: an approved transfer publishes the new commitment), and P2 was
  documented. The change
  invalidated the earlier fingerprints, commit references and test results; this
  revision carries the new ones (build `ceb3a16`, fingerprints `e89a387`).
- **Attack round D (4 October)** and the preprod runs of 4 and 5 October were added. The
  maintenance authority section was corrected to match the code after round D: the key is
  never on disk, and retiring installs an empty committee that anyone can see on chain.
- **Two statements of the 13 September revision change,** approved by Hunter Roberts on
  6 October 2026 and confirmed by Makoto Steiner on 7 October 2026: the authority is not
  held jointly at launch,
  and relinquishment is no longer the intended end state (`docs/maintenance-policy.md`).
  Notes were added there.
- **6 October 2026:** the maintenance decisions brought up to date (two paper copies of one
  key, one per founder; no retirement date; the claims contract with none, as the tool
  enforces), confirmed by Makoto Steiner on 7 October 2026; VeilCore-run, the
  managed service, stated to be outside the contracts' authority; the build reproduced
  again and the contract tests re-run at `a3d1884`; two statements in earlier drafts
  corrected (the 1,024-active-licence test has run to completion; `verify.ts` does refuse
  a presentation that landed before its challenge); the mainnet blanks marked. The
  material an external auditor starts from is in `docs/audit/README.md`.
- **6 and 7 October 2026:** the partner package (`@veilcore/contracts`, which exposes no
  deploy or maintenance operation) passed 23 of 23 checks against both preprod contracts
  (`docs/partner-check-run-6oct.md`); one independent review of the mainnet-only operator
  checks (address pins, fingerprint gates, the claims deploy guard, the site preflights)
  found nothing serious, and its one medium and two lows were fixed in `799c765`, which
  changes no contract; and the paper-key deploy path ran on preprod for the first time:
  a generated key written on two sheets and typed back from the second, a deploy finished
  and the authority provably retired with the key typed from the first
  (`docs/preprod-paper-key-practice.md`).

### Founder sign-off

Both founders approved the decisions this revision records before it was filed upstream
(`midnightntwrk/midnight-improvement-proposals`, `deployments/veilcore.md`), and it is filed
before either contract is deployed to mainnet.

- **Hunter Roberts:** approved the maintenance policy, the two-copy key and the claims
  contract with no authority on 6 October 2026, and files this revision on 7 October 2026
- **Makoto Steiner:** confirmed the maintenance policy, the two-copy key and the claims
  contract with no authority on 7 October 2026
- **Filed upstream:** 7 October 2026, as a pull request to
  `midnightntwrk/midnight-improvement-proposals` replacing `deployments/veilcore.md`
