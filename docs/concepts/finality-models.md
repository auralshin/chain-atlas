# Finality models

**Status:** stub — to be expanded once a non-EVM chain doc is also written, so the abstraction has at least two concrete instances.

A taxonomy of finality across chains. The most common indexer bug across chains is treating sequencer confirmation (or block inclusion) as equivalent to finality.

## Categories (preliminary)

| Model | Examples | Indexer-relevant property |
|---|---|---|
| Probabilistic | Bitcoin, pre-Merge Ethereum, BSC | "Final" is a confidence threshold (N confirmations); never absolute |
| Single-slot | Tendermint / CometBFT (Cosmos chains) | A committed block is final; reorgs of committed blocks impossible by protocol |
| Two-stage justification | Casper FFG (post-Merge Ethereum) | `head` → `safe` → `finalized`, with finality typically 2 epochs out |
| Sequencer-confirmed (L1-finalized) | OP Stack, Arbitrum Nitro, ZK rollups | L2 block is "soft" once sequencer signs; "final" only after L1 batch finalizes |
| Object-level | Sui (owned vs shared object paths) | Owned-object txs finalize via fast path; shared-object via consensus |

## Per-chain finality

Each chain doc has a `reorgs-finality.md` file that restates the model in chain-specific terms. The full cross-reference table will be added here once 3+ chains are deeply documented.

- [Ethereum](../chains/ethereum/reorgs-finality.md)
- ... (others to be added)

## Common pitfalls

- **Treating L2 sequencer confirmation as final** — sequencers can re-sequence within a window or omit txs from the batch. Only L1 batch finalization is final for a rollup.
- **Treating Tendermint commit as same as Ethereum finalization** — they're both "final", but the failure modes are different. Tendermint halts; Eth enters inactivity leak.
- **Counting confirmations on a PoS chain** — N-confirmations is a PoW heuristic. On PoS, confirmations are not the right axis; finality checkpoint is.

(More once additional chains are documented.)
