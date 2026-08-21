---
MPS: <Number> # assigned by editors
Title: Committee Bridge (mNIGHT to cNIGHT)
Authors: Karmoola
Status: Proposed
Category: Core
Created: 21-Aug-2026
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

NIGHT exists in two representations: cNIGHT on Cardano, where most supply resides today, and mNIGHT on the Midnight Network. Block production rewards under SOW-Q4-01 are distributed as mNIGHT, but exchange listings, existing holder balances, and much of the token’s liquidity remain on Cardano.

There is no protocol-native path from mNIGHT back to cNIGHT. Holders of mNIGHT cannot realize value on Cardano.

This MPS states the problem of the missing return path and what a solution must guarantee. It does not specify an implementation.

## Vision

A holder of mNIGHT can convert it to cNIGHT using published open source tooling, with no third-party bridge operator in the path. Conservation across the two representations is verifiable from public chain data. A conversion that fails leaves the holder able to recover their funds.

## Problem

NIGHT can move from Cardano to Midnight, but not back. There is no protocol-native way to convert mNIGHT to cNIGHT.

Value accumulates on Midnight as mNIGHT through block production rewards under SOW-Q4-01. Exchange listings, existing holder balances, and much of the token’s liquidity are in cNIGHT on Cardano. A holder of mNIGHT therefore holds a representation they cannot convert into the one the market uses.

The gap also makes movement into Midnight one-way. A holder converting cNIGHT to mNIGHT is committing to a representation with no exit, which is a reason not to convert in the first place.

## Use Cases

**Block producers paid in mNIGHT.** Rewards under SOW-Q4-01 arrive as mNIGHT, which cannot be converted to cNIGHT.

**Holders who moved to Midnight.** A holder who converted cNIGHT to mNIGHT has no way back.

## Goals

1. Any holder of mNIGHT on Midnight mainnet can initiate a conversion and receive the corresponding cNIGHT on Cardano mainnet, using published open source tooling (CLI or SDK), with no third-party bridge operator in the path.
2. Each release of cNIGHT is authorized by an attestation from the Midnight committee, and the released amount is cryptographically bound to the amount locked or burned on the Midnight side. Any mismatch in amount, decimals, or unit is rejected on-chain, not merely by an off-chain signer.
3. Conversions are replay-protected. A single Midnight-side lock or burn event can authorize at most one Cardano-side release, and attestations cannot be reused or split.
4. Total cNIGHT released by the bridge never exceeds total mNIGHT locked or burned through it. Conservation is independently verifiable from public chain data on both networks.
5. Failure handling is defined and tested. A conversion that cannot complete on Cardano leaves the holder able to recover their mNIGHT, with no state in which funds are unrecoverable on both chains.
6. Committee attestation thresholds, key rotation, and pause or halt procedures are documented, and a pause mechanism exists that can stop releases without loss of in-flight user funds.

## Copyright

This MPS is licensed under CC-BY-4.0.
