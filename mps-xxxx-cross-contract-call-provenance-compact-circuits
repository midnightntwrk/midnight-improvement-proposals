---
MPS: "0040"
Title: Cross-Contract Call Provenance in Compact Circuits
Authors: Hector Bulgarini (hbulgarini), Nicolas Di Prima (NicolasDP)
Status: Proposed
Category: Core
Created: 15-Sep-2026
Requires: none
Replaces: none
MIP: none

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

<!-- Before filing upstream: invite Alejandro Pestchanker (apestchanker),
     author of MPS-0029, to review. His document is the nearest neighbour and
     the invitation should precede the filing rather than follow it. -->

## Abstract

A Compact contract invoked as the callee of a cross-contract call cannot learn
whether a contract invoked it, which contract invoked it, or which circuit of
that contract made the call. The asymmetry is sharp. The protocol already
enforces the pairing from the caller's side: `kernel.claimContractCall` commits
the callee's address, the hash of the callee's circuit, and the communication
commitment to the caller's transcript; verification refuses a transaction whose
claim matches no call in the segment, and refuses a second claim on the same
triple; and the caller's effects reach the first public input of its proof, so
no claim enters a contract's transcript except by its own circuit. Any observer can
reconstruct the edge from the transaction alone, and the ledger derives the
calling contract's address per call frame and places it in the virtual machine
context. The one party that can read none of it is the callee, whose
authorisation logic depends on the answer. A merged MIP already waits on this:
MIP-0007 specifies contract-owned names, and that arm will stay unimplementable
even once cross-contract calls reach a public network, because the registry is
the callee. This document frames provenance between contracts and takes no
position on wallet identity, which is MPS-0029's subject. Every claim is cited
in the Problem section against `ledger-9.1.0.0-rc.3` (commit `4823b53`) and
compactc 0.34.0.

## Vision

A callee decides for itself who may compose with it. A name registry accepts an
update because the account contract that owns the name called it, not because a
preimage travelled in an argument. A contract that receives value attributes the
payment to the contract that sent it. A circuit refuses to run except when
driven by a designated counterparty. Composition becomes something a contract
consents to rather than something that happens to it, and callee-side policy is
stated once, on chain, where it binds.

## Problem

**Composition shipped without provenance, and no document reserves the gap.**
CoIP-0002, the Phase 1 design, names exactly three limitations (static
implementation resolution, undefined recursion, and no witnesses in callees),
MPS-0021 scopes Phase 2 to witnesses and private state, and provenance appears
in neither, in no other CoIP, and in no upstream issue.

**The protocol already enforces the pairing, from one side only.**
`kernel.claimContractCall(addr, entry_point, comm)` records the pairing as a
transcript effect (`compiler/midnight-ledger.ss:195-210`), verification refuses
the transaction unless a matching call is present in the same segment
(`ledger/src/verify.rs:1633-1653`), and the caller's transcript effects reach
the first public input of its proof (`verify.rs:1978-1991`, `1946-1960`), so no
claim enters a contract's transcript except by its own circuit. Any observer can
reconstruct the edge after the fact (`verify.rs:1113-1200`): an indexer can say
which contract called ours; our contract cannot.

**The calling contract is computed per frame, reaches the virtual machine, and
the language cannot read it.** The specification gives the `caller` derivation
in order as the calling contract's address, then the single-distinct-owner rule
over UTXO inputs, then no caller (`spec/contracts.md:205-211`); the
implementation matches at `ContractCall::context`
(`ledger/src/structure.rs:2678-2718`); and the value lands at slot 6 of the
context array, self-describing through the `PublicAddress` discriminant
(`onchain-runtime/src/context.rs:853-894`, the caller slot at `886-889`;
`coin-structure/src/coin.rs:719-722` for the type and `799-806` for the
aligned-value encoding, whose leading boolean is the discriminant). The Compact
Kernel ADT declares sixteen operations and exactly one identity read, `self`
(`compiler/midnight-ledger.ss:159`, `256`), and `kernel.caller` is rejected by
0.33.0-rc.2, 0.34.0-rc.0, and 0.34.0 alike with "operation caller undefined for
ledger field type Kernel".

