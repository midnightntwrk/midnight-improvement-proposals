---
MIP: 0006
Title: Peer-to-Peer Atomic Swaps
Authors: Andrew Fleming @andrew-fleming, Edward Alvarado <edward.alvarado@midnight.foundation>
Status: Proposed
Category: Standards
Created: 2026-02-12
Requires: MIP-0005
Replaces: none
License: Apache-2.0
---

<!--
 This file is part of midnight-improvement-proposals.
 Copyright (C) 2025-2026 Midnight Foundation
 SPDX-License-Identifier: Apache-2.0
 Licensed under the Apache License, Version 2.0 (the "License");
 You may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

This proposal specifies a scaffolding for peer-to-peer atomic swaps on Midnight,
inspired by [Dexie.space](https://dexie.space) on Chia.
It leverages Midnight's native transaction merging to enable trustless atomic swaps
without a custom smart contract.
Makers create partial (imbalanced) transactions offering tokens;
takers complete and submit them; the ledger handles all settlement logic.
This standard defines two payloads —
one published to a shared data-availability (DA) layer (recommended: Celestia, under a common namespace),
one served by indexers for discovery — and a reference indexer API.

Publishing to a shared DA namespace lets dApps share liquidity:
every UI, indexer, and bot reads the same offers,
so an offer made in one dApp is takeable in any other.

## Motivation

Midnight lacks a standardized mechanism for peer-to-peer token exchange.
A native P2P swap standard provides:

1. **Trustless exchange** — atomic swaps guarantee both parties receive their tokens or neither does.
   No counterparty risk.
2. **No smart-contract overhead** — swaps execute at the protocol level via native transaction merging,
   with no custom contract deployment or execution cost.
3. **Interoperability** — a standardized offer format lets multiple UIs, indexers,
   and tools share one liquidity pool.
4. **Privacy preservation** — on-chain activity stays private for shielded tokens 
(only revealing the token deltas, and nullifiers), for unshielded tokens the UTXOs are public.

## Specification

### Overview

The swap mechanism leverages Midnight's native support for imbalanced transactions.
A maker creates a partial transaction containing their spend authorization
and their expected payment outputs.
A taker completes it by adding their own input and output, then merges the transactions.

```text
MAKER'S PARTIAL TX (imbalanced):
  Inputs:  100 TokenA
  Outputs: 50 NIGHT to maker

TAKER'S PARTIAL TX (imbalanced):
  Inputs:  50 NIGHT
  Outputs: 100 TokenA to taker

MERGED TX (balanced):
  Inputs:  100 TokenA (maker), 50 NIGHT (taker)
  Outputs: 50 NIGHT to maker, 100 TokenA to taker
```

The merged transaction balances because each party's input funds the other's output.
Both parties' expected receipts are committed inside their own partial transactions,
so neither can be shortchanged.
The offer is carried by the single `Transaction` payload defined in MIP-0005,
which expresses shielded and unshielded swaps.

### Relationship to Offer Files (MIP-0005)

MIP-0005 defines the offer itself:
a proven, imbalanced `Transaction`, its raw-byte serialization,
and a bech32m string (`swapoffer1…`) for text sharing.
This MIP defines what wraps that offer for **publication** and for **discovery**:

- The **on-chain payload** carries the MIP-0005 offer as **raw bytes** (not the bech32m string)
  plus a minimal, untrusted wrapper.
- The **off-chain payload** is what an indexer serves:
  the offer plus derived discovery metadata (gives/wants, expiry) and indexer-observed fields.

> **Bytes, not bech32, on the DA layer.** bech32m is a display encoding (MIP-0005).
> Publishing the bech32m string to a DA layer wastes ~1.6× the bytes for no benefit,
> exactly as nobody submits a bech32 *address* on-chain.
> Publish the raw offer bytes; render bech32m only for humans.

### On-chain Offer Payload (published to the DA layer)

```typescript
interface OnchainOfferPayload {
  version: 1;

  // MIP-0005 payload: canonical serialized `Transaction` bytes (NOT bech32m).
  offer: Uint8Array;

  // Optional, UNTRUSTED free-form note. Placeholder until a ZK-authenticated
  // message bound to the UTXO secret is specified (see Future Work). Consumers
  // MUST NOT treat this as authenticated.
  unverifiedMessage?: string;
}
```

This is deliberately minimal.
It carries no `gives`/`wants` (derivable from the offer — see below),
no timestamps (not knowable from the offer),
and no public key or signature.
When serialized to a byte-oriented DA layer, `offer` is the payload's dominant content
and `unverifiedMessage`, if present, is a short message from the sender.

### Publishing to the data-availability layer

Makers publish the `OnchainOfferPayload` as a blob to a shared **data-availability (DA) layer**.
Publishing to a common, permissionless location —
rather than to a single dApp's private backend —
is what lets **dApps share liquidity**:
any UI, indexer, or bot reads the same stream of offers,
so no one service owns the order flow
and an offer created in one dApp is takeable in another.

This standard is **DA-agnostic**:
any blob store with a shared, readable keyspace qualifies,
and reimplementations MAY choose another.
In practice, implementations are converging on **[Celestia](https://celestia.org)**
to keep liquidity unified, and this document recommends it.
Offers are posted as Celestia blobs under a single well-known **namespace**
shared by all compliant implementations:

| Parameter | Value |
| --- | --- |
| DA layer | Celestia (recommended) |
| Namespace | version 0, 10-byte id suffix `mn-swap-v1` (see below) |
| Blob contents | serialized `OnchainOfferPayload` (raw MIP-0005 offer bytes, version + optional `unverifiedMessage`) |

- **Raw bytes, not bech32** — the DA blob carries the MIP-0005 offer as bytes
  (bech32m is a display encoding only; see MIP-0005).
- **One shared namespace = one shared liquidity pool.**
  All participants MUST agree on the namespace;
  per-dApp namespaces would re-silo liquidity, defeating the purpose.
  Celestia namespaces are 29 bytes — a version byte plus a 28-byte id —
  and for version-0 (user) namespaces the first 18 id bytes MUST be zero,
  leaving 10 freely chosen bytes.
  The shared namespace is version `0x00` with the 10-byte id suffix
  `0x6d6e2d737761702d7631` (ASCII `mn-swap-v1`);
  the full 29-byte namespace is `0x00`, followed by 18 zero bytes,
  followed by `6d6e2d737761702d7631`.
- **Permissionless read/write** — any maker posts;
  any indexer subscribes to the namespace, validates each offer
  (deserialize, derive `gives`/`wants`, enforce the two-sided rule, check liveness),
  and serves the `OffchainOfferPayload` for discovery (see Reference Indexer API).
  Celestia is recommended for its neutrality, low cost, and permissionless access.
- **Publication cost** — posting a blob is paid by the poster (the maker) by default;
  a dApp MAY subsidize or relay publication on the maker's behalf.
  Keeping offers minimal (MIP-0005) keeps this cost proportionally low.
- **Minimal coordination** — the namespace is the only value that must be standardized;
  everything else is implementation choice.

The DA layer is a *transport and broadcast* medium, not a settlement or ordering authority:
it makes offers discoverable, but settlement still happens on Midnight (see Overview)
and offer validity is always re-checked against Midnight state.

### Off-chain Offer Payload (served by indexers)

```typescript
type TokenKind = 'SHIELDED' | 'UNSHIELDED';
interface TokenLeg { token: string; amount: string; type: TokenKind; } // hex RawTokenType, stringified bigint

interface OffchainOfferPayload {
  version: 1;
  offerBech32: string;              // MIP-0005 bech32m rendering, for display
  unverifiedMessage?: string;       // echoed from the on-chain payload; UNTRUSTED
  computed: {                       // everything the indexer derives / observes
    gives: TokenLeg[];              // DERIVED from the offer's imbalances
    wants: TokenLeg[];              // DERIVED from the offer's imbalances
    expiresAt?: string;             // ISO 8601, from the earliest intent TTL, when present
    inputNullifiers: string[];      // shielded input nullifiers (hex) for liveness
    firstSeenAt: string;            // ISO 8601, when the indexer first saw the offer
    status: 'live' | 'consumed' | 'expired';
  };
}
```

Example:

```json
{
  "version": 1,
  "offerBech32": "swapoffer1…",
  "unverifiedMessage": "hello",
  "computed": {
    "gives": [{ "token": "4a8f…", "amount": "100", "type": "SHIELDED" }],
    "wants": [{ "token": "0200…", "amount": "50",  "type": "SHIELDED" }],
    "expiresAt": "2026-06-30T12:00:00Z",
    "inputNullifiers": ["7c1d9b…"],
    "firstSeenAt": "2026-06-16T09:30:00Z",
    "status": "live"
  }
}
```

Everything under `computed` is produced by the indexer —
either derived from the offer bytes (`gives`, `wants`, `expiresAt`, `inputNullifiers`)
or observed by the indexer (`firstSeenAt`, `status`).
None of it is trusted from the maker.
Only `offerBech32` and the untrusted `unverifiedMessage` carry over from what was published.

#### Field derivation

| Field | Source |
| --- | --- |
| `offerBech32` | MIP-0005 bech32m encoding of the published offer bytes |
| `computed.gives` / `computed.wants` | Net imbalances of the offer, tagged `SHIELDED`/`UNSHIELDED` (authoritative) |
| `computed.expiresAt` | The earliest intent TTL (when the offer carries an intent), or the time the offer's proof roots leave the node's root-recency window |
| `computed.inputNullifiers` | Nullifiers of the offer's shielded inputs — the keys an indexer watches to mark an offer `consumed` |
| `computed.firstSeenAt` | The indexer's clock when the offer was first seen on the DA layer |
| `computed.status` | Indexer bookkeeping: `live` until a backing input is spent (`consumed`) or the offer expires (`expired`; see Offer lifetime) |

### Deriving gives / wants

`gives` and `wants` are computed from the offer, not supplied by the maker.
For each token type, compute the net imbalance `delta = (inputs) − (outputs)`:

- Shielded: use `ZswapOffer.deltas` (which is already `inputs − outputs` per token),
  summed across guaranteed and fallible offers.
- Unshielded: sum `UtxoSpend.value` (inputs) minus `UtxoOutput.value` (outputs) per token,
  across all intents' guaranteed and fallible unshielded offers.

Then `delta > 0` ⇒ the maker is a net spender ⇒ the token belongs in `gives`;
`delta < 0` ⇒ net receiver ⇒ `wants` (with amount `−delta`).
Each leg is tagged with its value layer:
`SHIELDED` for deltas that come from a zswap offer,
`UNSHIELDED` for deltas summed from an intent's unshielded inputs/outputs.

Because the terms are derived from the proven transaction, they are authoritative:
there is no separate maker-asserted `gives`/`wants` that could disagree with the offer.
Indexers MUST derive these fields; they MUST NOT accept them as trusted input.

### Offer lifetime

Two independent ledger rules bound how long a published offer stays takeable:

- **Intent TTL.** Each intent's `ttl` must satisfy `t_block ≤ ttl ≤ t_block + global_ttl` at apply time,
  where `global_ttl` is a network parameter.
- **Merkle-root recency.** A shielded input's proof commits to a Zswap Merkle-tree root,
  and the ledger accepts only roots seen within its root-recency window (a network parameter).
  A purely shielded offer may carry no intent (and hence no TTL) at all;
  its effective lifetime is still bounded by this window.

Indexers SHOULD mark offers `expired` when the earliest intent TTL passes.
`firstSeenAt` is an indexer-side lower bound on an offer's age, not a maker claim.

### Removed: Authentication (`auth`)

The previous revision defined an optional BIP-340 Schnorr `auth` block over the offer JSON.
**It has been removed** because it was both unsound and privacy-harming:

1. **Not binding.** The signature was over the wrapper JSON,
   using a key unrelated to the secret that controls the offer's UTXO.
   Anyone can strip the wrapper from a published offer,
   re-wrap it with their own key and signature, and republish.
   The claim that it "prevents metadata tampering" was therefore false —
   an attacker simply produces a validly-signed wrapper of their own.
2. **Privacy leak.** Including the maker's public key in a public discovery record
   deanonymizes the maker and links their offers,
   defeating the purpose of a private offer file.

Until a sound primitive exists (see Future Work),
a maker MAY attach a free-form `unverifiedMessage`.
It is explicitly untrusted:
implementations MUST NOT display it as authenticated
and MUST NOT rely on it for any decision.

### Offer Validation

Takers SHOULD verify that an indexer's advertised `gives`/`wants`
match the transaction's actual imbalances before completing a swap —
the proven transaction is authoritative.
To validate:

1. Deserialize the offer (`Transaction`) from the offer bytes / bech32m string.
2. Compute the net imbalances (see Deriving gives / wants).
3. Verify they match the advertised `gives`/`wants`.

Indexers MUST perform this derivation themselves
and SHOULD reject or flag any offer whose imbalances cannot be computed.

#### An offer MUST be two-sided

A valid swap offer's net imbalance MUST contain **both**
at least one give (a token with positive net delta)
**and** at least one want (a token with negative net delta).
An offer whose imbalance has a give but **no want** is a *give-only* offer:
completing it forfeits the maker's tokens with no counterparty return —
it is a giveaway, not a swap.
Indexers MUST reject give-only offers, and takers MUST NOT settle one.

### Segment placement (informative)

Compliant P2P swap offers carry all value in the **guaranteed section**,
so both legs are applied atomically in the guaranteed execution phase
(all-or-nothing; no value-level partial success).
Concretely, wallet-built swap offers place the shielded leg in `guaranteed_coins` (segment 0)
and the unshielded leg in an intent's `guaranteed_unshielded_offer`.
The `fallible_coins` / `fallible_unshielded_offer` slots are not used for P2P swaps;
they exist for value legs bound to contingent contract execution, which is out of scope here.

### Cancellation (informative)

There is no explicit cancellation method:
once published, an offer stays takeable until it is consumed or expires.
A maker can only invalidate it indirectly,
by spending a backing input (or completing the swap themselves).
This is not handled in this MIP.

### Fee Handling

Either the platform or the taker pays transaction fees.
The maker cannot know in advance how much gas the transaction will need,
nor when it will happen,
and pre-paying would require locking a DUST UTXO into the swap.
Fee-splitting is out of scope.

### Reference Indexer API

Compliant indexers MUST implement the following REST API,
and SHOULD additionally expose a streaming endpoint
(e.g. a WebSocket at `/v1/offers/stream`)
that pushes new and updated offers as they are observed on the DA layer.
Publication carries the on-chain payload; queries return the `OffchainOfferPayload`.
The raw offer bytes live on the DA layer.

#### POST /v1/offers — publish

The request body carries the **on-chain payload** —
the same bytes that go to the DA layer, so the two can never disagree:

```json
{ "offer": "<base64-encoded OnchainOfferPayload bytes>" }
```

Response: `{ "offerId": "string" }`

Validation:

- MUST deserialize the offer and derive `gives`/`wants` itself
  (there are no maker-asserted terms to trust).
- MUST reject malformed payloads, undeserializable offers, and give-only offers.
- SHOULD reject duplicates.

#### GET /v1/offers — query

Indexers SHOULD support filtering by token type (in either `gives` or `wants`) and pagination.
Exact query semantics are left to implementations.

Response: `{ "offers": [ { "offerId": "string", "offer": { /* OffchainOfferPayload */ } } ], "total": 0 }`

#### GET /v1/health — health check

### Versioning

Both payloads carry an explicit `version` field (this revision: `1`).
Incompatible changes to either payload schema increment the version
and are specified by a superseding MIP per MIP-0001.
The embedded offer bytes are versioned independently by the ledger serialization (see MIP-0005).

## Rationale

### On-chain / off-chain split

The previous single `OfferPayload` conflated two concerns:
what is *published* and what is *discovered*.
Publishing wants only the offer bytes
(everything else is derivable, unknowable, or privacy-harming).
Discovery wants the derived, human-useful fields.
Separating them keeps the DA payload minimal (cheaper, more private)
and lets indexers enrich freely.
This mirrors the address model: bytes on the wire, bech32m for humans.

### Protocol-level settlement

Midnight natively supports transaction merging with homomorphic commitment balance checks.
The ledger already verifies that inputs carry valid proofs/signatures,
that commitments balance per token type, and that no double-spends occur.
This provides atomic-swap semantics with no custom logic.
A smart contract would add deployment complexity and cost without benefit.

### Metadata is derived or observed, never trusted

`gives`/`wants` are computed from the proven transaction
and **tagged with their value layer** (`SHIELDED`/`UNSHIELDED`),
so they cannot disagree with reality.
`expiresAt` comes from the intent TTL, when present.
`inputNullifiers` are read from the offer's shielded inputs
and are what an indexer watches to flip `status` to `consumed`.
`firstSeenAt` replaces the old `createdAt`:
an offer's creation time is **not knowable from the offer**,
and any maker-asserted timestamp would be unverifiable.
The only meaningful, verifiable time is when an indexer first observed the offer.

### Indexer as discovery layer

The **shared DA namespace is the source of truth** for which offers exist;
an indexer is just a convenience query layer over it.
Anyone can run one by subscribing to the namespace,
so the order flow is not owned by any single party —
that is what makes liquidity shared rather than siloed.
A given indexer offering REST discovery is a privacy leak
(it can observe IPs and browsing patterns) but does not affect security:
the cryptographic guarantees hold regardless of indexer behavior,
and a user can read the DA namespace directly, run their own indexer, or use Tor.
There is intentionally no DELETE endpoint —
cancellation is spending the backing input;
indexers MAY monitor nullifiers/UTXOs to retire stale offers.

## Path to Active

### Acceptance Criteria

- The shared DA namespace value is finalized and recorded in this document.
- A reference indexer implementation subscribes to the namespace,
  derives `gives`/`wants`, enforces the two-sided rule,
  and tracks liveness via nullifiers/UTXO references.
- At least two independent client implementations (UI, bot, or wallet integration)
  create and take offers through the shared namespace on a test network.
- The wallet SDK can build compliant swap offers.

### Implementation Plan

1. **Core library** — build offer payloads, derive `gives`/`wants`,
   complete and merge swaps, cancel by spending backing inputs.
2. **Reference indexer** — REST API serving `OffchainOfferPayload`,
   with gives/wants derivation and nullifier/UTXO monitoring.
3. **Reference UI** — browse, create, and complete swaps.

**Dependencies**: MIP-0005 codec, ledger transaction primitives, wallet API.

## Backwards Compatibility Assessment

This proposal introduces a new standard and modifies no existing protocol.
Relative to the previous revision it changes the payload shape
(split into on-chain/off-chain, `auth` removed,
derived fields nested under `computed` with type-tagged legs and `inputNullifiers`,
`createdAt` → `firstSeenAt`).
Since neither revision was deployed, there is no persisted data to migrate.
Implementations adopting this standard interoperate with each other
but not with the earlier draft.

## Security Considerations

### Transaction-level guarantees

Swap security derives from the proven transaction, not from any wrapper.
The maker's partial transaction cryptographically commits to the inputs spent
and the outputs received (token types, amounts, recipient).
The taker cannot modify these; the ledger enforces balance on merge.
Either both parties receive exactly what was committed, or the transaction fails.

### Metadata integrity

`gives`/`wants` are derived from the transaction,
so a malicious indexer that misrepresents them is caught by any taker
that re-derives from the offer (which they SHOULD do before completing).
`unverifiedMessage` is untrusted by construction.

### Stale offers

Offers become invalid when a backing input is spent elsewhere,
when an intent TTL passes,
or when the shielded proofs' Merkle roots age out of the ledger's recency window
(see Offer lifetime).
Takers discovering stale offers fail at submission;
indexers MAY monitor nullifiers/UTXOs and expiry bounds to retire them proactively.

### Denial of service

Indexers MAY enforce a minimum offer value;
the natural reference floor is the cost of publishing the blob to the DA layer —
an offer worth less than its own publication cost is presumptively spam.

## Future Work: Authenticated messages without key leakage

The valuable primitive the removed `auth` block failed to provide is:
a proof that a message accompanies an offer
and was authorized by the same secret that controls the offer's UTXO,
**without revealing a public key**.
This is expressible as a zero-knowledge proof
(prove knowledge of the spending secret for one of the offer's inputs
and that it signs the message)
and would let a maker attach a trustworthy note while preserving privacy.
The concrete goal is to attach an encrypted message (ciphertext) to the payload,
readable by its intended audience yet bound to the offer's owner.
No satisfying construction is known yet:
a plain signature leaks the maker's key material,
and a proof that simultaneously authenticates and carries a text
without leaking identity remains an open design problem.
Specifying it is deferred to a future MIP;
until then, `unverifiedMessage` is the untrusted placeholder.

## Testing

| Scenario | Expected Outcome |
| --- | --- |
| Happy-path swap | Both parties receive tokens |
| Advertised gives/wants ≠ derived | Indexer rejects / taker detects mismatch |
| Give-only offer published | Indexer rejects; taker refuses to settle |
| `unverifiedMessage` present | Surfaced as untrusted; ignored for decisions |
| On-chain payload round-trip | Raw bytes recreate the exact `Transaction` |
| Off-chain payload derivation | gives/wants equal the transaction's net imbalances |
| Cancelled offer | Take attempt fails gracefully |
| Expired offer (TTL or root recency) | Take attempt fails; indexer marks `expired` |
| Network failure during take | Transaction not submitted, tokens safe |

## References

- [Dexie.space](https://dexie.space) — offer-based P2P trading on Chia
- [Celestia](https://celestia.org) — recommended data-availability layer
- [Chia offer files](https://docs.chia.net/wallets/offers)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement,
which includes the assignment of copyright for your contributions to the Foundation.
