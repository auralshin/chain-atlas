# RPC Surface

Standard ETH JSON-RPC. Scroll's `l2geth` is a `geth` fork; the RPC surface mirrors `geth` with minor Scroll-specific extensions.

## Standard ETH JSON-RPC

All `eth_*` methods work as on Ethereum. Notable:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Standard. |
| `eth_getTransactionReceipt` | Standard ETH receipt **plus L1-fee fields** (`l1Fee`, `l1GasUsed`, `l1GasPrice`, `l1FeeScalar`, `l1BlobBaseFee` post-blob-DA). |
| `eth_call` / `eth_estimateGas` | L1 fee is **not** in the gas estimate. Use `L1GasPriceOracle.getL1Fee()` for that, similar to OP Stack. |
| `eth_getBlockByNumber("safe"/"finalized")` | Maps to L1-committed / L1-finalized batch states. |
| `eth_subscribe("newHeads")` | Standard. |

## Scroll-specific (`scroll_*`)

Limited namespace; mostly diagnostic:

| Method | Returns | Why |
|---|---|---|
| `scroll_clientVersion` | Client version string | Operational |
| `scroll_blockTraces` {{unsourced: confirm name}} | Per-block trace data optimized for prover input | Internal; rarely needed by indexers |

Most Scroll-specific data is exposed via standard `eth_*` methods; there's no `zks_*`-style rich namespace.

## Trace APIs

`l2geth` is a geth fork; standard `debug_*` traces work:
- `debug_traceBlockByNumber`
- `debug_traceTransaction`
- `debug_traceCall`
- All standard tracers

Trace output is identical to Ethereum's because Scroll runs the same EVM bytecode.

## Archive node requirements

Standard ETH archive profile. Scroll archive nodes are run by Scroll Foundation and a few third-party providers.

## Public provider quirks

- **Block range limits** on `eth_getLogs`: typical 1k–10k blocks; varies by provider.
- **`eth_getBlockReceipts`** is supported on most providers; faster than per-tx receipts.
- **Public RPC at `https://rpc.scroll.io/`** — rate-limited; not for serious indexing.
- **WebSocket reorg behavior**: `eth_subscribe("newHeads")` follows the L2 head; reorgs surface via parent-hash mismatch.

## Picking a transport

| Transport | Use when |
|---|---|
| HTTP | Bulk historical scans |
| WebSocket | Real-time `newHeads` + `logs` |
| IPC | Self-hosted; possible but uncommon for Scroll |

For most indexers, a hosted provider (Alchemy, BlockPI, official Scroll RPC) is sufficient. Scroll's volume is moderate, well within typical provider capacity.
