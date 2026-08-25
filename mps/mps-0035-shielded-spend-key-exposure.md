---
MPS: "0035"
Title: Shielded Spend Authorization Requires Exposing the Spend Key
Authors: Ricardo Rius <riusricardo>
Status: Proposed
Category: Core
Created: 20-Aug-2026
Requires: none
Replaces: none
MIP: none
---

<!--
 Copyright 2026 Midnight Foundation

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

Producing a Zswap shielded spend proof requires the coin secret key as a private witness inside the spend circuit. The nullifier is defined as a hash keyed by that secret, so the circuit cannot be satisfied without it. Whoever runs the prover therefore holds full spend authority over every coin under that key. Today the wallet serializes the key into the proving payload and posts it in cleartext to the proof server.

Three consequences follow. Any hosted proving service is a custodian of user funds, so no safe hosted proving product can exist. The only safe deployment is a self-hosted prover beside the wallet, which requires hundreds of megabytes of proving parameters and heavy compute and therefore excludes mobile and light clients. Hardware wallets are structurally impossible for shielded funds, because a secure element can sign but cannot run a SNARK, and the key must exist in cleartext wherever the prover runs.

The same ledger already authorizes unshielded inputs with a signature over the intent digest and never moves those keys. Shielded spends are the outlier. This MPS describes the gap and its impact; it does not prescribe a construction.

## Vision

Shielded spending is authorized the way every other spend on the network is authorized: by a signature the keyholder produces locally, over the transaction they intend to send.

Proof generation becomes untrusted work. A wallet can hand a proving job to any machine, including a public service, and the worst that operator can do is decline the job. Provers become fungible infrastructure rather than custodians.

The spend key does the work a spend key should do and nothing more. It signs. That makes it small enough to live in a secure element, so shielded balances can be held in hardware wallets from ordinary vendor firmware. It makes mobile and light clients first-class, since the wallet-side cost of authorizing a spend is milliseconds of curve arithmetic with no parameter download.

Every on-chain property users depend on stays intact: double-spend prevention, value hiding, recipient hiding, replay protection, and a single shared anonymity set across all shielded notes.

## Problem

### The spend key is a circuit input

The v1 spend circuit takes the secret key as an argument (`midnight-ledger/zswap/zswap.compact`). The nullifier is `SHA-256("midnight:zswap-cn[v1]" ‖ coin ‖ sk)`, computed in-circuit, so a valid proof is impossible without handing the prover `sk`.

| Fact | Where |
| --- | --- |
| The wallet writes the raw secret key as the first field elements of the proving payload (`ProofPreimage.inputs`) | `midnight-ledger/zswap/src/construct.rs`, serialization at `midnight-ledger/coin-structure/src/transfer.rs` |
| That payload is POSTed in cleartext to the proof server `/prove` endpoint | `midnight-ledger/proof-server/src/endpoints.rs` |
| The nullifier is keyed by that same key, so holding it grants full spend authority over every coin under the key | `midnight-ledger/coin-structure/src/coin.rs` |
| The `sign` (claim) circuit carries the same exposure | `midnight-ledger/zswap/src/construct.rs` |
| Outputs are unaffected; they carry only recipient public data | `midnight-ledger/zswap/src/construct.rs` |

### Impact

**Hosted proving cannot exist safely.** Any operator running a proof server for third parties custodies their funds. This removes an entire category of infrastructure from the ecosystem.

**Light clients are excluded.** The only safe configuration is a prover co-located with the wallet. Phones, browser extensions, and embedded clients cannot meet that bar.

**Hardware wallets as we know them are impossible, not merely unimplemented.** Because the key must be witnessed inside the circuit, it has to exist in cleartext on a machine that can run or reach a prover. A secure element can sign but can never run a SNARK. No vendor firmware can close this gap. It is a property of the circuit, not of any wallet implementation.

**Every spend widens the exfiltration surface.** The key crosses a process or network boundary on each spend. One compromised prover, or one TLS-terminating middlebox in a nominally local deployment, leaks it permanently.

