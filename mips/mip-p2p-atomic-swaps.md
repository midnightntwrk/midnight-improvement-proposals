---
MIP: (assigned by a MIP Editor) \
Title: Peer-to-Peer Atomic Swaps \
Authors: Andrew Fleming @andrew-fleming \
Status: Proposed \
Category: Standards \
Created: 2026-02-12
---

## Abstract

This proposal specifies a scaffolding implementation for peer-to-peer atomic swaps on Midnight,
similar to [Dexie.space](https://dexie.space) on Chia.
The architecture leverages Midnight's native Zswap transaction merging to enable trustless atomic swaps without requiring a custom smart contract.
Makers create partial transactions that offer tokens, takers complete and submit them,
and the ledger handles all settlement logic.
This standard defines the offer payload schema, authentication mechanism, and reference indexer API.

## Motivation

Midnight currently lacks a standardized mechanism for peer-to-peer token exchanges.

A native P2P swap standard provides several benefits:

1. **Trustless exchange**: Atomic swaps guarantee that either both parties receive their tokens or neither does—no counterparty risk.
2. **No smart contract overhead**: By leveraging Zswap's built-in transaction merging,
swaps execute at the protocol level without custom contract deployment or execution costs.
3. **Interoperability**: A standardized offer format enables multiple UIs,
indexers, and tools to participate in the same liquidity pool.
4. **Privacy preservation**: On-chain activity remains shielded;
only offer metadata on the discovery layer is public.

## Specification

### Overview

The swap mechanism leverages Zswap's native support for imbalanced transactions.
Makers create partial transactions containing their spend authorization and their expected payment output.
Takers complete these by adding their own input and output, then merging the transactions.

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

The merged transaction balances because the maker's input (100 TokenA) funds the taker's output,
and the taker's input (50 NIGHT) funds the maker's output.
Both parties' expected receipts are committed to in their respective partial transactions,
ensuring neither can be shortchanged.

### Relationship to Offer Files (MIP-?)

This MIP builds upon the Offer Files MIP,
which specifies the bech32 encoding for serializing the underlying `zswap::Offer` type.
The Offer Files MIP defines:

- Binary serialization of the core offer (proven partial transaction)
- Bech32 encoding with `zswapoffer` HRP for text-based sharing

This MIP extends that foundation by specifying:

- An application-layer payload wrapping the serialized offer
- Metadata (pricing, expiration)
- Authentication via [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) signatures
- Discovery protocol (indexer API)

The `transaction` field in this MIP's payload contains the bech32-encoded offer as specified in the Offer Files MIP.

### Offer Payload Schema

All compliant implementations MUST use the following offer payload structure:

```typescript
interface OfferPayload {
  version: 1;

  // Serialized proven transaction (bech32 encoded per Offer Files MIP)
  transaction: string;

  // What the maker wants in return
  wants: {
    token: string;   // Token type identifier
    amount: string;  // Stringified bigint
  };

  // What the maker is giving
  gives: {
    token: string;
    amount: string;
  };

  // Optional metadata
  metadata?: {
    createdAt?: string;      // ISO 8601 timestamp
    expiresAt?: string;      // ISO 8601 timestamp (informational only)
    makerNote?: string;      // Arbitrary message (max 256 chars)
  };

  // Authentication
  auth: {
    signerPublicKey: string;  // 32-byte x-only pubkey (hex)
    signature: string;        // 64-byte Schnorr signature (hex)
    scheme: 'schnorr-bip340';
  };
}
```

### Field Requirements

| Field | Required | Description |
| ------- | ---------- | ------------- |
| `version` | Yes | Must be `1` for this specification |
| `transaction` | Yes | Bech32-encoded offer per Offer Files MIP |
| `wants.token` | Yes | Token type identifier the maker wants |
| `wants.amount` | Yes | Amount wanted as stringified bigint |
| `gives.token` | Yes | Token type identifier being offered |
| `gives.amount` | Yes | Amount offered as stringified bigint |
| `auth.signerPublicKey` | Yes | Public key used for signature verification |
| `auth.signature` | Yes | BIP-340 Schnorr signature over offer |
| `auth.scheme` | Yes | Must be `schnorr-bip340` |

### Offer Authentication

Offers MUST be signed using BIP-340 Schnorr signatures.
The signature covers the entire offer payload (excluding the auth field itself),
preventing metadata tampering.

#### Signing Payload Construction

The signing payload MUST be constructed as follows:

1. Create a copy of the offer without the `auth` field
2. Serialize to canonical JSON per [RFC8785](https://www.rfc-editor.org/rfc/rfc8785)
3. Compute SHA-256 hash of the serialized bytes
4. Sign the 32-byte hash using BIP-340 Schnorr

```typescript
function createSigningPayload(offer: Omit<OfferPayload, 'auth'>): Uint8Array {
  const canonical = canonicalize(offer); // RFC 8785
  return sha256(new TextEncoder().encode(canonical));
}
```

#### Verification Requirements

Both indexers and takers MUST verify signatures:

- **Indexers**: MUST reject offers with invalid signatures on publish
- **Takers**: MUST verify signatures before completing swaps

### Cancellation

To cancel an offer, makers spend the backing UTXO to themselves.
This consumes the nullifier, making the original offer invalid.
No special protocol support is required.

### Fee Handling

The submitter (taker) pays transaction fees.
Fee-splitting patterns are out of scope for this specification.

### Reference Indexer API

Compliant indexers SHOULD implement the following REST API:

#### POST /offers

Publish a new offer.

**Request:**

```json
{
  "offer": { /* OfferPayload */ }
}
```

**Response:**

```json
{
  "offerId": "string"
}
```

**Validation:**

- MUST verify signature before accepting
- MUST reject malformed payloads
- SHOULD reject duplicate offers

#### GET /offers

Query available offers.

**Parameters:**

| Parameter | Type | Description |
| ----------- | ------ | ------------- |
| `givesToken` | string | Filter by token being sold |
| `wantsToken` | string | Filter by token wanted |
| `minAmount` | string | Minimum amount offered |
| `maxAmount` | string | Maximum amount offered |
| `limit` | number | Pagination limit |
| `offset` | number | Pagination offset |

**Response:**

```json
{
  "offers": [
    {
      "offerId": "string",
      "offer": { /* OfferPayload */ }
    }
  ],
  "total": 0
}
```

#### DELETE /offers/:offerId

Remove an offer (maker only).

#### GET /health

Health check endpoint.

## Rationale

### Protocol-Level Settlement

Midnight's Zswap protocol natively supports transaction merging with homomorphic commitment balance checks.
The ledger already verifies that all inputs have valid ZK proofs,
that input commitments equal output commitments per token type, and that no double-spends occur.
This provides atomic swap semantics without custom logic.
A smart contract would add deployment complexity, execution costs,
and potential blockers without meaningful benefit.
By operating at the protocol level, swaps inherit the full security guarantees of the base layer.

*Alternatives considered: None. The point of the protocol is to leverage the existing architecture.*

### Imbalanced Transactions

The maker creates a transaction containing their spend authorization and their expected payment output.
This transaction is imbalanced — it outputs a token type (the payment) that it doesn't input.
The taker creates a complementary imbalanced transaction with their input and their expected receipt.
When merged, the two transactions balance.
Critically, because the maker's expected payment is included as an output in their proven transaction,
the taker cannot modify the payment terms.
The maker's proof commits to receiving a specific amount of a specific token.
This provides on-chain enforcement of both sides of the swap terms.

*Alternatives considered: None. The protocol was designed to leverage imbalanced transactions.*

### Authenticating Offers

BIP-340 Schnorr signatures provide a standardized,
well-audited mechanism with compact 64-byte signatures and wide library support.
The signature prevents tampering with offer metadata,
ensuring the `gives` and `wants` fields accurately reflect the underlying proven transaction.
Both indexers and takers verify signatures, providing defense in depth.

*Alternatives considered: No authentication (vulnerable to metadata tampering), on-chain commitment (unnecessary complexity).*

### Metadata in Application Layer

The underlying zswap::Offer (as defined in the Offer Files MIP) contains the proven partial transaction,
which includes the maker's input, their expected payment output, and the receiving address.
This MIP adds application-layer metadata for discovery and authentication.
This separation allows the base offer format to remain focused on the cryptographic commitment while enabling rich marketplace functionality.

The `gives` and `wants` fields mirror what is already committed to in the proven transaction.
They exist for discoverability (filtering, indexing) and human readability,
but the actual swap terms are enforced by the transaction structure itself.

### Indexer as Discovery Layer

A centralized indexer provides simple, familiar REST semantics for offer discovery.
While this introduces a privacy leak (the indexer operator can observe IP addresses and browsing patterns),
it does not affect security—the cryptographic guarantees remain intact regardless of indexer behavior.
Users requiring stronger privacy can run their own indexer instance or access via Tor.

*Alternatives considered: On-chain offer registry (expensive, reduces privacy), DHT-based discovery (complexity, availability concerns), pure peer-to-peer (poor discoverability).*

## Backwards Compatibility Assessment

This proposal introduces a new standard and does not modify existing Midnight protocols.
There are no backwards compatibility concerns.

Implementations adopting this standard will interoperate with each other but not with non-compliant swap implementations.

## Security Considerations

### Signature Verification

Failure to verify signatures before completing swaps allows attackers to modify offer metadata,
potentially misleading takers about the swap terms.
Both indexers and takers MUST verify signatures.

Note that the core swap terms (what the maker gives and receives)
are enforced by the proven transaction itself.
The signature protects the metadata layer.

### Stale Offers

Offers become invalid when makers spend the backing UTXO elsewhere.
Takers discovering stale offers will fail at submission.
Indexers MAY monitor nullifiers to proactively remove stale offers.

### Privacy Leakage

While on-chain activity is shielded, the indexer is a privacy leak:

| Data | Visibility |
| ------ | ------------ |
| Token types and amounts | Public on indexer |
| Maker receiving keys | Embedded in the proven transaction |
| IP addresses | Visible to indexer operator |
| On-chain transaction details | Shielded |

Users requiring stronger privacy should access indexers via Tor or run their own instance.

### Denial of Service

Malicious actors could flood indexers with invalid offers.
Indexers SHOULD implement:

- Rate limiting
- Proof-of-work requirements
- Reputation systems

## Implementation

### Components

1. **Core Library** (`@midnight/p2p-swaps`)
   - `createOffer()`: Build and sign offer payloads
   - `takeOffer()`: Verify, complete, and submit swaps
   - `cancelOffer()`: Spend backing UTXO to self
   - Serialization and authentication utilities

2. **Reference Indexer**
   - REST API implementation
   - In-memory and persistent storage backends
   - Signature verification
   - Optional nullifier monitoring

3. **Reference UI**
   - Browse and filter offers
   - Create and publish offers
   - Complete swaps
   - Manage active offers

### Dependencies

- `@midnight-ntwrk/zswap`: Transaction primitives
- `@midnight-ntwrk/wallet-api`: Wallet interface
- `@noble/secp256k1`: BIP-340 signatures
- `@noble/hashes`: SHA-256

## Testing

### Unit Tests

- Offer creation with correct payload structure
- Signature creation and verification
- Serialization round-trips
- Error handling for invalid inputs

### Integration Tests

- Full swap flow: create → publish → query → take
- Cancellation flow
- Stale offer handling
- Tampered offer rejection

### Manual Test Scenarios

| Scenario | Expected Outcome |
| ---------- | ------------------ |
| Happy path swap | Both parties receive tokens |
| Tampered offer | Signature verification fails |
| Cancelled offer | Take attempt fails gracefully |
| Network failure during take | Transaction not submitted, tokens safe |

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement,
which includes the assignment of copyright for your contributions to the Foundation.
