---
MPS: "0028"
Title: Pre-Finality State Visibility for Indexer and Wallet
Authors: Giles Cope <giles.cope@shielded.io>
Status: Proposed
Category: Core
Created: 25-Jun-2026
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

## Terms

A few words are used throughout this document:

- **Block.** A batch of transactions added to the chain.
- **Finalization.** The step where the network marks a block as permanent, so it
  can never be undone. Midnight does this with a tool called GRANDPA. Finalization
  usually happens within about 3 blocks (around 18 seconds at a 6-second block
  time), and can take longer if the network is under stress.
- **Indexer.** A service that reads chain data and serves it to wallets and DApps.
- **DApp.** A decentralized application: software that talks to the chain, often
  through a wallet.
- **Reorg (reorganization).** When the chain drops a block it had before and
  replaces it with another. Finalized blocks can never be dropped this way.
- **Best chain.** The latest blocks the node has, including ones that are not yet
  finalized. These can still change in a reorg.
- **Optimistic view.** Showing the likely result early, before finalization, and
  marking it as not yet final.

## Abstract

A Midnight wallet or DApp shows the user balances, coins (UTXOs), whether a
transaction succeeded, and contract state. Today all of this updates only after a
block is finalized. Finalization usually takes about 3 blocks (around 18 seconds),
and can take much longer if the network is under stress.

The reason is the data path. The indexer reads only finalized blocks. The wallet
gets its data from the indexer, and it waits for the `Finalized` status before it
treats a transaction as done. So the user must wait for finalization to see any
change, even though the block was already executed and the result was ready the
moment the block was produced.

This problem statement describes that delay: the gap between when a transaction's
result exists on a node and when normal wallet or DApp tools can show it. It
describes the problem only. It does not propose a fix.

## Vision

A wallet or DApp shows the result of a transaction about one block after the
transaction is included. To do this it reads the best chain, not only finalized
blocks. On top of the data, it shows a clear signal for whether each piece is
finalized.

Most of the time, users get fast feedback. Apps that need only finalized data can
still ask for it and keep the guarantees they have today. The speed users feel
then depends on how fast blocks are produced, not on how fast finalization runs.
If finalization slows down, users have less confidence in the data, but they still
see the data.

## Problem

### Current State

**The indexer reads only finalized blocks.** The indexer's block stream comes only
from the finalized-blocks subscription:

```rust
// chain-indexer/src/application.rs — node_blocks()
let blocks = node.finalized_blocks(highest_block);
```

This is a deliberate design choice, not an oversight. A finalized block can never
be dropped in a reorg, so the indexer never has to undo data it has already stored.
The cost of that safety is the delay this document is about.

**The wallet follows the indexer.** The wallet gets its state from the indexer's
GraphQL API (the wallet `indexer-client` package). Its submission flow waits for
the `Finalized` status (`packages/capabilities/src/submission/submissionService.ts`,
`packages/node-client`). The wallet cannot show a balance or coin change that the
indexer has not stored yet. And the indexer stores nothing before finalization.

The result is one path that waits for finalization from start to end:

> block is produced and executed → result is ready → **(wait for finalization)** → indexer reads it → wallet or DApp asks for it → user sees it.

### Limitations

**The delay is not required by the protocol.** The result of a transaction is
ready, and the same on every node, as soon as the block is produced. Waiting for
finalization adds the full finalization delay (several blocks) to every change the
user sees. The protocol does not need this wait.

**No fast feedback is possible with normal tools.** A wallet cannot show
"submitted, in a block, waiting for finalization" together with the real result,
because the only data it reads is finalized data. The user sends a transaction, a
block includes it, and the wallet still shows nothing until finalization.

**The delay adds up across a series of transactions.** Many flows are a chain of
steps, where one transaction depends on the result of the one before it. If each
step waits for finalization before the next can start, the waits stack up: ten
steps cost about ten finalization delays (around 3 minutes), not one. But the next
transaction usually does not need its input to be final. It only needs the input
to exist on the best chain. Waiting for finalization between every step is rarely
required, yet today it is the only safe way to read the result of the previous
step.

The same cost applies to bulk submission, not just dependent steps. The node's own
load-testing toolkit (`midnight-node/util/toolkit`) sends many transactions in a
row. It was built to *not* wait for finalization between transactions: it submits
at a fixed rate and tracks each transaction on its own, in parallel
(`src/sender.rs`, `send_worker`). If it had to wait for each transaction to
finalize before sending the next, its throughput would drop sharply, from many
transactions per second to roughly one every 18 seconds. To do this the toolkit
reads transaction status straight from the node over RPC and does not use the
indexer at all. This shows two things: the finalization wait would be a serious
cost if it were on the critical path, and a consumer that reads from the node
directly can already avoid it. Wallets and DApps that use the standard indexer path
cannot.

**If finalization stops, visibility stops.** Sometimes finalization falls behind.
This is a known risk under heavy load (see *References*). When it happens, blocks
are still produced, but the indexer and every wallet behind it stop updating. No
new finalized blocks arrive, so there is nothing to read. Users lose all
visibility at the worst moment, even though new blocks are still being produced.

**User-felt speed depends on finalization, not block production.** Even if block
production becomes faster, users do not feel it, because they still wait for
finalization. The speed of the chain, as users experience it, is limited by
GRANDPA, not by how fast blocks are made.

## Use Cases

**UC1: Slow wallet balance.** A user sends NIGHT tokens. The transaction is included in
the next block, but the wallet balance does not change until finalization. To the
user, this looks the same as a stuck transaction. It makes users lose trust.

**UC2: DApp confirmation.** A DApp wants to show "your action is in a block,
finalizing now", with the real result already visible. Today it can only show
nothing, and then show everything at finalization. There is no accurate in-between
state.

**UC3: Finalization falls behind.** Finalization slows down under load. Blocks are
still produced, but every wallet that reads from the indexer stops updating: no
balances, no history, no contract state, until finalization catches up. The chain
is fine, but for users the service is down.

**UC4: Fast interaction.** Games, trading, and fast contract use need feedback at
the speed of block production. A finalized-only view makes every step feel as slow
as the finalization delay, no matter how fast blocks are produced.

**UC5: A series of dependent transactions.** A user or a script runs several
transactions in order, where each one uses the result of the previous one (for
example: receive coins, then split them, then send each part). Each step only
needs the previous result to exist on the best chain, not to be final. But today
the wallet must wait for finalization before it can safely read the previous
result, so the steps run one finalization delay apart. A ten-step flow that could
finish in about a minute instead takes several minutes.

## Goals

- **Show pre-finality state.** Make best-chain state visible to wallets and DApps
  for the data they already use: balances, coins (UTXOs), transaction status, and
  contract state.
- **Keep a clear finality signal.** For each piece of data, the user must still be
  able to tell whether it is finalized and therefore safe from being undone.
- **Do not break finalized-only users.** Apps that depend on finalized data today
  must keep working the same way.
- **Handle reorgs clearly.** Avoiding reorgs is the reason the current design uses
  finalized-only data. Any pre-finality view must define how it undoes the data
  from a dropped block.
- **Limit the delay.** Goal: a transaction's result is visible about one block
  after it is included, no matter how slow finalization is.
- **Support transaction sequences.** Let a flow start the next transaction as soon
  as the previous result is on the best chain, so the steps run at block speed and
  the delays do not stack up. Waiting for finalization should be a choice the app
  makes for the steps that truly need it, not a cost paid on every step.
- **Stay useful when finalization is slow.** If finalization falls behind, the
  system should lower the confidence of the data it shows, not stop showing data.

## Expected Outcomes

- Wallets and DApps can show transaction results at the speed of block production.
  Users get fast, accurate feedback instead of a long silence.
- Optimistic feedback (submitted → in a block → finalized, with the real result at
  each step) becomes possible with standard tools.
- The speed users feel no longer depends on finalization. It tracks block
  production, so faster blocks become faster for users too.
- Wallets keep showing data when finalization is slow. A total outage becomes a
  lower-confidence mode instead.
- A series of dependent transactions runs at block speed instead of one
  finalization delay per step, turning multi-minute flows into ones that finish in
  about a minute.

## Open Questions

- **Undoing dropped blocks.** How does a wallet or DApp undo data from a block that
  finalization later drops? This is the main risk the current finalized-only design
  avoids, so any fix must answer it.
- **Where does the best-chain view live?** In the indexer (two streams, best and
  finalized), in the wallet, or in the node? Each choice has different reorg
  handling and trust trade-offs.
- **How is confidence shown?** Should each piece of data carry a status (`InBlock`
  vs `Finalized`), or a block depth, so each app can choose its own safety level?

## Recommended MIPs

- **Best-chain state in the indexer.** Add a best-chain view next to the finalized
  view. It must undo data on a reorg and mark the finality status of each block, so
  apps can choose fast data or finalized data.
- **Optimistic wallet state with a finality tag.** Let the wallet show in-block
  state, tagged with its finality status. It should default to finalized data for
  actions that must not be undone, while showing pre-finality results for speed.

## References

- MIP "Node-Side Surfacing of Ledger Events via `frame_system::Events`"
  (open PR; branch `docs/mip-1474-node-expose-ledger-events`). It documents that
  events are readable at execution time, before finalization, and that today's
  indexer re-runs transactions on finalized blocks only. This MPS extends that
  pre-finality idea from events to state.
- Source: `midnight-indexer` — `chain-indexer/src/application.rs` (`node_blocks`
  reads from `finalized_blocks`), `chain-indexer/src/infra/subxt_node.rs`
  (`FINALIZATION_SAFETY_MARGIN`).
- Source: `midnight-wallet` — `packages/indexer-client` (state from the indexer
  GraphQL API), `packages/capabilities/src/submission/submissionService.ts` and
  `packages/node-client` (waits for `Finalized`).
- Source: `midnight-node` — `util/toolkit/src/sender.rs` (`send_worker` submits at
  a fixed rate and watches each transaction in parallel rather than serializing on
  finality; `--no-watch-progress` skips the finality watch) and
  `util/toolkit/src/client.rs` (tracks best-block and finalized height directly
  over RPC, without the indexer). Shows pre-finality, node-direct tracking is
  already done in practice.

## Acknowledgements

Authored by Giles Cope.

## Copyright

This MPS is licensed under CC-BY-4.0.
