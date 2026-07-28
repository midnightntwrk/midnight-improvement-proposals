# Deployment Request: gyotak-temp-log

## 1. Summary

`gyotak-temp-log` is a Compact smart contract implementing a cold-chain provenance layer for flash-frozen seafood, deployed as the second contract in the GYOTAK traceability system (the first, `gyotak-catch`, is already Mainnet-approved and live via PR #96).

The contract records two categories of cold-chain evidence to the Midnight ledger, using a deliberate public/private disclosure split:

- **Storage temperature** (`recordStorageTemp`) — hourly storage temperatures are written to the ledger in plaintext. This is intentional: storage temperature is a quality-transparency signal for buyers, not a trade secret. Any third party (including AI agents) can independently decode and verify these temperatures from on-chain state without trusting GYOTAK's servers.
- **Quick-freeze certification** (`recordQuickFreeze`) — the contract asserts, in zero knowledge, that a fish lot reached the quick-freeze quality bar (below −15 °C at a measurement point between 30 and 35 minutes after entry) and discloses only a PASS/FAIL (1/0) result. The actual measured temperature curve and the −15 °C threshold's meaning are kept as private witness data, protecting the operator's freezing know-how while still proving the quality claim.

The contract also provides inventory snapshots (`recordInventorySnapshot`), owner rotation (`rotateOwner`), and public verification circuits (`verifyStorageTemp`, `verifyQuickFreeze`).

A full technical report is published as a defensive publication: TR-2026-011 (https://gyotak-tr.pages.dev/tr-2026-011/, CC BY 4.0).

### 1.1 Risk rubric

| Category | Score | Summary |
| --- | --- | --- |
| Privacy-at-risk | 1 | The contract implements a deliberate three-tier disclosure model. Tier 1 (public): storage temperatures are written in plaintext for quality transparency and are independently verifiable on-chain; quick-freeze PASS/FAIL results are public. Tier 2 (ZK-hidden): the actual measured temperatures behind a quick-freeze judgment are never disclosed — an observer cannot reconstruct whether the reading was −16 °C or −40 °C, protecting freezer performance data. Tier 3 (fully private): the meaning and derivation of the −15 °C threshold are off-chain business know-how. No personal data or PII is stored in any tier. |
| Value-at-risk | 1 | The contract holds no on-chain assets — no tokens, NFTs, or escrowed value. Adversarial misuse cannot result in user fund loss. The state at risk is bounded ledger maps of temperature/quick-freeze records (see § 3). |
| State-space-at-risk | 1 | All state-mutating writes are owner-gated: an unauthorized `recordStorageTemp` / `recordQuickFreeze` / `recordInventorySnapshot` attempt fails before any chain state change. Write volume is bounded by physical operations (hourly temperature readings, per-lot quick-freeze judgments, inventory-change snapshots), preventing unbounded ledger growth. |

Detailed justification with cross-references follows in §§ 3–5.

## 2. Contract at a glance

- Contract name: `gyotak-temp-log`
- Preprod contract address: `718081e52b0235eeab7ddd08dbdbfa5aca9b0d988abe9d05a03338508a6beda7`
- Initial owner PK: `bf4b78efeb9c1979d77e487d40a20daefa43663c3cbd93dcd578d73333e08100`
- Circuits:
  - `recordStorageTemp` — writes an hourly storage-temperature record (plaintext temperature) to the `storageRecords` ledger map.
  - `recordQuickFreeze` — takes the measured temperature as a private witness, asserts the measurement point falls within the quick-freeze time window (between 30 and 35 minutes after entry) and that the temperature offset is below the threshold, and discloses only a PASS/FAIL (`Uint<8>` 1/0) result.
  - `recordInventorySnapshot` — writes a freezer inventory snapshot (fish-lot slots) on inventory change.
  - `rotateOwner` — owner-gated owner rotation.
  - `verifyStorageTemp` — public verification of a stored temperature record.
  - `verifyQuickFreeze` — public verification of a quick-freeze result.

The contract is deployed by the DApp developer (GYOTAK / ECOSUS CO., LTD.), not by end users. Temperature data originates from SwitchBot sensors → Cloudflare Worker → the contract's record circuits.

## 3. State and threat model

Ledger state:
- `storageRecords: Map` — public hourly temperature records (`avgTempOffset` encoded as `10,000,000 + °C × 100`, stored in plaintext).
- `quickFreezeResults: Map` — quick-freeze judgments, storing only the PASS/FAIL result and record identifier; the underlying measured temperature is never written.
- Inventory snapshot state, owner PK.

Private witness data (never reaches chain):
- The measured temperature used in a quick-freeze judgment.
- The −15 °C threshold's semantic meaning and derivation.

Threat model:
- Unauthorized writes: every record circuit is owner-gated. An adversary lacking the owner secret cannot mutate ledger state; the attempt fails at proof/settlement.
- Witness leakage: the quick-freeze private witness (measured temperature) is held only in the operator's local environment and never serialized to chain; only the PASS/FAIL disclosure and the ZK proof reach the ledger.
- Value extraction: not applicable — the contract custodies no assets.

## 4. State-space justification

State growth is bounded by physical cold-chain operations:
- `recordStorageTemp` is invoked at most hourly per freezer.
- `recordQuickFreeze` is invoked once per fish lot at registration.
- `recordInventorySnapshot` is invoked on inventory change.

All are owner-gated, so an adversary cannot force unbounded writes. A `recordQuickFreeze` attempt that does not meet the quality bar returns a FAIL (`result = 0`) rather than reverting, and an unauthorized caller cannot write at all — both enforcement paths were exercised on Preprod (§ 5).

## 5. Preprod evidence

The Score-1 design has been deployed and exercised on Preprod (contract `718081e52b0235eeab7ddd08dbdbfa5aca9b0d988abe9d05a03338508a6beda7`):

- `recordQuickFreeze` PASS: tx `007757f2…`, block 1,402,635, `result = 1` (temperature below −15 °C within the window).
- `recordQuickFreeze` FAIL: tx `002aeda5…`, block 1,533,223, `result = 0` (temperature −10 °C, offset 9,999,000 ≥ 9,998,500) — confirming the quality bar is enforced, not merely recorded. The measured temperature was supplied as a private witness and does not appear on-chain.
- `verifyQuickFreeze` (PASS): tx `00ffcc2b…`, block 1,403,064, result confirmed on-chain.
- `verifyQuickFreeze` (FAIL): the FAIL record above was verified on-chain — `result = 0` reads back correctly, and the measured temperature used in the judgment is confirmed absent from public state: no −10 °C / offset-9,999,000 value can be recovered from the contract's on-chain data. This demonstrates the Tier-2 disclosure property in practice: the PASS/FAIL result is public, the underlying measurement is not.
- `recordStorageTemp`: several hundred hourly records written to Preprod. These plaintext temperatures are independently decodable from on-chain contract state by any third party: query the indexer's `contractAction(address).state`, scan for the `0x43` tag followed by a 3-byte little-endian value, filter to the valid range, and compute `°C = (value − 10,000,000) / 100`. This recipe was validated against the operator's database with a 10/10 match, and is published for AI-agent verification.

## 6. Operational posture

- The quick-freeze private witness (measured temperature) is a local file in the operator's environment and is never published.
- Deployment and transaction submission use a warm-started wallet from a synced seed; cold start is prohibited by operational policy.
- The contract shares one wallet seed with `gyotak-catch`, with a per-contract owner PK derived via a domain tag, so authorization is scoped per contract.

## 7. Implementation repository

- Implementation: https://github.com/ecosus-co/gyotak-temp-log (public, Apache 2.0)
- Technical report (defensive publication): TR-2026-011 — https://gyotak-tr.pages.dev/tr-2026-011/ (CC BY 4.0)
- Preprod evidence: included in the implementation repository under `preprod-evidence/`.

### Notes

- Implementation code is in the linked external repository per the MIP submission convention; this PR adds only the deployment request document.
- License: both this PR's file and the implementation repository are Apache License 2.0.
- `gyotak-catch` (the first GYOTAK contract) was approved via PR #96 and is live on Mainnet; this request covers the second contract, `gyotak-temp-log`, which requires separate authorization as Circuit Hub approval is per-contract.
