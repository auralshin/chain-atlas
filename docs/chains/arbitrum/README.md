# Arbitrum

**Status:** in-progress (template populated — pending review)
**Family:** EVM L2 — Optimistic rollup, Nitro stack
**Mainnet launch (Classic):** 2021-08-31 (Arbitrum One alpha) — **out of scope for this guide**
**Mainnet launch (Nitro):** 2022-08-31 — **this guide covers Nitro+ only**
**Native node clients:** [`nitro`](https://github.com/OffchainLabs/nitro) (single binary; `nitro-geth` fork is embedded)

## TL;DR for indexers

- **Optimistic rollup, fundamentally different from OP Stack.** Different tx types, different receipt fields, different L1 contract layout, different fault proof mechanism (BoLD / WAVM), different L1→L2 deposit semantics (retryable tickets, not OP-style deposits).
- **Two stacks to worry about historically.** Classic (pre-Nitro, AVM-based) and Nitro (post-2022-08-31, EVM-equivalent). This guide is **Nitro only**. Classic indexers are a separate problem.
- **Two chains in the Arbitrum family.** Arbitrum One (full L1 calldata/blobs DA) and Arbitrum Nova (AnyTrust DAC). Same Nitro stack, different DA assumptions. Most of this doc covers One; deltas for Nova called out where relevant.
- **Stylus is a contract-level concern, not an indexer-level one.** Stylus contracts execute as activated WASM but the tx interface is unchanged.

## Read order

1. [architecture.md](architecture.md) — Nitro stack, sequencer, BoLD fault proofs
2. [data-model.md](data-model.md) — Arbitrum tx types (`0x64`–`0x6A`), receipt fields, ArbOS precompiles
3. [reorgs-finality.md](reorgs-finality.md) — three-head finality, retryable lifecycle, outbox state machine
4. [rpc-surface.md](rpc-surface.md) — `arb_*`, `arbtrace_*`, what differs from `eth_*`
5. [forks-changelog.md](forks-changelog.md) — ArbOS versions, BoLD, blob DA
6. [gotchas.md](gotchas.md) — retryable tickets, L1 block number lag, two-step exits
7. [storage.md](storage.md) — Postgres + ClickHouse sketches for Arbitrum-shaped data
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **Pre-Nitro Classic.** AVM-era data is a separate decoder problem. Most indexers ignore it or import a one-time snapshot at the Nitro genesis block.
- **L3s built on Arbitrum Orbit** (XAI, ApeChain, etc.). Same Nitro stack, but each chain has its own contracts, sequencer, and config. The patterns in this doc transfer; the addresses do not.
