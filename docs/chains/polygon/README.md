# Polygon PoS

**Status:** in-progress (template populated — pending review)
**Family:** EVM sidechain — PoS with Ethereum-checkpointed finality
**Mainnet launch:** 2020-05-30 (PoS chain; project predates as Matic)
**Native token:** POL (migrated from MATIC in 2024)
**Native node clients:** [`bor`](https://github.com/maticnetwork/bor) (execution, geth fork) + [`heimdall`](https://github.com/maticnetwork/heimdall) (consensus, Tendermint fork)

## TL;DR for indexers

- **EVM-compatible sidechain, not an L2.** State is **not** settled on Ethereum. Only **checkpoints** (Merkle roots of recent state-sync events and bor blocks) are posted to L1 every ~30 minutes.
- **Two-component node.** `bor` produces EVM blocks; `heimdall` runs the Tendermint-derived consensus that elects bor producers and posts checkpoints to Ethereum.
- **Reorgs are deep.** Folklore says ~128 blocks; this is real, observed, and documented. Indexers must plan for this — naive 12-block confirmation depth is insufficient.
- **State syncs are ghost activity.** L1→L2 messages arrive as system-injected state changes with no associated tx — invisible to standard receipt scans.
- **Polygon zkEVM is a separate chain** — different stack (CDK / zk rollup), different chain ID (1101). This doc covers Polygon PoS only.

## Read order

1. [architecture.md](architecture.md) — bor + heimdall split, span/sprint, checkpoint flow
2. [reorgs-finality.md](reorgs-finality.md) — the 128-block reorg story; checkpoint-based finality
3. [data-model.md](data-model.md) — state-sync system events, MATIC→POL token migration
4. [rpc-surface.md](rpc-surface.md) — bor RPC, heimdall RPC, fetching checkpoints
5. [forks-changelog.md](forks-changelog.md) — Bhilai, Indore, Napoli, Heimdall vs Bor fork heights
6. [gotchas.md](gotchas.md) — state-sync invisibility, 128-block reorgs, MATIC→POL
7. [storage.md](storage.md) — Postgres + ClickHouse with state-sync tracking
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **Polygon zkEVM** (chain ID 1101) — different stack entirely. Belongs in the ZK family chains.
- **Polygon CDK / Supernets** — chains built on Polygon's tooling; out of scope as separate chains.
- **AggLayer** — cross-chain settlement layer Polygon is building; relevant when indexing CDK-stack chains, not PoS itself.
- **Pre-PoS Plasma chain** — Plasma version of Matic from 2018–2020. Not PoS. Long deprecated.
