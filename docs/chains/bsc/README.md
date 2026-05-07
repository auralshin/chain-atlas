# BSC (BNB Smart Chain)

**Status:** stub
**Family:** EVM L1 — Proof-of-Staked-Authority sidechain
**Mainnet launch:** 2020-09-01
**Native node clients:** bsc-geth (geth fork)

## Anchor facts (verify in deep dive)

- PoSA consensus with a small validator set (historically 21, expanded since).
- ~3-second blocks; recent upgrades (Maxwell / Lorentz) altered timing — confirm current cadence.
- Reorgs deeper than people assume — naive Ethereum-style indexers misclassify state. Empirical depth needs sourcing.
- High throughput → PG hits limits faster here than on Ethereum.

## To populate

Use [template](../_template/). Reorg depth is the headline gotcha.