**The ledger is internally inconsistent.** Unshielded inputs are authorized by a signature over `Intent::data_to_sign(segment_id)` and verified at `midnight-ledger/ledger/src/verify.rs`. Shielded inputs are authorized by shipping the key. Two classes of value on the same chain have different custody models, and the private one is weaker.

## Use Cases

**Mobile wallet holding shielded funds.** A user wants to spend from their phone. Today the wallet must either bundle a prover the device cannot run or send the spend key to a remote one. Neither is shippable, so shielded balances remain a desktop feature.

**Hosted proving service.** An infrastructure provider wants to offer proving as a paid endpoint, the way RPC providers offer node access. Today accepting a proving request means accepting custody of the requester's funds. The provider cannot offer the service and the user cannot use it.

**Hardware wallet support.** A user holds unshielded funds on a hardware wallet and expects the same protection for shielded funds. There is no path to that today on any vendor's device, because the key cannot be confined to hardware that cannot run a prover.

**Institutional separation of duties.** An operator wants signing authority and proving compute held by different systems, with different trust levels and different audit boundaries. Today both roles collapse into whichever machine sees the key.

**Third-party wallet vendors.** A vendor integrating Midnight must reason about a key that leaves the device on every spend. Reviewers and security teams treat this as a blocking finding, slowing or preventing integrations.

## Goals

1. A shielded spend proof can be produced by an untrusted machine. The keyholder's secret never leaves the wallet.
2. What the prover receives cannot be used to build any transaction other than the one the wallet built.
3. Wallet-side authorization cost is signature-grade: single-digit milliseconds on commodity mobile hardware, with no parameter download and no proving key.
4. Authorization work is confinable to a secure element, so hardware wallet support becomes ordinary vendor firmware work rather than research.
5. Every existing on-chain property is preserved: double-spend prevention, value hiding, recipient hiding, replay protection, and a shared anonymity set with existing notes.
6. No new cryptographic assumptions and no new trusted setup. Solutions should reuse primitives already present in `midnight-ledger` and `midnight-zk`.
7. Wallet flow, transaction structure, on-chain lifecycle, and validator behavior remain recognizably identical to today.
8. Shielded spends reach parity with unshielded inputs, which already authorize by signature over the intent digest.

Goals 1 through 3 are the critical set. Goals 6 and 7 constrain acceptable solutions rather than defining success.

### Non-goals

Contract-owned shielded coins (`ContractAddress` spends) contain no user secret and are out of scope. No new authorization concepts such as permits, allowances, or spending policy. No changes to fees, viewing keys, or the encryption path.

## Expected Outcomes

Proof generation becomes a commodity service. Multiple independent provers can serve the same wallet, none of them trusted, which improves both availability and censorship resistance.

Shielded funds become spendable from phones, browser extensions, and other light clients, which is the precondition for shielded balances being the default rather than an advanced option.

Hardware wallet vendors gain a viable integration path. The division of labor follows the risk: the secure element holds the spend key and emits only signatures, the host and any remote prover do the heavy untrusted work, and host compromise costs privacy rather than funds.

Wallet integration review gets simpler. The security question changes from "who can see the key" to "who signed the transaction," which is a question every wallet team already knows how to answer.

The ledger becomes internally consistent, with one authorization model across shielded and unshielded value.

## Recommended MIPs

**MIP: VRF-based nullifier and signature-authorized shielded spend.** Redefine the nullifier so it is verifiable against public-key material rather than computed from a secret witness, and authorize the spend with a locally produced signature that also certifies the nullifier. Addresses the root cause in the Problem section and Goals 1, 2, 3, and 5. Should specify note commitment, address format, nullifier derivation, the wallet-side signing procedure, the circuit constraint set, and the validator changes required, and should include a benchmark gate against the v1 circuit.

**MIP: Hardware wallet signing interface for shielded spends.** Specify the device-side interface: what the secure element holds, what it emits, what the host may request without gaining spend authority, and the transcript and encoding rules a vendor firmware app must implement. Addresses Goal 4.

## Acknowledgements

Ricardo Rius authored this problem statement. Thanks to the Midnight ledger, cryptography, and wallet teams for review of the underlying analysis.

## Copyright

This MPS is licensed under CC-BY-4.0.
