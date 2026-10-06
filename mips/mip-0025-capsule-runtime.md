---
MIP: "0025"
Title: Managed Private State and Capsule Runtime
Authors:
  - Andrzej Kopeć (kapke)
  - Jonathan Sobel (jonathan-sobel)
Status: Draft
Category: Core
Created: 2026-08-19
Requires: none
Replaces: none
MPS: MPS-0021
License: Apache-2.0
---

<!-- Copyright 2026 Midnight Foundation

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in
compliance with the License. You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is
distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied. See the License for the specific language governing permissions and limitations under the
License. -->

## Abstract

With the Compact compiler and runtime available at the launch of Midnight Mainnet, Compact programs
say almost nothing about private state.  All definition, evolution, protection, and recovery of
private state is defined in the applications that call each contract, making it impossible for the
platform to limit the leakage of private data across contract boundaries.  This is a proposal
describing changes to Compact and the local contract execution platform to enable well-reasoned
management of private state in systems of multiple contracts.

The proposal describes
- managed private state evolution within contracts
- controlled access to the execution context from contracts
- controlled access to contract state from applications
- cross-contract calls among contracts with the above capabilities
- wallet participation in the initialization and preservation of private state.

The local execution of contracts no longer occurs within the context of the calling application, but
in a specialized *capsule runtime* that encapsulates a local representation of each relevant
contract's state.

## Motivation

Like many blockchains, Midnight allows portions of its public, shared ledger state to be governed by
so-called *smart contracts*, machine-executable rules limiting the ledger's state transitions.
Unlike most other blockchains, Midnight allows these rules to depend on private information.  There
are many ways that a system might keep certain information private, such as encrypting secrets with
a key known only to the owner.  Midnight uses the strongest mechanism of all: private data is never
required to leave the local system of the actor initiating a public state change.  Midnight's
zero-knowledge proof system guarantees that all changes to the public ledger respect the rules
imposed by a smart contract, even when the rules depend on the local, unshared data.

> An aside on terminology: *Private* state describes the intent that some of a transaction's state
> should remain unknown to parties other than the initiating actor.  *Local* state describes the
> location of the state.  Midnight achieves *privacy* through *locality*.  The designer of a smart
> contract preserves the boundary between private and shared/public information by holding some data
> in local state fields and other data in ledger state fields.  When this document focuses on the
> property of information being unknown to other parties, it refers to *private* state.  When
> discussing the programming model and mechanisms, however, it refers to *local* state.

### In What Sense Does Locality Imply Privacy?

The preceding description makes a dramatic oversimplification.  It equates maintaining a user's
privacy with keeping data on the user's local system.  If "the user's local system" means a whole
device (such as a phone or a laptop), then keeping the information "local" is not enough to preserve
the user's privacy.  In general, users do not want every piece of software that is running on their
local device to have access to every piece of information on that device.  For example, if a user
has an application that works with medical records or citizenship data, the presence of such data on
the user's computer should not imply that a different application on the same computer, such as a
securities trading application, should have full access to all the medical or citizenship data.  The
user expects the underlying operating system of the device (or the web browser, or whatever
environment hosts the data and the applications) to preserve secure boundaries between the various
applications and their data.

The goal of this improvement proposal is to deliver the same kind of secure boundaries among
Midnight applications and the local data associated with Midnight smart contracts.

The situation becomes more complicated when Midnight contracts can depend on one another.  For
example, suppose that the kinds of applications described above are implemented using Midnight smart
contracts.  One set of contracts could impose rules over the legal certification of birth and
citizenship data, with most of the details behind that certification remaining private.  Another set
of contracts could support securities trading, depending on the first set of contracts to certify
that the user is old enough and in the right legal jurisdiction to participate.  The securities
contracts should be able to call on the legal certification contracts to establish the acceptability
of the participant, without gaining access to all the private birth and citizenship data behind the
certification, even when a trading contract calls on a certification contact on behalf the trading
application.

More generally, allowing an application to exercise the public interface of a smart contract should
not imply that the application has been granted any access at all to private state associated with
the contract, even though the state is local to the same system as the application.  The same is
true when one contract calls another: the local state of each contract is "more local" than the
hosting web browser or computer.  A contract's local state should be truly private to that contract
and to an identified user account that owns the data.

### Motivating Examples

**Certified verification of age, legal status, or location:** The first type of scenario that
motivates this proposal is the one just described.  Some DApps require their users to be a certain
age or reside in certain jurisdictions.  Many financial applications are in this category.  A
government or trusted authority issues a certificate to confirm the legitimacy of a set of facts
about a user, and a smart contract guarantees that certain assertions about the facts can be made
only in the presence of the certificate.  The whole certification system probably has its own DApp.
The DApps that rely on such certification do not interact directly with the certification DApp;
instead, their smart contracts call the certification contract, but without direct access to the
backing data.  One of the major reasons that users demand partitioning of the private data in this
use case is because the levels of trust for the different DApps may be quite different.  For
example, suppose a user wants to experiment with a new trading DApp, but the user is not quite sure
whether to trust the new DApp yet.  Maybe it comes from an obscure source.  The not-yet-trusted DApp
should definitely not gain access to secrets about the user's birth and residency.

This first scenario puts useful design pressure on the different paths through which local state can
be initialized.  For many kinds of local state, there is an obvious "empty" starting state, which
can synthesized as necessary without user interaction.  There is no way to synthesize certified
personal information, though, which implies that some kinds of cross-contract interactions require
the ability to signal failure when a called contract demands prior initialization.

**Decentralized exchange (DEX) for tokens:** A second motivating scenario for this proposal is an
"open" DEX, in which new tokens can be registered for trading without requiring the DEX application
itself to be rewritten or updated.  Each type of token may have a governing contract, and someone
who holds a token of that type may have private state which records details about their holdings.
Neither the DEX, nor any specific liquidity pool for swapping tokens, nor any other token type
should have access to that private state.  Concretely, assume that token types *A* and *B* have been
registered with the DEX, and some party has enabled swaps between them by funding an *A-B* liquidity
pool.  Some user holds tokens of type *A* and wants to swap them for some of *B*.  The DEX DApp has
no code specific to *A*, *B*, or the *A-B* liquidity pool.  The user has never held *B* before and,
so far, has no private state associated with *B*'s contract.

This second scenario puts design pressure on the proposed solution, even beyond the requirement for
isolation among the private state associated with the DEX, the contracts for *A* and *B*, and the
liquidity pool.  Recall that Midnight contract execution begins with a user rehearsing each
transaction locally to produce a transcript of the local state updates and a transcript of the
ledger state updates.  A zero-knowledge proof binds the updates and confirms to the blockchain that,
if the preconditions of the rehearsal are met, the update to the public state can proceed.

