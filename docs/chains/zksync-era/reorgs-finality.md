# Reorgs and Finality

zkSync Era's finality is **three-stage** at the L1 level: commit → prove → execute. Each stage gates a different guarantee.

## The three stages

For each L1 batch:

| Stage | L1 event | What's settled |
|---|---|---|
| **Commit** | `BlockCommit` | Sequencer claims this state; L1 stores the claim |
| **Prove** | `BlocksVerification` | ZK proof verifies the state transition; cryptographic guarantee |
| **Execute** | `BlockExecution` | After execution delay, batch is canonical L1-known L2 state |

Once a batch is `executed`, the state is immutable on L1 — barring an L1 reorg of the execute tx itself (rare, requires deep L1 reorg).

The delay between `prove` and `execute` is a **safety window** allowing emergency response (governance can pause execution if a bug is suspected). Currently configured at ~24 hours {{unsourced: confirm exact delay}}.

## Finality heads for indexers

| Head | Means | Reorg likelihood |
|---|---|---|
| L2 head (sequencer) | Latest L2 block, not yet on L1 | Sequencer rewinds (rare in practice on Era) |
| L1-committed | In a `BlockCommit`-emitted batch | Only on L1 reorg of the commit tx |
| L1-proven | In a `BlocksVerification`-confirmed batch | Cryptographically secure |
| L1-executed | In a `BlockExecution`-finalized batch | Effectively never |

For most analytics indexers, the L2 head + reorg detection is sufficient; the L1 stages are for bridge / withdrawal accounting.

## Reorg sources

### 1. Sequencer rewinds

The Era sequencer is single-operator and rarely rewinds in practice — but it can. Subscribe to `eth_subscribe("newHeads")` and check parent-hash continuity on each new head.

In practice, sequencer-feed reorgs on Era are uncommon (not zero), and depths are small (1–3 L2 blocks).

### 2. L1 reorg invalidating commit / prove / execute

If L1 reorgs past one of the batch's L1 txs, the state of that batch's status is reset to the previous stage. Indexers tracking finality state must subscribe to L1 reorg events and reconcile.

### 3. Successful invalidity proof (theoretical)

A SNARK is mathematically guaranteed to be valid; an invalid state transition cannot produce a verifying proof. **There is no fault proof challenge.** In theory, the proof system itself could be broken (cryptographic break), but this is extraordinary.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics | L2 head |
| Token transfers, contract state | L2 head (Era's sequencer rewinds are rare and shallow) |
| Bridge accounting / audit-grade | L1-executed |

For bridge accounting specifically, withdrawals are claimable by users **only after** the batch reaches `execute`. So an indexer's "your withdrawal" UI shows:

```
1. Burn on L2 (L1Messenger event)
2. Batch committed to L1 (BlockCommit)
3. Batch proven on L1 (BlocksVerification)
4. Batch executed on L1 (BlockExecution)  ← user can now finalize on L1
5. User calls L1 finalize (transfers funds)
```

Stages 2–4 are the wait. Total time: hours to ~24 hours (commit/prove cadence + execute delay).

## L2 reorg handling

Same shape as ETH/OP: subscribe to `newHeads`, compare parent hash, walk back on mismatch. Era's sequencer is conservative; provision a small walk-back (~10 L2 blocks) for safety.

## Inactivity / sequencer downtime

If the sequencer is offline:
- L2 head stops advancing.
- L1 batch posting stops.
- L1-committed / proven / executed states stop advancing.
- After enough delay, **anyone can submit a priority op** via L1 `Mailbox`, which the protocol guarantees will be included once the sequencer comes back. There is **no force-include path that bypasses the sequencer entirely** in the current system {{unsourced: confirm — decentralization roadmap mentions this}}.

In practice, Era has had brief sequencer outages. Indexers should tolerate gaps in head advancement without crashing.

## Why "no fault proofs" is not "no exit"

ZK rollups don't have fault proofs because invalid state can't be proven. But the **emergency response** to a buggy upgrade or proof-system break is governance pause + manual intervention, not user-driven challenge. Users in such a scenario rely on the system's recovery, not on individual exits.

For an indexer, this means: if Era's L1 contracts go into emergency-pause mode, the L2 head may continue advancing while L1 batch operations halt. Reconcile carefully when the system resumes.
