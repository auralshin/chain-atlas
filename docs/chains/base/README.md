# Base

**Status:** stub
**Family:** EVM L2 — OP Stack optimistic rollup
**Mainnet launch:** 2023-08-09 (public mainnet)
**Native node clients:** op-geth + op-node (same as Optimism)

## Anchor facts (verify in deep dive)

- OP Stack chain operated by Coinbase. Architecturally near-identical to Optimism — most of the chain doc is a delta on `optimism/`.
- Sequenced by Coinbase; settles to Ethereum L1.
- Indexer code that works for Optimism Bedrock works here with chain-id and L1-contract address changes.

## To populate

Use [template](../_template/). Cross-link aggressively to `optimism/` rather than duplicating OP Stack content.
