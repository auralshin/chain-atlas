# RPC Surface

Arbitrum Nitro RPC = standard `eth_*` + `arb_*` + `arbtrace_*` (Erigon-style flat traces) + `nodeinterface_*` (precompile RPC helpers). Plus a separate WebSocket sequencer feed.

## Standard ETH JSON-RPC

All `eth_*`, `web3_*`, `net_*` methods work. Notable differences:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Includes `ArbitrumInternalTx` (`0x6A`) as the first tx of every L2 block. Includes the Arbitrum-specific block fields (`l1BlockNumber`, `sendRoot`, `sendCount`). |
| `eth_getTransactionReceipt` | Returns Arbitrum-specific fields: `gasUsedForL1`, `effectiveGasPrice`, `l1BlockNumber`. |
| `eth_call` / `eth_estimateGas` | Estimates **include** the L1 component (unlike OP Stack where it's separate). |
| `eth_feeHistory` | L2 base fee history. ArbOS-specific gas pricing means this can move differently from L1 patterns. |
| `eth_blockNumber` | Returns the latest L2 block. There is no separate "unsafe vs safe" RPC at the standard level — use `eth_getBlockByNumber("safe")` / `eth_getBlockByNumber("finalized")` for those. |
| `eth_subscribe("newHeads")` | Subscribes to the standard L2 head. Note this is **not** the sequencer feed. |

## Arbitrum-specific (`arb_*`)

Mostly diagnostic / informational methods. Not required for indexing but useful:

| Method | Returns | Why |
|---|---|---|
| `arb_getRawTransactionByHash` | Raw RLP of a tx | For replaying / debugging |
| `arbnode_getMessage` | Inbox message at index | For reconstructing the chain from L1 |
| `arbnode_getLatestSequencerInboxMessage` | Latest L1-batched message index | Liveness check |
| `nodeinterface_constructOutboxProof` | Merkle proof for an outbox message | Required for users to call `Outbox.executeTransaction` |
| `nodeinterface_estimateRetryableTicket` | Gas estimation for retryables | For UI flows |
| `nodeinterface_lookupMessageBatchProof` | Sub-proof inside a batch | Used by withdrawal-proving frontends |

`NodeInterface` precompile methods are accessed by RPC, not by on-chain calls — Arbitrum special-cases this. See [data-model.md](data-model.md#predeploys--system-addresses-arbos-precompiles).

## Trace APIs

Two flavors:

| Method | Format | Available on |
|---|---|---|
| `debug_traceBlockByNumber` | geth-style call/struct/prestate tracers | Most archive Arbitrum nodes |
| `arbtrace_block`, `arbtrace_transaction` | Parity-style flat traces | Specifically supported by `nitro` |
| `arbtrace_call`, `arbtrace_callMany` | Speculative flat traces | Same |

`arbtrace_*` is a Nitro extension that mirrors the Parity/OpenEthereum trace format. **Most third-party indexer tooling uses these on Arbitrum** because they're cheaper than `debug_traceBlockByNumber` and produce the canonical "internal txs" view.

System / deposit / retryable txs are traceable like any other tx. The `from` will be the L1-aliased address (or an ArbOS internal address for `0x6A`).

## Sequencer broadcast feed

A WebSocket separate from the standard JSON-RPC, exposing the sequencer's view of new blocks **before** they are batched to L1. Used by:
- Arbitrum nodes following the chain in fast (sequencer-feed) mode.
- Specialized indexers needing sub-second latency.

The wire format is Arbitrum-specific (compressed feed messages, not JSON-RPC). For most indexers, the standard `eth_subscribe("newHeads")` is sufficient — it follows the sequencer feed internally.

## Archive node requirements

Archive on Nitro = full historical state for `eth_call` / `debug_*` / `arbtrace_*` at any block, post-Nitro-genesis. Pre-Nitro Classic state is generally **not** in the same archive — it's a separate, legacy-format archive that few providers serve.

Archive disk size for Arbitrum One is in the multi-TB range as of 2026 {{unsourced: cite specific number}}.

## Public provider quirks

- **Block range limits** on `eth_getLogs`: typical 1k–10k blocks. Arbitrum's variable block time means "10k blocks" can be anywhere from 30 min to several hours of history.
- **`eth_getBlockReceipts`** is supported on most providers; dramatically cheaper than per-tx receipts for ingestion.
- **`arbtrace_*` is provider-dependent**: Alchemy, Infura, QuickNode all expose it on archive tier; some smaller providers do not.
- **Reorg-aware subscriptions**: `eth_subscribe("newHeads")` does not emit reorg events. Detection is up to the indexer.

## Picking a transport

| Transport | Use when |
|---|---|
| HTTP | Bulk historical scans (`eth_getLogs`, `eth_getBlockReceipts`, `arbtrace_block`) |
| WebSocket (`eth_subscribe`) | Real-time `newHeads` + `logs` |
| WebSocket (sequencer feed) | Sub-second latency requirements only |
| IPC (local) | Self-hosted indexer co-located with `nitro` |

Self-hosted production typical: archive `nitro` + IPC or local HTTP, with the L1 archive node accessible via separate connection for `SequencerInbox` / `Bridge` / `Outbox` / `RollupProxy` ingestion.
