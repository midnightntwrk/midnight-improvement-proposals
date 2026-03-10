# Signature Verification - **RSA 1024/2048**

# Abstract

This proposal implements optimized RSA signature verification functions for both 1024-bit and 2048-bit key sizes within the Compact smart contract language. This addresses the need for legacy system compatibility, enterprise integration, and specific compliance use cases, and has hard dependencies on the completion of the ZKIR V3 update.

## Motivation

The inability to verify RSA signatures in Compact is blocking advanced enterprise applications that rely on established asymmetric cryptography standards. 

- **Legacy System Compatibility:** Many enterprise and compliance systems rely on RSA as an established asymmetric cryptography standard. Without in-circuit RSA verification, these integrations cannot be built on Midnight.
- **Interoperability:** Developers need a tool to link off-chain or external chain state to on-chain actions in a verifiable, trustless manner.
- **Partner Use Cases:** Applications requiring trusted, in-circuit signature verification are directly blocked by the absence of this primitive.

## Specification

The following functions will be exposed within a dedicated Compact library:

`verify_rsa_1024(pubkey, message, signature) → Boolean
verify_rsa_2048(pubkey, message, signature) → Boolean`

**Key sizes:**

- RSA-1024 (for compatibility)
- RSA-2048 (recommended standard)

**Padding:** The highest security padding standard compatible with the specified key sizes will be used.

# Implementation

| User Story | Acceptance Criteria | Failure Scenarios |
| --- | --- | --- |
| **Compact API Exposure** | The Compact language exposes `verify_rsa_1024(pubkey, message, signature)` and `verify_rsa_2048(pubkey, message, signature)` within a dedicated library. | Proving the verification for RSA-2048 takes an unreasonable amount of time, making it non-viable for real-time applications. |
| **Correct Verification** | The implemented functions correctly return `True` for valid RSA signatures for both 1024-bit and 2048-bit keys against a set of MNF-provided test vectors. | Implementation fails verification against known, standards-compliant test vectors. |

---
