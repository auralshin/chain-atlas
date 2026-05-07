# Base

**Status:** in-progress (template populated — pending review)
**Family:** EVM L2 — OP Stack optimistic rollup
**Mainnet launch:** 2023-08-09 (public mainnet)
**Chain ID:** 8453
**Native node clients:** `op-geth` + `op-node` (same as Optimism)

## TL;DR for indexers

- **Base is an OP Stack chain.** The architecture, data model, derivation pipeline, deposit-tx semantics, fault-proof flow, and RPC surface are all **identical to Optimism**.
- **Read [optimism/](../optimism/) first.** This folder documents only the Base-specific deltas.
- **Born post-Bedrock.** Unlike OP Mainnet, Base has no OVM 1.0/2.0 era — the entire chain is post-Bedrock from genesis. Indexers do not need a pre-Bedrock decoder.
- **Sequenced by Coinbase.** Single sequencer operated by Coinbase Cloud.

## Read order

If you've already read the Optimism docs, the only Base files with substantive content are:

1. [forks-changelog.md](forks-changelog.md) — Base-specific activation timestamps; same upgrade names, different dates.
2. [gotchas.md](gotchas.md) — Coinbase-as-sequencer specifics; chain-id and L1-contract address differences.
3. [storage.md](storage.md) — what changes from the Optimism schema (almost nothing).
4. [references.md](references.md) — Base-specific operational links.

The rest are short deltas that point to `optimism/`.

## What's *not* different

| Aspect | Status |
|---|---|
| Block time (2 s) | Same as Optimism |
| Tx types (incl. deposit `0x7E`) | Same |
| Receipt fields and L1-fee formulas | Same |
| Predeploys at `0x4200…00xx` | Same addresses, same bytecode (post-upgrade) |
| Three-head finality model | Same |
| 7-day withdrawal window | Same |
| Fault-proof flow | Same (different rollout timeline) |

## What *is* different

| Aspect | Base | Optimism |
|---|---|---|
| Chain ID | 8453 | 10 |
| Genesis | 2023-07-13 (testnet); 2023-08-09 public | 2021-12 (legacy); 2023-06-06 Bedrock |
| Pre-Bedrock data | None | Two legacy eras |
| Sequencer operator | Coinbase Cloud | Optimism Foundation |
| L1 contract addresses | Base-specific deployments | OP-specific deployments |
| Fault-proof rollout | Scheduled separately | See OP timeline |
| Fee revenue split | **greater of (2.5% sequencer revenue) or (15% net on-chain sequencer revenue)** to OP Collective; rest to Coinbase ([2023-08 agreement](https://www.optimism.io/blog/welcoming-base)) | OP Collective |
