# Reorgs and Finality

Cross-link: [`docs/concepts/finality-models.md`](../../concepts/finality-models.md) (placeholder; concept doc to be written).

## Finality model

Casper FFG. Finalization happens at epoch boundaries when 2/3+ of validators attest to two consecutive checkpoints.

| State | Meaning | Reorg requirement | Typical age |
|---|---|---|---|
| `head` | LMD-GHOST tip | 1 honest proposer flips | 0–12 s (one slot) |
| `safe` | Justified checkpoint | Slashing 1/3+ stake | ~6.4 min (one epoch) |
| `finalized` | Two-checkpoint-justified | Slashing 1/3+ stake | ~12.8 min (two epochs) |

**An event is final** when the block containing it is at or below the `finalized` checkpoint reported by the beacon node. Use `/eth/v2/beacon/states/head/finality_checkpoints` and resolve the slot to an EL block.

## Empirical reorg depth

Post-Merge (2022-09-15 onward), reorgs of unfinalized blocks happen but are usually 1–2 slots. Multi-slot reorgs cluster around proposer outages, relay misbehavior, or slot-0 missed-block patterns.

The most-cited multi-slot reorg occurred **2022-05-25 at 08:55 UTC** — a 7-block reorg on the beacon chain attributed to a combination of a late block proposal, a proposer-boost fork-choice update, and the slot-0 timing edge case. ([Barnabé Monnot's analysis](https://barnabe.substack.com/p/pos-ethereum-reorg)) Reorgs deeper than 2 slots are rare enough that they're individually noteworthy.

Pre-Merge (PoW), reorgs were probabilistic. The historical "12 confirmations" rule for high-value transfers is a PoW heuristic and is irrelevant post-Merge — it overshoots head reorg risk and undershoots finality reorg risk.

## Safe confirmation depth

Different consumers tolerate different risk:

| Consumer | Recommended depth |
|---|---|
| UI showing "tx confirmed" | 1–2 blocks (head) |
| State that may be acted on automatically | Wait for `safe` checkpoint (~6.4 min) |
| Irreversible action (bridge mint, payout) | Wait for `finalized` (~12.8 min) |
| Compliance / audit trail | Finalized + buffer (e.g. 64 slots after finality) |

## Reorg detection

Two complementary mechanisms:

**EL-only (parent-hash linkage):** subscribe to `newHeads`. On each new block, compare its `parentHash` to the previously-stored block's hash. If they differ, walk backward until a common ancestor is found, mark divergent blocks as orphaned, and reapply the new chain.

**EL + CL (finality-driven):** subscribe to beacon SSE for `head` and `finalized_checkpoint` events. Use CL events to drive EL queries. This catches finality-based confidence transitions the EL alone cannot signal.

In production, run both. EL alone tells you "the chain forked"; CL tells you "this fork is now provably permanent".

## Recovery pattern

```rust
async fn handle_new_head(new: Block) -> Result<()> {
    let stored_parent = db.get_block_hash(new.number - 1).await?;
    if stored_parent == Some(new.parent_hash) {
        // common case: linear extension
        db.insert_block(new).await?;
        return Ok(());
    }
    // reorg: walk back to common ancestor
    let mut walk = new.parent_hash;
    let mut new_chain = vec![new];
    loop {
        let candidate = rpc.get_block_by_hash(walk).await?;
        if db.contains_block(candidate.hash).await? {
            // candidate is the common ancestor
            break;
        }
        new_chain.push(candidate.clone());
        walk = candidate.parent_hash;
    }
    db.mark_orphaned_above(walk).await?;
    for block in new_chain.into_iter().rev() {
        db.insert_block(block).await?;
    }
    Ok(())
}
```

Per-table reorg-aware-write strategies are in [storage.md](storage.md#reorg-aware-writes).

## Inactivity leak

Under non-finalizing conditions (mass validator outage, network partition, post-quantum migration crisis), the beacon chain enters **inactivity leak**: it keeps producing blocks but does not finalize. Inactive validators' stake is slowly burned to restore the 2/3 threshold; once it does, finality resumes.

For an indexer: **`finalized` can stop advancing for hours or days.** Indexers that block on finalization for downstream commits need a circuit breaker — either fall back to `safe` with an explicit downgrade and an alert, or pause and surface the stall to operators. **Don't silently retry on finality** — if Eth is in inactivity leak, retrying is a hot loop.

## Pre-Merge history

Blocks before 15,537,394 (2022-09-15) were produced under Proof-of-Work. Reorg behavior was probabilistic; finality concept didn't exist. An indexer covering pre-Merge data must implement two distinct reorg-handling regimes — most indexers handle this by treating the Merge block as a hard boundary and only running the post-Merge code path going forward, with the pre-Merge data treated as immutable history (the chain hasn't reorged that deep since).
