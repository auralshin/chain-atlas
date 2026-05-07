# RPC Surface

Standard Ethereum JSON-RPC. BSC's `bsc` client is a `geth` fork; the RPC surface mirrors `geth`.

## Standard ETH JSON-RPC

All `eth_*`, `web3_*`, `net_*` methods work as on Ethereum. Notable:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Standard. `extraData` carries validator-set + BLS info at epoch boundaries. |
| `eth_getTransactionReceipt` | Standard. No BSC-specific fields. |
| `eth_getBlockByNumber("safe")` | Post-Plato: justified head. |
| `eth_getBlockByNumber("finalized")` | Post-Plato: finalized head. |
| `eth_subscribe("newHeads")` | Standard. |
| `eth_feeHistory` | Post-London: standard. |

`safe` and `finalized` are reliable signals only **post-Plato fork**. Pre-Plato historical queries with these tags either error or return `head` depending on client version.

## BSC-specific (`parlia_*` / `bsc_*`)

Small namespace, mostly diagnostic:

| Method | Returns | Why |
|---|---|---|
| `parlia_getValidators` | Active validator set at a height | For validator-rotation indexing |
| `parlia_getJustifiedNumber` | Highest justified block (post-Plato) | Verifying finality state |
| `parlia_getFinalizedNumber` | Highest finalized block (post-Plato) | Same |

Names vary by client version; `eth_getBlockByNumber("safe"/"finalized")` is more portable.

## Trace APIs

`bsc` is a `geth` fork. Standard `debug_*` traces work:
- `debug_traceBlockByNumber`
- `debug_traceTransaction`
- `debug_traceCall`
- All standard tracers: `callTracer`, `prestateTracer`, `4byteTracer`, custom JS

Erigon-style `trace_*` flat traces are available on Erigon-equivalent BSC clients (third-party builds; not officially supported by `bnb-chain/bsc`) {{unsourced: confirm}}.

## Archive node requirements

BSC archive disk size is **substantial** — multi-TB and growing fast given high tx volume. Many providers price BSC archive higher than Ethereum archive due to disk pressure.

Most production indexers that need archive run a self-hosted `bsc` archive node, with backups, on dedicated hardware. Public-provider archive on BSC is expensive and rate-limited.

## Public provider quirks

- **`eth_getLogs` block range limits** are extremely tight on BSC public RPCs — often 500–2000 blocks per call. Plan for many small windows.
- **Public RPC at `https://bsc-dataseed.bnbchain.org/`** (JSON-RPC POST endpoint; root GET returns 404 — that's expected) — multiple round-robin endpoints exist; rate-limited; not for serious indexing.
- **WebSocket reorg behavior**: `eth_subscribe("newHeads")` follows the head. Reorgs surface implicitly via parent-hash mismatch.
- **`eth_getBlockReceipts`** is supported on most providers; faster than per-tx receipts at scale. Use it.
- **High request rate**: BSC's high throughput plus tight per-call limits means an indexer makes **many** concurrent requests. Provision connection pools accordingly.

## Picking a transport

| Transport | Use when |
|---|---|
| HTTP | Bulk historical scans (`eth_getLogs`, `eth_getBlockReceipts`) |
| WebSocket | Real-time `newHeads` + `logs` |
| IPC (local) | Self-hosted indexer co-located with `bsc` — strongly recommended for production |

Self-hosted production: archive `bsc` + IPC. Anything else hits rate limits at scale.
