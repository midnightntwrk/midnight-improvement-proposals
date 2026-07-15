---
MIP: "0012"  
Title: Contract Custody of Midnight-Native Assets  
Authors:
  - Nicolas Di Prima (NicolasDP)
  - Hector Bulgarini (hbulgarini)  
Status: Proposed  
Category: Standards  
Created: 2026-07-07  
License: Apache-2.0  
Requires: none  
Replaces: none  
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

This MIP specifies how a Compact contract custodies Midnight-native
values: unshielded tokens (Night and any other unshielded color) and
shielded Zswap coins. It is the
first building block of the multi-key account contract recommended by
MPS-0018, deliberately limited to the asset surface. Who may authorise a
release, how credentials are managed, and how control is recovered are
the subjects of companion building blocks; this document specifies only
the seam they attach to.

Deposits are permissionless. Releases are gated by a single internal
authorisation seam whose observable semantics this MIP fixes and whose
credential scheme it deliberately does not. Unshielded custody pairs the
protocol receive and send operations with an explicit per-color balance
mirror. Shielded custody is **stateless**: a held coin's description
(nonce, color, value) never enters public ledger state. Deposits pair
the protocol-level coin claim with an encrypted inbox entry, addressed to
an account encryption key advertised on-ledger, and spends supply the
coin description as a private witness from a wallet-local coin store. An
observer learns that the custody contract acted, but never the amounts,
the token types, the balances, or the counterparties of conforming
payments. This removes the holdings-publicity trade-off carried by the
prevailing `Map<color, QualifiedShieldedCoinInfo>` pattern, and the
stateless custody pattern has been validated end to end against a
production node, including a byte-level audit of every observer-visible
surface.

The same encryption that hides holdings also separates reading from
releasing: reconstructing the contract's shielded position needs only the
account encryption secret, whereas releasing assets needs the authorisation
seam, so that secret can be delegated as a read-only viewing capability —
to an accountant, auditor, or compliance function — without ceding custody
(Rationale R9).

The specification also makes normative a change-handling rule (persist
the change coin that leaves the transaction live, never a coin the
transaction consumed) whose violation has been observed to silently
strand funds in ecosystem library code; the defect is fixed upstream,
and the rule keeps the failure class out of conforming implementations.

## Motivation

MPS-0018 frames the problem: Midnight has no ratified model for an
on-chain account that custodies and authorises a user's native assets
under multi-key, multi-device control while satisfying lost-device
recovery, total-loss recovery, and key non-exfiltration at once. Its
first recommended MIP is the multi-key account contract, "the keystone
the others hang from". This document delivers the custody half of that
keystone as a free-standing building block.

**Why a building block rather than the whole keystone.** The custody
surface and the authorisation surface stabilise on different clocks. How
a contract holds and releases value is settled by ledger mechanics and
has been validated experimentally. How callers should be authenticated
is an active, unsettled conversation upstream: in-circuit verification
of native address ownership has been requested in the
[token-standards discussion](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/142),
a [stdlib caller-identity primitive](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/213)
has been proposed, and threshold and MPC custody impose their own
constraints. Binding the
custody rules to any one credential scheme would hold the settled part
hostage to the unsettled part. Splitting them also keeps each document
small enough to review well, and lets institutional custody profiles
(MPS-0006, MPS-0016, MPS-0024) reuse the asset surface under their own
authorisation regimes.

**A refinement to the problem statement.** MPS-0018 records that
contract custody of shielded notes "leaks holdings into public ledger
state". Experimental evidence summarised in the Rationale shows the
ledger itself forces no such leak: the publicity is a property of the
prevailing storage pattern, which writes each held coin's full
description into public contract state so later spends can look it up.
A stateless pattern removes the leak. This MIP therefore specifies
custody that does not pay the privacy price MPS-0018 had to assume, and
the corresponding MPS-0018 open question resolves to a narrower one
about residual metadata (Security Considerations S4).

**The prevailing recipe stranded change.** `sendShielded` already
routes change back to the sending contract as a live, self-owned
output. Widely used library code nevertheless followed it with a
redundant re-owning call, `sendImmediateShielded(change, self)`, which
consumes the change coin (revealing its nullifier in that same
transaction) and creates a replacement returned as the `sent` field of
its result; the code then persisted the consumed coin, so every later
spend of change was rejected by the node as a double spend. The defect
was reported upstream with a node-level reproducer and has been fixed
by removing the redundant step. A standard is still the right place to
state the surviving-coin rule normatively, because implementations
that deliberately route change onward re-enter the two-coin situation
the defect lived in (section 6.3).

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in RFC 2119.

### 1. Terminology

- **Custody contract**: the Compact contract instance specified here,
  holding assets on behalf of exactly one account.
