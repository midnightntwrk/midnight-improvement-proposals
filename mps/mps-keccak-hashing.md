# Signature Verification - **Native Keccak Hashing**

## Abstract

This proposal implements and exposes the Keccak-256 hashing algorithm as an accessible function within a dedicated Compact cryptographic library. 

## Motivation

Current cryptographic primitives available in Compact are not optimally suited for all use cases:

- **Ethereum Compatibility:** The Ethereum ecosystem uses Keccak-256. Relying solely on different hash functions (like Poseidon/Griffin) or traditional ones (like SHA-256) complicates external verification and requires developers to implement inefficient conversion or custom logic.
- **Protocol Versatility:** Exposing Keccak-256 offers a more diverse set of fundamental cryptographic tools, moving beyond reliance on traditional primitives like SHA-256.

## Specification

The following function will be exposed within a dedicated Compact cryptographic library:

`keccak256(data: Bytes<n>) → Bytes<32>`

**Behaviour:**

- Accepts arbitrary-length byte arrays as input.
- Returns a fixed-length `Bytes<32>` output.
- Output must be identical to standard Keccak-256 implementations across a range of test vectors.

## Implementation

| User Story | Acceptance Criteria | Failure Scenarios |
| --- | --- | --- |
| **Compact API Exposure** | The function `keccak256(data: Bytes<n>)` is exposed, accepting arbitrary-length byte arrays and returning a fixed-length `Bytes<32>` output. | Implementation results in a hash with a constraint count exceeding an acceptable threshold, making it economically non-viable for dApps. |
| **Correct Hashing** | The function output is identical to standard Keccak-256 implementations across a range of test vectors. | Mismatch between the off-circuit implementation and the in-circuit one. |

**Authoritative test vectors, the two best sources are:**

1. **Original Keccak KAT** — the `ShortMsgKAT_256.txt` and `ExtremelyLongMsgKAT_256.txt` files from `KeccakKAT-3.zip`, available from keccak.noekeon.org. [Uc](http://gauss.ececs.uc.edu/Courses/c653/lectures/Hashing/sha3/sha3test.c) These are the pre-NIST "pure Keccak" vectors that match Ethereum's implementation.
2. **Ethereum's own test vectors** — the `js-ethereum-cryptography` library maintained by the Ethereum Foundation has a dedicated Keccak-256 test vector file at `ethereum/js-ethereum-cryptography` on GitHub, which is the most directly relevant source for Ethereum compatibility testing.