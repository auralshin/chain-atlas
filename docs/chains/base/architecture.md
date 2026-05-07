# Architecture

**Architecturally identical to Optimism.** OP Stack components, derivation pipeline, fault proofs, finality model — all shared.

Read [optimism/architecture.md](../optimism/architecture.md). Everything in that file applies to Base, with the deltas below.

## Base-specific deltas

| Aspect | Value |
|---|---|
| Chain ID | 8453 |
| L1 batch inbox | Base-specific address (different from OP Mainnet) — see [superchain-registry](https://github.com/ethereum-optimism/superchain-registry) |
| L1 portal address | Base-specific |
| L1 SystemConfig | Base-specific |
| Sequencer | Coinbase Cloud |
| L1 batcher | Coinbase Cloud |
| L1 proposer | Coinbase Cloud {{unsourced: confirm operator}} |

To resolve the actual L1 contract addresses for Base at runtime, query `op-node`'s `optimism_rollupConfig` or read from the superchain-registry's [`base/mainnet.toml`](https://github.com/ethereum-optimism/superchain-registry).

## Why Base is "born Bedrock"

OP Mainnet launched as OVM 1.0 in 2021 and went through OVM 2.0 before Bedrock. Base launched **after** Bedrock was live, so its genesis is a Bedrock-format block — no legacy translation needed.

Indexer consequence: a Base indexer can use a single Bedrock-era decoder for the entire chain history. No snapshot import, no fork at genesis. This is the **simplest** OP Stack chain to index from genesis.

## Block production

Same as Optimism: 2-second L2 block time, single sequencer, batches posted to L1.

## Fault proofs on Base

Base's fault proof rollout is on its own timeline, **separate** from OP Mainnet's. Both chains use the same `DisputeGameFactory`/Cannon mechanism, but activation dates differ.

Verify the current state at [Coinbase's L2 status page](https://status.base.org/) {{unsourced: confirm URL}} and against the superchain-registry.

## Where Base data lives

Same split as Optimism: `op-geth` for L2 EL data, `op-node` for L2 CL/finality, L1 for batch inclusion + output roots + bridge events. The only difference is the L1 contract addresses to watch.
