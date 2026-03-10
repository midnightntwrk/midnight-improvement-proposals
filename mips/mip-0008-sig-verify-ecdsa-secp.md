# Signature Verification - **ECDSA/secp256k1**

# Summary

This proposal defines the addition of a native secp256k1 ECDSA signature verification function to the Compact smart contract language, exposed via a dedicated `CryptoLibrary`. The function enables developers to verify standard external digital signatures inside a Compact contract in a performant and auditable manner, supporting interoperability with Ethereum, Bitcoin, Avalanche, and other Layer 1 networks.

## Motivation

Developers are currently blocked from building dApps that rely on verifying standard external digital signatures (specifically ECDSA on secp256k1) inside a Compact contract. Without this primitive, partner use cases requiring trusted, in-circuit signature verification cannot be unblocked, and ecosystem adoption is hindered.

## Specification

A high-level verification function will be exposed within a dedicated `CryptoLibrary`, distinct from the `CompactStandardLibrary`:

`verify_ecdsa_secp256k1(pubkey, message, signature) → Boolean`

**Behaviour:**

- Accepts a public key, a message, and a signature as inputs.
- Internally hashes the provided message prior to verification.
- Returns `True` if the signature is valid for the given public key and message.
- Returns `False` for any invalid signature.

## **Acceptance criteria**

**Core secp256k1 Verification Functionality:**

| User Story | Acceptance Criteria | Failure Scenarios |
| --- | --- | --- |
| **Compact API Exposure** | The Compact language exposes a high-level function (e.g., `verify_ecdsa_secp256k1(pubkey, message, signature)`) accessible via a dedicated library. The message is hashed internally. | The function is excessively verbose or requires developers to directly manage low-level cryptographic primitives (e.g., manually calling curve addition/multiplication). |
| **Correct Verification** | The function correctly returns `True` for valid secp256k1 signatures and `False` for invalid ones, validated against Foundation-provided test vectors. | Implementation fails verification against industry-standard test vectors (e.g., mis-handling message hashing or signature padding). |
| **Dependency Management** | The implementation uses Compact primitives for elliptic curve and hashing operations compatible with the latest ZKIR V3 standard. | Implementation is blocked or delivers poor performance due to unresolved dependencies on the ZKIR V3 rollout. |

# **Dependencies:**

- **ZkirV3** must be implemented and available. The elliptic curve and hashing operations used by this library are built on Compact primitives that require ZKIR V3 compatibility.