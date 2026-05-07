# Reorgs and Finality

The 128-block reorg story. Polygon's reorg behavior is the single biggest deviation from ETH-trained indexer assumptions.

## Heads

| Head | Source | Lag | Reorg likelihood |
|---|---|---|---|
| **Bor head** | `eth_blockNumber` on `bor` | ~0 s | Real, **occasionally deep** |
| **Bor finalized** | `eth_getBlockByNumber("finalized")` on bor (post-Bhilai) | ~256 blocks {{unsourced}} | Effectively never |
| **Checkpoint-included** | Heimdall checkpoint on L1, plus L1 confirmation | ~30 min – a few hours | Only on L1 reorg + heimdall collusion |

Polygon's `bor` exposes `safe` and `finalized` block tags following the Bhilai upgrade, mapped to depth-based heuristics from the heimdall span/checkpoint state {{unsourced: confirm exact mapping}}.

## Why deep bor reorgs happen

Several causes interact:

### 1. Sprint producer absence

If a sprint's assigned producer is offline, the sprint is skipped. When the next sprint starts, the new producer can extend either the empty (skipped) chain or, if the absent producer comes back online, may include some of those late-arriving blocks. This produces head competition over **sprint boundaries** — and a sprint is 64 blocks.

### 2. Network partition between regions

Validators may diverge briefly. When connectivity heals, the longer-by-difficulty chain wins, but in practice both sides have produced 30–100+ blocks during the partition.

### 3. Validator misbehavior / spam

Historical incidents have produced reorgs of 50–150 blocks in the wild. Documented in Polygon's incident reports {{unsourced: link the canonical reorg incident retrospective}}.

The often-cited "128-block reorg" comes from observed historical maximum reorg depths in the wild, not from a protocol parameter. **There is no hard upper bound on bor reorg depth** short of the next checkpoint.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics, but tag rows | Bor head |
| Token transfers, contract state | Bor head + 256 blocks {{unsourced: justify depth}} OR `finalized` block tag |
| Bridge accounting, audit-grade ledgers | Checkpoint-included on L1 |

A typical production indexer for Polygon uses **256-block confirmation depth** as a reasonable safety margin against the historical-max ~128-block reorgs. This is 2× larger than ETH-trained indexers expect.

For audit-grade systems (exchanges crediting deposits): wait for **checkpoint inclusion on L1**, not just bor depth.

## Reorg detection

Same shape as ETH/OP: poll `bor` for new heads, on each new head compare parent hash to storage, walk back if mismatch. The difference is **expected reorg depth**: provision the walk-back to handle 200+ blocks gracefully without crashing on memory or DB transaction size.

```
poll bor newHeads:
  if storage.head.hash != head.parent_hash:
     walk_back_until_match(max_depth=512)  // generous
     re-ingest from common ancestor
```

If your walk-back hits the max depth without finding a common ancestor, you are likely at a heimdall-checkpoint reorg boundary — escalate to a full re-sync from a known-good checkpoint.

## Checkpoint-based finality

A bor block is **strongly final** when:

1. It is included in a heimdall checkpoint.
2. That checkpoint's L1 tx is included in an Ethereum block.
3. That Ethereum block has finalized (post-Merge: ~12.8 min).

Total: typical finality time is ~30–60 minutes from L2 production, dominated by the heimdall checkpoint cadence.

To verify a bor block is checkpointed:

```
1. Query heimdall: GET /checkpoints/list  (or watch L1 RootChain events)
2. For each checkpoint, get [start_block, end_block, root_hash]
3. A bor block N is checkpointed iff some checkpoint's [start, end] contains N
```

For indexers, maintain a `bor_checkpoints` table: `(checkpoint_id, start_block, end_block, l1_tx, l1_block, root_hash)` and join your bor data against it for the canonical "is this block final" answer.

## Finality from `eth_getBlockByNumber("finalized")` on bor

Post-Bhilai upgrade, bor exposes `safe` and `finalized` block tags. These are computed from heimdall span and checkpoint state and are usable for most purposes. They lag bor head by hundreds of blocks but are **not** as strong as L1-checkpointed finality.

For production indexers, prefer **L1-checkpoint-derived finality** for audit-grade workloads and the `finalized` block tag for general use.

## Bor reorgs cascade into state-sync timing

State syncs are injected at sprint boundaries (every 64 blocks). When bor reorgs, the **timing** of state sync injection can change — a state sync that was injected at block N might be re-injected at block N+10 in the new chain, with the same effects but a different block number.

Indexers tracking state syncs by `(block_number, log_index)` see them appear, disappear, and reappear during deep reorgs. Use the **state-sync ID** (a heimdall-issued monotonic ID) as the canonical key, not the bor block number.

## Inactivity / heimdall halts

If heimdall halts (Tendermint BFT requires 2/3+ online), bor can continue producing blocks **for the rest of the current span** but cannot transition to a new span (no producer election can occur). Eventually production halts.

Checkpoints stop being posted; bor's L1 finality stops advancing.

This has happened in production (briefly) {{unsourced: cite incident}}. Indexers should not assume continuous heimdall liveness.
