# Ethereum

**Status:** in-progress (template target — pending user review)
**Family:** EVM L1
**Mainnet launch:** 2015-07-30 (Frontier)
**Native execution clients:** geth, reth, erigon, nethermind, besu
**Native consensus clients:** lighthouse, prysm, teku, nimbus, lodestar, grandine

## Why index this chain

Ethereum is the largest EVM L1 and the settlement layer for every rollup family in this repo. An indexer that gets Ethereum right inherits most of the patterns it needs for L2s and EVM sidechains; an indexer that gets Ethereum wrong is silently wrong on every other EVM chain too. This doc is the **template target** — every other chain doc is a delta against this one.

## Indexer's TL;DR

- **Five transaction types live as of Pectra** (`0x00` legacy → `0x04` set-code). A naive `Tx::decode` written against legacy txs silently mis-parses the rest. See [data-model.md](data-model.md).
- **Finality is not "tx is in a block".** Finality is two-epoch (~12.8 min) under normal conditions, and can be hours under non-finalizing conditions. See [reorgs-finality.md](reorgs-finality.md).
- **The block your indexer sees was usually built by an external builder, not the proposer.** PBS / mev-boost is now the dominant block-production path. See [architecture.md](architecture.md).
- **Trace methods are not portable across clients.** geth, reth, erigon, nethermind, and besu support different subsets. See [rpc-surface.md](rpc-surface.md).
- **Blob data is not in JSON-RPC.** Blobs (EIP-4844) are retrieved from the beacon node and only retained ~18 days. Indexers that need historical blob bodies must persist them at ingest time.

## Doc map

- [Architecture](architecture.md) — consensus, finality, PBS, where data lives
- [Data model](data-model.md) — blocks, tx types 0–4, receipts, logs, withdrawals, blobs
- [RPC surface](rpc-surface.md) — JSON-RPC, trace methods × client matrix, beacon API
- [Reorgs and finality](reorgs-finality.md) — empirical depth, recovery
- [Forks and upgrades](forks-changelog.md) — Frontier through Fusaka, Glamsterdam scheduled
- [Storage](storage.md) — PG and CH schema sketches
- [Skeleton](skeleton.md) — alloy-based indexer pattern
- [Gotchas](gotchas.md) — PBS, blob retention, internal calls, withdrawals are not txs
- [References](references.md) — primary sources