**The calling circuit is recorded nowhere at all.** `ClaimedContractCallsValue`
is `(u64, ContractAddress, HashOutput, Fr)`, whose `HashOutput` is the callee's
entry-point hash, recorded by the caller
(`onchain-runtime/src/context.rs:578`, predicate at
`ledger/src/structure.rs:2739-2744`); `CallContext` has no entry-point field.
No platform surveyed exposes the calling function either: Aztec's
`function_selector` and Solidity's `msg.sig` are each the callee's own, Aleo's
`self.caller` names a program, and Solana's instructions sysvar excludes the
inner instructions that cross-program invocation creates. One mitigating fact
is worth stating: the caller derivation binds the caller's whole `ContractCall`
and keeps only its address (`ledger/src/structure.rs:2685-2693`), so the
caller's `entry_point` is in hand where `CallContext` is built and is simply
not carried.

**The substitutes do not substitute.** Authority travelling in arguments is
thirty-two caller-chosen bytes: a `ContractAddress` is constructible in-circuit
from arbitrary bytes, and an impostor contract forwarding a stored address
compiles cleanly on 0.34.0. A caller cannot authenticate itself either: a
contract's only secrets live in witnesses, a contract that is itself a callee
has none at run time ("calls to witnesses in non-root contracts are not yet
supported"), and a secret a root caller does hold must be disclosed to cross
the boundary, after which it authenticates nobody. A callback attestation is
refused twice by the toolchain, by the compact-runtime re-entrancy guard, on by
default (`compact-runtime 0.19.0`, `dist/contract.js:292` and `362`,
`dist/circuit-context.d.ts:151-159`), and by the rejection of a non-forest call
graph at transaction construction (`ledger/src/construct.rs:1049-1076`). A
capability coin proves which contract minted a color, not which contract is
calling. Move and Sui omit a caller primitive deliberately, answering with
witness types and capability objects, and that answer does not port: Compact's
contract types are structural, the implementation resolved from the caller's
own application context (CoIP-0002, limitation 1) rather than authenticated at
publish time as Move's verifier does, so a callee cannot require that a value
reaching it was constructed by a designated contract, and every
witness-derived value forwarded across the boundary is a hard `disclose()`
error under 0.34.0 where the wrapper is omitted. What survives is an
owner-signed bundle verified inside the callee, which authenticates the owner
rather than the caller, and so cannot express the converse rule: this operation
only when driven by contract C.

**Whatever authority rests on must be unforgeable, and the one residual is lending.**
The User arm rests on verified ownership of unshielded inputs; the Contract arm
rests on a claim a contract emits, and that claim is disciplined at
verification. Alongside the subset check above, `effects_check` rejects a
second claim on the same (callee address, entry-point hash, communication
commitment) triple within a segment (`ledger/src/verify.rs:1610-1631`); it runs
on every standard transaction (`verify.rs:654`); and the specification states
both checks as assertions (`spec/intents-transactions.md:600`, `611`). With the
caller's effects reaching the first public input of its proof, no third party
can fabricate or duplicate a claim. What the discipline does not prevent is
voluntary lending, and we have now observed it: `kernel.claimContractCall`
takes wholly chosen arguments from Compact source, and on node 2.1.0 with
`ledger-9.1.0.0-rc.3` a contract exporting such a circuit claimed a root call
made by the client, which no circuit of the claimant had made; the transaction
was admitted (`SucceedEntirely`), and the ledger's own `ContractCall::context`,
run over the admitted bytes off-node, resolves that call's caller to the
claimant (`find_map` over the intent's actions,
`ledger/src/structure.rs:2685-2693`) and the claimant's own frame to no caller.
The three disciplines were refused at node admission by name, as
`RealCallsSubsetCheckFailure`, `ClaimedCallsUniquenessFailure`, and
`CallSequencingViolation`, the last of which forces a claimant to precede the
call it claims; and over refused bytes carrying two claims on one call, the
derivation taken alone prefers the earlier claimant to the contract that
genuinely made the call, so the arm is lendable but, on the checks exercised,
not overridable. A value read from slot 6 would therefore mean the contract
whose transcript claims this call, which need not be a contract that called
anything. No upstream test exercises the arm: every context in
`ledger/tests/composable.rs` is built with `QueryContext::new`, whose derived
default gives no caller. The arm is, as far as we can establish, specified,
implemented, unexercised upstream, and observed by us in the derivation over
admitted bytes, though never in a callee reading the value, which no released
Compact can express.

### Non-goals

- Wallet or externally held identity and the unshielded-input-owner derivation,
  any shielded equivalent, and the security status of `ownPublicKey()` with its
  deprecation and diagnostics. All of that is MPS-0029.
- Transaction origin as a value distinct from the immediate caller; goal 6
  states the naming discipline that applies if one is ever offered.
- Re-entrancy control: provenance is an identity check, not a concurrency
  control.
- Delegated authority. Capability and signature seams remain the right answer,
  as Aztec shows by needing AuthWit despite having an authenticated
  `msg_sender`.
- Witnesses and private state across the call boundary (MPS-0021), and dynamic
  selection of a callee implementation (the open CoIP draft, PR #628).
- The design of any primitive: no mechanism, type, or namespace is chosen here.

## Relationship to adjacent work

**MPS-0029.** Both documents point at the same virtual machine context slot.
Nothing here contests MPS-0029's account of `ownPublicKey()` or its request for
a caller primitive, and this document depends on neither, which is why
`Requires` is `none`. The factual position is the specification's own: the
`caller` derivation is ordered, with the calling contract's address first and
the UTXO-owner rule second (`spec/contracts.md:205-211`;
`ledger/src/structure.rs:2678-2718`). MPS-0029's Goals scope the requested
return value to the unshielded-input case and the no-caller case, and its
Recommended MIP asks for a value "derived from verified unshielded UTXO input
ownership", so a MIP written faithfully to those Goals leaves every goal below
unmet, which is why the cross-contract requirement is written down before any
such MIP is specified. Only one primitive should contend for the slot: a MIP
answering MPS-0029 that exposes it with full `PublicAddress` semantics,
Contract arm included, discharges goals 1 and 2 here. This is a separate
document rather than a comment because MPS-0001 holds that a document at
Proposed takes no further substantive modification, and because MPS-0029 has no
Discussion thread, its originating PR (#213) having merged on 2026/07/14.

**MPS-0021.** Its Expected Outcomes now include cryptographic binding of caller
and callee proofs against replay or recombination. That is the closest upstream
text to this problem, and it is a different property: binding two proofs is not
the same as letting the callee read who the caller is.

**CoIP-0002, the dynamic-selection CoIP draft, and the venue.** Phase 1 lives
in the Compact repository rather than the MIPs repository, so a companion CoIP
may be the correct instrument for the language surface, with the MPS and its
MIP carrying the protocol semantics. The draft CoIP on dynamic selection of a
callee implementation (`LFDT-Minokawa/compact` PR #628, open, filed as
`coips/coip-xxxx.md`) strengthens the case: a callee resolved dynamically has
more reason to want to know who is calling it.

## Use Cases

### UC1: Contract-owned names

MIP-0007, under "Forward-looking authorization arms", specifies arm 1
normatively: an owner record "MAY designate a contract address instead of a
commitment", and an owner-gated circuit then accepts "a call when it is made by
the designated contract through a cross-contract call". The arm's availability
condition is network availability of the capability: "implementations MUST NOT
accept contract owners before the capability is available on the target
network". That condition names the wrong capability. Cross-contract calls have
shipped in the stable Compact toolchain, and we have exercised them end to end
on a ledger-9 localnet, although no public Midnight network runs that ledger
yet. On whichever network the capability does reach, the arm stays
unimplementable, because the registry is the callee and cannot identify its
caller. The cost of the workaround is that a name held behind a designated key
sits outside the account's own multi-key revocation and recovery policy.

### UC2: Callee-side composition policy

A callee cannot express that an operation may be driven only by a designated
counterparty, nor that a circuit may run only as the transaction root rather
than as a callee, even though composition is now live. NEAR's instruction to
mark callbacks private "so it can only be called by the contract itself" is the
same class of rule on a platform whose callbacks arrive as separate
transactions; the Midnight toolchain refuses re-entrant calls, so the self-call
form of it does not arise here, and the two forms above are the ones blocked.

### UC3: Attributing received value

A value-accepting callee credits a public mirror with no caller check of any
kind, so escrow and settlement flows cannot bind received value to its payer. A
claim circuit can be gated on an owner-signed bundle, as the account's own seam
is, but not on the identity of the contract that sent the value, which is the
gate these flows need.

### UC4: On-chain enforcement of dApp and wallet policy

Wallet-side evaluation is today the only place the "who is asking" question
can be answered, so no dApp-connection request shape or policy vocabulary may
assume a callee can verify who called it, and any rule the contract should
also enforce is written twice, with a compromised client as the sole gate on
the first copy. Our scoped-grants draft records the constraint directly: "a
separate verifier contract would need the caller identity MPS-0029 records
Compact lacks".

Two candidate cases are explicitly not blocked by this gap, and saying so is
part of the claim: account recovery is authorised behind the account's own seam
with no caller dependency, and Dust fee sponsorship is wallet-level rather than
a contract paymaster. Goal 3 asks for the same candour: UC1, UC3, and UC4 need
only goals 1 and 2, and goal 3 earns its place by tightening UC2 from a
designated counterparty to a designated operation of one, its only consumer in
our corpus.

## Goals

1. A callee can determine whether its invocation came from another contract or
   not, with the two cases unambiguously distinguishable.
2. A callee can learn the `ContractAddress` of its immediate caller, and of the
   immediate caller only.
3. A callee can learn which circuit of the calling contract made the call, in
   whatever identifier form the protocol already uses for entry points.
4. Whatever answers goals 1 to 3 is unforgeable by the caller, by any other
   contract in the same intent, and by the submitting client, and is never
   witnessed. Aleo shows this is achievable without recursion, by making the
   caller a public input the verifier recomputes from the call graph.
5. The semantics of every arm the derivation can produce are specified,
   including the absent case: the unshielded fallback consults only the
   intent's unshielded offers, so a Dust-only fee path, or one with disagreeing
   input owners, yields no caller.
6. Any origin-shaped value, if offered at all, is never interchangeable in
   appearance with the immediate caller. NEAR's `predecessor_account_id`
   against `signer_account_id` is the positive model; Solidity's "Never use
   `tx.origin` for authorization" is the negative one.
7. The mechanism does not encourage "is this an externally owned account"
   reasoning. A Passport account is a contract, and ERC-4337 and EIP-7702
   record what such checks cost elsewhere.
8. It requires no proof recursion, which Compact does not have (MPS-0014), and
   adds no observer-visible information, which is achievable because the
   caller-callee edge is already public to an indexer.
9. The failure surface when a callee refuses an invocation on provenance
   grounds is specified rather than left to implementations.

## Expected Outcomes

Callee-side authorisation becomes expressible. MIP-0007's contract-owner arm
becomes implementable once cross-contract calls reach a public network, and
contract-based accounts become first-class name owners and counterparties
rather than names held behind a designated key. Value-accepting contracts
attribute payments to their payer. Policy written twice today, in the wallet
and in the circuit, is written once and enforced where it binds. The ergonomic
delta is plain: a context read replaces a signature verification and its
argument bytes at every composed seam, and those bytes, being call arguments,
are formally disclosed under 0.34.0, while a context read need not appear in an
argument at all; whether such a value is itself disclosed when a circuit
branches on it is open question 5.

## Open Questions

1. **Enforcement surface.** Is a context read of the caller refused at node
   admission with a `ReadMismatch`, in the manner of a transcript read
   (`onchain-vm/src/result_mode.rs:47-59`), so that a provenance assertion
   fails at admission rather than failing to prove? We measured that for
   `kernel.blockTimeLessThan` on node 2.1.0, and the claim-discipline
   failures above likewise surfaced at admission rather than at proving, but
   neither is a context read, and the extrapolation is ours.
2. **Cost and layering.** The context array is already declared extensible "in
   a minor version increment" (`spec/onchain-runtime.md:205-207`). Do goals 1
   and 2 therefore need only a language surface, while goal 3 needs new wire
   data and so a ledger schema change with redeploy consequences? And should
   goal 3 name the entry-point hash the ledger already matches on, or a
   contract-operation identity carrying verifier-key version, the two differing
   under contract maintenance?
3. **Consent and binding.** A callee has no say today in who names it, and a
   claimant need not have called at all: a claim is caller-side consent to be
   named, indistinguishable on chain from a call the claimant's circuit made.
   Should the claim be bound to the runtime's own call trace, so that only a
   contract that called may be named, or is consent the intended semantics?
   And if provenance is exposed, is caller-side suppression (Aztec permits a
   null sender for private-to-public calls, the callee deciding whether null
   is acceptable) worth the interface complexity where the edge is already
   public?
4. **Cross-intent composition.** The matcher scans only the containing intent's
   actions, and the uniqueness and subset checks are likewise keyed per
   segment, so what is the caller value for a call composed across intents of
   one transaction?
5. **In-circuit disclosure.** Under 0.34.0's `disclose()` discipline, what is
   the status of a provenance value branched on inside a circuit, and can
   callee-side policy itself be private? We did not test this.

## Recommended MIPs

- **Cross-Contract Call Provenance in Compact.** Specify what a callee may read
  about its own invocation, covering goals 1 to 3, the unforgeability
  obligation in goal 4, the arm-by-arm semantics in goal 5, the naming
  discipline in goal 6, the failure surface in goal 9, and conformance
  vectors. Upstream already carries an acceptance harness: the end-to-end
  corpus holds a deliberately-red `kernel.caller()` dapp whose cross-contract
  scenario asserts that a proxy's forwarded call is observed as the proxy's own
  contract address. This is the keystone the others hang from. If a MIP
  answering MPS-0029 exposes context slot 6 with full `PublicAddress`
  semantics including the Contract arm, it discharges goals 1 and 2 and this
  MIP narrows to goals 3 and 4.
- **MIP-0007 amendment: activate the contract-owned-names arm** *(follow-on)*.
  Re-state the availability condition for arm 1 against the capability it
  actually needs, and specify the owner check for a contract owner, so that
  contract-based accounts can own names under their own authorisation policy.

## References

- **Our evidence:** the cross-contract-calls experiment, ten probes on a
  ledger-9 localnet including atomic failure, unshielded value movement, and
  shielded value movement (P0 to P7, 2026/09/03), caller derivation (P8,
  2026/09/14), and voluntary lending with its three controls (P9,
  2026/09/14), produced by Midnight Passport (ARC) at `midnightntwrk/passport`,
  `experiments/cross-contract-calls/`. Pins: compactc 0.34.0, language 0.26.0,
  compact-runtime 0.19.0, `ledger-9.1.0.0-rc.3`, node `2.1.0`, indexer
  `4.4.0-rc.2`, proof server `9.0.0-rc.6`, midnight-js `5.0.0-beta.7`. P8 runs
  the ledger's own `ContractCall::context` (`rust/caller-context/`, built from
  `midnightntwrk/midnight-ledger` at the same git tag) over our transaction
  bytes off-node, including bytes byte-identical to a finalised block; P9's
  refusals are the node's, named in its log and tied to each captured arm by
  transaction hash. Compiler probes, 2026/09/14, reconfirming that
  `kernel.caller` is rejected on 0.33.0-rc.2, 0.34.0-rc.0, and 0.34.0, and that
  both an impostor contract forwarding a stored address and a contract emitting
  `kernel.claimContractCall` with chosen arguments compile. No circuit of ours
  reads context slot 6, so no node has been observed evaluating such a read.
  One ARC working document is quoted in UC4 and is not a
  published standard: our scoped-grants MIP draft (`midnightntwrk/passport`
  PR #154, open; the quoted sentence is requirement R1).
- **Ledger sources,** `midnightntwrk/midnight-ledger` at tag
  `ledger-9.1.0.0-rc.3` (commit `4823b53`): `spec/contracts.md:184-282` (the
  ordered caller derivation in prose at `205-211`, the `context()` pseudocode
  at `226-261`, and `calls_with_seq` at `267-280`);
  `spec/intents-transactions.md:594-611`; `spec/onchain-runtime.md:205-207`;
  `ledger/src/structure.rs:2678-2718` (`ContractCall::context`, the caller
  search at `2685-2693`) and `2724-2747` (`calls_with_seq`, the matching
  predicate at `2742-2744`); `onchain-runtime/src/context.rs:308-318`, `578`,
  and `853-894`; `coin-structure/src/coin.rs:719-722` and `799-806`;
  `ledger/src/verify.rs:654`, `1113-1200` (`relate_nodes`), `1431`
  (`effects_check`) with the uniqueness check at `1610-1631` and the subset
  check at `1633-1653`, `1946-1960` (`public_inputs`), and `1962-2004`
  (`binding_input`, the transcript effects serialised at `1978` and `1988`);
  `ledger/src/construct.rs:1049-1076`; `onchain-vm/src/result_mode.rs:47-59`.
- **Language sources,** `LFDT-Minokawa/compact` at branch `main`, fetched
  2026/09/14: `compiler/midnight-ledger.ss` (blob
  `acf07ec2a46b7557610197207ebbd11ab60b3cf4`), the Kernel ADT at `159`,
  `claimContractCall` at `195-210`, and `self` at `256`; `coips/coip-0002.md`;
  the dynamic-selection CoIP draft as PR #628; and the compactc 0.34.0 release
  note (`doc/release-notes/toolchain-0.34.0.md`), which states that
  `ownPublicKey()` "always names the transaction submitter, never the calling
  contract". The acceptance harness is upstream's own:
  `midnightntwrk/compact-end-2-end` at commit `b3a0504` (2026/08/21),
  `dapps/caller/contracts/Caller.compact:23-27` (the `capture()` circuit
  reading `kernel.caller()`), `dapps/caller/contracts/Proxy.compact:14-26` (the
  cross-calling proxy), `dapps/caller/src/main.ts:158-190` (the scenario
  asserting the proxy's own contract address), and `dapps/caller/README.md`,
  section Current compile gap, which records the state as expected red. We
  compiled `Caller.compact` against three toolchains; we did not run the
  scenario.
- **Related documents:** MPS-0029 (caller identity access in Compact circuits),
  MPS-0021 (Phase 2, contract to contract), MPS-0014 (proof verification and
  recursion), MPS-0001 (the MPS process), and MIP-0007 arm 1 (contract-owned
  names).
- **Prior art:** NEAR `predecessor_account_id` against `signer_account_id`, with
  its guidance on private callbacks; Aleo (`snarkVM`), where the caller is a
  public input the verifier recomputes from the call graph and
  `transfer_public_as_signer` is separately named; Aztec, whose `CallContext`
  carries `msg_sender` and the callee's own `function_selector`, whose private
  kernel is recursive, and whose AuthWit remains necessary for delegated
  authority; Solana, whose instructions sysvar excludes inner instructions;
  Starknet `get_caller_address`; CosmWasm `MessageInfo`; the Move book on the
  witness and capability patterns; EIP-7, ERC-2771, ERC-4337, and EIP-7702;
  Solidity's security considerations on `tx.origin` and on re-entrancy; and
  Hardy (1988) on the confused deputy.

## Acknowledgements

Alejandro Pestchanker, whose MPS-0029 diagnosed the `ownPublicKey()` hazard and
located the caller slot; Jonathan Sobel and Karmel E for the CoIP-0002 and
MPS-0021 design record, against which this document positions itself; and the
IOG ARC department, whose reviewers read the cross-contract experiment this
document rests on.

## Copyright

This MPS is licensed under CC-BY-4.0.
