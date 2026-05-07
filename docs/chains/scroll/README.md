# Scroll

**Status:** in-progress (template populated — pending review)
**Family:** ZK-EVM rollup — **bytecode-equivalent** EVM
**Mainnet launch:** 2023-10-17
**Native node clients:** [`l2geth`](https://github.com/scroll-tech/go-ethereum) (geth fork)

## TL;DR for indexers

- **Bytecode-equivalent EVM.** Unlike zkSync Era, Scroll runs **the same EVM bytecode** as Ethereum — Solidity sources compile to standard EVM bytecode, no custom compiler. Most ETH-trained indexer code works unchanged.
- **The deltas are operational**, not data-model: L1-fee receipt fields, two-stage L1 finality (commit → finalize), L1↔L2 messaging via Scroll's Messenger contracts.
- **Standard tx types only** (0/1/2; type 4 EIP-7702 if/when ported). No system tx types like OP/Arbitrum.
- **Single sequencer**, single prover currently. Decentralization roadmap exists.
- **Closest cousin: Linea.** Both are ZK-EVMs with similar architectures; cross-link in [linea/](../linea/).

## Read order

1. [architecture.md](architecture.md) — sequencer + prover + L1 contracts; two-stage finality
2. [data-model.md](data-model.md) — ETH-equivalent + L1-fee receipt fields
3. [reorgs-finality.md](reorgs-finality.md) — commit / finalize on L1; sequencer rewinds
4. [rpc-surface.md](rpc-surface.md) — standard ETH RPC + Scroll-specific fee oracle
5. [forks-changelog.md](forks-changelog.md) — Bernoulli, Curie, Darwin, Euclid, Feynman upgrades
6. [gotchas.md](gotchas.md) — fee formula changes, messaging timing
7. [storage.md](storage.md) — Postgres + ClickHouse with batch correlation
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **Scroll Sepolia / Goerli** testnets — same architecture, different L1 contract addresses.
- **L3s on Scroll** — not yet a major ecosystem at the time of writing.
