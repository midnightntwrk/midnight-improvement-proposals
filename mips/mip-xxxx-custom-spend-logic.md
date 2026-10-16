---
MIP: X
Title: Custom spend logic
Authors:
  - Andrzej Kopeć (kapke)
  - {Note: alphabetize authors by last name}
Status: Draft
Category: {see MIP Categories in MIP-1}
Created: {creation date}
Requires: Capsule Runtime MIP ("Encapsulated contract execution and local state")
Replaces: {MIPs that this one will supersede when accepted}
MPS: {related MPS reference, can be added later, or "none"}
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

Implementing a compliant, feature-rich shielded token falls short on several fronts. The biggest gap seems to lie in a contract's inability to govern token movement once it reaches a user's wallet. That gap is stated directly in [MPS-0013](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0013-zswap-business-logic.md), and can be felt in [MPS-0025](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0025-shielded-source-of-funds.md) and [MPS-0024](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0024-custodian-safe-shielded-spends.md).

This document proposes an extension to the shielded token protocol, dubbed "custom spend logic" — that is, the protocol's ability to enforce that a circuit designated at a token's mint time is executed whenever the token is spent — enabling that circuit to act as an authorisation gate, a compliance hook, or simply enforceable transfer prevention for the "only this person" kind of token.

## Motivation

Many practical use-cases require that token movement is guarded by more logic than "is my wallet the owner of this token?" Compliance is an obvious one. Plenty of platforms have a "stop" switch to prevent token usage in case of a serious bug. Another real use-case is authentication — tokens are very convenient for it, but the possibility of transferring a token to another wallet address is sometimes unacceptable.

Compliance is an especially complex and wide topic, where the tension between extending native protocol capabilities and the ability to obey many different rules at the same time is most visible. Examples fitting under this umbrella include:

- a regulated entity such as a custodian needs a proof of the tokens' compliant origin, or a way to verify it;
- a regulated entity needs the ability to traverse the transaction graph for anomaly detection;
- there are lists of addresses that are not allowed to spend or receive tokens;
- the sender's identity needs to be known to the receiver to obey the travel rule.

But for non-monetary use-cases, compliance can take entirely different forms. Proofs of being a human, an adult or a resident are all valid use-cases.

Observing this, one could argue that the protocol should have primitives allowing each of the requirements to be enforced separately, so that a user or developer can choose the best subset for their use-case. This MIP rejects that direction, because of the churn it would introduce in a particularly security- and privacy-sensitive part of Midnight.

Instead, it proposes a single extension to the protocol, which aims to enable developers to cover as many use-cases as possible.

## Specification

A *spend rule* is a predicate that must hold for a shielded coin to be spent. The token's issuer chooses it once, at mint time, and it is thereafter unchangeable for that token. The rule is not carried by the coin, and it is not held in a registry that the ledger consults. It is bound into the token type itself, so that a coin of a given type cannot exist unless the rule that type names was satisfied by every spend that led to it.

Three changes make that work:

1. the token type comes to bind a *spend rule descriptor* as one operand of the type's preimage;
2. the Zswap `spend` circuit — which authorises a shielded spend, as distinct from the `Input` element of an offer whose validity it establishes — re-derives the token type from the descriptor and enforces the rule it names;
3. the descriptor names one of three authorisation modes, ranging from exactly today's behaviour to a contract call bound to the spend.

Everything else about Zswap — commitments, nullifiers, the commitment tree, value commitments, offer balancing, merging — is left alone.

### Rules are per token type

Today a user-defined token type is derived from a domain separator the minting contract chooses per token it issues, together with the contract's own address, under a fixed inner separator:

```
RawTokenType = SHA256("midnight:derive_token" padded to 32 bytes ‖ domain_sep ‖ contract_address)
```

The operand order is as written; `spec/intents-transactions.md` transposes the two and omits the inner separator, and the implementation is authoritative. This proposal binds a spend rule descriptor into that preimage, as a third operand:

```rust
struct SpendRule {
    mode: SpendRuleMode,
    // mode-specific payload: nothing, a gate circuit's verifying key, or a gate contract's address and entry point
}
```

