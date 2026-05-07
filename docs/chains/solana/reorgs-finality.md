# Reorgs and Finality

Solana exposes finality **explicitly** via three commitment levels. Indexers pick one and respect it.

## The three commitments

| Commitment | Means | Reorg likelihood | Lag |
|---|---|---|---|
| **`processed`** | Local validator has seen the slot. | Real — anything is possible. | 0–400 ms |
| **`confirmed`** | Supermajority of voting stake has voted on this slot or a descendant. | Theoretically possible but rare in practice. | ~1–2 seconds |
| **`finalized`** | Rooted by 2/3+ of voting stake; effectively immutable. | Effectively never. | ~12–32 seconds |

Specify commitment per RPC call:

```
getBlock(slot, { "commitment": "confirmed" })
getSignaturesForAddress(addr, { "commitment": "finalized" })
```

Indexers using Geyser plugins also receive a per-slot commitment update — watch for the slot transitioning to `confirmed` and `finalized`.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics, mempool-equivalent UI | `processed` (mark explicitly) |
| Token transfers, contract state | **`confirmed`** (the standard choice) |
| Bridge accounting, regulator-facing | `finalized` |

`confirmed` is the dominant choice. The reorg risk past `confirmed` is low enough for most workloads, and the latency is much better than `finalized`.

## Reorg sources

### 1. Fork during the unconfirmed window

Two leaders' descendants can compete briefly before votes accumulate on one branch. **Slot N being on a fork** is what "reorg at processed" means.

In practice, Solana validators converge in 1–2 slots; reorgs past `processed` of more than 1 slot are uncommon.

### 2. Long fork during instability

During cluster stress (degraded network, extreme load) the convergence can take longer. Reorgs of multiple slots happen but are rare.

### 3. Vote stalls due to non-finalization

If voting stalls (network partition, validator outage past 1/3+ stake), `confirmed` advances normally but `finalized` lags. Has happened during incidents.

### 4. Cluster halts

Solana has had multi-hour cluster halts (notably 2022). During a halt:
- All slots stop advancing.
- Indexers receive no new data.
- On restart, slots resume from the last finalized slot — there is **no replay** of skipped slots; they are simply gone.

Indexers must:
- Not crash on stale slot advancement.
- Reconcile carefully on restart — some "expected next slots" never existed.

## Reorg detection

When ingesting at `processed` (low latency), build reorg detection into the pipeline:

1. Track the highest slot observed.
2. On each new slot: `getBlock(slot, processed)`. If the parent slot doesn't match your storage, walk back.
3. When a slot transitions to `confirmed`: re-fetch with `confirmed` commitment and reconcile any differences.
4. Same for `finalized`.

In practice, ingesting at `confirmed` and never at `processed` is much simpler — your stream is already past the reorg-prone window.

## Skipped slots

Slot `N` may not produce a block — the leader was offline, missed timing, or was slashed. This is **normal**. Indexers must:
- Not error on `getBlock(N) → null`.
- Track `slot` (sequential, includes skipped) separately from `block_height` (only blocks).

The `getBlocks(start, end)` RPC method returns the slots that **did** produce blocks in a range — useful for backfilling.

## Confirmation lag during cluster congestion

Under heavy load (NFT mints, popular launches), Solana has historically experienced:
- **Forwarding delays**: txs not landing in blocks despite being broadcast.
- **Vote propagation delays**: `confirmed` lag growing to many seconds.
- **Drop in TPS** when validators can't keep up.

For indexers, this means: **never assume confirmation latency is bounded by a constant**. Use the actual commitment level transition signals from Geyser or RPC, not a "slots × 400ms" calculation.

## Withdrawal-equivalent: nothing to worry about

Solana has no L1 settlement (it's an L1 itself). There's no two-stage finality across systems. Once `finalized`, the data is canonical.

## Halts: indexer survival checklist

- **Heartbeat**: monitor slot-advancement rate; alert below threshold.
- **Tolerate prolonged silence**: don't reconnect-loop into the validator at high frequency.
- **Reconcile on restart**: when the cluster comes back, your stored "highest slot" may not match the cluster's current slot — there may be "missing" slots that genuinely never existed.
- **Read official channels** for any retroactive corrections.

Solana's halt history is documented; review it before going to production.