- **Instantiating standard**: a companion document that instantiates
  the authorisation seam (section 4) with a concrete credential scheme
  and lifecycle: the account standard for personal custody, or an
  institutional custody profile. This MIP does not depend on any
  particular one.
- **Coin description**: the `(nonce, color, value)` triple of a
  shielded coin; **qualified** when extended with its commitment-tree
  index (`mt_index`).
- **Coin store**: wallet-local private state holding the qualified coin
  descriptions of the contract's shielded holdings.
- **Inbox**: an append-only map of encrypted coin descriptions in the
  custody contract's public state; the discovery and recovery channel.
- **Viewing capability**: the account encryption secret used as a
  read-only credential — it decrypts the inbox and drives the discovery
  walk (section 6.5), reconstructing the contract's shielded holdings from
  chain data, and confers no authority to release assets (Rationale R9).
- **Color**: the 32-byte token-type identifier of an unshielded or
  shielded token, spelled throughout as the ecosystem term of art.
- **Night**: the native unshielded token, whose color is the
  all-zero value. This MIP treats it as one unshielded color among
  many.

### 2. Scope and composition

This MIP specifies the custody of unshielded tokens and shielded Zswap
coins, across all colors of each class, by a per-account contract
instance. The following are explicitly out of
scope, and conforming extensions MUST NOT weaken any invariant in
section 7 when adding them:

- **Authorisation schemes and key management**: credential formats,
  device sets, revocation, and ceremonies belong to the instantiating
  standard. This MIP fixes only the seam semantics (section 4).
- **Recovery mechanisms**: a companion standard may change who can
  satisfy the seam; the custody rules are unaffected by construction,
  because assets sit in the contract and never move during recovery.
  The availability and recovery of the account encryption secret is
  likewise the recovery standard's subject; this MIP requires only that
  a client holding that secret can execute the inbox walk (section
  6.5), and specifies the on-ledger key lifecycle (section 6.7).
- **Delegated or scoped spending policies** (grants, allowances,
  session permissions): these are authorisation-policy objects behind
  the same seam.
- **Dust**: the custody contract holds no Dust; where Dust balances
  live and how fees are paid without fragmenting custody is the open
  question MPS-0018 records, and is out of scope here.
- **Token issuance**: minting and burning of a contract's own colors
  is specified by MIP-0011 for the shielded class, and by the ledger's
  contract-mint effects for the unshielded class. A custody contract
  holds arbitrary colors it did not issue.
- **Name discovery**: locating a custody contract by human-readable
  name is the name service's subject (MIP-0007).

Each account is one contract instance. A registry topology was
considered and rejected (Rationale R2).

### 3. Public ledger state

```compact
export ledger round: Uint<64>;
// Strictly increased by every state-changing circuit. Gives the seam a
// replay anchor and observers a cheap consistency post-condition.

export ledger enc_key: Bytes<32>;
// The account encryption public key. Depositors encrypt coin
// descriptions to this key (section 6). Set at deploy; rotated only
// through the authorised circuit of section 6.7.

export ledger inbox: Map<Uint<64>, Bytes<192>>;
export ledger inbox_count: Uint<64>;
// Append-only encrypted coin descriptions (InboxEntry, section 6.4),
// keyed by ordinal.

export ledger unshielded_balances: Map<Bytes<32>, Uint<128>>;
// Explicit mirror of the contract's unshielded holdings per color.
// Night is the all-zero color.
```

