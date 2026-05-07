# Optimism

**Status:** in-progress (template populated — pending review)
**Family:** EVM L2 — OP Stack optimistic rollup
**Mainnet launch:** 2021-12 (OVM 1.0 alpha mainnet); **Bedrock upgrade 2023-06-06** (current data model)
**Native node clients:** [`op-geth`](https://github.com/ethereum-optimism/op-geth) (EL), [`op-node`](https://github.com/ethereum-optimism/optimism/tree/develop/op-node) (rollup CL); also [`op-reth`](https://github.com/paradigmxyz/reth) and [`op-erigon`](https://github.com/testinprod-io/op-erigon) for the EL.

## TL;DR for indexers

- **Optimistic rollup.** L2 blocks are produced by a sequencer at 2-second intervals. Final state is *derived* from L1 — the L2 chain is a function of L1 batches.
- **Three liveness states matter.** `unsafe` (sequencer-published, can vanish), `safe` (batched to L1, depends on L1 reorgs), `finalized` (L1-finalized + fault-proof-window settled).
- **Bedrock is the boundary.** Pre-Bedrock (OVM 1.0/2.0) is a different data model. Most modern indexers ignore it or import a one-time snapshot. This guide is **post-Bedrock only**.
- **Same template as Base.** Cross-link rather than duplicate when reading both.

## Read order

1. [architecture.md](architecture.md) — sequencer + derivation pipeline + fault proofs
2. [reorgs-finality.md](reorgs-finality.md) — three-state finality, L1 reorg cascades
3. [data-model.md](data-model.md) — deposit txs, L1-fee fields, predeploys
4. [rpc-surface.md](rpc-surface.md) — `optimism_*`, `rollup_*`, `engine_*`
5. [forks-changelog.md](forks-changelog.md) — Bedrock → Canyon → Delta → Ecotone → Fjord → Granite → Holocene → Isthmus
6. [gotchas.md](gotchas.md) — surprises that break ETH-trained indexers
7. [storage.md](storage.md) — Postgres + ClickHouse sketches
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources
