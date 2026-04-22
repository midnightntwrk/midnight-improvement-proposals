**dApp name:** zkArt

**Contract repository:** https://github.com/niclas-ART/zkArt

**Brief description:**

zkArt is a privacy-preserving digital art auction platform on Midnight. Sellers
mint a single NFT per artwork and escrow it into a sealed-bid auction contract.
Bidders commit NIGHT during a time-bounded bidding window, then reveal bids in a
separate window to determine the winner. Bidder identities are derived from local
secret keys and never published; the seller identity is stored in sealed contract
state. After the auction closes the winner claims the shielded NFT, the seller
claims payment (minus a 10 % platform fee), and losing bidders reclaim their
escrowed NIGHT.

Two Compact contracts are deployed per auction:

- **NFT contract** — mints a single shielded coin whose color uniquely
  identifies the artwork; stores artwork metadata (name, description, image URI,
  media type, edition) on the public ledger.
- **Auction contract** — manages the full auction lifecycle: NFT deposit,
  sealed-bid submission, commit-reveal winner determination, NFT transfer to
  winner, payment split to seller and auction house, and refunds to non-winners.

---

| Category            | Self-assessed score (1–3) |
| ------------------- | :-----------------------: |
| Privacy-at-risk     |             1             |
| Value-at-risk       |             2             |
| State-space-at-risk |             2             |

---

### Privacy-at-risk — 1

**Rationale:**

Bidder and seller identities are sealed. The on-chain `winning_bidder_key` is not
a wallet address — it is a `persistentHash` of a random 32-byte secret (generated
via `crypto.getRandomValues`, stored only in the user's `localStorage`) mixed with
an auction-specific tag (`"auction:pk:"` + `nft_color`). This key is unlinkable to
the user's wallet, real-world identity, or their participation in other auctions.
Seller identity and NFT color are sealed contract fields. Bid amounts become
visible during the reveal phase, but art auction bids are pseudonymous financial
signals in a public auction context, devoid of real-world harm. Artwork metadata
(name, image URI, edition) is public by design. No data exposed by the contract
links to real-world identities or reveals private behavioral patterns.

**Mitigations:**

- Per-auction key derivation (`persistentHash([tag, auction_id, sk])`) ensures
  on-chain keys are auction-scoped and unlinkable across contracts. The underlying
  secret key never leaves the browser's `localStorage`.
- Bid amounts are hidden during the commit phase via hash commitments, and even
  once revealed they pose no privacy risk beyond what any public auction
  inherently discloses.
- This initial version denominates bids in NIGHT (the only token currently
  available on mainnet), which is unshielded — bid transactions are therefore
  visible on-chain. The DApp includes disclaimers informing users of this
  limitation. A future version will adopt a shielded token once one is available,
  improving bid-amount privacy.

---

### Value-at-risk — 2

**Rationale:**

Bidders' NIGHT is escrowed in the contract for a bounded period between bid
submission and auction close. The escrow window is defined by `end_time` and
`winner_check_end_time` (set at contract deployment). After the auction closes,
the winner's funds are paid out and all losing bidders can reclaim their full
deposit. Funds do not accumulate across auctions — each auction is an independent
contract instance.

**Mitigations:**

- Time-bounded escrow with explicit deadlines enforced on-chain.
- Seller escape hatch (`reclaim_nft`) prevents permanent NFT lock if no valid
  bids are placed.
- Refund circuit (`claim_refund`) zeroes out escrowed amounts on claim to prevent
  double-spend.
- Payment circuit validates the 10 % fee split arithmetically before releasing
  funds.
- Each auction is a separate contract deployment, so an exploit in one instance
  cannot drain funds from another.

---

### State-space-at-risk — 2

**Rationale:**

Each auction contract is an independent deployment with a bounded lifecycle, and
after the auction closes no new state entries can be created. However, the
`bid_commitments` and `escrowed_amounts` maps grow with each unique bidder and
there is no hard cap on the number of participants — an arbitrarily large number
of bidders can submit during the bidding window. The NFT contract enforces a
single mint (`total_minted == 0` guard). State growth is bounded per user and has
a natural ceiling (the auction's time window limits participation), but it is not
statically bounded.

**Mitigations:**

- One contract per auction ensures no cross-auction state accumulation.
- The single-mint constraint on the NFT contract caps metadata storage at one
  entry.
- Auction time windows impose a deadline on state growth — no entries can be
  added after `end_time`.
- State-clearing is user-initiated: bidders call `claim_refund` which zeroes out
  their escrowed amount entry.
