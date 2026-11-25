---
MPS: "xxxx"
Title: High-Volume Income Distributions for Institutional Tokenised Products
Authors: "Jalal Hannan <Jalal-1>"
Status: Draft
Category: Ecosystem
Created: 25-Nov-2026
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

An interest-bearing tokenised product can owe different amounts to thousands of holders on the same payment date. An issuer choosing a payment window needs to know how many of those obligations its paying agent can settle and reconcile by the cutoff. If processing stops partway through, the agent also needs to determine which holders have already been paid before it resumes.

This MPS uses a hypothetical coupon distribution to 50,000 holders as a reference case. A single corporate action creates 50,000 payment obligations. Each has an entitled holder, a settlement asset and an amount due. The starting point is a funded distribution with an authorised list of those entitlements and a window defined by the product. The question is whether the paying agent can complete the distribution on Midnight within that window, including recovery from an interruption, while preserving the product’s confidentiality requirements.

## Vision

Before a payment date, a paying agent can demonstrate the volume it can support under stated operating conditions. During the run, it can identify outstanding obligations. At close, the issuer can verify that each holder received the amount due.

## Problem

**A paying agent needs a reproducible way to establish whether it can settle and reconcile a large set of individual entitlements by a defined cutoff, including when the outcome of some submitted payments is temporarily unknown.**

Midnight’s shielded wallet interface accepts multiple payment outputs. To plan a distribution, the agent also needs to know how long it takes from releasing the instructions to reconciling the last finalised credit. A payment can settle while its acknowledgement is lost. On restart, the agent must resolve that payment’s outcome before deciding whether to retry it.

An event total alone can hide errors. If two holders are owed equal coupons, paying one twice and omitting the other produces the expected total. The paying agent therefore needs evidence at the level of each entitlement. For a confidential product, an authorised reviewer must be able to check those payments without publishing holders’ positions or individual amounts.

## Use Cases

- **Coupon on a tokenised bond.** An issuer approves a coupon distribution to 50,000 eligible holders whose principal holdings differ. Its paying agent must credit each holder with the corresponding coupon in the designated settlement asset by the product’s cutoff.
- **Cash dividend on tokenised shares.** A declared dividend creates an entitlement for each eligible position on the record date. If holdings change before the distribution finishes, the agent must still reconcile payments to the authorised record-date entitlements, including any payments completed after a restart.

## Goals

1. **Meet the payment window.** Use 50,000 valid entitlements with varying amounts and declare the window before the run. Measure elapsed time from release of the instructions until all credits are final and reconciled. For a window of *W* hours, the required average is 50,000 ÷ *W* completed holder payments per hour.
2. **Recover safely.** Repeat the run with an interruption of stated duration after submission and before the agent records the result. Following restart, every entitlement must settle exactly once for the correct amount by the declared cutoff.
3. **Verify correctness and confidentiality.** An authorised reviewer can match every final credit to its entitlement and reconcile the total paid in the settlement asset to the total due. Verification must respect the product’s disclosure rules for holders, operators and public observers.
4. **Make capacity measurable.** Publish a synthetic workload, the operating conditions and results for both runs. Report completed payments by cutoff, time spent in preparation, settlement and reconciliation, total elapsed time and cost per completed payment so another operator can repeat the assessment.

## Expected Outcomes

Issuers can choose a payment window based on demonstrated capacity. If a run misses its cutoff, the issuer can identify which entitlements still require settlement or confirmation. Operators and maintainers can use the stage timings to locate the delay and assess whether a change improves the next run.

## Open Questions

- Which payment windows and background network loads should a reference assessment cover?
- What interruption timings and durations are representative of a paying agent’s operating conditions?
- What is the minimum evidence a holder, a paying agent and an independent reviewer each need to resolve a disputed payment under the product’s disclosure rules?

## Recommended MIPs

None proposed at this stage. Results from the reference workload can identify a specific gap for a follow-on MIP.

## References

- [DTC Distributions Service Guide](https://www.dtcc.com/-/media/Files/Downloads/legal/service-guides/Distributions-Service-Guide.pdf)
- [Midnight Shielded Wallet API, transfer transaction outputs](https://github.com/midnightntwrk/midnight-wallet/blob/d0c2133af2d78678698b8960dd866153a0405598/packages/shielded-wallet/src/ShieldedWalletAPI.ts#L249-L261)

## Acknowledgements

None at draft stage.

## Copyright

This MPS is licensed under CC-BY-4.0.
