---
MIP: "0015"
Title: Wallet-derived deterministic secrets
Authors:
  - Guido De Vita (dvgui)
  - Arturo López (Arturo-Lopez)
Status: Proposed
Category: Standards
Created: 2026-08-18
Requires: none
Replaces: none
MPS: none
License: Apache-2.0
---

<!--
 Copyright Midnight Foundation

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

Midnight DApps routinely need secrets to authenticate with contracts: HTLC preimages, minting and privilege credentials, and other witness values whose commitments live on-chain while the secrets themselves live nowhere durable. Because these secrets are not derived from the wallet, they are not restored by the seed. Today, frontends and services store them ad hoc, so cleared storage, a lost device, or a breached backend means permanently lost access or theft. The wallet cannot, and should not, expose the seed, and the current Connector API offers no deterministic alternative.

This proposal adds a single additive method, `deriveSecret`, to the Midnight DApp Connector `WalletConnectedAPI` (targeting the DApp Connector API v4.0.x line). It lets a DApp ask the wallet to deterministically derive a domain-scoped, context-bound secret from a hardened child of the wallet seed, without exposing the seed or any reusable key material. The derivation scheme is standardized end to end (a SLIP-0021 HMAC-SHA-512 node tree over the wallet's existing BIP-32 seed, followed by HKDF-SHA-256 expansion), so any conforming wallet reproduces the identical secret from the same seed, domain, and context.

```ts
const { secret } = await wallet.deriveSecret({
  domain: 'my-dapp:my-purpose:v1',
  context: instanceId,
});
```

## Motivation

### The problem

Midnight's privacy model puts private state and contract witnesses off-chain. Witness functions run locally and feed secrets into ZK circuits without revealing them on-chain. A common pattern stores only a hash or commitment on-chain and gates a later contract action on reproducing the underlying secret. This pattern is not limited to payments:

- **HTLC preimages.** Atomic swaps (which the connector already supports via `makeIntent` and `balanceSealedTransaction`) require the user to reproduce the preimage to claim or settle. Losing it locks the funds.
- **Minting and privilege credentials.** A contract may gate token minting, or the granting of roles and privileges, on a secret whose commitment it stores on-chain. Unlike funds, that authority is not linked to the wallet in any way: restoring the seed restores keys and balances, but not the contract secret. If the secret is lost, the tokens become permanently unmintable and the privileges permanently unassignable, with no recovery path for anyone.
- **Other contract-gated actions.** Any circuit whose witness must be reproduced later (commit-reveal schemes, access credentials, ownership proofs over private state) has the same shape: the on-chain state is safe, but the ability to act on it lives or dies with an off-chain secret.

In every case the secret must satisfy three properties:

1. It must be deterministic and recoverable from any frontend or device, given only the seed.
2. It must be unguessable to third parties.
3. It must never be hand-copied or persisted as raw key material.

### Why existing Midnight capabilities are inadequate

- **Storing the secret instead of generating it deterministically** (localStorage, IndexedDB, a server database) is strictly more volatile and less secure than derivation. Storage introduces a stateful single point of failure that the user's seed backup does not cover: clearing browser storage, losing the profile or device, or a deleted backend row destroys the secret, and with it the funds, the minting authority, or the privilege, forever. It also widens the attack surface, because every stored copy at rest is a theft target, while a derived secret has no copy at rest at all and can be recreated on demand. This defeats both self-custody and privacy.
- **Hacking determinism out of `signData`.** Developers sign a fixed string and hash the signature. This fails on Midnight by construction: the ledger's `signData` uses randomized Schnorr signing (the signing call draws from `OsRng`), so the same key and data produce a different signature on every invocation, and no secret can be reproduced from it at all. It is also unevenly available (the connector allows wallets to omit methods, and Lace currently does not implement `signData`), and it is unsafe in principle, because it conflates a secret-derivation operation with an authentication primitive, inviting blind signing and signature-reuse phishing.

### The gap and the proposed direction

The wallet cannot, and should not, hand out the seed. What it can safely do is derive a secret internally from a domain-scoped child key and return only the final bytes. The current Connector API provides no such method. `deriveSecret` fills that gap: the same seed, domain, and context yield the same secret on every conforming wallet and every frontend, so contract capabilities become as recoverable as the wallet itself. That recoverability is the entire point.

### Goals and non-goals

**Goals:**

- Deterministic, standardized, cross-wallet-reproducible secret derivation.
- Domain and context separation, so secrets for different purposes and instances never collide.
- Recoverability from any frontend, given only the seed, domain, and context.
- An API that matches the connector spec's existing TypeScript style, permission model, and error taxonomy.
- Hardened derivation, so a leaked derived secret never compromises the seed or sibling domains.

**Non-goals:**

- Exposing the seed, the mnemonic, or any reusable intermediate key.
- A general-purpose signing or encryption oracle.
- Replacing `signData` for authentication. `signData` answers "prove control of a key"; `deriveSecret` answers "reproduce a private witness or secret". They are complementary.
- Nonce handling. `deriveSecret` is deterministic and stateless by design; the wallet does not generate, track, or enforce nonces, freshness, or uniqueness. A DApp that needs per-use uniqueness injects it through the `context` value (for example, a fresh `instanceId`, counter, or random nonce chosen by the DApp), which binds that value into the derivation without requiring the wallet to keep state.
- Defining on-chain contract logic.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this document are to be interpreted as described in RFC 2119.

### API shape

`WalletConnectedAPI` (the type returned by `InitialAPI.connect(networkId)`) is a single object type whose members are methods (`getShieldedBalances`, `getDustBalance`, `balanceUnsealedTransaction`, `makeIntent`, `signData`, `submitTransaction`, `getProvingProvider`, `getConfiguration`, `getConnectionStatus`, and so on). This proposal adds exactly one method to that type:

```ts
export type WalletConnectedAPI = {
    // ... all existing methods, unchanged:
    // getShieldedBalances(), getUnshieldedBalances(), getDustBalance(),
    // getShieldedAddresses(), getUnshieldedAddress(), getDustAddress(),
    // getTxHistory(), balanceUnsealedTransaction(), balanceSealedTransaction(),
    // makeTransfer(), makeIntent(), signData(), submitTransaction(),
    // getProvingProvider(), getConfiguration(), getConnectionStatus()

    /**
     * Deterministically derive a domain-scoped, context-bound secret from a
     * hardened child of the wallet seed. The seed is never exposed. The same
     * (seed, domain, context, length, algorithm, version) MUST always produce
     * the same output on any conforming wallet.
     */
    deriveSecret(params: DeriveSecretParams): Promise<DerivedSecret>;
};

export type DeriveSecretParams = {
    /**
     * Purpose string, namespaced by convention as
     * "<dapp-namespace>:<purpose>:<version>" (e.g. "my-dapp:my-purpose:v1").
     * REQUIRED. UTF-8, NFC-normalized, 1..=256 bytes.
     */
    domain: string;
    /** Per-instance binding value (order id, swap id, nonce). 0..=1024 bytes after decoding. */
    context: string;
    /**
     * Encoding of `context`, mirroring SignDataOptions.encoding:
     * "hex" and "base64" are decoded to binary first; "text" is normalized to UTF-8.
     * Defaults to "text".
     */
    contextEncoding?: 'hex' | 'base64' | 'text';
    /** Output length in bytes. Default 32. Range 16..=64. */
    length?: number;
    /** Derivation scheme version. Default "1". */
    version?: string;
    /** KDF identifier. Default "MN-HKDF-SHA256-v1" (only value in v1). */
    algorithm?: string;
};

export type DerivedSecret = {
    /** Hex-encoded derived secret bytes. */
    secret: string;
    /** Echo of the resolved parameters actually used, for auditability. */
    domain: string;
    length: number;
    version: string;
    algorithm: string;
};
```

Version 1 returns hex-encoded bytes (`secret: string`) to match the connector's existing hex conventions (for example, the `signature` returned by `signData` and transaction hashes in `HistoryEntry`). See the open questions in the Rationale section for a `Uint8Array` or non-extractable `CryptoKey` variant.

### Derivation scheme (normative, version "1")

The scheme reuses the wallet's existing BIP-39 seed `S` (the same seed consumed by `@midnight-ntwrk/wallet-sdk-hd` for curve keys), but derives symmetric secrets through a parallel SLIP-0021 branch plus HKDF-SHA-256. SLIP-0021 is used because BIP-0032 was not designed for symmetric key derivation, and secp256k1 arithmetic is unnecessary here.

**Step 1. Symmetric master node (SLIP-0021):**

```
m = HMAC-SHA512(key = "Symmetric key seed", msg = S)
```

**Step 2. Purpose and label tree (SLIP-0021 child derivation):**

```
ChildNode(N, label) = HMAC-SHA512(key = N[0:32], msg = 0x00 || label)

N1  = ChildNode(m,  "MIP-xxxx-connector-secret")
N2  = ChildNode(N1, UTF8(NFC(domain)))
ikm = N2[32:64]   // 32-byte input keying material
```

A SLIP-0021 node's key is independent of the ability to derive child nodes, so disclosure of one `ikm` (or one output secret) cannot jeopardize sibling domains or the master node.

**Step 3. Context binding and output length via HKDF-SHA-256 (RFC 5869):**

```
salt   = "MIP-xxxx:v1"
info   = UTF8(NFC(domain)) || 0x00 || decode(context, contextEncoding)
secret = HKDF-Expand(HKDF-Extract(salt, ikm), info, length)
```

The `0x00` separator prevents canonicalization ambiguity (so that `("a","bc")` and `("ab","c")` cannot collide). The scheme is fully deterministic (no randomness, no signatures), and every step uses ubiquitous primitives, which is essential for cross-wallet reproducibility. Test vectors (see Testing) fix the exact byte layout. The literal `xxxx` in the SLIP-0021 label and the HKDF salt is replaced with the assigned MIP number before test vectors are published; the strings are then frozen permanently, because changing them would change every derived secret.

### Determinism and cross-wallet compatibility (normative)

A conforming wallet MUST implement version "1" exactly, so that switching wallets never changes the output. This mirrors the guarantees of MetaMask's `snap_getEntropy` (always the same for the same snap, account, and salt) and the WebAuthn PRF extension (reproducible per credential and salt).

Derivation MUST come from the account-independent seed root by default, so recovery needs only the seed, domain, and context. Output MUST NOT depend on network id, locale, wallet vendor, or time. This network independence matters because network ids other than `'mainnet'` are wallet-defined and vary between implementations.

### Permission and consent model

`deriveSecret` follows the connector's existing rule that presence in the type does not imply support or permission: wallets may omit methods, and DApps must feature-detect. On refusal, the wallet throws `PermissionRejected` (a session-level preference to deny the DApp, persistent until the user changes wallet settings) or `Rejected` (the user saw the specific request and declined this time).

Wallets MUST prompt for consent per `domain` on first use, displaying the sanitized `domain`, the requesting origin, and a plain-language note that a reproducible secret will be derived. Wallets MAY remember per-domain approvals. DApps SHOULD include `'deriveSecret'` in their `hintUsage(methodNames)` call, so the wallet can proactively request user permission for better UX (once the method is part of `WalletConnectedAPI`, `'deriveSecret'` becomes a valid `keyof WalletConnectedAPI` entry).

### Error conditions

All errors reuse the existing `ErrorCodes` enumeration, surfaced as `APIError` objects (`type: 'DAppConnectorAPIError'`, `code`, `reason`, detected via `error.type === 'DAppConnectorAPIError'` rather than `instanceof`):

- `InvalidRequest`: missing or oversized `domain` or `context`, `length` out of range, unknown `algorithm` or `version`, malformed encoding.
- `Rejected`: the user saw this request and declined it (per-request; the DApp may try again with a different request, but must not auto-retry).
- `PermissionRejected`: the user has set a session-level preference to deny this DApp (persistent; retrying yields the same result until wallet settings change).
- `InternalError`: derivation failure inside the wallet.
- `Disconnected`: the connection to the wallet was lost mid-session.

### Versioning

The Specification is versioned through the `version` parameter of `DeriveSecretParams`. This document defines version "1" (with the single KDF identifier "MN-HKDF-SHA256-v1"), whose byte-exact behavior is frozen by the published test vectors: once this MIP is accepted, the output for any `(seed, domain, context, length)` under version "1" never changes. Future revisions (for example, a post-quantum variant or additional output forms) define new `version` or `algorithm` values through a subsequent MIP that extends or supersedes this one. Wallets MUST reject unknown `version` or `algorithm` values with `InvalidRequest` rather than falling back to a guess, so a DApp can never silently receive bytes from a scheme it did not request.

### Example: HTLC preimage flow

```ts
const wallet = await initialApi.connect(networkId);

const swapId = '...'; // stable, shared, non-secret identifier for this swap

const { secret: preimageHex } = await wallet.deriveSecret({
  domain: 'my-dapp:htlc-preimage:v1',
  context: swapId,
  length: 32,
});
const preimage = hexToBytes(preimageHex);
const hashlock = sha256(preimage); // published on-chain in the HTLC

// Later, from a DIFFERENT frontend or device, same seed:
const { secret: sameHex } = await wallet.deriveSecret({
  domain: 'my-dapp:htlc-preimage:v1',
  context: swapId,
});
// sameHex === preimageHex, so the user can reclaim or settle with no stored state.
```

The `preimage` feeds the contract's witness (for example, a `Bytes<32>` returned by a witness function) to satisfy the circuit, exactly as Midnight private-state witnesses work today. All `domain` and `context` values in this document (`my-dapp:my-purpose:v1`, `my-dapp:htlc-preimage:v1`, `swapId`, `instanceId`) are placeholders illustrating the recommended `<dapp-namespace>:<purpose>:<version>` convention, not registered namespaces.

## Rationale

The design goal is a derivation that is reproducible from the seed alone, portable across wallet implementations, and safe to expose to arbitrary web pages. SLIP-0021 provides the hardened, label-based symmetric tree (avoiding curve arithmetic that symmetric derivation does not need), and HKDF-SHA-256 provides standardized context binding and variable-length output. Both are widely implemented, which keeps the cross-wallet conformance burden low. The composition (label, salt, and info layout) is this proposal's design choice built from those established standards; it should be reviewed by wallet implementers and a cryptographer, then pinned by the published test vectors.

The following alternatives were considered and rejected:

**`signData`-derived secrets.** Midnight's ledger-level signing is randomized Schnorr (fresh randomness per signature), so signature-derived secrets are irreproducible even within a single wallet. The connector also permits wallets to omit `signData` entirely (Lace currently does). Finally, `signData` is semantically an authentication primitive, and conflating the two invites blind signing and signature-reuse phishing. Even a hypothetical switch to deterministic signing (RFC 6979, BIP-340 style) would not make it a specified, portable KDF.

**DApp-side storage.** The status quo. Loss of state means loss of funds, a leak means theft, and it fails the cross-frontend recovery requirement entirely.

**Pure origin-scoped entropy (the `snap_getEntropy` model).** Snap-ID scoping gives strong isolation, but it breaks Midnight's core recoverability requirement. The ChainSafe Canton Snap documents this failure mode explicitly: the same seed produces a different identity under different snap IDs, with no migration path. Version 1 therefore keys on a global, DApp-declared `domain` string instead of the page origin.

**WebAuthn PRF (`hmac-secret`).** Excellent hardware-anchored prior art (a 32-byte secret per credential and salt, fed into HKDF), but it is credential-bound (lose the authenticator, lose the secret) and browser support remains uneven. Seed anchoring gives true mnemonic recoverability.

**Do nothing.** Leaves every privacy DApp to reinvent an unsafe KDF, fragmenting determinism across the ecosystem and putting user funds at risk.

### Open questions

1. **Domain registration and namespacing.** Should well-known domains be registered (with expected origins), or remain free-form with client-side heuristics?
2. **Origin-binding policy.** Should wallets offer an opt-in, origin-scoped mode per domain (the snap model) for DApps that prefer isolation over recoverability?
3. **Account scoping.** Derive from the seed root (maximum recoverability, recommended) or from a selected account?
4. **Maximum sizes.** Confirm the proposed 256-byte `domain` and 1024-byte `context` limits against wallet UI and transport constraints.
5. **Output form.** Add a `Uint8Array` and/or non-extractable `CryptoKey` variant to discourage holding secrets in JS strings?

## Path to Active

For a Standards MIP, Active means the method is part of the published DApp Connector API and available to users in deployed wallets. Because the change is confined to the off-chain connector interface and wallet software, no network upgrade is required; the path runs through the connector package release, wallet adoption, and cross-wallet conformance.

### Acceptance Criteria

- The `deriveSecret` method and its types are merged into `@midnight-ntwrk/dapp-connector-api` and released with a minor version bump of the v4 line.
- Canonical test vectors for derivation version "1" are published in this repository or a location linked from it.
- At least two independent wallet implementations (for example, Lace and 1AM) ship the method and reproduce all test vectors byte for byte in continuous integration.
- Developer documentation on docs.midnight.network covers the method, including a witness-recovery guide and error-reference entries.

### Implementation Plan

1. A connector package pull request adds the method and types, kept in lockstep with the specification document. Implementation code is linked from the pull request, not embedded in it.
2. A reference implementation lands in one wallet, together with a `deriveSymmetricSecret(domain, context, length)` helper in `@midnight-ntwrk/wallet-sdk-hd` (reusing the SDK's `clear()` to zeroize intermediate nodes) and a stub in `testkit-js`'s `DAppConnectorWalletAdapter` to keep test tooling in sync.
3. Test vectors and the conformance suite are published (see Testing).
4. A second, independent wallet implementation validates the vectors.
5. Documentation is rolled out on docs.midnight.network.

## Backwards Compatibility Assessment

The change is purely additive: one new method on `WalletConnectedAPI` and two new exported types (`DeriveSecretParams` and `DerivedSecret`, plus the `'deriveSecret'` key becoming valid in `hintUsage`), with no breaking changes. No hard fork is required: the ledger, node, consensus, and Compact language are untouched, since the change is confined to the off-chain DApp Connector interface and wallet software.

The connector already establishes that not every wallet implements every method, and that DApps must feature-detect (for example, `typeof wallet.deriveSecret === 'function'`) and degrade gracefully, so this fits the existing integration pattern. The `apiVersion` (semver of `@midnight-ntwrk/dapp-connector-api`, currently in the v4.0.x line) should be bumped a minor version when this ships. Wallets supporting multiple versions during a transition continue registering multiple `window.midnight` entries, as the spec already allows.

## Security Considerations

**Domain separation (three layers).** The SLIP-0021 label `"MIP-xxxx-connector-secret"` separates this scheme from all other uses of the seed. The `domain` node separates DApps and purposes. The HKDF `info` binds the `context`.

**Hardened, one-way derivation.** SLIP-0021 HMAC-SHA-512 nodes mean that knowledge of a derived `ikm` or output secret reveals neither the master node, nor sibling domains, nor the seed.

**No seed exposure.** All HMAC and HKDF steps run inside the wallet. Only the final `length`-byte secret crosses the connector boundary. Intermediate nodes never leave the wallet and should be zeroized after use.

**Context reuse.** Determinism cuts both ways: the same `(domain, context)` pair always yields the same secret. DApps MUST NOT reuse a `context` across instances that require distinct secrets (for example, two swaps sharing one identifier would share one preimage), and SHOULD construct a fresh, unique `context` per instance.

**Phishing and cross-DApp domain requests.** Because derivation is global-domain (not origin-bound), a malicious site could request another DApp's domain. Mitigations: mandatory per-domain consent showing the requesting origin; an optional wallet-maintained registry of well-known domain-to-origin mappings with mismatch warnings; collision-resistant, versioned domain namespaces; and encouraging high-entropy `context` values. This is an explicit, documented trade-off of recoverability against isolation.

**Cross-wallet determinism as a safety property.** If two wallets disagree on the output bytes, users lose funds. The scheme is therefore fully specified with mandatory published test vectors, and any deviation is a conformance bug, not an implementation choice.

**DApp secret handling.** Derived secrets should be treated as ephemeral: memory-only, never persisted, zeroized after use, and re-derived on demand. They are as sensitive as private keys. DApps should prefer `length: 32` unless a longer output is genuinely required.

**Rate limiting.** Wallets MAY rate-limit or batch-prompt to prevent a page from farming secrets across many domains.

## Implementation

The change is confined to off-chain components:

- **`@midnight-ntwrk/dapp-connector-api`:** adds the `deriveSecret` method and the `DeriveSecretParams` and `DerivedSecret` types; `'deriveSecret'` becomes a valid `hintUsage` key.
- **Wallet implementations (Lace, 1AM, and others):** implement the derivation scheme and the per-domain consent flow.
- **`@midnight-ntwrk/wallet-sdk-hd`:** an optional shared `deriveSymmetricSecret(domain, context, length)` helper, so wallets do not each re-implement the KDF, reusing the SDK's `clear()` for zeroization.
- **`testkit-js` (`DAppConnectorWalletAdapter`):** a stub implementation keeps DApp test tooling in sync.
- **docs.midnight.network:** API reference entries, an error-reference note, and a "Deterministic secrets and witness recovery" guide.

Dependencies are limited to HMAC-SHA-512 and HKDF-SHA-256 primitives, which are ubiquitous in every relevant environment. No ledger, node, consensus, or Compact changes are required.

## Testing

1. **Canonical test vectors.** Publish `(seed, domain, context, length) -> secret` vectors, including the SLIP-0021 published seed and empty or edge-case contexts, so every wallet can prove byte-for-byte conformance in continuous integration. This is the single most important deliverable for preventing fund loss, since cross-wallet determinism is a safety property of the scheme.
2. **Conformance suite.** A test harness runnable against any connector implementation, asserting exact output bytes for every vector, boundary validation (`domain` and `context` size limits, `length` range), encoding handling for `hex`, `base64`, and `text` contexts, rejection of unknown `version` and `algorithm` values, and correct `APIError` codes for invalid requests and refusals.
3. **Independent reproduction.** At least two independent wallet implementations reproduce all vectors before this MIP is proposed for acceptance.

## References

- Midnight DApp Connector API specification: https://github.com/midnightntwrk/midnight-docs/blob/main/api-reference/dapp-connector/_media/SPECIFICATION.md
- DApp Connector API reference (`WalletConnectedAPI`, `SignDataOptions`, `APIError`, `ErrorCodes`, `HintUsage`): https://docs.midnight.network/api-reference/dapp-connector
- DApp Connector error reference: https://docs.midnight.network/api-reference/error-reference
- `@midnight-ntwrk/wallet-sdk-hd` (HD derivation, BIP-32/BIP-44/CIP-1852 mix): https://www.npmjs.com/package/@midnight-ntwrk/wallet-sdk-hd
- SLIP-0021, Hierarchical derivation of symmetric keys: https://github.com/satoshilabs/slips/blob/master/slip-0021.md
- RFC 5869, HKDF: https://www.rfc-editor.org/rfc/rfc5869
- RFC 2119, Key words for use in RFCs: https://www.rfc-editor.org/rfc/rfc2119
- MetaMask SIP-6, deterministic snap-specific entropy: https://metamask.github.io/SIPs/SIPS/sip-6
- ChainSafe Canton Snap (origin-scoping failure mode): https://github.com/ChainSafe/canton-snap
- WebAuthn PRF extension: https://developers.yubico.com/WebAuthn/Concepts/PRF_Extension/

## Acknowledgements

None yet. Contributors from the review and discussion phase who are not authors will be listed here.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