In the case of the complex interaction among all the contracts of this scenario, the user needs to
run the contract code for all the contracts locally, *including code manipulating private state*, in
order to generate the transcripts and then the proofs.  When the user requests the trade, though,
the DEX DApp may have only its own contracts available.  It is necessary, therefore, to propose some
means for all the necessary contract code to be found, downloaded, and run safely.  Additionally,
any private state required by each contract must be initialized and protected.

### Status at Midnight's Initial Delivery

The combination of Midnight and its Compact smart contract language, as initially delivered, are
unable to satisfy the requirements just described.  Instead, the Compact language provides a
facility for declaring functions which can be called from contracts, but whose definitions are
provided by the surrounding application.  Those functions are responsible for defining their own
representations of local state and maintaining it as the application developer sees fit.  The
structure and evolution of local state is entirely opaque to the contract.

By placing the entire burden of local state design, management, and storage for each contract on the
calling application, the Compact implementation loses any ability to isolate the local state of one
contract from another and to hide any aspect of private state from the application itself.
Furthermore, in cases like the DEX example, where the application has no prior knowledge of some
contracts and must load them dynamically, requiring the application to provide the local-state
management code is nonsensical.

The lack of structured local state management in the initial versions of Compact leads to two
additional problems.  First, if transactions are reordered or only partially completed, which can
happen, the evolution of private state must respect the blockchain's version of reality.  This means
that the transaction rehearsals which enabled their submission to Midnight cannot be treated as the
"true" sequence of events.  Instead, the rehearsals must produce private transcripts that can be
applied to local state to evolve it, in the same way that the public transcripts are applied to the
shared ledger state.  In order to generate such transcripts, the shape of and actions on local state
must be well-defined, not possible in a system which leaves all maintenance of local state to
arbitrary application code.

Second, because the initial versions of Compact provided only a single call-back mechanism (the
"witness" function declaration), that mechanism is used to evolve private state *and* to access
necessary information from the contract's context.  For example, if the contract needs access to the
current date or some kind of random seed, it calls the same kind of function that it calls to update
private state.  The two kinds of interaction—accessing its execution context and evolving its
private state—are really quite different, though.  In fact, in the academic research on which
Midnight is based, the two are represented as distinct classes of information.  Without recovering
this distinction, the Compact runtime cannot support the user's need for private state
initialization, backup, recovery, and safe replay.

### Custom Spend Logic

This document does not propose a mechanism for "custom spend logic," in which attempting to spend a
token triggers the automatic execution of rules associated with that token type.  It should be
recognized, however, that the application doing the spending may not have included the code that
expresses the spending rules and must therefore discover and execute formerly unknown code.  This
has much in common with the DEX scenario above.  Thus, the contents of the present proposal may be
prerequisite to implementing custom spend logic.

## Specification

Any good solution to the problems described in the preceding section must satisfy the several
criteria.

1. Private state must be isolated per deployed contract instance.
2. Because the same computer might be used for multiple logical purposes, private state must be
   isolated per logical "account" or "profile".  Together with the preceding criterion, this implies
   that private state storage is indexed by the pair of contract address and account/profile
   identifier.
3. Private state must be encapsulated, with controlled, permissioned access from calling
   applications.
4. The shape and evolution rules for the private state of a contract must be defined explicitly, not
   hidden in call-backs to the application code.
5. Access to the non-deterministic context of a contract's execution (for initial values,
   randomness, time, and so on) must be controlled and well-defined.
6. Users must be given the opportunity to accept or reject access to each instance of private state
   by any application; access is never assumed to be allowed, and applications are never assumed to
   be trusted.
7. There must be some way of preserving and recovering private state, so that it exists beyond the
   confines of any one web browser or other hosting environment.
8. It must be possible to resolve a contract address to valid artifacts for local execution of the
   contract, and then to execute the contract, without those artifacts having been included with the
   calling application.
9. The evolution of local state must properly respect the associated evolution of ledger state.  The
   local state updates must be applied in the same order and with the same completeness or
   incompleteness as the on-chain transactions to which they belong.  The system should be able to
   roll back and replay local state updates as needed to maintain consistency.
10. All the preceding criteria must be satisfied in the presence of upgrades to contracts and/or
    applications.  For example, if a contract is upgraded without changing its interface, so that
    its address resolves to new artifacts for local execution, the calling application should not
    require an upgrade.  The situation should be detected, and the new artifacts should be
    downloaded automatically, just as they were for the initial use.
11. All the preceding criteria must be satisfied across multiple levels of cross-contract calls, in
    which one contract calls another without an external application interposed between them.
12. All the preceding criteria must be satisfied when every application and contract is authored by
    a different party, none of them have trust relationships with each other, and none is assumed in
    advance to be trusted by the end user.

This section describes a proposed design which satisfies these criteria.  The design has four parts,
each described in its own subsection:
1. the capsule and the trust boundary it draws, and the governed on-chain reference to a contract's
   canonical capsule executable, with how that executable is resolved;
2. what a DApp may ask of a capsule, what the user grants it, and how the contract authenticates the
   user;
3. the execution model: local and capsule API functions authored in Compact, foreign functions
   runtime-provided;
4. local-state persistence and correctness under transaction failure and chain reorganization.

In many places, this proposal describes the properties of the subsystems, rather than the mechanisms
used to implement them.  Where the mechanism requires a further decision, the open gap in the design
is identified in the text.

### Capsules, Capsule Identity, and the Governed On-Chain Reference

A central idea in this proposal is the *capsule*.  A capsule is the encapsulation and trust boundary
for executing a contract's operations locally. The *capsule runtime* is the newly introduced
component that creates, manages, and protects capsules. When an application calls one of the public
functions (circuits) of a deployed contract, the capsule runtime ensures that a capsule is
instantiated for that specific contract instance and for the *account* that the user chooses to
associate with that execution of the application.

Within any concrete hosting environment, possibly linked with a wallet, the capsule runtime owns
every capsule's life cycle and its storage of local state.  The capsule runtime also mediates all
access from the DApp and to/from the wallet.  The actual execution of a contract's circuits (that
is, the local rehearsal with access to private state) occurs entirely within the corresponding
capsule, with the capsule runtime guaranteeing that nothing outside the capsule can see its local
state directly.  Conversely, the contract-defined operations that execute inside the capsule have no
direct access to outside information; all operations are either deterministic (for any given initial
ledger and local state) or are explicit calls to previously defined procedures that provide
information from the surrounding context.

The precise technical mechanisms that implement the capsule runtime and make its services available
to the DApp environment and connect it to a wallet are unspecified in this proposal.  Only its
necessary properties are defined here.  The intent that is multiple implementations of compliant
capsule runtimes should be possible, written in different programming languages and running in
different hosting environments.