Implementations MAY extend this state (for example with the account
standard's credential structures) provided every invariant in section 7
still holds. Implementations MUST NOT add state that records any part of
a held shielded coin's description (section 6.1).

### 4. The authorisation seam

Every circuit that releases assets, and every state-changing circuit
other than deposits and inbox appends, MUST be gated by a single
internal authorisation check, hereafter `require_authorised()`, with the
following observable semantics:

1. The caller proves authority by a mechanism an instantiating
   standard specifies. The proof MUST bind to the specific call: at
   minimum to the circuit identity, its arguments, and a freshness
   element that the contract state advances (the `round` counter of
   section 3). A witness-supplied secret entering the call's own proof
   satisfies the binding, freshness included, by construction, because
   the transaction transcript pins the pre-state it executes against; a
   transferable artefact such as a signature MUST cover the freshness
   element explicitly, since INV-7 on its own does not make stale
   authorisations fail.
2. On success, the circuit proceeds and `round` is incremented.
3. On failure, the circuit aborts; no state changes.

Conforming contracts MUST document which instantiating standard their
seam implements.

The check MUST be one internal seam such that replacing the credential
scheme does not alter any other circuit's specification. Implementations
MUST NOT derive caller identity from wallet-supplied transaction
metadata that the circuit does not constrain; in particular,
`ownPublicKey()` MUST NOT be used for authorisation, a rule MIP-0011
states for token contracts and which applies here identically.

Candidate seam instantiations, none of which this MIP requires, include
proof of knowledge of a committed secret, in-circuit signature
verification, a future stdlib caller-identity primitive (proposed
upstream; note that a caller derived from unshielded input ownership
cannot authenticate shielded callers, and binds authority to a single
ledger-scheme key rather than to an account policy or to a threshold
scheme on another curve), and threshold schemes presenting any of the
former. The instantiating standard selects and profiles these.

Deposits (sections 5 and 6.2) and inbox appends (section 6.2) are
permissionless: anyone may fund the account or contribute a discovery
entry. They still increment `round`.

### 5. Unshielded custody

The unshielded class is color-generic: Night is the native color, and
any contract may mint further unshielded colors through the ledger's
contract-mint effects. This MIP custodies every unshielded color
identically; nothing in this section is specific to Night.

- `deposit_unshielded(color, amount)` (permissionless): receives the
  unshielded tokens for the contract and credits `unshielded_balances`
  for that color.
- `withdraw_unshielded(color, amount, recipient)` (authorised): checks
  and debits the mirror, then sends the tokens to a user address.

The mirror exists because contract unshielded balances are not
otherwise readable from contract state. Tokens transferred to the
contract outside `deposit_unshielded` are held but not mirrored;
clients MUST treat the mirror as a lower bound on holdings.
Implementations MAY provide an authorised reconciliation circuit that
lifts the mirror to observed deposits; such a circuit MUST NOT credit
beyond deposits observed on-chain, and the ledger's protocol-level
balance check remains the backstop against over-withdrawal even if the
mirror drifts high. Withdrawal recipients are user addresses;
contract-to-contract unshielded transfer is not part of this standard.

### 6. Shielded custody (stateless)

#### 6.1 Design rule

The custody contract MUST NOT write a held coin's description (nonce,
color, or value), in whole or in part, into public ledger state or into
any publicly disclosed circuit output. The only coin-derived values that
may appear publicly are the protocol-level hiding commitments and
nullifiers, and the single-bit comparison outcomes inherent to the
standard library send path.

#### 6.2 Deposit

```compact
export circuit deposit_shielded(coin: ShieldedCoinInfo, entry: Bytes<192>): []
```

Semantics: claims the coin for the contract (`receiveShielded`), appends
`entry` to the inbox, increments `inbox_count`, increments `round`.
Deposits are permissionless.

`entry` MUST be an `InboxEntry` (section 6.4) encrypting the coin
description to `enc_key`. The depositor knows the coin description by
construction: they built the Zswap output. The contract cannot verify
the correspondence between `coin` and `entry`; the consequences and the
trust model are addressed in Security Considerations S3.

The ledger rejects encrypted-payload fields on contract-owned Zswap
outputs, so the standard coin-ciphertext channel that user-held coins
enjoy is unavailable for deposits to contracts; the inbox is its
replacement. MIP-0011 documents the same constraint for contract-minted
coins and resorts to out-of-band delivery; custody cannot, because
discovery and total-loss recovery must work from chain data alone
(Rationale R5).

A separate circuit, `append_inbox(entry)`, appends an entry without a
coin claim. Clients use it to backfill entries for change coins, whose
descriptions only exist once the spend has executed (section 6.3): the
spend returns the surviving change description privately to the caller,
and the client encrypts it and appends the entry in a later
transaction. All InboxEntry cryptography is performed client-side, by
depositors and by the owner's client; the contract never encrypts,
decrypts, or inspects entries, and stores them as opaque bytes.
Implementations MAY gate `append_inbox` behind the authorisation seam:
change-coin backfill is always performed by the owner's own client, so
gating it loses nothing, while the entry paired with `deposit_shielded`
MUST remain permissionless. Spam against an ungated append surface is
addressed in Security Considerations S9.

#### 6.3 Spend and the change rule

```compact
witness held_coin(color: Bytes<32>): QualifiedShieldedCoinInfo;

export circuit withdraw_shielded(
  recipient: ZswapCoinPublicKey,
  color:     Bytes<32>,
  amount:    Uint<128>,
): Maybe<ShieldedCoinInfo>
```

Semantics (authorised):

1. The qualified coin description enters the circuit through the
   `held_coin` witness, from the wallet-local coin store. It is never
   read from ledger state. Contract-held coins are Merkle-path spends
   (`sendShielded` over a `QualifiedShieldedCoinInfo` with a valid
   `mt_index`), per the spend-path discipline MIP-0011 codifies.
2. `sendShielded(coin, recipient, amount)` executes the spend.
3. **Change rule (normative).** The coin recorded as held after a
   partial spend MUST be live at the end of the transaction: its
   commitment created by the transaction and its nullifier not
   revealed in it. `sendShielded` already routes change back to the
   contract as a self-owned output, so the baseline realisation is to
   persist the change coin of its result directly; no re-owning step
   is needed. A circuit MAY instead spend the change onward in the
   same transaction via `sendImmediateShielded`, which is meaningful
   only towards a recipient other than the contract itself; a circuit
   that does so MUST treat the **`sent` field of that call's result**
   as the surviving coin, and MUST NOT persist, return, or record the
   coin that call consumed, whose nullifier is revealed in that very
   transaction and whose later spend the node rejects as a double
   spend. Rationale R4 documents the failure this rule prevents and
   its upstream history.
