# Forks and Changelog

Scroll's upgrade timeline. Activation timestamps need verification at deep-dive time.

## Major upgrades

| Upgrade | Approx date | Indexer impact |
|---|---|---|
| **Genesis** | 2023-10-17 | Block 0. Pre-Bernoulli era. |
| **Bernoulli** | {{unsourced: 2024 Q1-Q2}} | EIP-4844 blob support — `op-batcher`-equivalent moves from calldata to blobs. **L1 fee formula changes.** |
| **Curie** | {{unsourced: 2024-Q2}} | EIP-1559 adoption on L2 (if not already), additional Cancun-equivalent EIPs. **L1 fee formula changes again.** |
| **Darwin** | {{unsourced: 2024-Q3}} | Compressed batch encoding improvements. Indexer impact: batch decoder version bump. |
| **Euclid** | {{unsourced: 2024-Q4}} | TBD. {{unsourced: confirm scope}} |
| **Feynman** | {{unsourced: 2025+}} | Pectra-equivalent (EIP-7702) port if/when shipped. |

Always pull current fork heights from the Scroll chain config or [`scroll-tech/scroll-contracts`](https://github.com/scroll-tech/scroll-contracts) release tags.

## L1 fee formula evolution

Tracks Bernoulli / Curie / Darwin shifts. Similar to OP Stack's pre-Ecotone / post-Ecotone / post-Fjord formulas — but with Scroll-specific scalars and FastLZ-style size estimation in later forks.

If your indexer recomputes L1 fees from receipt fields (instead of trusting `receipt.l1Fee`), version the formula by L2 block timestamp against the activation timestamps.

## What an indexer must do per upgrade

1. **Pin fork heights** at startup from the chain config.
2. **Version receipt L1-fee decoder** by upgrade.
3. **Update batch decoder** at Bernoulli (calldata → blob) and Darwin (compression changes).
4. **Backfill any derived fee-in-USD computations** for affected blocks if the formula shifted.

## Beyond the spec timeline: prover system upgrades

Scroll's prover has been upgraded multiple times for performance / cost reductions. These do not affect L2 RPC behavior — same state transitions, same proofs verifying — but the **L1 verifier contract** gets upgraded. An indexer that decodes L1 verification events must update accordingly.

## Where this doc is incomplete

The exact dates and scopes for Bernoulli, Curie, Darwin, Euclid, Feynman are flagged because Scroll's documentation has not always shipped a single canonical changelog. Verify against:

- Scroll Foundation blog posts.
- [`scroll-tech/scroll-contracts`](https://github.com/scroll-tech/scroll-contracts) release tags.
- `ScrollChain` L1 contract upgrade events (proxy implementation changes).
- L2BEAT's Scroll page for independent timeline tracking.
