# RPC Surface

Standard ETH JSON-RPC. Linea exposes a small `linea_*` namespace for chain-specific extensions.

## Standard ETH JSON-RPC

All `eth_*` methods. Same caveats as [scroll/rpc-surface.md](../scroll/rpc-surface.md):
- Receipts include L1-fee fields.
- `eth_call` / `eth_estimateGas` exclude the L1 component (use the fee oracle).
- `eth_getBlockByNumber("safe"/"finalized")` maps to L1-committed / L1-finalized {{unsourced: confirm}}.

## Linea-specific (`linea_*`)

| Method | Returns | Why |
|---|---|---|
| `linea_estimateGas` | Gas estimate including L1 fee component | Convenience over standard `eth_estimateGas` + manual L1-fee addition |
| `linea_getProof` {{unsourced: confirm}} | Storage proof at a block | For verifiers and bridge tooling |
| `linea_getTransactionExclusionStatusV1` {{unsourced: confirm name and existence}} | Sequencer-reject info for excluded txs | Operational; helps debug "why isn't my tx included" |

`linea_estimateGas` is genuinely useful for indexers building UIs. Standard `eth_estimateGas` returns an L2-only estimate; Linea's variant includes the L1 fee component.

## Trace APIs

Besu-based traces. `debug_*` methods work but with **Besu's trace shape**, which differs slightly from geth's:
- `debug_traceTransaction` available with Besu's tracers.
- Built-in tracers may have different names/options than geth.
- Some custom JS tracers may not work depending on Besu's tracer engine.

For most indexers, the standard `debug_traceBlockByNumber` with `callTracer` works on Linea, but verify behavior against your specific Besu version.

## Archive node requirements

Standard EVM archive profile. Linea archive disk is in the multi-hundred-GB to low-TB range as of 2026 {{unsourced: confirm}}.

Most production indexers use Consensys-operated RPC (Infura) for Linea archive access. Self-hosting Besu archive is possible but has Besu-specific operational characteristics.

## Public provider quirks

- **Public RPC** at `https://rpc.linea.build/` — rate-limited.
- **Block range limits** on `eth_getLogs`: typical 1k–10k blocks.
- **Infura's Linea endpoint** is the canonical paid option (Consensys eats own dog food).
- **`eth_getBlockReceipts`** support: confirmed on most providers; faster than per-tx receipts.

## Picking a transport

Same as Scroll. HTTP for bulk scans, WS for real-time, IPC for self-hosted.