```
RawTokenType = SHA256("midnight:derive_token" padded to 32 bytes ‖ domain_sep ‖ contract_address ‖ descriptor)
```

Mode 1 encodes as the empty descriptor, so its preimage is exactly today's ninety-six bytes.

The same preimage derives the unshielded token type, which differs only in the wrapping newtype; but unshielded spends are authorised by signature at the ledger rather than by a circuit, so circuit re-derivation guards shielded coins alone.

The rule is therefore a property of the token type, not of the individual coin. Every coin of a type is subject to the same rule, and two coins subject to different rules are coins of different types, not fungible with each other. An issuer that needs different treatment for different tranches of an asset mints them as different types — which the family-style token standards ([MIP-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md)) already accommodate.

Uniformity per type keeps the ledger's job tractable: it never has to determine which rule applies to which coin, because it never has to distinguish coins of the same type.

### Binding without lookup

A coin's type is private. It appears in a Zswap `Input` only inside the proof, as a preimage to the value commitment; the ledger sees a nullifier, a Merkle root, a Pedersen commitment and a proof, and learns nothing about what was spent. That rules out any implementation in which the ledger looks the rule up by type and selects a verifier: the lookup would require the type to be public, which is what shielding is for.

Binding is achieved by re-derivation instead. The `spend` circuit takes the descriptor as a private witness, re-derives the token type from it together with the minting contract's separator and address — both of them new witnesses, since the circuit receives neither today — and asserts that the result equals the coin's type. The assertion pins the three operands jointly. The circuit then enforces the mode the descriptor names. The derivation change adds nothing to ledger state, needs no per-type registry, and adds no public field to the `Input`. In circuit the assertion costs roughly two SHA-256 compressions, against three persistent hashes and a hash-to-curve already there.

The coin's colour is what makes this sound. It is private — reaching only the coin commitment, the nullifier and the value-commitment base, and never disclosed — and it is nonetheless pinned, by the commitment the circuit opens and the Merkle path it asserts against the tree root. Re-derivation is asserted unconditionally on every shielded spend, with no mode-gated path around it, so a guarded colour cannot be reached through the empty-descriptor preimage, which would take a second preimage for SHA-256 over preimages the spender witnesses freely: the length in the padding makes the strictly-longer encoding below bite. The prover chooses the descriptor it witnesses; the pinned colour decides which one it may choose.

The construction must also satisfy four further conditions. Mode 1 encodes as the empty descriptor, with no tag and no padding: byte-identity carries existing types through the unconditional assertion, and keeps the rule prospective, since blocks already accepted stay valid under the crate and key that accepted them. Every guarded encoding is strictly longer and canonically framed — fixed-width fields, or a length prefix — because the hash preimage is unframed concatenation and two descriptors must not serialise alike. The `proof-verifying` feature stays default-on, so no shipped build accepts a spend without checking its proof. And the change lands at a spec-version fork boundary, with the spend circuit's `KeyLocation` versioned, so that a stale proving key fails loudly.

The circuit change enforces itself. Altering `spend` alters its verifying key, and that key is compiled into the ledger crate: no key registry, no version negotiation, no per-transaction selection, no grace period. The node's support for two ledger versions classifies historical blocks by the version that accepted them rather than opening a concurrent window, so from the fork no live route accepts an old-circuit proof.

The consequence divides the modes. A descriptor that commits to the rule's own logic — a verifying key, as in mode 2 — cannot be swapped for a weaker one: a different descriptor hashes to a different token type, so substituting the rule substitutes the asset, and the substituted asset is one nobody minted, holds or accepts. A descriptor naming a contract address pins only that a named contract was consulted at every spend. Immutability by construction and revision behind a governed pointer exclude each other; mode 3 buys the second at the price of the first.

