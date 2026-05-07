# References

## Source code

- **`linea-besu`** — [github.com/Consensys/linea-besu](https://github.com/Consensys/linea-besu) — Besu fork, L2 execution client.
- **`linea-geth`** — [github.com/Consensys/linea-geth](https://github.com/Consensys/linea-geth) {{unsourced: confirm repo}} — geth fork used by sequencer.
- **`linea-contracts`** — [github.com/Consensys/linea-contracts](https://github.com/Consensys/linea-contracts) — L1 + L2 contracts (`ZkEvm`, `MessageService`, `TokenBridge`, predeploys).
- **Prover / circuits** — Multiple Consensys repos; not needed for indexing.

## Documentation

- **Linea docs** — [docs.linea.build](https://docs.linea.build/) — official; covers architecture, contract addresses, RPC, fork timeline.

## Block explorers

- **Lineascan** — [lineascan.build](https://lineascan.build/) — Etherscan-family.

## Operational

- **Linea status** — [status.linea.build](https://status.linea.build/) {{unsourced: confirm URL}}.
- **L2BEAT Linea page** — [l2beat.com/scaling/projects/linea](https://l2beat.com/scaling/projects/linea) — independent monitor.
- **Public RPC** — `https://rpc.linea.build/` — rate-limited.
- **Infura Linea endpoint** — recommended paid option (Consensys-operated).

## Type libraries

No Linea-specific Rust types crate. Use `alloy` for everything; supplement receipts with the L1-fee fields (same as for Scroll).

## Inherited references

For ZK-EVM concepts (proof systems, finalization windows, ZK rollup architecture), the canonical sources are in:
- [scroll/references.md](../scroll/references.md) for general ZK-EVM rollup operational patterns.
- [zksync-era/references.md](../zksync-era/references.md) for ZK rollup specs in the broader sense.

## Where this doc is incomplete

The following Linea-specific claims carry `{{unsourced}}` markers:

- Beta v1, Beta v2, Pectra-equivalent fork dates / heights.
- Linea predeploy addresses (L2 message service, fee oracle, gateway).
- Exact event names emitted by `ZkEvm.sol` for batch commit / finalize.
- Vortex/PLONK proof system specifics.
- Fee burn percentage (claimed 20%) and exact mechanism.
- L2 block time (claimed ~2 s).
- Whether `linea_getProof`, `linea_getTransactionExclusionStatusV1` exist as stated.
- Set of opcodes excluded in early Linea and per-fork coverage timeline.
- `eth_getBlockByNumber("safe"/"finalized")` behavior on Linea.
- Force-include / inclusion-guarantee mechanism details.

Verify against the Linea docs, the contracts repo, and L2BEAT's Linea timeline before promoting this doc to `deep`.
