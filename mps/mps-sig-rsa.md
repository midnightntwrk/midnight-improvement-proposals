# Signature Verification - **RSA 1024/2048**

# Abstract

This proposal implements optimized RSA signature verification functions for both 1024-bit and 2048-bit key sizes within the Compact smart contract language. This addresses the need for legacy system compatibility, enterprise integration, and specific compliance use cases, and has hard dependencies on the completion of the ZKIR V3 update.

## Motivation

The inability to verify RSA signatures in Compact is blocking advanced enterprise applications that rely on established asymmetric cryptography standards. 

- **Legacy System Compatibility:** Many enterprise and compliance systems rely on RSA as an established asymmetric cryptography standard. Without in-circuit RSA verification, these integrations cannot be built on Midnight.
- **Interoperability:** Developers need a tool to link off-chain or external chain state to on-chain actions in a verifiable, trustless manner.
- **Partner Use Cases:** Applications requiring trusted, in-circuit signature verification are directly blocked by the absence of this primitive.

## Specification

The following function will be exposed within a dedicated Compact library:

```
verify_rsa(n, e, padded_message, signature) → Boolean
```

where `n` is the modulus, `e` is the public exponent, `padded_message` is the padded hash of the message, and `signature` is the RSA signature. Padding is the responsibility of the caller and is out of scope for this primitive.
> Note: RSA circuit size scales with the bit length of the modulus and public exponent. Large moduli or exponents will result in significantly larger circuits and higher proving times, which may make them non-viable for real-time applications.   
# Implementation

| User Story | Acceptance Criteria | Failure Scenarios |
| --- | --- | --- |
| **Compact API Exposure** | The Compact language exposes `verify_rsa(n, e, padded_message, signature)` within a dedicated library. | Proving the verification for RSA-2048 takes an unreasonable amount of time, making it non-viable for real-time applications. |
| **Correct Verification** | The implemented functions correctly return `True` for valid RSA signatures for both 1024-bit and 2048-bit keys against a set of MNF-provided test vectors. | Implementation fails verification against known, standards-compliant test vectors. |

---


### Supporting Issues

- [Compact 105](https://github.com/LFDT-Minokawa/compact/issues/105)