**Open.** The unconditional assertion makes resolving `(separator, minting address)` a precondition for spending any user-defined shielded coin, not a convenience for guarded ones. Both values are published at mint time, as *Discovery* below sets out, and all shielded supply is contract-minted — an offer's deltas must net against a mint, and the mint fixes the separator — so every colour carrying value has a published separator behind it. A zero-value output is Pedersen-neutral and needs neither a mint nor a delta, so it may carry an arbitrary colour with none; such a coin is worth nothing and is unspendable. What remains open is resolvability: retention and backfill of the colour to `(separator, address)` mapping, and whether to mandate the separator in coin metadata so a coin carries its own derivation inputs.

### The three authorisation modes

**Mode 1 — no rule.** The descriptor is empty and authorisation is exactly today's: knowledge of the coin secret key, or, for a contract-held coin, being the named contract, with no further condition. Every token type that exists today is mode 1 and re-derives unchanged, which is what makes the change additive rather than a migration.

**Mode 2 — recursive verification of a pure circuit.** The descriptor names the verifying key of a *gate circuit*. The `spend` proof recursively verifies a proof against that key. Which public inputs the `spend` circuit fixes is left open; whatever they are, they must bind the gate proof to this spend so it cannot be replayed against another. A spend reveals nothing of the coin, its type, or which rule held; the mode costs no privacy. That holds because the gate-verification branch is as unconditional as the re-derivation: one compiled spend key means one relation for every shielded spend, mode 1 discharges the branch trivially, and recursion's cost in circuit is therefore borne by every shielded spend rather than by the guarded ones. Its other cost is reach: a pure circuit decides only from what it is handed, so it can enforce a statement about the spender or a signed credential, but it cannot consult chain state such as a freeze list or a supply commitment, a limit a later subsection returns to. It has no channel to publish either: the `Input` gains no public field. This mode depends on [MPS-0014](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0014-proof-verification-recursion.md), and on more than it currently scopes: what mode 2 needs is recursive verification against a verifying key the prover witnesses rather than supplies as a public input, its value being published in the mint effect.

**Mode 3 — a contract call bound to the spend.** The descriptor names a contract address and an entry point, and the spend is well-formed only if the same transaction carries a call to that entry point, bound to it. In exchange the rule gains the full expressiveness of a contract: it can read and write contract state, apply threshold authorisation, consult a list, emit an auditable public event, and be upgraded behind a governed pointer without the token type changing. That last capability is the other face of the qualification above: a holder relies on the gate's governance as well as on its code.

*Bound* is the word doing the work. The call must be bound to the spend it gates, and the binding has to originate in the Zswap `spend` circuit to mean anything: a gate needs evidence of who spent, proved by the party that spent, and only the spend proof can produce it. A binding asserted anywhere else establishes at most that a spend and a call travelled together, and the identity a gate's own context offers is no help — `ownPublicKey()` is a witness declaration, free-form at proving time and trivially substituted.

The shape follows contract-to-contract calls, where this is already solved. A caller declares its callee as an effect — sequence number, address, entry point hash, commitment — and the ledger matches that tuple to exactly one real call, deriving the callee's attested `caller` from the match; the binding itself is the `communication_commitment`, taken over the call's arguments *and* its return value under randomness shared between the two parties, to which both circuits prove an opening. Transposed to a spend, the `Input` gains a proved field committing to `(nullifier, coin, sender_pk, gate_address, ep_hash)`, opened inside the `spend` circuit from witnesses it already holds: the nullifier derives from the spending secret key, so the proof witnesses that key of necessity, and the circuit can prove that the `ZswapCoinPublicKey` it asserts is the one whose secret key derives this nullifier. The gate call's commitment must open to the same tuple, and a sibling effect field, keyed and checked as claimed contract calls are, requires each claimed authorisation to correspond uniquely to a real spend in the same segment. The converse begins in the circuit: only the spend proof witnesses the descriptor, so only it can require that a mode-3 spend carry a binding field. The field itself is public, so the ledger completes it, matching in both directions — every input carrying one to a claimed authorisation and every claimed authorisation to an input — keyed by segment, nullifier and gate address, since the commitment hides `gate_address` and a value-only match would let any contract emit the claim. The `nullifiers == claimed_nullifiers` template is keyed that way and reaches only inputs that carry a contract address. The gate then reads an attested caller rather than a forgeable witness, and unforgeability comes from where it does between contracts today — except that one of the two proofs opening the commitment is now the spend proof, which cannot be produced without the coin secret key — or, for a contract-held coin, without a call by the contract that holds it, which the ledger already requires. Little new cryptography is called for: `contract_address` is already a public input to the spend proof, so a binding field can be made public the same way, and the caller-to-callee proof binding, the uniqueness enforcement, the guaranteed and fallible nesting, the call ordering and the recomputation of effects at apply time are all reusable as they stand.