4. The circuit returns the surviving change coin description (or none)
   to the caller. Circuit return values travel in the transaction's
   communication commitment and are not public.
5. `round` is incremented.

The recipient type is deliberately `ZswapCoinPublicKey` and not
`Either<ZswapCoinPublicKey, ContractAddress>`: see section 6.6.

#### 6.4 InboxEntry format

`InboxEntry` version 1 is a fixed 192-byte container:

| Offset | Length | Field |
|---|---|---|
| 0 | 1 | `version` = `0x01` |
| 1 | 1 | `suite` = `0x01` (X25519 + HKDF-SHA256 + AES-256-GCM) |
| 2 | 32 | ephemeral X25519 public key |
| 34 | 12 | AEAD nonce |
| 46 | 16 | AEAD tag |
| 62 | 80 | ciphertext |
| 142 | 50 | zero padding |

Plaintext (80 bytes): `nonce (32) || color (32) || value as unsigned
128-bit big-endian (16)`. The AEAD key is
`HKDF-SHA256(ikm = X25519(eph_sk, enc_key), info = "midnight:custody:inbox:v1")`
with an empty salt (the RFC 5869 default) and an output length of 32
bytes; the AEAD associated data is the first two container bytes
(`version || suite`). Writers MUST set the padding to zero; readers
MUST ignore it. The `info` tag is to be registered under the
domain-separation registry recommended by MPS-0027. The `mt_index` is
deliberately not stored: it is recomputed from chain data (section
6.5), which keeps entries valid independently of tree state.

Readers MUST skip entries with an unknown `version` or `suite` rather
than treat them as errors; new suites require a revision of this
section.

#### 6.5 Coin store, index capture, and discovery

Clients MUST maintain a wallet-local coin store, with the following
behaviour:

- After a deposit, the client records the coin description and captures
  `mt_index` from the depositing transaction's commitment-tree
  positions; for a single-output transaction this is the transaction's
  start index.
- After a spend with change, the client records the returned change
  coin and captures its `mt_index` from the spend transaction's
  position window, then backfills the inbox via `append_inbox` no later
  than its next inbox write (invariant INV-4).
- Where a transaction carries several commitments, clients MAY resolve
  the index by candidate retry: an incorrect qualified description
  yields an unsatisfiable witness at proving time and no transaction is
  submitted, so retry cannot mis-spend (INV-5 relies on this).

Discovery MUST be possible from public chain data alone. The normative
procedure is the **inbox walk**: enumerate `inbox[0 .. inbox_count)`,
decrypt each entry with the account encryption secret, skip entries that
fail authentication or version checks, verify each candidate coin by
recomputing its commitment against chain data, and capture indices as
above. This procedure serves both third-party-deposit discovery and
total-loss reconstruction of the coin store. Locating the depositing
transactions benefits from an indexer surface that enumerates a
contract's transactions by address; absent one, clients fall back to a
block-range scan or candidate retry. The indexer surface is recorded as
an ecosystem dependency in Path to Active.

#### 6.6 Counterparty rules

- A conforming custody contract MUST NOT create a shielded output whose
  recipient is a contract address, except the self-recipient change
  output of section 6.3. A shielded output addressed to a contract
  carries that contract address in cleartext and must be claimed by the
  receiving contract in the same transaction, so a direct
  contract-to-contract payment publishes the pair of accounts. (Outputs
  addressed to a different contract are additionally rejected by the
  current ledger.)
- Payments between two custody contracts MUST be routed as one hop: the
  paying contract sends to a shielded coin public key controlled by the
  recipient; the recipient deposits into their own contract in a
  separate transaction. The observer then sees two events that share no
  address, amount, or token type.
- Because a coin deposited directly into a custody contract has no
  owner secret, its depositor can compute its nullifier and observe its
  first onward move, including deterministically derived same-value
  descendants. Clients SHOULD break this trail for received funds (for
  example by splitting value across spends, or routing through a
  user-held coin) before privacy-sensitive use.

#### 6.7 Encryption-key lifecycle

`enc_key` is set at deploy. Implementations MUST expose exactly one way
to change it: an authorised circuit, `rotate_enc_key(new_key)`, gated by
the seam and incrementing `round`. After a rotation the client SHOULD
re-encrypt the descriptions of all held coins into fresh inbox entries
under the new key, so that the inbox walk (6.5) succeeds with the new
secret alone; entries under the old key remain on-ledger and readable to
whoever holds the old secret (Security Considerations S2). Custody of
the encryption secret itself, including its recovery after total loss,
is the instantiating and recovery standards' subject.

