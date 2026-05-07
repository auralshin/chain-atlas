# Polygon PoS

**Status:** stub
**Family:** EVM sidechain — PoS with Ethereum-checkpointed finality
**Mainnet launch:** 2020-05-30 (PoS chain; the project predates this as Matic)
**Native node clients:** bor (execution) + heimdall (consensus)

## Anchor facts (verify in deep dive)

- Two-component node: `bor` runs EVM, `heimdall` (Tendermint-derived) handles consensus and Ethereum checkpoints.
- Finality story: probabilistic on `bor`; L1 checkpoint posted to Ethereum at intervals.
- Reorg depth historically large compared to Ethereum — folklore says ~128 blocks; verify with primary source.
- Polygon zkEVM is a **separate** chain — do not conflate.

## To populate

Use [template](../_template/).
