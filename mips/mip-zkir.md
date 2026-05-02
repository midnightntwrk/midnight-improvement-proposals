# ZKIR

## Summary

**ZKIR** (Zero-Knowledge Intermediate Representation) is the format used by the Compact compiler to express smart contracts. It serves as the bridge between the high-level Compact programming language and the low-level cryptographic proofs executed on the Midnight blockchain.

- **Role:** ZKIR is used to translate high-level Compact logic into the specific constraints required to generate SNARKs (proofs) and verifier keys on the ledger.
- **Format:** It exists in both JSON and binary representations.
- **Execution:** The ledger uses ZKIR to perform key generation and proof generation. In the context of v3, the binary representation of a circuit is intended to be interpreted as a program capable of running witnesses.
- **Evolution:** Previously (in v2), ZKIR was a "unitype" language where every value was a field element. ZKIR v3 transforms it into a **typed language**.

## Motivation

The primary goal of ZKIR is to evolve the format from an experimental feature into a robust standard that supports advanced cryptographic primitives.

- **Advanced Curve Support:** Enable the support native curve points (specifically `JubjubPoint` and `secp256k1`) to enable critical features like ECDSA signatures which are required by partners.
- **"Latent" Mainnet Features:** Deploy capabilities into the mainnet ledger (Ledger 7.0) that are present but "locked." This allows features to be exposed later via compiler/toolchain updates without requiring a risky blockchain hard fork[cite: 369, 377].
- **Incremental Delivery:** Move away from a "big release" to a "demand-driven" approach, adding features like specific curve types only when needed to keep PRs small and manageable.

### **Functional Requirements**

**Type System Implementation**

To support complex cryptographic operations, ZKIR must strictly enforce types.

- **Native Types:** Introduce specific types for cryptographic elements, starting with **`JubjubPoint`** (a native curve point).
- **Foreign Curves:** Implement support for foreign curve types. Example: **`secp256k1`**.
- **Type Signatures:** Provide explicit type signatures for circuits, including the types of their witnesses, to support the interpretation of ZKIR.
- **BigUint Support:** Create entry points and support for `BigUint` interaction to unlock handling of large integers.

**Arithmetic & Operations**

- **Extended Arithmetic:** Update the signatures of the standard library’s `ecAdd`, `ecMul`, and `ecMulGenerator` circuits to support the new typed inputs.
- **Flexible Crypto Operations:** The format must allow the Crypto team to easily change or extend the available operations (e.g., changing the hash function or signature verification logic).

**Control Flow & Execution**

- **SSA Control Flow:** Implement explicit Static Single Assignment (SSA) control flow to improve analysis and execution safety.
- **Witness Calls:** Support explicit witness calls within the format to facilitate the generation of proofs for cross-contract calls.
- **Code Simplification:** Remove legacy features such as "communications commitment" and "Pi skip" to simplify the execution model and reduce technical debt.

### **Success Criteria**

Upon completion, Compact team will be able to expose the crypto features on a new Compact version, that will alow devs to have access to the newly added:

- **Signature primitives:**
    - secp256k1
    - ECDSA
    - Keccak-256
- **Identity primitives:**
    - RSA Signature Verification Primitive both RSA-1024 and RSA-2048