### 7. Invariants

A conforming implementation MUST satisfy all of the following.

- **INV-1 (single seam).** Every asset-releasing operation is gated by
  `require_authorised()`; no circuit releases assets on any other
  authority.
- **INV-2 (no public coin material).** No execution of any circuit
  writes a held coin's nonce, color, or value into public ledger state
  or into the public transcript, beyond protocol commitments,
  nullifiers, and the send path's single-bit comparison outcomes.
- **INV-3 (change continuity).** After a partial spend, the coin
  recorded as held is spendable in a later transaction. Equivalently: a
  coin whose nullifier the spend transaction revealed is never what
  survives.
- **INV-4 (reconstruction completeness).** For every coin the contract
  holds, the inbox contains an entry from which the coin description is
  recoverable by the holder of the account encryption secret, subject
  to the depositor trust assumption (S3). For change coins the client
  MUST append the entry no later than its next inbox write.
- **INV-5 (capture safety).** No coin-store capture strategy can cause
  an unintended spend: an incorrect qualified description fails at
  proving, before any transaction exists.
- **INV-6 (counterparty unlinkability).** A conforming payment between
  two custody contracts produces no transaction containing both
  contract addresses.
- **INV-7 (round monotonicity).** Every state-changing call strictly
  increases `round`.
- **INV-8 (mirror soundness).** `unshielded_balances` never exceeds
  the contract's actual unshielded holdings per color; a conforming
  withdrawal never debits below zero.

### 8. Versioning

- This specification is versioned by its MIP number and revision
  history; substantive changes after acceptance require a new MIP that
  lists this one in `Replaces`.
- The `InboxEntry` container is self-versioned (leading version byte);
  readers MUST tolerate unknown versions (6.4).
- Contracts SHOULD expose a constant `spec_version` cell so clients can
  gate features; instances are otherwise immutable per the ledger's
  deploy semantics, and fleet migration expectations belong to the
  implementation, not to this specification.

## Rationale

**R1. Contract custody rather than address custody.** Address custody
(assets at seed-derived addresses) reintroduces a seed-shaped root and
decouples assets from the account object, so revocation and recovery
stop covering them, violating the MPS-0018 goals. Contract custody keeps
"recovered account implies recovered assets" true by construction:
assets sit in the contract, and a change of who satisfies the seam moves
nothing. The historical objection to contract custody of shielded assets
was the publicity leak, which section 6 removes.

**R2. Per-account contracts rather than a registry.** A registry
concentrates every account's activity behind one address, which improves
the metadata anonymity set, but it makes the ledger schema
ecosystem-wide, concentrates upgrade risk, and turns the registry into a
single censorship and correlation point at the application layer.
Per-account instances keep schema evolution per-user and isolate
failure. A future shared-custody profile with witness-private account
binding can widen the anonymity set without changing this
specification's invariants; it is left as a successor proposal.

**R3. Stateless shielded custody rather than the public-map pattern.**
The alternatives were measured against a production node, with a
byte-level audit of every observer-visible surface (raw transactions and
per-call contract state), running identical deposit, partial-spend, and
change lifecycles through both patterns:

| Property | Public map (`insertCoin`) | Stateless (this MIP) |
|---|---|---|
| Coin nonce visible to observers | yes, verbatim | no |
| Token type visible | yes, verbatim | no |
| Amounts and balances visible | yes | no |
| Spend watchable in advance (nullifier precomputable by anyone) | yes | only by parties who know the coin description |
| Deposit-to-spend linkable by observers | yes | no |
| Node accepts spends | yes | yes, including change chains |
| Client complexity | none beyond calls | coin store, index capture, inbox handling |

The compiler's disclosure analysis independently bounds the stateless
pattern's public surface: the forced disclosures are hash links
(commitments and nullifiers) and single-bit comparison outcomes, never
raw coin fields. MIP-0011's own analysis of coin operations reaches the
same conclusion from the issuance side: coin operations emit only
commitments and nullifiers, and, mint effects aside (which do not arise
in custody), the only way a value becomes public is an explicit ledger
write. What the stateless design costs is wallet-side
state management, which section 6.5 specifies, with the inbox as its
durable backstop.

