# <Chain Name>

**Status:** stub | in-progress | deep
**Family:** <EVM L1 | EVM L2 (rollup) | EVM L2 (sidechain) | ZK-EVM | non-EVM (Move / Solana / Cosmos / ...)>
**Mainnet launch:** <YYYY-MM-DD>
**Native node clients:** <list>

## Why index this chain

<One paragraph: what's on this chain that's worth indexing? Who cares? What's the killer use case that makes the indexer worth running?>

## Indexer's TL;DR

The 3–5 things that will surprise someone who has only indexed Ethereum:

- ...
- ...
- ...

## Doc map

- [Architecture](architecture.md) — consensus, fork choice, block production, finality model
- [Data model](data-model.md) — blocks, transactions, receipts, events, state
- [RPC surface](rpc-surface.md) — endpoints, trace methods, archive requirements
- [Reorgs and finality](reorgs-finality.md) — depth, detection, recovery
- [Forks and upgrades](forks-changelog.md) — chronological changelog, indexer impact
- [Storage](storage.md) — Postgres + ClickHouse schema sketches and tradeoffs
- [Skeleton](skeleton.md) — illustrative Rust indexer pattern
- [Gotchas](gotchas.md) — known surprises and indexer bugs
- [References](references.md) — primary sources
