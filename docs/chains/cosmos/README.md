# Cosmos (family)

**Status:** stub
**Family:** non-EVM family — Tendermint / CometBFT consensus + ABCI app chains
**Native node clients:** Per-chain (gaiad for Cosmos Hub, osmosisd for Osmosis, etc.)

## Anchor facts (verify in deep dive)

- This is a **family**, not a single chain. Each Cosmos chain is its own ABCI app. Indexer architecture should reflect that — one adapter per chain, not one Cosmos adapter.
- Instant finality — Tendermint/CometBFT commits a block once 2/3+ validators vote. No reorgs of committed blocks.
- IBC creates cross-chain message flow; indexers covering multiple chains need to correlate IBC packets.
- Events are emitted from message handlers and proto-encoded; ABCI events are key-value pairs, not structured logs.
- Tendermint RPC and gRPC are the standard interfaces; per-chain indexers (e.g. CosmWasm contracts) layer on top.

## To populate

Use [template](../_template/). The doc covers the family; per-chain quirks (Hub vs Osmosis vs Injective) get sub-pages if depth warrants it.
