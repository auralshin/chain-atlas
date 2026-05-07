# Scroll

**Status:** stub
**Family:** ZK-EVM rollup — bytecode-equivalent EVM
**Mainnet launch:** 2023-10-17
**Native node clients:** l2geth (geth fork)

## Anchor facts (verify in deep dive)

- Bytecode-level EVM equivalence — Solidity contracts deployed identically work without modification.
- Indexer code written for Ethereum largely works; deltas are batch-posting, fee model, and L1 finality watch.
- ZK proofs do not affect indexer logic directly, but the prover lag affects the "finalized on L1" timeline.

## To populate

Use [template](../_template/). Cross-link to `linea/` and `optimism/` where they overlap.
