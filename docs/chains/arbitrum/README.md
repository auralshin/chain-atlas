# Arbitrum One

**Status:** stub
**Family:** EVM L2 — Nitro optimistic rollup
**Mainnet launch:** 2021-08-31 (Arbitrum One); Nitro upgrade 2022-08-31
**Native node clients:** nitro

## Anchor facts (verify in deep dive)

- Optimistic rollup with custom ArbOS layer above EVM.
- Custom precompiles for L1↔L2 messaging (e.g. `ArbSys`, `ArbAddressTable`).
- `eth_getBlockByNumber` returns blocks at the L2 cadence; correlation to L1 batches is via separate APIs.
- Nitro's data format differs from pre-Nitro Classic; historical-data indexers need to handle the transition.
- Stylus (Wasm contracts) adds a non-EVM execution path — relevant for full-coverage indexers.

## To populate

Use [template](../_template/).
