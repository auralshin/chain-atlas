# Reorgs and Finality

StarkNet's three-stage finality is exposed **explicitly** as a `status` field on blocks and txs — much cleaner than Ethereum's implicit confirmation depth.

## The three stages

| Stage | Means | Reorg likelihood |
|---|---|---|
| **`PENDING`** | In the pending block being built; not yet sealed | Real — anything in pending can be reordered or dropped |
| **`ACCEPTED_ON_L2`** | Sealed in an L2 block by the sequencer | Sequencer rewinds (rare in practice) |
| **`ACCEPTED_ON_L1`** | Proof verified on L1 | Effectively never |

Indexers query block / tx status directly:

```
GET starknet_getBlockWithTxs(block_id)
  → { ..., "status": "ACCEPTED_ON_L1" | "ACCEPTED_ON_L2" | "PENDING" }
```

No need to compare confirmation depth — the chain tells you the stage.

## The PENDING block

Unique to StarkNet (relative to Ethereum). A "pending" block is the **current open block being built** by the sequencer. It contains txs that have been accepted but not yet sealed.

Indexer impact:
- `block_id = "pending"` returns the in-progress block.
- Querying `starknet_getBlockNumber()` returns the latest **sealed** (ACCEPTED_ON_L2) block, **not** the pending one.
- A tx in the pending block has `block_number = null` (not yet assigned), `block_hash = null`.

For low-latency indexing (live dashboards, mempool-equivalent), poll the pending block. Treat its data as ephemeral.

## Reorg sources

### 1. Sequencer rewinds the unsealed pending block

Common — the sequencer is allowed to drop or reorder pending txs.

### 2. Sequencer rewinds an ACCEPTED_ON_L2 block

Rare in practice. The StarkWare-operated sequencer is conservative. When it happens, depths are typically 1–3 L2 blocks.

### 3. L1 reorg invalidates the proof tx

If L1 reorgs past the `Starknet Core.updateState` tx that finalized a batch, the L2 batch's status reverts from ACCEPTED_ON_L1 back to ACCEPTED_ON_L2 until re-finalized. Rare; requires deep L1 reorg.

### 4. Successful invalidity proof

Cryptographically impossible — STARKs cannot verify against invalid state transitions. There is no fault-proof challenge.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics, live dashboards | `PENDING` (mark explicitly) |
| Token transfers, contract state | `ACCEPTED_ON_L2` |
| Bridge accounting, audit-grade ledgers | `ACCEPTED_ON_L1` |

Most indexers use `ACCEPTED_ON_L2` as the canonical commit point. The L2 sequencer rewinds are rare enough that this works for almost all workloads.

## Reorg detection

`starknet_*` does not emit explicit reorg events. Detect by polling:

```
1. Subscribe to starknet_subscribeNewHeads (WebSocket; if supported)
   OR poll starknet_blockNumber every few seconds.
2. On each new head, check parent_hash continuity against your storage.
3. On mismatch, walk back until parent matches. Re-ingest from common ancestor.
```

Provision walk-back ~10 blocks; deeper rewinds are exceptional.

For the L1-finalized stage: subscribe to L1 `Starknet Core.LogStateUpdate` events; on each, mark the corresponding L2 block range as ACCEPTED_ON_L1.

## L2 batch / proof cadence

A single L1 proof submission covers many L2 blocks (the prover batches L2 blocks into a single proof). For indexers:
- An L2 block can be ACCEPTED_ON_L2 for hours before becoming ACCEPTED_ON_L1.
- The lag varies with prover throughput and gas conditions.

## Inactivity / sequencer downtime

If the sequencer is offline:
- `PENDING` block stops accepting new txs.
- `ACCEPTED_ON_L2` head stops advancing.
- Existing ACCEPTED_ON_L2 blocks continue to advance to ACCEPTED_ON_L1 if proofs are already submitted.
- After enough delay, force-include via L1 messaging is possible {{unsourced: confirm — StarkNet's force-include path is via L1 message queue}}.

In practice, brief sequencer outages have happened during major upgrades. Indexers should not crash on stale heads.

## Withdrawals

Withdrawal flow: L2 contract emits `send_message_to_l1` → L2 batch proven on L1 → user calls `consumeMessageFromL2` on L1 to release funds.

Indexers track:
1. **L2 side**: receipt's `messages_sent` array.
2. **L1 stage**: when the batch reaches ACCEPTED_ON_L1.
3. **L1 consume**: `Starknet Core.LogConsumedMessage` event (or similar).

Total time from emission to claimable on L1: hours (proof cadence). No challenge window.
