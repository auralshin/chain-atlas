# Optimism

**Status:** stub
**Family:** EVM L2 — OP Stack optimistic rollup
**Mainnet launch:** 2021-12 (alpha mainnet); Bedrock upgrade 2023-06-06
**Native node clients:** op-geth (execution) + op-node (rollup)

## Anchor facts (verify in deep dive)

- Optimistic rollup. Sequencer confirms quickly; finality requires L1 batch inclusion + finalization.
- Bedrock changed the data model substantially — pre-Bedrock indexers do not work post-Bedrock.
- Two-component node: `op-geth` for execution, `op-node` for rollup-layer (L1 derivation, sequencer interface).
- Fault proofs were initially permissioned; status of permissionless fault proofs needs verification at deep-dive time.
- Same OP Stack as Base — the chain-doc is largely shared, with chain-specific deltas.

## To populate

Use [template](../_template/).