The identity of a capsule is defined by
1. the on-chain address of the deployed contract it represents and
2. an account.

Nothing else defines the identity of a capsule.  Changing the capsule executable never changes
capsule identity, so an upgrade does not discard the user's local state. An account is a concept
much like a profile in an operating system or a browser.  It is created by the capsule runtime,
which assigns the number identifying it, and named by the user, who chooses the label and may change
it at any time without changing any capsule's identity. Account labels never cross the capsule
boundary: a capsule cannot observe the label of the account it belongs to. Accounts are what make
several capsules for the same contract possible: under different accounts one user holds separate,
mutually isolated local states for that contract (e.g. to separate personal and work
operations). 

Some representation of a compiled contract must governed by an on-chain reference.  The details of
what this means are not fully specified in this proposal, because several possibilities remain open.
For example, the fully-compiled executable code for the contract could be stored directly on chain
and downloaded by the capsule runtime (along with other public information about the contract) when
instantiating a capsule.  Alternatively, some intermediate representation of the compiled contract
could be stored on chain, and the runtime could interpret this representation or further compile it
into efficient local code.  Still another possibility is that the on-chain reference is a signature
for artifacts produced by compiling a contract, but the storage for those artifacts is elsewhere.

In any case, every capsule for one contract address resolves to the same governed on-chain
reference.  In case of a contract upgrade, causing the reference to provide a new contract
executable, the capsule crosses that version boundary at its next call while none of its
transactions is still in flight.  At that point, the capsule runtime retains a snapshot of the
capsule's final state under the outgoing version and runs the next call using the new version. A
capsule on a switched-off device, or one with a transaction still to settle, stays on the version it
is on until its next use, so two capsules of one contract may be executing using different versions
at the same time.