The price is that the contract call is visible in the transaction, though its arguments are not, reaching the chain only inside the binding commitment. An observer learns that a particular public nullifier was authorised by a particular contract, and therefore learns the coin's token type, or at least narrows it to the set of types sharing that rule. That narrowing is permanent, and opted into per token type by the issuer. Whether the proved sender is disclosed more widely is undecided and materially changes the design: opening it only to the gate, inside the commitment, is a different proposal from publishing it. Whether the gate's writes to its own state leak it is a property of the contract, not of the mechanism.

An offer containing such an input is also no longer freely mergeable, which remains unresolved: the authorising call lives in an intent, and offers merge below the intent level, so the transaction is the lowest level at which the input and its call can travel together.

**Open.** Prior to how much the routing leaks is whether it is achievable at all: binding a call to a spend requires publicly identifying *which* rule to run, and the type that names the rule is deliberately never disclosed. Mode 2 is enforced wholly in-circuit and is unaffected. Beyond that, much of what this shape needs does not exist, and none of it is designed here. An attested shielded sender is the first absence: `CallContext.caller` derives from the calling contract's address, else the single common owner of the intent's *unshielded* inputs, else `None`, so a shielded-only intent yields `None` — which [MPS-0029](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0029-compact-caller-identity.md) records as an open question. `kernel.caller()` is no help either: a top-level call reading it fails on chain whenever the intent's unshielded inputs do belong to one user. Derived per intent it would attest one `ZswapCoinPublicKey` across all an intent's shielded inputs, and so could not say which coin was spent by which sender. Nor is there a `PublicAddress` shape for a shielded caller: the type is `User(UserAddress) | Contract(ContractAddress)`, a `ZswapCoinPublicKey` is neither, and it rotates with the wallet's shielded key rather than persisting. It is unsettled whether a spend authorisation joins the call sequencing graph, how guaranteed and fallible lifetime rules apply when one endpoint of a binding is an offer rather than a call, and whether one gate call covers a spend, a spender, or a coin — the last being the question the mergeability note runs into. Re-entrancy is unanalysed: [MPS-0021](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0021-phase2-contract-to-contract.md) keeps it disallowed, and a spend-as-caller edge in the call graph has not been checked against the cycle and sequencing rules. And a mode-3 binding field is a new public input on the `spend` circuit, so it rides the same fork boundary as the derivation change.

### What a gate is handed

A gate is a circuit the issuer writes, handed the list below by the spend circuit. Mode 2's recursion interface makes that list its whole interface; under mode 3 it is a contract entry point and may declare further arguments its caller supplies:

```compact
// the spend guard
circuit <issuer-chosen name>(
  coin: ShieldedCoinInfo,
  nullifier: Bytes<32>,
  spender: Either<ZswapCoinPublicKey, ContractAddress>,
  recipient: Option<Either<ShieldedAddress, ContractAddress>>
): [];

// the receipt-side counterpart, reserved
circuit <issuer-chosen name>(
  coin: ShieldedCoinInfo,
  recipient: Either<ZswapCoinPublicKey, ContractAddress>
): [];
```

A gate sees the coin moving, the value that identifies this movement, the party moving it, and the payee that party declares. `ShieldedCoinInfo` carries nonce, type and value, which is what a rule about an amount needs. The nullifier binds the gate's judgement to this spend and gives a gate per-coin accounting with no further field. The spender is an `Either` because a coin's owner may be a contract: a bare coin public key would put every coin held by a DEX, an escrow or a treasury beyond the mechanism's reach, and that is where much institutional flow sits. The gate approves by returning and refuses by failing an assertion.

