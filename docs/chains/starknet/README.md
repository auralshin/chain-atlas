# StarkNet

**Status:** in-progress (template populated — pending review)
**Family:** ZK rollup — **Cairo VM, not EVM**
**Mainnet launch:** 2021-11 (alpha) → general availability 2022
**Native node clients:** [`pathfinder`](https://github.com/eqlabs/pathfinder) (Equilibrium, Rust), [`juno`](https://github.com/NethermindEth/juno) (Nethermind, Go), [`papyrus`](https://github.com/starkware-libs/papyrus) (StarkWare, Rust). Sequencer software is closed-source StarkWare.

## TL;DR for indexers

- **Treat as a non-EVM chain that happens to settle on Ethereum.** Cairo VM has nothing in common with EVM at the bytecode or RPC level.
- **Native account abstraction.** No EOAs. Every account is a Cairo contract. Validation logic is custom per account.
- **Custom JSON-RPC schema.** `starknet_*` methods, not `eth_*`. Field types are different (251-bit field elements; hex-encoded as `0x...`).
- **Tx types are entirely different from Ethereum:** `INVOKE`, `DECLARE`, `DEPLOY_ACCOUNT`, `L1_HANDLER`. No legacy / EIP-1559 / EIP-2930.
- **Contract deployment is a two-step process:** DECLARE the class (register the contract code), then DEPLOY (instantiate at an address). Different from `CREATE` / `CREATE2`.
- **Indexer code shares almost nothing with EVM chains** beyond "watch L1 for state commitments." Plan to write StarkNet ingestion from scratch, not adapt an EVM indexer.

## Read order

1. [architecture.md](architecture.md) — sequencer, prover, Cairo VM, Sierra/CASM, L1 settlement
2. [data-model.md](data-model.md) — felt252, tx types, events, classes vs contracts
3. [reorgs-finality.md](reorgs-finality.md) — PENDING → ACCEPTED_ON_L2 → ACCEPTED_ON_L1
4. [rpc-surface.md](rpc-surface.md) — `starknet_*` JSON-RPC spec
5. [forks-changelog.md](forks-changelog.md) — Cairo 0 → Cairo 1/2, V0.10/V0.11/V0.12/V0.13 protocol versions
6. [gotchas.md](gotchas.md) — class hashes vs addresses, fee in STRK or ETH, AA validation phases
7. [storage.md](storage.md) — Postgres + ClickHouse with felt252-aware schemas
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **Pre-Cairo-1 contracts** — Cairo 0 contracts still exist on chain but cannot be newly deployed. Indexers covering historical data must handle Cairo 0 ABI conventions differently.
- **Madara** — StarkNet stack used to build appchains (Madara-style L3s). Same patterns; chain-specific config differs.
- **Starknet's `kakarot` zkEVM** — a Cairo-implemented EVM running on StarkNet. Out of scope; if you need to index it, treat as a separate EVM chain that happens to live inside StarkNet.
