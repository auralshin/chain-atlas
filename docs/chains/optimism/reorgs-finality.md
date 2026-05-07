# Reorgs and Finality

The most load-bearing file in this chain doc. Misunderstanding L2 finality is the single biggest source of indexer bugs in the OP Stack ecosystem.

## The three heads

`op-node` exposes them via `optimism_syncStatus`:

```json
{
  "current_l1": { "hash": "0x…", "number": 123 },
  "current_l1_finalized": { … },
  "head_l1": { … },
  "safe_l1": { … },
  "finalized_l1": { … },
  "unsafe_l2": { "hash": "0x…", "number": 9000 },
  "safe_l2": { "hash": "0x…", "number": 8997 },
  "finalized_l2": { "hash": "0x…", "number": 8400 }
}
```

| L2 head | Source | Lag from real time | Reorg likelihood |
|---|---|---|---|
| `unsafe_l2` | Sequencer gossip | ~0 s | Real, see below |
| `safe_l2` | L1 batch inclusion | ~30 s – 10 min | Only on L1 reorg |
| `finalized_l2` | L1 finalization + (post-fault-proofs) dispute resolution | ≥ 12.8 min, often hours | Effectively never |

## Reorg sources, ranked

### 1. Sequencer rewinds the unsafe head

The sequencer can rewrite its own `unsafe` chain at will. Common reasons:
- Mempool was front-run, sequencer re-ordered.
- Sequencer crashed and rebuilt from a slightly older state.
- Sequencer software bug.

This is the dominant source of "reorgs" on L2 in practice. **An indexer that commits on the unsafe head will see its data change.** Recommended: shadow-write unsafe data into a hot table, promote to canonical only when `safe_l2` advances past it.

### 2. L1 reorg invalidates a batch

If L1 reorgs past the block that included an `op-batcher` transaction, the batch disappears. `op-node` re-runs derivation, often producing a *different* L2 chain because:
- Batch ordering changed.
- Different deposits were included.
- L1 origin per L2 block shifted.

The L2 reorg depth is bounded by **how many L2 blocks were derived from the now-orphaned L1 range**, which can be substantial — at 2-second L2 blocks, a single 12-second L1 slot represents ~6 L2 blocks.

### 3. Sequencer equivocation

Cryptographically detectable: same L2 block number, different content, both signed by the sequencer. Today this would result in social/governance response; in a decentralized-sequencer future it would result in slashing. Rare in practice, but indexers should be prepared to detect it (track sequencer signatures per block).

### 4. Fault proof challenge succeeds (theoretical)

A successful challenge changes the canonical output root posted to L1. To date no honest fault proof challenge has overturned a posted output on Optimism Mainnet {{unsourced: verify}}. If it happened, derivation would not change — but downstream contracts trusting the output root would.

## Recommended commit policy

For most indexers:

| Use case | Commit on |
|---|---|
| Real-time analytics dashboards | `unsafe_l2`, but tag rows with head; rewrite when `safe_l2` advances |
| Token transfer history, contract state | `safe_l2` |
| Bridge accounting, audit-grade ledgers | `finalized_l2` |

**Never** commit on `unsafe_l2` to a system that downstream consumers treat as final (e.g. an exchange's deposit-credit pipeline).

## Reorg detection

`op-node` emits no explicit reorg event. Detect by polling `optimism_syncStatus` and comparing the heads to your storage:

```
loop:
  ss = optimism_syncStatus()
  if storage.head(safe).hash != ss.safe_l2.hash:
     → walk back until parent matches, rewind, re-ingest
  if storage.head(unsafe).hash != ss.unsafe_l2.hash:
     → rewind unsafe shadow only
```

This is identical in shape to ETH reorg handling, but the depth distribution is different:
- L2 unsafe reorgs: usually 1–3 L2 blocks, occasionally larger after a sequencer hiccup.
- L2 safe reorgs: only when L1 reorgs — then bounded by L1 reorg depth × (L1 slot / L2 slot) ≈ 6× L1 depth.

## Finality, withdrawals, and the 7-day window

Two distinct 7-day windows have existed at different times:

1. **Withdrawal challenge window** (always present in OP Stack): a withdrawal proven on L1 must wait 7 days before it can be `finalized()` and the funds claimed. Independent of fault proof maturity.
2. **Output root challenge window** (post-fault-proofs only): the output root that anchors a withdrawal must itself survive a 7-day dispute game on L1.

Pre-fault-proofs, output roots were trusted (Optimism Foundation could overrule via `Guardian`). Post-fault-proofs, output roots are trust-minimized but the timing is the same to the user.

For an indexer tracking bridge state:

```
L2 burn (L2ToL1MessagePasser.MessagePassed)
  └─► output root posted on L1 (DisputeGameFactory or L2OutputOracle)
       └─► challenge window expires (7 days)
            └─► user calls L1 prove + L1 finalize
                 └─► funds released
```

Track every stage. Users will ask "where's my withdrawal" and the answer is one of those four states.

## Inactivity / sequencer downtime

If the sequencer goes offline:
- `unsafe_l2` stops advancing (no new blocks).
- `safe_l2` continues to advance up to whatever was already batched.
- After enough L1 blocks pass without a batch, **anyone** can submit a force-inclusion deposit through `OptimismPortal` to push transactions into the next batch's deposit list. The L2 will then advance from those force-included txs.

In practice, indexers should not block on `unsafe_l2` advancement. Use `safe_l2` for liveness checks.
