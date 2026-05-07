# Architecture

OP Stack post-Bedrock. Two coupled processes: a **sequencer** that produces L2 blocks fast, and a **derivation pipeline** that reconstructs the canonical L2 chain deterministically from L1 data.

## Components

| Component | Role | Run by |
|---|---|---|
| `op-node` | Rollup-CL: drives derivation, talks to L1, feeds engine API to `op-geth` | Every L2 node |
| `op-geth` (or `op-reth`/`op-erigon`) | Execution layer: block-by-block state transitions | Every L2 node |
| `op-batcher` | Posts L2 batches to L1 (calldata pre-Ecotone, blobs post-Ecotone) | Sequencer operator only |
| `op-proposer` | Posts output roots to L1 (`L2OutputOracle` pre-fault-proofs, `DisputeGameFactory` post) | Sequencer operator only |
| L1 contracts | `OptimismPortal`, `L1CrossDomainMessenger`, `L1StandardBridge`, `SystemConfig`, `DisputeGameFactory`, `AnchorStateRegistry` | Deployed once on L1 |

Indexers care about the first two (run as full nodes) and the L1 contracts (events provide the deposit/withdrawal/output-root truth).

## Block production (sequencer)

| Parameter | Value |
|---|---|
| L2 block time | 2 seconds |
| Sequencer | Single, currently operated by Optimism Foundation {{unsourced: confirm current operator}} |
| Max sequencer drift | 600 seconds (L2 timestamp ahead of L1 origin) |
| Channel timeout | ~50 L1 blocks |

The sequencer publishes L2 blocks to its peers immediately (`unsafe` head). Periodically, `op-batcher` collects ranges of L2 blocks into a **channel**, encodes them with the current batch format (Brotli post-Fjord), and posts them to L1. Once an L1 batch is included and the L1 block hosting it is no longer reorgable, the corresponding L2 blocks become the `safe` head.

## Derivation pipeline

The L2 chain is **deterministic in L1 data**. Given the L1 chain and the rollup config, every full node computes the same L2 chain — no consensus among L2 validators is needed.

```
L1 blocks ──► Channels ──► Batches ──► Block attributes ──► Engine API ──► op-geth ──► L2 blocks
        (parsed by    (decompressed,    (fed via         (executes the
         op-node)      ordered)          payloadAttributes) deterministic block)
```

Two paths produce the same L2 head:
1. **Sequencer mode** — block hashes flow first via gossip, then get confirmed by derivation later.
2. **Verifier mode** — block hashes only after derivation. Slower, but trust-minimized.

## Fork choice

There is no LMD-GHOST equivalent on L2. The canonical L2 chain is whatever derivation produces from the canonical L1 chain. Reorgs on L2 happen only because:
- L1 reorged, dropping batches that defined some L2 blocks.
- The sequencer published `unsafe` blocks that derivation later contradicts.

See [reorgs-finality.md](reorgs-finality.md).

## Finality model

Three heads, all reported by `op-node`:

| Head | Meaning | Reorg requirement |
|---|---|---|
| `unsafe` | Sequencer-published, not yet on L1 | Sequencer equivocation OR successful challenge |
| `safe` | L1 batch included, L1 block hosting it is not reorging | L1 reorg deeper than the batch's L1 block |
| `finalized` | L1 block hosting the batch is finalized AND output root has settled fault proofs (when applicable) | Slashing of 1/3+ of L1 stake (≈ ETH finality) |

Pre-fault-proofs (before mid-2024 on Optimism Mainnet), `finalized` corresponded to the L1-finalized batch — the 7-day window applied only to **withdrawals** at the bridge. Post-fault-proofs, the same 7-day window now also gates the canonical output root on L1.

## Fault proofs (Cannon / FPVM)

Fault proofs went live on Optimism Mainnet in **2024-06** (`{{unsourced}}` exact date — see [forks-changelog.md](forks-changelog.md)). The mechanism:

1. `op-proposer` posts an output root to L1 via `DisputeGameFactory.create()`.
2. Anyone can challenge by creating a counter-claim and bisecting the disputed state down to a single instruction.
3. The disputed instruction is executed on L1 by the FPVM (Cannon — a MIPS interpreter for `op-program`).
4. Honest party wins the bond. After a 7-day clock + airgap, the output root is `RESOLVED_DEFENDER_WINS` and finalizes.

Indexers can use `DisputeGameFactory` events to track which output roots are *proposed* vs *finalized*.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| L2 blocks, txs, receipts, logs | `op-geth` JSON-RPC | Standard ETH RPC + extras |
| L2 unsafe/safe/finalized heads | `op-node` `optimism_syncStatus` | Authoritative source |
| Batch L1 inclusion | L1 — events from `BatchInbox` calldata or blobs | Calldata pre-Ecotone, blobs post |
| Output roots / fault games | L1 — `DisputeGameFactory`, `AnchorStateRegistry` | Post-fault-proofs |
| Deposits | L1 — `OptimismPortal.TransactionDeposited` | These appear as L2 deposit txs |
| Withdrawals | L2 — `L2ToL1MessagePasser.MessagePassed`; finalized on L1 | Two-stage (prove + finalize) |
| L1 blob data | L1 beacon node (≤ 18 days) or specialized blob archive (e.g. blobscan) | See [../../concepts/](../../concepts/) — TBD |

A complete OP Stack indexer reads **L1 + L2 in lockstep** to track derivation state and bridge accounting.
