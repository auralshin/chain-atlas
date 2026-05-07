# StarkNet

**Status:** stub
**Family:** ZK rollup — Cairo VM (not EVM)
**Mainnet launch:** 2021-11 (alpha)
**Native node clients:** pathfinder, juno, papyrus

## Anchor facts (verify in deep dive)

- **Not EVM.** Cairo VM, custom contract model, Cairo language.
- Native account abstraction — every account is a contract; no EOAs.
- Custom JSON-RPC schema — does not match `eth_*` methods.
- Indexer logic shares almost nothing with Ethereum-family chains beyond "watch L1 for state commitments".
- Treat as a non-EVM chain that happens to settle on Ethereum.

## To populate

Use [template](../_template/). Bulk of the doc is "what's different from EVM".