A core assumption of this design is that a given local state is governed by exactly one capsule
executable version at a time. Whether that version needs more than one artifact—such as a second
encoding for mobile platforms—is not settled here; it depends on the aforementioned details of the
executable format. This *capsule executable encoding decision* is discussed the section describing
[the execution model](#the-execution-model-local-context-and-introspective-functions).

The relationships among a capsule's identity, the governed reference to a contract's compiled form,
and exactly which version is current in use by a capsule are expressed below in SQL DDL, along with
their invariants. (This is not meant to imply that the capsule runtime would be implemented with a
SQL database. The DDL is simply intended to make the relationships and constraints explicit and
checkable.)

The DDL is not expressive enough to capture two important invariants, so they are stated here
instead.

1. **A capsule runs the executable identified by its own contract reference**, never one reached
   through another contract's reference, even in the context of cross-contract calls.  Its own
   reference is always the currently canonical executable, or, where this capsule's adoption up an
   upgraded contract lags behind the on-chain reference, the one it last adopted. No key below
   carries it, because capsule executables are deduplicated globally: identical bytes are identical
   executables, and every contract deploying that code—such as all the instances of a DEX's
   liquidity pool contracts—resolves to that same code, so no executable belongs to any one
   deployed contract instance.
2. **Application of local-state transcripts always resumes from the snapshot retained at the
   capsule's latest version boundary**, and from genesis only until it has crossed such a
   boundary—which is what keeps a transcript produced by one version from being applied in the
   context of another version. (Successive application of local transcripts is how a capsule's local
   state is reconstructed: the runtime replays the transcript of each call—the record of what that
   call did—in the order actually witnessed on chain.)  The snapshots have no representation in the
   DDL below, and their absence is deliberate: "latest" needs an ordering of the capsule's own
   boundaries, and nothing modeled below orders anything. Keying a snapshot by capsule and
   executable would instead assert that a capsule never returns to an executable it ran before;
   keying it by capsule alone would assert that only the newest snapshot is ever retained; a table
   carrying neither key would carry no invariant at all. It is the same missing ordering that leaves
   forward-only movement of the reference an open point below.  A later section about
   [persistence](#persistence-and-correctness) discussed this in more detail.

Furthermore, the DDL does not express the distinction of **whether anything has initialized a
capsule's local state.** A capsule can exist while nothing has yet initialized its state, and no key
in the data model below distinguishes between the initialized and uninitialized states.

```sql
-- Not a storage schema: only the keys and constraints that carry the invariants.

-- Created by the capsule runtime, which assigns the number; named by the user, who
-- owns the label. The label is part of no key, so renaming changes nothing.
CREATE TABLE account (
  account_number       INTEGER NOT NULL PRIMARY KEY,
  label                TEXT NOT NULL
);

-- Content-addressed and deduplicated globally: identical bytes are one executable,
-- belonging to no contract, and any number of contracts may resolve to it.
CREATE TABLE capsule_executable (
  content_hash         TEXT NOT NULL PRIMARY KEY
);

-- The contract carries the governed on-chain reference to its canonical capsule
-- executable, and carrying it here is what makes the executable resolvable from a
-- contract address alone, with no DApp involved; it is also the only thing tying an
-- executable to a contract as such. It answers a different question from
-- capsule.governing_executable: that column is what one capsule runs, this one is what
-- the chain currently says is canonical for the contract, and any difference between the
-- two is that capsule's lag, until it crosses the version boundary. Superseding rewrites
-- this column rather than adding a row, so nothing here orders one canonical executable
-- after another. NULL is a contract with no capsule executable at all.
CREATE TABLE contract (
  address              TEXT NOT NULL PRIMARY KEY,
  canonical_executable TEXT REFERENCES capsule_executable (content_hash)
);

-- Capsule identity is (account, contract) and nothing else, and this row is where the
-- capsule's local state lives: the state has no identifier of its own, so no capsule holds
-- two and none is reachable from two capsules. The executable is part of no key, so
-- no change of executable can alter a capsule's identity. Exactly one capsule
-- executable version governs the capsule at a time -- one NOT NULL column, which may
-- lag the canonical reference until this capsule next crosses the version boundary.
-- That it is an executable this contract's own reference names is asserted in the
-- prose above; no key here can carry it. That a row exists says nothing about whether
-- anything has initialised the local state; the prose below carries that distinction.
CREATE TABLE capsule (
  account              INTEGER NOT NULL REFERENCES account (account_number),
  contract             TEXT NOT NULL REFERENCES contract (address),
  governing_executable TEXT NOT NULL REFERENCES capsule_executable (content_hash),
  PRIMARY KEY (account, contract)
);

-- One transcript per transaction per capsule: the predicted change and the
-- confirmed capture are the same row, reconciled by transaction identity. Each is
-- pinned to the executable whose code produced it, so none can be folded through
-- another version's code.
CREATE TABLE transcript (
  account              INTEGER NOT NULL,
  contract             TEXT NOT NULL,
  transaction_id       TEXT NOT NULL,
  captured_under       TEXT NOT NULL REFERENCES capsule_executable (content_hash),
  PRIMARY KEY (account, contract, transaction_id),
  FOREIGN KEY (account, contract) REFERENCES capsule (account, contract)
);
```

A contract's on-chain capsule reference names its canonical capsule executable and is governed; the
mechanism is left unspecified, but three properties are not:

- The reference is updated only through a gated, authorized action.
- A superseding write replaces the reference in place, so every capsule of that contract resolves
  to the new executable from its next version-boundary crossing.
- A superseded reference never resolves to an ungoverned fallback.

Forward-only movement is wanted and does not follow from those three. The reference as specified
carries a content hash and nothing that orders one canonical executable after another, so a capsule
that has not yet adopted an upgrade sees that the hash differs and cannot tell an upgrade from a
return to an executable that was canonical earlier.  How to impose an ordering is an open point
of this proposal; the governed write path below enforces no order of its own.

While this proposal depends on governed on-chain references to the compilation artifacts of
contracts, no such references exist on Midnight today. A governed write path does exist: ledger 9.1
gives every contract operation a metadata slot whose contents are uninterpreted by the
ledger. Whether that slot may carry the necessary reference is not yet agreed; 
[gap 3](#backwards-compatibility-assessment) describes the required conditions.

Capsule executables are resolved by content hash from a capsule executable registry—IPFS is the
familiar example—and integrity-checked on load. The registry is not a trust anchor: canonicality
comes from the chain, so it needs no authority at all. The chain is the source, and a read failure
must never resolve to "no reference"; no unverified source substitutes. A failure to fetch the
referenced executable is a storage failure or a genuinely wrong capsule reference.

Because a capsule executable comes via a chain reference rather than from a DApp, a client can
resolve and execute a contract it has never held.  In cross-contract calls, each callee brings its
own executable, rather than relying one the caller already holds.

Resolving a capsule does not install a DApp. If DApp *X* indirectly causes a call into a capsule
whose primary (direct) interface is DApp *Y*, the user does not thereby have *Y*.  The state updates
triggered by *X* are held in the appropriate capsule independent of any DApp.  If the user later
obtains *Y* (and invokes it with the same account), the state produced by *X* will be seen by *Y*.

Resolving a contract address and account to a capsule does not guarantee that the capsule holds
usable state. Whether a capsule exists and whether its local state has been initialized are separate
questions, and no key in the DDL above distinguishes them. The capsule can exist while nothing has
yet initialized its state.  (The "certified verification" example in the Motivation section raises
this possibility, when no certified identity has been established yet.)  The calling DApp must be
able to distinguish an uninitialized capsule from other types of failure. How the user is then
directed to an application that can initialize the capsule is not settled here. Capsule-carried
metadata, the capsule itself identifying where it can be initialized, might be a genuinely useful
way to address it, leaving the question with the capsule rather than with the calling DApp. This
proposal suggests that as a candidate, but it does not demand it.

Inline on-chain storage of a sufficiently small capsule executable is a possible optimization that
the resolution design neither requires nor assumes. Either way, the only potential impact on the
Midnight ledger is storing an uninterpreted payload indicating the capsule executable.

### DApp Access and In-Contract Authentication

A DApp reaches a capsule by asking the capsule runtime to connect it to a contract address, and the
runtime asks the user to approve that connection and the permissions it carries. Two kinds of action
are possible once connected: calling a contract's introspetive functions and calling the contract's
circuits.  A contract defines introspetive functions to provide read-only access to capsule data
from DApps in a controlled way, so that a DApp can, for instance, show the user their portfolio.
The exact permission model is deliberately left unspecified here. What is required is that the user
has the ability to grant or deny access.

The DApp provides the contract address, and the user provides the account for that session, together
determining the identity of the required capsule, The capsule runtime attests the calling DApp's
origin from the channel the connection arrives on (when the hosting environment is a web browser,
this would be the browser's message metadata), never from a value the DApp supplies.  Without an
attested origin, access to the capsule is not granted.

Many Midnight contracts rely on some kind of in-contract authentication, using a previously
generated secret.  Because this usage pattern is so common, this proposal requires that the capsule
runtime interact with a wallet to produce a per-capsule secret, independent of the keys holding
native tokens and derived purely from the wallet seed along a predictable path keyed by the account
and the contract address.  The capsule never sees the wallet seed itself.  Each capsule accesses its
per-capsule secret by calling a context API function. A circuit using the secret necessarily
receives it as a zero-knowledge witness, exactly as in the current model.

### The Execution Model: Local, Context, and Introspective functions

As the Motivation section discussed, the "witness functions" declared in the initially-delivered
version of Compact carried multiple overlapping responsibilities.  They served both as the mechanism
for maintaining local state and as the means of accessing nondeterministic context beyond the local
state.  These responsibilities are now proposed to be handled by two distinct kinds of functions, one
for updating local state and one for accessing context outside the capsule.  A third kind of
function provides DApps with controlled introspective access to capsule state.

One important reason for distinguishing the different kinds of functions is that, after transactions
have been rehearsed locally in one order, they can be executed in a different order on chain.
Individual transactions can also fail, partially or entirely. A call's effect on local state is
predicted in the rehearsal, and later its transaction settles, so the effects cannot simply be
applied permanently at rehearsal time.  Instead, the sequence of update operations themselves must
be retained in the form of a local-state transcript.  Upon observing the actual transaction order
and completeness from the blockchain, the matching portions of the transcripts are applied in the
observed order.  Applying the same transcript to the same input state will always yield the same
output state, but *only* if the operations are deterministic. No such guarantee is possible with the
old application-defined witness functions.  The new local-state updating operations and functions
are deterministic by construction.  Their transcripts can be re-run and re-ordered as needed.

The values returned by the context-accessing function are not known to be deterministic (and often
they are not), so the local transcripts captures *results* from the context functions, not the
operations.  Upon a need for replay, the previously captured results are used, instead of calling
the functions again.

Replay also shapes how local state and local functions are expressed: the primary way of evolving
local state is data-structure manipulation using abstract data types, whose operations can be
recorded in a transcript and applied later, including as part of a replay sequence to reconstruct
local state.  The representation of these local operations will have some characteristics in common
with Impact, the VM language used to express ledger state updates, but it will include more
operations and more expressive combinations.  Local-state evolution is not constrained in the same
ways that ledger state is.

Putting all this together, it is proposed that Compact gain the ability to declare **local** state
fields, in the same way that it already provides the ability to declare ledger state fields.  The
available types for local fields will include most or all of the ones available for ledger fields,
but the local ADTs will have an extended set of operations available, and entirely new types will
also be available.

Additionally, several kinds of functions will be able to be defined in Compact:

1. **pure** functions
   - currently known as pure circuits
   - no interaction with state or context allowed
   - calling one does not require capsule instantiation
   - fully deterministic: outputs determined entirely by inputs
   - ZK proofs can be generated for their execution, when called from a circuit
2. **circuit** functions (which could be called ledger functions for parallelism with ledger state declarations)
   - currently known as impure circuits
   - deterministic, for fixed results of context API calls
   - can interact with ledger state, local state, and context, as needed
   - prover/verifier keys can be generated for them
   - ZK proofs can be generated for their execution
3. **introspective** functions
   - can be called only from the application context or other introspective functions, not by any other function types
   - read-only access to ledger state and local state, but cannot call circuits/ledger functions
4. **local** functions
   - deterministic, for fixed results of context API calls
   - no interaction with ledger state
   - any uses of `assert` are checked at local execution time only, with no ZK proof
   - may contain computations for which ZK proofs cannot be generated

When a local function performs only read-only operations on local state, it is a **read-only local**
function.

In addition to these function types, the Compact execution environment must provides **context** API
functions.  These are implemented outside of Compact and written in the programming language(s)
supported by the capsule runtime.  Some context functions are guaranteed to be available in every
compliant capsule runtime, while others may be extensions of the standard environment.  Context
functions may be nondeterministic, but they have no access to local state, unless the caller passes
local state explicitly as argument values.  Every non-deterministic input—randomness, external data,
the clock—enters a capsule only through a context function.  Results of context API calls are
captured as values at rehearsal time in the local transcript, so that their nondeterminism does not
interfere with replay or reordering.  Context values are considered non-disclosed private
information, like local state; in the theoretical sense, they are both zero-knowledge "witnesses."

Here is a summary of the function types and how they are allowed to interact:
| | ledger state ops? | local state ops? | context API calls? | pure function calls? | circuit calls? | unproven `assert`? | unprovable computation? | local function calls? |
| ---:  | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **pure functions:** | - | - | - | YES | - | - | - | - |
| **circuits:** | YES | YES | YES | YES | YES | - | - | YES |
| **context API impls:** | read-only | - | YES | - | - | ??? | YES | - |
| **introspective functions:** | read-only | read-only | YES | YES | - | YES | YES | read-only |
| **read-only local functions:** | - | read-only | YES | YES | - | YES | YES | read-only |
| **read/write local functions:** | - | YES | YES | YES | - | YES | YES | YES |

The settlement-triggered execution point is part of the execution model in its own right: once the
chain establishes a transaction's relative order, and the runtime applies its transcript, the
capsule may act on that transaction's outcome.  Conversely, while the chain does not report the
non-occurrence of a transaction, the runtime can locally signal a capsule and cause a transcript not
to be applied to produce a final state.  Effects intended to be visible outside the capsule should
be deferred to the point of known settlement, not rendered in rehearsal.

Because introspective functions can be called only from the DApp and are required to be read-only,
they are not relevant to the evolution or preservation of capsule state.  The only way for a DApp to
trigger a change in state is to call an exported circuit function.  Communication in the other
direction, to notify a DApp about a capsule's progress, is outside the scope of this proposal.

The settlement-triggered execution point fires only on a transaction that the capsule itself
produced: a capsule runs no contract-authored code on observing another capsule's
progress. Deterministic confirmation forces that exclusion, since a local-state change triggered by
another party's transaction would belong to no call, and local state updates are applied only call
by call. Another party's transaction reaches a capsule's local state only by being public knowledge
at its next call.  It is the capsule runtime's obligation to keep public state up-to-date, including
across chain reorganization.

Introspective functions are the contract's DApp-facing query surface: read-only operations, callable
from a DApp and never from a circuit or another contract. Circuits are the other API surface and an
entirely distinct one—the operations a DApp can have executed and proven. The capsule runtimes
preserves the capsule boundary by ensuring that no local-state-derived value crosses to a DApp
except through a contract-defined introspective function to which the user has granted that DApp
access.  This consent-gated boundary governs the release of information from the capsule, but it has
no control beyond that.  What becomes of a disclosed value is analyzed in the section
[Security Considerations](#security-considerations).

Exactly how local functions and introspective functions are expressed in Compact is not specified
here.  That level of detail belongs to a dedicated CoIP.

> The need for a canonical representation of compiled contracts, including the definitions of these
> new kinds of functions, exceeds the capacity of the current zero-knowledge intermediate
> representation (ZKIR) used to convey circuit information for generating prover and verifier keys.
> MPS-0022 calls for a standard, language-agnostic representation of compiled contracts, and the
> needs of the capsule runtime may answer that call.  Such a representation is not defined here.

Two aspects of the proposed kinds of functions require their own separate proposals:
1. the calling conventions and mechanisms of context API functions, including how their
   implementation is resolved and how the API can be extended
2. the calling conventions and mechanisms of the introspective functions, including how the DApp
   binds to them, which will extend the existing DApp Connector API specification.

### Persistence and Correctness

Under this proposal, a DApp developer does not write code that directly manages local state. The
goal is that the whole local state of a contract can be reconstructed from the wallet seed phrase,
the capsule identifier (that is, its contract address and account ID), and that capsule's encrypted
transcript log including snapshots. No code written by the DApp developer takes part in that
reconstruction, and the user supplies nothing beyond those three inputs. The capsule runtime drives
the replay, re-running the contract's own local functions as it goes, so those functions dictate
what is replayable.

A solution has to cope with three things at once: concurrency, transactions that fail, and users who
migrate and restore their state on another device. Satisfying all three together is non-trivial, and
they tend to converge on the same building blocks. Changes to the local state are captured as a
replayable transcript, encrypted at rest, carrying the status of the transaction that contained the
contract call. The transcripts are stored as the source of truth, and a snapshot of the local state
is obtained by replaying them. The capsule's secret unlocks that replay and then takes part in it:
first to decrypt the transcripts, then as part of the replay context. The secret derives from the
seed phrase and the capsule identifier, and the transcripts are the material being replayed—the
three recovery inputs above, and nothing else.

Those building blocks become part of the runtime, leaving no need for DApp code to handle any of
them: current local state is directed purely by the application of the transcripts of known
transactions, in the order the chain establishes—which is why the local functions that a transcript
re-runs must be deterministic and pure. The incremental application of transcripts holds only within
one capsule executable version: a version boundary produces a retained snapshot, and no transcript
is ever applied through another version's code. Conceptually, then, the whole state can be replayed
from genesis, with version-bump markers indicating where the upgrades happened. In practice, to
avoid the risk that a past version's executable is unavailable, the steady state is
snapshot plus incremental application.

A step is the capsule's execution of one call, and its inputs are: 
1. the prior local state, 
2. the call's inputs, 
3. the ledger state which the step reads, and 
4. the captured results of the context API functions which the step invoked. 

All but the prior local state are fixed when the call is prepared and captured in its transcript;
the prior local state is whatever has been reached at the step's position in chain order. Applying a
step re-executes the call's local functions over those four inputs, at the position dictated by
chain order.  The operations that the transcript records are the account of that execution rather
than a replacement for it, since a value computed from a predicted local state can apply cleanly at
a later position and still be wrong. That ledger state and the step's block time both come from the
basis block—the chain block the call is prepared against—never from a later height or the host
clock, so one step reads one ledger height, and the call fails when the basis block is
unavailable. Capturing context-function results is what makes the incremental reconstruction sound:
nondeterminism reaches a capsule only through a context0function call, as the execution model
requires.

A transaction that does not complete takes its transcript with it—discarded whole—so a failed
transaction leaves no local-state change behind.  Any token deduction in a rehearsal disappears
along with the transcript. Confirmation of a completed transaction re-derives the call in chain
order, reusing the inputs pinned at preparation, so a reorder changes the order of application and
re-samples nothing. A locally rehearsed state change and the confirmed transcript for the same
transaction are one event, reconciled by transaction identity and never by arrival
order. Concurrency is answered by those same two moves: with several calls in flight at once, chain
order fixes the sequence, and transaction identity fixes which confirmation belongs to which local
rehearsal. A capsule may prepare several calls before any of them settles, so a transcript may meet
a local state later than the one its call was prepared against. A contract's local functions must
therefore produce a correct result from any reachable local state.  This is obligation on the
contract author, and it is the condition under which re-derivation always succeeds.

Local state must be durably retained and recoverable. The capsule secret is derived, so the seed
phrase and the capsule identifier recover it and nothing extra has to be backed up; local state is
reconstructed from retained transcripts.  Retaining them durably is the capsule runtime's
obligation, a requirement not addressed further here. A cleared browser cache therefore loses
nothing: the transcript log lives outside the cache. Losing that retained log, on the other hand,
causes the loss of local state; the seed phrase and the capsule identifier are not enough to
reconstruct it on their own.

Every piece of capsule data at rest—local state, snapshots, and transcripts alike—must be
encrypted. That is a hard requirement of this specification, not a host quality-of-implementation
matter: a transcript captures the capsule secret, so anything less exposes it to whoever can read
the host's storage. The key material derives from the wallet seed phrase and the capsule identifier,
so those same two inputs re-derive it with no separate backup, and it never leaves the capsule
runtime.

## Rationale

The preceding design follows directly from the need to manage trust boundaries when one call
orchestrates several contracts. No contract executable should ever reach another contract's local
state.  Sometimes the code to be executed is discovered only during execution; sometimes a newer
version of it is supplied. Local state management needs persistence and *must* follow the chain,
especially where the chain forks or rejects the transaction.

Everything here builds on three principles. 
1. A contract's code and its local state go into a single place, where the code runs. That makes
   dynamic code resolution possible and unifies local state management. 
2. Anchoring contract executables on chain removes variation in the shipped code
   and makes an executable both upgradeable and resolvable from an identifier that already exists.
3. Contract-authored local state management and its introspective API unify each contract's state
   and its DApp-facing API—exactly the unification required by persistence, correctness, and the
   proper trust boundaries.
   
All of this leaves the contract executable self-contained, which simplifies resolution. On that
basis, several DApps interacting with the same contract becomes a question of permissions alone, and
several local states per contract becomes a question of one further identifier, managed entirely
locally.

The canonical capsule executable settles **provenance, not correctness**: local-state values still
reach a circuit as private inputs supplied by whatever runs on the user's device, and a malicious
actor can supply invalid or self-serving ones. Circuits must go on enforcing their invariants
in-circuit. What canonical resolution buys is a uniform honest path.

### What Is Lost by Having a Canonical Capsule Executable

Prior to this proposal, the code responsible for the evolution of local state was supplied by each
application.  What disappears when the capsule executable is resolved from the chain is **per-DApp
variation in how local state evolves**. Two applications, each internally consistent, could
otherwise leave the user with two histories of one local state.  Having a single contract-authored
executable removes the variation, which is what makes correctness under failure and reordering
achievable at all. That removal rests on a core assumption, stated in [the capsule and on-chain
reference subsection](#capsules-capsule-identity-and-the-governed-on-chain-reference).

The prior behavior, although it allowed different DApps to define their own implementations and (to
some degree) semantics of local state, came with three big costs:

1. **Executables from different entities had to be reconciled**, by machinery that raises trust and
   compatibility questions.
2. **Local state could diverge past recovery**, into histories no related DApp could bring back
   together.
3. Trying to create a multi-contract runtime would cause **migration between local-state shapes to
  fall to the runtime**, because nobody else is placed to do it.

In contrast, a canonical executable makes migration at most the contract author's concern.

**Eliminating these shortcomings is not free.** In the newly proposed model, contract authors lose
the liberty of shipping arbitrary private-state code and updating it per DApp. And all of it rests
on chain-anchored resolution alone: a [governed on-chain
reference](#capsules-capsule-identity-and-the-governed-on-chain-reference), which does not exist
prior to this proposal.

### Are the Criteria Met?

The [Specification](#specification) section began with an enumeration of criteria to be satisfied.
Have they been met?

1. **Private state isolation** — yes: the state and the code manipulating it reside together inside
   the capsule, and they are managed *only* by the capsule runtime.
2. **Multiple logical profiles** — yes: capsules are identified by contract address and account ID.
3. **Controlled access to private state from applications** — yes: a value reaches a DApp only
   through an introspective API function whose disclosure the contract declares and for which the
   user has granted permission.
4. **Explicit private state evolution** — yes: the declaration and updating of private state is
   moved from the DApp to the contract.
5. **Well-defined access to non-deterministic context** — yes: contracts can access contextual
   information only through the context API provided by the capsule runtime.
6. **Explicit grant/deny access to private state by user** — yes: by the same mechanism as
   criterion 3.
7. **Private-state persistence** — yes: the whole state can be reconstructed from the seed phrase,
   capsule identifier, and encrypted transcript log, and retaining that log durably is an obligation
   on the capsule runtime.
8. **Resolving and executing a contract discovered dynamically** — yes: answered by resolution from
   the governed on-chain reference, the chain being the source.
9. **Private-state correctness under failure and reordering** — yes, given an obligation on the
   contract author: the pure application of transcripts in chain order, held within one governing
   executable version and carried across version boundaries by retained snapshots, with each call
   re-derived on confirmation, using the inputs fixed at rehearsal time.
10. **Upgrades without cascading effects** — yes: superseding the on-chain reference makes the new
    executable canonical for every capsule of that contract, with each one adopting it when it next
    crosses that version boundary.  Capsule identity and local state remain unchanged across
    upgrades, so a DApp requires updates only when the public interface to the contract changes.
11. **Local state isolation across cross-contract calls** — yes: each capsule maintains its state
    independently.  When one contract calls another, the callee's capsule owns its state.
12. **Controlled sharing without an assumed trust relationship** — partially: the capsule runtime
    supplies the boundary at which permission is request, recorded, and bound to an attested origin;
    how that request occurs and how its response is handled by the capsule runtime or the calling
    entity is not fully specified. It is left to other proposals to define the granularity,
    vocabulary, and shapes for access permissions.

Of the twelve, all but one are fully answered, and the last is partially answered.

### The Rejected Alternative: Same-Origin Separation

The rejected alternative makes the browser's same-origin policy the separation mechanism: handle a
cross-contract call by resolving to a DApp associated with the callee.  Then, open a page of that
origin, call, and take the result back.  This would be an OAuth pop-up flow.

**Why this solution would be appealing:** It reuses a boundary every browser and web developer
understands, and it needs no new execution model, no new artifact, and no on-chain state.  If
encapsulation were the only criterion, it would be cheaper.

**Why it falls short::**

- **It cannot reach a contract without a no known DApp, and it makes the choice of DApp fragile.**
  The mechanism needs an origin to open, while the DEX example supplies only an address.  Being able
  to reach it at all depends on which origin holds the local state.
- **It does not address persistence and correctness.** It supplies encapsulation and nothing else.
  Both persistence and correctness under reordering remain each origin's problem to solve, or else
  they are solved by moving all local state into a wallet, eventually arriving at something like
  this proposal's design, by a longer road.
- **Every DApp deployment pays a tax.** Any application that might hold a callee's local state owes
  a page to open, a protocol to speak, and availability to strangers.
- **A local-scheme frame cannot be given a policy of its own.** A frame from a local scheme
  (`srcdoc`, `blob:`, `data:`) clones its creator's policy container, and embedded enforcement only
  tightens what was inherited, where implemented at all: the boundary cannot be stronger than the
  page opening it.
- **The remote-origin variant imports a new trust anchor.** Whoever controls that origin controls
  the boundary, and must own its hosting, availability, versioning, and review.

## Path to Active

*Not yet drafted. The acceptance milestones and the sequence behind them need detail which this
draft deliberately does not carry.*

## Backwards Compatibility Assessment

First, deployed contracts and already-compiled DApps keep working indefinitely. Nothing here is
retroactive and nothing is broken. A contract with no on-chain capsule reference gets no
capsule-runtime service at all and continues unchanged under today's model until its author attaches
one. The two models coexist. There is no flag day, no deprecation, and no end date. A path for
migrating existing private state into a capsule may exist, but is never a precondition, a transition
period, or a fallback.

Second, from the Compact release that first supports capsules, the compiler no longer emits
old-model private-state support. Updating a contract therefore means adopting the capsule model.
There is no release in which the compiler emits both. That ends support for the old model in new
compilations without contradicting the first claim: it constrains what the compiler emits, while
what the runtime accepts belongs to the third claim. The compiler claim also has a hosting-side
consequence: a contract recompiled under a capsule-enabled release will not run on an execution host
that lacks the capsule runtime. Adoption therefore takes two acts: recompilation and the governed
write of the capsule reference.  They land together, because the reference is carried at deployment
or written with the verifier keys produce by recompilation. No deployed contract sits recompiled
with no reference, served by neither model.

Third, about the runtime rather than the compiler: the capsule runtime never accepts DApp-supplied
private-state code. There is no bounded period in which it does.

For a contract author adopting a capsule this is materially breaking: a contract's local state
becomes contract-controlled, deterministic, and Compact-declared, no longer a DApp's to supply. The
mapping of former witness function into the new model is described in the specification of [the
execution model](#the-execution-model-local-context-and-introspective-functions).

The expected first capsule runtime is likely embedded in a wallet shipped as a browser extension,
which means that extension-store policy influences the design. Contract-authored code must run
isolated from extension APIs, with at least per-capsule granularity.  This is the trust boundary
drawn in [the capsule and on-chain reference
subsection](#capsules-capsule-identity-and-the-governed-on-chain-reference).  Chrome Web Store
policy is one environment's expression of that requirement, but not its source: extension stores in
general prohibit remotely fetched code in an API-bearing context, classifying by execution context.
Thus, packaging the code as data changes nothing.  One dependency of this proposal is to confirm the
classification of the various elements in the context of intended distribution avenues, such as
browser extension stores.

No Midnight network upgrade is anticipated on the path proposed here, which is to reuse an existing
ledger field.  Midnight Ledger 9.1 places, beside each entry point's verifier keys, a slot holding
an uninterpreted byte string—the entry point's `ir` field.  The "IR slot" named in the gaps below
refers to this field. The slot is carried from deployment or, thereafter, written and cleared only
by the contract's maintenance authority, under the same threshold signature and replay counter that
govern those keys. The field thus satisfies this proposal's need for a gated, authorized write path,
and it is how a contract's on-chain code artifacts are already governed. A capsule reference, or an
executable small enough to fit the ledger's contract-metadata limit, can adopt it.

Three gaps remain:

1. **Usage of the IR slot** — the slot is present per entry point, on the expectation that each
   would provide its own IR; the runtime needs a single one for the contract. Which one to read,
   what to do when several are present, and questions like them are real decisions still to be made.
   Regardless, the field already exists, with nothing new to build once its use is agreed.
2. **IR slot versioning** — the ledger cannot verify any property of the uploaded information, which
   leaves all validation to off-chain components. One example is the need for forward-only
   versioning; this proposal calls for it, but the slot cannot inherently guarantee it.  Even to
   verify the ordering off-chain, a decision must be made about encoding the reference so that it
   carries order information.
3. **The slot's assigned purpose** — the slot named `ir` already has an assigned meaning, which is
   the intermediate representation of the operation for constructing proof-related artifacts,
   although the ledger does not enforce this.  Putting a capsule reference in its place therefore
   depends on agreement among the stakeholders that the slot may be shared.  If the change is agreed
   to, nothing more need be resolved beyond the decisions described in gap 1.  On the other hand, if
   the slot cannot be shared, the alternative is a sibling field on `ContractOperation`—the
   serialization change that took `contract-operation[v5]` to `[v6]`—plausibly a network upgrade,
   and thus the one gap in this list that could affect the consensus layer.

## Security Considerations

**The capsule secret.** The secret is deterministically derived from the main seed, separately for
each capsule, and is available to the capsule from its initialization onward. It belongs to the
capsule's *context*, not to its local state.  Under this proposal, the secret can be reached only
though a runtime-provided context API function, and only contract code may call it. Contracts and
the runtime must *treat* it as a secret key, disclosing it neither to the chain nor to the DApp; it
is as secret as the binary representation of the wallet spending key. It is also used to derive the
keys that encrypt the capsule data at rest.

**Data at rest.** Every foreign-function result is captured in the persisted transcript so that a
replay reads the pinned value without re-invoking; the capsule secret arrives as one such result,
and the same uniform rule captures it, so transcript storage is secret-grade. That is what the
at-rest encryption requirement in [persistence](#persistence-and-correctness) rests on.

**Blast radius.** The secret is per-capsule and contract-specific, its whole role being the
contract's own authentication; native tokens stay with the wallet and its own keys. Compromising one
capsule's secret therefore reaches exactly that capsule's authentication, leaving the user's funds,
the wallet and every other capsule untouched.

**Reading the on-chain reference.** An inability to retrieve some capsule's executable
representation is the principal attack surface for the required canonicity of executables.  The
protection is the requirement not to proceed with any sort of fallback.  A source that lies falls
outside that rule: the chain is the source, and this design assumes the client's view of it is
honest. Whether the existing governed write path may carry the reference is not yet agreed ([gap
3](#backwards-compatibility-assessment)).

**Disclosure.** A capsule's introspective functions return values to DApps, and those values may
reach an unknown public, exactly as a ledger write may, with only the wallet's permission system to
protect from disclosing secrets.  This is a permission boundary, but not fundamentally different
from public disclosure. Such a disclosure is never inherently contained, and the analysis is
accordingly of that boundary—where the user grants a DApp access to a capsule—still to be built.

**Residual limitations.** 
- Encapsulation provides no inherent protection against resource exhaustion, so that remains an
  attack vector.
- The context API function set is shared across all contracts, so it could be understand as a single
  point of failure for revealing private state, especially because that API also provides the
  secret.  Context functions being read-only with respect to state outside the capsule is a
  requirement the runtime cannot enforce once a function performs I/O. 
- The contract-author obligation on local functions that [persistence](#persistence-and-correctness)
  sets out is likewise unenforceable, and its breach breaks local-state correctness. 
- The granularity at which permissions are granted by the user has been left unspecified by this
  proposal, and coarse granularity is a residual limitation and risk.
- Local state can be reconstructed only from retained transcripts.  Retention is the capsule
  runtime's obligation, but this proposal has specified no mechanism, and losing access to stored
  transcripts means local state. The transcript log is the one recovery input that nothing else can
  regenerate. 
- Additional limitations imposed by the canonical executable model are discussed in [the Rationale
  section](#rationale).  Specifically, note that canonical resolution establishes provenance, but it
  does not guarantee correctness.

## Implementation

The capsule runtime must ship as a well-packaged, well-specified SDK, so that a wallet—or whatever
else hosts it—can integrate it easily. This is not merely a packaging detail, but rather a
requirement on the deliverable. The capsule runtime is available to a user only through a hosting
environment that has adopted it, and adoption cost is what decides whether it does. Being well
specified enables wider adoption and design diversity, so that alternatives to the initial SDK can
be independently developed and maintained.

The Midnight components that require changes or need to be created, relative to their prior
releases, are:

- the Compact compiler — local state declarations, local functions, introspective functions, and a
  checkable disclosure declaration; Compact declares witness functions, but has no implementation of
  local state update in the contract code today, so these changes touch all layers of the compiler;
- the compiled-contract representation — currently unspecified, other than ZKIR;
- the Compact runtime — to be adjusted into a lower-level execution driver that yields control to
  the capsule runtime on calls to other contracts;
- the capsule runtime — to provide the execution boundaries and to orchestrate the whole process of
  calling circuits and capsule API functions;
- the wallet SDK — secret derivation, capsule runtime interaction;
- the wallets themselves;
- the DApp Connector API, via the dedicated connector MIP;
- DApp SDKs such as Midnight.js — to adopt the new model and APIs for newly compiled contracts;
- the ZKIR and its tooling — if the relavant CoIP takes the ZKIR-extension direction, to become the
  carrier of the whole contract executable and to provide an ad-hoc prover-key generation API.

## Testing

*Not yet drafted.*

## Versioning

Different aspects of what is proposed here will be versioned in different ways:
- The changes to the Compact language will be specified and versioned in Compact Improvement
  Proposals (CoIPs) and governed under the LFDT Minokawa project.
- Specifications for the interfaces that define the boundaries between DApps and the capsule runtime
  and between wallets and the capsule runtime will likely have their own distinct MIPs in the
  future, with their own versioning rules.
- Standards for representing compiled contracts directly on a Midnight blockchain or indirectly
  through references on the blockchain will also have their own MIPs and versioning rules.

If this proposal is accepted, it will become **version 1.0** of the architectural requirements and
design outline for the capsule runtime and for the handling of private state among contracts.  Small
changes to the design or requirements will be delivered as new versions of the same specification.

Major changes to the understanding of the requirements or to the specification should be delivered
not as an incremental version, but as a superseding MIP.

## References

- [HTML Standard — Policy
  containers](https://html.spec.whatwg.org/multipage/browsers.html#policy-containers)
- [MIP-1 — Midnight Improvement Proposal
  Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-mip-process.md)
- [CoIP-1 — Compact Improvement Proposal
  Process](https://github.com/LFDT-Minokawa/compact/blob/main/coips/coip-0001.md) — the process that
  defines what a CoIP is and where one lives
- [MPS-0021 — Phase 2: Contract to
  Contract](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0021-phase2-contract-to-contract.md)
  — the problem statement this MIP answers
- [MPS-0022 — A Standard, Language-Agnostic Representation of Compiled Compact
  Contracts](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0022-standard-contract-representation.md)
  — a related problem statement, which the CoIP encoding the contract-authored function kinds may
  also answer
- [DApp Connector API
  specification](https://github.com/midnightntwrk/midnight-dapp-connector-api/blob/main/SPECIFICATION.md)
- [Midnight glossary — custom spend logic](https://docs.midnight.network/glossary) — the feature's
  only authoritative description

## Acknowledgements

*Not yet drafted — the people who contributed to the design research behind this proposal and its
reviews, and the workshop participants.*

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement
[Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