The declared payee is whom the sender says is being paid: a `ShieldedAddress` — proposed here as the Compact form of the shielded address a sender already holds, paying a shielded user being impossible without one — or a contract, optional since a spend need not declare one and a change output has none. Nothing binds the declaration and nothing needs to: a key offered as the payee's is a claim about the party deciding whether to accept, checked by decrypting. A disclosure sealed to the wrong key is unreadable by exactly that party, so it is worth making honestly. A rule like *you may only pay X* is not enforced: the spender picks the witness. A contract has no encryption key, so a disclosure-to-payee rule has no honest declaration to make when a contract is paid.

How much a gate can trust these arguments differs by mode. The `spend` circuit fixes the coin, the nullifier and the spender against its own witnesses — a gate judging numbers the spender picked judges nothing — the payee being declared, as above. Under mode 2 it can do that in-circuit, `spender` against the same secret key the nullifier derives from; under mode 3 they are worth the binding sketched above and no more. The spender names a key rather than a person — one key spans that wallet's coins between rotations, so a mode 3 gate retaining it builds a linkable history — and a rule about identity needs its own mapping from key to subject, which [Attribution designs](attribution-designs.md) works through for two such rules.

One omission is deliberate. The qualified form of the coin struct carries `mtIndex`, the coin's position in the commitment tree, and no gate decision needs it — the spend proof already establishes membership. Withholding it is not on its own a privacy guarantee: the arguments a gate does receive are the coin commitment's preimage, so a mode 3 gate that publishes them lets anyone find the leaf. Receipt-side rules belong to the output guard, whose signature the block above reserves, whose recipient is the coin's own, fixed by the circuit that creates it, and whose details are in *Future: guarding receipt as well as spending*.

The circuit's name is the issuer's to choose. It is load-bearing under mode 3, where the descriptor carries the hash of the entry point; under mode 2 the descriptor is a verifying key and the name is a source-level convention.

### Discovery: wallets learn the rule from the chain

Minting is already public, and public in the form this mechanism needs. A contract's transcript records `shielded_mints: HashMap<HashOutput, u64>`, where `HashOutput` is the ledger's generic thirty-two-byte newtype and the key holds the domain separator the minting contract chose. Mints are keyed by that separator, `unshielded_mints` as well as `shielded_mints`; the fields keyed by a derived `TokenType` are `unshielded_inputs` and `unshielded_outputs`. The type is never stored at all: the ledger derives it at validation time from the mint key and the calling contract's address, which is what makes a separator unforgeable across contracts.

Because the separator is published verbatim, and the address of the call carrying the mint is public alongside it, an indexer can build the token type to (contract, domain separator) mapping today. That mapping is what every shielded spend now resolves, guarded or not, and it is free of ledger change; its retention and backfill are what *Binding without lookup* leaves open. The descriptor is not: it has to be published in the mint effect as a field of its own.

That costs a ledger change. `Effects` would have to widen, bumping the `contract-effects[v3]` storable tag and breaking serialization; both VM encode and decode paths follow, including the assert fixing the effects array at nine elements; the kernel operation `kernel.mintShielded` changes, and with it the standard library's `mintShieldedToken` that calls it; and the cost model needs an entry pricing the added field. None of it is prohibitive, and none avoidable: there is no cheap route to a discoverable descriptor.

Without a published descriptor, a guarded coin is unspendable by any wallet not told about the token out of band, which defeats the point of a protocol-level mechanism. The gate's own code — its prover key under mode 2, its executable and call shape under mode 3 — is a further dependency, and not one an indexer can supply.

### Dependency: the capsule runtime

A spender's wallet has to obtain and *run* a gate it has never held, for a token type it may be meeting for the first time. That is the practical crux: without a canonical executable resolvable from the chain, every wallet needs out-of-band delivery of gate code — and, under mode 2, of proving keys — per token type, which makes constructing a valid spend impractical for anyone not already in the issuer's confidence.

