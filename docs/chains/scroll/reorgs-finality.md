# Reorgs and Finality

Scroll's finality is **two-stage** at L1: commit → finalize. Simpler than zkSync Era's three-stage flow.

## Heads

| Head | Means | Reorg likelihood |
|---|---|---|
| L2 head (sequencer) | Latest L2 block, not yet on L1 | Sequencer rewinds (rare) |
| L1-committed | In a `CommitBatch`-emitted L1 batch | Only on L1 reorg |
| L1-finalized | In a `FinalizeBatch`-confirmed L1 batch | Effectively never |

Scroll's `l2geth` exposes `eth_getBlockByNumber("safe"/"finalized")` mapped to the L1-committed and L1-finalized heads {{unsourced: confirm exact mapping}}.

## Reorg sources

### 1. Sequencer rewinds

Single-sequencer rollup. Sequencer can rewind its unsafe head; in practice this is rare on Scroll (small operator, conservative rollout).

### 2. L1 reorg invalidating commit / finalize

If L1 reorgs past one of the batch's L1 txs, the affected batch reverts to the previous stage. Indexers must reconcile.

### 3. Successful invalidity proof

Cryptographically impossible — a SNARK cannot verify against an invalid state transition. There is no fault-proof challenge.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics | L2 head |
| Token transfers, contract state | L2 head (Scroll sequencer rewinds are rare) |
| Bridge accounting / audit-grade | L1-finalized |

For bridge accounting specifically, withdrawals are claimable on L1 only after the batch reaches `finalized`. Total time from L2 burn to L1 release: on the order of **hours** (commit cadence + proof generation time + L1 inclusion).

## Reorg detection

Subscribe to `eth_subscribe("newHeads")` on L2; compare parent hash on each new head. Walk back on mismatch.

For the L1 stages: subscribe to `ScrollChain.CommitBatch` and `ScrollChain.FinalizeBatch` events on L1. Maintain a batch table; advance per-block status as events arrive.

## Sequencer downtime

Scroll has a force-include mechanism via `EnforcedTxGateway` {{unsourced: confirm name}} — if the sequencer is offline beyond a threshold, anyone can submit txs that bypass the sequencer via L1.

In practice, brief sequencer outages have happened. Indexers should not block on L2 head advancement; use L1 batch status for liveness.

## Inactivity / governance pause

If Scroll's L1 contracts go into emergency pause (governance-coordinated response to a bug), L2 production may continue but L1 batch posting halts. Indexers should detect prolonged "L2 head advances but no L1 commits" as a special state.