**R4. The change rule.** `sendImmediateShielded` consumes the coin it
is given (its nullifier is revealed in that same transaction) and
creates a replacement with an evolved nonce, returned as `.sent`. The
prevailing treasury recipe applied it redundantly to change that
`sendShielded` had already routed back to the contract, and then
persisted the consumed coin; every later change spend failed as an
attempted double spend while the replacement sat unrecorded. The
failure was reproduced on a production node through both custody
patterns, in the treasury modules of a widely used contract library
(the same pattern MPS-0018 cites as the working recipe), reported
upstream with the reproducer, and resolved by removing the redundant
step so implementations persist the live change coin of the send
result. This MIP states the rule at the level of the surviving coin so
both realisations conform: the baseline (persist the send result's
change directly, which is also MIP-0011's guidance for its burn flows)
and deliberate onward routing (persist that call's `sent` coin). The
node-level evidence behind this rationale covers both realisations:
the re-owning variant was validated when the defect was reproduced,
and the baseline realisation has since passed the same change chains
on a node through both custody patterns, including the
cross-transaction change spend that simulator tests cannot exercise.
Conformance test 3 keeps that check as the standing regression gate.

**R5. Inbox rather than out-of-band delivery or ledger ciphertexts.**
Ledger ciphertexts on contract-owned outputs are rejected by the node,
so the encrypted channel that user coins enjoy is unavailable.
Out-of-band delivery, which MIP-0011 accepts for mint recipients, breaks
the "chain data is sufficient" property (INV-4) and adds a liveness
dependency between depositor and owner; for custody, discovery and
total-loss reconstruction are requirements, not conveniences. An
on-ledger encrypted inbox costs contract state but makes both possible
from chain data alone.

**R6. An abstract seam rather than authorisation profiles.** An earlier
draft of the keystone specified credential profiles (hash preimage,
in-circuit signature) alongside custody. Profiles belong to the
instantiating standards: the upstream conversation on caller identity
is active, and
custody must not re-open every time it moves. What custody genuinely
depends on is only the seam semantics of section 4, which every
candidate scheme can satisfy. This also lets institutional custody
(MPS-0006, MPS-0016, MPS-0024) adopt the asset surface unchanged:
MPS-0024's core problem, that asset movement is proof-authorised rather
than signature-authorised, is eased under contract custody, because
spend authority becomes the contract's seam rather than possession of a
coin-secret witness, and the seam can be instantiated with whatever
threshold-controlled mechanism the custodian's control framework
requires.

**R7. What stays visible.** Two leaks are structural to any contract
custody on today's ledger and are documented rather than hidden:
deposits to and spends from a custody contract are labelled with its
contract address (activity metadata: counts and timing, not contents);
and the depositor of a coin can watch that coin's first onward move
(section 6.6 mitigations). Within those bounds, amounts, token types,
balances, and conforming counterparties are hidden.

**R8. Relationship to neighbouring standards.** MIP-0011 governs a
contract's own issued colors (mint and burn); this MIP governs holding
arbitrary colors on an account's behalf; the two compose, and this MIP
adopts MIP-0011's spend-path discipline and caller-authentication
prohibition. Token registry and NFT standardisation effects are
upstream of custody only at the metadata layer. The name service
(MIP-0007) locates accounts; nothing here depends on it.

**R9. Viewing delegation (read/write separation).** Reconstructing the
contract's shielded holdings and history from chain data requires only the
account encryption secret: it drives the inbox walk (section 6.5), which
yields every coin description, amount, color, and reconstructable
counterparty. Releasing those assets requires the authorisation seam
(INV-1). The two capabilities live in different secrets, so read and write
separate cleanly — the encryption secret is a pure viewing capability that
confers no power to move a single coin, a corollary of INV-1 rather than a
new mechanism. This makes read-only delegation a first-class affordance of
the design: a holder can hand a purpose-specific encryption secret to an
accountant, an auditor, a tax authority, or an institutional compliance
function — the oversight need the institutional problem statements
(MPS-0006, MPS-0016, MPS-0024) carry — and obtain third-party bookkeeping
over otherwise private holdings without ceding custody. The unshielded
balances are already public through the section 5 mirror, so a delegate
holding the viewing capability sees the whole position: public unshielded,
private shielded. The delegation is out of band — it shares a secret,
invokes no circuit, and needs no ledger support — which is why this MIP adds
no viewing mechanism; custody of the encryption secret itself remains the
instantiating and recovery standards' subject (section 2, section 6.7).

The capability is coarse, deliberately. A single encryption secret governs
the whole inbox, so disclosing it reveals the contract's entire shielded
past and future until `enc_key` is rotated (section 6.7); there is no native
scoping to a date range, a coin subset, or a color, and revoking a
delegate's view means rotating the key and re-encrypting the live entries,
after which the delegate retains whatever it has already read (Security
Considerations S2). Finer-grained viewing — hierarchical or epoch-scoped
viewing keys, or an incoming/outgoing split in the manner of shielded-pool
viewing keys — advertises additional keys without disturbing any invariant
here, and is left to a successor standard, the read-side analogue of the
scoped-spending policies section 2 defers.

## Path to Active

### Acceptance Criteria

- [ ] MIP number assigned by an editor; listed in MPS-0018's header as
      a proposed solution, with the MPS-0018 problem statement refined
      per the Motivation.
- [ ] Cryptographer review of section 6 (secretless-coin analysis, the
      InboxEntry construction, and the disclosure bounds) with findings
      addressed.
- [ ] The change rule (6.3, R4) upheld upstream: the defect report and
      library fix are merged; node-level validation of the baseline
      realisation's cross-transaction change spend (conformance test 3)
      completed against a fixed release.
- [ ] A public reference implementation passing the conformance suite
      (Testing section), including the observer leak audit with a
      passing positive control.
- [ ] A second, independent implementation by a wallet provider or
      ecosystem team interoperating with the reference (deposit by one,
      discovery and spend by the other).
- [ ] Indexer surface (or documented block-scan procedure) for
      enumerating a contract's transactions by address, sufficient for
      third-party-deposit discovery without depositor cooperation.
- [ ] Deployment exercised on a public test network.

### Implementation Plan

1. Publish the reference implementation and conformance suite.
2. Track the upstream change-rule fix (report filed, fix merged)
   through node-level validation of the fixed pattern; no MIP-0011
   change is needed, since its guidance matches the baseline
   realisation.
3. File and track the indexer discovery surface.
4. Engage a wallet provider for the independent implementation.
5. Submit for editor numbering; iterate through community commentary.
6. Author the companion account standard against the seam, followed by
   the recovery-paths MIP recommended by MPS-0018.

## Backwards Compatibility Assessment

This MIP introduces a new contract standard. It requires no ledger,
consensus, or node change and no hard fork: every mechanism used
(contract-owned Zswap coins, witness-supplied spends, send-level
change routing back to the contract, contract state) exists in the
current stable network protocol. Existing deployments of the public-map pattern remain
valid; they do not conform to INV-2 and carry the documented publicity
leak, and migrating requires spending held coins into a conforming
instance. Wallets unaware of this standard are unaffected.

## Security Considerations

- **S1. Secretless coins.** A contract-owned coin's commitment and
  nullifier are deterministic functions of its description and the
  contract address. Anyone who learns a coin description can watch that
  coin. The standard therefore treats coin descriptions as secrets
  (witness supply, encrypted inbox) and documents the depositor
  exception (R7, 6.6).
- **S2. Encryption-key compromise.** Disclosure of the account
  encryption secret reveals the contract's holdings and enables
  watching, but does not enable theft: releasing assets still requires
  the authorisation seam (INV-1). Key rotation is the procedure of
  section 6.7; old entries remain readable to the attacker, so rotation
  bounds future, not past, exposure. The same property is a feature under
  deliberate disclosure (delegated viewing, R9): the encryption secret is a
  read capability and never a release capability, so the trust boundary is
  identical whether it leaks or is shared on purpose.
- **S3. Inbox poisoning.** The contract cannot verify that a deposit's
  inbox entry matches the deposited coin. A malicious depositor can
  deposit a coin whose entry is garbage, producing value the owner
  cannot reconstruct from chain data; the depositor, who knows the
  description, could later reveal or exploit it. The owner loses nothing
  they previously held, and honest clients detect mismatches at
  discovery by recomputing the commitment against chain data. Binding
  the entry to the coin in-circuit (proving correct encryption of the
  claimed coin) is possible in principle at significant circuit cost; it
  is left as a hardening extension.
- **S4. Metadata.** Activity metadata per custody contract is public
  (R7): event counts, timing, and the contract address on every deposit
  and spend. Applications with strong unlinkability requirements SHOULD
  consider the shared-custody successor profile (R2) and traffic-shaping
  practices. This is the residual form of the MPS-0018 privacy question.
- **S5. Seam misuse.** The seam MUST NOT be satisfiable by
  wallet-supplied, circuit-unconstrained data; `ownPublicKey()` is the
  canonical counter-example (section 4). A seam satisfiable without any
  witness-supplied secret or verified credential is not conforming: a
  custody contract whose seam is vacuous publishes no coin material but
  protects nothing.
- **S6. Hash discipline.** All domain-separation tags used by this
  standard (currently the InboxEntry HKDF `info` string) MUST be
  registered under the registry recommended by MPS-0027 once it is
  established, and commitments stored in ledger state MUST use hash
  constructions whose outputs are stable across toolchain versions.
- **S7. Availability of the coin store.** Between a change-producing
  spend and the corresponding inbox append (INV-4 allows the next
  write), the only holder of the change description is the client.
  Clients SHOULD minimise this window; the reference behaviour appends
  immediately.
- **S8. Mirror drift.** The unshielded mirror is a lower bound
  (section 5).
  Treating it as exact can under-report holdings but cannot enable
  over-withdrawal (INV-8): the ledger enforces actual balances at the
  protocol level.
- **S9. Inbox spam.** An ungated `append_inbox` lets anyone grow the
  contract's public state and inflate the cost of the inbox walk.
  Entries that fail AEAD authentication are skipped at negligible cost,
  and transaction fees bound the attack economically; implementations
  for which this is insufficient MAY gate `append_inbox` behind the
  seam (section 6.2) without losing any specified behaviour.

## Implementation

- Evidence base: the stateless custody experiment in the Midnight
  Passport workspace validates the custody mechanics against a
  production node (deposits, witness spends, change chains, third-party
  deposit and discovery, and the observer leak audit), using a 160-byte
  unversioned inbox container with a key-derivation stand-in; the
  versioned container of section 6.4 is specified here and awaits the
  reference implementation required by Path to Active. The
  account-custody prototype validates the unshielded surface (under
  Night-specific naming) and a working seam instantiation; it predates
  this specification and diverges from it in known ways (its deposits
  do not increment `round`, and its shielded path is the public-map
  control pattern).
- Upstream status: the change-rule defect report and the library fix
  are filed and merged (References); the fixed pattern is validated on
  a node by the stateless custody experiment, whose change chains run
  the baseline realisation through both custody patterns. Remaining
  dependency: an indexer surface for contract-transaction enumeration
  (Path to Active).
- This MIP contains no implementation code; the artefacts above are its
  evidence base, and the reference implementation is an acceptance
  criterion, not an existing artefact.

## Testing

Conformance is demonstrated by a suite exercising, against a real node
(the simulator does not enforce the ledger's nullifier set and cannot
observe the failures this standard rules out):

1. **Deposit conformance**: deposit lands; public state holds no coin
   material; the inbox entry decrypts to the deposited coin; index
   capture succeeds, including a wrong-index candidate-retry probe
   that fails at proving without producing a transaction (INV-2,
   INV-4, INV-5).
2. **Witness spend**: full-amount spend from the coin store is accepted
   by the node (INV-1, INV-2).
3. **Change chain**: partial spend, re-capture of the surviving
   change, spend of the change in a later transaction (INV-3),
   exercised under the realisation the implementation uses (6.3). This
   test is the regression gate for the change rule and fails on the
   consumed-coin defect.
4. **Third-party deposit and discovery**: a second, independent wallet
   deposits against the advertised key; the owner discovers and spends
   using the inbox walk and chain data only (INV-4).
5. **Observer leak audit**: identical lifecycles through this standard
   and through the public-map pattern; a byte-scan of raw transactions
   and contract state finds coin material in the control (validating
   the scanner) and none in the conforming path (INV-2).
6. **Unshielded mirror**: deposit, withdraw, and attempted
   over-withdrawal against the mirror (INV-8), plus an unmirrored
   transfer confirming lower-bound semantics; run for Night and for at
   least one non-native unshielded color.
7. **One-hop payment**: a payment between two custody contracts routed
   per section 6.6; no transaction in the flow contains both contract
   addresses (INV-6).

`round` monotonicity (INV-7) is asserted as a post-condition of every
test above; INV-1 is exercised by running each authorised circuit once
without a satisfying witness and observing the abort.

## References (Optional)

- [MPS-0018](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md):
  Multi-key Account Custody for Midnight-Native Assets (the parent
  problem statement).
- [MIP-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md):
  Native Shielded Token Standard (spend-path discipline, disclosure
  semantics, caller-authentication prohibition).
- [MPS-0027](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0027-domain-separation.md):
  Domain Separation for Midnight Hash Constructions (tag registry).
- MPS-0006, MPS-0016, MPS-0024, MPS-0025: the institutional custody
  problem statements this surface is designed to be reusable under.
- MIP-0003: ECDSA support; MIP-0007: name service registry and resolver
  (context, not dependencies).
- Midnight ledger sources for the on-chain structures relied on here:
  Zswap input and output structures, coin commitment and nullifier
  derivations, the contract-call communication commitment, and the
  rejection of ciphertexts on contract-owned outputs.
- OpenZeppelin Compact contracts: the public-map treasury pattern and
  the stateless precedents; the change-rule
  [defect report](https://github.com/OpenZeppelin/compact-contracts/issues/656)
  and its
  [resolution](https://github.com/OpenZeppelin/compact-contracts/pull/661).
- Midnight Passport workspace: the stateless custody experiment
  (evidence and conformance probes, `experiments/stateless-shielded-custody/`)
  and the account-custody prototype
  (`experiments/account-custody-prototype/`); to be linked at their
  public locations on submission.

## Acknowledgements

The IOG Advanced Research and Creativity department, the Midnight
Foundation, and the OpenZeppelin Compact contracts team, whose treasury
patterns this standard builds on.

## Copyright Waiver

This document is licensed under the Apache License, Version 2.0, and its
contribution is made under the Midnight Foundation Contributor License
Agreement.

Portions of this document were drafted with the assistance of a large
language model. The named authors reviewed all content and are solely
accountable for it.
