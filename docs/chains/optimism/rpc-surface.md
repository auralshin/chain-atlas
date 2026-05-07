# RPC Surface

OP Stack node = `op-geth` (EL) + `op-node` (CL). They speak different RPC namespaces and an indexer typically queries **both**.

## Standard ETH JSON-RPC (`op-geth`)

All standard `eth_*`, `web3_*`, `net_*` methods work as on Ethereum. Notable:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Includes the deposit system tx as the first tx of every block. |
| `eth_getTransactionReceipt` | Returns OP-specific fields (`l1Fee`, `l1GasUsed`, `l1GasPrice`, `l1FeeScalar`, `l1BlobBaseFee`, `l1BaseFeeScalar`, `l1BlobBaseFeeScalar`). Ecotone changed the field set — version your decoder. |
| `eth_call` / `eth_estimateGas` | L1 fee is *not* in the gas estimate. Wallets must add it separately via `GasPriceOracle.getL1Fee()`. |
| `eth_feeHistory` | L2 base fee history. Does **not** include L1 component. |
| `eth_blockNumber` | Returns **unsafe** head by default. Use `safe` / `finalized` block tags for the others. |
| `eth_getBlockByNumber("safe")` | Returns the safe head. Backed by `op-node`'s notion of safe. |
| `eth_getBlockByNumber("finalized")` | Returns the finalized head. |

Block tags `safe` and `finalized` are supported and **mean the same as on `op-node`**: you can use them at the EL without a separate CL query for most cases. Use the CL when you need the L1 origin or sync diagnostics.

## Trace APIs (`op-geth`)

OP Stack's `op-geth` is a fork of `geth` and supports the same trace methods — but most public providers expose only a subset.

| Method | Available | Cost |
|---|---|---|
| `debug_traceBlockByNumber` | Yes (`callTracer`, `prestateTracer`, `4byteTracer`, custom JS) | Heavy. Archive only. |
| `debug_traceTransaction` | Yes | Heavy. Archive only. |
| `debug_traceCall` | Yes | Speculative — runs against historical state if archive. |
| Parity-style `trace_*` | Only on Erigon-based clients (`op-erigon`) | Erigon's flat-trace format. |
| `ots_*` (Otterscan) | If client is `op-erigon` or patched `op-geth` | Same semantics as ETH Otterscan. |

System / deposit txs are traced like any other tx, but the `from` will be the L1-aliased address (or the OP system address `0xDeaDDEaDDeAdDeAdDEAdDEaddeAddEAdDEAd0001` for system deposits). Tracers that expect a recoverable signer for `from` will need to special-case type `0x7E`.

## Rollup CL methods (`op-node`)

Exposed on the rollup-node JSON-RPC port (default 9545):

| Method | Returns | Why an indexer cares |
|---|---|---|
| `optimism_syncStatus` | Current `unsafe`/`safe`/`finalized` heads (L1 + L2) | Authoritative source for finality state. |
| `optimism_outputAtBlock` | Output root + state root for a given L2 block | Use to verify L2-state inclusion in L1-posted output roots. |
| `optimism_rollupConfig` | Static rollup config (chain id, batch sender, fork timestamps) | Resolve fork activations at runtime. |
| `optimism_version` | Software version | Operational. |
| `admin_*` | Add peers, etc. | Operational, not for indexers. |

`optimism_syncStatus` is the single most-called RPC for an OP Stack indexer's finality logic. Poll it every 2–6 seconds.

## Engine API

`op-node` ↔ `op-geth` is connected via the standard Ethereum Engine API (`engine_newPayloadVN`, `engine_forkchoiceUpdatedVN`). **Indexers should not call this** — it's an internal control-plane API, not a query API. If a provider exposes it, that's a misconfiguration.

## Archive node requirements

Archive on OP Stack means the same as on Ethereum: full historical state for `eth_call` / `debug_*` at any block. Caveats:

- The pre-Bedrock data set is a **separate** archive. Modern archive nodes either contain a converted snapshot of pre-Bedrock state or simply do not serve those blocks. Confirm with your provider.
- Erigon-style flat-trace archives are smaller than geth archives. For trace-heavy workloads (call traces, internal txs), consider `op-erigon`.

## Public provider quirks (operational, not in spec)

These bite indexers in practice:

- **Block range limits** on `eth_getLogs` are common (often 1k–10k blocks). At 2-second blocks that's only minutes of history per call. Plan for batched scans.
- **Receipts in batches**: `eth_getBlockReceipts` is supported on most OP Stack endpoints and is dramatically cheaper than per-tx receipts.
- **Reorg-aware subscriptions**: `eth_subscribe("newHeads")` only emits heads, not reorgs. Reorg detection is your problem (see [reorgs-finality.md](reorgs-finality.md)).
- **Free-tier rate limits**: vary wildly between providers. Always set per-method backoff.

## Picking a transport

| Transport | Use when |
|---|---|
| HTTP | Bulk historical scans (`eth_getLogs`, `eth_getBlockReceipts`). |
| WebSocket | Real-time `newHeads` + `logs` subscriptions; `optimism_syncStatus` polls. |
| IPC (local) | Indexer co-located with the node — fastest, no JSON encoding overhead. Recommended for self-hosted. |

For self-hosted production indexing, IPC + a local archive `op-geth` + a local `op-node` is the canonical setup.
