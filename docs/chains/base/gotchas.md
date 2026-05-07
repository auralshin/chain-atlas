# Gotchas

All gotchas in [optimism/gotchas.md](../optimism/gotchas.md) apply to Base. The list below adds Base-specific ones.

## Higher tx volume changes operational defaults

- **Default `eth_getLogs` block ranges** that work fine on OP Mainnet may time out on Base for log-heavy contracts. Tune block-range chunks per provider.
- **WebSocket subscription backpressure** is a real failure mode under burst load (e.g. NFT mints, popular memecoin launches). Buffer aggressively in the subscriber, or fall back to polling.
- **Mempool** behavior differs — Base's public mempool is currently only the sequencer's view; private orderflow patterns differ from OP Mainnet. Indexers tracking pending txs should not assume the same mempool semantics.

## Coinbase as sole sequencer

This is operational, not protocol-level, but it changes the threat model:

- **Censorship resistance** is degraded relative to a multi-sequencer setup. Force-inclusion via L1 deposit still works (same as Optimism).
- **Sequencer downtime** is correlated with Coinbase's infrastructure incidents — including incidents that affect the centralized exchange. Plan for these in alerting.

## No pre-Bedrock data

Some indexer libraries implicitly assume an OP Stack chain has a pre-Bedrock era and try to import a snapshot at "Bedrock activation." On Base, the genesis block IS the Bedrock-format start — there's nothing before it. Skip the snapshot import logic entirely.

## L1 contract addresses differ

Hardcoding OP Mainnet's `OptimismPortal` / `L2OutputOracle` / `DisputeGameFactory` addresses is a common copy-paste error when extending an OP indexer to Base. Always:

```rust
let cfg = op_node.optimism_rollup_config().await?;
let portal = cfg.l1_system_config.optimism_portal;  // Base-specific
```

Or pin from the superchain-registry. **Never** copy addresses across OP Stack chains.

## Withdrawal proving requires Base-specific output roots

A user withdrawing from Base must call `proveWithdrawalTransaction` on **Base's** `OptimismPortal` (on L1), with an output root posted by **Base's** `DisputeGameFactory`. An indexer building "your withdrawals" UIs must scope all queries to Base's L1 contracts.

## Fee revenue goes to Coinbase, partially to OP Collective

Per the [August 2023 Base ↔ OP Collective agreement](https://www.optimism.io/blog/welcoming-base), Base contributes the **greater of (2.5% of total sequencer revenue) or (15% of net on-chain sequencer revenue)** to the Optimism Collective. Net = L2 transaction revenue minus L1 data submission costs. In return, Base received the right to earn up to ~118M OP tokens over six years (capped at 9% of votable supply for governance purposes). The on-chain accounting for fee distribution happens at the vault level (`SequencerFeeVault` etc.) — same predeploys as Optimism, different recipients.

For indexers building revenue dashboards, the relevant data is the L2 vault withdrawal txs and their L1 destination — same code path as OP Mainnet, different addresses on the receiving end.

## Genesis-time predeploys may differ

Base's genesis allocation includes Coinbase-specific configurations (sequencer addresses, batch sender) baked into the predeploys at genesis. The bytecode is the same OP Stack bytecode; the storage is initialized differently. Indexers that decode predeploy state at genesis must read the actual values, not assume defaults.
