# BSC (BNB Smart Chain)

**Status:** in-progress (template populated — pending review)
**Family:** EVM L1 — Proof-of-Staked-Authority (PoSA) sidechain to BNB Beacon Chain (until 2024) → standalone L1
**Mainnet launch:** 2020-09-01
**Native node clients:** [`bsc`](https://github.com/bnb-chain/bsc) (geth fork) — single binary

## TL;DR for indexers

- **EVM-equivalent at execution.** Standard ETH RPC. Standard tx types. Standard receipts.
- **PoSA consensus.** ~21 active validators per epoch, rotating from a candidate set of ~41. Validators take turns producing blocks.
- **Fast finality (BEP-126 / Plato fork).** Pre-Plato, BSC had no protocol-level finality — indexers used 15-block heuristics. Post-Plato, BSC has Casper-FFG-style finality at ~2 blocks.
- **Block time has changed.** 3 seconds from genesis through the Lorentz fork (2025-04-29), reduced toward 1.5 s thereafter. See [forks-changelog.md](forks-changelog.md) for activation timestamps from `BSCChainConfig`.
- **Validator-collusion incidents are real.** The 2022 Cube DAO / cross-chain bridge exploit and follow-on validator response events shape current operational thinking. Indexers should not assume the validator set is honest.
- **BNB Beacon Chain (BC) is deprecated.** The cross-chain layer that originally separated BC from BSC has been retired. Indexers no longer need to track BC for cross-chain state.

## Read order

1. [architecture.md](architecture.md) — PoSA, validator rotation, fast finality
2. [reorgs-finality.md](reorgs-finality.md) — pre-Plato vs post-Plato finality, validator-collusion threat model
3. [data-model.md](data-model.md) — ETH-equivalent; BSC-specific system contracts
4. [rpc-surface.md](rpc-surface.md) — standard ETH RPC + BSC-specific extensions
5. [forks-changelog.md](forks-changelog.md) — BEP timeline (Moran, Plato, Hertz, Tycho, Pascal, Lorentz, Maxwell)
6. [gotchas.md](gotchas.md) — fast-finality caveats, validator-set incidents, beacon chain history
7. [storage.md](storage.md) — Postgres + ClickHouse with high-throughput sizing
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **opBNB** — OP Stack L2 on BSC. Belongs in the EVM L2 family; documented separately if at all.
- **BNB Greenfield** — decentralized storage chain. Not an EVM-equivalent transaction chain; different indexer entirely.
- **BNB Beacon Chain (BC)** — deprecated 2024. Historical BC data exists but is read-only and out of scope here.