Issuer state privacy is the second thread. Today's execution model has the application driving a call supply and run the rehearsal code, and it may therefore see the local state that code reads. That is the wrong shape for a compliance gate: checking a spender against an issuer's private eligibility list should not hand the list, or the query, to the spender's wallet — which is exactly where the check would run.

The capsule runtime removes both obstacles. A contract runs one canonical build inside a capsule that is the only place its local state exists, named by content hash and resolvable through a capsule executable registry at execution time, so a client can execute a gate contract it has never held without seeing the gate's local state. This proposal therefore depends on the Capsule Runtime MIP, "Encapsulated contract execution and local state", Category Core, which answers [MPS-0021](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0021-phase2-contract-to-contract.md). Paired with it, mode 3 is any spend rule expressible as a contract: holding private issuer state, and revisable behind a governed reference without the token type or the holders' coins changing.

Capsules do not, however, supply a provable sender. The runtime's only identity mechanism is a per-capsule secret, independent of the keys holding native tokens and necessarily received by a circuit as a zero-knowledge witness; it authenticates a user to one contract, not a caller to a callee and not a spender to a coin. The capsule proposal is explicit that it is not a coin key. Mode 3's attested caller is a separate dependency.

### The ledger state becomes the evidence

Because the type binds the rule, and the rule is enforced by re-derivation inside every spend, the presence of a coin of type `T` and nonzero value in the commitment tree is *itself* evidence that `T`'s rule held at every hop in that coin's history. There is no attestation to verify, no transaction graph to traverse, and no reliance on the sender's honesty or cooperation. A party that cares about a rule computes the token type that rule implies and checks that the coin it holds is of that type — and it holds the coin, so it knows.

The guarantee runs from the mint, not from the last hop: a coin that skipped the rule would not be a coin of this type.

Under mode 3 the evidence is about the gate rather than its rule: the named contract was consulted at every hop, though what it enforced may have been revised between them. Under mode 2 the descriptor is a verifying key, and the rule itself is fixed.

The strength of the evidence is exactly the strength of the rule the issuer chose, and no more: this proves that a stated rule was enforced, not that any particular fact about a sender is true. A party with a compliance obligation therefore has a different decision to make: instead of assessing each incoming asset, it assesses each *token type* — reviewing the contract or circuit that guards it and deciding whether that guard is adequate for its own legal position. That the guarantee is structural rather than reconstructed is what makes it useful to a party deciding whether to accept an incoming shielded asset, and is the property most relevant to [MPS-0025](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0025-shielded-source-of-funds.md).

### Future: a provable state root for pure rules

Mode 2's blind spot is chain state, and most compliance rules are state-dependent. The way to close that without falling back to mode 3 is to give pure gate circuits a state input they can prove against: a SNARK-friendly, Poseidon-merklized commitment over a designated slice of ledger state, whose root is exposed as a public input to every shielded spend proof. A gate circuit could then prove inclusion or exclusion against that root — absence from a denylist, absence from a freeze list, membership in an approved-holder set — with no contract call, no exposure of the contract, and no loss of privacy. Two details decide whether it works: the root a gate proves against has to be the current one, since a historic-root window of the kind the commitment tree keeps would let a frozen holder prove against a root from before the freeze; and proving absence needs a tree shaped for non-membership.

The prerequisite is bootstrapping such a state in the chain and threading its root into the spend circuit, which is worth doing even before anything writes to it: read paths first, so the input plumbing exists and can be relied on, with write paths — contracts publishing into the merklized state — as a later iteration. State closes mode 2's largest gap and is the direction of travel rather than a prerequisite for the rest; it does not close them all.

Two absences remain. The standard library has the curve and hash primitives an encrypting gate needs, no encryption primitive, and no randomness beyond what the prover witnesses. What it wants is a VRF, and one whose output is not the prover's free choice. None exists there, though an EC-VRF is constructible from `hashToCurve` and `ecMul`, so the gap is a primitive and a cost model rather than missing cryptography. Even then a gate is not self-sufficient: a VRF is evaluated under a secret key and a gate holds none, so evaluation stays in the `spend` circuit and its output is handed in.

### Future: guarding receipt as well as spending

The same construction guards the other side of a transfer. A coin cannot be *created* unless a designated circuit or contract approves the recipient — which is what it takes to prevent a transfer *to* a party that fails a compliance check, rather than merely preventing that party from spending afterwards. Preventing receipt is the stronger and, for some obligations, the only acceptable form: a sanctioned party that can be paid but not spend has still been paid.

Under such an extension a token type binds a pair of guards, and the mint takes `(separator, spend_guard, output_guard)`, and the descriptor carrying them into the derivation is simply longer. The output guard is enforced by the output circuit, which knows the recipient, on the same re-derivation principle.

One consequence has to be settled early, though the extension itself is out of scope. Because the guards are bound into the token type, adding the output-guard slot *later* changes the derivation and therefore changes every **guarded** token type derived under it; unguarded types, whose descriptor stays empty, are unaffected. An already-issued asset cannot adopt an output guard in any case: a guard it did not have makes it a different type. If receipt-side guarding is wanted at all, the descriptor should reserve the slot from the outset, with "no output guard" as the default, as mode 1 is on the spend side.

The output side binds on the same terms. The output circuit has no Merkle path to appeal to — it is creating the coin, not opening an existing one — but it computes the commitment it inserts, and so holds the type it is committing to. Re-derivation asserted there unconditionally therefore pins the output guard exactly as it pins the spend rule, the empty slot again leaving today's preimage byte-identical.

### Future: privacy-preserving seizure

Seizure needs two decouplings from the owner's secret. **Discovery**: the authority has to open the coin's commitment, which a disclosure encrypted to the authority at output time supplies, provided the output circuit proves that ciphertext well-formed under a key pinned in the descriptor alongside the rule. Today's mechanism only fixes an arbitrary input to the proof, so without that step discovery rests on the sender's honesty, which is the one thing seizure cannot assume. Pinning the key inherits the reservation argument above: a key added later rederives every guarded type, and rotating it changes the token type. **Retirement**: the authority has to compute the coin's single canonical nullifier without the owner's secret. The second is the protocol change.

Retirement turns on what the nullifier binds to. A signature scheme names an authoriser the owner chose, so the nullifier may keep deriving from owner-held material; seizure names an authoriser the owner did not choose, so the nullifier must be derivable with no owner secret at all. That is what costs unlinkability, and what the cheap constructions get wrong: an escrow key fixed at mint makes the authority a permanent unilateral spender, and a nullifier bound to public data makes the coin publicly watchable.

Burn-and-reissue by the issuer is a different thing: it needs no protocol change, mint and burn authority being sufficient, but it does not retire the original note, which stays live in the commitment tree and spendable by its holder, and so breaks the supply invariant.

[MPS-0013](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0013-zswap-business-logic.md) is the document that asks for seizure proper — involuntary reassignment under issuer authority, with a mandatory public event, on death or by court order. A spend rule can express the authority's own conditions in either mode; retirement it cannot yet express, and this proposal does not specify it.

## Rationale

Explain the design decisions that were made and the reasons behind them.
Why was this particular approach chosen?
What alternatives were considered, and why were they rejected?

## Path to Active

What does it mean to get from Accepted to Active, and how this will be achieved.

### Acceptance Criteria

Explain what objective milestones need to be achieved in order for the MIP to achieve Active status.

### Implementation Plan

Describe how the MIP will be put into practice.

## Backwards Compatibility Assessment

Describe how the proposed change affects existing systems, applications, and users.
Will it require a hard fork?
Are there any compatibility issues?
How will they be addressed?

## Security Considerations

Analyze the potential security implications of the proposed change.
Are there any new attack vectors or vulnerabilities introduced?
How will they be mitigated?

## Implementation

Describe how the proposed change will be implemented.
Which parts/components of the Midnight need to be modified?
What are the dependencies, if any?

## Testing

Describe the testing procedures for the proposed change.
What tests will be performed to ensure that it works as expected and does not introduce any regressions?

## References (Optional)

Are there any external sources that are referenced in this document, or that add to the efficacy of this MIP?

## Acknowledgements

List the contributors that were not the Authors, this will include any workshop participants.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
